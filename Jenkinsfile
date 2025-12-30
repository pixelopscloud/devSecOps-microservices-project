pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub-credentials') // Add in Jenkins
        DOCKER_HUB_USERNAME = 'pixelopscloud' // Change this
        FRONTEND_IMAGE = "${DOCKER_HUB_USERNAME}/devsecops-frontend"
        BACKEND_IMAGE = "${DOCKER_HUB_USERNAME}/devsecops-backend"
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = credentials('kubeconfig') // Add in Jenkins
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                script {
                    echo '📦 Checking out code from Git...'
                    checkout scm
                }
            }
        }
        
        stage('Git Secret Scanning') {
            steps {
                script {
                    echo '🔍 Scanning for secrets in repository...'
                    sh '''
                        # Install git-secrets if not present
                        if ! command -v git-secrets &> /dev/null; then
                            echo "Installing git-secrets..."
                            git clone https://github.com/awslabs/git-secrets.git /tmp/git-secrets
                            cd /tmp/git-secrets
                            sudo make install
                        fi
                        
                        # Scan repository
                        git secrets --scan || echo "⚠️ Warning: Potential secrets found!"
                    '''
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        script {
                            echo '🏗️ Building Frontend Docker Image...'
                            sh """
                                cd frontend
                                docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
                                docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                            """
                        }
                    }
                }
                
                stage('Build Backend') {
                    steps {
                        script {
                            echo '🏗️ Building Backend Docker Image...'
                            sh """
                                cd backend
                                docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} .
                                docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Trivy - Container Image Scanning') {
            parallel {
                stage('Scan Frontend Image') {
                    steps {
                        script {
                            echo '🔒 Scanning Frontend Image with Trivy...'
                            sh """
                                # Install Trivy if not present
                                if ! command -v trivy &> /dev/null; then
                                    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
                                    echo "deb https://aquasecurity.github.io/trivy-repo/deb \$(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
                                    sudo apt-get update
                                    sudo apt-get install trivy -y
                                fi
                                
                                # Scan image
                                trivy image --severity HIGH,CRITICAL ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                                trivy image --format json --output frontend-trivy-report.json ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            """
                            archiveArtifacts artifacts: 'frontend-trivy-report.json', allowEmptyArchive: true
                        }
                    }
                }
                
                stage('Scan Backend Image') {
                    steps {
                        script {
                            echo '🔒 Scanning Backend Image with Trivy...'
                            sh """
                                trivy image --severity HIGH,CRITICAL ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                                trivy image --format json --output backend-trivy-report.json ${BACKEND_IMAGE}:${IMAGE_TAG}
                            """
                            archiveArtifacts artifacts: 'backend-trivy-report.json', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo '📤 Pushing images to Docker Hub...'
                    sh """
                        echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin
                        
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                        
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest
                        
                        docker logout
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo '🚀 Deploying to Kubernetes cluster...'
                    sh """
                        # Apply MongoDB first (with PVC)
                        kubectl apply -f k8s-manifests/mongodb-pvc.yaml
                        kubectl apply -f k8s-manifests/mongodb-deployment.yaml
                        
                        # Wait for MongoDB to be ready
                        kubectl wait --for=condition=ready pod -l app=mongodb --timeout=120s
                        
                        # Update image tags in deployments
                        kubectl set image deployment/backend backend=${BACKEND_IMAGE}:${IMAGE_TAG}
                        kubectl set image deployment/frontend frontend=${FRONTEND_IMAGE}:${IMAGE_TAG}
                        
                        # Apply backend and frontend
                        kubectl apply -f k8s-manifests/backend-deployment.yaml
                        kubectl apply -f k8s-manifests/frontend-deployment.yaml
                        
                        # Wait for deployments
                        kubectl rollout status deployment/backend
                        kubectl rollout status deployment/frontend
                        
                        echo "✅ Deployment successful!"
                        kubectl get pods
                        kubectl get svc
                    """
                }
            }
        }
        
        stage('OWASP ZAP - Dynamic Security Testing') {
            steps {
                script {
                    echo '🕷️ Running OWASP ZAP Security Scan...'
                    sh """
                        # Get frontend service URL
                        FRONTEND_URL=\$(kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}' || echo "localhost")
                        
                        # Run ZAP Docker container
                        docker run --rm -v \$(pwd):/zap/wrk/:rw \
                            -t ghcr.io/zaproxy/zaproxy:stable \
                            zap-baseline.py \
                            -t http://\${FRONTEND_URL} \
                            -r zap-report.html \
                            -J zap-report.json || true
                    """
                    archiveArtifacts artifacts: 'zap-report.*', allowEmptyArchive: true
                    publishHTML(target: [
                        reportDir: '.',
                        reportFiles: 'zap-report.html',
                        reportName: 'OWASP ZAP Security Report'
                    ])
                }
            }
        }
        
        stage('SQL Injection Testing') {
            steps {
                script {
                    echo '💉 Testing for SQL Injection vulnerabilities...'
                    sh """
                        # Get backend service endpoint
                        BACKEND_URL=\$(kubectl get svc backend -o jsonpath='{.spec.clusterIP}'):5000
                        
                        # Test SQL injection patterns
                        echo "Testing SQL Injection patterns..."
                        
                        # Test login endpoint with SQL injection payloads
                        curl -X POST http://\${BACKEND_URL}/api/auth/login \
                            -H "Content-Type: application/json" \
                            -d '{"email":"admin@test.com","password":"' OR '1'='1"}' || true
                        
                        curl -X POST http://\${BACKEND_URL}/api/auth/login \
                            -H "Content-Type: application/json" \
                            -d '{"email":"admin@test.com' OR 1=1--","password":"test"}' || true
                        
                        echo "✅ SQL Injection tests completed (check logs for vulnerabilities)"
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Cleaning up Docker images...'
            sh '''
                docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} || true
            '''
        }
        success {
            echo '✅ Pipeline completed successfully!'
            emailext (
                subject: "✅ Jenkins Pipeline Success - Build #${BUILD_NUMBER}",
                body: "The DevSecOps pipeline completed successfully. Check console output for details.",
                to: 'your-email@example.com'
            )
        }
        failure {
            echo '❌ Pipeline failed!'
            emailext (
                subject: "❌ Jenkins Pipeline Failed - Build #${BUILD_NUMBER}",
                body: "The DevSecOps pipeline failed. Please check the logs.",
                to: 'your-email@example.com'
            )
        }
    }
}
