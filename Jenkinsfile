pipeline {
    agent { label "agent-2" }

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKER_HUB_REPO = 'keanghor31/keanghor-app'
        DOCKER_HUB_CREDENTIALS_ID = 'docker-hub-credentials'
        BASE_VERSION = '1.0'  // base version, change if needed
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
                    echo "Building docker image with tag: ${imageTag}"

                    dockerImage = docker.build("${DOCKER_HUB_REPO}:${imageTag}")

                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                // If trivy CLI is installed on your agent, use this:
                sh "trivy image --severity HIGH,CRITICAL --no-progress --format table -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}"

                // OR uncomment below to run Trivy via Docker container (if CLI missing)
                /*
                sh """
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \
                    --severity HIGH,CRITICAL --no-progress --format table \
                    -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}
                """
                */
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                script {
                    echo "Pushing docker image ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} and latest to DockerHub..."
                    docker.withRegistry('', "${DOCKER_HUB_CREDENTIALS_ID}") {
                        dockerImage.push(env.IMAGE_TAG)
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // Optional Kubernetes/ArgoCD stage here if needed
        /*
        stage('Apply Kubernetes Manifests & Sync App with ArgoCD') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig', serverUrl: '34.87.128.146:8080') {
                        sh '''
                            argocd login 104.154.141.175 --username admin --password $(kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) --insecure
                            argocd app sync argocdjenkins
                        '''
                    }
                }
            }
        }
        */
    }

    post {
        success {
            echo 'Build & Deploy completed successfully!'
            archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
        }
        failure {
            echo 'Build & Deploy failed. Check logs.'
            archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
        }
    }
}
