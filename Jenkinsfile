pipeline {
    agent any 

    tools {
        maven 'Maven3.9.11' 
        jdk 'JDK 17'
    }

    environment {
        APP_PORT = "8080"
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
                    withCredentials([string(credentialsId: 'seeker-access-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh "rm -rf seeker || true"
                        sh """
                            sh -c "\$(curl -k -X GET -fsSL --header 'Accept: application/x-sh' \
                            '${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=\${SEEKER_ACCESS_TOKEN}')"
                        """
                        if (!fileExists('seeker/seeker-agent.jar')) {
                            error "Lỗi: Download Agent thất bại!"
                        }
                    }
                }
            }
        }

        stage('3. Run App with IAST') {
            steps {
                script {
                    echo '--- [Run] Starting WebGoat + Seeker ---'
                    
                    def webgoatJar = sh(script: "ls \${JAR_PATTERN} | head -n 1", returnStdout: true).trim()
                    if (!webgoatJar) { error "Không tìm thấy file .jar để chạy!" }

                    sh "pkill -f webgoat || true"

                    // WebGoat 2023.8 chạy tốt trên Java 17
                    String startCmd = """
                        nohup java \
                        -javaagent:\${WORKSPACE}/seeker/seeker-agent.jar \
                        -Dseeker.server.url=\${SEEKER_SERVER_URL} \
                        -Dseeker.project.key=\${SEEKER_PROJECT_KEY} \
                        -Dserver.port=\${APP_PORT} \
                        -Dserver.address=0.0.0.0 \
                        -jar \${webgoatJar} \
                        > app_webgoat.log 2>&1 &
                    """
                    sh startCmd
                    
                    echo ">>> Waiting 45s for WebGoat to start..."
                    sleep 45 
                }
            }
        }

        stage('4. Test & Generate Traffic') {
            steps {
                script {
                    echo '--- [Test] Sending Requests ---'
                    // WebGoat 2023 URL có thể hơi khác, ta thử cả 2 đường dẫn
                    sh "curl -v http://localhost:\${APP_PORT}/WebGoat/login || true"
                    sh "curl -v http://localhost:\${APP_PORT}/webgoat/login || true"
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
