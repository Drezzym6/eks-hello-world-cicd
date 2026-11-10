pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        ECR_REPO       = '<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hello-world-app'
        CLUSTER_NAME   = 'hello-world-eks'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build --platform linux/amd64 -t $ECR_REPO:$IMAGE_TAG -t $ECR_REPO:latest ./app'
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
                    docker push $ECR_REPO:$IMAGE_TAG
                    docker push $ECR_REPO:latest
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
                    kubectl set image deployment/hello-world-app hello-world=$ECR_REPO:$IMAGE_TAG
                    kubectl rollout status deployment/hello-world-app
                '''
            }
        }
    }
}