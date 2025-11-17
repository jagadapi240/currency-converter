pipeline {
    agent any

    tools {
        // MUST match the Maven installation name in Jenkins (Manage Jenkins → Global Tool Configuration)
        maven 'Maven'
    }

    environment {
        // SonarQube installation name from Jenkins (Manage Jenkins → Configure System → SonarQube servers)
        SONARQUBE_ENV = 'MySonarQube'

        // DockerHub image name
        DOCKER_IMAGE = 'jagadapi240/currency-converter'

        // Docker tag = Jenkins build number
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

        /* 3. MAVEN BUILD (PACKAGE WAR) */
        stage('Maven Package') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }

        /* 4. UPLOAD ARTIFACT TO NEXUS */
        stage('Nexus Upload') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    sh '''
                        mvn deploy -DskipTests \
                          -DaltDeploymentRepository=nexus::default::http://$NEXUS_USER:$NEXUS_PASS@51.21.202.150:8081/repository/maven-releases/
                    '''
                }
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
                          http://51.21.202.150:8081/repository/maven-releases/com/ajacs/currency-converter-web/1.0.1/currency-converter-web-1.0.1.war
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

