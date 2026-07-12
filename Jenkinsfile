pipeline {
    agent { docker { image 'docker:27-cli'; args '-u 0:0 -v /var/run/docker.sock:/var/run/docker.sock' } }

    environment {
        // הגדרות קבועות
        APP_USER = 'ec2-user'
        DEPLOY_DIR = '/opt/calculator-app'
        IMAGE_NAME = "calculator-app:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        // חישוב דינמי פשוט
        IS_MASTER = "${env.BRANCH_NAME == 'master'}"
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // בניית האימג' ישירות - בלי סקריפטים מסורבלים
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }

        stage('Test') {
            steps {
                sh "docker run --rm ${IMAGE_NAME} python -m pytest --junitxml=results.xml"
            }
            post { always { junit 'results.xml' } }
        }

        stage('Push to ECR') {
            when { expression { return params.PUSH_TO_ECR ?: true } }
            steps {
                script {
                    // שימוש ב-Docker pipeline plugin אם מותקן, או פקודות פשוטות
                    sh "aws ecr get-login-password | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker tag ${IMAGE_NAME} ${ECR_URI}:${IMAGE_NAME}"
                    sh "docker push ${ECR_URI}:${IMAGE_NAME}"
                }
            }
        }

        stage('Deploy') {
            when { branch 'master' }
            steps {
                sshagent(['application-ec2-ssh']) {
                    sh """
                        ssh ${APP_USER}@${APP_HOST} "mkdir -p ${DEPLOY_DIR}"
                        scp docker-compose.yml ${APP_USER}@${APP_HOST}:${DEPLOY_DIR}/
                        ssh ${APP_USER}@${APP_HOST} "cd ${DEPLOY_DIR} && docker compose pull && docker compose up -d"
                    """
                }
            }
        }
    }
    
    post { always { cleanWs() } } // ניקוי אוטומטי של סביבת העבודה
}
