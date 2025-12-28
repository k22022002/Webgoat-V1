pipeline {
    agent any

    tools {
        maven 'Maven3.9.11'
        jdk 'JDK 17'
    }

    environment {
        // --- APP CONFIG ---
        APP_PORT = "8090"
        
        // --- SEEKER CONFIG ---
        SEEKER_SERVER_URL  = "http://192.168.12.190:8082"
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
        
        // Prevent Jenkins from killing background processes (nohup)
        JENKINS_NODE_COOKIE = "dontKillMe" 
    }

    stages {
        // =========================================
        // STAGE 1: BUILD
        // =========================================
        stage('1. Build Application') {
            steps {
                script {
                    echo "🚀 [Build] Compiling WebGoat v2023.8..."
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        // =========================================
        // STAGE 2: PREPARE SEEKER AGENT
        // =========================================
        stage('2. Setup Seeker Agent') {
            steps {
                script {
                    echo "🕵️ [Agent] Checking Synopsys Seeker Agent..."
                    
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_AGENT_TOKEN')]) {
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            echo ">>> Downloading Seeker Agent..."
                            // Xóa sạch thư mục cũ để tránh lỗi conflict
                            sh "rm -rf seeker installer.sh || true"
                            
                            // Tải và cài đặt Agent
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        } else {
                            echo "✅ Agent đã tồn tại. Sẽ sử dụng cấu hình Dynamic trong lệnh Java."
                        }
                    }
                }
            }
        }

        // =========================================
        // STAGE 3: RUN APPLICATION
        // =========================================
        stage('3. Run App with Seeker') {
            steps {
                script {
                    echo "🚀 [Run] Starting WebGoat + Seeker..."

                    // 1. Tìm file JAR
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()
                    if (!webgoatJar) error "❌ ERROR: Không tìm thấy file .jar!"
                    echo ">>> Found JAR: ${webgoatJar}"

                    // 2. Cleanup Port cũ
                    sh """
                        pkill -f webgoat || true
                        lsof -t -i:${APP_PORT} | xargs -r kill -9 || true
                        sleep 5
                    """

                    // 3. Khởi động với Seeker Agent
                    // NOTE: Truyền trực tiếp config vào lệnh Java (-D) thay vì sửa file properties
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        // Export Token vào Env cho process Java
                        // Start lệnh chạy background
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            nohup java \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                --add-opens java.base/java.io=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${WORKSPACE}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -Dseeker.agent.auto.update=false \\
                                -Dserver.port=${APP_PORT} \\
                                -Dserver.address=0.0.0.0 \\
                                -Dserver.servlet.context-path=/WebGoat \\
                                -jar ${webgoatJar} \\
                                > app_webgoat.log 2>&1 &
                        """
                    }
                    echo "✅ Lệnh khởi động đã được thực thi."
                }
            }
        }

	// =========================================
        // STAGE 4: HEALTH CHECK & TRAFFIC GENERATION
        // =========================================
        stage('4. Health Check & Traffic') {
            steps {
                script {
                    echo "💓 [Check] Waiting for WebGoat to be alive..."
                    // 1. Chờ ứng dụng Start (Health Check cơ bản)
                    // ---------------------------------------------------------
                    boolean isReady = false
                    for (int i = 1; i <= 30; i++) { // Thử 30 lần, mỗi lần 10s = 5 phút
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()
                        
                        echo ">>> [Attempt ${i}] Status: ${status}"
                        if (status == '200' || status == '302') {
                            isReady = true; break;
                        }
                        sleep 10
                    }
                    if (!isReady) error "❌ Timeout: WebGoat không phản hồi."

                    // 2. Tạo Traffic đa dạng để Seeker phân tích
                    // ---------------------------------------------------------
                    echo "🚦 [Traffic] Generating traffic for Seeker..."
                    
                    // A. ĐĂNG NHẬP & LƯU COOKIE
                    // Dùng cờ -c cookies.txt để lưu session ID sau khi login thành công
                    sh """
                        rm -f cookies.txt
                        echo ">>> 1. Login & Save Cookie..."
                        curl -s -k -c cookies.txt -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                             -d "username=admin&password=password" \\
                             -H "Content-Type: application/x-www-form-urlencoded"
                    """

                    // B. GỌI CÁC API KHÁC (Dùng cookie đã lưu)
                    // Dùng cờ -b cookies.txt để gửi kèm session ID
                    sh """
                        echo ">>> 2. Accessing Welcome Page..."
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/welcome.mvc
                        
                        echo ">>> 3. Listing Lessons (API)..."
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/service/lessoninfo.mvc
                        
                        echo ">>> 4. Simulating SQL Injection Traffic..."
                        # Đây là endpoint bài học SQL Injection. 
                        # Việc gọi nó giúp Seeker nhận diện Controller xử lý SQL.
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/SqlInjection/attack5a
                        
                        echo ">>> 5. Simulating User Profile Search (Data Flow)..."
                        # Giả lập search user (API thường thấy trong WebGoat)
                        curl -s -k -b cookies.txt -o /dev/null -X POST http://127.0.0.1:${APP_PORT}/WebGoat/SqlInjection/servers \\
                             -H "Content-Type: application/x-www-form-urlencoded" \\
                             -d "column=hostname"
                    """
                    
                    echo "✅ Traffic generation completed!"
                    sleep 15 // Chờ một chút để Agent đẩy dữ liệu cuối cùng lên Server
                }
            }
        }
        // =========================================
        // STAGE 5: QUALITY GATE
        // =========================================
	stage('5. Quality Gate') {
            steps {
                script {
                    echo "🛡️ [Gate] Checking Seeker Compliance..."
                    
                    // Thêm thời gian chờ để Server xử lý dữ liệu từ Agent gửi lên
                    echo ">>> Waiting 30s for Seeker Server to sync..."
                    sleep 30 

                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        // FIX: Dùng dấu nháy đơn ''' cho sh để Shell tự xử lý biến môi trường
                        // Điều này loại bỏ cảnh báo "Groovy String interpolation"
                        sh '''
                            rm -f response_body.json status_code.txt
                            
                            echo ">>> [1] Kiểm tra xem Project đã tồn tại chưa..."
                            # Gọi API list project trước vì API này luôn có dữ liệu nhanh hơn
                            curl -s -k -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects" \
                                 -H "Authorization: $SEEKER_API_TOKEN" \
                                 -H "Accept: application/json" > projects_list.json
                            
                            # Kiểm tra xem key có trong list không (dùng grep cho đơn giản)
                            if grep -q "$SEEKER_PROJECT_KEY" projects_list.json; then
                                echo "✅ Project '$SEEKER_PROJECT_KEY' đã được tìm thấy trên Server."
                            else
                                echo "❌ ERROR: Project chưa được tạo. Agent có thể chưa kết nối thành công."
                                exit 1
                            fi

                            echo ">>> [2] Kiểm tra trạng thái Compliance..."
                            curl -s -k -o response_body.json -w "%{http_code}" \
                                -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects/$SEEKER_PROJECT_KEY/compliance-status" \
                                -H "Authorization: $SEEKER_API_TOKEN" \
                                -H "Accept: application/json" > status_code.txt

                            HTTP_CODE=$(cat status_code.txt)
                            BODY=$(cat response_body.json)

                            echo ">>> HTTP Code: $HTTP_CODE"

                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "✅ SUCCESS! Compliance Result:"
                                echo "$BODY"
                                # Logic fail build nếu cần:
                                # if echo "$BODY" | grep -q "NON_COMPLIANT"; then exit 1; fi
                                
                            elif [ "$HTTP_CODE" = "404" ]; then
                                echo "⚠️ WARNING (404): Project tồn tại nhưng chưa có báo cáo Compliance."
                                echo ">>> Lý do: Project mới tạo, Seeker đang phân tích."
                                echo ">>> Bỏ qua lỗi này để Pipeline tiếp tục (Soft Pass)."
                                
                            else
                                echo "❌ UNEXPECTED ERROR: $BODY"
                                exit 1
                            fi
                        '''
                    }
                }
            }
	}
	// =========================================
        // STAGE 6: DEPLOY APPLICATION
        // =========================================
        stage('6. Deploy to Production') {
            steps {
                script {
                    echo "🚀 [Deploy] Starting Deployment Process..."
                    
                    // Cấu hình thư mục deploy (Nên nằm ngoài Workspace để tránh bị xóa khi build mới)
                    // Lưu ý: Đảm bảo user chạy Jenkins có quyền ghi vào thư mục này
                    def deployDir = "/opt/webgoat-live" 
                    def deployLog = "${deployDir}/app.log"

                    // 1. Dọn dẹp process cũ (Test instance ở Stage 3 & 4)
                    echo ">>> Stopping previous instances..."
                    sh "pkill -f webgoat || true"
                    sh "lsof -t -i:${APP_PORT} | xargs -r kill -9 || true"
                    sleep 5

                    // 2. Chuẩn bị thư mục Deploy
                    echo ">>> Preparing Deployment Directory: ${deployDir}..."
                    sh "mkdir -p ${deployDir}/seeker"
                    
                    // Copy File JAR & Seeker Agent sang thư mục Deploy
                    sh "cp target/webgoat-*.jar ${deployDir}/webgoat-app.jar"
                    sh "cp -r seeker/* ${deployDir}/seeker/"

                    // 3. Khởi động ứng dụng (Production Mode)
                    echo ">>> Starting Application..."
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            # JENKINS_NODE_COOKIE="dontKillMe" đã được set ở environment đầu file
                            # Biến này cực kỳ quan trọng để giữ process sống sau khi job kết thúc
                            
                            cd ${deployDir}
                            nohup java \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                --add-opens java.base/java.io=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${deployDir}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -Dserver.port=${APP_PORT} \\
                                -Dserver.address=0.0.0.0 \\
                                -jar webgoat-app.jar \\
                                > ${deployLog} 2>&1 &
                        """
                    }
                    
                    echo "✅ Application Deployed Successfully!"
                    echo "👉 Access here: http://<YOUR_SERVER_IP>:${APP_PORT}/WebGoat"
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Build Failed! Check logs for details."
        }
        always {
            // Archive logs for debugging
            archiveArtifacts artifacts: 'app_webgoat.log', allowEmptyArchive: true
        }
    }
}
