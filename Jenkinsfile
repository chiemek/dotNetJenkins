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
                // Debug and install dotnet-sonarscanner
                sh '''
                    echo "DOTNET_CLI_HOME is $DOTNET_CLI_HOME"
                    echo "HOME is $HOME"
                    echo "Checking .NET SDK version..."
                    dotnet --version
                    echo "Attempting global installation of dotnet-sonarscanner..."
                    unset DOTNET_CLI_HOME  # Temporarily unset to use default global path
                    dotnet tool install --global dotnet-sonarscanner || echo "Global install failed"
                    export PATH="$PATH:$HOME/.dotnet/tools"
                    # Verify tool installation
                    ls -l $HOME/.dotnet/tools || echo "Directory $HOME/.dotnet/tools not found"
                    dotnet-sonarscanner --version || echo "dotnet-sonarscanner not found in $HOME/.dotnet/tools"
                    # Fallback to local installation if global fails
                    if ! command -v dotnet-sonarscanner >/dev/null 2>&1; then
                        echo "Falling back to local installation..."
                        dotnet tool install dotnet-sonarscanner --tool-path ./sonarscanner
                        export PATH="$PATH:$PWD/sonarscanner"
                        ls -l ./sonarscanner
                        ./sonarscanner/dotnet-sonarscanner --version || echo "Local install failed"
                    fi
                '''
                // Run SonarQube analysis with secure variable handling
                withCredentials([
                    string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN'),
                    string(credentialsId: 'SONAR_HOST_URL', variable: 'SONAR_HOST_URL'),
                    string(credentialsId: 'SONAR_PROJECT_KEY', variable: 'SONAR_PROJECT_KEY')
                ]) {
                sh '''
                    export PATH="$PATH:$HOME/.dotnet/tools:$PWD/sonarscanner"
                    dotnet-sonarscanner begin \
                    /k:"$SONAR_PROJECT_KEY" \
                    /o:"dotnetjenkins" \
                    /d:sonar.login="$SONAR_TOKEN" \
                    /d:sonar.host.url="$SONAR_HOST_URL"
                '''
                sh 'dotnet build'
                sh '''
                    export PATH="$PATH:$HOME/.dotnet/tools:$PWD/sonarscanner"
                    dotnet-sonarscanner end /d:sonar.login="$SONAR_TOKEN"
                '''
                    }
                }
            }
        }
        
        stage('Snyk Dependency Scan') {
            steps {
                // Run Snyk to scan for vulnerabilities
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh '''
                        echo "Checking for snyk CLI..."
                        command -v snyk >/dev/null 2>&1 && snyk --version || echo "snyk not found"
                        # Install snyk if not found
                    if ! command -v snyk >/dev/null 2>&1; then
                        echo "Installing snyk CLI..."
                        npm install -g snyk || echo "Failed to install snyk"
                        export PATH="$PATH:$HOME/.npm-global/bin"
                        # Verify installation
                        ls -l $HOME/.npm-global/bin || echo "Directory $HOME/.npm-global/bin not found"
                        snyk --version || echo "snyk still not found after installation"
                    fi
                    # Run snyk authentication and scan
                    snyk auth $SNYK_TOKEN
                    snyk test --file=practiceCI.sln --severity-threshold=high
                    '''
                }
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
