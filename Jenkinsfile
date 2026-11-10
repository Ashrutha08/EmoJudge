
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'

        WEB_IMAGE = 'emojudge-web'
        API_IMAGE = 'emojudge-api'

        WEB_CONTAINER = 'emojudge-web'
        API_CONTAINER = 'emojudge-api'

        WEB_PORT = '5000'
        API_PORT = '8000'
    }

    stages {

        stage('Git Checkout') {
            steps {
                echo 'Checking out EmoJudge source code...'
                checkout scm

                bat 'git log -1 --oneline'
            }
        }

        stage('Check Environment') {
            steps {
                bat 'whoami'
                bat '"%DOCKER%" version'
                bat 'python --version'
                bat 'python -m pip --version'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Python dependencies...'

                bat '''
                    if exist requirements.txt (
                        python -m pip install -r requirements.txt
                    ) else (
                        echo requirements.txt not found
                        exit /b 1
                    )
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo 'Running automated tests...'

                bat '''
                    python -m pytest -v
                '''
            }
        }

        stage('Build Web Image') {
            steps {
                echo 'Building EmoJudge Web Docker image...'

                bat '''
                    "%DOCKER%" build -t %WEB_IMAGE%:%BUILD_NUMBER% -t %WEB_IMAGE%:latest -f Dockerfile .
                '''
            }
        }

        stage('Build API Image') {
            steps {
                echo 'Building EmoJudge API Docker image...'

                bat '''
                    "%DOCKER%" build -t %API_IMAGE%:%BUILD_NUMBER% -t %API_IMAGE%:latest -f Dockerfile.api .
                '''
            }
        }

        stage('Verify Docker Images') {
            steps {
                bat '''
                    "%DOCKER%" images %WEB_IMAGE%
                    "%DOCKER%" images %API_IMAGE%
                '''
            }
        }

        stage('Deploy Containers') {
            steps {
                echo 'Deploying EmoJudge applications...'

                bat '''
                    "%DOCKER%" rm -f %WEB_CONTAINER% 2>NUL
                    "%DOCKER%" rm -f %API_CONTAINER% 2>NUL

                    "%DOCKER%" run -d --restart unless-stopped --name %API_CONTAINER% -p %API_PORT%:8000 %API_IMAGE%:latest

                    "%DOCKER%" run -d --restart unless-stopped --name %WEB_CONTAINER% -p %WEB_PORT%:5000 %WEB_IMAGE%:latest
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Checking deployed containers...'

                bat '''
                    "%DOCKER%" ps

                    "%DOCKER%" inspect -f "{{.State.Status}}" %WEB_CONTAINER%

                    "%DOCKER%" inspect -f "{{.State.Status}}" %API_CONTAINER%
                '''

                powershell '''
                    $urls = @(
                        "http://localhost:5000",
                        "http://localhost:8000"
                    )

                    foreach ($url in $urls) {
                        $success = $false

                        for ($i = 1; $i -le 10; $i++) {
                            try {
                                $response = Invoke-WebRequest -Uri $url -TimeoutSec 5 -UseBasicParsing

                                Write-Host "$url returned HTTP $($response.StatusCode)"

                                $success = $true
                                break
                            }
                            catch {
                                Write-Host "Waiting for $url (attempt $i)..."
                                Start-Sleep -Seconds 5
                            }
                        }

                        if (-not $success) {
                            throw "Application health check failed: $url"
                        }
                    }
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: EmoJudge CI/CD pipeline completed!'
        }

        failure {
            echo 'FAILED: EmoJudge CI/CD pipeline failed. Check Console Output.'
        }

        always {
            bat '"%DOCKER%" ps -a'
        }
    }
}
