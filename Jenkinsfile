pipeline {
    agent any

    tools {
        jdk 'JDK 25' 
        maven 'Maven3.9.11' 
    }

    environment {
        TEST_PORT = "9595"
	WOLF_TEST_PORT = "9096" 
        PROD_PORT = "8090"
	WOLF_PROD_PORT = "9092"
        
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2025-demo"
        JENKINS_NODE_COOKIE = "dontKillMe"
        
        TZ = "Asia/Ho_Chi_Minh"
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo "[Build] Compiling WebGoat v2025.3..."
                    //  Dùng ./mvnw clean install  Maven Wrapper 
                    // Thêm chmod để đảm bảo quyền thực thi
                    sh "chmod +x mvnw"
                    sh "./mvnw clean install -DskipTests -Dmaven.test.skip=true -Dprocess-exec.skip=true"
                }
            }
        }

        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            sh "rm -rf seeker installer.sh || true"
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        }
                    }
                }
            }
        }

        stage('3. Run App with Seeker (Test)') {
            steps {
                script {
                    echo " [Run] Starting WebGoat 2025 (Test Mode on Port ${TEST_PORT})..."

                    // Tìm file JAR 
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | head -n 1', returnStdout: true).trim()
                    if (!webgoatJar) error "❌ ERROR: No JAR file found!"
                    
		    sh """
                        echo ">>> Cleaning up ports ${TEST_PORT} and ${WOLF_TEST_PORT}..."
                        fuser -k ${TEST_PORT}/tcp || true
                        fuser -k ${WOLF_TEST_PORT}/tcp || true
                        sleep 5
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            nohup java \
                                -Dfile.encoding=UTF-8 \
                                -Duser.timezone=${TZ} \
                                --add-opens java.base/java.lang=ALL-UNNAMED \
                                -Xmx2g \
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
                                -jar ${webgoatJar} \
                                --server.address=0.0.0.0 \
                                --webgoat.port=${TEST_PORT} \
                                --webwolf.port=${WOLF_TEST_PORT} \
                                > app_webgoat_test.log 2>&1 < /dev/null &
                        """
                    }
                }
            }
        }
	stage('4. Health Check & Traffic') {
            steps {
                script {
                    echo "[Check] Waiting for WebGoat (9595) & WebWolf (9096)..."
                    // Tăng thời gian chờ lên tối đa 5 phút (WebWolf khởi động khá lâu)
                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            script {
                                def statusGoat = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                                def statusWolf = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                                
                                echo "   ...Ping Status -> Goat: ${statusGoat} | Wolf: ${statusWolf}"
                                
                                // Chấp nhận 200 (OK) hoặc 302 (Redirect)
                                return (statusGoat == '200' || statusGoat == '302') && (statusWolf == '200' || statusWolf == '302')
                            }
                        }
                    }
                    echo "✅ Services are UP! Starting Advanced Traffic Generation..."

                    sh """
                        # Xóa cookie cũ
                        rm -f cookies.txt cookies_wolf.txt goat_login.html wolf_login.html payload.txt
                        
                        # Tạo payload giả để test upload và put
                        echo "Seeker Test Data" > payload.txt

                        # ====================================================
                        # 1. WEBGOAT TARGETING (Port ${TEST_PORT})
                        # ====================================================
                        echo ">>> [WebGoat] Login & CSRF Setup..."
                        # Lấy Cookie & CSRF Token
                        curl -s -c cookies.txt http://127.0.0.1:${TEST_PORT}/WebGoat/login > goat_login.html
                        GOAT_CSRF=\$(grep -oP 'name="_csrf" value="\\K[^"]+' goat_login.html || echo "none")

                        curl -s -c cookies.txt -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                             -d "username=webgoatadmin&password=password&_csrf=\$GOAT_CSRF" \\
                             -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        echo ">>> [WebGoat] Hitting Specific Missing Endpoints..."
                        
                        # Target: /WebGoat/service/debug/labels.mvc (PUT & POST)
                        # Endpoint này nhận JSON
                        curl -s -b cookies.txt -X POST -H "X-CSRF-TOKEN: \$GOAT_CSRF" -H "Content-Type: application/json" -d '{}' http://127.0.0.1:${TEST_PORT}/WebGoat/service/debug/labels.mvc || true
                        curl -s -b cookies.txt -X PUT  -H "X-CSRF-TOKEN: \$GOAT_CSRF" -H "Content-Type: application/json" -d '{}' http://127.0.0.1:${TEST_PORT}/WebGoat/service/debug/labels.mvc || true

                        # Target: /WebGoat/SecurityMisconfiguration/task2/config (GET)
                        curl -s -b cookies.txt -X GET http://127.0.0.1:${TEST_PORT}/WebGoat/SecurityMisconfiguration/task2/config || true

                        # Target: /WebGoat/crypto/hashing/md5 (HEAD)
                        # curl -I gửi method HEAD
                        curl -s -b cookies.txt -I http://127.0.0.1:${TEST_PORT}/WebGoat/crypto/hashing/md5 || true

                        # Target: /WebGoat/JWT/secret/gettoken (CONNECT)
                        # Thêm timeout (-m 2) vì CONNECT thường treo connection
                        curl -s -b cookies.txt -X CONNECT -m 2 http://127.0.0.1:${TEST_PORT}/WebGoat/JWT/secret/gettoken || true


                        # ====================================================
                        # 2. WEBWOLF TARGETING (Port ${WOLF_TEST_PORT})
                        # ====================================================
                        echo ">>> [WebWolf] Login & CSRF Setup..."
                        curl -s -c cookies_wolf.txt http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login > wolf_login.html
                        WOLF_CSRF=\$(grep -oP 'name="_csrf" value="\\K[^"]+' wolf_login.html || echo "none")

                        curl -s -c cookies_wolf.txt -X POST http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login \\
                             -d "username=webgoatadmin&password=password&_csrf=\$WOLF_CSRF" \\
                             -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        echo ">>> [WebWolf] Fuzzing /landing & /file-server-location..."

                        # Loop qua TẤT CẢ method có trong hình (GET, POST, PUT, DELETE, PATCH, TRACE, OPTIONS, HEAD, CONNECT)
                        for method in GET POST PUT DELETE PATCH TRACE OPTIONS HEAD CONNECT; do
                            
                            # A. Target /landing
                            curl -s -b cookies_wolf.txt -X \$method -m 2 \\
                                 -H "X-CSRF-TOKEN: \$WOLF_CSRF" \\
                                 http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/landing > /dev/null || true

                            # B. Target /file-server-location (Simulate File Upload)
                            # Gửi kèm file payload để request hợp lệ hơn với endpoint này
                            curl -s -b cookies_wolf.txt -X \$method -m 2 \\
                                 -H "X-CSRF-TOKEN: \$WOLF_CSRF" \\
                                 -F "file=@payload.txt" \\
                                 http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/file-server-location > /dev/null || true
                        done

                        echo ">>> [WebWolf] Hitting /mail (DELETE)..."
                        curl -s -b cookies_wolf.txt -X DELETE -H "X-CSRF-TOKEN: \$WOLF_CSRF" http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/mail || true

                        # ====================================================
                        # 3. TRIGGER ERROR PATHS (\${server.error.path})
                        # ====================================================
                        echo ">>> [Global] Triggering Error Pages..."
                        
                        # Cách 1: Gọi trang không tồn tại (404) để trigger error path
                        curl -s -b cookies_wolf.txt http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/force-404-error || true
                        
                        # Cách 2: Gọi trực tiếp endpoint error mặc định của Spring
                        curl -s -b cookies_wolf.txt -H "Accept: application/json" http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/error || true
                        
                        # Cách 3: Gửi Bad Request (400) để trigger error path
                        curl -s -b cookies_wolf.txt -X POST -H "Content-Type: application/json" -d "{bad-json" http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/landing || true

                        echo ">>> Traffic Generation Finished!"
                    """
                    
                    sleep 5
                    echo "[Cleanup] Stopping Test Processes..."
                    sh """
                        fuser -k ${TEST_PORT}/tcp || true
                        fuser -k ${WOLF_TEST_PORT}/tcp || true
                    """
                }
            }
        }
	stage('5. Quality Gate') {
            steps {
                script {
                    echo "[Gate] Checking Seeker Compliance..."
                    sleep 10 
                    
                    int maxCritical = 100
                    int maxHigh = 100

                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        def apiUrl = "http://192.168.12.190:8082/rest/api/latest/vulnerabilities?format=JSON&projectKeys=${SEEKER_PROJECT_KEY}&status=DETECTED&minSeverity=HIGH"
                        
                        echo "Querying Seeker API: ${apiUrl}"

                        try {
                            def response = sh(
                                script: 'curl -s -k -X GET -H "Authorization: $SEEKER_API_TOKEN" -H "accept: */*" "' + apiUrl + '"',
                                returnStdout: true
                            ).trim()

                            if (!response || (!response.startsWith("{") && !response.startsWith("["))) {
                                echo "Raw Response: ${response}"
                                error "Invalid Response Format: Server did not return JSON."
                            }

                            def jsonResult = new groovy.json.JsonSlurper().parseText(response)
                            
                            def vulnList = []
                            if (jsonResult instanceof List) {
                                vulnList = jsonResult
                            } else if (jsonResult instanceof Map) {
                                if (jsonResult.containsKey('content')) vulnList = jsonResult.content
                                else if (jsonResult.containsKey('vulnerabilities')) vulnList = jsonResult.vulnerabilities
                                else if (jsonResult.containsKey('code') && jsonResult.code != 200) {
                                     error "API Error: ${jsonResult.message}"
                                }
                            }

                            echo "📊 Found ${vulnList.size()} vulnerabilities."

                            int criticalCount = 0
                            int highCount = 0
                            def failReasons = []

                            vulnList.each { vuln ->
                                String currentSev = vuln.Severity ? vuln.Severity.toString().trim().toUpperCase() : "UNKNOWN"

                                if (currentSev == 'CRITICAL') {
                                    criticalCount++
                                    failReasons.add("[CRITICAL] ${vuln.VulnerabilityName}")
                                }
                                if (currentSev == 'HIGH') {
                                    highCount++
                                    failReasons.add("[HIGH] ${vuln.VulnerabilityName}")
                                }
                            }

                            echo "Seeker Report Summary:"
                            echo "   - Critical: ${criticalCount} / Allowed: ${maxCritical}"
                            echo "   - High:     ${highCount} / Allowed: ${maxHigh}"

                            if (criticalCount > maxCritical || highCount > maxHigh) {
                                echo "Details of violations:"
                                failReasons.each { echo "   - ${it}" }
                                error "Quality Gate FAILED: Found ${criticalCount} Critical & ${highCount} High vulnerabilities."
                            } else {
                                echo "Quality Gate PASSED."
                            }

                        } catch (Exception e) {
                            echo "Script Error: ${e.getMessage()}"
                            error "Failed to verify Quality Gate."
                        }
                    }
                }
            }
        }
	stage('6. Deploy to Production') {
            steps {
                script {
                    echo "[Deploy] Deploying v2025.3 to Production on Port ${PROD_PORT}..."
                    def deployDir = "${WORKSPACE}/deploy_prod" 
                    
                    // Tìm JAR (loại bỏ "original" và "deploy_prod")
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "deploy_prod" | head -n 1', returnStdout: true).trim()

                    if (!webgoatJar) {
                        error "ERROR: No new WebGoat JAR file found!"
                    }
                    echo "Found JAR: ${webgoatJar}"

                    // 1. Dọn dẹp & Chuẩn bị thư mục (Thêm tạo webwolf-data và cấp quyền)
                    sh """
                        echo "Cleaning up ports ${PROD_PORT} and ${WOLF_PROD_PORT}..."
                        fuser -k ${PROD_PORT}/tcp || true
                        fuser -k ${WOLF_PROD_PORT}/tcp || true
                        sleep 2
                        
                        mkdir -p ${deployDir}/webgoat-data
                        mkdir -p ${deployDir}/webwolf-data
                        cp ${webgoatJar} ${deployDir}/webgoat-app.jar
                        cp -r seeker/* ${deployDir}/seeker/
                        
                        # Cấp quyền ghi để tránh lỗi 500 khi upload file
                        chmod -R 777 ${deployDir}/webgoat-data
                        chmod -R 777 ${deployDir}/webwolf-data
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            echo "Starting WebGoat & WebWolf (Prod)..."
                            
                            nohup java -Xmx2g \\
                                -Dfile.encoding=UTF-8 \\
                                -Duser.timezone=${TZ} \\
                                -javaagent:${deployDir}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -jar ${deployDir}/webgoat-app.jar \\
                                --server.address=0.0.0.0 \\
                                --webgoat.port=${PROD_PORT} \\
                                --webwolf.port=${WOLF_PROD_PORT} \\
                                --webwolf.address=0.0.0.0 \\
                                --webgoat.server.directory=${deployDir}/webgoat-data \\
                                --webwolf.server.directory=${deployDir}/webwolf-data \\
                                > ${deployDir}/app_webgoat_prod.log 2>&1 < /dev/null &
                        """
                    }

                    // 2. Đợi Server lên (Health Check cho cả 2)
                    echo "Waiting for WebGoat & WebWolf to initialize..."
                    boolean prodReady = false
                    
                    for (int i = 1; i <= 60; i++) {
                        def gStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PROD_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                        def wStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_PROD_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                        
                        if (gStatus == '200' && (wStatus == '200' || wStatus == '302')) {
                            echo "Both servers are UP!"
                            // Tự động đăng ký admin
                            sh """
                                curl -s -k -X POST http://127.0.0.1:${PROD_PORT}/WebGoat/register.mvc \\
                                    -d "username=webgoatadmin&password=password&matchingPassword=password&agree=agree" \\
                                    -H "Content-Type: application/x-www-form-urlencoded"
                            """
                            prodReady = true
                            break
                        }
                        echo "Waiting... (Goat: ${gStatus}, Wolf: ${wStatus}) [${i}/60]"
                        sleep 5
                    }
                    
                    if (!prodReady) {
                        sh "cat ${deployDir}/app_webgoat_prod.log || echo 'Log file not found!'"
                        error "Deployment Failed: Production server did not start properly."
                    }
                    echo "Deployment Process Finished!"
                }
            }
        }
    } // Kết thúc khối stages
    post {
        always {
             archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
        }
    }
} // Kết thúc khối pipeline
