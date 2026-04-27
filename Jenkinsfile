pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = "173554685967"
        AWS_REGION = "ap-south-1"
        FRONTEND_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-ui"
        BACKEND_API_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-api"
        BACKEND_CHAT_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-chat"
        BACKEND_STREAMING_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-streaming"
        BACKEND_ADMIN_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-admin"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/harishmsgit/Container-Orchestration-Scaling.git'
            }
        }
        stage('Build Frontend') {
            steps {
                sh 'docker build -t $FRONTEND_REPO:latest ./frontend'
            }
        }
        stage('Build Backend API') {
            steps {
                sh 'docker build -t $BACKEND_API_REPO:latest ./backend/api'
            }
        }
        stage('Build Backend Chat') {
            steps {
                sh 'docker build -t $BACKEND_CHAT_REPO:latest ./backend/chat'
            }
        }
        stage('Build Backend Streaming') {
            steps {
                sh 'docker build -t $BACKEND_STREAMING_REPO:latest ./backend/streaming'
            }
        }
        stage('Build Backend Admin') {
            steps {
                sh 'docker build -t $BACKEND_ADMIN_REPO:latest ./backend/admin'
            }
        }
        stage('Push to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                
                docker push $FRONTEND_REPO:latest
                docker push $BACKEND_API_REPO:latest
                docker push $BACKEND_CHAT_REPO:latest
                docker push $BACKEND_STREAMING_REPO:latest
                docker push $BACKEND_ADMIN_REPO:latest
                '''
            }
        }
    }
    triggers {
        githubPush()
    }
}
