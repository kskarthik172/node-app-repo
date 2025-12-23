pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        APP_NAME    = "sample-node-app"
        EC2_USER    = "ec2-user"
        EC2_HOST    = "13.233.139.139"
        DEPLOY_PATH = "/opt/node-app"

        NEXUS_URL   = "http://13.232.63.14:8081"
        NEXUS_REPO  = "node-artifacts"
        ARTIFACT    = "app.tar.gz"
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
                script {
                    def scannerHome = tool 'SonarQube Scanner'
                    withSonarQubeEnv('SonarQube') {
                        withCredentials([
                            string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')
                        ]) {
                            withEnv(["PATH=${scannerHome}/bin:${env.PATH}"]) {
                                sh '''
                                npm test
                                sonar-scanner \
                                  -Dsonar.login=$SONAR_TOKEN
                                '''
                            }
                        }
                    }
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
                tar -czf app.tar.gz \
                  package.json \
                  package-lock.json \
                  server.js \
                  node_modules
                '''
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'jenkins',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                      --upload-file ${ARTIFACT} \
                      ${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${BUILD_NUMBER}/${ARTIFACT}
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh """
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                        "mkdir -p ${DEPLOY_PATH}"

                    scp -i $SSH_KEY ${ARTIFACT} \
                        ${EC2_USER}@${EC2_HOST}:${DEPLOY_PATH}

                    ssh -i $SSH_KEY ${EC2_USER}@${EC2_HOST} '
                        cd ${DEPLOY_PATH}
                        tar -xzf ${ARTIFACT}
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

