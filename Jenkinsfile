pipeline {
    agent any

    tools {
        // ⬇️ Change this to the exact name of your Maven tool in Jenkins
        maven 'Maven_3_9_11'
    }

    environment {
        // ⬇️ Change to your DockerHub repo name
        DOCKER_IMAGE = 'jagadapi240/currency-converter'
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  echo "Building Docker image ${DOCKER_IMAGE}:${VERSION}"
                  docker build -t ${DOCKER_IMAGE}:${VERSION} .
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pat',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    sh '''
                      echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                      docker push ${DOCKER_IMAGE}:${VERSION}
                      docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                      docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy Container on 8082') {
            steps {
                sh '''
                  # Stop & remove old container if exists
                  docker rm -f currency-converter-app || true

                  # Run new container
                  docker run -d --name currency-converter-app \
                    -p 8082:8080 \
                    ${DOCKER_IMAGE}:${VERSION}
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

