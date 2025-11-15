pipeline {
  agent any

  environment {
    REPO_URL = "https://github.com/Mayur7225/node-todo-cicd.git"
    BRANCH = "main"
    IMAGE = "todo-devops-app"
    DOCKERHUB_USER = "mayur7225"
    SONAR_SERVER = "Sonar"        // must match Configure System name
  }

  options {
    timeout(time: 60, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Cloning repo..."
        git url: "${REPO_URL}", branch: "${BRANCH}"
      }
    }

    stage('Prepare') {
      steps {
        echo "Installing node modules (ci)"
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
        '''
      }
    }

    stage('SAST - ESLint') {
      steps {
        echo "Running ESLint"
        sh '''
          if [ -f .eslintrc ] || [ -f .eslintrc.js ] || [ -f .eslintrc.json ]; then
            npx eslint . --max-warnings=0 || true
          else
            npm i --no-audit --no-fund --no-package-lock eslint@8 -D || true
            npx eslint . --max-warnings=0 || true
          fi
        '''
      }
    }

    stage('Dependency Scan (npm audit)') {
      steps {
        echo "Running npm audit"
        sh 'npm audit --audit-level=high || true'
      }
    }

    stage('Secret Scan - Gitleaks') {
      steps {
        echo "Running Gitleaks (secret scan)"
        sh '''
          # try to use local gitleaks if installed, otherwise use docker
          if command -v gitleaks >/dev/null 2>&1; then
            gitleaks detect --source . --exit-code 1 || true
          else
            docker run --rm -v "$PWD":/code zricethezav/gitleaks:latest detect --source /code --exit-code 1 || true
          fi
        '''
      }
    }

    stage('Generate SBOM (Syft)') {
      steps {
        echo "Generating SBOM with syft (docker fallback)"
        sh '''
          if command -v syft >/dev/null 2>&1; then
            syft packages-dir:. -o cyclonedx-json=sbom-cyclonedx.json || true
          else
            docker run --rm -v "$PWD":/project anchore/syft:latest packages dir:/project -o cyclonedx-json=/project/sbom-cyclonedx.json || true
          fi
        '''
        archiveArtifacts artifacts: 'sbom-cyclonedx.json', allowEmptyArchive: true
      }
    }

    stage('SAST - SonarQube (Docker scanner)') {
      steps {
        echo "Running SonarScanner via Docker image"
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
          // withSonarQubeEnv supplies SONAR_HOST_URL from configured Sonar server
          withSonarQubeEnv("${SONAR_SERVER}") {
            sh '''
              # run sonar-scanner in docker to avoid installing scanner in Jenkins container
              docker run --rm -v "$PWD":/usr/src -e SONAR_HOST_URL=$SONAR_HOST_URL -e SONAR_TOKEN=$SONAR_AUTH_TOKEN sonarsource/sonar-scanner-cli \
                -Dsonar.projectKey=${IMAGE} \
                -Dsonar.sources=/usr/src \
                -Dsonar.login=$SONAR_AUTH_TOKEN \
                -Dsonar.host.url=$SONAR_HOST_URL
            '''
          }
        }
      }
      post {
        always {
          echo "Check Sonar dashboard for analysis results."
        }
      }
    }

    stage('Quality Gate') {
      steps {
        echo "Waiting for SonarQube Quality Gate..."
        // abortPipeline false so pipeline continues even if quality gate fails - adjust if you want to fail
        timeout(time: 2, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: false
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        echo "Building Docker image"
        sh "docker build -t ${IMAGE}:latest ."
      }
    }

    stage('Container Scan - Trivy') {
      steps {
        echo "Running Trivy image scan"
        sh '''
          if command -v trivy >/dev/null 2>&1; then
            trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE}:latest || true
          else
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE}:latest || true
          fi
        '''
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerHubCreds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo $DH_PASS | docker login -u $DH_USER --password-stdin
            docker tag ${IMAGE}:latest $DH_USER/${IMAGE}:latest
            docker push $DH_USER/${IMAGE}:latest
          '''
        }
      }
    }

    stage('Deploy (docker compose)') {
      steps {
        echo "Deploying with docker compose"
        sh '''
          # ensure docker compose v2
          docker compose down || true
          docker compose pull || true
          docker compose up -d --build
        '''
      }
    }
  } // stages

  post {
    success {
      echo "Pipeline SUCCESS ✅"
    }
    unstable {
      echo "Pipeline UNSTABLE — check warnings"
    }
    failure {
      echo "Pipeline FAILED ❌ Check console logs"
    }
  }
}

