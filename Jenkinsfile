pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8090"
        // IP của máy Jenkins/Seeker Server
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
                    // Bỏ qua test unit để build nhanh hơn
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo '--- [Agent] Downloading Synopsys Seeker Agent ---'
                    // Xóa file cũ nếu có
                    sh "rm -rf seeker installer.sh || true"

                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // FIX: Dùng """ (3 dấu nháy kép) để nhận diện biến $
                        sh """
                            curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_ACCESS_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                            
                            chmod +x installer.sh
                            sh installer.sh
                        """
                        
                        // Kiểm tra lại xem file agent đã tải về chưa
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            echo "--- CONTENT OF INSTALLER.SH (ERROR LOG) ---"
                            sh "cat installer.sh || true"
                            error "Lỗi: Vẫn chưa tải được Agent. Vui lòng kiểm tra lại URL hoặc Token."
                        }
                    }
                }
            }
        }
	stage('3. Run App with IAST (Fixed)') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    // 1. Tìm file JAR (SỬA: WebGoat main build ra trong module webgoat-server)
                    // Lưu ý: Tìm trong webgoat-server/target
                    def webgoatJar = sh(script: 'find webgoat-server/target -name "webgoat-*.jar" | grep -v "sources" | head -n 1', returnStdout: true).trim()
                    
                    if (webgoatJar == "") { 
                        // Fallback: Nếu không tìm thấy, thử tìm rộng hơn nhưng ưu tiên file server
                        echo "Không tìm thấy trong webgoat-server, thử tìm toàn cục..."
                        webgoatJar = sh(script: 'find . -name "webgoat-*.jar" | grep "webgoat-server" | grep -v "sources" | head -n 1', returnStdout: true).trim()
                    }

                    if (webgoatJar == "") { error "ERROR: Không tìm thấy file .jar thực thi của WebGoat!" }
                    echo ">>> Found JAR: ${webgoatJar}"

                    // 2. DỌN DẸP TIẾN TRÌNH CŨ
                    echo ">>> [Cleanup] Đang kiểm tra tiến trình chiếm cổng ${APP_PORT}..."
                    
                    // Tìm PID đang chiếm port
                    def checkPidCmd = "lsof -t -i:${APP_PORT} || true"
                    def pid = sh(script: checkPidCmd, returnStdout: true).trim()

                    if (pid != "" && pid.isInteger()) {
                        echo ">>> Kill process ID: ${pid}"
                        sh "kill -9 ${pid} || true"
                    }
                    
                    // Kill theo tên để chắc chắn
                    sh "pkill -f webgoat || true"
                    sh "sleep 5" // Đợi giải phóng port

                    // 3. KHỞI ĐỘNG ỨNG DỤNG
                    echo ">>> Đang khởi động WebGoat trên cổng ${APP_PORT}..."
                    
                    // QUAN TRỌNG: 
                    // 1. -Dserver.servlet.context-path=/WebGoat : Để đảm bảo URL là /WebGoat/login (như Stage 4 mong đợi)
                    // 2. -Dserver.address=0.0.0.0 : Để cho phép truy cập từ bên ngoài (IP 192.168...)
                    String startCmd = """
                        nohup java \
                        --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
                        --add-opens java.base/java.io=ALL-UNNAMED \
                        -Xmx2g \
                        -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \
                        -Dseeker.server.url=${SEEKER_SERVER_URL} \
                        -Dseeker.project.key=${SEEKER_PROJECT_KEY} \
                        -Dseeker.agent.auto.update=false \
                        -Dserver.port=${APP_PORT} \
                        -Dserver.address=0.0.0.0 \
                        -Dserver.servlet.context-path=/WebGoat \
                        -jar ${webgoatJar} \
                        > app_webgoat.log 2>&1 &
                    """
                    sh startCmd
                    
                    echo ">>> Lệnh khởi động đã được thực thi."
                }
            }
        }

        stage('4. Deep Health Check') {
            steps {
                script {
                    echo "--- [Check] Đang chờ cổng ${APP_PORT} phản hồi ---"
                    
                    def maxRetries = 30 // 30 lần x 10s = 5 phút
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // SỬA: Dùng 127.0.0.1 để check nội bộ trên agent cho nhanh và ổn định
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> [Wait ${(i*10)}s] Check Localhost: ${status}"

                        if (status == '200' || status == '302') {
                            echo "✅ SERVER ĐÃ SỐNG! (HTTP ${status})"
                            isReady = true
                            break
                        }
                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ TIMEOUT! In 50 dòng log cuối cùng:"
                        sh "tail -n 50 app_webgoat.log || true" 
                        error "WebGoat không khởi động được."
                    }
                    
                    // --- TRAFFIC TEST (Gửi request vào IP LAN để Seeker ghi nhận đúng luồng network) ---
                    echo "--- [Traffic] Bắn request qua IP LAN (${SEEKER_SERVER_URL} context) ---"
                    // IP máy Jenkins mà Seeker Server nhìn thấy
                    def jenkinsIP = "192.168.12.190" 
                    
                    sh "curl -L -v http://${jenkinsIP}:${APP_PORT}/WebGoat/login || true"
                    sh "curl -L -v http://${jenkinsIP}:${APP_PORT}/WebGoat/registration || true"
                }
            }
        }
        stage('5. Quality Gate') {
            steps {
                script {
                     echo '--- [Gate] Checking Seeker Compliance ---'
                     // Đợi một chút để Seeker Server xử lý dữ liệu vừa gửi
                     sleep 15 
                     
                     withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        def complianceStatus = sh(
                            script: """
                                curl -s -k -X GET "${SEEKER_SERVER_URL}/rest/api/latest/projects/${SEEKER_PROJECT_KEY}/compliance-status" \
                                -H "Authorization: Bearer ${SEEKER_ACCESS_TOKEN}" \
                                -H "Accept: application/json"
                            """, 
                            returnStdout: true
                        ).trim()
                        
                        echo ">>> Compliance Raw Response: ${complianceStatus}"
                        
                        if (complianceStatus.contains("NON_COMPLIANT")) {
                            echo "⚠️ CẢNH BÁO: Dự án không đạt chuẩn bảo mật (NON_COMPLIANT)!"
                            // Uncomment dòng dưới nếu muốn Build Fail khi có lỗi bảo mật
                            // error "Quality Gate Failed: Security vulnerabilities found."
                        } else {
                            echo "✅ Dự án đạt chuẩn bảo mật (COMPLIANT)."
                        }
                     }
                }
            }
        }
    }

    post {
        always {
            echo "--- [Cleanup] Dọn dẹp process ---"
            sh "pkill -f webgoat || true"
        }
    }
}
