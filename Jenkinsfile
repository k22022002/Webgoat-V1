pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8090"
        // IP máy Jenkins (để Seeker Server connect về nếu cần)
        SEEKER_SERVER_URL = "http://192.168.12.190:8082" 
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo '--- [Build] Compiling WebGoat v2023.8 ---'
                    // Lệnh build của bạn đã chạy ngon
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo '--- [Agent] Checking Synopsys Seeker Agent ---'
                    // Nếu chưa có file agent thì mới tải
                    if (!fileExists('seeker/seeker-agent.jar')) {
                        sh "rm -rf seeker installer.sh || true"
                        withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_ACCESS_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        }
                    } else {
                        echo ">>> Agent đã tồn tại, bỏ qua bước tải."
                    }
                }
            }
        }

        stage('3. Run App (Fixed Location)') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    // --- SỬA LOGIC TÌM FILE DỰA TRÊN LOG CỦA BẠN ---
                    // Log cho thấy file nằm ở: target/webgoat-2023.8.jar
                    // Ta dùng grep -v "original" để tránh lấy nhầm file backup của maven
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()

                    if (webgoatJar == "") { 
                        // Fallback phòng hờ
                        webgoatJar = sh(script: 'find . -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    }

                    if (webgoatJar == "") { 
                        sh "ls -R target" // Debug nếu lỗi
                        error "ERROR: Không tìm thấy file .jar! (Đã check trong target/)" 
                    }
                    
                    echo ">>> Found JAR: ${webgoatJar}"

                    // 2. DỌN DẸP PORT 8090
                    echo ">>> [Cleanup] Dọn dẹp cổng ${APP_PORT}..."
                    def pid = sh(script: "lsof -t -i:${APP_PORT} || true", returnStdout: true).trim()
                    if (pid != "" && pid.isInteger()) {
                        sh "kill -9 ${pid} || true"
                    }
                    sh "pkill -f webgoat || true"
                    sh "sleep 5"

                    // 3. KHỞI ĐỘNG
                    echo ">>> Đang khởi động WebGoat trên cổng ${APP_PORT}..."
                    
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
                    echo "--- [Check] Đang chờ WebGoat khởi động ---"
                    def maxRetries = 30 
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // Check localhost 127.0.0.1 cho ổn định
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> [Wait ${(i*10)}s] Status: ${status}"
                        if (status == '200' || status == '302') {
                            echo "✅ SERVER ONLINE!"
                            isReady = true
                            break
                        }
                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ TIMEOUT! Log 50 dòng cuối:"
                        sh "tail -n 50 app_webgoat.log || true" 
                        error "WebGoat không phản hồi."
                    }
                    
                    // --- TRAFFIC TEST (IP LAN) ---
                    echo "--- [Traffic] Bắn request mẫu để Seeker bắt ---"
                    sh "curl -L -v http://192.168.12.190:${APP_PORT}/WebGoat/login || true"
                }
            }
        }

	stage('5. Quality Gate') {
            steps {
                script {
                    echo '--- [Gate] Kiểm tra kết quả Seeker ---'
                    sleep 30 // Chờ 30s cho Agent đồng bộ dữ liệu
                    
                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // QUAN TRỌNG: Dùng ''' (nháy đơn) thay vì """ (nháy kép)
                        // Lúc này $SEEKER_ACCESS_TOKEN là biến môi trường, an toàn tuyệt đối.
                        sh '''
                            echo ">>> Đang gọi API kiểm tra..."
                            
                            # Lưu ý: Trong block nháy đơn này, các biến environment {} ở đầu file
                            # cũng được gọi bằng $VAR (ví dụ $SEEKER_SERVER_URL)
                            
                            RESPONSE=$(curl -s -k -w "%{http_code}" -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: Bearer $SEEKER_ACCESS_TOKEN" \
                                -H "Accept: application/json")

                            # Cắt chuỗi để lấy HTTP Code (3 ký tự cuối)
                            HTTP_CODE=${RESPONSE: -3}
                            # Lấy nội dung JSON (trừ 3 ký tự cuối)
                            BODY=${RESPONSE:0:${#RESPONSE}-3}

                            echo ">>> HTTP Code: $HTTP_CODE"
                            echo ">>> Response Body: $BODY"

                            if [ "$HTTP_CODE" == "404" ]; then
                                echo "❌ ERROR: Không tìm thấy Project Key ($SEEKER_PROJECT_KEY) trên Server."
                                echo "--- DEBUG: Danh sách các Project đang có trên Seeker ---"
                                
                                # Lệnh debug để bạn xem tên project đúng là gì
                                curl -s -k -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects" \
                                    -H "Authorization: Bearer $SEEKER_ACCESS_TOKEN" \
                                    -H "Accept: application/json"
                                    
                                echo "" # Xuống dòng cho đẹp
                                exit 1
                            elif [ "$HTTP_CODE" != "200" ]; then
                                echo "❌ API Error: $HTTP_CODE"
                                exit 1
                            else
                                echo "✅ Thành công! Trạng thái bảo mật:"
                                echo "$BODY"
                            fi
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "--- [Cleanup] ---"
            // Tắt WebGoat sau khi xong việc
            sh "pkill -f webgoat || true"
        }
    }
}
