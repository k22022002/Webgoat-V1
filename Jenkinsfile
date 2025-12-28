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
                    echo "[Check] Waiting for WebGoat & WebWolf (Test Instance)..."
                    boolean isReady = false
                
                    // 1. Health Check Loop
                    for (int i = 1; i <= 60; i++) { 
                        def statusGoat = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                        def statusWolf = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                    
                        if ((statusGoat == '200' || statusGoat == '302') && (statusWolf == '200' || statusWolf == '302')) {
                            isReady = true;
                            echo "✅ All Services are UP! (WebGoat: ${statusGoat}, WebWolf: ${statusWolf})"
                            break;
                        }
                        sleep 5
                    }

                    if (!isReady) error "Timeout: Services did not start properly."

                    echo "[Traffic] Generating SMARTER traffic for Seeker..."
                    
                    sh """
                        rm -f cookies.txt cookies_wolf.txt wolf_page.html
                        
                        # --- 1. WEBGOAT TRAFFIC ---
                        # (Giữ nguyên phần WebGoat vì nó đã chạy ổn)
                        curl -s -k -c cookies.txt -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                             -d "username=webgoatadmin&password=password" \\
                             -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${TEST_PORT}/WebGoat/welcome.mvc

                        # ====================================================
                        # --- 2. WEBWOLF TRAFFIC (NÂNG CAO) ---
                        # ====================================================
                        
                        echo ">>> [WebWolf] 1. Login & Get CSRF Token..."
                        # Lưu trang login về file để tìm CSRF Token
                        curl -s -k -c cookies_wolf.txt -L http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login > wolf_login.html
                        
                        # Trích xuất CSRF Token (tìm dòng chứa _csrf)
                        # WebGoat/WebWolf thường để token trong thẻ input hidden hoặc meta tag
                        CSRF_TOKEN=\$(cat wolf_login.html | grep -oP 'name="_csrf" value="\\K[^"]+' || echo "none")
                        
                        echo "   + CSRF Token Found: \$CSRF_TOKEN"

                        # Thực hiện Login thật sự kèm Token
                        curl -s -k -c cookies_wolf.txt -X POST http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login \\
                             -d "username=webgoatadmin&password=password&_csrf=\$CSRF_TOKEN" \\
                             -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        echo ">>> [WebWolf] 2. Fuzzing /landing (Check Status Code)..."
                        # Test GET (Không cần CSRF)
                        CODE=\$(curl -s -k -b cookies_wolf.txt -w "%{http_code}" -o /dev/null http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/landing)
                        echo "   + GET /landing -> \$CODE"

                        # Test POST/PUT/DELETE (Cần CSRF Header)
                        # Đối với Spring Security, thường cần gửi token qua header X-CSRF-TOKEN hoặc tham số _csrf
                        for method in POST PUT DELETE PATCH; do
                            CODE=\$(curl -s -k -b cookies_wolf.txt -X \$method \\
                                -H "X-CSRF-TOKEN: \$CSRF_TOKEN" \\
                                -w "%{http_code}" -o /dev/null \\
                                http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/landing)
                            echo "   + \$method /landing -> \$CODE"
                        done

                        echo ">>> [WebWolf] 3. Fuzzing /file-server-location (Simulate Upload)..."
                        # Tạo file giả để upload
                        echo "malicious data" > virus.txt
                        
                        # GET request
                        curl -s -k -b cookies_wolf.txt -o /dev/null http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/file-server-location

                        # POST Upload request (Multipart)
                        CODE=\$(curl -s -k -b cookies_wolf.txt -X POST \\
                            -H "X-CSRF-TOKEN: \$CSRF_TOKEN" \\
                            -F "file=@virus.txt" \\
                            -w "%{http_code}" -o /dev/null \\
                            http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/file-server-location)
                        echo "   + POST /file-server-location (Upload) -> \$CODE"
                        
                        # Loop các method lạ khác
                        for method in OPTIONS HEAD TRACE CONNECT; do
                            curl -s -k -b cookies_wolf.txt -X \$method -H "X-CSRF-TOKEN: \$CSRF_TOKEN" -o /dev/null http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/file-server-location || true
                        done

                        echo ">>> [WebWolf] 4. Triggering Errors..."
                        # Request endpoint sai để kích hoạt ErrorController
                        curl -s -k -b cookies_wolf.txt -X GET http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/error-trigger-page > /dev/null 2>&1 || true
                    """
                    
                    echo "[Traffic] Completed. Check logs above for HTTP 200/302 (Success) vs 403 (Failed)."
                    sleep 10 
                    
                    echo "[Cleanup] Stopping Test Instance..."
                    sh """
                        lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true
                        lsof -t -i:${WOLF_TEST_PORT} | xargs -r kill -9 || true
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
