pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        AWS_DEFAULT_REGION = 'us-east-2'
        AWS_ACCOUNT_ID = '373102893310'
        ECR_REPOSITORY = 'jenkins-cicd-demo'
    }

    stages {
        stage('Verify Tools') {
            steps {
                sh 'git --version'
                sh 'docker ps'
                sh 'aws --version'
            }
        }
        stage('Build and Push Image') {
            steps {
                sh '''
                    set -eu
                    REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    IMAGE_URI="${REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

                    aws ecr get-login-password --region "${AWS_DEFAULT_REGION}" \
                      | docker login --username AWS --password-stdin "${REGISTRY}"

                    docker build -t "${IMAGE_URI}" .
                    docker push "${IMAGE_URI}"
                    echo "Published ${IMAGE_URI}"
                '''
            }
        }
    }
}