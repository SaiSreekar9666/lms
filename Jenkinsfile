pipeline {
    agent any
    environment {
        DOCKER_IMAGE_BACKEND = "gsaisreekar9666/lms-be:${env.BUILD_NUMBER}"
        DOCKER_IMAGE_FRONTEND = "gsaisreekar9666/lms-fe:${env.BUILD_NUMBER}"
        DOCKER_REGISTRY = "https://hub.docker.com/repositories/gsaisreekar9666"
        DATABASE_URL = "postgresql://postgres:lms@12345@lms-db:5432/postgres"
        SONARQUBE_URL = "http://18.208.120.137:9000/"
        SONARQUBE_TOKEN = credentials('sonarqube-token')  // Correct ID used here
        NEXUS_URL = "http://18.208.120.1374:8081"
        NEXUS_REPO = "lms-nexus"
    }
    stages {
        stage('Setup Network and Database') {
            steps {
                script {
                    // Create Docker network if not exists
                    sh 'docker network create lms-network || true'
                    // Run database container
                    sh 'docker run -dt --name lms-db --network lms-network -e POSTGRES_PASSWORD=lms@12345 postgres || true'
                }
            }
        }
        stage('SonarQube Analysis - Backend') {
            steps {
                echo 'Running SonarQube analysis on Backend'
                dir('lms/api') {
                    sh """
                    docker run --rm \
                        -e SONAR_HOST_URL=${SONARQUBE_URL} \
                        -e SONAR_TOKEN=${SONARQUBE_TOKEN} \
                        -v "$PWD:/usr/src" \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=lms-api
                    """
                }
                echo 'SonarQube analysis completed for Backend'
            }
        }
        stage('SonarQube Analysis - Frontend') {
            steps {
                echo 'Running SonarQube analysis on Frontend'
                dir('lms/webapp') {
                    sh """
                    docker run --rm \
                        -e SONAR_HOST_URL=${SONARQUBE_URL} \
                        -e SONAR_TOKEN=${SONARQUBE_TOKEN} \
                        -v "$PWD:/usr/src" \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=lms-webapp
                    """
                }
                echo 'SonarQube analysis completed for Frontend'
            }
        }
        stage('Build Docker Images - Backend') {
            steps {
                script {
                    dir('lms/api') {
                        docker.build("${DOCKER_IMAGE_BACKEND}", "-f Dockerfile .")
                    }
                }
                echo 'Docker image built for Backend'
            }
        }
        stage('Build Docker Images - Frontend') {
            steps {
                script {
                    dir('lms/webapp') {
                        docker.build("${DOCKER_IMAGE_FRONTEND}", "-f Dockerfile .")
                    }
                }
                echo 'Docker image built for Frontend'
            }
        }
        stage('Push Docker Images - Backend') {
            steps {
                script {
                    docker.withRegistry("${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_IMAGE_BACKEND}").push()
                    }
                }
                echo 'Docker image pushed for Backend'
            }
        }
        stage('Push Docker Images - Frontend') {
            steps {
                script {
                    docker.withRegistry("${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_IMAGE_FRONTEND}").push()
                    }
                }
                echo 'Docker image pushed for Frontend'
            }
        }
        stage('Release Artifacts') {
            steps {
                script {
                    echo 'Preparing artifacts for release'
                    def backendVersion = sh(script: "cd lms/api && npm version | grep version", returnStdout: true).trim()
                    def frontendVersion = sh(script: "cd lms/webapp && npm version | grep version", returnStdout: true).trim()
                    sh "zip -r lms-backend-${backendVersion}.zip lms/api"
                    sh "zip -r lms-frontend-${frontendVersion}.zip lms/webapp"
                    sh """
                    curl -u admin:admin123 \
                    --upload-file lms-backend-${backendVersion}.zip \
                    ${NEXUS_URL}/repository/${NEXUS_REPO}/lms-backend-${backendVersion}.zip
                    """
                    sh """
                    curl -u admin:admin123 \
                    --upload-file lms-frontend-${frontendVersion}.zip \
                    ${NEXUS_URL}/repository/${NEXUS_REPO}/lms-frontend-${frontendVersion}.zip
                    """
                    echo 'Artifacts released'
                }
            }
        }
        stage('Deploy Containers') {
            steps {
                script {
                    // Pull the latest images
                    sh 'docker pull ${DOCKER_IMAGE_BACKEND}'
                    sh 'docker pull ${DOCKER_IMAGE_FRONTEND}'
                    // Remove old containers if they exist
                    sh 'docker rm -f lms-backend || true'
                    sh 'docker rm -f lms-frontend || true'
                    // Run new containers
                    sh 'docker run -d --name lms-backend --network lms-network -e DATABASE_URL=${DATABASE_URL} -p 3000:3000 ${DOCKER_IMAGE_BACKEND}'
                    sh 'docker run -d --name lms-frontend --network lms-network -p 80:80 ${DOCKER_IMAGE_FRONTEND}'
                }
            }
        }
        stage('Clean Up') {
            steps {
                echo 'Cleaning Up'
                cleanWs()
            }
        }
    }
}
