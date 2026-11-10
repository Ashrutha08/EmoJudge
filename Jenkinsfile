
pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        WEB_IMAGE = 'emojudge-web'
        API_IMAGE = 'emojudge-api'

        WEB_CONTAINER = 'emojudge-web'
        API_CONTAINER = 'emojudge-api'

        WEB_PORT = '5000'
        API_PORT = '8000'

        GIT_EXE = 'C:\\Program Files\\Git\\cmd\\git.exe'
        DOCKER_EXE = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'

        PYTHON = 'C:\\Users\\ANAND L\\AppData\\Local\\Programs\\Python\\Python310\\python.exe'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                bat '"%GIT_EXE%" log -1 --oneline'
            }
        }

        stage('Check Tools') {
            steps {
                bat 'whoami'
                bat '"%GIT_EXE%" --version'
                bat '"%DOCKER_EXE%" version'
                bat '"%PYTHON%" --version'
            }
        }

        stage('Install Dependencies') {
            steps {
                bat '''
                echo Checking and installing only required dependencies...

                "%PYTHON%" -m pip install ^
                    --disable-pip-version-check ^
                    --no-warn-script-location ^
                    -r requirements.txt

                if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Run Tests') {
            steps {
                bat '''
                powershell -NoProfile -ExecutionPolicy Bypass -Command "$testFiles = Get-ChildItem -Path . -Recurse -File | Where-Object { $_.Name -match '^(test_.*|.*_test)\\.py$' }; if ($testFiles) { Write-Host 'Test files found. Running pytest...'; & \"$env:PYTHON\" -m pytest -v; exit $LASTEXITCODE } else { Write-Host 'WARNING: No test files found. Skipping pytest.'; exit 0 }"
                '''
            }
        }

        stage('Build Web Image') {
            steps {
                bat '"%DOCKER_EXE%" build -t %WEB_IMAGE%:%BUILD_NUMBER% -t %WEB_IMAGE%:latest -f Dockerfile .'
            }
        }

        stage('Build API Image') {
            steps {
                bat '"%DOCKER_EXE%" build -t %API_IMAGE%:%BUILD_NUMBER% -t %API_IMAGE%:latest -f Dockerfile.api .'
            }
        }

        stage('Verify Docker Images') {
            steps {
                bat '"%DOCKER_EXE%" images %WEB_IMAGE%'
                bat '"%DOCKER_EXE%" images %API_IMAGE%'
            }
        }

        stage('Deploy Web and API') {
            steps {
                script {
                    bat(returnStatus: true, script: '"%DOCKER_EXE%" rm -f %WEB_CONTAINER%')
                    bat(returnStatus: true, script: '"%DOCKER_EXE%" rm -f %API_CONTAINER%')
                }

                bat '"%DOCKER_EXE%" run -d --restart unless-stopped --name %API_CONTAINER% -p %API_PORT%:8000 %API_IMAGE%:%BUILD_NUMBER%'

                bat '"%DOCKER_EXE%" run -d --restart unless-stopped --name %WEB_CONTAINER% -p %WEB_PORT%:5000 %WEB_IMAGE%:%BUILD_NUMBER%'
            }
        }

        stage('Verify Deployment') {
            steps {
                bat '"%DOCKER_EXE%" ps'

                powershell '''
                    $containers = @(
                        "emojudge-web",
                        "emojudge-api"
                    )

                    foreach ($container in $containers) {
                        $status = & $env:DOCKER_EXE inspect -f '{{.State.Status}}' $container

                        Write-Host "$container : $status"

                        if ($status -ne "running") {
                            throw "$container deployment failed"
                        }
                    }
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: EmoJudge CI/CD pipeline completed.'
            echo 'Both Web and API containers deployed successfully.'
        }

        failure {
            echo 'FAILED: EmoJudge CI/CD pipeline failed.'
            echo 'Check Console Output for the failed stage.'
        }

        always {
            bat '"%DOCKER_EXE%" ps -a'
        }
    }
}
