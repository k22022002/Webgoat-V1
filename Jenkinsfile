pipeline {
    agent any 

    tools {
        // LƯU Ý: Bạn cần cài đặt tên Tool này trong "Global Tool Configuration" của Jenkins
        // WebGoat bản mới thường yêu cầu JDK 17
        maven 'Maven3.9.11' 
        jdk 'JDK 17'   
    }

    environment {
        // WebGoat mặc định chạy 8080, ta đổi sang 8090 để tránh trùng port Jenkins nếu chạy chung
        APP_PORT = "8090"
        SEEKER_PROJECT_KEY = "webgoat-2023-demo"
    }

    stages {
        // --- BƯỚC 1: SETUP & BUILD (Java/Maven) ---
        stage('1. Smart Build') {
            steps {
                script {
                    echo '--- [Setup] Checking Source & Build Artifacts ---'
                    
                    // 1. Kiểm tra Source Code
                    if (!fileExists('pom.xml')) {
                        echo 'Source code not found. Checking out...'
                        checkout scm
                    } else {
                        echo '>>> Source code found (pom.xml).'
                    }

                    // 2. Kiểm tra file .jar đã build chưa (để tiết kiệm thời gian)
                    // WebGoat build xong thường nằm trong folder target/
                    def jarExists = sh(script: "ls target/webgoat-*.jar 2>/dev/null || echo 'NO'", returnStdout: true).trim()

                    if (jarExists == 'NO') {
                        echo 'Build artifact not found. Running Maven Build...'
                        // Skip test unit để build nhanh hơn, vì ta đang cần chạy IAST
                        sh 'mvn clean install -DskipTests'
                    } else {
                        echo '>>> Build artifact found in target/. Skipping Maven Build.'
                    }
                }
            }
        }
	stage('4. IAST (Synopsys Seeker)') {
            steps {
                script {
                    echo '--- [Step] Synopsys Seeker IAST Setup for JAVA ---'
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        
                        // 1. Dọn dẹp
                        sh "rm -rf seeker install_agent.sh app_iast.log || true"

                        // 2. Tải Seeker Agent (FIXED)
                        // Dùng nháy đơn (Single Quote) để an toàn bảo mật và tránh lỗi cú pháp
                        echo "--- Downloading Seeker Java Agent Script ---"
                        sh '''
                            curl -k -X GET -fsSL \
                            "http://192.168.12.190:8082/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=DEFAULT&flavor=DEFAULT&accessToken=$SEEKER_ACCESS_TOKEN" \
                            -o install_agent.sh
                        '''

                        // 3. Kiểm tra file script tải về có lỗi không (Đôi khi tải về file lỗi HTML)
                        // Nếu file có chứa chữ "html" hoặc "error" ở dòng đầu thì báo lỗi ngay
                        def fileType = sh(script: "head -n 1 install_agent.sh", returnStdout: true).trim()
                        echo "Downloaded file header: ${fileType}"

                        // 4. Chạy script cài đặt
                        echo "--- Installing Agent ---"
                        sh "chmod +x install_agent.sh"
                        sh "./install_agent.sh"

                        // 5. Setup Agent File
                        echo "--- Checking Agent ---"
                        // Sau khi chạy script, nó thường tạo folder 'seeker' chứa 'seeker-agent.jar'
                        if (!fileExists('seeker/seeker-agent.jar')) {
                             // Fallback: Tìm file jar bất kỳ trong folder seeker
                             sh "find seeker -name '*.jar'"
                             error "LỖI: Không tìm thấy file seeker-agent.jar. Kiểm tra lại token hoặc URL."
                        }

                        // 6. Cấu hình môi trường Seeker
                        env.SEEKER_SERVER_URL = "http://192.168.12.190:8082"
                        env.SEEKER_PROJECT_KEY = "${SEEKER_PROJECT_KEY}" 
                        
                        // 7. Chạy WebGoat với IAST Agent
                        echo "--- Starting WebGoat with Java Agent ---"
                        
                        def webgoatJar = sh(script: "ls target/webgoat-*.jar | head -n 1", returnStdout: true).trim()
                        
                        sh "pkill -f 'webgoat' || true"
                        
                        // Chạy Java với Agent
                        // Lưu ý: -Dseeker.server.url là bắt buộc để agent biết gửi dữ liệu về đâu
                        sh """
                            nohup java -javaagent:${env.WORKSPACE}/seeker/seeker-agent.jar \
                            -Dseeker.server.url=${env.SEEKER_SERVER_URL} \
                            -Dseeker.project.key=${env.SEEKER_PROJECT_KEY} \
                            -Dserver.port=${APP_PORT} \
                            -Dserver.address=0.0.0.0 \
                            -jar ${webgoatJar} > app_iast.log 2>&1 &
                        """
                        
                        echo "Waiting for WebGoat to start (60s)..."
                        sh "sleep 60" 

                        // 8. Kiểm tra log
                        echo "================ APP LOGS (Last 50 lines) ================"
                        sh "tail -n 50 app_iast.log"
                        echo "=========================================================="
                        
                        def isRunning = sh(script: "pgrep -f 'webgoat' > /dev/null && echo 'YES' || echo 'NO'", returnStdout: true).trim()

                        if (isRunning == 'YES') {
                            echo ">>> SUCCESS: WebGoat is running!"
                            try {
                                echo "--- Sending Traffic ---"
                                sh "curl -v http://localhost:${APP_PORT}/WebGoat/login || true"
                            } finally {
                                sh "sleep 5"
                                sh "pkill -f 'webgoat' || true"
                            }
                        } else {
                            error ">>> App crashed. WebGoat không chạy."
                        }
                    }
                }
            }
        }
    }
}
