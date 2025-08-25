pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        REPO_NAME      = "my-apache-app"
        ACCOUNT_ID     = "147237731579" // replace with your AWS account ID
        IMAGE_TAG      = "${env.BUILD_NUMBER}" // Jenkins build number
        CLUSTER_NAME   = "my-ecs-cluster" // ECS cluster name
        SERVICE_NAME   = "my-ecs-service" // ECS service name
        TASK_FAMILY    = "my-ecs-taskdef" // ECS task definition family
        CONTAINER_NAME = "my-apache-container" // must match container name in task definition
        SUBNET_1       = "subnet-093812277454ebbea" // replace with your subnet IDs
        SUBNET_2       = "subnet-00f123192c9ab3108"
        SECURITY_GROUP = "sg-07144bd7e3fdbb654" // replace with your ECS service security group
    }
    
    stages {
        stage('Ensure Prerequisites') {
            steps {
                sh '''
                set -e

                echo "==> Ensuring ECR repository exists....."
                aws ecr describe-repositories --repository-names ${REPO_NAME} --region ${AWS_REGION} > /dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${REPO_NAME} --region ${AWS_REGION}

                echo "==> Ensuring IAM role ecsTaskExecutionRole exists..."
                if ! aws iam get-role --role-name ecsTaskExecutionRole >/dev/null 2>&1; then
                  aws iam create-role \
                    --role-name ecsTaskExecutionRole \
                    --assume-role-policy-document '{
                      "Version": "2012-10-17",
                      "Statement": [
                        {
                          "Effect": "Allow",
                          "Principal": { "Service": "ecs-tasks.amazonaws.com" },
                          "Action": "sts:AssumeRole"
                        }
                      ]
                    }'
                  aws iam attach-role-policy \
                    --role-name ecsTaskExecutionRole \
                    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
                fi

                echo "==> Ensuring ECS cluster exists..."
                aws ecs describe-clusters --clusters ${CLUSTER_NAME} --region ${AWS_REGION} | grep ${CLUSTER_NAME} >/dev/null 2>&1 || \
                aws ecs create-cluster --cluster-name ${CLUSTER_NAME} --region ${AWS_REGION}
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${REPO_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${REPO_NAME}:${IMAGE_TAG} ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                sh "docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Register Task Definition') {
            steps {
                sh '''
                echo "==> Registering new ECS task definition..."
                NEW_TASK_DEF=$(jq -n --arg IMAGE "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}" '{
                    family: "${TASK_FAMILY}",
                    networkMode: "awsvpc",
                    requiresCompatibilities: ["FARGATE"],
                    cpu: "256",
                    memory: "512",
                    executionRoleArn: "arn:aws:iam::\${ACCOUNT_ID}:role/ecsTaskExecutionRole",
                    containerDefinitions: [
                        {
                            name: "${CONTAINER_NAME}",
                            image: $IMAGE,
                            essential: true,
                            portMappings: [
                              { containerPort: 80, protocol: "tcp" }
                            ]
                        }
                    ]
                }')

                echo "$NEW_TASK_DEF" > taskdef.json
                aws ecs register-task-definition \
                    --cli-input-json file://taskdef.json \
                    --region ${AWS_REGION}
                '''
            }
        }

        stage('Deploy to ECS Fargate') {
            steps {
                sh '''
                echo "==> Ensuring ECS service exists..."
                if ! aws ecs describe-services --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME} --region ${AWS_REGION} | grep ${SERVICE_NAME} >/dev/null 2>&1; then
                  aws ecs create-service \
                    --cluster ${CLUSTER_NAME} \
                    --service-name ${SERVICE_NAME} \
                    --task-definition ${TASK_FAMILY} \
                    --desired-count 1 \
                    --launch-type FARGATE \
                    --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_1},${SUBNET_2}],securityGroups=[${SECURITY_GROUP}],assignPublicIp=ENABLED}" \
                    --region ${AWS_REGION}
                else
                  echo "==> Updating ECS service with new task definition..."
                  aws ecs update-service \
                    --cluster ${CLUSTER_NAME} \
                    --service ${SERVICE_NAME} \
                    --task-definition ${TASK_FAMILY} \
                    --force-new-deployment \
                    --region ${AWS_REGION}
                fi
                '''
            }
        }
    }
}
