
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

        K8S_NAMESPACE = 'default'

        GIT_EXE = 'C:\\Program Files\\Git\\cmd\\git.exe'
        DOCKER_EXE = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'

        PYTHON = 'C:\\Users\\ANAND L\\AppData\\Local\\Programs\\Python\\Python310\\python.exe'
    }

    stages {

        // 1. Checkout source code
        stage('Checkout') {
            steps {
                checkout scm

                bat '"%GIT_EXE%" log -1 --oneline'
            }
        }

        // 2. Check required tools
        stage('Check Tools') {
            steps {
                bat 'whoami'

                bat '"%GIT_EXE%" --version'

                bat '"%DOCKER_EXE%" version'

                bat '"%PYTHON%" --version'

                bat 'kubectl version --client'

                bat 'kubectl config current-context'

                bat 'kubectl cluster-info'
            }
        }

        // 3. Install only required dependencies
        stage('Install Dependencies') {
            steps {
                bat '''
                echo Checking and installing required dependencies...

                "%PYTHON%" -m pip install ^
                    --disable-pip-version-check ^
                    --no-warn-script-location ^
                    -r requirements.txt

                if errorlevel 1 exit /b 1
                '''
            }
        }

        // 4. Run tests if available
        stage('Run Tests') {
            steps {
                bat '''
                powershell -NoProfile -ExecutionPolicy Bypass -Command "$testFiles = Get-ChildItem -Path . -Recurse -File | Where-Object { $_.Name -match '^(test_.*|.*_test)\\.py$' }; if ($testFiles) { Write-Host 'Test files found. Running pytest...'; & \"$env:PYTHON\" -m pytest -v; exit $LASTEXITCODE } else { Write-Host 'WARNING: No test files found. Skipping pytest.'; exit 0 }"
                '''
            }
        }

        // 5. Build Web Docker image
        stage('Build Web Image') {
            steps {
                bat '''
                "%DOCKER_EXE%" build ^
                    -t %WEB_IMAGE%:%BUILD_NUMBER% ^
                    -t %WEB_IMAGE%:latest ^
                    -f Dockerfile.web .
                '''
            }
        }

        // 6. Build API Docker image
        stage('Build API Image') {
            steps {
                bat '''
                "%DOCKER_EXE%" build ^
                    -t %API_IMAGE%:%BUILD_NUMBER% ^
                    -t %API_IMAGE%:latest ^
                    -f Dockerfile.api .
                '''
            }
        }

        // 7. Verify Docker images
        stage('Verify Docker Images') {
            steps {
                bat '"%DOCKER_EXE%" images %WEB_IMAGE%'

                bat '"%DOCKER_EXE%" images %API_IMAGE%'
            }
        }

        // 8. Deploy application to Kubernetes
        stage('Deploy to Kubernetes') {
            steps {
                echo 'Starting EmoJudge Kubernetes deployment...'

                // Apply Persistent Volume and Claims
                bat 'kubectl apply -f k8s/persistent-volume.yaml -n %K8S_NAMESPACE%'

                // Apply Web and API deployments
                bat 'kubectl apply -f k8s/deployment.yaml -n %K8S_NAMESPACE%'

                // Apply Kubernetes services
                bat 'kubectl apply -f k8s/service.yaml -n %K8S_NAMESPACE%'

                // Update Web deployment image
                bat '''
                kubectl set image deployment/%WEB_IMAGE% ^
                    %WEB_CONTAINER%=%WEB_IMAGE%:%BUILD_NUMBER% ^
                    -n %K8S_NAMESPACE%
                '''

                // Update API deployment image
                bat '''
                kubectl set image deployment/%API_IMAGE% ^
                    %API_CONTAINER%=%API_IMAGE%:%BUILD_NUMBER% ^
                    -n %K8S_NAMESPACE%
                '''

                echo 'Waiting for Web deployment...'

                // Verify Web rollout
                bat '''
                kubectl rollout status deployment/%WEB_IMAGE% ^
                    -n %K8S_NAMESPACE% ^
                    --timeout=180s
                '''

                echo 'Waiting for API deployment...'

                // Verify API rollout
                bat '''
                kubectl rollout status deployment/%API_IMAGE% ^
                    -n %K8S_NAMESPACE% ^
                    --timeout=180s
                '''
            }
        }

        // 9. Verify Kubernetes deployment
        stage('Verify Deployment') {
            steps {
                echo 'Checking Kubernetes deployment status...'

                bat 'kubectl get deployments -n %K8S_NAMESPACE%'

                bat 'kubectl get pods -n %K8S_NAMESPACE% -o wide'

                bat 'kubectl get services -n %K8S_NAMESPACE%'

                bat '''
                kubectl get deployment %WEB_IMAGE% -n %K8S_NAMESPACE%

                kubectl get deployment %API_IMAGE% -n %K8S_NAMESPACE%
                '''
            }
        }
    }

    post {

        success {
            echo 'SUCCESS: EmoJudge CI/CD pipeline completed.'
            echo 'Web and API deployed successfully to Kubernetes.'
        }

        failure {
            echo 'FAILED: EmoJudge CI/CD pipeline failed.'
            echo 'Check Jenkins Console Output for the failed stage.'
        }

        always {
            echo 'Final Docker image status:'

            bat(
                returnStatus: true,
                script: '"%DOCKER_EXE%" images emojudge-web'
            )

            bat(
                returnStatus: true,
                script: '"%DOCKER_EXE%" images emojudge-api'
            )

            echo 'Final Kubernetes resource status:'

            bat(
                returnStatus: true,
                script: 'kubectl get all -n %K8S_NAMESPACE%'
            )
        }
    }
}
