pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        string(name: 'REACT_APP_AUTH_API_URL',
               defaultValue: 'http://localhost:3001/api',
               description: 'Auth API URL; update for EKS deployment')
        string(name: 'REACT_APP_STREAMING_API_URL',
               defaultValue: 'http://localhost:3002/api',
               description: 'Streaming API URL; update for EKS deployment')
        string(name: 'REACT_APP_STREAMING_PUBLIC_URL',
               defaultValue: 'http://localhost:3002',
               description: 'Streaming public URL; update for EKS deployment')
        string(name: 'REACT_APP_ADMIN_API_URL',
               defaultValue: 'http://localhost:3003/api/admin',
               description: 'Admin API URL; update for EKS deployment')
        string(name: 'REACT_APP_CHAT_API_URL',
               defaultValue: 'http://localhost:3004/api/chat',
               description: 'Chat API URL; update for EKS deployment')
        string(name: 'REACT_APP_CHAT_SOCKET_URL',
               defaultValue: 'http://localhost:3004',
               description: 'Chat socket URL; update for EKS deployment')
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = '665217029177.dkr.ecr.ap-south-1.amazonaws.com'
    }

    stages {
        stage('Check Build Tools') {
            steps {
                sh '''
                    set -eu
                    git --version
                    docker --version
                    aws --version
                '''
            }
        }

        stage('Build and Push Images to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set -eu
                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        aws sts get-caller-identity --query Account --output text

                        aws ecr get-login-password --region "$AWS_REGION" |
                          docker login --username AWS --password-stdin "$ECR_REGISTRY"

                        build_push() {
                            name="$1"
                            dockerfile="$2"
                            context="$3"

                            image="$ECR_REGISTRY/$name"

                            echo "Building $name"
                            docker build \
                              -t "$image:$BUILD_NUMBER" \
                              -f "$dockerfile" \
                              "$context"

                            docker tag "$image:$BUILD_NUMBER" "$image:latest"

                            docker push "$image:$BUILD_NUMBER"
                            docker push "$image:latest"
                        }

                        build_push streamingapp-auth \
                          backend/authService/Dockerfile backend/authService

                        build_push streamingapp-streaming \
                          backend/streamingService/Dockerfile backend

                        build_push streamingapp-admin \
                          backend/adminService/Dockerfile backend

                        build_push streamingapp-chat \
                          backend/chatService/Dockerfile backend

                        image="$ECR_REGISTRY/streamingapp-frontend"

                        docker build \
                          -t "$image:$BUILD_NUMBER" \
                          --build-arg REACT_APP_AUTH_API_URL="$REACT_APP_AUTH_API_URL" \
                          --build-arg REACT_APP_STREAMING_API_URL="$REACT_APP_STREAMING_API_URL" \
                          --build-arg REACT_APP_STREAMING_PUBLIC_URL="$REACT_APP_STREAMING_PUBLIC_URL" \
                          --build-arg REACT_APP_ADMIN_API_URL="$REACT_APP_ADMIN_API_URL" \
                          --build-arg REACT_APP_CHAT_API_URL="$REACT_APP_CHAT_API_URL" \
                          --build-arg REACT_APP_CHAT_SOCKET_URL="$REACT_APP_CHAT_SOCKET_URL" \
                          -f frontend/Dockerfile frontend

                        docker tag "$image:$BUILD_NUMBER" "$image:latest"
                        docker push "$image:$BUILD_NUMBER"
                        docker push "$image:latest"

                        echo "All five images built and pushed successfully."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully.'
        }
        failure {
            echo 'CI pipeline failed. Review the stage logs.'
        }
        always {
            sh 'docker logout "$ECR_REGISTRY" || true'
        }
    }
}
