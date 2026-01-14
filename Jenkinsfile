pipeline {
    agent any 

    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        DOCKER_USERNAME       = 'pixelopscloud'
        BACKEND_IMAGE  = "${DOCKER_USERNAME}/devsecops-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "${DOCKER_USERNAME}/devsecops-frontend:${BUILD_NUMBER}"
        KUBECONFIG_CRED_ID = 'microk8s-kubeconfig'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    dir('backend') {
                        sh "docker build -t ${BACKEND_IMAGE} ."
                    }
                    dir('frontend') {
                        sh "docker build -t ${FRONTEND_IMAGE} ."
                    }
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    // Backend scan with timeout + only vuln scanner
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        --exit-code 0 \
                        --no-progress \
                        --format table \
                        --timeout 20m \
                        --scanners vuln \
                        ${BACKEND_IMAGE} > backend-trivy-scan.txt || true
                    """

                    // Frontend scan same settings
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        --exit-code 0 \
                        --no-progress \
                        --format table \
                        --timeout 20m \
                        --scanners vuln \
                        ${FRONTEND_IMAGE} > frontend-trivy-scan.txt || true
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
                withKubeConfig([credentialsId: KUBECONFIG_CRED_ID]) {
                    script {
                        sh """
                        sed -i 's|image:.*backend.*|image: ${BACKEND_IMAGE}|g' k8s-manifests/backend-deployment.yaml
                        sed -i 's|image:.*frontend.*|image: ${FRONTEND_IMAGE}|g' k8s-manifests/frontend-deployment.yaml
                        """
                        // Use full path to microk8s kubectl
                        sh '/snap/bin/microk8s.kubectl apply -f k8s-manifests/'
                        sh '/snap/bin/microk8s.kubectl rollout status deployment/backend'
                        sh '/snap/bin/microk8s.kubectl rollout status deployment/frontend'
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true' 
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
