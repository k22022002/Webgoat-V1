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
                    // Tăng số lần thử lên 90 lần (90 * 10s = 900s = 15 phút)
                    // Vì log cho thấy app mất hơn 5 phút (318s) mới lên
                    def maxRetries = 90 
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // Check localhost 127.0.0.1
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> [Wait ${(i*10)}s / 900s] Status: ${status}"
                        
                        if (status == '200' || status == '302') {
                            echo "✅ SERVER ONLINE! WebGoat đã sẵn sàng."
                            isReady = true
                            break
                        }
                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ TIMEOUT! WebGoat khởi động quá lâu (> 15 phút)."
                        echo "--- Log 50 dòng cuối để debug ---"
                        sh "tail -n 50 app_webgoat.log || true" 
                        error "WebGoat không phản hồi."
                    }
                    
                    // --- TRAFFIC TEST (IP LAN) ---
                    // Bước này cực quan trọng để Seeker Agent nhận diện Project
                    echo "--- [Traffic] Bắn request mẫu để Seeker bắt ---"
                    sleep 10 // Chờ thêm chút cho chắc
                    sh "curl -L -v http://192.168.12.190:${APP_PORT}/WebGoat/login || true"
                }
            }
        }
	stage('5. Quality Gate') {
            steps {
                script {
                    echo '--- [Gate] Kiểm tra kết quả Seeker ---'
                    // Tăng thời gian chờ lên 45s để Agent kịp đồng bộ dữ liệu sau khi WebGoat khởi động
                    sleep 45 
                    
                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh '''
                            # Xóa file tạm cũ nếu có
                            rm -f response_body.json status_code.txt

                            echo ">>> Đang gọi API kiểm tra trạng thái bảo mật..."
                            echo ">>> Target: $SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status"
                            
                            # Kỹ thuật tách Status Code và Body ra 2 nơi để tránh lỗi cú pháp shell
                            curl -s -k -o response_body.json -w "%{http_code}" \
                                -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: Bearer $SEEKER_ACCESS_TOKEN" \
                                -H "Accept: application/json" > status_code.txt

                            # Đọc kết quả vào biến
                            HTTP_CODE=$(cat status_code.txt)
                            BODY=$(cat response_body.json)

                            echo ">>> HTTP Code: $HTTP_CODE"
                            echo ">>> Response Body: $BODY"

                            # Xử lý logic
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ THÀNH CÔNG! Đã lấy được trạng thái bảo mật."
                                # Kiểm tra xem Project có pass hay fail (dựa vào JSON trả về)
                                if echo "$BODY" | grep -q '"pass":true'; then
                                    echo "🎉 CLIMBING THE LADDER: Dự án đạt chuẩn bảo mật!"
                                else
                                    echo "⚠️ WARNING: Dự án chưa đạt chuẩn hoặc có lỗ hổng (Compliance = false)."
                                    # Tùy bạn muốn fail build hay không, nếu muốn fail thì bỏ comment dòng dưới:
                                    # exit 1 
                                fi
                            elif [ "$HTTP_CODE" = "404" ]; then
                                echo "❌ ERROR (404): Server không tìm thấy dữ liệu cho Project Key: $SEEKER_PROJECT_KEY"
                                echo "   -> Nguyên nhân 1: Agent chưa kết nối thành công (Check log WebGoat)."
                                echo "   -> Nguyên nhân 2: Chưa có traffic đi qua WebGoat (Check Stage 4)."
                                exit 1
                            elif [ "$HTTP_CODE" = "403" ]; then
                                echo "❌ ERROR (403): Token bị từ chối quyền truy cập!"
                                echo "   -> Vui lòng kiểm tra lại 'seeker-access-token' trong Jenkins Credentials."
                                exit 1
                            else
                                echo "❌ API Error Unknown: $HTTP_CODE"
                                exit 1
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
