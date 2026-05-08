pipeline {
    agent any
    parameters {
        booleanParam(name: 'FORCE_COVERITY', defaultValue: false, description: 'Tích vào đây nếu muốn chạy quét Coverity Full Scan') 
    }
    tools {
        jdk 'JDK 25' 
        maven 'Maven3.9.11' 
    }

    environment {
        TEST_PORT = "9595"
        WOLF_TEST_PORT = "9096" 
        PROD_PORT = "8099"
        WOLF_PROD_PORT = "9092"
        
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2025-demo"
     
        JENKINS_NODE_COOKIE = "dontKillMe"
        TZ = "Asia/Ho_Chi_Minh"

        // --- Thêm biến cho SRM ---
        SRM_SERVER_URL = "http://192.168.12.190:6060"
        SRM_PROJECT_ID = "1" 
    }
    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo "[Build] Compiling latest WebGoat..."
                    // Dùng ./mvnw clean install Maven Wrapper 
                    // Thêm chmod để đảm bảo quyền thực thi
                    sh "chmod +x mvnw"
                    sh "./mvnw clean install -DskipTests -Dmaven.test.skip=true -Dprocess-exec.skip=true"
                }
            }
        }

        stage('2. Black Duck SCA & Binary Analysis (BDBA)') {
            steps {
                script {
                    echo "[SCA & BDBA] Running Open Source and Binary Analysis..."
                    
                    // 1. Tìm file JAR vừa được build
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | head -n 1', returnStdout: true).trim()
                  
                    if (!webgoatJar) error "❌ ERROR: No JAR file found for BDBA!"
                    echo "Found JAR to scan: ${webgoatJar}"
                    
                    // 2. Chạy Black Duck Detect CLI (Quét cả Source và Binary)
                    withCredentials([string(credentialsId: 'blackduck-api-token', variable: 'BLACKDUCK_API_TOKEN')]) {
                        sh """
                            # Tải Detect CLI
                            curl -k -SL -O https://detect.blackduck.com/detect10.sh && chmod +x detect10.sh
                            
                            ./detect10.sh \\
                                --blackduck.url="https://192.168.12.204" \\
                                --blackduck.api.token="\$BLACKDUCK_API_TOKEN" \\
                                --blackduck.trust.cert=true \\
                                --detect.project.name="${SEEKER_PROJECT_KEY}" \\
                                --detect.project.version.name="latest" \\
                                --detect.binary.scan.file.path="${webgoatJar}" \\
                                --detect.tools=DETECTOR,SIGNATURE_SCAN,BINARY_SCAN
                        """
                    }
                }
            }
        }
	stage('SAST (Coverity)') { 
            when {
                anyOf {
                    triggeredBy 'TimerTrigger' 
                    expression { return params.FORCE_COVERITY == true }
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'coverity-credentials', usernameVariable: 'COV_USER', passwordVariable: 'COV_PASS')]) { // [cite: 23]
		script {
                        echo '--- [Step] Synopsys Coverity SAST ---'
                        def buildVer = "1.0.${env.BUILD_NUMBER}"
                        def covStream = "webgoat-stream" 
                        def covBin = "/home/ubuntu/cov-analysis-linux64-2025.9.2/bin"
                        def covUrl = "http://192.168.12.191:8081"

                        // Dọn dẹp thư mục cũ
                        sh "rm -rf idir"
                        
                        // Sử dụng CLI mới tự động nhận diện và capture dự án Maven
                        sh "${covBin}/coverity capture --project-dir . --dir idir"
                        
                        // Phân tích mã nguồn
                        sh "${covBin}/cov-analyze --dir idir --all --webapp-security --strip-path \$(pwd)"

                        // Commit kết quả lên Coverity Server
                        sh """
                            ${covBin}/cov-commit-defects --dir idir \
                            --url ${covUrl} \
                            --stream ${covStream} \
                            --user \$COV_USER --password \$COV_PASS \
                            --version "${buildVer}" \
                            --description "WebGoat Build ${env.BUILD_NUMBER}" 
                        """
                        
                        // Xuất báo cáo
                        sh "${covBin}/cov-format-errors --dir idir --html-output coverity-report"
                        sh "${covBin}/cov-format-errors --dir idir --json-output-v7 coverity_results.json"
                        
                        def defectCount = sh(script: "jq '.issues | length' coverity_results.json", returnStdout: true).trim().toInteger()
                        echo "Coverity found: ${defectCount} defects"
                    }
                }
            }
        }
	stage('3. Build & Scan Docker Image') {
            steps {
		script {
    echo "[Docker] Building Docker Image for WebGoat..."
    
    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | grep -v "deploy_prod" | head -n 1', returnStdout: true).trim()
    if (!webgoatJar) error "❌ ERROR: No JAR file found for Docker Build!"
    
    // 1. Tạo file Dockerfile cơ bản và Build Image (Removed docker save)
	// 1. Build Image (Vẫn giữ BuildKit mặc định) và LƯU RA FILE TAR
    sh """
        echo "FROM eclipse-temurin:17-jre-alpine" > Dockerfile
        echo "COPY ${webgoatJar} /app/webgoat.jar" >> Dockerfile
        echo "EXPOSE 8080" >> Dockerfile
        echo "ENTRYPOINT [\\"java\\", \\"-jar\\", \\"/app/webgoat.jar\\"]" >> Dockerfile
        
        docker build -t webgoat-docker-demo:latest .
        
        # Bắt buộc phải xuất ra file tar cho công cụ quét mới
        docker save -o webgoat-docker.tar webgoat-docker-demo:latest
        chmod 777 webgoat-docker.tar
    """
    
    // 2. Chạy quét bằng công cụ CONTAINER_SCAN thay vì DOCKER
    withCredentials([string(credentialsId: 'blackduck-api-token', variable: 'BLACKDUCK_API_TOKEN')]) {
        sh """
            ./detect10.sh \\
                --blackduck.url="https://192.168.12.204" \\
                --blackduck.api.token="\$BLACKDUCK_API_TOKEN" \\
                --blackduck.trust.cert=true \\
                --detect.project.name="${SEEKER_PROJECT_KEY}-docker" \\
                --detect.project.version.name="latest" \\
                --detect.container.scan.file.path="webgoat-docker.tar" \\
                --detect.tools=CONTAINER_SCAN
        """
    }
}
            }
        }
        stage('4. Setup Seeker Agent') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            sh "rm -rf seeker installer.sh || true"
                            sh """
                                curl -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        }
                    }
                }
            }
        }

        stage('5. Run App with Seeker (Test)') {
            steps {
                script {
                    echo " [Run] Starting latest WebGoat (Test Mode on Port ${TEST_PORT})..."

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
                            nohup java \\
                                -Dfile.encoding=UTF-8 \\
                                -Duser.timezone=${TZ} \\
                                --add-opens java.base/java.lang=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -jar ${webgoatJar} \\
                                --server.address=0.0.0.0 \\
                                --webgoat.port=${TEST_PORT} \\
                                --webwolf.port=${WOLF_TEST_PORT} \\
                                > app_webgoat_test.log 2>&1 < /dev/null &
                        """
                    }
                }
            }
        }

        stage('6. Health Check & Traffic') {
            steps {
                script {
                    echo "[Check] Waiting for Services (Custom Loop)..."
                    boolean isReady = false
                 
                    // 1. Health Check Loop
                    for (int i = 1; i <= 60; i++) { 
                        def statusGoat = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                        def statusWolf = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                    
                        if ((statusGoat == '200' || statusGoat == '302') && (statusWolf == '200' || statusWolf == '302')) {
                            isReady = true;
                            echo "✅ Services are UP! (Goat: ${statusGoat}, Wolf: ${statusWolf})"
                            break;
                        }
                        echo "Waiting... [Attempt ${i}/60] Goat: ${statusGoat} | Wolf: ${statusWolf}"
                        sleep 10
                    }

                    if (!isReady) error "Timeout: Services did not start."

                    echo "[Traffic] Executing Basic Register & Login Test..."

                    // 2. Traffic Generation: Chỉ test Đăng ký và Đăng nhập
                    sh """
                        # Dọn dẹp file tạm
                        rm -f cookies.txt goat_login.html

                        # ====================================================
                        # A. TEST REGISTER (Đăng ký tài khoản)
                        # ====================================================
                        echo ">>> [WebGoat] Testing Registration..."
                        curl -s -m 5 -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/register.mvc \\
                            -d "username=testuser&password=password123&matchingPassword=password123&agree=agree" \\
                            -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        # ====================================================
                        # B. TEST LOGIN (Đăng nhập)
                        # ====================================================
                        echo ">>> [WebGoat] Getting CSRF token for Login..."
                        curl -s -m 5 -c cookies.txt http://127.0.0.1:${TEST_PORT}/WebGoat/login > goat_login.html
             
                        GOAT_CSRF=\$(grep -oP 'name="_csrf" value="\\K[^"]+' goat_login.html || echo "none")

                        echo ">>> [WebGoat] Testing Login..."
                        curl -s -m 5 -c cookies.txt -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                            -d "username=testuser&password=password123&_csrf=\$GOAT_CSRF" \\
                            -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        echo ">>> Traffic testing completed successfully!"
                    """

                    // 3. Cleanup (Kill process)
                    echo "[Cleanup] Stopping Test Instances..."
                    sh """
                        lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true
                        lsof -t -i:${WOLF_TEST_PORT} | xargs -r kill -9 || true
                    """
                }
            }
        }

        stage('7. Quality Gate') {
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
                                script: 'curl -s -X GET -H "Authorization: $SEEKER_API_TOKEN" -H "accept: */*" "' + apiUrl + '"',
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
	stage('8. Push Data to SRM') {
            steps {
                script {
                    echo "[SRM] Đóng gói source code và đẩy kết quả lên SRM..."
                    
                    // 1. Nén mã nguồn (Rất quan trọng để SRM map lỗi với dòng code)
                    // Bỏ qua các file build/binaries không cần thiết để tối ưu thời gian upload
                    sh "zip -rq source_code.zip . -x '*.git*' '*idir*' '*target*' '*deploy_prod*' '*.jar' '*.tar'"
                    
                    withCredentials([string(credentialsId: 'srm-api-token', variable: 'SRM_API_TOKEN')]) {
                        // 2. Kiểm tra xem có file kết quả Coverity không để đính kèm
                        def covParam = fileExists('coverity_results.json') ? '-F "file=@coverity_results.json"' : ''
                        
                        echo ">>> Gửi dữ liệu tới SRM (Project ID: ${SRM_PROJECT_ID})..."
                        
                        // 3. Gọi API tạo luồng phân tích mới
                        sh """
                            curl -k -s -X POST "${SRM_SERVER_URL}/srm/api/projects/${SRM_PROJECT_ID}/analysis" \\
                                -H "accept: application/json" \\
                                -H "Authorization: Bearer \$SRM_API_TOKEN" \\
                                -F "file=@source_code.zip" \\
                                ${covParam}
                        """
                        echo "✅ Đã đẩy dữ liệu thành công lên SRM!"
                    }
                }
            }
        }
	stage('9. SRM Quality Gate') {
            steps {
                script {
                    echo "[Gate] Đang chờ SRM hoàn tất phân tích và kiểm tra Quality Gate..."
                    
                    // Thời gian chờ tối đa (ví dụ: 5 phút = 300 giây)
                    sleep 30 // Chờ khởi động luồng phân tích
                    
                    int maxCritical = 0 // Không chấp nhận bất kỳ lỗi Critical nào
                    int maxHigh = 10    // Chấp nhận tối đa 10 lỗi High
                    
                    withCredentials([string(credentialsId: 'srm-api-token', variable: 'SRM_API_TOKEN')]) {
                        // 1. Lấy dữ liệu Findings từ SRM (Lọc theo mức độ Critical và High)
                        // Lưu ý: Endpoint này áp dụng cho các bản SRM/Code Dx tiêu chuẩn.
                        def apiUrl = "${SRM_SERVER_URL}/srm/api/projects/${SRM_PROJECT_ID}/findings/count?severities=Critical,High"
                        
                        try {
                            def response = sh(
                                script: """curl -s -k -X GET "${apiUrl}" \\
                                    -H "accept: application/json" \\
                                    -H "Authorization: Bearer \$SRM_API_TOKEN" """,
                                returnStdout: true
                            ).trim()

                            echo "Raw SRM Response: ${response}"
                            
                            // Phân tích JSON trả về (Giả định SRM trả về Map có chứa số lượng theo severity)
                            def jsonResult = new groovy.json.JsonSlurper().parseText(response)
                            
                            int criticalCount = jsonResult.severities?.Critical ?: 0
                            int highCount = jsonResult.severities?.High ?: 0
                            
                            echo "📊 SRM Report Summary (Gộp từ tất cả các tool):"
                            echo "   - Critical: ${criticalCount} / Allowed: ${maxCritical}"
                            echo "   - High:     ${highCount} / Allowed: ${maxHigh}"

                            if (criticalCount > maxCritical || highCount > maxHigh) {
                                error "❌ Quality Gate FAILED: Tổng số lỗi vượt quá ngưỡng cho phép (Critical: ${criticalCount}, High: ${highCount}). Vui lòng kiểm tra SRM UI để xem chi tiết."
                            } else {
                                echo "✅ Quality Gate PASSED. Code đủ an toàn để đi tiếp!"
                            }

                        } catch (Exception e) {
                            echo "⚠️ Lỗi khi gọi API của SRM: ${e.getMessage()}"
                            error "Quality Gate bị gián đoạn do không thể kết nối tới SRM."
                        }
                    }
                }
            }
        }
        stage('10. Deploy to Production') {
            steps {
                script {
                    echo "[Deploy] Deploying latest WebGoat to Production on Port ${PROD_PORT}..."
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
                        mkdir -p ${deployDir}/seeker

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
                                curl -s -X POST http://127.0.0.1:${PROD_PORT}/WebGoat/register.mvc \\
                                    -d "username=webgoatadmin&password=password&matchingPassword=password&agree=agree" \\
                                    -H "Content-Type: application/x-www-form-urlencoded"
                            """
                            prodReady = true
                            break
                        }
                        echo "Waiting... (Goat: ${gStatus}, Wolf: ${wStatus}) [${i}/60]"
                        sleep 15
                    }
                    
                    if (!prodReady) {
                        sh "cat ${deployDir}/app_webgoat_prod.log || echo 'Log file not found!'"
                        error "Deployment Failed: Production server did not start properly."
                    }
                    echo "Deployment Process Finished!"
                } 
            } 
        } 
    } 

    post {
        always {
             archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
        }
    }
}
