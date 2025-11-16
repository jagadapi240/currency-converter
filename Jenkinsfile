pipeline {
    agent any

    tools {
        maven 'Maven'    // MUST match Jenkins Global Tool Config
    }

    environment {

        // Change to your SonarQube server name from Jenkins
        SONARQUBE_ENV = 'MySonarQube'

        // Nexus repository (CHANGE this if needed)
        NEXUS_RELEASE_REPO = "http://51.21.202.150:8081/repository/maven-releases/"

        // Docker image name
        DOCKER_IMAGE = "jagadapi240/currency-converter"

        // Version for Docker tags
        VERSION = "${env.BUILD_NUMBER}"
    }

    stages {

        /* ----------------------------------------------------
           1. CHECKOUT CODE FROM SCM
        ---------------------------------------------------- */
        stage('SCM Checkout') {
            steps {
                checkout scm
            }
        }

        /* ----------------------------------------------------
           2. SONARQUBE ANALYSIS
        ---------------------------------------------------- */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        mvn clean verify sonar:sonar \
                         -Dsonar.projectKey=currency-converter \
                         -Dsonar.projectName="Currency Converter"
                    '''
                }
            }
        }

        /* ----------------------------------------------------
           3. MAVEN BUILD
        ---------------------------------------------------- */
        stage('Maven Package') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }

        /* ----------------------------------------------------
           4. STAGE ARTIFACT IN NEXUS
        ---------------------------------------------------- */
        stage('Nexus Upload') {
            steps {
                sh """
                    mvn deploy -DskipTests \
                      -Dnexus.url=${NEXUS_RELEASE_REPO} \
                      -DaltDeploymentRepository=nexus::default::${NEXUS_RELEASE_REPO}
                """
            }
        }

        /* ----------------------------------------------------
           5. DOWNLOAD ARTIFACT FROM NEXUS FOR DOCKER BUILD
        ---------------------------------------------------- */
        stage('Download WAR from Nexus') {
            steps {
                sh '''
                    # Create deploy directory
                    rm -rf deploy || true
                    mkdir deploy

                    # Download the latest WAR uploaded
                    echo "Downloading WAR from Nexus..."
                    curl -u $NEXUS_USER:$NEXUS_PASS -o deploy/app.war \

