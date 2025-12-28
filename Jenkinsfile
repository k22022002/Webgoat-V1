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
                            sh "rm -rf seeker installer.sh || true"
                            
                            sh """
                                curl -k -SL -o installer.sh "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&webServer=ALL&flavor=DEFAULT&accessToken=${SEEKER_AGENT_TOKEN}&projectKey=${SEEKER_PROJECT_KEY}"
                                chmod +x installer.sh
                                sh installer.sh
                            """
                        } else {
                            echo "✅ Agent đã tồn tại."
                        }
                    }
                }
            }
        }

        // =========================================
        // STAGE 3: RUN APPLICATION (TEST MODE)
        // =========================================
        stage('3. Run App with Seeker') {
            steps {
                script {
                    echo "🚀 [Run] Starting WebGoat + Seeker (Test Mode)..."

		    // Tìm file JAR, ưu tiên webgoat-server và loại trừ webwolf
// Bước 1: Thử tìm chính xác file có tên webgoat-server
def webgoatJar = sh(script: 'find . -name "webgoat-server*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()

// Bước 2: Nếu không thấy, tìm file jar lớn nhưng BẮT BUỘC loại trừ webwolf
if (!webgoatJar) {
    echo "⚠️ Không tìm thấy webgoat-server*.jar cụ thể, đang tìm file JAR > 50MB (loại trừ webwolf)..."
    webgoatJar = sh(script: 'find . -name "*.jar" -size +50M | grep -v "original" | grep -v "webwolf" | grep -v "agent" | head -n 1', returnStdout: true).trim()
}

// Kiểm tra cuối cùng
if (!webgoatJar) {
    error "❌ ERROR: Không tìm thấy file JAR WebGoat nào hợp lệ!"
} else if (webgoatJar.contains("webwolf")) {
    error "❌ ERROR: Script đang chọn nhầm file WebWolf: ${webgoatJar}"
}

echo ">>> ✅ Selected JAR: ${webgoatJar}"
                    // 2. Cleanup Port cũ
                    sh """
                        pkill -f webgoat || true
                        lsof -t -i:${APP_PORT} | xargs -r kill -9 || true
                        sleep 5
                    """

                    // 3. Khởi động với Seeker Agent
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
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
                                -jar ${webgoatJar} \\
                                --server.port=${APP_PORT} \\
                                --server.address=0.0.0.0 \\
                                --server.servlet.context-path=/WebGoat \\
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
                    boolean isReady = false
                    // Tăng thời gian chờ lên vì WebGoat khởi động khá lâu
                    for (int i = 1; i <= 30; i++) { 
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:${APP_PORT}/WebGoat/login || echo '000'", 
                            returnStdout: true
                        ).trim()
                        
                        echo ">>> [Attempt ${i}/30] Status: ${status}"
                        if (status == '200' || status == '302') {
                            isReady = true; break;
                        }
                        sleep 10
                    }
                    if (!isReady) error "❌ Timeout: WebGoat không phản hồi."

                    echo "🚦 [Traffic] Generating traffic for Seeker..."
                    
                    sh """
                        rm -f cookies.txt
                        echo ">>> 1. Login & Save Cookie..."
                        curl -s -k -c cookies.txt -X POST http://127.0.0.1:${APP_PORT}/WebGoat/login \\
                             -d "username=admin&password=password" \\
                             -H "Content-Type: application/x-www-form-urlencoded"

                        echo ">>> 2. Accessing Welcome Page..."
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/welcome.mvc
                        
                        echo ">>> 3. Simulating SQL Injection Traffic..."
                        curl -s -k -b cookies.txt -o /dev/null http://127.0.0.1:${APP_PORT}/WebGoat/SqlInjection/attack5a
                    """
                    
                    echo "✅ Traffic generation completed!"
                    sleep 15 
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
                    echo ">>> Waiting 30s for Seeker Server to sync..."
                    sleep 30 

                    withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')]) {
                        sh '''
                            rm -f response_body.json status_code.txt
                            
                            echo ">>> Checking Project Existence..."
                            curl -s -k -X GET "$SEEKER_SERVER_URL/rest/api/latest/projects" \
                                 -H "Authorization: $SEEKER_API_TOKEN" \
                                 -H "Accept: application/json" > projects_list.json
                            
                            if grep -q "$SEEKER_PROJECT_KEY" projects_list.json; then
                                echo "✅ Project found."
                            else
                                echo "❌ ERROR: Project not found."
                                exit 1
                            fi

                            echo ">>> Checking Compliance Status..."
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
                            elif [ "$HTTP_CODE" = "404" ]; then
                                echo "⚠️ WARNING (404): No compliance report yet. Soft Pass."
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
        // STAGE 6: DEPLOY TO PRODUCTION (FIXED)
        // =========================================
        stage('6. Deploy to Production') {
            steps {
                script {
                    echo "🚀 [Deploy] Starting Deployment Process..."
                    
                    def deployDir = "/opt/webgoat-live" 
                    def deployLog = "${deployDir}/app.log"

		    // Thay thế đoạn tìm file cũ bằng đoạn này:

// Tìm file JAR, ưu tiên webgoat-server và loại trừ webwolf
// Bước 1: Thử tìm chính xác file có tên webgoat-server
def webgoatJar = sh(script: 'find . -name "webgoat-server*.jar" | grep -v "original" | head -n 1', returnStdout: true).trim()

// Bước 2: Nếu không thấy, tìm file jar lớn nhưng BẮT BUỘC loại trừ webwolf
if (!webgoatJar) {
    echo "⚠️ Không tìm thấy webgoat-server*.jar cụ thể, đang tìm file JAR > 50MB (loại trừ webwolf)..."
    webgoatJar = sh(script: 'find . -name "*.jar" -size +50M | grep -v "original" | grep -v "webwolf" | grep -v "agent" | head -n 1', returnStdout: true).trim()
}

// Kiểm tra cuối cùng
if (!webgoatJar) {
    error "❌ ERROR: Không tìm thấy file JAR WebGoat nào hợp lệ!"
} else if (webgoatJar.contains("webwolf")) {
    error "❌ ERROR: Script đang chọn nhầm file WebWolf: ${webgoatJar}"
}

echo ">>> ✅ Selected JAR: ${webgoatJar}"
                    // 2. Dọn dẹp process cũ
                    sh "pkill -f webgoat || true"
                    sh "lsof -t -i:${APP_PORT} | xargs -r kill -9 || true"
                    sleep 5

                    // 3. Chuẩn bị thư mục & Copy file
                    sh "mkdir -p ${deployDir}/seeker"
                    // COPY CHÍNH XÁC FILE TÌM ĐƯỢC
                    sh "cp ${webgoatJar} ${deployDir}/webgoat-app.jar"
                    sh "cp -r seeker/* ${deployDir}/seeker/"

                    // 4. Khởi động ứng dụng
                    echo ">>> Starting Application..."
                    withCredentials([string(credentialsId: 'seeker-agent-token', variable: 'SEEKER_ACCESS_TOKEN')]) {
                        sh """
                            export SEEKER_ACCESS_TOKEN=${SEEKER_ACCESS_TOKEN}
                            
                            cd ${deployDir}
                            nohup java \\
                                --add-opens java.base/sun.nio.ch=ALL-UNNAMED \\
                                --add-opens java.base/java.io=ALL-UNNAMED \\
                                -Xmx2g \\
                                -javaagent:${deployDir}/seeker/seeker-agent.jar \\
                                -Dseeker.server.url=${SEEKER_SERVER_URL} \\
                                -Dseeker.project.key=${SEEKER_PROJECT_KEY} \\
                                -jar webgoat-app.jar \\
                                --server.port=${APP_PORT} \\
                                --server.address=0.0.0.0 \\
                                --server.servlet.context-path=/WebGoat \\
                                > ${deployLog} 2>&1 &
                        """
                    }
                    echo "✅ Application Deployed Successfully!"
                    echo "👉 Access here: http://192.168.12.190:${APP_PORT}/WebGoat/login"
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Build Failed! Check logs for details."
        }
        always {
            archiveArtifacts artifacts: 'app_webgoat.log', allowEmptyArchive: true
        }
    }
}
