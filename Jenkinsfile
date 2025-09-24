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

  //       stage('Install Kubectl & ArgoCD CLI'){
		// 	steps {
		// 		sh '''
		// 		echo 'installing Kubectl & ArgoCD cli...'
		// 		curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
		// 		chmod +x kubectl
		// 		mv kubectl /usr/local/bin/kubectl
		// 		curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
		// 		chmod +x /usr/local/bin/argocd
		// 		'''
		// 	}
		// }

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
            archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive
