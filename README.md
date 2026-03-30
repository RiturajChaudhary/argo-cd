# 🚀 CI/CD Pipeline with Jenkins + ArgoCD + AWS EC2 + Kubernetes

A complete end-to-end CI/CD pipeline for a **Node.js** application using **Jenkins**, **AWS EC2**, **Docker**, **ArgoCD**, and **Kubernetes (Minikube)** — fully automated with GitOps principles.

---

## 📌 Project Repositories

| Repo | Description |
|---|---|
| [App Source Code](https://github.com/RiturajChaudhary/argo-cd.git) | Node.js App + Dockerfile + Jenkinsfile |
| [K8s Manifests](https://github.com/RiturajChaudhary/k8s-config.git) | Kubernetes deployment.yaml |
| [DockerHub Image](https://hub.docker.com/r/rituraj4164/frontend-app) | frontend-app Docker image |

---

## 🏗️ Architecture

```
Developer pushes code
        ↓
GitHub (argo-cd repo)
        ↓
Jenkins Pipeline (polls every 1 min for new builds)
        ↓
SSH into AWS EC2 → Docker Build
        ↓
Push Image to DockerHub
        ↓
Update deployment.yaml (k8s-config repo)
        ↓
ArgoCD detects change
        ↓
Auto Deploy to Kubernetes (Minikube) ✅
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **Node.js 20 Alpine** | Application runtime |
| **Docker** | Containerization |
| **DockerHub** | Container registry |
| **AWS EC2** | Remote Docker build server |
| **Jenkins** | CI — polls GitHub every 1 min, builds, pushes, updates manifest |
| **ArgoCD** | CD — GitOps-based auto deployment |
| **Kubernetes (Minikube)** | Container orchestration on local machine |
| **GitHub** | Single source of truth for app code & k8s manifests |

---

## 🔑 Step 1: SSH Key Setup (Jenkins Server → AWS EC2)

> Goal: Allow Jenkins to SSH into EC2 without a password so it can copy code and run Docker commands remotely.

```bash
# 1. Generate SSH key pair on the Jenkins server
ssh-keygen -t rsa -b 4096

# 2. View the generated public key
cat ~/.ssh/id_rsa.pub
```

> Copy the output of `cat ~/.ssh/id_rsa.pub`

```bash
# 3. Connect to EC2 manually via SSH
ssh ec2-user@100.53.223.165

# 4. Once inside EC2, open the authorized_keys file
vi ~/.ssh/authorized_keys

# 5. Paste the copied public key here and save (Ctrl+X → Y → Enter)

# 6. Exit EC2
exit

# 7. Test passwordless SSH connection from Jenkins server
ssh ec2-user@100.53.223.165
```

> ✅ If it connects without asking for a password — SSH setup is complete!

---

## 🖥️ Step 2: AWS EC2 — Install Docker

> Goal: EC2 will be used as a remote Docker build server by Jenkins.

```bash
# Connect to EC2
ssh ec2-user@100.53.223.165

# Update packages
sudo yum update -y

# Install Docker
sudo yum install docker -y

# Start Docker service
sudo systemctl start docker

# Enable Docker to start on reboot
sudo systemctl enable docker

# Add ec2-user to docker group (so docker runs without sudo)
sudo usermod -aG docker ec2-user

# Verify Docker is installed
docker --version

# Exit EC2
exit
```

---

## 🐳 Step 3: Docker — Build & Push

> No manual Docker commands were run. Jenkins automatically SSHs into AWS EC2, copies the app code, builds the Docker image, logs into DockerHub, and pushes the image — all handled by the Jenkinsfile pipeline stages on every build trigger.

---

## ☸️ Step 4: Minikube — Start Kubernetes Cluster

```bash
# Start Minikube
minikube start

# Check Minikube is running
minikube status

# Get Minikube IP (used to access ArgoCD and app)
minikube ip
```

> In this project Minikube IP was: **192.168.49.2**

---

## 🔄 Step 5: ArgoCD — Install on Minikube

> Goal: ArgoCD watches the k8s-config GitHub repo and auto-deploys whenever deployment.yaml changes.

```bash
# Create dedicated namespace for ArgoCD
kubectl create namespace argocd

# Install ArgoCD using official manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Watch pods come up (wait until all show 1/1 Running)
kubectl get pods -n argocd -w
```

Expected all pods Running:
```
argocd-application-controller-0       1/1   Running
argocd-applicationset-controller      1/1   Running
argocd-dex-server                     1/1   Running
argocd-notifications-controller       1/1   Running
argocd-redis                          1/1   Running
argocd-repo-server                    1/1   Running
argocd-server                         1/1   Running
```

---

## 🌐 Step 6: Access ArgoCD UI

```bash
# Expose ArgoCD server using Minikube service
minikube service argocd-server -n argocd --url
```

> This prints two URLs — use the **second one (HTTPS port)**

```bash
# Get the default admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

Login to ArgoCD UI:
```
URL:      https://<url-from-above>
Username: admin
Password: <output from above command>
```

---

## 📋 Step 7: ArgoCD — Connect GitHub Repo & Create App

### Connect k8s-config Repo
```
Settings → Repositories → Connect Repo
  Type:           git
  Project:        default
  Repository URL: https://github.com/RiturajChaudhary/k8s-config.git
  Username:       RiturajChaudhary
  Password:       <GitHub Personal Access Token>
```

### Create Application
```
New App →
  Application Name:  frontend-app
  Project:           default
  Sync Policy:       Automatic
    ✅ Prune Resources
    ✅ Self Heal
  Repository URL:    https://github.com/RiturajChaudhary/k8s-config.git
  Revision:          main
  Path:              .
  Cluster URL:       https://kubernetes.default.svc
  Namespace:         default
```

> ✅ Once created ArgoCD will immediately sync and deploy the app

---

## ⚙️ Step 8: Jenkins — Pipeline Setup

### Jenkins Credentials to Add
```
Manage Jenkins → Credentials → Global → Add Credentials

1. GitHub Token
   Kind:     Username with password
   ID:       github-argo
   Username: RiturajChaudhary
   Password: <GitHub Personal Access Token with repo scope>

2. DockerHub Token
   Kind:     Username with password
   ID:       dockerhub-creds
   Username: rituraj4164
   Password: <DockerHub Access Token with Read+Write scope>
```

### Jenkins Pipeline Trigger — Poll SCM Every 1 Minute
```
Pipeline → Configure → Build Triggers
  ✅ Poll SCM
  Schedule: * * * * *
```

> This tells Jenkins to check GitHub **every 1 minute** for new commits and trigger a build automatically.

---

## 🔁 Full CI/CD Flow (Step by Step)

```
1.  Developer pushes code to GitHub (argo-cd repo)
2.  Jenkins polls GitHub every 1 minute
3.  Jenkins detects new commit → triggers pipeline
4.  Stage 1: Jenkins pulls latest app code from GitHub
5.  Stage 2: Jenkins SSHs into AWS EC2
            → Copies app code to EC2 via scp
            → Builds Docker image on EC2
            → Tags image with Jenkins BUILD_NUMBER
6.  Stage 3: Jenkins SSHs into EC2
            → Logs into DockerHub
            → Pushes image rituraj4164/frontend-app:<build-number>
7.  Stage 4: Jenkins checks out k8s-config repo
8.  Stage 5: Jenkins runs sed to update image tag in deployment.yaml
            → Commits change with git
            → Pushes to k8s-config GitHub repo
9.  ArgoCD detects new commit in k8s-config repo
10. ArgoCD pulls updated deployment.yaml
11. ArgoCD applies changes to Minikube cluster
12. Old pod terminated → New pod with latest image started ✅
```

---

## 📊 Port Reference

| Service | Port | URL |
|---|---|---|
| Jenkins | 8080 | `http://localhost:8080` |
| ArgoCD | NodePort (via minikube service) | `https://192.168.49.2:<port>` |
| Frontend App | 3000 | `http://192.168.49.2:<nodeport>` |

---

## ✅ Jenkins Credentials Summary

| Credential ID | Type | Scope |
|---|---|---|
| `github-argo` | Username + Password | GitHub token — repo scope |
| `dockerhub-creds` | Username + Password | DockerHub token — **Read + Write** scope |

---

## 👨‍💻 Author

**Rituraj Chaudhary**
- GitHub: [RiturajChaudhary](https://github.com/RiturajChaudhary)
- Email: manojchy4164@gmail.com

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).
