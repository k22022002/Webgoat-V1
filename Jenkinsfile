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
                    
                    // --- TRAFFIC TEST MỚI (QUAN TRỌNG) ---
                    echo "--- [Traffic] Bắn Request Đăng Nhập để kích hoạt Agent ---"
                    sleep 5
                    
                    // 1. Request GET (để warm-up)
                    sh "curl -s -o /dev/null http://192.168.12.190:${APP_PORT}/WebGoat/login"
                    
                    // 2. Request POST (Giả lập đăng nhập -> Kích hoạt IAST Agent)
                    // Agent sẽ thấy dữ liệu đi vào sink và tạo project ngay
                    sh """
                        curl -s -X POST http://192.168.12.190:${APP_PORT}/WebGoat/login \\
                        -d "username=admin&password=password" \\
                        -H "Content-Type: application/x-www-form-urlencoded"
                    """
                    
                    echo ">>> Traffic đã gửi. Chờ Agent đồng bộ..."
                }
            }
        }
	stage('5. Quality Gate') {
            steps {
                script {
                    echo '--- [Gate] Kiểm tra kết quả Seeker ---'
                    sleep 45 
                    
                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh '''
                            rm -f response_body.json status_code.txt

                            echo ">>> Đang gọi API kiểm tra..."
                            
                            curl -s -k -o response_body.json -w "%{http_code}" \
                                -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: Bearer $SEEKER_ACCESS_TOKEN" \
                                -H "Accept: application/json" > status_code.txt

                            HTTP_CODE=$(cat status_code.txt)
                            BODY=$(cat response_body.json)

                            echo ">>> HTTP Code: $HTTP_CODE"
                            
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ THÀNH CÔNG! Kết quả:"
                                echo "$BODY"
                            elif [ "$HTTP_CODE" = "404" ]; then
                                echo "❌ ERROR (404): Không tìm thấy Project Key: $SEEKER_PROJECT_KEY"
                                echo "--- DEBUG: Liệt kê các Project đang có trên Server ---"
                                
                                # Lệnh này sẽ in ra danh sách các project server đang thấy
                                curl -s -k -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects" \
                                    -H "Authorization: Bearer $SEEKER_ACCESS_TOKEN" \
                                    -H "Accept: application/json"
                                    
                                echo ""
                                echo "--------------------------------------------------------"
                                
                                # QUAN TRỌNG: In log kết nối của Agent để xem tại sao nó chưa tạo project
                                echo "--- DEBUG: Kiểm tra log kết nối của Agent ---"
                                grep -i "Connected to Seeker" app_webgoat.log || echo "KHÔNG TÌM THẤY LOG KẾT NỐI!"
                                grep -i "Project" app_webgoat.log || true
                                
                                exit 1
                            else
                                echo "❌ API Error: $HTTP_CODE"
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
