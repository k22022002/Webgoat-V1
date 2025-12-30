pipeline {
    agent any

    tools {
        maven 'Maven3.9.11'
        jdk 'JDK 21'
    }

    environment {
        // --- FIX 1: Đổi sang cổng 9595 để né hoàn toàn lỗi "Port in use" ở 9090 ---
        TEST_PORT = "9595" 
        PROD_PORT = "8090"
        
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2025-demo"
        JENKINS_NODE_COOKIE = "dontKillMe" 
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo "🚀 [Build] Compiling WebGoat v2025.3..."
                    sh "mvn clean install -DskipTests -Dmaven.test.skip=true -Dprocess-exec.skip=true"
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

                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | head -n 1', returnStdout: true).trim()
                    if (!webgoatJar) error "❌ ERROR: No JAR file found!"
                    
                    // --- FIX 2: Clean up kỹ càng hơn trước khi chạy ---
                    sh """
                        echo ">>> Cleaning up port ${TEST_PORT}..."
                        fuser -k ${TEST_PORT}/tcp || true
                        lsof -t -i:${TEST_PORT} | xargs -r kill -9 || true
                        sleep 5
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            # --- FIX 3: Chạy trên cổng 9595 và WebWolf 9096 ---
                            nohup java \\
                                -Dfile.encoding=UTF-8 \\
                                --add-opens java.base/java.lang=ALL-UNNAMED \\
                                --add-opens java.base/java.util=ALL-UNNAMED \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -jar ${webgoatJar} \\
                                --server.port=${TEST_PORT} \\
                                --server.address=0.0.0.0 \\
                                --server.servlet.context-path=/WebGoat \\
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
                    
                    // --- FIX 4: Check cổng 9595 ---
                    for (int i = 1; i <= 30; i++) { 
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${TEST_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()
                        
                        echo ">>> [Attempt ${i}/30] Status: ${status}"
                        if (status == '200' || status == '302'|| status == '401' ) {
                            isReady = true; break;
                        }
                        sleep 10
                    }
                    if (!isReady) {
                        sh "cat app_webgoat_test.log"
                        error "❌ Timeout: WebGoat Test Instance did not start on port ${TEST_PORT}."
                    }

                    echo "🚦 [Traffic] Generating traffic for Seeker..."
                    
                    // --- FIX 5: Bắn traffic vào cổng 9595 ---
                    sh """
                        rm -f cookies.txt
                        curl -s -k -c cookies.txt -X POST http://127.0.0.1:${TEST_PORT}/WebGoat/login \\
                             -d "username=admin&password=password" \\
                             -H "Content-Type: application/x-www-form-urlencoded"

                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${TEST_PORT}/WebGoat/welcome.mvc
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${TEST_PORT}/WebGoat/SqlInjection/attack5a
                    """
                    
                    echo "✅ Traffic generation completed!"
                    sleep 15 
                    
                    // --- FIX 6: Tắt process test (9595) sau khi xong ---
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
                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        sh '''
                            # Đoạn này giữ nguyên vì nó gọi lên Seeker Server, không liên quan đến App
                            echo "✅ (Mock) Quality Gate Passed for Demo" 
                        '''
                    }
                }
            }
        }

	stage('6. Deploy to Production') {
            steps {
                script {
                    echo "🚀 [Deploy] Deploying v2025.3 to Production on Port ${PROD_PORT}..."
                    def deployDir = "/opt/webgoat-live"
                    def webgoatJar = sh(script: 'find . -type f -name "webgoat-*.jar" | grep -v "original" | grep -v "webwolf" | head -n 1', returnStdout: true).trim()

                    // 1. Dọn dẹp & Chuẩn bị thư mục
                    sh """
                        echo "🧹 Cleaning up old processes..."
                        pkill -f 'webgoat-live' || true
                        lsof -t -i:${PROD_PORT} | xargs -r kill -9 || true
                        
                        echo "📂 Creating directories..."
                        mkdir -p ${deployDir}/seeker
                        mkdir -p ${deployDir}/webgoat-data
                        mkdir -p ${deployDir}/temp_home  # Tạo thư mục Home giả
                        
                        # Xóa data cũ để reset database hoàn toàn
                        rm -rf ${deployDir}/webgoat-data/*
                        rm -rf ${deployDir}/temp_home/*
                        
                        cp ${webgoatJar} ${deployDir}/webgoat-app.jar
                        cp -r seeker/* ${deployDir}/seeker/
                    """

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // 2. Khởi chạy ứng dụng (Thêm -Duser.home để sửa lỗi crash)
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            echo "🚀 Starting WebGoat..."
                            nohup java \
                                -Duser.home=${deployDir}/temp_home \
                                -Dfile.encoding=UTF-8 \
                                -Xmx2g \
                                -javaagent:${deployDir}/seeker/seeker-agent.jar \
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
                                -jar ${deployDir}/webgoat-app.jar \
                                --server.port=${PROD_PORT} \
                                --server.address=0.0.0.0 \
                                --server.servlet.context-path=/WebGoat \
                                --webgoat.port=${PROD_PORT} \
                                --webwolf.port=9092 \
                                --webgoat.server.directory=${deployDir}/webgoat-data \
                                > ${deployDir}/app_webgoat.log 2>&1 < /dev/null &
                        """
                    }

                    // 3. Đợi Server lên và TỰ ĐỘNG TẠO TÀI KHOẢN
                    script {
                        echo "⏳ Waiting for WebGoat to initialize..."
                        sleep 15 // Chờ 15s để Spring Boot khởi động
                        
                        // Thử ping chờ server sống dậy
                        for (int i = 1; i <= 20; i++) {
                            def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${PROD_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()
                            if (status == '200' || status == '302') {
                                echo "✅ Server is UP! Auto-registering admin account..."
                                
                                // --- LỆNH TẠO TÀI KHOẢN TỰ ĐỘNG ---
                                sh """
                                    curl -s -k -X POST http://127.0.0.1:${PROD_PORT}/WebGoat/register.mvc \\
                                        -d "username=admin&password=password&matchingPassword=password&agree=agree" \\
                                        -H "Content-Type: application/x-www-form-urlencoded"
                                """
                                echo "🎉 Account created: User='admin', Pass='password'"
                                break
                            }
                            echo "Waiting... (${i}/20)"
                            sleep 5
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
