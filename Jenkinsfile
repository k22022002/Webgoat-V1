pipeline {
    agent any

    tools {
        maven 'Maven3.9.11'
        jdk 'JDK 17'
    }

    environment {
        // --- APP CONFIG ---
        APP_PORT = "8090"
        
        // --- SEEKER CONFIG ---
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
        
        // Prevent Jenkins from killing background processes (nohup)
        JENKINS_NODE_COOKIE = "dontKillMe" 
    }

    stages {
        // =========================================
        // STAGE 1: BUILD
        // =========================================
        stage('1. Build Application') {
            steps {
                script {
                    echo "🚀 [Build] Compiling WebGoat v2023.8..."
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        // =========================================
        // STAGE 2: PREPARE SEEKER AGENT
        // =========================================
        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo "🕵️ [Agent] Checking Synopsys Seeker Agent..."
                    
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            echo ">>> Downloading Seeker Agent..."
                            // Xóa sạch thư mục cũ để tránh lỗi conflict
                            sh "rm -rf seeker installer.sh || true"
                            
                            // Tải và cài đặt Agent
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        } else {
                            echo "✅ Agent đã tồn tại. Sẽ sử dụng cấu hình Dynamic trong lệnh Java."
                        }
                    }
                }
            }
        }

        // =========================================
        // STAGE 3: RUN APPLICATION
        // =========================================
        stage('3. Run App with Seeker') {
            steps {
                script {
                    echo "🚀 [Run] Starting WebGoat + Seeker..."

                    // 1. Tìm file JAR
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    if (!webgoatJar) error "❌ ERROR: Không tìm thấy file .jar!"
                    echo ">>> Found JAR: ${webgoatJar}"

                    // 2. Cleanup Port cũ
                    sh """
                        pkill -f webgoat || true
                        lsof -t -i:${APP_PORT} | xargs -r kill -9 || true
                        sleep 5
                    """

                    // 3. Khởi động với Seeker Agent
                    // NOTE: Truyền trực tiếp config vào lệnh Java (-D) thay vì sửa file properties
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // Export Token vào Env cho process Java
                        // Start lệnh chạy background
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            nohup java \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                --add-opens java.base/java.io=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -Dseeker.agent.auto.update=false \\
                                -Dserver.port=${APP_PORT} \\
                                -Dserver.address=0.0.0.0 \\
                                -Dserver.servlet.context-path=/WebGoat \\
                                -jar ${webgoatJar} \\
                                > app_webgoat.log 2>&1 &
                        """
                    }
                    echo "✅ Lệnh khởi động đã được thực thi."
                }
            }
        }

        // =========================================
        // STAGE 4: HEALTH CHECK & TRAFFIC
        // =========================================
        stage('4. Deep Health Check') {
            steps {
                script {
                    echo "💓 [Check] Waiting for WebGoat..."
                    boolean isReady = false
                    int maxRetries = 90

                    // Loop check status
                    for (int i = 1; i <= maxRetries; i++) {
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> [Wait ${i*10}s] HTTP Status: ${status}"
                        
                        if (status == '200' || status == '302') {
                            echo "✅ SERVER ONLINE!"
                            isReady = true
                            break
                        }
                        sleep 10
                    }

                    if (!isReady) {
                        sh "tail -n 50 app_webgoat.log || true"
                        error "❌ Timeout: WebGoat không phản hồi sau 15 phút."
                    }

                    // Generate Traffic to activate Seeker
                    echo "🚦 [Traffic] Sending login request to activate Seeker..."
                    sh """
                        curl -s -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                        -d "username=admin&password=password" \\
                        -H "Content-Type: application/x-www-form-urlencoded"
                    """
                    sleep 15 // Chờ Agent đồng bộ dữ liệu lên Server
                }
            }
        }

        // =========================================
        // STAGE 5: QUALITY GATE
        // =========================================
        stage('5. Quality Gate') {
            steps {
                script {
                    echo "🛡️ [Gate] Checking Seeker Compliance..."
                    
                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        def checkScript = """
                            rm -f response_body.json status_code.txt
                            
                            # Call API
                            curl -s -k -o response_body.json -w "%{http_code}" \\
                                -X GET "${SEEKER_SERVER_URL}/rest/api/latest/projects/${SEEKER_PROJECT_KEY}/compliance-status" \\
                                -H "Authorization: ${SEEKER_API_TOKEN}" \\
                                -H "Accept: application/json" > status_code.txt

                            HTTP_CODE=\$(cat status_code.txt)
                            BODY=\$(cat response_body.json)

                            echo ">>> HTTP Code: \$HTTP_CODE"

                            if [ "\$HTTP_CODE" = "200" ]; then
                                echo "✅ SUCCESS! Compliance Result:"
                                echo "\$BODY"
                                # TODO: Add grep/jq logic here to fail build if needed
                            elif [ "\$HTTP_CODE" = "404" ]; then
                                echo "❌ ERROR (404): Project Key not found: ${SEEKER_PROJECT_KEY}"
                                echo "--- List of available projects ---"
                                curl -s -k -X GET "${SEEKER_SERVER_URL}/rest/api/latest/projects" \\
                                     -H "Authorization: ${SEEKER_API_TOKEN}" \\
                                     -H "Accept: application/json" | head -c 500
                                exit 1
                            else
                                echo "❌ UNEXPECTED ERROR: \$BODY"
                                exit 1
                            fi
                        """
                        sh checkScript
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Build Failed! Check logs for details."
        }
        always {
            // Archive logs for debugging
            archiveArtifacts artifacts: 'app_webgoat.log', allowEmptyArchive: true
        }
    }
}
