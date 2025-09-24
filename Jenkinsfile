pipeline {
    agent { label "agent-2" }

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKER_HUB_REPO = 'keanghor31/keanghor-app'
        DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
        BASE_VERSION = '1.0'  // Base version for tagging
    }

    stages {
        stage('Test Node') {
            steps {
                sh 'node -v'
                sh 'npm -v'
            }
        }

        stage('Checkout Github') {
            steps {
                git branch: 'main', credentialsId: 'GitOps-Token-GitHub', url: 'https://github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git'
            }
        }

        stage('Install node dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${BASE_VERSION}.${env.BUILD_NUMBER}"
                    echo "📦 Building Docker image with tag: ${imageTag}"
                    dockerImage = docker.build("${DOCKER_HUB_REPO}:${imageTag}")
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                // You can use local Trivy CLI or Docker method if CLI not available
                // sh '''
                //     trivy image --severity HIGH,CRITICAL \
                //         --no-progress --format table \
                //         -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${IMAGE_TAG} || echo "Trivy scan failed but continuing..."
                // '''
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                script {
                    echo "🚀 Pushing Docker image: ${DOCKER_HUB_REPO}:${IMAGE_TAG} and :latest"
                    docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Apply Kubernetes Manifests & Sync App with ArgoCD') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        echo "🔐 Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project keanghor
                        gcloud container clusters get-credentials cluster-1 --zone us-central1-a --project keanghor

                        echo "🔁 Logging into ArgoCD & syncing app..."
                        argocd login 34.134.61.204 --username admin --password $(kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) --insecure
                        argocd app sync argocdjenkins
                    '''
                }
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
