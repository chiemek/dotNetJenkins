pipeline {
    agent any // Runs on any agent; ensure .NET SDK is available
    tools {
        dotnet 'dotnet-sdk-8.0' // Must match Global Tool Configuration
    }
    stages {
        stage('Check SDK') {
            steps {
                sh 'dotnet --version' // Debug SDK version
            }
        }
        stage('Checkout') {
            steps {
                git url: 'https://github.com/chiemek/dotNetJenkins.git', branch: 'master'
                // If private repo, add: credentialsId: 'your-credential-id'
            }
        }
        stage('Restore') {
            steps {
                sh 'dotnet restore'
            }
        }
        stage('Build') {
            steps {
                sh 'dotnet build --configuration Release --no-restore'
            }
        }
        stage('Test') {
            when {
                expression { fileExists('**/*Tests.csproj') } // Run only if test projects exist
            }
            steps {
                sh 'dotnet test --no-build --configuration Release'
            }
        }
        stage('Publish') {
            steps {
                sh 'dotnet publish --configuration Release --output ./publish'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'publish/**', allowEmptyArchive: true
        }
        failure {
            echo 'Build failed! Check console output.'
        }
        success {
            echo 'Build succeeded!'
        }
    }
}