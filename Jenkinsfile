
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
                    if errorlevel 1 exit /b 1
                '''

                powershell '''
                    $ready = $false

                    for ($i = 0; $i -lt 15; $i++) {
                        try {
                            $response = Invoke-WebRequest -Uri "http://localhost:18080/" -TimeoutSec 3 -UseBasicParsing

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
                        throw "Website test failed."
                    }

                    Write-Host "Website test passed."
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    bat '''
                        docker login -u "%DH_USER%" -p "%DH_TOKEN%"
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        docker tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:latest
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
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
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get deployment %DEPLOYMENT_NAME%
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=university-lms
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get service %SERVICE_NAME%
                        if errorlevel 1 exit /b 1
                    '''

                    powershell '''
                        $ready = $false

                        for ($i = 0; $i -lt 10; $i++) {
                            try {
                                $response = Invoke-WebRequest -Uri "http://localhost:30011/" -TimeoutSec 5 -UseBasicParsing

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

                        Write-Host "Kubernetes health check passed."
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f lms-test-%BUILD_NUMBER% 2>NUL'
        }

        success {
            echo 'University LMS pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Attempting rollback if deployment was started.'

            script {
                if (env.DEPLOY_ATTEMPTED == 'true') {
                    withCredentials([
                        file(
                            credentialsId: 'kuberconfig',
                            variable: 'KUBECONFIG_FILE'
                        )
                    ]) {
                        bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout undo deployment/%DEPLOYMENT_NAME%
                            if errorlevel 1 exit /b 1

                            kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=180s
                            if errorlevel 1 exit /b 1
                        '''
                    }
                } else {
                    echo 'Deployment was not started; rollback not required.'
                }
            }
        }
    }
}
