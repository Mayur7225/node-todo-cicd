pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKER_IMAGE = "mayur7225/node-todo:latest"
    }

    stages {

        stage('Checkout') {
            steps {
                dir('source') {
                    deleteDir()
                    git branch: 'main', url: 'https://github.com/Mayur7225/node-todo-cicd.git'
                }
            }
        }

        stage('Install Node Modules') {
            steps {
                dir('source') {
                    sh '''
                        echo "PWD inside Jenkins = $(pwd)"
                        ls -l
                        echo "Installing Node Modules..."
                        npm install
                    '''
                }
            }
        }


        stage('SAST Scan - SonarQube') {
            steps {
                dir('source') {
                    withSonarQubeEnv('sonarqube-server') {
                        script {
                            def scanner = tool 'sonar-scanner'
                            sh """${scanner}/bin/sonar-scanner \
                            -Dsonar.projectKey=node-todo \
                            -Dsonar.sources=. \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info"""
                        }
                    }
                }
            }
        }


        stage('Secrets Scan - GitLeaks') {
            steps {
                dir('source') {
                    sh '''
                        docker run --rm -v $(pwd):/scan zricethezav/gitleaks:latest detect \
                        --source=/scan --no-banner || true
                    '''
                }
            }
        }


        stage('Dependency Scan - Trivy FS Scan') {
            steps {
                dir('source') {
                    sh '''
                        docker run --rm -v $(pwd):/app aquasec/trivy fs \
                        --exit-code 1 --severity HIGH,CRITICAL /app || true
                    '''
                }
            }
        }


        stage('Build Docker Image') {
            steps {
                dir('source') {
                    sh """
                        docker build -t ${DOCKER_IMAGE} .
                    """
                }
            }
        }


        stage('Image Scan - Trivy Image Scan') {
            steps {
                sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_IMAGE} || true
                """
            }
        }


        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                    '''
                    sh """
                        docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }


        stage('Deploy') {
            steps {
                dir('source') {
                    sh '''
                        docker compose down || true
                        docker compose up -d
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "DevSecOps Pipeline Completed Successfully! 🎉"
        }
        failure {
            echo "Pipeline Failed ❌"
        }
    }
}
