pipeline {
    agent { label 'dev-server' }

    environment {
        REPO_URL = "https://github.com/Mayur7225/node-todo-cicd.git"
        BRANCH = "main"
        IMAGE_NAME = "todo-devops-app"
        DOCKERHUB_USER = "mayur7225"
        SONAR = "Sonar"
    }

    stages {

        stage("Clone Code") {
            steps {
                git url: "${REPO_URL}", branch: "${BRANCH}"
            }
        }

        stage("SAST - SonarQube") {
            steps {
                withSonarQubeEnv("${SONAR}") {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${IMAGE_NAME} \
                        -Dsonar.projectName=${IMAGE_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.language=js \
                        -Dsonar.sourceEncoding=UTF-8
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage("OWASP Dependency Scan") {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'OWASP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh "trivy image --severity HIGH,CRITICAL --ignore-unfixed ${IMAGE_NAME}:latest"
            }
        }

        stage("Push to DockerHub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId:"dockerHubCreds",
                    usernameVariable:"USER",
                    passwordVariable:"PASS"
                )]) {
                    sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker tag ${IMAGE_NAME}:latest $USER/${IMAGE_NAME}:latest
                        docker push $USER/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage("Deploy") {
            steps {
                sh "docker compose down || true"
                sh "docker compose pull || true"
                sh "docker compose up -d"
            }
        }
    }

    post {
        success {
            echo "DevSecOps Pipeline Successfully Completed! 🚀"
        }
        failure {
            echo "Pipeline Failed ❌ Check the logs."
        }
    }
}

