pipeline {
    agent { docker { image 'python:3.9-slim' } }
    
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = '123456789012.dkr.ecr.us-east-1.amazonaws.com/my-calculator'
        IMAGE_TAG = "pr-${env.CHANGE_ID ?: 'main'}-${env.BUILD_NUMBER}"
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
    }
}
