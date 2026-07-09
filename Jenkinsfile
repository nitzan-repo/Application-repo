
pipeline {
    agent any
    
    triggers {
        githubPush()
    }
    
    stages {
        stage('Checkout') {
            steps {
                
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    
                    sh 'docker build -t my-calculator-app .'
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    
                    docker.withRegistry('https://<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com', 'aws-ecr-creds') {
                        sh 'docker tag my-calculator-app:latest <YOUR_REPO_URI>:latest'
                        sh 'docker push <YOUR_REPO_URI>:latest'
                    }
                }
            }
        }
    }
}
