# 🚀 Complete Deployment Architecture Guide for Beginners

## 📋 Table of Contents
1. [Overview](#overview)
2. [Understanding Each Component](#understanding-each-component)
3. [Complete Deployment Flow](#complete-deployment-flow)
4. [Architecture Diagram](#architecture-diagram)
5. [Step-by-Step Execution](#step-by-step-execution)

---

## 🎯 Overview

This project uses a **GitOps CI/CD pipeline** to automatically deploy your application to Kubernetes whenever you push code to GitHub.

### Technologies Used:
- **Docker** - Packages your application into containers
- **Jenkins** - Automates the build and deployment process
- **Kubernetes (Minikube)** - Runs your containers in a cluster
- **ArgoCD** - Automatically syncs and deploys your app
- **GitHub** - Version control and triggers the pipeline

---

## 📦 Understanding Each Component

### 1. **Dockerfile** - The Application Container Blueprint

```dockerfile
FROM python:3.10-slim
```

**Purpose:** This file tells Docker how to package your application into a container.

**What it does:**
- **Base Image:** Starts with Python 3.10 (lightweight version)
- **Environment Variables:** 
  - `PYTHONDONTWRITEBYTECODE=1` - Don't create .pyc files (cleaner)
  - `PYTHONUNBUFFERED=1` - See logs in real-time
- **Working Directory:** Creates `/app` folder inside container
- **System Dependencies:** Installs build tools and curl
- **Copy Files:** Copies your entire project into the container
- **Install Dependencies:** Runs `pip install -e .` to install your Python packages
- **Expose Port:** Opens port 8501 (Streamlit default port)
- **Start Command:** Runs `streamlit run application.py` when container starts

**Think of it as:** A recipe that creates a ready-to-run version of your app

---

### 2. **Jenkinsfile** - The Automation Pipeline

This file defines **6 automated stages** that run every time you push code:

#### **Stage 1: Checkout GitHub**
```groovy
checkout scmGit(branches: [[name: '*/main']], ...)
```
- **What:** Downloads latest code from GitHub
- **Why:** Jenkins needs the latest version of your code to build

#### **Stage 2: Build Docker Image**
```groovy
dockerImage = docker.build("${DOCKER_HUB_REPO}:${IMAGE_TAG}")
```
- **What:** Creates a Docker image using your Dockerfile
- **Image Name:** `dataguru97/studybuddy:v1`, `v2`, `v3`, etc.
- **Tag:** Uses Jenkins build number (e.g., v17, v18)
- **Why:** Each build gets a unique version number

#### **Stage 3: Push to DockerHub**
```groovy
dockerImage.push("${IMAGE_TAG}")
```
- **What:** Uploads the Docker image to DockerHub (like GitHub but for containers)
- **Credentials:** Uses stored DockerHub token
- **Why:** Kubernetes will pull this image to run your app

#### **Stage 4: Update Deployment YAML**
```groovy
sed -i 's|image: dataguru97/studybuddy:.*|image: dataguru97/studybuddy:${IMAGE_TAG}|' manifests/deployment.yaml
```
- **What:** Automatically updates the image version in deployment.yaml
- **Example:** Changes `v16` → `v17` in the YAML file
- **Why:** Tells Kubernetes which version to deploy

#### **Stage 5: Commit Updated YAML**
```groovy
git add manifests/deployment.yaml
git commit -m "Update image tag to v17"
git push
```
- **What:** Pushes the updated deployment.yaml back to GitHub
- **Why:** ArgoCD watches GitHub for changes - this triggers deployment

#### **Stage 6: Install kubectl & ArgoCD CLI + Sync**
```groovy
argocd app sync study
```
- **What:** 
  - Installs command-line tools
  - Logs into ArgoCD
  - Triggers immediate sync of your application
- **Why:** Ensures Kubernetes immediately deploys the new version

---

### 3. **manifests/deployment.yaml** - Kubernetes Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llmops-app
spec:
  replicas: 2
```

**Purpose:** Tells Kubernetes HOW to run your application

**Key Sections:**

#### **Replicas: 2**
- Runs **2 copies** of your app simultaneously
- **Benefits:**
  - High availability (if one crashes, the other handles requests)
  - Load balancing (distributes traffic)

#### **Container Specifications:**
```yaml
containers:
- name: llmops-app
  image: dataguru97/studybuddy:v17
  ports:
  - containerPort: 8501
```
- **Image:** Which Docker image to use (updated by Jenkins)
- **Port:** Your Streamlit app listens on port 8501

#### **Environment Variables (Secrets):**
```yaml
env:
- name: GROQ_API_KEY
  valueFrom:
    secretKeyRef:
      name: groq-api-secret
      key: GROQ_API_KEY
```
- **What:** Securely injects your API key into the container
- **Why:** Keeps secrets out of code (never hardcode API keys!)
- **How:** Kubernetes pulls from a Secret object you created

**Think of it as:** Instructions for Kubernetes to create and manage your app containers

---

### 4. **manifests/service.yaml** - Network Access Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: llmops-service
spec:
  selector:
    app: llmops-app
  ports:
    - port: 80
      targetPort: 8501
  type: NodePort
```

**Purpose:** Exposes your application to the network

**Key Sections:**

#### **Selector:**
```yaml
selector:
  app: llmops-app
```
- **What:** Finds all pods with label `app: llmops-app`
- **Why:** Connects the service to your deployment

#### **Port Mapping:**
```yaml
ports:
  - port: 80          # External access port
    targetPort: 8501  # Container internal port
```
- **Port 80:** Users access via `http://your-ip:80`
- **TargetPort 8501:** Forwards to Streamlit inside container

#### **Service Type: NodePort**
- **What:** Makes your app accessible from outside the cluster
- **How:** Kubernetes assigns a random port (30000-32767)
- **Access:** `http://VM_EXTERNAL_IP:NodePort`

**Think of it as:** A load balancer that routes traffic to your app containers

---

## 🔄 Complete Deployment Flow

### **The Automated GitOps Workflow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOU PUSH CODE TO GITHUB                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (Webhook triggers Jenkins)
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  JENKINS PIPELINE STARTS AUTOMATICALLY                          │
├─────────────────────────────────────────────────────────────────┤
│  Stage 1: Clone code from GitHub                                │
│  Stage 2: Build Docker image (dataguru97/studybuddy:v17)       │
│  Stage 3: Push image to DockerHub                               │
│  Stage 4: Update deployment.yaml with new version               │
│  Stage 5: Commit & push updated YAML back to GitHub            │
│  Stage 6: Tell ArgoCD to sync the application                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (GitHub updated)
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  ARGOCD DETECTS CHANGES IN GITHUB                               │
├─────────────────────────────────────────────────────────────────┤
│  • Reads manifests/deployment.yaml                              │
│  • Sees new image version (v17)                                 │
│  • Compares with current cluster state                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (Sync application)
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER DEPLOYMENT                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. Pull new Docker image from DockerHub                        │
│  2. Create 2 new pods with v17 image                            │
│  3. Wait for pods to be healthy (readiness checks)              │
│  4. Terminate old pods running v16                              │
│  5. Service routes traffic to new pods                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  APPLICATION LIVE AND ACCESSIBLE                                │
│  http://VM_EXTERNAL_IP:NodePort                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GOOGLE CLOUD VM                               │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    DOCKER CONTAINERS                            │  │
│  │  ┌──────────────┐         ┌────────────────────────────────┐  │  │
│  │  │   JENKINS    │         │     MINIKUBE CLUSTER           │  │  │
│  │  │   (CI/CD)    │────────▶│                                │  │  │
│  │  │   Port 8080  │         │  ┌──────────────────────────┐  │  │  │
│  │  └──────────────┘         │  │    ARGOCD NAMESPACE      │  │  │  │
│  │                           │  │  ┌────────────────────┐  │  │  │  │
│  │                           │  │  │   ArgoCD Server    │  │  │  │  │
│  │                           │  │  │   Port: 31704      │  │  │  │  │
│  │                           │  │  └────────────────────┘  │  │  │  │
│  │                           │  │  ┌────────────────────┐  │  │  │  │
│  │                           │  │  │  Your App Pods     │  │  │  │  │
│  │                           │  │  │  • Pod 1 (v17)     │  │  │  │  │
│  │                           │  │  │  • Pod 2 (v17)     │  │  │  │  │
│  │                           │  │  │  Port: 8501        │  │  │  │  │
│  │                           │  │  └────────────────────┘  │  │  │  │
│  │                           │  │  ┌────────────────────┐  │  │  │  │
│  │                           │  │  │  Service           │  │  │  │  │
│  │                           │  │  │  llmops-service    │  │  │  │  │
│  │                           │  │  │  Port: 80→8501     │  │  │  │  │
│  │                           │  │  └────────────────────┘  │  │  │  │
│  │                           │  └──────────────────────────┘  │  │  │
│  │                           └────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             │ Pulls images from
                             ▼
                    ┌─────────────────┐
                    │   DOCKERHUB     │
                    │  Repository:    │
                    │  dataguru97/    │
                    │  studybuddy     │
                    └─────────────────┘
                             ▲
                             │ Pushes images
                             │
                    ┌─────────────────┐
                    │     GITHUB      │
                    │   Repository    │
                    │   • Code        │
                    │   • Jenkinsfile │
                    │   • Manifests   │
                    └─────────────────┘
                             ▲
                             │
                        (You push code)
```

---

## 📝 Step-by-Step Execution (What Actually Happens)

### **Scenario: You Fix a Bug and Push Code**

#### **Step 1: Developer Action**
```bash
git add .
git commit -m "Fixed bug in question generator"
git push origin main
```
- You push code to GitHub
- GitHub webhook immediately notifies Jenkins

---

#### **Step 2: Jenkins Wakes Up**
- Jenkins receives webhook notification
- Starts pipeline automatically (no manual trigger needed)
- Console output shows: `Started by GitHub push`

---

#### **Step 3: Code Checkout (5 seconds)**
```
[Jenkins] Checking out code from GitHub...
Cloning repository: https://github.com/data-guru0/STUDY-BUDDY-AI.git
✓ Checked out branch: main
```

---

#### **Step 4: Docker Image Build (2-3 minutes)**
```
[Jenkins] Building Docker image...
Step 1/8: FROM python:3.10-slim
Step 2/8: ENV PYTHONDONTWRITEBYTECODE=1
Step 3/8: WORKDIR /app
Step 4/8: RUN apt-get update...
Step 5/8: COPY . .
Step 6/8: RUN pip install -e .
Step 7/8: EXPOSE 8501
Step 8/8: CMD ["streamlit", "run", "application.py"]
✓ Successfully built dataguru97/studybuddy:v17
```

---

#### **Step 5: Push to DockerHub (1 minute)**
```
[Jenkins] Pushing image to DockerHub...
The push refers to repository [docker.io/dataguru97/studybuddy]
v17: digest: sha256:abc123... size: 2841
✓ Image pushed successfully
```

---

#### **Step 6: Update Deployment File (2 seconds)**
```
[Jenkins] Updating manifests/deployment.yaml...
Changed: image: dataguru97/studybuddy:v16
To:      image: dataguru97/studybuddy:v17
✓ File updated
```

---

#### **Step 7: Commit Back to GitHub (5 seconds)**
```
[Jenkins] Committing updated YAML...
git add manifests/deployment.yaml
git commit -m "Update image tag to v17"
git push origin main
✓ Changes pushed to GitHub
```

---

#### **Step 8: Install Tools (30 seconds)**
```
[Jenkins] Installing kubectl and ArgoCD CLI...
✓ kubectl installed
✓ argocd CLI installed
```

---

#### **Step 9: ArgoCD Login & Sync (10 seconds)**
```
[Jenkins] Logging into ArgoCD...
✓ Logged in to 34.45.193.5:31704

[Jenkins] Syncing application...
argocd app sync study
✓ Application synced successfully
```

---

#### **Step 10: Kubernetes Deployment (30-60 seconds)**

**ArgoCD detects changes and applies them:**

```
[ArgoCD] Syncing application 'study'
[ArgoCD] Comparing desired state (GitHub) vs actual state (Cluster)
[ArgoCD] Change detected: image v16 → v17

[Kubernetes] Starting deployment...
[Kubernetes] Pulling image dataguru97/studybuddy:v17
[Kubernetes] Creating new pods...
  • llmops-app-7d8f9c5b-xk2ln: ContainerCreating → Running (✓)
  • llmops-app-7d8f9c5b-mp9qw: ContainerCreating → Running (✓)

[Kubernetes] Health checks passed
[Kubernetes] Terminating old pods...
  • llmops-app-6c7d8b4a-abc12: Terminating → Terminated
  • llmops-app-6c7d8b4a-def34: Terminating → Terminated

[Service] Routing traffic to new pods
✓ Deployment successful
```

---

#### **Step 11: Access Your Application**

**Option A: Port Forwarding (Temporary)**
```bash
kubectl port-forward svc/llmops-service -n argocd --address 0.0.0.0 9090:80
```
Access: `http://VM_EXTERNAL_IP:9090`

**Option B: Minikube Tunnel (Persistent)**
```bash
minikube tunnel
```
Access via NodePort assigned by Kubernetes

---

## 🔍 Key Concepts Explained

### **1. Why DockerHub?**
- Kubernetes can't access files on Jenkins
- DockerHub is a centralized registry
- Kubernetes pulls images from DockerHub (like npm or pip for containers)

### **2. Why Update deployment.yaml?**
- GitOps principle: GitHub is the "source of truth"
- ArgoCD watches GitHub, not Jenkins
- Updating GitHub triggers ArgoCD to deploy

### **3. Why ArgoCD?**
- **Continuous Deployment:** Automatically deploys when GitHub changes
- **Self-Healing:** If someone manually changes Kubernetes, ArgoCD reverts it
- **Rollback:** Easy to revert to previous versions via GitHub

### **4. Why 2 Replicas?**
- **High Availability:** App stays online even if 1 pod crashes
- **Load Balancing:** Traffic distributed across 2 pods
- **Zero Downtime:** Rolling updates (new pods start before old ones stop)

### **5. Why NodePort Service?**
- **Development/Testing:** Easy external access
- **Production Alternative:** Use LoadBalancer or Ingress for real deployments

---

## 🛡️ Security Best Practices

### **1. Secrets Management**
```bash
kubectl create secret generic groq-api-secret \
  --from-literal=GROQ_API_KEY="your-actual-key" \
  -n argocd
```
- **Never** commit API keys to GitHub
- Always use Kubernetes Secrets
- Reference secrets in deployment.yaml

### **2. Jenkins Credentials**
- GitHub token stored in Jenkins (not in Jenkinsfile)
- DockerHub token stored in Jenkins
- Kubeconfig stored as secret file

### **3. Network Security**
- Jenkins: Port 8080 (should be restricted to your IP)
- ArgoCD: Port 31704 (should be behind VPN in production)
- Application: Behind service (controlled access)

---

## 🚨 Troubleshooting Common Issues

### **Issue 1: Jenkins Pipeline Fails at Docker Build**
```
Error: Cannot connect to Docker daemon
```
**Solution:**
```bash
docker exec -it jenkins bash
docker ps  # Test Docker access
```
Ensure Jenkins container has Docker socket mounted

---

### **Issue 2: Image Pull Error in Kubernetes**
```
ErrImagePull: Failed to pull image "dataguru97/studybuddy:v17"
```
**Solution:**
1. Check if image exists on DockerHub
2. Verify image tag matches deployment.yaml
3. Check Jenkins push logs

---

### **Issue 3: ArgoCD Sync Fails**
```
ComparisonError: Failed to sync application
```
**Solution:**
```bash
# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-server

# Manually sync from UI
# ArgoCD UI → Application → Sync
```

---

### **Issue 4: Pods in CrashLoopBackOff**
```
CrashLoopBackOff: Container keeps restarting
```
**Solution:**
```bash
# Check pod logs
kubectl logs -n argocd pod/llmops-app-xxx

# Common causes:
# - Missing environment variables (GROQ_API_KEY)
# - Application errors in code
# - Wrong port configuration
```

---

## 📊 Monitoring Your Deployment

### **Check Jenkins Build Status**
```
Jenkins Dashboard → Pipeline → Build History
✓ Green = Success
✗ Red = Failed (check console output)
```

### **Check ArgoCD Application Status**
```
ArgoCD UI → Applications → study
Health: Healthy
Sync Status: Synced
```

### **Check Kubernetes Resources**
```bash
# Check pods
kubectl get pods -n argocd

# Check deployments
kubectl get deployments -n argocd

# Check services
kubectl get svc -n argocd

# Check events (for debugging)
kubectl get events -n argocd --sort-by='.lastTimestamp'
```

---

## 🎓 Summary for Beginners

### **What You Built:**
A fully automated CI/CD pipeline where:
1. **You push code** → Jenkins automatically builds and deploys
2. **No manual steps** → Everything is automated
3. **Always in sync** → GitHub is the source of truth
4. **Self-healing** → ArgoCD ensures desired state
5. **Scalable** → Can easily increase replicas

### **Key Files:**
- **Dockerfile:** How to build your app container
- **Jenkinsfile:** Automation pipeline (CI/CD)
- **deployment.yaml:** How Kubernetes runs your app
- **service.yaml:** How users access your app

### **Flow:**
```
Code Push → Jenkins Build → DockerHub → GitHub Update → ArgoCD Sync → Kubernetes Deploy → Live App
```

### **Benefits:**
✅ Automated deployments
✅ Version control for infrastructure
✅ Easy rollbacks
✅ Consistent environments
✅ Professional DevOps practices

---

## 🚀 Next Steps

1. **Add Health Checks:** Implement liveness and readiness probes
2. **Resource Limits:** Set CPU and memory limits in deployment.yaml
3. **Ingress:** Replace NodePort with proper domain name
4. **Monitoring:** Add Prometheus and Grafana
5. **Logging:** Centralize logs with ELK stack
6. **Multi-Environment:** Create dev, staging, prod pipelines

---

**Congratulations! You now understand the complete deployment architecture! 🎉**
