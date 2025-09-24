pipeline {
  agent { label "agent-2" }
  tools { nodejs 'NodeJS' }

  environment {
    DOCKER_HUB_REPO           = 'keanghor31/keanghor-app'
    DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
    GIT_CREDENTIALS_ID        = 'github-jenkins-token'   // MUST have write access
    BASE_VERSION              = '1.0'
    IMAGE_FILE                = 'manifests/deployment.yaml'   // file to update
    GIT_BRANCH                = 'main'
    GIT_URL                   = 'https://github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git'
    GIT_USER_NAME             = 'kshrd13thgeneration'
    GIT_USER_EMAIL            = 'hengenghour5@gmail.com'
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

    stage('Install Node Dependencies') {
      steps { sh 'npm install' }
    }

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

    stage('Trivy Scan') {
      steps {
        script {
          echo "🔍 Running Trivy scan..."
          sh '''
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \
              --severity HIGH,CRITICAL --no-progress --format table \
              -o trivy-scan-report.txt "$DOCKER_HUB_REPO:$IMAGE_TAG" || echo "Trivy scan failed but continuing..."
          '''
        }
      }
    }

    stage('Bump manifest image tag') {
      steps {
        script {
          echo "📝 Updating $IMAGE_FILE to use tag: $IMAGE_TAG"
          sh '''
            set -e
            echo "--- Before:"
            grep -n 'image:' "$IMAGE_FILE" || true

            # Replace only the tag for the target image
            sed -i -E "s|(image:\\s*$DOCKER_HUB_REPO:).*|\\1$IMAGE_TAG|" "$IMAGE_FILE"

            echo "--- After:"
            grep -n 'image:' "$IMAGE_FILE" || true
          '''
        }
      }
    }

    stage('Commit & Push manifest change') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${GIT_CREDENTIALS_ID}",
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          // NOTE: single-quoted Groovy string => no Groovy interpolation of secrets.
          // The token is supplied via a temporary credential helper (not visible on cmdline).
          sh '''
            set -e
            git config user.name  "$GIT_USER_NAME"
            git config user.email "$GIT_USER_EMAIL"

            git add "$IMAGE_FILE"
            git diff --staged || true

            # Commit may be a no-op if image already matches
            git commit -m "ci: deploy $DOCKER_HUB_REPO:$IMAGE_TAG" || echo "Nothing to commit"

            # Push securely using a throwaway credential helper (no token in URL or logs)
            git -c credential.helper='!f() { echo username=$GIT_USER; echo password=$GIT_TOKEN; }; f' \
                push https://github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git HEAD:$GIT_BRANCH
          '''
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
