pipeline {
    agent any
    tools {
        // Use 'dotnetsdk' or 'dotnet' depending on plugin version; 'dotnetsdk' is typically correct
        dotnetsdk 'dotnet-8.0' // Must match the name in Global Tool Configuration
    }
    environment {
        // Set temporary directory for .NET CLI to avoid permission issues
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
        stage('Test') {
            steps {
                // Run unit tests
                bat 'dotnet test --no-build --configuration Release --logger "trx;LogFileName=test-results.trx"'
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
    }
    post {
        always {
            // Publish test results (requires MSTest or JUnit plugin)
            junit '**/test-results.trx'
        }
        success {
            echo 'Build and tests succeeded!'
        }
        failure {
            echo 'Build or tests failed.'
        }
    }
}