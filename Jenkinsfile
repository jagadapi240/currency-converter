pipeline {
    agent any

    tools {
        // MUST match Maven name in "Manage Jenkins → Global Tool Configuration"
        maven 'Maven'
    }

    environment {
        // SonarQube installation name from Jenkins (Manage Jenkins → Configure System → SonarQube)
        SONARQUBE_ENV = 'MySonarQube'

        // Nexus URL (releases repo)
        NEXUS_RELEASE_REPO = 'http://51.21.202.150:8081/repository/maven-releases'

        // DockerHub image
        DOCKER_IMAGE = 'jagadapi240/currency-converter'

        // Docker image version
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        /* 1. CHECKOUT SOURCE CODE */
        stage('SCM Checkout') {
            steps {
                checkout scm
            }
        }

        /* 2. SONARQUBE ANALYSIS */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(env.SONARQUBE_ENV) {
                    sh '''
                        mvn clean verify sonar:sonar \
                          -Dsonar.projectKey=currency-converter \
                          -Dsonar.projectName="Currency Converter"
                    '''
                }
            }
        }

        /* 3. MAVEN BUILD */
        stage('Maven Package') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }

        /* 4. DEPLOY ARTIFACT TO NEXUS */
        stage('Nexus Upload') {
            steps {
                sh '''
                    mvn deploy -DskipTests \
                      -DaltDeploymentRepository=nexus::default::http://51.21.202.150:8081/repository/maven-releases/
                '''
            }
        }

        /* 5. DOWNLOAD WAR FROM NEXUS FOR DOCKER BUILD */
        stage('Download WAR from Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    sh '''
                        rm -rf deploy || true
                        mkdir deploy

                        echo "Downloading WAR from Nexus..."
                        curl -f -u $NEXUS_USER:$NEXUS_PASS \
                          -o deploy/app.war \
                          http://51.21.202.150:8081/repository/maven-releases/com/ajacs/currency-converter-web/1.0-SNAPSHOT/currency-converter-web-1.0-SNAPSHOT.war
                    '''
                }
            }
        }

        /* 6. BUILD DOCKER IMAGE (TOMCAT + WAR) */
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image ${DOCKER_IMAGE}:${VERSION}"
                    docker build -t ${DOCKER_IMAGE}:${VERSION} .
                '''
            }
        }

        /* 7. PUSH IMAGE TO DOCKERHUB */
        stage('Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-pat',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        /* 8. RUN CONTAINER (DEPLOY ON TOMCAT) */
        stage('Deploy on Tomcat Docker') {
            steps {
                sh '''
                    docker rm -f currency-converter-app || true

                    docker run -d \
                      --name currency-converter-app \
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

