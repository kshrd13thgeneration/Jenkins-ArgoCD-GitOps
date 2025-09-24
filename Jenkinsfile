pipeline {
    agent { label "agent-2" }

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKER_HUB_REPO = 'keanghor31/keanghor-app'
        DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
        BASE_VERSION = '1.0'  // Base version for tagging
        K8S_NAMESPACE = 'argocd' // namespace where argocd is installed
        ARGOCD_APP_NAME = 'argocdjenkins' // your argocd app name
        ARGOCD_SERVER_IP = '104.154.141.175' // fixed ArgoCD LoadBalancer IP
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
                script {
                    echo "🔍 Running Trivy scan with Docker container..."
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \\
                            --severity HIGH,CRITICAL --no-progress --format table \\
                            -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} || echo 'Trivy scan failed but continuing...'
                    """
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                script {
                    echo "🚀 Pushing Docker image: ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} and :latest"
                    docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push("${env.IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Login to ArgoCD & Sync App') {
            steps {
                kubeconfig(credentialsId: 'gcp-k8s-service-account', serverUrl: "https://${env.ARGOCD_SERVER_IP}") {
                    script {
                        echo "🔐 Logging into ArgoCD and syncing app..."
                        sh """
                        argocd login ${env.ARGOCD_SERVER_IP} --username admin --password \$(kubectl get secret -n ${K8S_NAMESPACE} argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) --insecure
                        argocd app sync ${ARGOCD_APP_NAME}
                        """
                    }
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
