
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        JAVA_HOME = 'C:\\Program Files\\Java\\jdk-21.0.12'
        PATH = "${JAVA_HOME};${JAVA_HOME}\\bin;${env.PATH}"

        BACKEND_IMAGE = 'hruthik16ks/smartims-backend'
        FRONTEND_IMAGE = 'hruthik16ks/smartims-frontend'
        IMAGE_TAG = 'v1'
        BACKEND_HEALTH_URL = 'http://localhost:8081/actuator/health'
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
                bat '''
                    echo Checking Java...
                    java -version
                    if not exist "backend\\mvnw.cmd" exit /b 1
                    echo Checking Docker...
                    docker --version
                    docker compose version
                '''
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
                    bat '''
                        if not exist "package.json" exit /b 1
                        echo Frontend source files found.
                    '''
                }
            }
        }

        stage('Backend Docker Build') {
            steps {
                bat '''
                    docker build -t %BACKEND_IMAGE%:%IMAGE_TAG% -f backend\\Dockerfile backend
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Frontend Docker Build') {
            steps {
                bat '''
                    docker build -t %FRONTEND_IMAGE%:%IMAGE_TAG% -f frontend\\Dockerfile frontend
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Docker Environment') {
            steps {
                bat 'docker info'
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-pat-v2',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_PASSWORD% | docker login -u "%DOCKER_USER%" --password-stdin
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                bat '''
                    docker push %BACKEND_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1

                    docker push %FRONTEND_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Validate Compose Configuration') {
            steps {
                bat '''
                    if not exist "docker-compose.yml" exit /b 1
                    docker compose config -q
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Deploy') {
            steps {
                bat '''
                    docker compose up -d
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Verify Containers') {
            steps {
                bat 'docker compose ps'
            }
        }

        stage('Health Check') {
            steps {
                script {
                    int attempts = 12
                    boolean healthy = false

                    for (int i = 1; i <= attempts; i++) {
                        int result = bat(
                            script: '''
                                @echo off
                                powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; try { $r = Invoke-WebRequest -Uri $env:BACKEND_HEALTH_URL -UseBasicParsing -TimeoutSec 5; if ($r.StatusCode -eq 200) { exit 0 } else { exit 1 } } catch { exit 1 }"
                            ''',
                            returnStatus: true
                        )

                        if (result == 0) {
                            healthy = true
                            echo 'Backend health check passed.'
                            break
                        }

                        echo "Backend not ready. Attempt ${i}/${attempts}."
                        if (i < attempts) {
                            sleep time: 10, unit: 'SECONDS'
                        }
                    }

                    if (!healthy) {
                        bat 'docker compose logs --tail=100 backend'
                        error('Backend health check failed.')
                    }
                }
            }
        }

        stage('Deployment Summary') {
            steps {
                echo 'Smart Incident Management pipeline completed.'
                echo 'SonarQube analysis is disabled.'
                echo 'Backend image: hruthik16ks/smartims-backend:v1'
                echo 'Frontend image: hruthik16ks/smartims-frontend:v1'
                echo 'Backend: http://localhost:8081'
                echo 'Frontend: http://localhost:5173'
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Smart Incident Management pipeline completed.'
        }

        failure {
            echo 'FAILURE: Check Console Output for the failed stage.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
