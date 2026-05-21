pipeline {
    agent any

    environment {
        // Docker Hub settings
        DOCKER_CREDS_ID = 'docker-hub-creds'
        FRONTEND_IMAGE  = 'shitalkandel/frontend'
        BACKEND_IMAGE   = 'shitalkandel/backend'
        
        // Discord notification credential
        DISCORD_WEBHOOK = credentials('https://discord.com/api/webhooks/1491988177567744151/I1IO7GeK-avC-XaPkxGkPAOqa7J2cLkbSQDIFeB-Le9EXPK2LgQ-2x-M_heAjbPcvRE3')
        
        // Dynamic variables initialized cleanly at runtime
        BUILD_USER      = 'Unknown'
        COMMIT_HASH     = ''
        BUILD_TIME      = ''
    }

    stages {
        stage('Initialize & Metadata') {
            steps {
                script {
                    // Capture git commit hash safely inside the default workspace
                    COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    
                    // Format current date and time cleanly
                    BUILD_TIME = sh(script: 'date "+%Y-%m-%d %H:%M:%S %Z"', returnStdout: true).trim()
                    
                    // Fetch the user who triggered the build using your installed plugin
                    wrap([$class: 'BuildUser']) {
                        if (env.BUILD_USER) {
                            BUILD_USER = env.BUILD_USER
                        }
                    }
                }
                echo "Build started by ${BUILD_USER} at ${BUILD_TIME} on commit ${COMMIT_HASH}"
            }
        }

        stage('Build & Push Backend') {
            steps {
                script {
                    // Injecting DOCKER_USER and DOCKER_PASSWORD securely from Jenkins credentials
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}"
                        
                        echo "Building Backend..."
                        sh "docker build -t ${env.BACKEND_IMAGE}:${COMMIT_HASH} ./backend"
                        sh "docker tag ${env.BACKEND_IMAGE}:${COMMIT_HASH} ${env.BACKEND_IMAGE}:latest"
                        
                        echo "Pushing Backend..."
                        sh "docker push ${env.BACKEND_IMAGE}:${COMMIT_HASH}"
                        sh "docker push ${env.BACKEND_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Build & Push Frontend') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        echo "Building Frontend..."
                        sh "docker build -t ${env.FRONTEND_IMAGE}:${COMMIT_HASH} ./frontend"
                        sh "docker tag ${env.FRONTEND_IMAGE}:${COMMIT_HASH} ${env.FRONTEND_IMAGE}:latest"
                        
                        echo "Pushing Frontend..."
                        sh "docker push ${env.FRONTEND_IMAGE}:${COMMIT_HASH}"
                        sh "docker push ${env.FRONTEND_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Applying Database, Backend, Frontend, and Autoscalers to Cluster..."
                    // Applies statefulsets, services, deployments, and HPA configurations
                    sh "kubectl apply -f k8s/database-statefulset.yml"
                    
                    // Using sed to dynamically replace image version with current commit hash before deploying
                    sh "sed -i 'image: ${env.BACKEND_IMAGE}:.*|image: ${env.BACKEND_IMAGE}:latest' k8s/backend-deployment.yml"
                    sh "kubectl apply -f k8s/backend-deployment.yml"
                    
                    sh "sed -i 'image: ${env.FRONTEND_IMAGE}:.*|image: ${env.FRONTEND_IMAGE}:latest' k8s/frontend-deployment.yml"
                    sh "kubectl apply -f k8s/frontend-deployment.yml"
                    
                    sh "kubectl apply -f k8s/hpa.yml"
                    
                    // Verifying rollouts to ensure everything comes up smoothly
                    sh "kubectl rollout status deployment/backend-deployment --timeout=90s"
                    sh "kubectl rollout status deployment/frontend-deployment --timeout=90s"
                }
            }
        }
    }

    post {
        success {
            script {
                def payload = """
                {
                  "username": "Jenkins CI/CD",
                  "avatar_url": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                  "embeds": [{
                    "title": "🚀 Deployment Successful!",
                    "color": 3066993,
                    "fields": [
                      {"name": "Project", "value": "3-tier-app-k8s", "inline": true},
                      {"name": "Triggered By", "value": "${BUILD_USER}", "inline": true},
                      {"name": "Commit Hash", "value": "`${COMMIT_HASH}`", "inline": true},
                      {"name": "Completed At", "value": "${BUILD_TIME}", "inline": false}
                    ],
                    "footer": { "text": "Environment: Production (Private Network)" }
                  }]
                }
                """
                sendDiscordNotification(payload)
            }
        }
        failure {
            script {
                def payload = """
                {
                  "username": "Jenkins CI/CD",
                  "avatar_url": "https://www.jenkins.io/images/logos/jenkins/jenkins.png",
                  "embeds": [{
                    "title": "❌ Pipeline Broken / Deployment Failed",
                    "color": 15158332,
                    "fields": [
                      {"name": "Project", "value": "3-tier-app-k8s", "inline": true},
                      {"name": "Triggered By", "value": "${BUILD_USER}", "inline": true},
                      {"name": "Commit Hash", "value": "`${COMMIT_HASH}`", "inline": true},
                      {"name": "Failed At", "value": "${BUILD_TIME}", "inline": false},
                      {"name": "Action Required", "value": "Check the Jenkins console logs immediately to diagnose the crash.", "inline": false}
                    ],
                    "footer": { "text": "Environment: Production (Private Network)" }
                  }]
                }
                """
                sendDiscordNotification(payload)
            }
        }
        cleanup {
            cleanWs()
            echo "Cleaned up the workspace successfully."
        }
    }
}

// Reusable helper function to POST raw json data safely to Discord without needing third-party plugins
def sendDiscordNotification(String jsonPayload) {
    sh(
        script: "curl -H 'Content-Type: application/json' -X POST -d '${jsonPayload.replaceAll("\n", "").trim()}' ${env.DISCORD_WEBHOOK}",
        returnStdout: false
    )
}
