pipeline {
    agent { label "agent-2" }

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKER_HUB_REPO = 'keanghor31/keanghor-app'
        DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
        BASE_VERSION = '1.0'
        K8S_NAMESPACE = 'argocd'
        ARGOCD_APP_NAME = 'argocdjenkins'
        ARGOCD_SERVER_IP = '104.154.141.175'
        DEPLOYMENT_FILE = 'manifests/deployment.yaml'
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
                            -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${IMAGE_TAG} || echo 'Trivy scan failed but continuing...'
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "🚀 Pushing Docker image: ${DOCKER_HUB_REPO}:${IMAGE_TAG} and :latest"
                    docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Update Deployment Manifest & Push to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-jenkins-token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    script {
                        echo "✏️ Updating deployment manifest with image tag: ${IMAGE_TAG}"

                        sh """
                            sed -i 's|image: ${DOCKER_HUB_REPO}:.*|image: ${DOCKER_HUB_REPO}:${IMAGE_TAG}|' ${DEPLOYMENT_FILE}

                            git config user.email "hengenghour5@gmail.com"
                            git config user.name "kshrd13thgeneration"
                            git add ${DEPLOYMENT_FILE}
                            git commit -m "🔧 Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                            git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/kshrd13thgeneration/Jenkins-ArgoCD-GitOps.git
                            git push origin main
                        """
                    }
                }
            }
        }

        stage('Sync ArgoCD Application') {
            steps {
                kubeconfig(
                    credentialsId: 'gcp-k8s-service-account',
                    serverUrl: "https://${ARGOCD_SERVER_IP}",
                    caCertificate: ''
                ) {
                    script {
                        echo "🔐 Logging into ArgoCD and syncing application..."
                        sh """
                            argocd login ${ARGOCD_SERVER_IP} \\
                                --username admin \\
                                --password \$(kubectl get secret -n ${K8S_NAMESPACE} argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \\
                                --insecure

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
