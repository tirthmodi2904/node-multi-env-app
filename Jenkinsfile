pipeline {
    agent any

    environment {
        APP_NAME = "node-multi-env-app"
        NEXUS_URL = "http://172.31.16.65:8081"
        NEXUS_REPO = "node-app-repo"
        GIT_URL = "https://github.com/tirthmodi2904/node-multi-env-app.git"
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

        stage('SonarQube Quality Gate') {
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

        // =========================
        // DEV DEPLOYMENT (AUTO)
        // =========================
        stage('Deploy to DEV') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp("98.81.247.44", "dev")
                }
            }
        }

        // =========================
        // STAGE DEPLOYMENT (MANUAL)
        // =========================
        stage('Approval for STAGE') {
            when {
                branch 'stage'
            }
            steps {
                input message: "Deploy to STAGE environment?"
            }
        }

        stage('Deploy to STAGE') {
            when {
                branch 'stage'
            }
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp("54.164.129.101", "stage")
                }
            }
        }

        // =========================
        // PROD DEPLOYMENT (MANUAL)
        // =========================
        stage('Approval for PROD') {
            when {
                branch 'main'
            }
            steps {
                input message: "Deploy to PRODUCTION environment?"
            }
        }

        stage('Deploy to PROD') {
            when {
                branch 'main'
            }
            steps {
                sshagent(['ec2-ssh-key']) {
                    deployApp("54.83.115.62", "prod")
                }
            }
        }
    }

    post {
        success {
            echo "PIPELINE COMPLETED SUCCESSFULLY"
        }
        failure {
            echo "PIPELINE FAILED"
        }
    }
}


// ========================================
// REUSABLE DEPLOYMENT FUNCTION
// ========================================
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
