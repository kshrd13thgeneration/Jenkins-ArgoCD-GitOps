pipeline {
  agent { label "agent-2" }

  tools {
    nodejs 'NodeJS'
  }

  environment {
    DOCKER_HUB_REPO           = 'keanghor31/keanghor-app'
    DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
    BASE_VERSION              = '1.0'
    K8S_NAMESPACE             = 'argocd'
    ARGOCD_APP_NAME           = 'argocdjenkins'
    ARGOCD_SERVER             = '104.154.141.175' // Argo CD endpoint (not K8s API)
  }

  stages {
    stage('Test Node') {
      steps {
        sh 'node -v'
        sh 'npm -v'
      }
    }

    stage('Checkout GitHub') {
      steps {
        git branch: 'main',
            credentialsId: 'github-jenkins-token',
            url: 'https://github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git'
      }
    }

    stage('Install Node Dependencies') {
      steps { sh 'npm install' }
    }

    stage('Install CLIs (kubectl & argocd)') {
      steps {
        sh '''
          set -e
          if ! command -v kubectl >/dev/null 2>&1; then
            echo "Installing kubectl..."
            KVER=$(curl -sL https://dl.k8s.io/release/stable.txt)
            curl -sLO https://dl.k8s.io/release/${KVER}/bin/linux/amd64/kubectl
            install -m 0755 kubectl /usr/local/bin/kubectl
          fi
          if ! command -v argocd >/dev/null 2>&1; then
            echo "Installing argocd CLI..."
            curl -sLO https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
            install -m 0755 argocd-linux-amd64 /usr/local/bin/argocd
          fi
          kubectl version --client
          argocd version --client
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def imageTag = "${BASE_VERSION}.${env.BUILD_NUMBER}"
          echo "📦 Building Docker image with tag: ${imageTag}"
          def dockerImage = docker.build("${DOCKER_HUB_REPO}:${imageTag}")
          env.IMAGE_TAG = imageTag

          // Push inside this stage or keep your next stage:
          docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
            dockerImage.push("${IMAGE_TAG}")
            dockerImage.push("latest")
          }
        }
      }
    }

    stage('Trivy Scan') {
      steps {
        script {
          echo "🔍 Running Trivy scan with Docker container..."
          sh """
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \\
              --severity HIGH,CRITICAL --no-progress --format table \\
              -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${IMAGE_TAG} || echo 'Trivy scan failed but continuing...'
          """
        }
      }
    }

    // If you kept pushing in Build stage, you can delete this stage.
    // Kept here in case you prefer separate concerns:
    stage('Push Docker Image') {
      when { expression { return false } } // disable if already pushed
      steps {
        script {
          echo "🚀 Pushing Docker image: ${DOCKER_HUB_REPO}:${IMAGE_TAG} and :latest"
          docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
            docker.image("${DOCKER_HUB_REPO}:${IMAGE_TAG}").push()
            docker.image("${DOCKER_HUB_REPO}:${IMAGE_TAG}").push('latest')
          }
        }
      }
    }

    stage('Sync ArgoCD Application') {
      environment { ARGOCD_TOKEN = credentials('argocd-token') } // <- create this in Jenkins
      steps {
        sh """
          # Ensure server entry exists; then auth by token (preferred)
          argocd login ${ARGOCD_SERVER} --insecure --grpc-web --username dummy --password dummy || true
          argocd --server ${ARGOCD_SERVER} --grpc-web --auth-token "$ARGOCD_TOKEN" app sync ${ARGOCD_APP_NAME}
        """
      }
    }
  }

  post {
    success {
      echo '✅ Build & Deploy completed successfully!'
      archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
    }
    failure {
      echo '❌ Build & Deploy failed. Check logs.'
      archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
    }
  }
}
