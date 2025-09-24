pipeline {
  agent { label "agent-2" }
  tools { nodejs 'NodeJS' }

  environment {
    DOCKER_HUB_REPO           = 'keanghor31/keanghor-app'
    DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
    GIT_CREDENTIALS_ID        = 'github-jenkins-token'
    BASE_VERSION              = '1.0'
    IMAGE_FILE                = 'manifests/deployment.yaml'   // file to update
    GIT_BRANCH                = 'main'
    GIT_URL                   = 'https://github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git'
    GIT_USER_NAME             = 'jenkins-bot'
    GIT_USER_EMAIL            = 'jenkins-bot@local'
  }

  stages {
    stage('Check Tooling') {
      steps {
        sh 'node -v && npm -v'
        sh 'docker version || true'
      }
    }

    stage('Checkout') {
      steps {
        git branch: "${GIT_BRANCH}", credentialsId: "${GIT_CREDENTIALS_ID}", url: "${GIT_URL}"
      }
    }

    stage('Install Node Dependencies') { steps { sh 'npm install' } }

    stage('Build & Push Image') {
      steps {
        script {
          def imageTag = "${BASE_VERSION}.${env.BUILD_NUMBER}"
          echo "📦 Building ${DOCKER_HUB_REPO}:${imageTag}"
          def img = docker.build("${DOCKER_HUB_REPO}:${imageTag}")
          env.IMAGE_TAG = imageTag

          echo "🚀 Pushing ${DOCKER_HUB_REPO}:${imageTag} and :latest"
          docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
            img.push("${imageTag}")
            img.push("latest")
          }
        }
      }
    }

    stage('Bump manifest image tag') {
      steps {
        script {
          echo "📝 Updating ${IMAGE_FILE} to use tag: ${IMAGE_TAG}"
          sh """
            set -e
            grep -n 'image:' ${IMAGE_FILE} || true
            sed -i -E "s|(image:\\s*${DOCKER_HUB_REPO}:).*|\\1${IMAGE_TAG}|" ${IMAGE_FILE}
            echo '--- Updated image line(s):'
            grep -n 'image:' ${IMAGE_FILE} || true
          """
        }
      }
    }

    stage('Commit & Push manifest change') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
          sh """
            set -e
            git config user.name  "${GIT_USER_NAME}"
            git config user.email "${GIT_USER_EMAIL}"
            git add ${IMAGE_FILE}
            git commit -m "ci: deploy ${DOCKER_HUB_REPO}:${IMAGE_TAG}"
            # Push via https with credentials
            git push https://${GIT_USER}:${GIT_PASS}@${GIT_URL.replace('https://','')} ${GIT_BRANCH}
          """
        }
      }
    }

    stage('(Info) Wait for ArgoCD auto-sync') {
      steps {
        echo '🧠 Argo CD will auto-sync this commit. No CLI calls needed.'
      }
    }
  }

  post {
    success {
      echo '✅ Build & GitOps update completed! Argo CD will deploy automatically.'
      archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
    }
    failure {
      echo '❌ Build failed.'
      archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
    }
  }
}
