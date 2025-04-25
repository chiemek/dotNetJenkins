pipeline {
    agent any
    tools {
        dotnetsdk 'dotnet-8.0' // Must match the name in Global Tool Configuration
        git 'default-git'
    }
    environment {
        // Fetch credentials from Jenkins
        SONAR_TOKEN = credentials('SONAR_TOKEN')
        SNYK_TOKEN = credentials('SNYK_TOKEN')
        DOCKER_CREDENTIALS = credentials('DOCKER_CREDENTIALS')

        // Fetch environment variables from Jenkins global/job config
        DOCKER_IMAGE = "mekus1085/dotNetJenkins"
        REGISTRY = "${env.DOCKER_REGISTRY ?: 'docker.io'}"
        SONAR_HOST_URL = credentials('SONAR_HOST_URL')
        SONAR_PROJECT_KEY = credentials('SONAR_PROJECT_KEY')
        DOTNET_CLI_HOME = "${WORKSPACE}/dotnet_temp"
    }
    stages {
        stage('Checkout') {
            steps {
                // Pull code from your Git repository
                git branch: 'master', url: 'https://github.com/chiemek/dotNetJenkins.git'
            }
        }
        stage('Restore') {
            steps {
                // Restore NuGet packages
                sh 'dotnet restore'
            }
        }
        stage('Build') {
            steps {
                // Build the solution in Release mode
                sh 'dotnet build --configuration Release --no-restore'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                // Run SonarQube scanner
                withSonarQubeEnv('SonarQube') {
                    sh 'dotnet tool install --global dotnet-sonarscanner'
                    sh """
                        dotnet-sonarscanner begin \
                        /k:"${SONAR_PROJECT_KEY}" \
                        /o:"dotnetjenkins" \
                        /d:sonar.login="${SONAR_TOKEN}" \
                        /d:sonar.host.url="${SONAR_HOST_URL}"
                    """
                    sh 'dotnet build'
                    sh 'dotnet-sonarscanner end /d:sonar.login="${SONAR_TOKEN}"'
                }
            }
        }
        stage('Snyk Dependency Scan') {
            steps {
                // Run Snyk to scan for vulnerabilities
                sh 'snyk auth $SNYK_TOKEN'
                sh 'snyk test --file=practiceCI.sln --severity-threshold=high'
            }
        }
        stage('Publish') {
            steps {
                // Publish the app for deployment
                sh 'dotnet publish --configuration Release --no-build --output ./publish'
            }
        }
        stage('Archive Artifacts') {
            steps {
                // Archive the published output
                archiveArtifacts artifacts: 'publish/**', allowEmptyArchive: true
            }
        }
        stage('Build Docker Image') {
            steps {
                // Build Docker image
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
            }
        }
        stage('Trivy Image Scan') {
            steps {
                // Scan Docker image with Trivy
                sh "trivy image --severity HIGH,CRITICAL --exit-code 1 ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            }
        }
        stage('Push Docker Image') {
            steps {
                // Log in to Docker Hub and push image
                sh """
                    echo \$DOCKER_CREDENTIALS_PSW | docker login -u \$DOCKER_CREDENTIALS_USR --password-stdin \$REGISTRY
                """
                sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                sh "docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest"
                sh "docker push ${DOCKER_IMAGE}:latest"
            }
        }
    }
    post {
        always {
            // Clean up Docker images
            sh "docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true"
            sh "docker rmi ${DOCKER_IMAGE}:latest || true"
        }
        failure {
            // Notify on failure
            echo "pipeline Failed"
        }
    }
}
