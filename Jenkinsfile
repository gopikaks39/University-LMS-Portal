
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'gopikaks/university-lms-portal'
        DEPLOYMENT_NAME = 'university-lms'
        CONTAINER_NAME = 'university-lms'
        SERVICE_NAME = 'university-lms-service'
        DEPLOY_ATTEMPTED = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'dir'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Website') {
            steps {
                bat '''
                    docker rm -f lms-test-%BUILD_NUMBER% 2>NUL
                    docker run -d --name lms-test-%BUILD_NUMBER% -p 18080:80 %IMAGE_NAME%:%BUILD_NUMBER%
                '''

                powershell '''
                    $ready = $false

                    for ($i = 0; $i -lt 15; $i++) {
                        try {
                            $response = Invoke-WebRequest `
                                -Uri "http://localhost:18080/" `
                                -TimeoutSec 3 `
                                -UseBasicParsing

                            if ($response.StatusCode -eq 200) {
                                $ready = $true
                                break
                            }
                        }
                        catch {
                            Start-Sleep -Seconds 2
                        }
                    }

                    if (-not $ready) {
                        throw "Website health check failed."
                    }

                    Write-Host "Website health check passed."
                '''
            }
        }

        stage('Push Versioned Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    powershell '''
                        $env:DH_TOKEN |
                            docker login -u $env:DH_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed."
                        }
                    '''

                    bat 'docker push %IMAGE_NAME%:%BUILD_NUMBER%'

                    bat '''
                        docker tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest
                        docker push %IMAGE_NAME%:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        env.DEPLOY_ATTEMPTED = 'true'
                    }

                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%BUILD_NUMBER%

                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=180s

                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get deployment %DEPLOYMENT_NAME%
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=university-lms
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get service %SERVICE_NAME%
                    '''

                    powershell '''
                        $ready = $false

                        for ($i = 0; $i -lt 10; $i++) {
                            try {
                                $response = Invoke-WebRequest `
                                    -Uri "http://localhost:30011/" `
                                    -TimeoutSec 5 `
                                    -UseBasicParsing

                                if ($response.StatusCode -eq 200) {
                                    $ready = $true
                                    break
                                }
                            }
                            catch {
                                Start-Sleep -Seconds 3
                            }
                        }

                        if (-not $ready) {
                            throw "Deployed website health check failed."
                        }

                        Write-Host "Kubernetes website health check passed."
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f lms-test-%BUILD_NUMBER% 2>NUL || exit /b 0'
        }

        success {
            echo 'University LMS pipeline completed successfully.'
            echo 'The versioned Docker image has been deployed and verified.'
        }

        failure {
            echo 'Pipeline failed. Checking whether a deployment rollback is required.'

            script {
                if (env.DEPLOY_ATTEMPTED == 'true') {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG_FILE'
                        )
                    ]) {
                        bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout undo deployment/%DEPLOYMENT_NAME%

                            if errorlevel 1 exit /b 1

                            kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=180s
                        '''
                    }

                    echo 'Rollback command executed. Check rollout status to confirm recovery.'
                } else {
                    echo 'Deployment was not attempted; no Kubernetes rollback was run.'
                }
            }
        }
    }
}
