# 🚀 DevOps CI/CD Pipeline Project

> End-to-end CI/CD pipeline using Jenkins, Docker, SonarQube, and Kubernetes (K3s) on AWS EC2

---

## 📐 Architecture Overview

```
Developer → GitHub → Jenkins Pipeline
                          │
              ┌───────────┼───────────────┐
              ▼           ▼               ▼
           Maven       SonarQube       Docker
           Build        Scan           Build
              │                          │
              └──────────────────────────┘
                              │
                         DockerHub
                              │
                     K3s Kubernetes (EC2)
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
               Deployment           Service
               (2 replicas)        (NodePort)
```

---

## 🛠️ Tech Stack

| Tool        | Purpose                        |
|-------------|--------------------------------|
| GitHub      | Source code management         |
| Jenkins     | CI/CD automation               |
| Maven       | Java build & dependency mgmt   |
| SonarQube   | Code quality & security scan   |
| Docker      | Containerization               |
| DockerHub   | Container image registry       |
| K3s         | Lightweight Kubernetes cluster |
| Helm        | Kubernetes package management  |
| AWS EC2     | Cloud infrastructure           |

---

## ☁️ Infrastructure Setup

### EC2 Instance
- **OS:** Ubuntu 22.04 LTS
- **Type:** t2.large
- **Storage:** 30 GB

### Security Group — Open Ports

| Port        | Purpose              |
|-------------|----------------------|
| 22          | SSH                  |
| 8080        | Jenkins              |
| 9000        | SonarQube            |
| 6443        | Kubernetes API       |
| 30000-32767 | Kubernetes NodePort  |

---

## ⚙️ Installation Guide

### Step 1 — System Packages

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk docker.io curl git

# Add users to docker group
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
```

### Step 2 — Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update && sudo apt install jenkins -y
sudo systemctl enable jenkins && sudo systemctl start jenkins
```

Access Jenkins: `http://<EC2-IP>:8080`

### Step 3 — Install K3s (Kubernetes)

```bash
curl -sfL https://get.k3s.io | sh -

# Configure kubectl access
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
```

### Step 4 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Step 5 — Run SonarQube

```bash
docker run -d \
  --name sonar \
  -p 9000:9000 \
  sonarqube:lts-community
```

Access SonarQube: `http://<EC2-IP>:9000`  
Default credentials: `admin / admin`

---

## 🔌 Jenkins Configuration

### Required Plugins
- Pipeline
- Docker Pipeline
- Git
- Maven Integration
- SonarQube Scanner
- Kubernetes CLI
- Credentials Binding

### Credentials to Add

| Credential ID     | Type                | Purpose              |
|-------------------|---------------------|----------------------|
| `dockerhub-creds` | Username/Password   | DockerHub login      |
| `kubeconfig-file` | Secret File         | K3s kubeconfig       |
| `aws-creds`       | AWS Credentials     | AWS access           |
| `sonar-token`     | Secret Text         | SonarQube auth token |

### Global Tool Configuration
Navigate to **Manage Jenkins → Global Tool Configuration** and add:
- JDK 17
- Maven 3
- SonarQube Scanner

---

## 📁 Project Structure

```
.
├── spring-boot-app/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── spring-boot-app-manifests/
│   ├── deployment.yml
│   └── service.yml
├── helm/
│   └── java-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
└── Jenkinsfile
```

---

## 🐳 Dockerfile

```dockerfile
FROM openjdk:17
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

## ☸️ Kubernetes Manifests

### deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  labels:
    app: spring-boot-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: spring-boot-app
        image: prasanna369/ultimate-cicd:IMAGE_TAG   # replaced by pipeline sed
        ports:
        - containerPort: 8080
```

### service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  type: NodePort
  selector:
    app: spring-boot-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30007
```

---

## 🔧 Jenkins Pipeline

```groovy
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        IMAGE              = "prasanna369/ultimate-cicd"
        AWS_DEFAULT_REGION = "us-east-1"
    }
    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/mhprasanna-spec/DevOps-Project-18.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -f spring-boot-app/pom.xml clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn -f spring-boot-app/pom.xml sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    dir('spring-boot-app') {
                        sh """
                            docker build -t ${IMAGE}:${BUILD_NUMBER} .
                            docker tag  ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )
                ]) {
                    sh """
                        echo \$PASS | docker login -u \$USER --password-stdin
                        docker push ${IMAGE}:${BUILD_NUMBER}
                        docker push ${IMAGE}:latest
                        docker rmi ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest || true
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    script {
                        env.KUBECONFIG = KUBECONFIG_FILE
                        sh """
                            cp spring-boot-app-manifests/deployment.yml /tmp/deployment-${BUILD_NUMBER}.yml
                            sed -i "s|IMAGE_TAG|${BUILD_NUMBER}|g" /tmp/deployment-${BUILD_NUMBER}.yml
                            grep "image:" /tmp/deployment-${BUILD_NUMBER}.yml
                            kubectl apply -f /tmp/deployment-${BUILD_NUMBER}.yml
                            kubectl apply -f spring-boot-app-manifests/service.yml
                            kubectl rollout status deployment/spring-boot-app --timeout=120s
                            rm -f /tmp/deployment-${BUILD_NUMBER}.yml
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build #${BUILD_NUMBER} deployed — ${IMAGE}:${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Build #${BUILD_NUMBER} failed"
        }
        always {
            cleanWs()
        }
    }
}
```

---

## 🪖 Helm Deployment (Alternative)

```bash
# Create chart
helm create java-app

# Install
helm install java-app ./java-app

# Upgrade on new build
helm upgrade java-app ./java-app \
  --set image.tag=$BUILD_NUMBER
```

---

## ✅ Verify Deployment

```bash
# Check pods
kubectl get pods

# Check service
kubectl get svc

# Access app
curl http://<EC2-IP>:30007

# Watch rollout
kubectl rollout status deployment/spring-boot-app
```

---

## 🐛 Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | `IMAGE_TAG` not replaced | Ensure `sed` runs before `kubectl apply` |
| `amazonWebServices` DSL error | Wrong plugin / method name | Use `AmazonWebServicesCredentialsBinding` class |
| SonarQube gate timeout | Analysis took too long | Increase timeout or check SonarQube logs |
| Pods not starting | OOM / resource limits | Increase EC2 instance size or set resource requests |
| Jenkins can't reach K3s | KUBECONFIG not set | Ensure `env.KUBECONFIG = KUBECONFIG_FILE` is set in pipeline |

---

## 📊 Pipeline Flow Summary

```
Code Push
    │
    ▼
Jenkins Triggered
    │
    ├─► Checkout from GitHub
    ├─► Maven Build (JAR)
    ├─► SonarQube Scan
    ├─► Quality Gate Check ──► FAIL → Pipeline Aborts
    ├─► Docker Build & Tag
    ├─► Push to DockerHub
    └─► Deploy to K3s
              │
              ├─► sed replaces IMAGE_TAG → BUILD_NUMBER
              ├─► kubectl apply deployment + service
              └─► Rollout status verified ──► FAIL → Pipeline Fails
```

---

## 👨‍💻 Author

**Prasanna**  
DevOps Engineer  
GitHub: [@mhprasanna-spec](https://github.com/mhprasanna-spec)

---

## 📄 License

MIT License — free to use for learning and portfolio purposes.
