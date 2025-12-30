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
                    echo "[Check] Waiting for WebGoat (Test Instance)..."
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
                            echo "WebGoat is UP!"
                            break;
                        }
                        sleep 5
                    }

                    if (!isReady) {
                        sh "cat app_webgoat_test.log"
                        error "Timeout: WebGoat Test Instance did not start on port ${TEST_PORT}."
                    }

                    echo "[Traffic] Generating traffic for Seeker..."
                    
                    sh """
                        rm -f cookies.txt
                        
                        
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
                    
                    echo "Traffic generation completed!"
                    sleep 10 
                    
                    echo "[Cleanup] Stopping Test Instance..."
                    sh "lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true"
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
                    
                    // Find the new JAR
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | grep -v "deploy_prod" | head -n 1', returnStdout: true).trim()

                    if (!webgoatJar) {
                        error "ERROR: No new WebGoat JAR file found in build folders!"
                    }
                    echo "Found JAR: ${webgoatJar}"

                    // 1. Dọn dẹp & Chuẩn bị thư mục
		    sh """
                        echo "Cleaning up old processes on ${PROD_PORT} and ${WOLF_PROD_PORT}..."
                        fuser -k ${PROD_PORT}/tcp || true
                        fuser -k ${WOLF_PROD_PORT}/tcp || true
                        sleep 2
                        
                        cp ${webgoatJar} ${deployDir}/webgoat-app.jar
                        cp -r seeker/* ${deployDir}/seeker/
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            echo "Starting WebGoat & WebWolf (Prod)..."
                            
			mkdir -p ${deployDir}/webwolf-data \
    
    nohup java -Xmx2g \
        -Dfile.encoding=UTF-8 -Duser.timezone=${TZ} \
        -javaagent:${deployDir}/seeker/seeker-agent.jar \
        -Dseeker.server.url=${SEEKER_SERVER_URL} \
        -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
        -jar ${deployDir}/webgoat-app.jar \
        --server.address=0.0.0.0 \
        --webgoat.port=${PROD_PORT} \
        --webwolf.port=${WOLF_PROD_PORT} \
        --webwolf.address=0.0.0.0 \
        --webgoat.server.directory=${deployDir}/webgoat-data \
        --webwolf.server.directory=${deployDir}/webwolf-data \ 
        > ${deployDir}/app_webgoat_prod.log 2>&1 < /dev/null &
"""
                    }
                    // 3. Đợi Server lên và TỰ ĐỘNG TẠO TÀI KHOẢN
                    script {
                        echo "Waiting for WebGoat (Prod) to initialize..."
                        boolean prodReady = false
                        
                        for (int i = 1; i <= 60; i++) {
                            def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PROD_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                            
                            if (status == '200') {
                                echo "Server is UP! Auto-registering admin account..."
                                
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
                             echo "Deployment FAILED. Printing application logs:"
                             sh "cat ${deployDir}/app_webgoat_prod.log || echo 'Log file not found!'"
                             error "Deployment Failed: Production server did not start on port ${PROD_PORT}."
                        }
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
