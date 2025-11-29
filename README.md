# MIND - CI Pipeline

[![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939.svg)](https://www.jenkins.io/)
[![Docker](https://img.shields.io/badge/Docker-Hub-2496ED.svg)](https://hub.docker.com/)
[![Trivy](https://img.shields.io/badge/Security-Trivy-4B89DC.svg)](https://aquasecurity.github.io/trivy/)

> Continuous Integration pipeline for building, scanning, and pushing Docker images to Docker Hub.

## 📋 Table of Contents

- [Overview](#overview)
- [Pipeline Flow](#pipeline-flow)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Pipeline Stages](#pipeline-stages)
- [Configuration](#configuration)
- [Docker Images](#docker-images)
- [Security Scanning](#security-scanning)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

This repository contains the Jenkins CI pipeline that:

1. **Builds** Docker images for frontend and backend
2. **Scans** images for security vulnerabilities
3. **Pushes** images to Docker Hub
4. **Updates** Kubernetes manifests with new image tags
5. **Triggers** ArgoCD deployment

### Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    GitHub Push Event                             │
└────────────────────────┬─────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Jenkins Pipeline                              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Stage 1: Checkout Source Code                              │  │
│  │  └─ Clone MIND repository (frontend + backend)             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 2: Checkout Manifests Repository                     │  │
│  │  └─ Clone mind-argocd-pipeline repository                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 3: Build Docker Images (Parallel)                    │  │
│  │  ├─ Build Frontend (React + Nginx)                         │  │
│  │  └─ Build Backend (Go + Alpine)                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 4: Security Scan (Parallel)                          │  │
│  │  ├─ Scan Frontend with Trivy                               │  │
│  │  └─ Scan Backend with Trivy                                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 5: Push to Docker Hub                                │  │
│  │  ├─ whosam1/notes-app-frontend:latest                      │  │
│  │  ├─ whosam1/notes-app-frontend:<git-sha>                   │  │
│  │  ├─ whosam1/notes-app-backend:latest                       │  │
│  │  └─ whosam1/notes-app-backend:<git-sha>                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 6: Update Kubernetes Manifests                       │  │
│  │  ├─ Update frontend-deployment.yaml                        │  │
│  │  ├─ Update backend-deployment.yaml                         │  │
│  │  └─ Set image tags to <git-sha>                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 7: Commit & Push Manifests                           │  │
│  │  └─ Push to mind-argocd-pipeline repository                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │ Stage 8: Trigger ArgoCD Sync                               │  │
│  │  └─ ArgoCD auto-syncs on manifest changes                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                   ArgoCD Deploys to Kubernetes                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 Pipeline Flow

### Detailed Flow Diagram

```
Developer Commit → GitHub Push
        │
        ├─  Webhook Trigger → Jenkins
        │
        ├─  Checkout Repositories
        │   ├─ Source: who-sam/MIND
        │   └─ Manifests: who-sam/mind-argocd-pipeline
        │
        ├─  Build Images (Parallel)
        │   ├─ Frontend Build (Multi-stage Docker)
        │   │  └─ Node build → Nginx serve
        │   └─ Backend Build (Multi-stage Docker)
        │      └─ Go compile → Alpine runtime
        │
        ├─  Security Scan (Parallel)
        │   ├─ Trivy scan frontend
        │   └─ Trivy scan backend
        │
        ├─  Push to Docker Hub
        │   ├─ Login to Docker Hub
        │   ├─ Push frontend:latest
        │   ├─ Push frontend:<sha>
        │   ├─ Push backend:latest
        │   ├─ Push backend:<sha>
        │   └─ Logout
        │
        ├─  Update Manifests
        │   ├─ sed replace frontend image tag
        │   ├─ sed replace backend image tag
        │   └─ Verify changes
        │
        ├─  Git Commit & Push
        │   ├─ git add deployments
        │   ├─ git commit
        │   └─ git push to manifests repo
        │
        └─  ArgoCD Sync
            └─ Auto-sync on manifest changes

Total Time: ~5-7 minutes
```

---

## ✅ Prerequisites

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **Jenkins** | 2.x | CI/CD automation |
| **Docker** | 20.x+ | Container builds |
| **Git** | 2.x+ | Version control |

### Required Plugins

Install these Jenkins plugins:
- Git Plugin
- Docker Pipeline
- GitHub Integration
- Pipeline
- Credentials Binding

### Required Credentials

Configure in Jenkins (`Manage Jenkins → Credentials`):

1. **dockerhub-credentials**
   - Type: Username with password
   - ID: `dockerhub-credentials`
   - Username: Your Docker Hub username
   - Password: Docker Hub access token

2. **github-token**
   - Type: Secret text
   - ID: `github-token`
   - Secret: GitHub personal access token
   - Scopes: `repo` (full control of private repositories)

### GitHub Access Token

Create token at: https://github.com/settings/tokens

Required scopes:
- ✅ `repo` - Full control of repositories
- ✅ `admin:repo_hook` - Webhook management

---

## 🚀 Setup Instructions

### 1. Fork/Clone Repository

```bash
git clone https://github.com/who-sam/mind-ci-pipeline.git
cd mind-ci-pipeline
```

### 2. Configure Jenkins Job

#### Create Pipeline Job

1. In Jenkins UI: **New Item**
2. Name: `mind-ci-pipeline`
3. Type: **Pipeline**
4. Click **OK**

#### Configure Pipeline

**General:**
- Description: `CI pipeline for MIND application`
- ✅ GitHub project
- Project URL: `https://github.com/who-sam/MIND`

**Build Triggers:**
- ✅ GitHub hook trigger for GITScm polling

**Pipeline:**
- Definition: **Pipeline script from SCM**
- SCM: **Git**
- Repository URL: `https://github.com/who-sam/mind-ci-pipeline.git`
- Branch: `main`
- Script Path: `Jenkinsfile`

**Save** configuration

### 3. Configure GitHub Webhook

#### Add Webhook

1. Go to source repository: `https://github.com/who-sam/MIND`
2. **Settings → Webhooks → Add webhook**
3. Configure:
   - Payload URL: `http://<jenkins-url>/github-webhook/`
   - Content type: `application/json`
   - Secret: (optional)
   - Events: ✅ **Just the push event**
   - ✅ Active

4. **Add webhook**

#### Test Webhook

```bash
# Make a test commit to MIND repository
cd MIND
echo "test" >> README.md
git commit -am "Test CI pipeline"
git push

# Check Jenkins for triggered build
```

### 4. Verify Setup

```bash
# Check Jenkins job is created
curl http://jenkins-url/job/mind-ci-pipeline/

# Check Docker Hub credentials
# In Jenkins UI: Credentials → dockerhub-credentials

# Check GitHub token
# In Jenkins UI: Credentials → github-token
```

---

## 🔄 Pipeline Stages

### Stage 1: Checkout Notes App Code

```groovy
stage('Checkout Notes App Code') {
    steps {
        git branch: 'main', url: 'https://github.com/who-sam/MIND.git'
    }
}
```

**Purpose:** Clone source code repository  
**Time:** ~5 seconds

### Stage 2: Checkout Manifests Repo

```groovy
stage('Checkout Manifests Repo') {
    steps {
        dir('manifests-repo') {
            git branch: 'main', 
                credentialsId: 'github-token',
                url: 'https://github.com/who-sam/mind-argocd-pipeline.git'
        }
    }
}
```

**Purpose:** Clone Kubernetes manifests repository  
**Time:** ~5 seconds

### Stage 3: Build Docker Images

```groovy
stage('Build Docker Images') {
    parallel {
        stage('Build Frontend') {
            steps {
                script {
                    sh """
                    docker build \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${FRONTEND_IMAGE}:latest \
                        -f frontend/Dockerfile ./frontend
                    """
                }
            }
        }
        stage('Build Backend') {
            steps {
                script {
                    sh """
                    docker build \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${BACKEND_IMAGE}:latest \
                        -f backend/Dockerfile ./backend
                    """
                }
            }
        }
    }
}
```

**Purpose:** Build optimized multi-stage Docker images  
**Time:** ~2-3 minutes (parallel)

### Stage 4: Security Scan

```groovy
stage('Security Scan') {
    parallel {
        stage('Scan Frontend Image') {
            steps {
                script {
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image --severity HIGH,CRITICAL \
                        ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }
        stage('Scan Backend Image') {
            steps {
                script {
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image --severity HIGH,CRITICAL \
                        ${BACKEND_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }
    }
}
```

**Purpose:** Scan for security vulnerabilities  
**Time:** ~1-2 minutes (parallel)

### Stage 5: Push to Docker Hub

```groovy
stage('Push to Docker Hub') {
    steps {
        script {
            sh """
            echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
            
            docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
            docker push ${FRONTEND_IMAGE}:latest
            docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
            docker push ${BACKEND_IMAGE}:latest
            
            docker logout
            """
        }
    }
}
```

**Purpose:** Push images to Docker Hub registry  
**Time:** ~1-2 minutes

### Stage 6: Update K8s Manifests

```groovy
stage('Update K8s Manifests') {
    steps {
        script {
            dir('manifests-repo') {
                sh """
                sed -i 's|image: whosam1/notes-app-frontend:.*|image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|' frontend-deployment.yaml
                sed -i 's|image: whosam1/notes-app-backend:.*|image: ${BACKEND_IMAGE}:${IMAGE_TAG}|' backend-deployment.yaml
                
                grep "image:" frontend-deployment.yaml
                grep "image:" backend-deployment.yaml
                """
            }
        }
    }
}
```

**Purpose:** Update manifests with new image tags  
**Time:** ~5 seconds

### Stage 7: Commit and Push Manifests

```groovy
stage('Commit and Push Manifests') {
    steps {
        script {
            dir('manifests-repo') {
                sh """
                git config user.name "Jenkins CI"
                git config user.email "jenkins@ci.example.com"
                git add frontend-deployment.yaml backend-deployment.yaml
                
                git commit -m "CI: Update Notes App images to ${IMAGE_TAG}"
                git push https://${GITHUB_TOKEN}@github.com/who-sam/mind-argocd-pipeline.git main
                """
            }
        }
    }
}
```

**Purpose:** Commit and push manifest changes  
**Time:** ~10 seconds

### Stage 8: Trigger ArgoCD Sync

```groovy
stage('Trigger ArgoCD Sync') {
    steps {
        script {
            sh """
            echo "ArgoCD will automatically sync the changes"
            """
        }
    }
}
```

**Purpose:** ArgoCD auto-detects and syncs  
**Time:** Handled by ArgoCD (~30 seconds)

---

## ⚙️ Configuration

### Environment Variables

Defined in Jenkinsfile:

```groovy
environment {
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
    FRONTEND_IMAGE = 'whosam1/notes-app-frontend'
    BACKEND_IMAGE = 'whosam1/notes-app-backend'
    GITHUB_TOKEN = credentials('github-token')
    IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
    BUILD_DATE = new Date().format('yyyyMMdd-HHmmss')
}
```

### Customization

#### Change Docker Hub Repository

Edit `Jenkinsfile`:
```groovy
FRONTEND_IMAGE = 'your-username/frontend-app'
BACKEND_IMAGE = 'your-username/backend-app'
```

#### Change Branch

Edit `Jenkinsfile`:
```groovy
git branch: 'develop',  // Change from 'main'
```

#### Change Image Tag Strategy

Edit `Jenkinsfile`:
```groovy
// Use build number instead of git commit
IMAGE_TAG = "${env.BUILD_NUMBER}"

// Use timestamp
IMAGE_TAG = new Date().format('yyyyMMdd-HHmmss')
```

---

## 🐳 Docker Images

### Image Tags

Each build creates two tags:

1. **Git Commit SHA** (e.g., `abc123d`)
   - Specific version tracking
   - Enables rollbacks
   - Immutable reference

2. **latest**
   - Always points to newest build
   - For development/testing
   - Not recommended for production

### Example Images

```
whosam1/notes-app-frontend:latest
whosam1/notes-app-frontend:abc123d

whosam1/notes-app-backend:latest
whosam1/notes-app-backend:abc123d
```

### Image Sizes

| Image | Size | Layers |
|-------|------|--------|
| Frontend | ~25 MB | 6 |
| Backend | ~20 MB | 5 |

### Pull Images

```bash
# Pull latest
docker pull whosam1/notes-app-frontend:latest
docker pull whosam1/notes-app-backend:latest

# Pull specific version
docker pull whosam1/notes-app-frontend:abc123d
docker pull whosam1/notes-app-backend:abc123d
```

---

## 🔒 Security Scanning

### Trivy Scanner

Scans for:
- CVEs (Common Vulnerabilities and Exposures)
- Misconfigurations
- Secrets in images
- License issues

### Scan Results

Pipeline shows:
```
HIGH: 2 vulnerabilities found
CRITICAL: 0 vulnerabilities found
```

**Action:** Review and fix high/critical vulnerabilities

### Manual Scan

```bash
# Scan frontend
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image \
  whosam1/notes-app-frontend:latest

# Scan backend
trivy image whosam1/notes-app-backend:latest
```

---

## 🐛 Troubleshooting

### Issue: Build Fails - Docker Permission Denied

```
Permission denied while trying to connect to Docker daemon
```

**Solution:**
```bash
# Add Jenkins user to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue: GitHub Token Authentication Failed

```
fatal: Authentication failed
```

**Solution:**
```bash
# Verify token has correct scopes
# In GitHub: Settings → Developer settings → Personal access tokens
# Required: repo (full control)

# Update credential in Jenkins
# Manage Jenkins → Credentials → github-token → Update
```

### Issue: Docker Hub Push Fails

```
denied: requested access to the resource is denied
```

**Solution:**
```bash
# Verify Docker Hub credentials
# Test login manually:
echo "YOUR_TOKEN" | docker login -u YOUR_USERNAME --password-stdin

# Update Jenkins credential
```

### Issue: Manifest Update Not Triggering Deployment

```
ArgoCD not detecting changes
```

**Solution:**
```bash
# Check ArgoCD sync policy
kubectl get application -n argocd notes-app -o yaml

# Manually trigger sync
argocd app sync notes-app

# Check for conflicts
argocd app diff notes-app
```

### Debug Pipeline

Enable verbose logging:

```groovy
// Add to Jenkinsfile
options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
    ansiColor('xterm')
}
```

View logs:
```bash
# In Jenkins UI
# Build → Console Output

# Or via CLI
jenkins-cli console <job-name> <build-number>
```

---

## 📊 Pipeline Metrics

### Typical Build Times

| Stage | Time | % of Total |
|-------|------|------------|
| Checkout | ~10s | 3% |
| Build Images | ~3m | 50% |
| Security Scan | ~1.5m | 25% |
| Push to Hub | ~1m | 17% |
| Update Manifests | ~30s | 5% |
| **Total** | **~6m** | **100%** |

### Success Rate

- Target: >95% success rate
- Common failures: Network timeouts, credential issues

---

## 🔗 Related Repositories

- **Source Code**: [MIND](https://github.com/who-sam/MIND)
- **Infrastructure**: [mind-infra-pipeline](https://github.com/who-sam/mind-infra-pipeline)
- **ArgoCD**: [mind-argocd-pipeline](https://github.com/who-sam/mind-argocd-pipeline)
