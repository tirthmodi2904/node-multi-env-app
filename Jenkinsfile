pipeline {
    agent any

    environment {
        APP_NAME   = "node-multi-env-app"
        NEXUS_URL  = "http://172.31.16.65:8081"
        NEXUS_REPO = "node-app-repo"
        DEV_IP     = "100.48.94.29"
        STAGE_IP   = "54.234.182.140"
        PROD_IP    = "54.83.115.62"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('sonarserver') {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=node-multi-env-app \
                            -Dsonar.sources=. \
                            -Dsonar.login=$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'rm -f app.tar.gz'
                sh 'tar -czf app.tar.gz *'
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexuslogin',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app.tar.gz \
                    ${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}-${BUILD_NUMBER}.tar.gz
                    """
                }
            }
        }

        // ============================
        // DEV DEPLOY (AUTO)
        // ============================
        stage('Deploy to DEV') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp(env.DEV_IP, "dev")
                }
            }
        }

        // ============================
        // STAGE APPROVAL + DEPLOY
        // ============================
        stage('Approve STAGE') {
            steps {
                input message: "Promote build to STAGE environment?"
            }
        }

        stage('Deploy to STAGE') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp(env.STAGE_IP, "stage")
                }
            }
        }

        // ============================
        // PROD APPROVAL + DEPLOY
        // ============================
        stage('Approve PROD') {
            steps {
                input message: "Promote build to PRODUCTION environment?"
            }
        }

        stage('Deploy to PROD') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp(env.PROD_IP, "prod")
                }
            }
        }
    }

    post {
        success {
            echo "FULL ENVIRONMENT PROMOTION COMPLETED SUCCESSFULLY"
        }
        failure {
            echo "PIPELINE FAILED"
        }
    }
}


// ======================================
// COMMON DEPLOYMENT FUNCTION
// ======================================
def deployApp(serverIp, envName) {

    withCredentials([usernamePassword(
        credentialsId: 'nexuslogin',
        usernameVariable: 'NEXUS_USER',
        passwordVariable: 'NEXUS_PASS'
    )]) {

        sh """
        curl -u $NEXUS_USER:$NEXUS_PASS \
        -o app.tar.gz \
        http://172.31.16.65:8081/repository/node-app-repo/node-multi-env-app-${BUILD_NUMBER}.tar.gz

        scp -o StrictHostKeyChecking=no app.tar.gz ec2-user@${serverIp}:/var/www/nodeapp/

        ssh -o StrictHostKeyChecking=no ec2-user@${serverIp} "
            cd /var/www/nodeapp &&
            tar -xzf app.tar.gz &&
            npm install &&
            pm2 delete nodeapp || true &&
            APP_ENV=${envName} pm2 start app.js --name nodeapp
        "
        """
    }
}
