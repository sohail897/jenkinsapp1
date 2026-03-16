// Jenkins pipeline - fill in after Phase 6

pipeline {
    // CRITICAL: 'ecs-slave' matches the label we set on the slave node
    // This means ALL stages run on the slave — master only orchestrates
    agent { label 'ecs-slave' }

    environment {
        // Replace these values with your own
        AWS_REGION      = 'ap-south-1'
        AWS_ACCOUNT_ID  = '066380525387'
        ECR_REPO        = 'demo-ecs-app'
        ECS_CLUSTER     = 'demo-cluster'
        ECS_SERVICE     = 'dmo-app-service-t33kcaw7'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        ECR_URI         = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image with tag: ${IMAGE_TAG}"
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Push to ECR') {
            steps {
                echo "Authenticating with ECR..."
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """

                echo "Tagging and pushing image..."
                sh """
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:latest
                    docker push ${ECR_URI}:${IMAGE_TAG}
                    docker push ${ECR_URI}:latest
                """
            }
        }

        stage('Deploy to ECS') {
            steps {
                echo "Forcing new ECS deployment..."
                sh """
                    aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --force-new-deployment \
                        --region ${AWS_REGION}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Waiting for service to stabilize..."
                sh """
                    aws ecs wait services-stable \
                        --cluster ${ECS_CLUSTER} \
                        --services ${ECS_SERVICE} \
                        --region ${AWS_REGION}
                """
                echo "Deployment successful!"
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded! New version deployed to ECS."
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
        always {
            // Clean up local Docker images to save disk space on slave
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_URI}:${IMAGE_TAG} || true"
        }
    }
}
