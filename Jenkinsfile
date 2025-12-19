pipeline {
    agent any

    environment {
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        APP_NAME = 'learn-jenkins-app'
        AWS_DOCKER_REGISTRY = '481961344643.dkr.ecr.eu-west-2.amazonaws.com'
        AWS_DEFAULT_REGION = 'eu-west-2'
        AWS_ECS_CLUSTER = 'learning-jenkins-app-cluster-prod'
        AWS_ECS_SERVICE_PROD = 'learn-jenkins-app-service-prod'
        AWS_ECS_TASK_DEFINITION_PROD = 'learn-jenkins-app-task-definition-prod'
    }

    stages {
        
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Build Docker Image') {
            agent {
                docker {
                    image 'my-aws-cli'
                    reuseNode true
                    args "-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-access-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh'''
                        aws ecr get-login-password | docker login --username AWS    --password-stdin $AWS_DOCKER_REGISTRY

                        docker buildx create --use --name multiarch || true

                        docker buildx build \
                            --platform linux/amd64,linux/arm64 \
                            -t $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION \
                            --push \
                            .
                    '''
                }
            }
        }

        stage('Deploy to AWS') {
            agent {
                docker {
                    image 'my-aws-cli'
                    reuseNode true
                    args "--entrypoint=''"
                }
            }

            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-access-key', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh'''
                        aws --version
                        IMAGE_URI="$AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION"
                        echo "Deploying image: $IMAGE_URI"
                        jq \
                            --arg IMAGE_URI "$IMAGE_URI" \
                            '.containerDefinitions[0].image = $IMAGE_URI' \
                            aws/task-definition-prod.json \
                            > aws/task-definition-prod-final.json
                        LATEST_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod-final.json | jq '.taskDefinition.revision')
                        echo "Registered task definition revision: $LATEST_REVISION"
                        aws ecs update-service \
                            --cluster $AWS_ECS_CLUSTER \
                            --service $AWS_ECS_SERVICE_PROD  \
                            --task-definition $AWS_ECS_TASK_DEFINITION_PROD:$LATEST_REVISION
                        aws ecs wait services-stable \
                            --cluster $AWS_ECS_CLUSTER \
                            --services $AWS_ECS_SERVICE_PROD
                    '''
                }
            }
        }

        
    }
}
