pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(
            logRotator(
                numToKeepStr: '10',
                artifactNumToKeepStr: '5'
            )
        )
    }

    environment {
        JAVA_HOME = 'C:\\Program Files\\Java\\jdk-21.0.11'
        PATH = "${JAVA_HOME}\\bin;${env.PATH}"

        BACKEND_IMAGE = 'akshaygouda646/smartims-backend'
        FRONTEND_IMAGE = 'akshaygouda646/smartims-frontend'
        IMAGE_TAG = 'v1'

        BACKEND_HEALTH_URL = 'http://localhost:8081/actuator/health'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Environment Check') {
            steps {
                bat '''
                    @echo off

                    echo ===== Java =====
                    java -version
                    if errorlevel 1 exit /b 1

                    echo.
                    echo ===== Git =====
                    git --version
                    if errorlevel 1 exit /b 1

                    echo.
                    echo ===== Docker =====
                    docker --version
                    if errorlevel 1 exit /b 1

                    echo.
                    echo ===== Docker Compose =====
                    docker compose version
                    if errorlevel 1 exit /b 1

                    echo.
                    echo ===== Node.js =====
                    node --version
                    if errorlevel 1 exit /b 1

                    echo.
                    echo ===== npm =====
                    npm --version
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Backend Build') {
            steps {
                dir('backend') {
                    bat '''
                        @echo off

                        call mvnw.cmd clean package -DskipTests

                        if errorlevel 1 (
                            echo Backend Maven build failed.
                            exit /b 1
                        )

                        echo Backend Maven build completed successfully.
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('Local SonarQube') {
                        bat '''
                            @echo off

                            call mvnw.cmd sonar:sonar ^
                              -Dsonar.projectKey=smart-incident-management ^
                              "-Dsonar.projectName=Smart Incident Management Backend"

                            if errorlevel 1 (
                                echo SonarQube analysis failed.
                                exit /b 1
                            )

                            echo SonarQube analysis completed successfully.
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    bat '''
                        @echo off

                        call npm ci

                        if errorlevel 1 (
                            echo npm ci failed.
                            exit /b 1
                        )

                        call npm run build

                        if errorlevel 1 (
                            echo Frontend build failed.
                            exit /b 1
                        )

                        echo Frontend build completed successfully.
                    '''
                }
            }
        }

        stage('Backend Docker Build') {
            steps {
                bat '''
                    @echo off

                    docker build ^
                      --pull ^
                      -t %BACKEND_IMAGE%:%IMAGE_TAG% ^
                      backend

                    if errorlevel 1 (
                        echo Backend Docker image build failed.
                        exit /b 1
                    )

                    echo Backend Docker image built successfully.
                '''
            }
        }

        stage('Frontend Docker Build') {
            steps {
                bat '''
                    @echo off

                    docker build ^
                      --pull ^
                      --build-arg VITE_API_URL=http://localhost:8081 ^
                      -t %FRONTEND_IMAGE%:%IMAGE_TAG% ^
                      frontend

                    if errorlevel 1 (
                        echo Frontend Docker image build failed.
                        exit /b 1
                    )

                    echo Frontend Docker image built successfully.
                '''
            }
        }

        stage('Docker Environment') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "===== Docker Environment ====="

                    Write-Host "Current Windows account:"
                    whoami

                    Write-Host ""

                    Write-Host "Docker executable:"
                    Get-Command docker

                    Write-Host ""

                    Write-Host "Windows user profile:"
                    Write-Host $env:USERPROFILE

                    Write-Host ""

                    Write-Host "Docker configuration path:"
                    Write-Host $env:DOCKER_CONFIG

                    Write-Host ""

                    Write-Host "Docker contexts:"
                    docker context ls

                    Write-Host ""

                    docker version

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker is not available to the Jenkins service."
                    }
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-pat-v2',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        Write-Host "===== Docker Hub Login ====="

                        if ([string]::IsNullOrWhiteSpace($env:DOCKER_USERNAME)) {
                            throw "Docker Hub username is empty."
                        }

                        if ([string]::IsNullOrWhiteSpace($env:DOCKER_PASSWORD)) {
                            throw "Docker Hub PAT is empty."
                        }

                        Write-Host "Docker Hub username: $env:DOCKER_USERNAME"
                        Write-Host "Docker Hub PAT length: $($env:DOCKER_PASSWORD.Length)"

                        $tempFile = Join-Path `
                            $env:TEMP `
                            "docker-pat-$PID.txt"

                        try {
                            [System.IO.File]::WriteAllText(
                                $tempFile,
                                $env:DOCKER_PASSWORD,
                                [System.Text.Encoding]::ASCII
                            )

                            docker logout 2>$null

                            cmd.exe /D /S /C `
                                "type `"$tempFile`" | docker login --username $env:DOCKER_USERNAME --password-stdin"

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker Hub login failed."
                            }

                            Write-Host "Docker Hub login successful."
                        }
                        finally {
                            if (Test-Path $tempFile) {
                                Remove-Item $tempFile -Force
                            }
                        }
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                bat '''
                    @echo off

                    echo Pushing backend image...
                    docker push %BACKEND_IMAGE%:%IMAGE_TAG%

                    if errorlevel 1 (
                        echo Backend Docker image push failed.
                        exit /b 1
                    )

                    echo.
                    echo Pushing frontend image...
                    docker push %FRONTEND_IMAGE%:%IMAGE_TAG%

                    if errorlevel 1 (
                        echo Frontend Docker image push failed.
                        exit /b 1
                    )

                    echo Both Docker images pushed successfully.
                '''
            }
        }

        stage('Validate Compose Configuration') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'smartims-env-properties',
                        variable: 'SMARTIMS_ENV_FILE'
                    )
                ]) {
                    bat '''
                        @echo off

                        echo Validating Docker Compose configuration...

                        docker compose ^
                          --env-file "%SMARTIMS_ENV_FILE%" ^
                          config --quiet

                        if errorlevel 1 (
                            echo Docker Compose configuration validation failed.
                            exit /b 1
                        )

                        echo Docker Compose configuration is valid.
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'smartims-env-properties',
                        variable: 'SMARTIMS_ENV_FILE'
                    )
                ]) {
                    bat '''
                        @echo off

                        echo Pulling Docker images...

                        docker compose ^
                          --env-file "%SMARTIMS_ENV_FILE%" ^
                          pull

                        if errorlevel 1 (
                            echo Docker Compose pull failed.
                            exit /b 1
                        )

                        echo.
                        echo Deploying containers...

                        docker compose ^
                          --env-file "%SMARTIMS_ENV_FILE%" ^
                          up -d ^
                          --force-recreate ^
                          --remove-orphans

                        if errorlevel 1 (
                            echo Docker Compose deployment failed.
                            exit /b 1
                        )

                        echo.
                        echo Container status:

                        docker compose ^
                          --env-file "%SMARTIMS_ENV_FILE%" ^
                          ps

                        if errorlevel 1 (
                            echo Docker Compose status check failed.
                            exit /b 1
                        )

                        echo Deployment command completed successfully.
                    '''
                }
            }
        }

        stage('Verify Backend Variables') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "Checking backend JWT environment variables..."

                    $jwtExpiration = docker exec `
                        smartims-backend `
                        printenv JWT_EXPIRATION

                    if ($LASTEXITCODE -ne 0) {
                        throw "Unable to read JWT_EXPIRATION from backend container."
                    }

                    if ([string]::IsNullOrWhiteSpace($jwtExpiration)) {
                        throw "JWT_EXPIRATION is empty inside the backend container."
                    }

                    Write-Host "JWT_EXPIRATION is configured."

                    $jwtSecret = docker exec `
                        smartims-backend `
                        printenv JWT_SECRET

                    if ($LASTEXITCODE -ne 0) {
                        throw "Unable to read JWT_SECRET from backend container."
                    }

                    if ([string]::IsNullOrWhiteSpace($jwtSecret)) {
                        throw "JWT_SECRET is empty inside the backend container."
                    }

                    Write-Host "JWT_SECRET is configured."
                '''
            }
        }

        stage('Health Check') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'smartims-env-properties',
                        variable: 'SMARTIMS_ENV_FILE'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $healthUrl = $env:BACKEND_HEALTH_URL
                        $maxAttempts = 18
                        $waitSeconds = 10
                        $backendHealthy = $false

                        Write-Host "Waiting for backend to become healthy..."
                        Write-Host "Health URL: $healthUrl"

                        for ($attempt = 1; $attempt -le $maxAttempts; $attempt++) {

                            Write-Host ""
                            Write-Host "Health check attempt $attempt of $maxAttempts"

                            try {
                                $response = Invoke-RestMethod `
                                    -Uri $healthUrl `
                                    -Method Get `
                                    -TimeoutSec 10

                                Write-Host "Backend status: $($response.status)"

                                if ($response.status -eq "UP") {
                                    $backendHealthy = $true
                                    break
                                }
                            }
                            catch {
                                Write-Host "Backend is not ready yet."
                                Write-Host $_.Exception.Message
                            }

                            if ($attempt -lt $maxAttempts) {
                                Write-Host "Waiting $waitSeconds seconds..."
                                Start-Sleep -Seconds $waitSeconds
                            }
                        }

                        if (-not $backendHealthy) {
                            Write-Host ""
                            Write-Host "Backend did not become healthy."

                            docker compose `
                                --env-file "$env:SMARTIMS_ENV_FILE" `
                                ps

                            docker compose `
                                --env-file "$env:SMARTIMS_ENV_FILE" `
                                logs `
                                --tail=150 `
                                backend

                            throw "Backend health check failed."
                        }

                        Write-Host ""
                        Write-Host "Backend health check passed."
                    '''
                }
            }
        }

        stage('Deployment Summary') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'smartims-env-properties',
                        variable: 'SMARTIMS_ENV_FILE'
                    )
                ]) {
                    bat '''
                        @echo off

                        echo ===== Deployment Status =====

                        docker compose ^
                          --env-file "%SMARTIMS_ENV_FILE%" ^
                          ps

                        if errorlevel 1 (
                            echo Failed to display deployment status.
                            exit /b 1
                        )
                    '''
                }
            }
        }
    }

    post {
        always {
            powershell '''
                Write-Host "Logging out from Docker Hub..."
                docker logout 2>$null
                exit 0
            '''
        }

        success {
            echo '================================================'
            echo 'Smart Incident Management deployment successful'
            echo '================================================'
            echo 'Frontend:   http://localhost:5173'
            echo 'Backend:    http://localhost:8081'
            echo 'Health:     http://localhost:8081/actuator/health'
            echo 'SonarQube:  http://localhost:9000'
            echo 'Prometheus: http://localhost:9092'
            echo 'Grafana:    http://localhost:3002'
        }

        failure {
            echo 'Smart Incident Management pipeline failed.'
            echo 'Open the failed Jenkins stage and check its Console Output.'
        }
    }
}
