# DevSecOps Microservices Project

A complete DevSecOps implementation of a User Management System with security scanning, containerization, and Kubernetes deployment.

## 🏗️ Project Structure

```
devsecops-microservices-project/
├── frontend/
│   ├── index.html
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── backend/
│   ├── src/
│   │   └── server.js
│   ├── package.json
│   ├── Dockerfile
│   ├── .dockerignore
│   └── .env
├── k8s-manifests/
│   ├── mongodb-pvc.yaml
│   ├── mongodb-deployment.yaml
│   ├── backend-deployment.yaml
│   └── frontend-deployment.yaml
├── Jenkinsfile
├── docker-compose.yml
└── README.md
```

## 🚀 Tech Stack

- **Frontend**: HTML, CSS, JavaScript
- **Backend**: Node.js, Express.js
- **Database**: MongoDB with Persistent Volume
- **Containerization**: Docker
- **Orchestration**: Kubernetes
- **CI/CD**: Jenkins
- **Security Tools**: 
  - Trivy (Container Image Scanning)
  - OWASP ZAP (Dynamic Application Security Testing)
  - Git Secrets (Secret Detection)

## 📋 Prerequisites

- Docker & Docker Compose
- Kubernetes Cluster (Minikube/EKS/GKE/AKS)
- Jenkins Server
- kubectl CLI
- DockerHub Account

## 🛠️ Local Setup with Docker Compose

### 1. Clone Repository
```bash
git clone https://github.com/your-username/devsecops-microservices-project.git
cd devsecops-microservices-project
```

### 2. Start All Services
```bash
docker-compose up -d
```

### 3. Access Application
- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:5000
- **MongoDB**: localhost:27017

### 4. Stop Services
```bash
docker-compose down
```

## 🐳 Manual Docker Build & Push

### Build Images
```bash
# Build Frontend
cd frontend
docker build -t your-dockerhub-username/devsecops-frontend:latest .

# Build Backend
cd ../backend
docker build -t your-dockerhub-username/devsecops-backend:latest .
```

### Push to DockerHub
```bash
docker login
docker push your-dockerhub-username/devsecops-frontend:latest
docker push your-dockerhub-username/devsecops-backend:latest
```

## ☸️ Kubernetes Deployment

### 1. Update Image Names
Edit `k8s-manifests/*.yaml` files and replace:
```yaml
image: your-dockerhub-username/devsecops-frontend:latest
```

### 2. Deploy MongoDB (with PVC)
```bash
kubectl apply -f k8s-manifests/mongodb-pvc.yaml
kubectl apply -f k8s-manifests/mongodb-deployment.yaml
```

### 3. Deploy Backend
```bash
kubectl apply -f k8s-manifests/backend-deployment.yaml
```

### 4. Deploy Frontend
```bash
kubectl apply -f k8s-manifests/frontend-deployment.yaml
```

### 5. Check Status
```bash
kubectl get pods
kubectl get svc
kubectl get pvc
```

### 6. Access Application
```bash
# For Minikube
minikube service frontend

# For Cloud Providers (LoadBalancer)
kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

## 🔐 DevSecOps Pipeline (Jenkins)

### Jenkins Setup

1. **Install Required Plugins**:
   - Docker Pipeline
   - Kubernetes CLI
   - OWASP ZAP
   - HTML Publisher

2. **Add Credentials**:
   - `dockerhub-credentials`: DockerHub username/password
   - `kubeconfig`: Kubernetes config file

3. **Create Pipeline**:
   - New Item → Pipeline
   - Pipeline from SCM
   - Repository URL: Your Git repo
   - Script Path: `Jenkinsfile`

### Pipeline Stages

1. ✅ **Checkout Code** - Clone repository
2. 🔍 **Git Secret Scanning** - Detect hardcoded secrets
3. 🏗️ **Build Docker Images** - Frontend & Backend
4. 🔒 **Trivy Scanning** - Container vulnerability scan
5. 📤 **Push to DockerHub** - Upload images
6. 🚀 **Deploy to K8s** - Rolling deployment
7. 🕷️ **OWASP ZAP** - Dynamic security testing
8. 💉 **SQL Injection Testing** - Backend security validation

### Security Scanning

#### Trivy (Image Scanning)
```bash
trivy image your-dockerhub-username/devsecops-backend:latest
```

#### OWASP ZAP (Web App Scanning)
```bash
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://your-app-url
```

## 📊 API Endpoints

### Authentication
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login user

### Users (Protected - Requires JWT Token)
- `GET /api/users` - Get all users
- `GET /api/users/:id` - Get single user
- `PUT /api/users/:id` - Update user
- `DELETE /api/users/:id` - Delete user

### Health Check
- `GET /health` - Backend health status

## 🧪 Testing

### Manual Testing
```bash
# Register User
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"password123"}'

# Login
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Get Users (with token)
curl -X GET http://localhost:5000/api/users \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

### SQL Injection Testing
```bash
# Test vulnerable payloads
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"' OR '1'='1"}'
```

## 🔧 Troubleshooting

### MongoDB Connection Issues
```bash
kubectl logs -l app=mongodb
kubectl describe pod -l app=mongodb
```

### Backend Not Starting
```bash
kubectl logs -l app=backend
kubectl get events
```

### PVC Issues
```bash
kubectl get pvc
kubectl describe pvc mongodb-pvc
```

## 📈 Monitoring & Logs

```bash
# View pod logs
kubectl logs -f <pod-name>

# View all pod logs
kubectl logs -l app=backend --tail=100

# Check resource usage
kubectl top pods
kubectl top nodes
```

## 🔄 Rollback Deployment

```bash
kubectl rollout undo deployment/backend
kubectl rollout undo deployment/frontend
```

## 🛡️ Security Best Practices Implemented

✅ Non-root user in Docker containers  
✅ Secret management with Kubernetes Secrets  
✅ Image vulnerability scanning with Trivy  
✅ Dynamic security testing with OWASP ZAP  
✅ SQL injection prevention (parameterized queries)  
✅ JWT authentication  
✅ Password hashing with bcrypt  
✅ CORS protection  
✅ Health checks and readiness probes  
✅ Resource limits in K8s  

## 📝 Environment Variables

### Backend (.env)
```
PORT=5000
MONGO_URI=mongodb://mongodb:27017/userdb
JWT_SECRET=your-super-secret-jwt-key
```

## 👨‍💻 Author

Your Name - [Your LinkedIn/GitHub]

## 📄 License

MIT License

## 🙏 Acknowledgments

- Node.js & Express community
- Kubernetes documentation
- OWASP ZAP project
- Aqua Security (Trivy)
