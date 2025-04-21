pipeline {
    agent any
    tools {
        // Specify the .NET SDK version installed on the Jenkins agent
        dotnet 'dotnet-sdk-8.0' // Adjust to your .NET version (e.g., 'dotnet-sdk-6.0')
    }
    stages {
        stage('Checkout') {
            steps {
                // Clone the repository
                git url: 'https://github.com/your-repo/your-dotnet-app.git', branch: 'main'
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
                // Build the application in Release configuration
                sh 'dotnet build --configuration Release --no-restore'
            }
        }
        stage('Test') {
            steps {
                // Run tests (if the project includes a test project)
                sh 'dotnet test --no-build --configuration Release'
            }
        }
        stage('Publish') {
            steps {
                // Publish the application (e.g., for deployment)
                sh 'dotnet publish --configuration Release --output ./publish'
            }
        }
    }
    post {
        always {
            // Archive build artifacts (e.g., published files)
            archiveArtifacts artifacts: 'publish/**', allowEmptyArchive: true
        }
        failure {
            // Notify on failure (optional: configure email or Slack notifications)
            echo 'Build failed!'
        }
        success {
            echo 'Build succeeded!'
        }
    }
}