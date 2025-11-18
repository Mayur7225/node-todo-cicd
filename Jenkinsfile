pipeline {
    agent any

    environment {
        REPO_URL = "https://github.com/Mayur7225/node-todo-cicd.git"
        DOCKER_CREDS = credentials('dockerhub')   // DockerHub credentials ID
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning clean repo..."
                sh '''
                    rm -rf source
                    git clone -b main ${REPO_URL} source
                    ls -l source
                '''
            }
        }

        stage('Install Node Modules') {
            steps {
                echo "Running npm install..."
                sh '''
                    HOST_SOURCE="/var/lib/docker/volumes/jenkins_home/_data/workspace/${JOB_NAME}/source"
                    echo "Using host source path: $HOST_SOURCE"

                    docker run --rm \
                      -v $HOST_SOURCE:/app \
                      -w /app \
                      node:18-alpine \
                      sh -c "npm install"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker Image..."
                sh '''
                    docker build -t mayur7225/node-todo:latest source
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing Docker image..."

                sh '''
                    echo $DOCKER_CREDS_PSW | docker login \
                      -u $DOCKER_CREDS_USR \
                      --password-stdin

                    docker push mayur7225/node-todo:latest
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying with docker-compose..."
                sh '''
                    cd source
                    docker compose down || true
                    docker compose up -d
                '''
            }
        }
    }

    post {
        success {
            echo "PIPELINE SUCCESSFUL ✅"
        }
        failure {
            echo "PIPELINE FAILED ❌"
        }
    }
}

