pipeline {
    agent any

    environment {
        REPO_URL = "https://github.com/Mayur7225/node-todo-cicd.git"
        BRANCH   = "main"
        IMAGE    = "mayur7225/node-todo:latest"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning clean repo..."
                sh '''
                    rm -rf source
                    git clone -b ${BRANCH} ${REPO_URL} source
                    ls -l source
                '''
            }
        }

        stage('Install Node Modules') {
            steps {
                echo "Running npm install..."
                sh '''
                    docker run --rm \
                      -v "$PWD/source":/app \
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
                    docker build -t ${IMAGE} source
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing Docker image..."
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE}
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying app with docker-compose..."
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

