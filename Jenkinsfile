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
	stage('3. Run App with IAST (Optimized)') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    def webgoatJar = sh(script: 'find target -name "webgoat-*.jar" | head -n 1', returnStdout: true).trim()
                    if (webgoatJar == "") { error "Không tìm thấy file .jar!" }

                    sh "pkill -f webgoat || true"

                    // CẤU HÌNH MỚI:
                    // 1. Thêm --add-opens để sửa lỗi Java 17 (theo gợi ý trong log của bạn)
                    // 2. Dùng -Dserver.address=0.0.0.0 để chắc chắn mở cổng ra ngoài
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
                    
                    echo ">>> WebGoat đang khởi động (Máy Lab chậm nên sẽ đợi tối đa 10 phút)..."
                }
            }
        }

        stage('4. Deep Health Check') {
            steps {
                script {
                    echo "--- [Check] Đang chờ cổng ${APP_PORT} phản hồi ---"
                    
                    // Tăng lên 60 lần x 10s = 600 giây (10 phút)
                    def maxRetries = 60
                    def isReady = false

                    for (int i = 1; i <= maxRetries; i++) {
                        // Lấy HTTP Code.
                        // WebGoat có thể trả về 200, 302.
                        // WebWolf (nếu chiếm cổng 8090) có thể trả về 404 cho đường dẫn /WebGoat.
                        // NHƯNG: Chỉ cần code != 000 (Connection Refused) nghĩa là Server đã sống!
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()

                        echo ">>> Phút thứ ${(i*10)/60}: HTTP Code = ${status}"

                        // Chấp nhận 200 (OK), 302 (Redirect), 404 (Not Found - nghĩa là WebWolf đang chạy)
                        // Chỉ cần không phải 000 (Chưa kết nối được)
                        if (status != '000' && status != 'FAIL') {
                            echo "✅ SERVER ĐÃ SỐNG! (HTTP ${status})"
                            isReady = true
                            break
                        }

                        sleep 10
                    }

                    if (!isReady) {
                        echo "❌ TIMEOUT 10 PHÚT! In log lần cuối:"
                        sh "tail -n 50 app_webgoat.log" 
                        error "WebGoat không thể mở cổng sau 10 phút."
                    }
                    
                    // --- TRAFFIC TEST ---
                    echo "--- [Traffic] Bắn request để Seeker bắt lỗi ---"
                    // Thử cả đường dẫn WebGoat và WebWolf để đảm bảo trúng đích
                    sh "curl -L -v http://localhost:${APP_PORT}/WebGoat/login || true"
                    sh "curl -L -v http://localhost:${APP_PORT}/WebWolf/login || true"
                    sh "curl -L -v http://localhost:${APP_PORT}/WebGoat/registration || true"
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
