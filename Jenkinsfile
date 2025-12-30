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
                    // FIX: Added -v /var/run/docker.sock mount
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --no-progress --format table \
                        ${BACKEND_IMAGE}
                    """
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --no-progress --format table \
                        ${FRONTEND_IMAGE}
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                // withCredentials automatically logs out, but post always block has logout too
                withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIALS, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh "docker push ${BACKEND_IMAGE}"
                    sh "docker push ${FRONTEND_IMAGE}"
                }
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                // Ensure the "Kubernetes Continuous Deploy" plugin is installed
                withKubeConfig([credentialsId: KUBECONFIG_CRED_ID]) {
                    script {
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
    }

    post {
        always {
            // Check if logged in before trying to logout to avoid errors
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
