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
                    
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    if (webgoatJar == "") { 
                        webgoatJar = sh(script: 'find . -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    }
                    if (webgoatJar == "") { error "ERROR: Không tìm thấy file .jar!" }

                    // Dọn dẹp cổng cũ
                    sh "pkill -f webgoat || true"
                    sh "sleep 5"

                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        echo ">>> Đang khởi động WebGoat trên cổng ${APP_PORT}..."
                        
                        // --- SỬA ĐỔI: Thêm lại các tham số cấu hình trực tiếp ---
                        String startCmd = """
                            nohup java \\
                            --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                            --add-opens java.base/java.io=ALL-UNNAMED \\
                            -Xmx2g \\
                            -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                            -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                            -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                            -Dseeker.agent.debug=true \\
                            -Dserver.port=${APP_PORT} \\
                            -Dserver.address=0.0.0.0 \\
                            -Dserver.servlet.context-path=/WebGoat \\
                            -jar ${webgoatJar} \\
                            > app_webgoat.log 2>&1 &
                        """
                        
                        // Vẫn giữ export Token
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
                    // ... (Giữ nguyên phần check WebGoat port 200 như cũ) ...
                    
                    // --- THÊM: Kiểm tra log Agent kết nối ---
                    echo "--- [Check] Kiểm tra kết nối Seeker Agent ---"
                    int agentRetries = 0
                    while (agentRetries < 20) {
                        // Grep tìm dòng "Connected to Seeker"
                        def agentLog = sh(script: "grep -i 'Connected to Seeker' app_webgoat.log || echo 'waiting'", returnStdout: true).trim()
                        if (agentLog != 'waiting') {
                            echo "✅ Seeker Agent đã kết nối thành công!"
                            break
                        }
                        echo ">>> Đang chờ Agent kết nối... (${agentRetries}/20)"
                        sleep 5
                        agentRetries++
                    }
                    
                    if (agentRetries >= 20) {
                         echo "⚠️ CẢNH BÁO: Agent chưa log 'Connected'. Kiểm tra debug log:"
                         sh "cat app_webgoat.log | grep -i 'seeker' | tail -n 20"
                         // Không error ngay để cho traffic chạy thử
                    }

                    // --- TRAFFIC TEST (Giữ nguyên) ---
                    echo "--- [Traffic] Bắn Request Đăng Nhập ---"
                    sleep 5
                    sh """
                        curl -s -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                        -d "username=admin&password=password" \\
                        -H "Content-Type: application/x-www-form-urlencoded"
                    """
                    echo ">>> Traffic done. Chờ đồng bộ (15s)..."
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
                                -H "Authorization: $SEEKER_API_TOKEN" \
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
                                    -H "Authorization: $SEEKER_API_TOKEN" \
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
