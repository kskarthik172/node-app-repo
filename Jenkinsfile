pipeline {
    agent any

    tools {
        nodejs 'node18'
        sonarScanner 'SonarQube Scanner'
    }

    environment {
        APP_NAME     = "sample-node-app"
        EC2_USER     = "ec2-user"
        EC2_HOST     = "13.200.222.111"
        DEPLOY_PATH  = "/opt/node-app"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clean Workspace') {
            steps {
                sh '''
                rm -rf node_modules package-lock.json app.tar.gz
                npm cache clean --force
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test & SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    npm test
                    sonar-scanner
                    '''
                }
            }
        }

        stage('Quality Gate (70%)') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                tar -czf app.tar.gz package.json package-lock.json server.js node_modules
                '''
            }
        }

        stage('Push Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app.tar.gz \
                    http://13.127.155.166:8081/repository/node-artifacts/app.tar.gz
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ec2-ssh-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "mkdir -p ${DEPLOY_PATH}"

                    scp -i $SSH_KEY app.tar.gz ${EC2_USER}@${EC2_HOST}:${DEPLOY_PATH}

                    ssh -i $SSH_KEY ${EC2_USER}@${EC2_HOST} '
                        cd ${DEPLOY_PATH}
                        tar -xzf app.tar.gz
                        npm install --production
                        pkill -f server.js || true
                        nohup node server.js > app.log 2>&1 &
                    '
                    """
                }
            }
        }
    }
}

