# 🚀 Book My Show Clone – DevOps Project

Welcome to the **Book My Show App Deployment** project!  
This guide walks you through deploying a Book My Show-clone app using modern DevOps and DevSecOps practices.

---

## 👤 Author

**Mohammed CHERKAOUI**

---

## 🛠️ Tools & Services Used

| Category           | Tools                                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------|
| Version Control    | ![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)    |
| CI/CD              | ![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white) |
| Code Quality       | ![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=flat-square&logo=sonarqube&logoColor=white) |
| Containerization   | ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)    |
| Orchestration      | ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white) |
| Monitoring         | ![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white) ![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white) |
| Security           | ![OWASP](https://img.shields.io/badge/OWASP-000000?style=flat-square&logo=owasp&logoColor=white) ![Trivy](https://img.shields.io/badge/Trivy-00979D?style=flat-square&logo=trivy&logoColor=white) |

---

## 📦 Project Structure

- **Part I:** Docker Deployment
- **Part II:** Kubernetes (EKS) Deployment & Monitoring

---

## 🚦 Project Stages

### **Part I: Docker Deployment**

#### 1. Infrastructure Setup
- Launch Ubuntu VM (e.g., t2.large, 28GB RAM, Ubuntu 24.04).
- Open required ports: 22 (SSH), 80/443 (HTTP/HTTPS), 8080 (Jenkins), 9000 (SonarQube), 3000 (App), 6443 (K8s API), 30000-32767 (K8s NodePort), etc.

#### 2. Tools Installation
- **Jenkins:** Install via script, open port 8080.
- **Docker:** Install via script, test with `docker pull hello-world`.
- **Trivy:** Install for vulnerability scanning.
- **SonarQube:** Run via Docker on port 9000.

#### 3. Jenkins Setup
- Install plugins: SonarQube Scanner, NodeJS, Docker, Kubernetes, Prometheus, etc.
- Configure credentials for GitHub, DockerHub, SonarQube, and email.
- Set up SonarQube token and Jenkins system configuration.

#### 4. Email Integration
- Configure Gmail App Password for Jenkins email notifications.
- Set up SMTP in Jenkins for build alerts.

#### 5. Pipeline Job
- Create a Jenkins pipeline for:
  - Code checkout
  - SonarQube analysis & quality gate
  - Dependency install
  - Trivy scan
  - Docker build & push
  - Container deployment
  - Email notifications

<details>
<summary>Sample Jenkins Pipeline (Docker Only)</summary>

```groovy
pipeline {
    agent any
    tools { jdk 'jdk17'; nodejs 'node23' }
    environment { SCANNER_HOME = tool 'sonar-scanner' }
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }
        stage('Checkout from Git') { steps { git branch: 'main', url: 'https://github.com/KastroVKiran/Book-My-Show.git' } }
        stage('SonarQube Analysis') { steps { withSonarQubeEnv('sonar-server') { sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BMS -Dsonar.projectKey=BMS' } } }
        stage('Quality Gate') { steps { script { waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' } } }
        stage('Install Dependencies') { steps { sh 'cd bookmyshow-app && npm install' } }
        stage('Trivy FS Scan') { steps { sh 'trivy fs . > trivyfs.txt' } }
        stage('Docker Build & Push') { steps { script { withDockerRegistry(credentialsId: 'docker', toolName: 'docker') { sh 'docker build -t kastrov/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app && docker push kastrov/bms:latest' } } } }
        stage('Deploy to Container') { steps { sh 'docker stop bms || true && docker rm bms || true && docker run -d --restart=always --name bms -p 3000:3000 kastrov/bms:latest' } }
    }
    post {
        always {
            emailext attachLog: true, subject: "'${currentBuild.result}'", body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}<br/>", to: 'kastrokiran@gmail.com', attachmentsPattern: 'trivyfs.txt'
        }
    }
}
```
</details>

---

### **Part II: Kubernetes (EKS) Deployment & Monitoring**

#### 1. AWS & EKS Setup
- Create IAM user with EKS permissions and access keys.
- Install AWS CLI, kubectl, eksctl on your VM.
- Create EKS cluster and node group using `eksctl`.
- Configure kubectl context for EKS.

#### 2. Jenkins Pipeline (with K8s Stage)
- Add EKS deployment stage to pipeline:
  - Update kubeconfig
  - Deploy app using `kubectl apply -f deployment.yml` and `service.yml`
  - Verify deployment and service

<details>
<summary>Sample Jenkins Pipeline (With K8s Stage)</summary>

```groovy
pipeline {
    agent any
    tools { jdk 'jdk17'; nodejs 'node23' }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'kastrov/bms:latest'
        EKS_CLUSTER_NAME = 'kastro-eks'
        AWS_REGION = 'us-east-1'
    }
    stages {
        // ...previous stages...
        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    sh '''
                    aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }
    post {
        always {
            emailext attachLog: true, subject: "'${currentBuild.result}'", body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}<br/>", to: 'kastrokiran@gmail.com', attachmentsPattern: 'trivyfs.txt'
        }
    }
}
```
</details>

#### 3. Monitoring Setup

- **Prometheus:**
  - Create a dedicated VM for monitoring.
  - Install Prometheus and Node Exporter as system services.
  - Configure Prometheus to scrape Node Exporter and Jenkins metrics.

- **Grafana:**
  - Install Grafana on the monitoring VM.
  - Add Prometheus as a data source.
  - Import dashboards for Node Exporter and Jenkins.

---

## 📂 Code Repository

[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/mohammed761-dl/Book-My-Show)

---

## 📣 Feedback

After deploying the app, share your feedback on LinkedIn! Tag me and include the project link.

---

## 🎉 Happy Learning!

---
**Kastro Kiran V**
