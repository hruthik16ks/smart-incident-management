
pipeline {
    agent any

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Source code checked out successfully.'
            }
        }

        stage('Environment Check') {
            steps {
                bat 'java -version'
                bat 'node --version'
                bat 'npm --version'
            }
        }

        stage('Backend Build') {
            steps {
                dir('backend') {
                    bat 'call mvnw.cmd -B clean package -DskipTests'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    bat 'npm ci'
                    bat 'npm run build'
                }
            }
        }

        stage('Build Summary') {
            steps {
                echo 'SUCCESS: Backend and frontend builds completed without Docker.'
            }
        }
    }

    post {
        success {
            echo 'Jenkins pipeline completed successfully.'
        }
        failure {
            echo 'Build failed. Check Console Output.'
        }
    }
}
