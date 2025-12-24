pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8090"
        
        // --- CẤU HÌNH SEEKER ---
        // Agent sẽ tự động đọc 2 biến môi trường này, không cần khai báo lại trong lệnh java
        SEEKER_SERVER_URL = "http://192.168.12.190:8082" 
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
    }

    stages {
        stage('1. Build Application') {
            steps {
                script {
                    echo '--- [Build] Compiling WebGoat v2023.8 ---'
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo '--- [Agent] Checking Synopsys Seeker Agent ---'
                    // Sử dụng Token dành riêng cho Agent
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            // Tải Agent về nếu chưa có
                            sh "rm -rf seeker installer.sh || true"
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        } else {
                            echo ">>> Agent đã tồn tại, bỏ qua bước tải."
                            // Cập nhật lại config trong thư mục seeker nếu cần (để đảm bảo project key đúng)
                            sh """
                                sed -i 's/^seeker.project.key=.*/seeker.project.key=${SEEKER_PROJECT_KEY}/' seeker/seeker.properties || true
                                sed -i 's|^seeker.server.url=.*|seeker.server.url=${SEEKER_SERVER_URL}|' seeker/seeker.properties || true
                            """
                        }
                    }
                }
            }
        }

        stage('3. Run App (Fixed Location)') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    // Tìm file JAR
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    if (webgoatJar == "") { 
                        webgoatJar = sh(script: 'find . -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    }
                    if (webgoatJar == "") { 
                        error "ERROR: Không tìm thấy file .jar!" 
                    }
                    echo ">>> Found JAR: ${webgoatJar}"

                    // Dọn dẹp cổng cũ
                    echo ">>> [Cleanup] Dọn dẹp cổng ${APP_PORT}..."
                    sh "pkill -f webgoat || true"
                    def pid = sh(script: "lsof -t -i:${APP_PORT} || true", returnStdout: true).trim()
                    if (pid != "" && pid.isInteger()) {
                        sh "kill -9 ${pid} || true"
                    }
                    sh "sleep 5"

                    // KHỞI ĐỘNG
                    // QUAN TRỌNG: Cần pass Token Agent vào biến môi trường SEEKER_ACCESS_TOKEN cho process chạy Java
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        echo ">>> Đang khởi động WebGoat trên cổng ${APP_PORT}..."
                        
                        // Lưu ý: 
                        // 1. Đã xóa -Dseeker.project.key và -Dseeker.server.url để tránh trùng lặp config (vì đã khai báo ở environment block trên cùng)
                        // 2. Biến SEEKER_ACCESS_TOKEN được inject vào môi trường chạy của lệnh java
                        String startCmd = """
                            nohup java \
                            --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
                            --add-opens java.base/java.io=ALL-UNNAMED \
                            -Xmx2g \
                            -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \
                            -Dseeker.agent.auto.update=false \
                            -Dserver.port=${APP_PORT} \
                            -Dserver.address=0.0.0.0 \
                            -Dserver.servlet.context-path=/WebGoat \
                            -jar ${webgoatJar} \
                            > app_webgoat.log 2>&1 &
                        """
                        // Xuất biến env và chạy lệnh
                        sh "export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN} && ${startCmd}"
                        
                        echo ">>> Lệnh khởi động đã được thực thi."
                    }
                }
            }
        }

        stage('4. Deep Health Check') {
            steps {
                script {
                    echo "--- [Check] Đang chờ WebGoat khởi động ---"
                    def maxRetries = 90 
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
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
                        echo "❌ TIMEOUT! Log 50 dòng cuối:"
                        sh "tail -n 50 app_webgoat.log || true" 
                        error "WebGoat không phản hồi."
                    }
                    
                    // --- TRAFFIC TEST ---
                    echo "--- [Traffic] Bắn Request Đăng Nhập để kích hoạt Agent ---"
                    sleep 5
                    sh "curl -s -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/login"
                    
                    // Giả lập đăng nhập -> Kích hoạt IAST Agent tạo Project
                    sh """
                        curl -s -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                        -d "username=admin&password=password" \\
                        -H "Content-Type: application/x-www-form-urlencoded"
                    """
                    
                    echo ">>> Traffic đã gửi. Chờ Agent đồng bộ (15s)..."
                    sleep 15
                }
            }
        }

        stage('5. Quality Gate') {
            steps {
                script {
                    echo '--- [Gate] Kiểm tra kết quả Seeker ---'
                    
                    // QUAN TRỌNG: Ở bước này dùng TOKEN API (CI/CD Token) chứ KHÔNG dùng Token Agent
                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        sh '''
                            rm -f response_body.json status_code.txt

                            echo ">>> Đang gọi API kiểm tra trạng thái compliance..."
                            
                            curl -s -k -o response_body.json -w "%{http_code}" \
                                -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: Bearer $SEEKER_API_TOKEN" \
                                -H "Accept: application/json" > status_code.txt

                            HTTP_CODE=$(cat status_code.txt)
                            BODY=$(cat response_body.json)

                            echo ">>> HTTP Code: $HTTP_CODE"
                            
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ THÀNH CÔNG! Kết quả:"
                                echo "$BODY"
                                # Bạn có thể thêm logic parse JSON ở đây để fail build nếu status là NON_COMPLIANT
                            elif [ "$HTTP_CODE" = "404" ]; then
                                echo "❌ ERROR (404): Không tìm thấy Project Key: $SEEKER_PROJECT_KEY"
                                echo "--- DEBUG: Liệt kê các Project đang có trên Server ---"
                                
                                # Dùng Token API để liệt kê project (Lúc này sẽ chạy được vì Token đúng quyền)
                                curl -s -k -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects" \
                                    -H "Authorization: Bearer $SEEKER_API_TOKEN" \
                                    -H "Accept: application/json"
                                    
                                echo ""
                                echo "--------------------------------------------------------"
                                echo "--- DEBUG: Kiểm tra log kết nối của Agent ---"
                                grep -i "Connected to Seeker" app_webgoat.log || echo "KHÔNG TÌM THẤY LOG KẾT NỐI!"
                                
                                exit 1
                            else
                                echo "❌ API Error ($HTTP_CODE): $BODY"
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
            sh "pkill -f webgoat || true"
        }
    }
}
