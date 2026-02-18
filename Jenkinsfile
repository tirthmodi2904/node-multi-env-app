pipeline {
    agent any

    environment {
        APP_NAME = "node-multi-env-app"
        NEXUS_URL = "http://172.31.16.65:8081"
        NEXUS_REPO = "node-app-repo"
        GIT_URL = "https://github.com/tirthmodi2904/node-multi-env-app.git"
        BRANCH = "develop"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git url: "${GIT_URL}", branch: "${BRANCH}"
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

        stage('Deploy to DEV') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexuslogin',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    -o app.tar.gz \
                    ${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}-${BUILD_NUMBER}.tar.gz

                    scp -o StrictHostKeyChecking=no app.tar.gz ec2-user@98.81.247.44:/var/www/nodeapp/

                    ssh -o StrictHostKeyChecking=no ec2-user@98.81.247.44 "
                        cd /var/www/nodeapp &&
                        tar -xzf app.tar.gz &&
                        npm install &&
                        pm2 delete nodeapp || true &&
                        APP_ENV=dev pm2 start app.js --name nodeapp
                    "
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'DEV DEPLOYMENT SUCCESS'
        }
        failure {
            echo 'PIPELINE FAILED'
        }
    }
}
