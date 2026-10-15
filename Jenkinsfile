
pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timeout(time: 15, unit: 'MINUTES')
    }

    environment {
        IMAGE_REPO = 'dhan01/university-lms'
        APP_NAME = 'university-lms'
        SERVICE_NAME = 'university-lms-service'
        NODE_PORT = '30010'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_REPO%:%BUILD_NUMBER% .'
                bat 'docker tag %IMAGE_REPO%:%BUILD_NUMBER% %IMAGE_REPO%:latest'
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    bat 'docker rm -f university-lms-ci-test 2>NUL || exit /b 0'
                    bat 'docker run -d --name university-lms-ci-test -p 18080:80 %IMAGE_REPO%:%BUILD_NUMBER%'

                    try {
                        int testResult = bat(
                            returnStatus: true,
                            script: '''
                                powershell -NoProfile -Command "$ok = $false; for ($i=0; $i -lt 12; $i++) { try { $r = Invoke-WebRequest -UseBasicParsing http://localhost:18080/healthz -TimeoutSec 3; if ($r.StatusCode -eq 200 -and $r.Content -match 'healthy') { $ok = $true; break } } catch {}; Start-Sleep -Seconds 2 }; if (-not $ok) { exit 1 }; $p = Invoke-WebRequest -UseBasicParsing http://localhost:18080/ -TimeoutSec 5; if ($p.StatusCode -ne 200 -or $p.Content -notmatch 'University Learning Management Portal') { exit 1 }"
                            '''
                        )

                        if (testResult != 0) {
                            error('Docker image test failed')
                        }
                    } finally {
                        bat 'docker rm -f university-lms-ci-test'
                    }
                }
            }
        }

        stage('Publish to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    bat 'echo %DOCKERHUB_TOKEN%| docker login -u %DOCKERHUB_USER% --password-stdin'
                    bat 'docker push %IMAGE_REPO%:%BUILD_NUMBER%'
                    bat 'docker push %IMAGE_REPO%:latest'
                    bat 'docker logout'
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        bat 'kubectl apply -f deployment.yaml'
                        bat 'kubectl set image deployment/%APP_NAME% %APP_NAME%=%IMAGE_REPO%:%BUILD_NUMBER%'

                        int rolloutResult = bat(
                            returnStatus: true,
                            script: 'kubectl rollout status deployment/%APP_NAME% --timeout=120s'
                        )

                        if (rolloutResult != 0) {
                            echo 'NEW RELEASE FAILED. Initiating automatic rollback.'

                            int undoResult = bat(
                                returnStatus: true,
                                script: 'kubectl rollout undo deployment/%APP_NAME%'
                            )

                            if (undoResult != 0) {
                                error('Automatic rollback command failed. Manual recovery is required.')
                            }

                            int recoveryResult = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/%APP_NAME% --timeout=120s'
                            )

                            if (recoveryResult != 0) {
                                error('Rollback was initiated but the previous release did not become healthy.')
                            }

                            bat 'kubectl get pods -l app=university-lms -o wide'
                            error('Release failed. Previous stable release restored; pipeline marked FAILED.')
                        }

                        bat 'kubectl get deployment %APP_NAME%'
                        bat 'kubectl get pods -l app=university-lms'
                        bat 'kubectl get service %SERVICE_NAME%'

                        int healthResult = bat(
                            returnStatus: true,
                            script: '''
                                powershell -NoProfile -Command "$ok = $false; for ($i=0; $i -lt 10; $i++) { try { $r = Invoke-WebRequest -UseBasicParsing http://localhost:30010/healthz -TimeoutSec 3; if ($r.StatusCode -eq 200) { $ok = $true; break } } catch {}; Start-Sleep -Seconds 2 }; if (-not $ok) { exit 1 }; $p = Invoke-WebRequest -UseBasicParsing http://localhost:30010/ -TimeoutSec 5; if ($p.StatusCode -ne 200 -or $p.Content -notmatch 'University Learning Management Portal') { exit 1 }"
                            '''
                        )

                        if (healthResult != 0) {
                            echo 'Service health check failed. Initiating automatic rollback.'

                            bat 'kubectl rollout undo deployment/%APP_NAME%'

                            int recoveryResult = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/%APP_NAME% --timeout=120s'
                            )

                            if (recoveryResult != 0) {
                                error('Service health failed and rollback did not restore a healthy rollout.')
                            }

                            error('Post-deployment health check failed. Previous release restored; pipeline marked FAILED.')
                        }

                        echo 'Deployment and application health checks passed.'
                    }
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f university-lms-ci-test 2>NUL || exit /b 0'
            echo 'CI/CD pipeline execution finished.'
        }
    }
}
