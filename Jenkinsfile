pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO   = 'ecr-repo-nitzan'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                withCredentials([string(credentialsId: 'ecr-account-id', variable: 'AWS_ACCOUNT_ID')]) {
                    script {
                        env.ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
                        docker.build("${ECR_URI}:${IMAGE_TAG}", ".")
                    }
                }
            }
        }

        stage('Test') {
            steps {
                sh "docker run --rm ${ECR_URI}:${IMAGE_TAG} sh -c 'pytest --junitxml=test-results/results.xml'"
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/test-results/*.xml'
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'ecr-account-id', variable: 'AWS_ACCOUNT_ID')
                ]) {
                    sh '''
                        set +x
                        aws ecr get-login-password --region "$AWS_REGION" | \
                        docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
                    '''
                    script {
                        docker.image("${ECR_URI}:${IMAGE_TAG}").push()
                        docker.image("${ECR_URI}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to Production EC2') {
            steps {
                withCredentials([
                    string(credentialsId: 'ecr-account-id', variable: 'AWS_ACCOUNT_ID'),
                    string(credentialsId: 'ec2-host', variable: 'EC2_HOST'),
                    sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')
                ]) {
                    sh '''
                        set +x
                        ssh -i "$SSH_KEY_FILE" -o StrictHostKeyChecking=no "${SSH_USER}@${EC2_HOST}" "
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com &&
                            docker pull ${ECR_URI}:${IMAGE_TAG} &&
                            docker stop my-app || true &&
                            docker rm my-app || true &&
                            docker run -d --name my-app -p 80:8080 --restart unless-stopped ${ECR_URI}:${IMAGE_TAG} &&
                            docker image prune -f
                        "
                    '''
                }
            }
        }

        stage('Health Verification') {
            steps {
                withCredentials([string(credentialsId: 'ec2-host', variable: 'EC2_HOST')]) {
                    script {
                        def maxRetries = 10
                        def retryInterval = 6
                        def healthy = false

                        for (int i = 0; i < maxRetries; i++) {
                            def status = sh(
                                script: 'curl -s -o /dev/null -w "%{http_code}" http://$EC2_HOST/health || true',
                                returnStdout: true
                            ).trim()

                            if (status == '200') {
                                healthy = true
                                echo "Health check passed (attempt ${i + 1})"
                                break
                            } else {
                                echo "Health check attempt ${i + 1} failed (status: ${status}), retrying in ${retryInterval}s..."
                                sleep retryInterval
                            }
                        }

                        if (!healthy) {
                            error("Health check failed after ${maxRetries} attempts.")
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed build ${IMAGE_TAG} to production successfully."
        }
        failure {
            echo "Pipeline failed. Consider rollback."
        }
        always {
            sh "docker image prune -f || true"
        }
    }
}
