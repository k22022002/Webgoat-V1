pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8090"
        SEEKER_SERVER_URL = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
        // Phiên bản 2023.8 build ra file tên khác một chút, ta dùng * để bắt
        JAR_PATTERN = "webgoat-server/target/webgoat-server*.jar" 
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo '--- [Build] Compiling WebGoat v2023.8 ---'
                    // Bỏ qua test để build cho nhanh
                    sh "mvn clean install -DskipTests"
                }
            }
        }
	stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo '--- [Agent] Downloading Synopsys Seeker Agent ---'
                    sh "rm -rf seeker installer.sh || true"

                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // FIX: Thêm tham số &projectKey=$SEEKER_PROJECT_KEY vào URL
                        sh '''
                            curl -k -SL -o installer.sh "http://192.168.12.190:8082/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=$SEEKER_ACCESS_TOKEN&projectKey=$SEEKER_PROJECT_KEY"
                            
                            chmod +x installer.sh
                            sh installer.sh
                        '''
                        
                        // Kiểm tra lại
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            echo "--- CONTENT OF INSTALLER.SH (ERROR LOG) ---"
                            sh "cat installer.sh || true"
                            error "Lỗi: Vẫn chưa tải được Agent. Xem nội dung file ở trên."
                        }
                    }
                }
            }
        }
	stage('3. Run App with IAST') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | head -n 1', returnStdout: true).trim()
                    if (webgoatJar == "") { error "Không tìm thấy file .jar!" }

                    sh "pkill -f webgoat || true"

                    // THÊM: Cấp thêm RAM cho Java (WebGoat + IAST khá nặng)
                    // -Xmx2g: Cho phép dùng tối đa 2GB RAM
                    String startCmd = """
                        nohup java \
                        -Xmx2g \
                        -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \
                        -Dseeker.server.url=${SEEKER_SERVER_URL} \
                        -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
                        -Dserver.port=${APP_PORT} \
                        -Dserver.address=0.0.0.0 \
                        -jar ${webgoatJar} \
                        > app_webgoat.log 2>&1 &
                    """
                    sh startCmd
                    
                    echo ">>> Waiting 60s for WebGoat to start..."
                    sleep 60 
                }
            }
        }

	stage('4. Smart Health Check & Test') {
            steps {
                script {
                    echo "--- [Check] Đang chờ WebGoat mở cổng ${APP_PORT} ---"
                    
                    // Thời gian chờ tối đa: 5 phút (30 lần x 10 giây)
                    // Vì quá trình "re-transformation" trong log của bạn rất lâu, nên cần chờ lâu hơn
                    def maxRetries = 30
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // Thử kết nối tới cổng 8090 (chỉ lấy HTTP Header)
                        // Nếu kết nối được (200 hoặc 302) nghĩa là WebGoat đã sống dậy sau khi Seeker cài xong
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${APP_PORT}/WebGoat/login || echo 'FAIL'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> Lần thử ${i}/${maxRetries}: Trạng thái = ${status}"

                        if (status == '200' || status == '302') {
                            echo "✅ KẾT NỐI THÀNH CÔNG! WebGoat đã khởi động xong."
                            isReady = true
                            break
                        }

                        // Nếu chưa được thì đợi 10s rồi thử lại
                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ QUÁ THỜI GIAN CHỜ (TIMEOUT)!"
                        echo "--- In 100 dòng log cuối cùng để kiểm tra lỗi ---"
                        sh "tail -n 100 app_webgoat.log" 
                        error "WebGoat không khởi động được sau 5 phút."
                    }
                    
                    // --- BẮT ĐẦU TEST TRAFFIC ---
                    // Chỉ chạy khi WebGoat đã sẵn sàng
                    echo "--- [Traffic] Gửi Traffic giả lập để Seeker bắt lỗi ---"
                    
                    // Gửi request Login
                    sh "curl -L -v http://localhost:${APP_PORT}/WebGoat/login"
                    
                    // Gửi request Registration (Trang này hay có lỗi XSS)
                    sh "curl -L -v http://localhost:${APP_PORT}/WebGoat/registration"
                    
                    // Gửi thêm một vài request giả lập khác nếu cần
                }
            }
        }
        stage('5. Quality Gate') {
            steps {
                script {
                     echo '--- [Gate] Checking Seeker Compliance ---'
                     sleep 10
                     withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        def status = sh(
                            script: """
                                curl -s -k -X GET "\${SEEKER_SERVER_URL}/rest/api/latest/projects/\${SEEKER_PROJECT_KEY}/compliance-status" \
                                -H "Authorization: Bearer \${SEEKER_ACCESS_TOKEN}" \
                                -H "Accept: application/json"
                            """, 
                            returnStdout: true
                        ).trim()
                        
                        echo ">>> Compliance Status: \${status}"
                        if (status.contains("NON_COMPLIANT")) {
                            echo "⚠️ CẢNH BÁO: Phát hiện lỗ hổng bảo mật!"
                        }
                     }
                }
            }
        }
    }

    post {
        always {
            sh "pkill -f webgoat || true"
        }
    }
}
