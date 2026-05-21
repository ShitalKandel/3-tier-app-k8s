pipeline {
    agent any

    environment {
        DOCKER_USER       = 'shitalkandel'
        GITHUB_REPO_OWNER = 'ShitalKandel'
        GITHUB_REPO_NAME  = '3-tier-app-k8s'
        
        // Wrapped securely in single quotes to protect parsing integrity
        DISCORD_WEBHOOK   = 'https://discord.com/api/webhooks/1491988177567744151/I1IO7GeK-avC-XaPkxGkPAOqa7J2cLkbSQDIFeB-Le9EXPK2LgQ-2x-M_heAjbPcvRE3'

        // Credential mappings setup in your Jenkins UI
        DOCKER_CREDS      = credentials('docker-hub-creds')
        GITHUB_TOKEN      = credentials('github-token')
    }

    stages {
        stage('Initialize') {
            steps {
                echo "Initializing Pipeline Build #${env.BUILD_NUMBER}..."
            }
        }
        stage('Build & Push Components') {
            steps {
                echo "Executing component compilation and image packaging..."
                // Insert your core docker build / push commands here
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                echo "Applying manifests to MicroK8s cluster..."
                // Insert your kubectl deployment commands here
            }
        }
    }

    post {
        always {
            script {
                // Zero-plugin variable resolution strategy
                def currentStatus = currentBuild.currentResult ?: 'UNKNOWN'
                def targetNode    = env.NODE_NAME ?: 'Devops-Server'
                def buildUser     = 'Shital Kandel' // Default user fallback to prevent plugin dependency crashes
                
                // Fetch dynamic execution time natively via shell to completely bypass Java timezone quirks
                def timestamp = sh(script: 'date "+%Y-%m-%d %H:%M:%S %Z"', returnStdout: true).trim()

                // Determine notification color (Green for SUCCESS, Red for everything else)
                def embedColor = (currentStatus == 'SUCCESS') ? 3066993 : 15158332
                def statusEmoji = (currentStatus == 'SUCCESS') ? "🚀 Deployment Successful!" : "❌ Pipeline Action Failed!"

                // Construct clean payload matching raw payload standards
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

                // Execute payload transfer to Discord webhook endpoint
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
