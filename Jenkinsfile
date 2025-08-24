pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        REPO_NAME  = "my-apache-app"
        ACCOUNT_ID = "147237731579" // replace with your AWS account ID
        IMAGE_TAG  = "${env.BUILD_NUMBER}" // Jenkins build number
    }
    
    stages {
        stage('build') {
            steps {
                sh "docker build -t ${REPO_NAME}:${env.BUILD_NUMBER} ."
            }
        }


        stage('Push to ECR') {
            steps {
                sh """
                echo "Pushing Docker image to ECR..."
                docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/${REPO_NAME}:${IMAGE_TAG}
                """
            }
        }

        
        stage('deploy') {
            steps {
                sh "docker run -d -p 808${env.BUILD_NUMBER}:80 ${REPO_NAME}:${env.BUILD_NUMBER}"
            }
        }
    }
}
