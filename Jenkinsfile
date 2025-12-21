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
            
            // 1. Tìm file JAR
            def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | head -n 1', returnStdout: true).trim()
            if (webgoatJar == "") { error "No .jar file found!" }

            // 2. DỌN DẸP TIẾN TRÌNH CŨ (Logic mạnh hơn)
            echo ">>> [Cleanup] Đang kiểm tra tiến trình chiếm cổng ${APP_PORT}..."

            // Cách 1: Tìm PID bằng lsof (nếu có)
            // Cách 2: Tìm PID bằng netstat (phổ biến hơn)
            // Lấy PID ra biến để kill đích danh
            def checkPidCmd = """
                pid=\$(lsof -t -i:${APP_PORT} 2>/dev/null || netstat -nlp 2>/dev/null | grep :${APP_PORT} | awk '{print \$7}' | cut -d'/' -f1)
                echo \$pid
            """
            def pid = sh(script: checkPidCmd, returnStdout: true).trim()

            if (pid != "" && pid.isInteger()) {
                echo ">>> PHÁT HIỆN tiến trình (PID: ${pid}) đang chiếm cổng. Đang cưỡng chế dừng (Kill -9)..."
                sh "kill -9 ${pid} || true"
            } else {
                echo ">>> Không tìm thấy PID cụ thể trên cổng ${APP_PORT}. Chạy rà soát tổng thể..."
            }

            // Phòng hờ: Kill theo tên file (nếu PID không tìm ra nhưng process vẫn treo)
            sh "pkill -f webgoat || true"
            sh "pkill -f ${webgoatJar} || true"

            // QUAN TRỌNG: Đợi hệ điều hành giải phóng cổng hoàn toàn
            echo ">>> Đợi 5 giây để cổng được giải phóng..."
            sh "sleep 5"

            // 3. KIỂM TRA LẦN CUỐI TRƯỚC KHI CHẠY
            // Nếu vẫn còn process chiếm cổng thì báo lỗi ngay lập tức để không start đè
            def finalCheck = sh(script: "netstat -an | grep :${APP_PORT} || echo 'OK'", returnStdout: true).trim()
            if (finalCheck != 'OK' && finalCheck != '') {
                echo "WARNING: Cổng ${APP_PORT} vẫn chưa thực sự rảnh. Log netstat: ${finalCheck}"
            }

            // 4. KHỞI ĐỘNG ỨNG DỤNG
            echo ">>> Đang khởi động WebGoat trên cổng ${APP_PORT}..."
            
            // Sử dụng BUILD_ID=dontKillMe để tránh Jenkins tự kill process khi stage kết thúc (nếu cần debug)
            // Tuy nhiên, với lệnh nohup & chuẩn thì process sẽ chạy nền.
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
                    
                    // Tăng thời gian chờ lên 10 phút (60 lần x 10s) phòng trường hợp máy chậm
                    def maxRetries = 60
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // Lấy HTTP Code. Nếu lỗi connection refused trả về 000
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://192.168.12.190:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> [Wait ${(i*10)}s] HTTP Code: ${status}"

                        // Chấp nhận 200 (OK), 302 (Redirect)
                        // 404 đôi khi xuất hiện lúc App chưa load xong context, ta đợi thêm chút
                        if (status == '200' || status == '302') {
                            echo "✅ SERVER ĐÃ SỐNG! (HTTP ${status})"
                            isReady = true
                            break
                        }

                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ TIMEOUT! In log lần cuối để debug:"
                        sh "tail -n 50 app_webgoat.log || true" 
                        error "WebGoat không thể mở cổng sau 10 phút."
                    }
                    
                    // --- TRAFFIC TEST ---
                    echo "--- [Traffic] Bắn request để Seeker bắt lỗi ---"
                    // Thử cả đường dẫn WebGoat và WebWolf
                    sh "curl -L -v http://192.168.12.190:${APP_PORT}/WebGoat/login || true"
                    sh "curl -L -v http://192.168.12.190:${APP_PORT}/WebGoat/registration || true"
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
