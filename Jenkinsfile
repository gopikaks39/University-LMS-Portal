
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = 'gopikaks/university-lms-portal'
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% .'
                bat 'docker tag %IMAGE%:%BUILD_NUMBER% %IMAGE%:latest'
            }
        }

        stage('Test Application') {
            steps {
                bat 'docker run -d --name lms-test-%BUILD_NUMBER% -p 18080:80 %IMAGE%:%BUILD_NUMBER%'
                bat 'powershell -NoProfile -Command "$ok=$false; for($i=0;$i -lt 15;$i++){ try {$r=Invoke-WebRequest http://localhost:18080 -UseBasicParsing -TimeoutSec 2; if($r.StatusCode -eq 200 -and $r.Content -match \'University Learning Management Portal\') {$ok=$true; break} } catch {}; Start-Sleep -Seconds 2 }; if(-not $ok){exit 1}"'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat 'docker login -u "%DH_USER%" --password-stdin <<< "%DH_TOKEN%"'
                    bat 'docker push %IMAGE%:%BUILD_NUMBER%'
                    bat 'docker push %IMAGE%:latest'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat 'kubectl apply -f deployment.yaml'
                bat 'kubectl set image deployment/university-lms university-lms=%IMAGE%:%BUILD_NUMBER%'
            }
        }

        stage('Health Check') {
            steps {
                bat 'kubectl rollout status deployment/university-lms --timeout=120s'
                bat 'kubectl get pods -l app=university-lms'
                bat 'kubectl get service university-lms-service'
            }
        }
    }

    post {
        always {
            bat 'docker rm -f lms-test-%BUILD_NUMBER%'
        }

        failure {
            echo 'Pipeline failed. Attempting to restore the previous stable deployment.'
            bat 'kubectl rollout undo deployment/university-lms'
            bat 'kubectl rollout status deployment/university-lms --timeout=120s'
        }
    }
}
