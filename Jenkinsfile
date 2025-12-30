pipeline {
    agent any  // Ya 'docker' agent agar containerized build chahiye

    environment {
        // Change these as per your setup
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'   // tumhara ID
        GITHUB_CREDENTIALS    = 'github-credentials'       // optional, agar private repo
        DOCKER_USERNAME       = 'pixelopscloud'            // tumhara Docker Hub username
        APP_NAME              = 'user-management'          // optional branding
        
        // Image tags — latest ya build number use kar sakte ho
        BACKEND_IMAGE  = "${DOCKER_USERNAME}/devsecops-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "${DOCKER_USERNAME}/devsecops-frontend:${BUILD_NUMBER}"
        
        // MicroK8s kubeconfig credential ID (next step mein add karenge)
        KUBECONFIG_CRED_ID = 'microk8s-kubeconfig'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm  // GitHub se pull karega (credentials already global)
                sh 'ls -la'   // Debug: files dikhao
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Backend build
                    dir('backend') {
                        sh "docker build -t ${BACKEND_IMAGE} ."
                    }
                    // Frontend build
                    dir('frontend') {
                        sh "docker build -t ${FRONTEND_IMAGE} ."
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    // Trivy install karna padega agent pe agar nahi hai, ya docker run use karo
                    sh """
                    docker run --rm \
                        aquasec/trivy image --exit-code 0 --no-progress --format table \
                        ${BACKEND_IMAGE}
                    """
                    sh """
                    docker run --rm \
                        aquasec/trivy image --exit-code 0 --no-progress --format table \
                        ${FRONTEND_IMAGE}
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh "docker push ${BACKEND_IMAGE}"
                    sh "docker push ${FRONTEND_IMAGE}"
                }
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                withKubeConfig([credentialsId: KUBECONFIG_CRED_ID, serverUrl: 'https://127.0.0.1:16443']) {  // MicroK8s default API server
                    script {
                        // Update image in manifests (sed ya kustomize use kar sakte ho)
                        sh """
                        sed -i 's|image:.*backend.*|image: ${BACKEND_IMAGE}|g' k8s-manifests/backend-deployment.yaml
                        sed -i 's|image:.*frontend.*|image: ${FRONTEND_IMAGE}|g' k8s-manifests/frontend-deployment.yaml
                        """
                        sh 'kubectl apply -f k8s-manifests/'
                        sh 'kubectl rollout status deployment/backend'
                        sh 'kubectl rollout status deployment/frontend'
                    }
                }
            }
        }

        // Optional: OWASP ZAP (agar Jenkins mein ZAP plugin + tool installed hai)
        stage('OWASP ZAP Scan') {
            when { expression { false } }  // Abhi disable — enable karna ho toh remove when
            steps {
                echo "OWASP ZAP dynamic scan would run here against frontend service..."
                // Example: zap: archivedArtifacts allowEmptyArchive: true, artifacts: 'zap-report.html'
            }
        }
    }

    post {
        always {
            sh 'docker logout'  // Security best practice
            echo 'Pipeline finished!'
        }
        success {
            echo 'Deployment successful 🎉'
        }
        failure {
            echo 'Pipeline failed — check logs!'
        }
    }
}
