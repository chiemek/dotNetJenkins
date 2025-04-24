pipeline {
    agent any
    tools {
        // Use 'dotnetsdk' or 'dotnet' depending on plugin version; 'dotnetsdk' is typically correct
        dotnetsdk 'dotnet-8.0' // Must match the name in Global Tool Configuration
        git 'default-git'
    }
    environment {
       // Fetch credentials from Jenkins
        SONAR_TOKEN = credentials("SONAR_TOKEN")
        SNYK_TOKEN = credentials("SYNK_TOKEN")
        DOCKER_CREDENTIALS = credentials("DOCKER_CREDENTIALS")
        
        // Fetch environment variables from Jenkins global/job config
        DOCKER_IMAGE = "${env.DOCKER_IMAGE_NAME ?: 'mekus1085/dotNetJenkins'}"
        REGISTRY = "${env.DOCKER_REGISTRY ?: 'docker.io'}"
        SONAR_HOST_URL = credentials("SONAR_HOST_URL")
        SONAR_PROJECT_KEY = credentials("SONAR_PROJECT_KEY")
        DOTNET_CLI_HOME = "${WORKSPACE}\\dotnet_temp"
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
                bat 'dotnet restore'
            }
        }
        stage('Build') {
            steps {
                // Build the solution in Release mode
                bat 'dotnet build --configuration Release --no-restore'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Run SonarQube scanner
                withSonarQubeEnv('SonarQube') {
                    bat 'dotnet tool install --global dotnet-sonarscanner'
                    bat """
                        dotnet-sonarscanner begin ^
                        /k:"${SONAR_PROJECT_KEY}" ^
                        /o:'practiceCI' ^
                        /d:sonar.login="${SONAR_TOKEN}" ^
                        /d:sonar.host.url="${SONAR_HOST_URL}"
                    """
                    bat 'dotnet build'
                    bat 'dotnet-sonarscanner end /d:sonar.login="${SONAR_TOKEN}"'
                }
            }
        }

        stage('Snyk Dependency Scan') {
            steps {
                // Run Snyk to scan for vulnerabilities
                bat 'snyk auth %SNYK_TOKEN%'
                bat 'snyk test --file=practiceCI.sln --severity-threshold=high'
            }
        }

        stage('Publish') {
            steps {
                // Publish the app for deployment
                bat 'dotnet publish --configuration Release --no-build --output .\\publish'
            }
        }
        stage('Archive Artifacts') {
            steps {
                // Archive the published output
                archiveArtifacts artifacts: 'publish\\**', allowEmptyArchive: true
            }
        }

        stage('Build Docker Image') {
            steps {
                // Build Docker image
                bat 'docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                // Scan Docker image with Trivy
                bat 'trivy image --severity HIGH,CRITICAL --exit-code 1 %DOCKER_IMAGE%:%BUILD_NUMBER%'
            }
        }

        stage('Push Docker Image') {
            steps {
                // Log in to Docker Hub and push image
                powershell """
                    \$password = ConvertTo-SecureString \$env:DOCKER_CREDENTIALS_PSW -AsPlainText -Force
                    \$credential = New-Object System.Management.Automation.PSCredential (\$env:DOCKER_CREDENTIALS_USR, \$password)
                    docker login -u \$credential.UserName -p \$credential.GetNetworkCredential().Password \$env:REGISTRY
                """
                bat 'docker push %DOCKER_IMAGE%:%BUILD_NUMBER%'
                bat 'docker tag %DOCKER_IMAGE%:%BUILD_NUMBER% %DOCKER_IMAGE%:latest'
                bat 'docker push %DOCKER_IMAGE%:latest'
            }
        }

    }

    post {
        always {
            // Clean up Docker images
            bat 'docker rmi %DOCKER_IMAGE%:%BUILD_NUMBER% || exit 0'
            bat 'docker rmi %DOCKER_IMAGE%:latest || exit 0'
        }
        failure {
            // Notify on failure
            mail to: "${env.NOTIFICATION_EMAIL ?: 'mekus1085@gmail.com'}",
                 subject: "Jenkins Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check the build logs at ${env.BUILD_URL}"
        }
    }
       
      
}
