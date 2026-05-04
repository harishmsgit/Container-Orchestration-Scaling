pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = "173554685967"
        AWS_REGION = "ap-south-1"
        FRONTEND_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-ui"
        BACKEND_AUTH_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-auth"
        BACKEND_ADMIN_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-admin"
        BACKEND_CHAT_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-chat"
        BACKEND_STREAMING_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-streaming"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/harishmsgit/Container-Orchestration-Scaling.git'
            }
        }
        stage('Build Frontend') {
            steps {
                sh '''
                docker build -t streamingapp-ui:latest ./frontend
                docker tag streamingapp-ui:latest $FRONTEND_REPO:latest
                '''
            }
        }
        stage('Build Backend Auth') {
            steps {
                sh '''
                docker build -t streamingapp-auth:latest ./backend/authService
                docker tag streamingapp-auth:latest $BACKEND_AUTH_REPO:latest
                '''
            }
        }
        stage('Build Backend Admin') {
            steps {
                sh '''
                docker build -t streamingapp-admin:latest ./backend/adminService
                docker tag streamingapp-admin:latest $BACKEND_ADMIN_REPO:latest
                '''
            }
        }
        stage('Build Backend Chat') {
            steps {
                sh '''
                docker build -t streamingapp-chat:latest ./backend/chatService
                docker tag streamingapp-chat:latest $BACKEND_CHAT_REPO:latest
                '''
            }
        }
        stage('Build Backend Streaming') {
            steps {
                sh '''
                docker build -t streamingapp-streaming:latest ./backend/streamingService
                docker tag streamingapp-streaming:latest $BACKEND_STREAMING_REPO:latest
                '''
            }
        }
        stage('Push to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

                docker push $FRONTEND_REPO:latest
                aws ecr list-images --repository-name streamingapp-ui --region $AWS_REGION

                docker push $BACKEND_AUTH_REPO:latest
                aws ecr list-images --repository-name streamingapp-auth --region $AWS_REGION

                docker push $BACKEND_ADMIN_REPO:latest
                aws ecr list-images --repository-name streamingapp-admin --region $AWS_REGION

                docker push $BACKEND_CHAT_REPO:latest
                aws ecr list-images --repository-name streamingapp-chat --region $AWS_REGION

                docker push $BACKEND_STREAMING_REPO:latest
                aws ecr list-images --repository-name streamingapp-streaming --region $AWS_REGION
                '''
            }
        }
    }
    triggers {
        githubPush()
    }
}
