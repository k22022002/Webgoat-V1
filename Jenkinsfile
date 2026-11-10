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
        PROD_PORT           = "8099"
        WOLF_PROD_PORT      = "9092"
        COMMON_VERSION      = "Build-${env.BUILD_NUMBER}"        
        JENKINS_NODE_COOKIE = "dontKillMe"
        TZ                  = "Asia/Ho_Chi_Minh"
        
        // --- CẤU HÌNH BLACK DUCK ON-PREM ---
        BLACKDUCK_URL       = "https://192.168.12.204"
        
        // --- CẤU HÌNH COVERITY SAST ---
        COV_BIN             = "/home/ubuntu/cov-analysis-linux64-2026.9.0/bin"
        COV_URL             = "http://192.168.12.190:8081"
        COV_STREAM          = "webgoat-stream"
    }
    
    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo "[Build] Compiling latest WebGoat..."
                    sh "chmod +x mvnw"
                    sh "./mvnw clean package -DskipTests -Dmaven.test.skip=true -DskipITs -Dprocess.skip=true"          
                    
                    env.WEBGOAT_JAR = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | grep -v "deploy_prod" | head -n 1', returnStdout: true).trim()
                    if (!env.WEBGOAT_JAR) error "❌ ERROR: No compiled JAR file found!"
                    echo "✅ Found JAR for Pipeline: ${env.WEBGOAT_JAR}"
                }
            }
        }

        stage('2. Black Duck SCA & Binary Analysis (BDBA)') {
            steps {
                script {
                    echo "[SCA & BDBA] Running Open Source and Binary Analysis..."
                    withCredentials([string(credentialsId: 'blackduck-sca-token', variable: 'BLACKDUCK_API_TOKEN')]) {
                        sh """
                            java -jar /home/ubuntu/detect-airgap/detect-12.1.0.jar \\
                                --blackduck.url="${BLACKDUCK_URL}" \\
                                --blackduck.api.token="\$BLACKDUCK_API_TOKEN" \\
                                --blackduck.trust.cert=true \\
                                --detect.project.name="webgoat-project" \\
                                --detect.project.version.name="${COMMON_VERSION}" \\
                                --detect.binary.scan.file.path="${env.WEBGOAT_JAR}" \\
                                --detect.tools=DETECTOR,SIGNATURE_SCAN,BINARY_SCAN \\
                                --detect.air.gap.mode=true \\
                                --detect.air.gap.path=/home/ubuntu/detect-airgap
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
                withCredentials([file(credentialsId: 'coverity-credentials', variable: 'COV_AUTH_KEY')]) {
                    script {
                        echo '--- [Step] Synopsys Coverity SAST ---'
                        sh "rm -rf idir coverity-report coverity_results.json"
                        
                        sh "${COV_BIN}/coverity capture --project-dir . --dir idir"
                        sh "${COV_BIN}/cov-analyze --dir idir --all --webapp-security --strip-path \$(pwd)"

                        sh """
                            ${COV_BIN}/cov-commit-defects --dir idir \\
                                --url ${COV_URL} \\
                                --stream ${COV_STREAM} \\
                                --auth-key-file "\$COV_AUTH_KEY" \\
                                --version "${COMMON_VERSION}" \\
                                --description "WebGoat Build ${env.BUILD_NUMBER}" 
                        """
                        
                        sh "${COV_BIN}/cov-format-errors --dir idir --html-output coverity-report || true"
                        sh "${COV_BIN}/cov-format-errors --dir idir --json-output-v7 coverity_results.json || true"
                        
                        if (fileExists('coverity_results.json')) {
                            def defectCount = sh(script: "jq '.issues | length' coverity_results.json", returnStdout: true).trim().toInteger()
                            echo "Coverity found: ${defectCount} defects"
                        }
                    }
                }
            }
        }
        
        stage('3. Build Docker Image') {
            steps {
                script {
                    echo "[Docker] Building Docker Image..."
                    sh """
                        echo "FROM eclipse-temurin:17-jre-alpine" > Dockerfile
                        echo "COPY ${env.WEBGOAT_JAR} /app/webgoat.jar" >> Dockerfile
                        echo "EXPOSE 8080" >> Dockerfile
                        echo "ENTRYPOINT [\\"java\\", \\"-jar\\", \\"/app/webgoat.jar\\"]" >> Dockerfile
                        
                        docker build -t webgoat-docker-demo:latest .
                    """
                    sh "rm -f Dockerfile"
                }
            }
        }

        /*
        // ==============================================================
        // TẠM ĐÓNG STAGE 4: SETUP SEEKER AGENT
        // ==============================================================
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
                                rm -f installer.sh
                            """
                        }
                    }
                }
            }
        }
        */

        /*
        // ==============================================================
        // TẠM ĐÓNG STAGE 5: RUN APP WITH SEEKER (TEST)
        // ==============================================================
        stage('5. Run App with Seeker (Test)') {
            steps {
                script {
                    echo " [Run] Starting latest WebGoat (Test Mode on Port ${TEST_PORT})..."
                    sh """
                        fuser -k ${TEST_PORT}/tcp || true
                        fuser -k ${WOLF_TEST_PORT}/tcp || true
                        sleep 3
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            nohup java \\
                                -Dfile.encoding=UTF-8 \\
                                -Duser.timezone=${TZ} \\
                                --add-opens java.base/java.lang=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -Dseeker.project.version=${COMMON_VERSION} \\
                                -Dseeker.agent.accessToken=${SEEKER_ACCESS_TOKEN} \\
                                -jar ${env.WEBGOAT_JAR} \\
                                --server.address=0.0.0.0 \\
                                --webgoat.port=${TEST_PORT} \\
                                --webwolf.port=${WOLF_TEST_PORT} \\
                                > app_webgoat_test.log 2>&1 < /dev/null &
                        """
                    }
                }
            }
        }
        */

        /*
        // ==============================================================
        // TẠM ĐÓNG STAGE 6: HEALTH CHECK & TRAFFIC
        // ==============================================================
        stage('6. Health Check & Traffic') {
            steps {
                script {
                    echo "[Check] Waiting for Services..."
                    boolean isReady = false
                 
                    for (int i = 1; i <= 60; i++) { 
                        def statusGoat = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                        def statusWolf = sh(script: "curl -s -L -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_TEST_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                    
                        if ((statusGoat == '200' || statusGoat == '302') && (statusWolf == '200' || statusWolf == '302')) {
                            isReady = true;
                            echo "✅ Services are UP!"
                            break;
                        }
                        sleep 10
                    }

                    if (!isReady) error "Timeout: Services did not start."
                    echo "[Traffic] Executing Basic Register & Login Test..."
                    sh """
                        # Test Register
                        curl -s -m 5 -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/register.mvc \\
                            -d "username=testuser&password=password123&matchingPassword=password123&agree=agree" \\
                            -H "Content-Type: application/x-www-form-urlencoded" > /dev/null

                        # Test Login
                        curl -s -m 5 -c cookies.txt http://127.0.0.1:${TEST_PORT}/WebGoat/login > goat_login.html
                        GOAT_CSRF=\$(grep -oP 'name="_csrf" value="\\K[^"]+' goat_login.html || echo "none")

                        curl -s -m 5 -c cookies.txt -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                            -d "username=testuser&password=password123&_csrf=\$GOAT_CSRF" \\
                            -H "Content-Type: application/x-www-form-urlencoded" > /dev/null
                    """

                    echo "[Cleanup] Stopping Test Instances & Removing temp files..."
                    sh """
                        fuser -k ${TEST_PORT}/tcp || true
                        fuser -k ${WOLF_TEST_PORT}/tcp || true
                        rm -f cookies.txt goat_login.html
                    """
                }
            }
        }
        */

        /*
        // ==============================================================
        // TẠM ĐÓNG STAGE 7: QUALITY GATE
        // ==============================================================
        stage('7. Quality Gate') {
            steps {
                script {
                    echo "[Gate] Checking Seeker Compliance..."
                    sleep 10 

                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        def apiUrl = "${SEEKER_SERVER_URL}/rest/api/latest/vulnerabilities?format=JSON&projectKeys=${SEEKER_PROJECT_KEY}&status=DETECTED&minSeverity=HIGH"
                        
                        try {
                            def response = sh(
                                script: 'curl -s -X GET -H "Authorization: \$SEEKER_API_TOKEN" -H "accept: *//*" "' + apiUrl + '"',
                                returnStdout: true
                            ).trim()

                            def jsonResult = new groovy.json.JsonSlurper().parseText(response)
                            def vulnList = (jsonResult instanceof List) ? jsonResult : 
                                           (jsonResult.content ?: jsonResult.vulnerabilities ?: [])
                            
                            int criticalCount = 0, highCount = 0
                            def failReasons = []

                            vulnList.each { vuln ->
                                String sev = vuln.Severity?.toString()?.trim()?.toUpperCase() ?: "UNKNOWN"
                                if (sev == 'CRITICAL') { criticalCount++; failReasons.add("[CRITICAL] \${vuln.VulnerabilityName}") }
                                if (sev == 'HIGH') { highCount++; failReasons.add("[HIGH] \${vuln.VulnerabilityName}") }
                            }

                            echo "Seeker Gate: Critical: \${criticalCount}/100, High: \${highCount}/100"

                            if (criticalCount > 100 || highCount > 100) {
                                failReasons.each { echo "   - \${it}" }
                                error "Quality Gate FAILED: Found \${criticalCount} Critical & \${highCount} High vulnerabilities."
                            } else {
                                echo "Quality Gate PASSED."
                            }
                        } catch (Exception e) {
                            error "Failed to verify Quality Gate: \${e.getMessage()}"
                        }
                    }
                }
            }
        }
        */
        
        stage('8. Deploy to Production') {
            steps {
                script {
                    echo "[Deploy] Deploying latest WebGoat to Production on Port ${PROD_PORT}..."
                    def deployDir = "${WORKSPACE}/deploy_prod" 

                    sh """
                        fuser -k ${PROD_PORT}/tcp || true
                        fuser -k ${WOLF_PROD_PORT}/tcp || true
                        sleep 2
           
                        mkdir -p ${deployDir}/webgoat-data
                        mkdir -p ${deployDir}/webwolf-data

                        cp ${env.WEBGOAT_JAR} ${deployDir}/webgoat-app.jar
                        chmod -R 777 ${deployDir}/webgoat-data ${deployDir}/webwolf-data
                    """

                    // Khởi chạy WebGoat độc lập mà không cần Seeker agent
                    sh """
                        export JENKINS_NODE_COOKIE=dontKillMe

                        nohup java -Xmx2g \\
                            -Dfile.encoding=UTF-8 \\
                            -Duser.timezone=${TZ} \\
                            -jar ${deployDir}/webgoat-app.jar \\
                            --server.address=0.0.0.0 \\
                            --webgoat.port=${PROD_PORT} \\
                            --webwolf.port=${WOLF_PROD_PORT} \\
                            --webwolf.address=0.0.0.0 \\
                            --webgoat.server.directory=${deployDir}/webgoat-data \\
                            --webwolf.server.directory=${deployDir}/webwolf-data \\
                            > ${deployDir}/app_webgoat_prod.log 2>&1 < /dev/null &
                    """

                    echo "Waiting for Production WebGoat & WebWolf to initialize..."
                    boolean prodReady = false
 
                    for (int i = 1; i <= 60; i++) {
                        def gStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PROD_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                        def wStatus = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${WOLF_PROD_PORT}/WebWolf/login || echo '000'", returnStdout: true).trim()
                  
                        if (gStatus == '200' && (wStatus == '200' || wStatus == '302')) {
                            echo "✅ Both servers are UP!"
                            sh """
                                curl -s -X POST http://127.0.0.1:${PROD_PORT}/WebGoat/register.mvc \\
                                    -d "username=webgoatadmin&password=password&matchingPassword=password&agree=agree" \\
                                    -H "Content-Type: application/x-www-form-urlencoded" > /dev/null
                            """
                            prodReady = true
                            break
                        }
                        sleep 15
                    }
                    
                    if (!prodReady) {
                        sh "cat ${deployDir}/app_webgoat_prod.log || echo 'Log file not found!'"
                        error "Deployment Failed: Production server did not start properly."
                    }
                } 
            } 
        } 

        /*
        // ==============================================================
        // TẠM ĐÓNG STAGE 9: TRIGGER SRM TOOL CONNECTORS & VERSIONING
        // ==============================================================
        stage('9. Trigger SRM Tool Connectors & Versioning') {
            steps {
                script {
                    echo "[SRM] Đang thực hiện quy trình kích hoạt và gắn nhãn phiên bản chuẩn..."
                    
                    withCredentials([string(credentialsId: 'srm-api-token', variable: 'SRM_API_TOKEN')]) {
                        sh """
                            RESPONSE=\$(curl -k -X POST "${SRM_SERVER_URL}/api/analysis-prep" \\
                                 -H "API-Key: \$SRM_API_TOKEN" \\
                                 -H "Content-Type: application/json" \\
                                 -d '{"projectId": ${SRM_PROJECT_ID}}')
                            
                            PREP_ID=\$(echo "\$RESPONSE" | grep -o '"prepId":"[^"]*"' | cut -d':' -f2 | tr -d '"')
                            echo "✅ Có mã Prep ID: \$PREP_ID"
                            
                            RUN_RESPONSE=\$(curl -k -X POST "${SRM_SERVER_URL}/api/analysis-prep/\$PREP_ID/analyze" \\
                                 -H "API-Key: \$SRM_API_TOKEN" \\
                                 -H "Content-Type: application/json")
                            
                            ANALYSIS_ID=\$(echo "\$RUN_RESPONSE" | grep -o '"analysisId":[0-9]*' | cut -d':' -f2)
                            echo "✅ Đã tạo lượt phân tích số: \$ANALYSIS_ID"
                            
                            curl -k -X PUT "${SRM_SERVER_URL}/api/analyses/\$ANALYSIS_ID/metadata" \\
                                 -H "API-Key: \$SRM_API_TOKEN" \\
                                 -H "Content-Type: application/json" \\
                                 -d '{"versionName": "${COMMON_VERSION}", "tags": ["${COMMON_VERSION}"]}'
                            
                            echo "🚀 Đã gắn thành công nhãn ${COMMON_VERSION} vào hệ thống!"
                        """
                    }
                }
            }
        }
        */
    } 

    post {
        always {
             archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
        }
    }
}