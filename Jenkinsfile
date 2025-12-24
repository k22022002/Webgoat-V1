pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8090"
        
        // --- CẤU HÌNH SEEKER (Theo ảnh) ---
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

	stage('2. Get Seeker Attacher') {
            steps {
                script {
                    echo '--- [Agent] Downloading Seeker Attacher (via Binary API) ---'
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        
                        // Xóa file cũ nếu có
                        sh "rm -f seeker-attacher.jar"

                        // --- SỬ DỤNG API CHUẨN CỦA SEEKER ---
                        // Endpoint: /rest/api/latest/installers/agents/binaries/JAVA
                        // Tham số: flavor=ATTACHER (Để lấy riêng file attacher)
                        def DOWNLOAD_URL = "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/binaries/JAVA?flavor=ATTACHER&accessToken=${SEEKER_AGENT_TOKEN}"
                        
                        echo ">>> Đang tải từ API chuẩn: ${DOWNLOAD_URL}"

                        try {
                            // Thêm -f để báo lỗi ngay nếu 404/500
                            // Thêm -L để follow redirect (quan trọng)
                            sh """
                                curl -f -L -v -o seeker-attacher.jar "${DOWNLOAD_URL}"
                            """
                        } catch (Exception e) {
                            echo "❌ DOWNLOAD THẤT BẠI. Kiểm tra lại Token hoặc kết nối mạng."
                            error "Curl Error"
                        }

                        // Kiểm tra kỹ file tải về
                        def fileType = sh(script: "file seeker-attacher.jar", returnStdout: true).trim()
                        echo ">>> Loại file tải về: ${fileType}"

                        // File đúng phải là "Java archive" hoặc "Zip archive"
                        if (!fileType.contains("Zip") && !fileType.contains("Java") && !fileType.contains("JAR")) {
                            echo "❌ FILE HỎNG: Server trả về text/html chứ không phải file JAR."
                            echo ">>> Nội dung file (20 dòng đầu):"
                            sh "head -n 20 seeker-attacher.jar"
                            error "Invalid Jar File"
                        } else {
                            echo "✅ Đã tải seeker-attacher.jar thành công!"
                        }
                    }
                }
            }
        }
        stage('3. Run App & Attach Agent') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat (Standalone) ---'
                    
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    if (webgoatJar == "") { 
                        webgoatJar = sh(script: 'find . -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    }

                    sh "pkill -f webgoat || true"
                    sh "sleep 2"

                    // BƯỚC 1: Khởi động WebGoat BÌNH THƯỜNG (Không có -javaagent)
                    // Lưu lại PID vào file pid.txt để dùng cho lệnh attach
                    sh """
                        echo ">>> Đang khởi động WebGoat..."
                        nohup java \\
                        --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                        --add-opens java.base/java.io=ALL-UNNAMED \\
                        -Xmx2g \\
                        -Dserver.port=${APP_PORT} \\
                        -Dserver.address=0.0.0.0 \\
                        -Dserver.servlet.context-path=/WebGoat \\
                        -jar ${webgoatJar} > app_webgoat.log 2>&1 & echo \$! > pid.txt
                    """
                    
                    // Chờ một chút để tiến trình Java ổn định PID
                    sh "sleep 5"
                    
                    // BƯỚC 2: Thực hiện lệnh Attach như trong HÌNH ẢNH 1
                    echo '--- [Attach] Connecting Seeker Agent to Process ---'
                    def pid = readFile('pid.txt').trim()
                    echo ">>> WebGoat PID: ${pid}"
                    
                    // Lệnh copy y nguyên từ hướng dẫn trong ảnh (Image 1)
                    sh """
                        java -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                             -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                             -jar seeker-attacher.jar ${pid}
                    """
                    
                    echo ">>> Đã gửi lệnh Attach. Kiểm tra log ứng dụng để xác nhận kết nối."
                }
            }
        }

        stage('4. Deep Health Check') {
            steps {
                script {
                    echo "--- [Check] Đang chờ WebGoat khởi động (Max 5 phút) ---"
                    
                    def isStarted = false
                    for (int i = 0; i < 30; i++) { 
                        // Kiểm tra port
                        def checkPort = sh(script: "netstat -an | grep ${APP_PORT} | grep LISTEN || echo 'not_listening'", returnStdout: true).trim()
                        // Kiểm tra HTTP Status
                        def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", returnStdout: true).trim()

                        if (status == '200' || status == '302') {
                            echo "✅ WebGoat đã Online! (Status: ${status})"
                            isStarted = true
                            break
                        }
                        
                        echo ">>> [Wait ${i*10}s] Đang chờ ứng dụng... (Status: ${status})"
                        
                        // Kiểm tra xem Agent đã báo log kết nối chưa
                        sh "grep -i 'Seeker' app_webgoat.log | tail -n 2 || true"
                        sleep 10
                    }

                    if (!isStarted) {
                        echo "❌ TIMEOUT: WebGoat không khởi động được."
                        sh "cat app_webgoat.log"
                        error "Application Start Timeout"
                    }

                    // --- TRAFFIC TEST ---
                    echo "--- [Traffic] Bắn Traffic để kích hoạt Agent ---"
                    sh """
                        curl -s -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                        -d "username=admin&password=password" \\
                        -H "Content-Type: application/x-www-form-urlencoded"
                    """
                    echo ">>> Chờ 15s để dữ liệu đồng bộ về Server..."
                    sleep 15
                }
            }
        }
        
        stage('5. Quality Gate') {
            steps {
                script {
                    echo '--- [Gate] Kiểm tra kết quả Seeker (Image 2) ---'
                    
                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        sh '''
                            rm -f response_body.json status_code.txt
                            
                            # Kiểm tra trạng thái Compliance
                            curl -s -k -o response_body.json -w "%{http_code}" \
                                -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: Bearer $SEEKER_API_TOKEN" \
                                -H "Accept: application/json" > status_code.txt

                            HTTP_CODE=$(cat status_code.txt)
                            BODY=$(cat response_body.json)

                            echo ">>> HTTP Code: $HTTP_CODE"
                            
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ KẾT NỐI THÀNH CÔNG & ĐÃ CÓ DỮ LIỆU!"
                                echo "$BODY"
                            else
                                echo "❌ LỖI: Chưa thấy dữ liệu hoặc không kết nối được."
                                echo "Response: $BODY"
                                echo "--- Debug Log ---"
                                cat app_webgoat.log
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
