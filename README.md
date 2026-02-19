# 🚀 End-to-End GitOps CI/CD Pipeline

> **Stack:** GitHub → Tekton → Maven → SonarQube → Docker → Docker Hub → ArgoCD → Minikube (Helm)

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Pipeline Flow](#pipeline-flow)
3. [Infrastructure Requirements](#infrastructure-requirements)
4. [Tool Installation](#tool-installation)
5. [Repository Structure](#repository-structure)
6. [Application Code](#application-code)
7. [Helm Chart Setup](#helm-chart-setup)
8. [Tekton Pipeline Setup](#tekton-pipeline-setup)
9. [Secrets Configuration](#secrets-configuration)
10. [ArgoCD Setup](#argocd-setup)
11. [Running the Pipeline](#running-the-pipeline)
12. [Dev vs Production Workflow](#dev-vs-production-workflow)
13. [Manual Approval Gate](#manual-approval-gate)
14. [Troubleshooting](#troubleshooting)

---

## 🗺️ Architecture Overview

```
Developer pushes code
        ↓
    GitHub Repo
        ↓
  Tekton Pipeline (runs inside Minikube)
        ↓
  ┌─────────────────────────────────┐
  │  Step 1: Maven Build & Test     │
  │  Step 2: SonarQube Code Scan    │
  │  Step 3: Docker Build & Push    │
  │  Step 4: Deploy to DEV (auto)   │
  │  Step 5: Manual Approval Gate   │
  │  Step 6: Deploy to PROD         │
  └─────────────────────────────────┘
        ↓
  ArgoCD watches GitHub
        ↓
  Auto-deploys to Minikube (K8s)
```

| Tool | Role |
|------|------|
| **GitHub** | Source code + Helm values (GitOps source of truth) |
| **Tekton** | Cloud-native CI/CD engine inside Kubernetes |
| **Apache Maven** | Java app build + unit tests |
| **SonarQube** | Static code analysis & quality gate |
| **Docker/Kaniko** | Container image build & push |
| **Docker Hub** | Container image registry |
| **Helm** | Kubernetes package manager (templated deployments) |
| **ArgoCD** | GitOps continuous delivery — watches GitHub, deploys to K8s |
| **Minikube** | Local Kubernetes cluster on EC2 |

---

## 🔁 Pipeline Flow

```
GitHub Push
    │
    ▼
Tekton Pipeline
    │
    ├─► Maven Build (clone repo → mvn clean package)
    │
    ├─► SonarQube Scan (code quality analysis)
    │
    ├─► Docker Build & Push (Kaniko → Docker Hub)
    │
    ├─► Deploy DEV (update values-dev.yaml → ArgoCD auto-syncs)
    │
    ├─► ⏸️ Manual Approval Gate (team lead approves/rejects)
    │
    └─► Deploy PROD (update values-prod.yaml → ArgoCD syncs)
```

---

## 🖥️ Infrastructure Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| EC2 Instance | t3.medium | m7i-flex.large |
| RAM | 4GB | 8GB |
| CPU | 2 vCPUs | 2+ vCPUs |
| Disk | 20GB | 30GB |
| OS | Ubuntu 22.04 | Ubuntu 24.04 LTS |

### AWS Security Group — Open these ports:

| Port | Service |
|------|---------|
| 22 | SSH |
| 9000 | SonarQube UI |
| 8080 | ArgoCD UI |
| 30080-30082 | App NodePorts |

---

## 🔧 Tool Installation

### Step 1 — Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install Docker

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker ubuntu
newgrp docker

# Verify
docker --version
```

### Step 3 — Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Verify
kubectl version --client
```

### Step 4 — Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Start Minikube with Docker driver
minikube start --driver=docker --cpus=2 --memory=6144 --disk-size=20g

# Verify
minikube status
kubectl get nodes
```

### Step 5 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

### Step 6 — Install Tekton

```bash
# Install Tekton Pipelines
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Wait for pods
kubectl wait --for=condition=ready pod --all -n tekton-pipelines --timeout=300s

# Install Tekton Dashboard
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Install Tekton CLI
curl -LO https://github.com/tektoncd/cli/releases/download/v0.40.0/tkn_0.40.0_Linux_x86_64.tar.gz
tar xvzf tkn_0.40.0_Linux_x86_64.tar.gz tkn
sudo mv tkn /usr/local/bin/
rm tkn_0.40.0_Linux_x86_64.tar.gz

# Fix PodSecurity for Tekton namespace (IMPORTANT — required for Minikube)
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite

# Verify
kubectl get pods -n tekton-pipelines
tkn version
```

### Step 7 — Install SonarQube via Helm

```bash
# Add Helm repo
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update

# Create namespace
kubectl create namespace sonarqube

# Install SonarQube (community edition)
helm install sonarqube sonarqube/sonarqube \
  --namespace sonarqube \
  --set community.enabled=true \
  --set service.type=NodePort \
  --set persistence.enabled=false \
  --set monitoringPasscode=admin123 \
  --set resources.requests.memory=1Gi \
  --set resources.requests.cpu=500m \
  --set resources.limits.memory=2Gi \
  --set resources.limits.cpu=1000m

# Wait for pod to be ready (3-5 minutes)
kubectl get pods -n sonarqube -w

# Expose SonarQube UI
kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000 --address 0.0.0.0 &
```

**Access:** `http://YOUR_EC2_IP:9000`
**Default credentials:** `admin` / `admin` (you'll be asked to change it — must be 12+ characters)

### Step 8 — Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s

# Expose ArgoCD UI
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0 &

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo

# Install ArgoCD CLI
curl -sSLo /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
```

**Access:** `http://YOUR_EC2_IP:8080`
**Username:** `admin` | **Password:** from command above

---

## 📁 Repository Structure

```
devops-project-gitops/
├── app/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/java/com/gitops/App.java
├── helm/
│   └── gitops-app/
│       ├── Chart.yaml
│       ├── values.yaml          # Base/common values
│       ├── values-dev.yaml      # Dev environment overrides
│       ├── values-prod.yaml     # Prod environment overrides
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
├── tekton/
│   ├── tasks/
│   │   ├── maven-task.yaml
│   │   ├── sonarqube-task.yaml
│   │   ├── docker-task.yaml
│   │   ├── approval-task.yaml
│   │   └── deploy-to-env-task.yaml
│   ├── pipeline.yaml
│   ├── pipelinerun.yaml
│   └── approval-rbac.yaml
└── argocd/
    ├── dev-application.yaml
    └── prod-application.yaml
```

### Create directory structure:

```bash
git clone git@github.com:YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO

mkdir -p app/src/main/java/com/gitops
mkdir -p helm/gitops-app/templates
mkdir -p tekton/tasks
mkdir -p argocd
```

---

## ☕ Application Code

### `app/src/main/java/com/gitops/App.java`

```java
package com.gitops;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class App {

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }

    @GetMapping("/")
    public String home() {
        return "Hello from GitOps Pipeline! Version 1.0.0";
    }

    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

### `app/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.gitops</groupId>
    <artifactId>gitops-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <properties>
        <java.version>17</java.version>
        <sonar.projectKey>gitops-app</sonar.projectKey>
        <sonar.projectName>gitops-app</sonar.projectName>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.sonarsource.scanner.maven</groupId>
                <artifactId>sonar-maven-plugin</artifactId>
                <version>3.10.0.2594</version>
            </plugin>
        </plugins>
    </build>
</project>
```

### `app/Dockerfile`

```dockerfile
FROM maven:3.9.5-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## ⎈ Helm Chart Setup

Helm uses a **single chart with multiple values files** — one per environment. This is the correct Helm pattern for multi-environment deployments.

### `helm/gitops-app/Chart.yaml`

```yaml
apiVersion: v2
name: gitops-app
description: A Helm chart for GitOps Pipeline App
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### `helm/gitops-app/values.yaml` (Base — common defaults)

```yaml
replicaCount: 1

image:
  repository: YOUR_DOCKERHUB_USERNAME/gitops-app
  tag: "latest"
  pullPolicy: Always

service:
  type: NodePort
  port: 8080
  nodePort: 30080

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"

app:
  name: gitops-app
```

### `helm/gitops-app/values-dev.yaml` (Dev overrides)

```yaml
# Dev — minimal resources, auto deploy by ArgoCD
replicaCount: 1

image:
  tag: "latest"

service:
  nodePort: 30081

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### `helm/gitops-app/values-prod.yaml` (Prod overrides)

```yaml
# Prod — more replicas, manual approval required
replicaCount: 1  # Increase to 2+ on real multi-node clusters

image:
  tag: "latest"

service:
  nodePort: 30082

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### `helm/gitops-app/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
        - name: {{ .Values.app.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: {{ .Values.resources.requests.memory }}
              cpu: {{ .Values.resources.requests.cpu }}
            limits:
              memory: {{ .Values.resources.limits.memory }}
              cpu: {{ .Values.resources.limits.cpu }}
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
```

### `helm/gitops-app/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Values.app.name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Values.app.name }}
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: 8080
      nodePort: {{ .Values.service.nodePort }}
```

### Validate Helm chart:

```bash
helm lint helm/gitops-app/
helm template helm/gitops-app/ -f helm/gitops-app/values-dev.yaml
```

---

## 🔩 Tekton Pipeline Setup

### Configure SonarQube First

1. Login to SonarQube at `http://YOUR_EC2_IP:9000`
2. Create Project → Local → name: `gitops-app`, key: `gitops-app`
3. Go to My Account → Security → Generate Token
   - Name: `tekton-token`, Type: Global Analysis Token
   - Copy the token (starts with `sqa_...`)

### `tekton/tasks/maven-task.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: maven-build
  namespace: tekton-pipelines
spec:
  params:
    - name: IMAGE_TAG
      type: string
  workspaces:
    - name: source
  steps:
    - name: clone
      image: alpine/git
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        rm -rf ./* ./.* 2>/dev/null || true
        git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git .
        echo "✅ Repo cloned successfully"

    - name: maven-build
      image: maven:3.9.5-eclipse-temurin-17
      workingDir: $(workspaces.source.path)/app
      script: |
        #!/bin/sh
        mvn clean package -DskipTests
        echo "✅ Maven build successful"
```

### `tekton/tasks/sonarqube-task.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: sonarqube-scan
  namespace: tekton-pipelines
spec:
  workspaces:
    - name: source
  steps:
    - name: sonar-scan
      image: maven:3.9.5-eclipse-temurin-17
      workingDir: $(workspaces.source.path)/app
      script: |
        mvn sonar:sonar \
          -Dsonar.projectKey=gitops-app \
          -Dsonar.projectName=gitops-app \
          -Dsonar.host.url=http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
          -Dsonar.login=YOUR_SONARQUBE_TOKEN
        echo "✅ SonarQube scan complete"
```

### `tekton/tasks/docker-task.yaml`

> Uses **Kaniko** — builds Docker images inside Kubernetes without Docker daemon.

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: docker-build-push
  namespace: tekton-pipelines
spec:
  params:
    - name: IMAGE_TAG
      type: string
  workspaces:
    - name: source
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:latest
      workingDir: $(workspaces.source.path)/app
      args:
        - "--dockerfile=Dockerfile"
        - "--context=$(workspaces.source.path)/app"
        - "--destination=YOUR_DOCKERHUB_USERNAME/gitops-app:$(params.IMAGE_TAG)"
        - "--destination=YOUR_DOCKERHUB_USERNAME/gitops-app:latest"
        - "--insecure"
        - "--skip-tls-verify"
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
  volumes:
    - name: kaniko-secret
      secret:
        secretName: kaniko-secret
        items:
          - key: config.json
            path: config.json
```

### `tekton/tasks/approval-task.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: manual-approval
  namespace: tekton-pipelines
spec:
  params:
    - name: IMAGE_TAG
      type: string
    - name: TIMEOUT
      type: string
      default: "600"
  steps:
    - name: wait-for-approval
      image: bitnami/kubectl:latest
      script: |
        #!/bin/bash
        echo "================================================"
        echo "⏸️  PRODUCTION DEPLOYMENT - MANUAL APPROVAL REQUIRED"
        echo "================================================"
        echo "📦 Image: YOUR_DOCKERHUB_USERNAME/gitops-app:$(params.IMAGE_TAG)"
        echo "🌍 Target: PRODUCTION namespace"
        echo ""
        echo "✅ To APPROVE run:"
        echo "kubectl patch configmap pipeline-approval -n tekton-pipelines --type merge -p '{\"data\":{\"status\":\"approved\"}}'"
        echo ""
        echo "❌ To REJECT run:"
        echo "kubectl patch configmap pipeline-approval -n tekton-pipelines --type merge -p '{\"data\":{\"status\":\"rejected\"}}'"
        echo "================================================"

        # Reset to pending
        kubectl patch configmap pipeline-approval \
          -n tekton-pipelines \
          --type merge \
          -p '{"data":{"status":"pending"}}' 2>/dev/null || \
        kubectl create configmap pipeline-approval \
          -n tekton-pipelines \
          --from-literal=status=pending

        TIMEOUT=$(params.TIMEOUT)
        ELAPSED=0
        while [ $ELAPSED -lt $TIMEOUT ]; do
          STATUS=$(kubectl get configmap pipeline-approval \
            -n tekton-pipelines \
            -o jsonpath='{.data.status}')

          echo "⏳ Awaiting approval... [$STATUS] (${ELAPSED}s / ${TIMEOUT}s)"

          if [ "$STATUS" = "approved" ]; then
            echo "✅ APPROVED! Deploying to production..."
            exit 0
          fi

          if [ "$STATUS" = "rejected" ]; then
            echo "❌ REJECTED! Production deployment cancelled."
            exit 1
          fi

          sleep 15
          ELAPSED=$((ELAPSED + 15))
        done

        echo "⏰ Timed out after ${TIMEOUT}s"
        exit 1
```

### `tekton/tasks/deploy-to-env-task.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: deploy-to-env
  namespace: tekton-pipelines
spec:
  params:
    - name: IMAGE_TAG
      type: string
    - name: ENVIRONMENT
      type: string
  workspaces:
    - name: source
  steps:
    - name: update-helm-values
      image: alpine/git
      script: |
        #!/bin/sh
        # Setup SSH key
        mkdir -p /tmp/.ssh
        cp /ssh-key/ssh-privatekey /tmp/.ssh/id_ed25519
        chmod 600 /tmp/.ssh/id_ed25519
        ssh-keyscan github.com >> /tmp/.ssh/known_hosts
        export GIT_SSH_COMMAND="ssh -i /tmp/.ssh/id_ed25519 -o UserKnownHostsFile=/tmp/.ssh/known_hosts"

        # Clone repo fresh
        rm -rf /tmp/gitops-repo
        git clone git@github.com:YOUR_USERNAME/YOUR_REPO.git /tmp/gitops-repo
        cd /tmp/gitops-repo

        # Update image tag in correct environment values file
        sed -i "s|tag: .*|tag: \"$(params.IMAGE_TAG)\"|" helm/gitops-app/values-$(params.ENVIRONMENT).yaml

        echo "✅ Updated values-$(params.ENVIRONMENT).yaml:"
        cat helm/gitops-app/values-$(params.ENVIRONMENT).yaml

        # Commit and push
        git config user.email "tekton@gitops.com"
        git config user.name "Tekton Pipeline"
        git add helm/gitops-app/values-$(params.ENVIRONMENT).yaml
        git commit -m "$(params.ENVIRONMENT): Update image tag to $(params.IMAGE_TAG)"
        git push git@github.com:YOUR_USERNAME/YOUR_REPO.git main
        echo "✅ $(params.ENVIRONMENT) deployment triggered via ArgoCD"
      volumeMounts:
        - name: ssh-key
          mountPath: /ssh-key
  volumes:
    - name: ssh-key
      secret:
        secretName: github-ssh-secret
        defaultMode: 0400
```

### `tekton/pipeline.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: gitops-pipeline
  namespace: tekton-pipelines
spec:
  params:
    - name: IMAGE_TAG
      type: string
      default: "latest"
  workspaces:
    - name: shared-workspace
  tasks:
    - name: maven-build
      taskRef:
        name: maven-build
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)

    - name: sonarqube-scan
      taskRef:
        name: sonarqube-scan
      runAfter:
        - maven-build
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: docker-build-push
      taskRef:
        name: docker-build-push
      runAfter:
        - sonarqube-scan
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)

    - name: deploy-dev
      taskRef:
        name: deploy-to-env
      runAfter:
        - docker-build-push
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: ENVIRONMENT
          value: "dev"

    - name: manual-approval
      taskRef:
        name: manual-approval
      runAfter:
        - deploy-dev
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: TIMEOUT
          value: "600"

    - name: deploy-prod
      taskRef:
        name: deploy-to-env
      runAfter:
        - manual-approval
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: ENVIRONMENT
          value: "prod"
```

### `tekton/pipelinerun.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: gitops-pipeline-run-
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: gitops-pipeline
  params:
    - name: IMAGE_TAG
      value: "v1.0.0"        # ← Change version here for each release
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
```

### `tekton/approval-rbac.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: approval-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: default
    namespace: tekton-pipelines
```

---

## 🔐 Secrets Configuration

### GitHub SSH Key Setup

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your-email@gmail.com" -f ~/.ssh/id_ed25519 -N ""

# Display public key — add this to GitHub Settings → SSH Keys
cat ~/.ssh/id_ed25519.pub

# Test connection
ssh -T git@github.com

# Set repo remote to SSH
git remote set-url origin git@github.com:YOUR_USERNAME/YOUR_REPO.git
```

### Kubernetes Secrets

```bash
# 1. Docker Hub secret for Kaniko image push
kubectl create secret generic kaniko-secret \
  --from-literal=config.json="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"$(echo -n 'YOUR_DOCKERHUB_USERNAME:YOUR_DOCKERHUB_TOKEN' | base64 -w 0)\"}}}" \
  -n tekton-pipelines

# 2. GitHub SSH secret for pushing Helm values
kubectl create secret generic github-ssh-secret \
  --from-file=ssh-privatekey=/home/ubuntu/.ssh/id_ed25519 \
  --type=kubernetes.io/ssh-auth \
  -n tekton-pipelines

# Verify secrets
kubectl get secrets -n tekton-pipelines
```

> **Note:** Generate Docker Hub token at hub.docker.com → Account Settings → Security → New Access Token

---

## 🔄 ArgoCD Setup

### `argocd/dev-application.yaml` — Auto sync

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    targetRevision: main
    path: helm/gitops-app
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### `argocd/prod-application.yaml` — Manual sync only

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    targetRevision: main
    path: helm/gitops-app
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # No automated sync — requires manual ArgoCD sync after approval
```

### Apply ArgoCD apps:

```bash
kubectl apply -f argocd/dev-application.yaml
kubectl apply -f argocd/prod-application.yaml

# Verify
kubectl get applications -n argocd
```

---

## 🚀 Running the Pipeline

### Apply all Tekton resources:

```bash
# Apply RBAC first
kubectl apply -f tekton/approval-rbac.yaml

# Apply all tasks
kubectl apply -f tekton/tasks/maven-task.yaml
kubectl apply -f tekton/tasks/sonarqube-task.yaml
kubectl apply -f tekton/tasks/docker-task.yaml
kubectl apply -f tekton/tasks/approval-task.yaml
kubectl apply -f tekton/tasks/deploy-to-env-task.yaml

# Apply pipeline
kubectl apply -f tekton/pipeline.yaml

# Verify
tkn task list -n tekton-pipelines
tkn pipeline list -n tekton-pipelines
```

### Trigger a pipeline run:

```bash
# Edit pipelinerun.yaml to set your version tag first
# Then create (not apply — generateName requires create)
kubectl create -f tekton/pipelinerun.yaml
```

### Watch logs:

```bash
tkn pipelinerun logs --last -f -n tekton-pipelines
```

### Check status:

```bash
tkn pipelinerun list -n tekton-pipelines
```

---

## 🌍 Dev vs Production Workflow

| Feature | DEV | PROD |
|---------|-----|------|
| Namespace | `dev` | `prod` |
| NodePort | 30081 | 30082 |
| ArgoCD Sync | Automatic | Manual |
| Replicas | 1 | 1 (2+ on real clusters) |
| Approval | Not required | Required |
| Values file | `values-dev.yaml` | `values-prod.yaml` |

```
New image built
      │
      ▼
deploy-dev task
  → updates values-dev.yaml in GitHub
  → ArgoCD auto-syncs DEV namespace ✅
      │
      ▼
manual-approval task (waits up to 600s)
      │
      ├─ Approved → deploy-prod task
      │               → updates values-prod.yaml
      │               → ArgoCD syncs PROD namespace ✅
      │
      └─ Rejected → pipeline stops ❌
```

---

## ⏸️ Manual Approval Gate

When the pipeline reaches the approval stage, you'll see this in the logs:

```
⏸️  PRODUCTION DEPLOYMENT - MANUAL APPROVAL REQUIRED
📦 Image: YOUR_USERNAME/gitops-app:v2.0.0
```

### To APPROVE (in a second terminal):

```bash
kubectl patch configmap pipeline-approval \
  -n tekton-pipelines \
  --type merge \
  -p '{"data":{"status":"approved"}}'
```

### To REJECT:

```bash
kubectl patch configmap pipeline-approval \
  -n tekton-pipelines \
  --type merge \
  -p '{"data":{"status":"rejected"}}'
```

### Then sync ArgoCD prod manually:

```bash
argocd login localhost:8080 --username admin --password YOUR_PASSWORD --insecure
argocd app sync gitops-app-prod
```

---

## 🔍 Troubleshooting

### Port-forwards stopped working

```bash
# Restart both port-forwards
kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000 --address 0.0.0.0 &
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0 &
```

### Git push rejected (Tekton pushed to GitHub, local is behind)

```bash
git stash
git pull origin main --rebase
git stash pop
git add .
git commit -m "your message"
git push origin main
```

### Tekton pods failing with PodSecurity error

```bash
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite
```

### Pods Pending — Insufficient CPU

```bash
# Check resource usage
kubectl describe node minikube | grep -A 10 "Allocated resources"

# Reduce resource requests in values files
# requests.cpu: "100m" is safe for Minikube lab
```

### Pipeline approval configmap not found

```bash
# Create it manually before approving
kubectl create configmap pipeline-approval \
  -n tekton-pipelines \
  --from-literal=status=approved
```

### Check pipeline run details

```bash
tkn pipelinerun describe $(tkn pipelinerun list -n tekton-pipelines -o name | head -1) -n tekton-pipelines
kubectl get events -n tekton-pipelines --sort-by='.lastTimestamp' | tail -20
```

---

## 📊 Verify Everything is Running

```bash
# All namespaces
kubectl get pods -n tekton-pipelines
kubectl get pods -n sonarqube
kubectl get pods -n argocd
kubectl get pods -n dev
kubectl get pods -n prod

# ArgoCD apps
kubectl get applications -n argocd

# Pipeline history
tkn pipelinerun list -n tekton-pipelines
```

---

## 🎯 Key Concepts Learned

- **GitOps** — Git is the single source of truth. Every deployment is a Git commit.
- **Helm Values Override Pattern** — One chart, multiple `values-env.yaml` files per environment.
- **Tekton** — Every CI step runs as a Kubernetes pod. No Jenkins server needed.
- **Kaniko** — Builds Docker images inside Kubernetes without Docker daemon.
- **ArgoCD** — Watches Git, auto-deploys when values.yaml changes.
- **Manual Approval Gate** — Production safety using a Kubernetes ConfigMap as a flag.
- **Pod Security** — Kubernetes namespaces need correct security labels for Tekton to work.

---

*Built with ❤️ — GitHub → Tekton → Maven → SonarQube → Docker → DockerHub → ArgoCD → Minikube*
