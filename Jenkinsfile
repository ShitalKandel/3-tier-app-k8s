pipeline {
    agent any

    environment {
        // Securely pull variables out of the vault to protect your privacy
        DOCKER_USER      = "${env.shitalkandel}"
        GITHUB_REPO_OWNER= 'ShitalKandel'
        GITHUB_REPO_NAME = '3-tier-app-k8s'
        
        // Discord Webhook URL - Set this up inside Jenkins Global Environment or paste directly
        DISCORD_WEBHOOK  = 'https://discord.com/api/webhooks/1491988177567744151/I1IO7GeK-avC-XaPkxGkPAOqa7J2cLkbSQDIFeB-Le9EXPK2LgQ-2x-M_heAjbPcvRE3'
        
        // Credentials mapping
        DOCKER_CREDS     = credentials('docker-hub-creds')
        GITHUB_TOKEN     = credentials('github-token')
    }

    stages {
        stage('Initialize & Pending Status') {
            steps {
                script {
                    // Wrap with BuildUser plugin to find out who initiated the action
                    wrap([$class: 'BuildUser']) {
                        env.BUILD_AUTHOR = env.BUILD_USER ?: 'Git Push Trigger'
                    }
                    githubStatus('pending', 'Jenkins build has started...')
                    sendDiscordNotification("🔄 **Build Started**", "Pipeline execution initiated by **${env.BUILD_AUTHOR}**\nTracking Build: #${env.BUILD_NUMBER}", "3447003") // Blue accent
                }
            }
        }

        stage('Build & Push Components') {
            steps {
                script {
                    // catchError catches issues in this block and flags it as a BUILD failure
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        echo "Building Backend Image..."
                        def backendImage = docker.build("${DOCKER_USER}/backend:latest", "./backend")
                        
                        echo "Pushing Backend to Docker Hub..."
                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials-id') {
                            backendImage.push()
                        }

                        echo "Building Frontend Image..."
                        def frontendImage = docker.build("${DOCKER_USER}/frontend:latest", "./frontend")
                        
                        echo "Pushing Frontend to Docker Hub..."
                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials-id') {
                            frontendImage.push()
                        }
                    }
                }
            }
            post {
                failure {
                    script {
                        sendDiscordNotification("🛑 **Build Failed!**", "Code compilation or Docker packaging failed.\nTriggered by: **${env.BUILD_AUTHOR}**", "15158332") // Red accent
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // catchError here isolates issues to the DEPLOYMENT engine phase
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        configFileProvider([configFile(fileId: 'kubeconfig-secret-id', variable: 'KUBECONFIG')]) {
                            sh '''
                                export KUBECONFIG=${KUBECONFIG}
                                
                                kubectl apply -f k8s/database-statefulset.yml
                                kubectl apply -f k8s/backend-deployment.yml
                                kubectl apply -f k8s/frontend-deployment.yml
                                kubectl apply -f k8s/hpa.yml
                                
                                kubectl rollout restart deployment/backend-deploy
                                kubectl rollout restart deployment/frontend-deploy
                            '''
                        }
                    }
                }
            }
            post {
                failure {
                    script {
                        sendDiscordNotification("⚠️ **Deployment Failed!**", "Docker containers built fine, but Kubernetes cluster rollout rejected the manifests.\nTriggered by: **${env.BUILD_AUTHOR}**", "16750848") // Orange accent
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                githubStatus('success', "Build passed! Triggered by: ${env.BUILD_AUTHOR}")
                sendDiscordNotification("🚀 **Deployment Successful!**", "All 3-tiers packaged and successfully instantiated on the K8s cluster.\nTriggered by: **${env.BUILD_AUTHOR}**", "3066993") // Green accent
            }
        }
        failure {
            script {
                githubStatus('failure', "Build pipeline broke. Triggered by: ${env.BUILD_AUTHOR}")
            }
        }
    }
}

// Reusable Helper Function for Discord API rich embedded payloads
def sendDiscordNotification(String title, String description, String colorHex) {
    def jsonPayload = """
    {
      "embeds": [{
        "title": "${title}",
        "description": "${description}",
        "color": ${colorHex},
        "footer": {
          "text": "Jenkins CI/CD Automation"
        }
      }]
    }
    """
    sh """
        curl -H "Content-Type: application/json" \
             -X POST \
             -d '${jsonPayload}' \
             ${DISCORD_WEBHOOK}
    """
}

// Helper function to update status directly back to GitHub's UI
def githubStatus(String state, String description) {
    def commitSha = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    def payload = """
    {
      "state": "${state}",
      "target_url": "${env.BUILD_URL}",
      "description": "${description}",
      "context": "continuous-integration/jenkins"
    }
    """
    sh """
        curl -s -X POST \
          -H "Authorization: token ${GITHUB_TOKEN}" \
          -H "Accept: application/vnd.github.v3+json" \
          -d '${payload}' \
          https://api.github.com/repos/${GITHUB_REPO_OWNER}/${GITHUB_REPO_NAME}/statuses/${commitSha}
    """
}
