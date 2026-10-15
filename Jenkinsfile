pipeline {
    agent any

    environment {
        IMAGE = 'pratheebapari/lms-portal'
        DOCKERHUB = credentials('dockerhub-creds')
        KUBECONFIG = credentials('kubeconfig')
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/pratheebapari/lms-portal.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% -t %IMAGE%:latest .'
            }
        }

        stage('Test Application') {
            steps {
                bat '''
                docker rm -f lms-ci-test 2>nul
                docker run -d --name lms-ci-test -p 8085:80 %IMAGE%:%BUILD_NUMBER%
                ping -n 6 127.0.0.1 >nul
                curl -f http://localhost:8085 || exit /b 1
                docker rm -f lms-ci-test
                '''
            }
        }

        stage('Push Image') {
            steps {
                bat '''
                echo %DOCKERHUB_PSW%| docker login -u %DOCKERHUB_USR% --password-stdin
                docker push %IMAGE%:%BUILD_NUMBER%
                docker push %IMAGE%:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat '''
                powershell -Command "(Get-Content deployment.yaml) -replace 'lms-portal:latest','lms-portal:%BUILD_NUMBER%' | Set-Content rendered.yaml"
                kubectl apply -f rendered.yaml
                '''
            }
        }

        stage('Health Check and Rollback') {
            steps {
                script {
                    def status = bat(returnStatus: true, script: 'kubectl rollout status deployment/lms-portal --timeout=60s')
                    if (status != 0) {
                        bat 'kubectl rollout undo deployment/lms-portal'
                        bat 'kubectl rollout status deployment/lms-portal --timeout=90s'
                        bat 'kubectl get pods'
                        error('New release failed health check. Rolled back to previous stable release.')
                    }
                    bat 'kubectl get pods'
                }
            }
        }
    }
}