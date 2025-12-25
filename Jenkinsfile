pipeline {
    agent any 

    tools {
        // LƯU Ý: Bạn cần cài đặt tên Tool này trong "Global Tool Configuration" của Jenkins
        // WebGoat bản mới thường yêu cầu JDK 17
        maven 'Maven' 
        jdk 'JDK17'   
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
                    // Sử dụng seeker-agent-token như bạn yêu cầu
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        
                        // 1. Reset thư mục
                        sh "rm -rf seeker app_iast.log || true"

                        // 2. Tải Seeker Agent (JAVA)
                        // Đã đổi URL từ NODEJS sang JAVA
                        echo "--- Downloading Seeker Java Agent ---"
                        sh """
                            sh -c "\$(curl -k -X GET -fsSL --header 'Accept: application/x-sh' \
                            'http://192.168.12.190:8082/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&agentName=&accessToken=${SEEKER_ACCESS_TOKEN}')"
                        """

                        // 3. Setup Agent File
                        echo "--- Checking Agent ---"
                        // Script tải về của Seeker cho Java thường tự tạo thư mục 'seeker' và file 'seeker-agent.jar'
                        if (!fileExists('seeker/seeker-agent.jar')) {
                             // Fallback: Tìm file jar nếu cấu trúc thư mục khác
                             sh "find . -name seeker-agent.jar"
                             error "LỖI: Không tìm thấy file seeker-agent.jar sau khi tải."
                        }

                        // 4. Cấu hình môi trường Seeker
                        env.SEEKER_SERVER_URL = "http://192.168.12.190:8082"
                        // Project Key bạn cung cấp
                        env.SEEKER_PROJECT_KEY = "${SEEKER_PROJECT_KEY}" 
                        
                        // 5. Chạy WebGoat với IAST Agent
                        echo "--- Starting WebGoat with Java Agent ---"
                        
                        // Tìm file WebGoat jar vừa build
                        def webgoatJar = sh(script: "ls target/webgoat-*.jar | head -n 1", returnStdout: true).trim()
                        
                        // Kill process cũ nếu có (dựa vào tên file jar)
                        sh "pkill -f 'webgoat' || true"
                        
                        // Lệnh chạy quan trọng:
                        // -javaagent: Đường dẫn tới seeker-agent.jar
                        // -Dserver.port: Cấu hình port cho WebGoat
                        // -Dserver.address: Để truy cập được từ bên ngoài container/server
                        sh """
                            nohup java -javaagent:${env.WORKSPACE}/seeker/seeker-agent.jar \
                            -Dseeker.server.url=${env.SEEKER_SERVER_URL} \
                            -Dseeker.project.key=${env.SEEKER_PROJECT_KEY} \
                            -Dserver.port=${APP_PORT} \
                            -Dserver.address=0.0.0.0 \
                            -jar ${webgoatJar} > app_iast.log 2>&1 &
                        """
                        
                        // WebGoat khởi động RẤT LÂU (Java Spring Boot), cần đợi ít nhất 60-90s
                        echo "Waiting for WebGoat to start (60s)..."
                        sh "sleep 60" 

                        // 6. Kiểm tra log và kết quả
                        echo "================ APP LOGS (Last 100 lines) ================"
                        sh "tail -n 100 app_iast.log"
                        echo "==========================================================="
                        
                        // Kiểm tra process Java còn sống không
                        def isRunning = sh(script: "pgrep -f 'webgoat' > /dev/null && echo 'YES' || echo 'NO'", returnStdout: true).trim()

                        if (isRunning == 'YES') {
                            echo ">>> SUCCESS: WebGoat is running with Seeker Agent!"
                            try {
                                echo "--- Sending Traffic to trigger IAST ---"
                                // Gửi request đơn giản để kích hoạt Agent
                                sh "curl -v http://localhost:${APP_PORT}/WebGoat/login || true"
                                
                                // Nếu bạn có file test script (ví dụ JMeter hoặc Python), chạy ở đây
                                // sh "python3 trigger_traffic.py"
                            } finally {
                                echo "--- Cleaning up ---"
                                sh "sleep 10"
                                sh "pkill -f 'webgoat' || true"
                            }
                        } else {
                            error ">>> App crashed/Stopped. WebGoat không chạy. Kiểm tra log ở trên."
                        }
                    }
                }
            }
        }
    }
}
