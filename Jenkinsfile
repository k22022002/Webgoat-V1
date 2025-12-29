pipeline {
    agent any

    tools {
        // Cập nhật JDK lên phiên bản 25 theo yêu cầu trong README 
        // Lưu ý: Cần đảm bảo Jenkins Global Tool Configuration đã cấu hình 'JDK 25'
        jdk 'JDK 25' 
        // Vẫn giữ Maven tool phòng trường hợp script cần, nhưng sẽ ưu tiên dùng ./mvnw
        maven 'Maven3.9.11' 
    }

    environment {
        TEST_PORT = "9595" 
        PROD_PORT = "8090"
        
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2025-demo"
        JENKINS_NODE_COOKIE = "dontKillMe"
        
        // Thêm cấu hình Timezone theo khuyến nghị của README (ví dụ: Asia/Ho_Chi_Minh) 
        TZ = "Asia/Ho_Chi_Minh"
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo "🚀 [Build] Compiling WebGoat v2025.3..."
                    // SỬA ĐỔI: Dùng ./mvnw clean install theo hướng dẫn 'Run from sources' 
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
                    echo "🚀 [Run] Starting WebGoat 2025 (Test Mode on Port ${TEST_PORT})..."

                    // Tìm file JAR (WebGoat 2023.8+ cấu trúc tên có thể thay đổi, lệnh này vẫn giữ nguyên để tìm file sinh ra)
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | head -n 1', returnStdout: true).trim()
                    if (!webgoatJar) error "❌ ERROR: No JAR file found!"
                    
                    sh """
                        echo ">>> Cleaning up port ${TEST_PORT}..."
                        fuser -k ${TEST_PORT}/tcp || true
                        lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true
                        sleep 5
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            # SỬA ĐỔI: Chỉnh sửa tham số chạy theo mục "3.1 Running on a different port" trong README 
                            # - Loại bỏ --server.port (WebGoat tự map từ webgoat.port)
                            # - Giữ --server.address=0.0.0.0 để bind IP (Mục 4 README)
                            # - Thêm Timezone
                            
                            nohup java \\
                                -Dfile.encoding=UTF-8 \\
                                -Duser.timezone=${TZ} \\
                                --add-opens java.base/java.lang=ALL-UNNAMED \\
                                --add-opens java.base/java.util=ALL-UNNAMED \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -jar ${webgoatJar} \\
                                --server.address=0.0.0.0 \\
                                --webgoat.port=${TEST_PORT} \\
                                --webwolf.port=9096 \\
                                > app_webgoat_test.log 2>&1 < /dev/null &
                        """
                    }
                }
            }
        }

	stage('4. Health Check & Traffic') {
            steps {
                script {
                    echo "💓 [Check] Waiting for WebGoat (Test Instance)..."
                    boolean isReady = false
                
                    // Check health loop
                    for (int i = 1; i <= 60; i++) { 
                        def status = sh(
                            script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()
                    
                        // Chấp nhận 200 hoặc 401 (App đã lên)
                        if (status == '200' || status == '401') {
                            isReady = true;
                            echo "✅ WebGoat is UP!"
                            break;
                        }
                        sleep 5
                    }

                    if (!isReady) {
                        sh "cat app_webgoat_test.log"
                        error "❌ Timeout: WebGoat Test Instance did not start on port ${TEST_PORT}."
                    }

                    echo "🚦 [Traffic] Generating traffic for Seeker..."
                    
                    sh """
                        rm -f cookies.txt
                        
                        # --- FIX: Đổi user thành 'webgoat_admin' (đủ 6 ký tự) ---
                        
                        # 1. ĐĂNG KÝ
                        echo "--- Registering Account ---"
                        curl -s -k -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/register.mvc \\
                             -d "username=webgoatadmin&password=password&matchingPassword=password&agree=agree" \\
                             -H "Content-Type: application/x-www-form-urlencoded"

                        # 2. LOGIN
                        echo "--- Logging in ---"
                        curl -s -k -c cookies.txt  -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                             -d "username=webgoatadmin&password=password" \\
                             -H "Content-Type: application/x-www-form-urlencoded"

                        # 3. TẠO TRAFFIC
                        echo "--- Accessing Welcome Page ---"
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${TEST_PORT}/WebGoat/welcome.mvc
                        
                        echo "--- Triggering SQL Injection Lesson ---"
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${TEST_PORT}/WebGoat/SqlInjection/attack5a
                    """
                    
                    echo "✅ Traffic generation completed!"
                    sleep 10 
                    
                    echo "🛑 [Cleanup] Stopping Test Instance..."
                    sh "lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true"
                }
            }
        }
	stage('5. Quality Gate') {
            steps {
                script {
                    echo "🛡️ [Gate] Checking Seeker Compliance..."
                    sleep 30 
                    
                    int maxCritical = 0
                    int maxHigh = 0

                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        // Lưu ý: IP giữ nguyên, chỉ thay port và protocol
                        // Thêm tham số 'format=JSON' để ép kiểu dữ liệu trả về
                        def apiUrl = "http://192.168.12.190:8082/rest/api/latest/vulnerabilities?format=JSON&language=en&projectKeys=${SEEKER_PROJECT_KEY}&status=DETECTED&minSeverity=HIGH"
                        
                        echo "🔍 Querying Seeker API: ${apiUrl}"

                        try {
                            // --- FIX 2: Header 'accept: */*' và dùng curl -k (bỏ qua SSL self-signed) ---
                            def response = sh(
                                script: 'curl -s -k -X GET -H "Authorization: Bearer $SEEKER_API_TOKEN" -H "accept: */*" "' + apiUrl + '"',
                                returnStdout: true
                            ).trim()

                            // --- DEBUG: In ra một phần response để kiểm tra ---
                            echo "🔍 Raw Response (First 200 chars): ${response.take(200)}..."

                            if (!response || response == "") {
                                error "❌ API Response is empty!"
                            }

                            def jsonResult = new groovy.json.JsonSlurper().parseText(response)
                            
                            // Kiểm tra lỗi API (nếu có)
                            if (jsonResult instanceof Map && jsonResult.containsKey('code') && jsonResult.code != 200) {
                                error "❌ API Error: ${jsonResult.message} (HTTP ${jsonResult.code})"
                            }

                            // Xử lý dữ liệu (Seeker trả về list hoặc map tùy version)
                            def vulnList = []
                            if (jsonResult instanceof List) {
                                vulnList = jsonResult
                            } else if (jsonResult instanceof Map) {
                                if (jsonResult.containsKey('content')) vulnList = jsonResult.content
                                else if (jsonResult.containsKey('vulnerabilities')) vulnList = jsonResult.vulnerabilities
                                else echo "⚠️ Unknown JSON structure, attempting to verify keys..."
                            }

                            echo "📊 Found ${vulnList.size()} vulnerabilities."

                            // Đếm lỗi
                            int criticalCount = 0
                            int highCount = 0

                            vulnList.each { vuln ->
                                def sev = vuln.severity ? vuln.severity.toString().toUpperCase() : "UNKNOWN"
                                if (sev == 'CRITICAL') criticalCount++
                                if (sev == 'HIGH') highCount++
                            }

                            echo "📊 Seeker Report Summary:"
                            echo "   - Critical: ${criticalCount} (Max allowed: ${maxCritical})"
                            echo "   - High:     ${highCount} (Max allowed: ${maxHigh})"

                            if (criticalCount > maxCritical || highCount > maxHigh) {
                                def criticalNames = vulnList.findAll { it.severity == 'CRITICAL' }.collect { it.vulnerabilityName }
                                if (criticalNames) echo "   [!] Critical Issues: ${criticalNames}"
                                error "❌ Quality Gate FAILED: Security standards not met."
                            } else {
                                echo "✅ Quality Gate PASSED."
                            }

                        } catch (Exception e) {
                            echo "⚠️ Script Error: ${e.getMessage()}"
                            error "❌ Failed to verify Quality Gate."
                        }
                    }
                }
            }
        }
	stage('6. Deploy to Production') {
            steps {
                script {
                    echo "🚀 [Deploy] Deploying v2025.3 to Production on Port ${PROD_PORT}..."
                    def deployDir = "${WORKSPACE}/deploy_prod" 
                    
                    // Find the new JAR
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | grep -v "deploy_prod" | head -n 1', returnStdout: true).trim()

                    if (!webgoatJar) {
                        error "❌ ERROR: No new WebGoat JAR file found in build folders!"
                    }
                    echo "📦 Found JAR: ${webgoatJar}"

                    // 1. Dọn dẹp & Chuẩn bị thư mục
                    sh """
                        echo "🧹 Cleaning up old processes..."
                        lsof -t -i:${PROD_PORT} | xargs -r kill -9 || true
                        lsof -t -i:9092 | xargs -r kill -9 || true
                        sleep 2
                        
                        echo "📂 Creating directories at ${deployDir}..."
                        rm -rf ${deployDir}
                        mkdir -p ${deployDir}/seeker
                        mkdir -p ${deployDir}/webgoat-data
                        
                        echo "📋 Copying files..."
                        cp ${webgoatJar} ${deployDir}/webgoat-app.jar
                        cp -r seeker/* ${deployDir}/seeker/
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // 2. Khởi chạy ứng dụng
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            echo "🚀 Starting WebGoat (Prod)..."
                            
                            # --- FIX: Removed -Duser.home to fix FileNotFoundException ---
                            nohup java \
                                -Dfile.encoding=UTF-8 \
                                -Duser.timezone=${TZ} \
                                -Xmx2g \
                                -javaagent:${deployDir}/seeker/seeker-agent.jar \
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
                                -jar ${deployDir}/webgoat-app.jar \
                                --server.address=0.0.0.0 \
                                --webgoat.port=${PROD_PORT} \
                                --webwolf.port=9092 \
                                --webgoat.server.directory=${deployDir}/webgoat-data \
                                > ${deployDir}/app_webgoat_prod.log 2>&1 < /dev/null &
                        """
                    }

                    // 3. Đợi Server lên và TỰ ĐỘNG TẠO TÀI KHOẢN
                    script {
                        echo "⏳ Waiting for WebGoat (Prod) to initialize..."
                        boolean prodReady = false
                        
                        for (int i = 1; i <= 60; i++) {
                            def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PROD_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                            
                            if (status == '200') {
                                echo "✅ Server is UP! Auto-registering admin account..."
                                
                                sh """
                                    curl -s -k -X POST http://127.0.0.1:${PROD_PORT}/WebGoat/register.mvc \\
                                        -d "username=webgoatadmin&password=password&matchingPassword=password&agree=agree" \\
                                        -H "Content-Type: application/x-www-form-urlencoded" \\
                                        -H "Accept: text/html"
                                """
                                echo "🎉 Account created: User='webgoatadmin', Pass='password'"
                                prodReady = true
                                break
                            }
                            echo "Waiting... (${i}/60)"
                            sleep 5
                        }
                        
                        if (!prodReady) {
                             echo "🔴 Deployment FAILED. Printing application logs:"
                             sh "cat ${deployDir}/app_webgoat_prod.log || echo 'Log file not found!'"
                             error "❌ Deployment Failed: Production server did not start on port ${PROD_PORT}."
                        }
                    }
                    echo "✅ Deployment Process Finished!"
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
