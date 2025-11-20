pipeline {
    agent any

    tools {
        nodejs 'node18'
        sonarScanner 'sonar-scanner'
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
                        echo "PWD = $(pwd)"
                        ls -l
                        echo "Installing Node Modules..."
                        npm install
                    '''
                }
            }
        }

        stage('SAST Scan - SonarQube') {
