pipeline {
    agent any

    environment {
        DOCKER_USER       = 'shitalkandel'
        GITHUB_REPO_OWNER = 'ShitalKandel'
        GITHUB_REPO_NAME  = '3-tier-app-k8s'
        
        DISCORD_WEBHOOK   = 'https://discord.com/api/webhooks/1491988177567744151/I1IO7GeK-avC-XaPkxGkPAOqa7J2cLkbSQDIFeB-Le9EXPK2LgQ-2x-M_heAjbPcvRE3'

        // Secure credential scopes mapped directly from your Jenkins global credentials store
        DOCKER_CREDS      = credentials('docker-hub-creds')
        GITHUB_TOKEN      = credentials('github-token')
    }

    stages {
        stage('Initialize & System Check') {
            steps {
                echo "Initializing Pipeline Build #${env.BUILD_NUMBER}..."
                // Ensure local microk8s and docker instances are healthy on your host
                sh "docker --version"
                sh "microk8s status --wait-ready"
            }
        }
        
        stage('Build & Push Components') {
            steps {
                echo "Compiling code layers and building target Docker images..."
                script {
                    // Safe injection authentication utilizing your Docker Hub access token
                    sh "echo '${DOCKER_CREDS_PSW}' | docker login -u '${DOCKER_CREDS_USR}' --password-stdin"
                    
                    // 1. Build & Push Backend Components
                    echo "Packaging Backend Engine..."
                    sh "docker build -t ${DOCKER_USER}/backend:latest -t ${DOCKER_USER}/backend:${env.BUILD_NUMBER} ./backend"
                    sh "docker push ${DOCKER_USER}/backend:latest"
                    sh "docker push ${DOCKER_USER}/backend:${env.BUILD_NUMBER}"
                    
                    // 2. Build & Push Frontend Components
                    echo "Packaging Frontend Engine..."
                    sh "docker build -t ${DOCKER_USER}/frontend:latest -t ${DOCKER_USER}/frontend:${env.BUILD_NUMBER} ./frontend"
                    sh "docker push ${DOCKER_USER}/frontend:latest"
                    sh "docker push ${DOCKER_USER}/frontend:${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "Applying manifests to MicroK8s cluster..."
                script {
                    // 1. Deploy the stateful database layer first so it is ready for incoming connections
                    echo "Deploying infrastructure foundation (Database StatefulSet)..."
                    sh "microk8s kubectl apply -f k8s/database-statefulset.yml"
                    
                    // 2. Deploy your microservice application components
                    echo "Deploying application service tiers..."
                    sh "microk8s kubectl apply -f k8s/backend-deployment.yml"
                    sh "microk8s kubectl apply -f k8s/frontend-deployment.yml"
                    sh "microk8s kubectl apply -f k8s/hpa.yml"
                    
                    // 3. Force rolling restarts to bypass local image caching and pull your newly pushed builds
                    echo "Enforcing container image updates..."
                    sh "microk8s kubectl rollout restart deployment/backend || microk8s kubectl rollout restart deployment/backend-deployment || true"
                    sh "microk8s kubectl rollout restart deployment/frontend || microk8s kubectl rollout restart deployment/frontend-deployment || true"
                    
                    // 4. Print running resources for verification visibility
                    echo "Current deployment runtime footprint:"
                    sh "microk8s kubectl get pods,svc,statefulset,hpa -o wide"
                }
            }
        }
    }

    post {
        always {
            script {
                // Kept completely intact to protect your successful notification logic
                def currentStatus = currentBuild.currentResult ?: 'UNKNOWN'
                def targetNode    = env.NODE_NAME ?: 'Devops-Server'
                def buildUser     = 'Shital Kandel' 
                
                def timestamp = sh(script: 'date "+%Y-%m-%d %H:%M:%S %Z"', returnStdout: true).trim()

                def embedColor = (currentStatus == 'SUCCESS') ? 3066993 : 15158332
                def statusEmoji = (currentStatus == 'SUCCESS') ? "🚀 Deployment Successful!" : "❌ Pipeline Action Failed!"

                def payload = """{
                  "embeds": [{
                    "title": "${statusEmoji}",
                    "color": ${embedColor},
                    "fields": [
                      { "name": "Project Repository", "value": "${env.GITHUB_REPO_NAME}", "inline": true },
                      { "name": "Build Number", "value": "#${env.BUILD_NUMBER}", "inline": true },
                      { "name": "Execution Status", "value": "${currentStatus}", "inline": true },
                      { "name": "Triggered By", "value": "${buildUser}", "inline": false },
                      { "name": "Target Environment", "value": "${targetNode}", "inline": true },
                      { "name": "Timestamp", "value": "${timestamp}", "inline": true }
                    ]
                  }]
                }"""

                sh """
                    curl -X POST \
                    -H 'Content-Type: application/json' \
                    -d '${payload}' \
                    '${env.DISCORD_WEBHOOK}'
                """
            }
        }
    }
}
