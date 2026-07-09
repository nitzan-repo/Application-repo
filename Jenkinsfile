pipeline {
    agent { docker { image 'python:3.9-slim' } }
    
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = '992382545251.dkr.ecr.us-east-1.amazonaws.com/ecr-repo-nitzan'
        IMAGE_TAG = "pr-${env.CHANGE_ID ?: 'main'}-${env.BUILD_NUMBER}"
        PROD_SERVER = 'ubuntu@44.195.88.151'
    }
    
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        
        stage('Build') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }
        
        stage('Test') {
            steps {
                sh 'pip install pytest'
                sh 'pytest --junitxml=results.xml'
            }
            post {
                always { junit 'results.xml' }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding', 
                        credentialsId: '992382545251'
                    ]]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                        sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                script {
                    sshagent(['prod-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${PROD_SERVER} "
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker pull ${ECR_REPO}:${IMAGE_TAG}
                                docker stop my-calculator-app || true
                                docker rm my-calculator-app || true
                                docker run -d --name my-calculator-app -p 80:80 ${ECR_REPO}:${IMAGE_TAG}
                            "
                        """
                    }
                }
            }
        }
        
        stage('Health Verification') {
            when { branch 'main' }
            steps {
                script {
                    sh """
                        for i in {1..5}; do
                            sleep 5
                            if curl -f http://44.195.88.151/health; then
                                echo 'Health check passed!'
                                exit 0
                            fi
                            echo 'Health check failed, retrying...'
                        done
                        exit 1
                    """
                }
            }
        }
    }
}
