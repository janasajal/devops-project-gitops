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

| Tool | Role | Why We Use It |
|------|------|--------------|
| **GitHub** | Source code + Helm values | Single source of truth for code AND deployment config (GitOps principle) |
| **Tekton** | CI/CD engine inside Kubernetes | Cloud-native — no separate Jenkins server needed, runs as K8s pods |
| **Apache Maven** | Java app build + unit tests | Industry standard Java build tool, compiles code and runs tests |
| **SonarQube** | Static code analysis & quality gate | Catches bugs, security vulnerabilities before they reach production |
| **Docker/Kaniko** | Container image build & push | Packages app into portable container; Kaniko works without Docker daemon inside K8s |
| **Docker Hub** | Container image registry | Stores versioned images so Kubernetes can pull them during deployment |
| **Helm** | Kubernetes package manager | Templated K8s deployments — one chart, multiple environments, no YAML duplication |
| **ArgoCD** | GitOps continuous delivery | Watches GitHub, auto-deploys when Helm values change — Git is the source of truth |
| **Minikube** | Local Kubernetes cluster | Runs full K8s on a single EC2 server for lab/learning purposes |

---

## 🔁 Pipeline Flow

```
GitHub Push
    │
    ▼
Tekton Pipeline
    │
    ├─► Step 1: Maven Build     → Compiles Java app, runs tests
    │
    ├─► Step 2: SonarQube Scan  → Checks code quality
    │
    ├─► Step 3: Docker Push     → Builds image, pushes to Docker Hub
    │
    ├─► Step 4: Deploy DEV      → Updates values-dev.yaml → ArgoCD auto-deploys
    │
    ├─► Step 5: Approval Gate   → Pipeline PAUSES, waits for human approval
    │
    └─► Step 6: Deploy PROD     → Updates values-prod.yaml → ArgoCD deploys to prod
```

---

## 🖥️ Infrastructure Requirements

| Resource | Minimum | Recommended | Why |
|----------|---------|-------------|-----|
| EC2 Instance | t3.medium | m7i-flex.large | SonarQube + ArgoCD + Tekton are memory-heavy |
| RAM | 4GB | 8GB | Multiple services run simultaneously |
| CPU | 2 vCPUs | 2+ vCPUs | Minikube needs at least 2 CPUs |
| Disk | 20GB | 30GB | Docker images and Maven cache consume space |
| OS | Ubuntu 22.04 | Ubuntu 24.04 LTS | LTS = Long Term Support, stable and well-supported |

### AWS Security Group — Open these ports:

| Port | Service | Why Needed |
|------|---------|-----------|
| 22 | SSH | Remote access to EC2 |
| 9000 | SonarQube UI | Access code analysis dashboard from browser |
| 8080 | ArgoCD UI | Access GitOps dashboard from browser |
| 30080-30082 | App NodePorts | Access deployed app from browser |

---

## 🔧 Tool Installation

### Step 1 — Update System

**What it does:** Refreshes package lists and upgrades all installed packages to latest versions.

**Why:** Outdated packages can cause dependency conflicts when installing new tools. Always start with a clean, updated system to avoid broken dependencies later.

```bash
sudo apt update && sudo apt upgrade -y
```

---

### Step 2 — Install Docker

**What it does:** Installs Docker Engine — the container runtime that builds and runs containers.

**Why:** Docker is the foundation of everything in this pipeline. Minikube runs inside Docker (Docker driver), Kaniko builds Docker images, and our app is packaged as a Docker container. Without Docker, nothing else works.

```bash
# Install required dependencies for adding Docker's repo
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key — verifies packages are authentic and not tampered with
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker's official repository to apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and all plugins
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add ubuntu user to docker group — so we don't need 'sudo' before every docker command
sudo usermod -aG docker ubuntu
newgrp docker

# Verify
docker --version
```

---

### Step 3 — Install kubectl

**What it does:** Installs the Kubernetes command-line tool.

**Why:** `kubectl` is how you communicate with your Kubernetes cluster (Minikube). You use it to check pods, apply YAML files, read logs, manage secrets, and everything else in Kubernetes. Tekton and ArgoCD are both managed through `kubectl`.

```bash
# Download latest stable kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install it system-wide with correct permissions
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Verify
kubectl version --client
```

---

### Step 4 — Install Minikube

**What it does:** Creates a single-node Kubernetes cluster inside Docker on your EC2 server.

**Why:** We need a Kubernetes cluster to run Tekton, SonarQube, ArgoCD, and our app. Minikube gives us a full K8s cluster on a single machine — perfect for learning and labs. We use `--driver=docker` so Minikube runs as a Docker container (no VirtualBox needed on EC2).

```bash
# Download Minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Start Minikube with Docker driver
# --cpus=2        → give Minikube 2 CPUs (minimum required)
# --memory=6144   → give 6GB RAM to Minikube (leaves 2GB for OS)
# --disk-size=20g → 20GB disk for images, volumes, and data
minikube start --driver=docker --cpus=2 --memory=6144 --disk-size=20g

# Verify cluster is healthy
minikube status
kubectl get nodes
```

---

### Step 5 — Install Helm

**What it does:** Installs Helm — the package manager for Kubernetes.

**Why:** Writing raw Kubernetes YAML for every environment is repetitive and error-prone. Helm lets you create **templates** with variables (like image tag, replica count, resource limits). You then provide different `values.yaml` files per environment (dev/prod) to customize deployments — this is the standard industry approach to multi-environment K8s deployments.

```bash
# Official Helm install script — downloads and installs latest version
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

### Step 6 — Install Tekton

**What it does:** Installs Tekton Pipelines (core CI engine), Tekton Dashboard (web UI), and Tekton CLI (`tkn`) inside Kubernetes.

**Why Tekton instead of Jenkins?** Tekton is **cloud-native** — it runs entirely inside Kubernetes as pods. No separate Jenkins server to maintain. Every pipeline step is a container, so it scales with your cluster, uses K8s secrets natively, and needs no extra infrastructure outside of Kubernetes.

**Why the PodSecurity label?** Kubernetes 1.25+ enforces Pod Security Standards. The `tekton-pipelines` namespace defaults to `restricted` which blocks Tekton's internal init containers from running. Setting it to `privileged` allows Tekton to function correctly — this is standard practice for CI/CD namespaces.

```bash
# Install Tekton Pipelines (core engine — defines Tasks, Pipelines, PipelineRuns)
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Wait until all Tekton pods are running before proceeding (2-3 minutes)
kubectl wait --for=condition=ready pod --all -n tekton-pipelines --timeout=300s

# Install Tekton Dashboard (visual web UI to see pipeline runs)
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Install Tekton CLI (command line tool to trigger and monitor pipelines)
curl -LO https://github.com/tektoncd/cli/releases/download/v0.40.0/tkn_0.40.0_Linux_x86_64.tar.gz
tar xvzf tkn_0.40.0_Linux_x86_64.tar.gz tkn
sudo mv tkn /usr/local/bin/
rm tkn_0.40.0_Linux_x86_64.tar.gz

# CRITICAL: Fix Pod Security policy for tekton-pipelines namespace
# Without this, Tekton pods get blocked by Kubernetes security admission control
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite

# Verify everything is running
kubectl get pods -n tekton-pipelines
tkn version
```

---

### Step 7 — Install SonarQube via Helm

**What it does:** Deploys SonarQube inside Minikube using a Helm chart.

**Why SonarQube?** SonarQube performs **static code analysis** — it scans your source code without running it, looking for bugs, security vulnerabilities, code smells, and test coverage gaps. Catching these issues before they reach production saves time and prevents security incidents.

**Why deploy it inside Kubernetes via Helm?** Running it inside the same cluster as Tekton means they communicate via internal Kubernetes DNS — fast, secure, no public internet needed. Using Helm means one command installs all required K8s resources (Deployment, Service, ConfigMap, etc.).

```bash
# Add SonarQube's official Helm chart repository
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update

# Create a dedicated namespace — isolates SonarQube from other services
kubectl create namespace sonarqube

# Install SonarQube using Helm with these important flags:
# --set community.enabled=true   → use free Community edition (newer charts require this explicitly)
# --set service.type=NodePort    → expose UI so we can access from browser
# --set persistence.enabled=false → skip persistent volume (lab setup — data resets on pod restart)
# --set monitoringPasscode=...   → required by newer chart for /api/system/health endpoint
# resources → prevent SonarQube from consuming all available RAM/CPU
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

# Watch pod start — takes 3-5 minutes (SonarQube is slow to initialize)
kubectl get pods -n sonarqube -w
# Press Ctrl+C when you see: sonarqube-sonarqube-0   1/1   Running

# Expose SonarQube UI to your browser via port-forward
# --address 0.0.0.0 → allows access from outside the EC2 server
# & → runs in background so your terminal stays usable
kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000 --address 0.0.0.0 &
```

**Access:** `http://YOUR_EC2_IP:9000`
**Default login:** `admin` / `admin` — you'll be forced to change password (must be 12+ characters)

**If you forget your password, reset it from inside the pod:**
```bash
kubectl exec -it sonarqube-sonarqube-0 -n sonarqube -- bash
curl -u admin:admin -X POST \
  "http://localhost:9000/api/users/change_password?login=admin&previousPassword=admin&password=YourNewPassword123"
exit
```

---

### Step 8 — Install ArgoCD

**What it does:** Deploys ArgoCD inside Minikube — the GitOps continuous delivery tool.

**Why ArgoCD?** ArgoCD **watches your GitHub repository** and automatically deploys to Kubernetes whenever it detects a change in your Helm values files. This is the heart of GitOps — Git is the single source of truth. When Tekton pushes a new image tag to `values-dev.yaml`, ArgoCD detects it within minutes and deploys the new version automatically. For production, ArgoCD waits for a human to trigger sync.

```bash
# Create dedicated namespace for ArgoCD
kubectl create namespace argocd

# Install ArgoCD — all components in one manifest (server, repo-server, dex, redis, etc.)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all 7 ArgoCD pods to be ready (3-5 minutes)
kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s

# Verify all pods are running
kubectl get pods -n argocd

# Expose ArgoCD UI — maps EC2 port 8080 to ArgoCD's internal HTTPS port 443
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0 &

# Get the auto-generated initial admin password (base64 encoded in a secret)
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo

# Install ArgoCD CLI — for syncing apps from command line
curl -sSLo /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
```

**Access:** `http://YOUR_EC2_IP:8080`
**Username:** `admin` | **Password:** from the command above

---

## 📁 Repository Structure

**Why this structure?** Every folder has a single, clear responsibility. This separation makes it easy for different team members to work on different parts — developers touch `app/`, DevOps engineers touch `helm/` and `argocd/`, and the CI team manages `tekton/`. ArgoCD watches the `helm/` folder; Tekton tasks clone from and push to the same repo.

```
devops-project-gitops/
├── app/                              # Java Spring Boot application source code
│   ├── Dockerfile                    # Instructions to build the container image
│   ├── pom.xml                       # Maven build config, dependencies, SonarQube plugin
│   └── src/main/java/com/gitops/
│       └── App.java                  # Main REST API application
│
├── helm/
│   └── gitops-app/                   # Single Helm chart for the app (used for ALL environments)
│       ├── Chart.yaml                # Chart metadata (name, version)
│       ├── values.yaml               # Base/common default values (shared across envs)
│       ├── values-dev.yaml           # Dev overrides — nodePort 30081, auto-deploy
│       ├── values-prod.yaml          # Prod overrides — nodePort 30082, manual approval
│       └── templates/                # Kubernetes resource templates with Helm variables
│           ├── deployment.yaml       # K8s Deployment — how to run the app
│           └── service.yaml          # K8s Service — how to expose the app
│
├── tekton/                           # CI Pipeline definitions (all run inside K8s)
│   ├── tasks/
│   │   ├── maven-task.yaml           # Task: clone repo + mvn clean package
│   │   ├── sonarqube-task.yaml       # Task: mvn sonar:sonar code analysis
│   │   ├── docker-task.yaml          # Task: Kaniko build + push to Docker Hub
│   │   ├── approval-task.yaml        # Task: pause pipeline, wait for human approval
│   │   └── deploy-to-env-task.yaml   # Task: update Helm values file + git push
│   ├── pipeline.yaml                 # Orchestrates all tasks in sequence with runAfter
│   ├── pipelinerun.yaml              # Triggers the pipeline with version tag parameter
│   └── approval-rbac.yaml            # RBAC — gives Tekton permission to read/write ConfigMaps
│
└── argocd/
    ├── dev-application.yaml          # ArgoCD app — watches values-dev.yaml, auto-sync ON
    └── prod-application.yaml         # ArgoCD app — watches values-prod.yaml, manual sync only
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

**What it does:** A simple Spring Boot REST API with two endpoints — `/` returns a greeting and `/health` returns "OK".

**Why the `/health` endpoint?** Kubernetes uses this endpoint for **liveness and readiness probes**. Before sending traffic to a pod, Kubernetes calls `/health`. If it returns OK, the pod is ready. If it starts failing, Kubernetes restarts the pod automatically. This is how Kubernetes manages application health in production.

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

    // Kubernetes liveness + readiness probe endpoint
    // K8s calls this to decide if pod is healthy and ready for traffic
    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

### `app/pom.xml`

**What it does:** Defines the Maven project — its dependencies, build plugins, and SonarQube project configuration.

**Why Maven?** Maven is the industry-standard Java build tool. It automatically downloads libraries, compiles code, runs tests, and packages everything into a JAR file. The `sonar-maven-plugin` adds the `mvn sonar:sonar` command that Tekton uses to send analysis to SonarQube.

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

    <!-- Spring Boot parent handles dependency versions automatically -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <properties>
        <java.version>17</java.version>
        <!-- These tell the SonarQube plugin which project to upload results to -->
        <sonar.projectKey>gitops-app</sonar.projectKey>
        <sonar.projectName>gitops-app</sonar.projectName>
    </properties>

    <dependencies>
        <!-- Spring Boot web framework — gives us REST API support -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- JUnit test framework — scope=test means not included in final JAR -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Packages app as executable JAR (java -jar app.jar) -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- Enables: mvn sonar:sonar — sends analysis results to SonarQube -->
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

**What it does:** Defines how to build the Docker image in two stages — build stage and run stage.

**Why multi-stage build?** Stage 1 uses a large Maven+JDK image (~500MB) to compile the code. Stage 2 copies only the compiled JAR into a tiny JRE-only Alpine image (~80MB). The final image is much smaller — faster to push/pull from Docker Hub, faster to deploy, and has a smaller security attack surface (fewer packages = fewer vulnerabilities).

```dockerfile
# ─── Stage 1: BUILD ───────────────────────────────────────────────────────────
# Uses full Maven + JDK image — needed to compile Java code
FROM maven:3.9.5-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
# Compile and package into JAR (-DskipTests to keep build fast)
RUN mvn clean package -DskipTests
# Result: /app/target/gitops-app-1.0.0.jar

# ─── Stage 2: RUN ─────────────────────────────────────────────────────────────
# Uses minimal JRE-only Alpine image — much smaller than full JDK
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Copy only the JAR from build stage — NOT the Maven cache or source code
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## ⎈ Helm Chart Setup

**The correct Helm pattern for multi-environment deployments:**

Instead of creating separate Helm charts for dev and prod (which leads to YAML duplication), we use **one chart with multiple values files**. The base `values.yaml` has common defaults, and `values-dev.yaml` / `values-prod.yaml` only override what differs per environment.

```
ArgoCD reads them like this:
  DEV  → values.yaml + values-dev.yaml  (dev values override base)
  PROD → values.yaml + values-prod.yaml (prod values override base)
```

This means when you change a common setting (like resource limits), you change it in ONE place — `values.yaml` — and both environments get the update.

### `helm/gitops-app/Chart.yaml`

**What it does:** Metadata file that Helm requires to identify a directory as a chart.

**Why:** Helm will not recognize a folder as a chart without this file. It defines the chart name (used by ArgoCD to reference it), version (increment when chart structure changes), and app version (documentation only).

```yaml
apiVersion: v2
name: gitops-app
description: A Helm chart for GitOps Pipeline App
type: application
version: 0.1.0       # Helm chart version — increment when you change templates
appVersion: "1.0.0"  # App version — for documentation/reference only
```

### `helm/gitops-app/values.yaml` — Base defaults (all environments)

**What it does:** Defines default values shared across all environments. Every key here can be overridden by environment-specific values files.

**Why base values?** Centralizing common config means less repetition. Only what's different per environment goes in `values-dev.yaml` and `values-prod.yaml`. If dev and prod share the same memory limit, define it once here — not twice.

```yaml
replicaCount: 1

image:
  repository: YOUR_DOCKERHUB_USERNAME/gitops-app
  tag: "latest"        # Tekton pipeline updates this automatically per environment
  pullPolicy: Always   # Always pull — important when using "latest" tag

service:
  type: NodePort
  port: 8080
  nodePort: 30080      # Default port (overridden per environment)

resources:
  requests:
    memory: "128Mi"    # Minimum memory Kubernetes reserves for this pod
    cpu: "100m"        # 100 millicores = 0.1 CPU core
  limits:
    memory: "256Mi"    # Maximum memory pod can use before being killed
    cpu: "200m"        # Maximum CPU pod can use

app:
  name: gitops-app
```

### `helm/gitops-app/values-dev.yaml` — Dev overrides

**What it does:** Overrides only the values that differ in the dev environment.

**Why a separate NodePort?** Dev (30081) and prod (30082) need different ports so they can both run on the same Minikube cluster without conflict. In a real multi-cluster setup, they'd be on completely separate clusters.

```yaml
# Dev environment — lightweight, auto deployed by ArgoCD on every pipeline run
# Only override what's different from values.yaml

replicaCount: 1   # One replica is enough for dev testing

image:
  tag: "latest"   # Tekton updates this to the new version tag on each build

service:
  nodePort: 30081  # Dev uses 30081 — different from prod to avoid port conflict
```

### `helm/gitops-app/values-prod.yaml` — Prod overrides

**What it does:** Overrides values for the production environment — different port, potentially more replicas.

**Why manual ArgoCD sync for prod?** Even though Tekton updates this file, ArgoCD does NOT auto-deploy prod (no `automated` in prod ArgoCD app). A human must run `argocd app sync gitops-app-prod`. This gives the operations team control over exactly when production changes go live — a critical safety requirement in real organizations.

```yaml
# Prod environment — requires manual ArgoCD sync after Tekton approval gate
# Only override what's different from values.yaml

replicaCount: 1   # Increase to 2+ on real multi-node clusters for high availability

image:
  tag: "latest"   # Tekton updates this ONLY after manual approval in pipeline

service:
  nodePort: 30082  # Prod uses 30082 — different from dev
```

### `helm/gitops-app/templates/deployment.yaml`

**What it does:** Kubernetes Deployment template using Helm variables (`{{ .Values.xxx }}`). Helm renders this template by substituting variables with values from `values.yaml` + the environment override file.

**Why readiness and liveness probes?** Readiness probe: Kubernetes won't send traffic to a pod until `/health` returns 200. This prevents users from hitting a pod that's still starting up. Liveness probe: if `/health` starts failing (app crashed or hung), Kubernetes automatically restarts the pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicaCount }}   # From values.yaml or env override
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
          # Image tag comes from values-dev.yaml or values-prod.yaml
          # Tekton updates this tag on each build
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
          # Readiness probe: K8s only sends traffic after this passes
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30   # Wait 30s before first check (app startup time)
            periodSeconds: 10
          # Liveness probe: K8s restarts pod if this starts failing
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60   # Give app 60s to fully start before checking
            periodSeconds: 15
```

### `helm/gitops-app/templates/service.yaml`

**What it does:** Exposes the app pod as a network service inside Kubernetes using NodePort.

**Why NodePort?** NodePort exposes the service on a fixed port on the Kubernetes node (Minikube VM), making it accessible from your browser at `minikube_ip:nodePort`. Dev gets port 30081, prod gets port 30082. In real production clusters, you'd use a LoadBalancer or Ingress instead.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Values.app.name }}
spec:
  type: {{ .Values.service.type }}   # NodePort for Minikube access
  selector:
    app: {{ .Values.app.name }}   # Routes traffic to pods with this label
  ports:
    - protocol: TCP
      port: {{ .Values.service.port }}         # Port inside the cluster
      targetPort: 8080                          # Port the container listens on
      nodePort: {{ .Values.service.nodePort }}  # Port exposed on Minikube node (30081/30082)
```

### Validate Helm chart:

```bash
# Check chart for syntax errors and best practices
helm lint helm/gitops-app/

# Preview rendered templates — see the actual K8s YAML that will be generated
# Useful to verify variable substitution is correct
helm template helm/gitops-app/ \
  -f helm/gitops-app/values.yaml \
  -f helm/gitops-app/values-dev.yaml
```

---

## 🔩 Tekton Pipeline Setup

### Configure SonarQube First (before pipeline can run)

**What you're doing:** Creating a project in SonarQube and generating an auth token. Tekton needs the token to authenticate when sending analysis results.

**Why token instead of password?** Tokens are more secure — they can be scoped (read-only vs. full access), can be revoked individually without changing your password, and are the recommended approach for service-to-service authentication.

1. Login to SonarQube at `http://YOUR_EC2_IP:9000`
2. Click **Create Project** → **Local Project**
   - Project name: `gitops-app`
   - Project key: `gitops-app` (must match `sonar.projectKey` in pom.xml)
   - Main branch: `main`
3. Go to **My Account** (top right) → **Security** → **Generate Token**
   - Name: `tekton-token`
   - Type: **Global Analysis Token**
   - Expiry: **No expiration**
4. Click **Generate** — **copy the token immediately** (starts with `sqa_`, shown only once)

---

### `tekton/tasks/maven-task.yaml`

**What it does:** Clones the GitHub repository into the shared workspace, then runs `mvn clean package` to compile the Java app and produce an executable JAR.

**Why clean the workspace first?** The Tekton shared workspace (a Kubernetes PVC) persists between steps in a pipeline run. If a previous pipeline run left files there, `git clone` would fail with "destination path already exists". Cleaning first ensures a fresh, reproducible build every time.

**Why `-DskipTests` in Maven?** Skipping tests keeps the build fast. In a more advanced pipeline, you'd run unit tests as a separate step so test results are clearly reported.

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
    - name: source   # Shared volume that all pipeline tasks read/write
  steps:
    - name: clone
      image: alpine/git   # Lightweight ~30MB image — only needs git
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        # Clean workspace to avoid "destination path already exists" error
        # (PVC may have files from a previous pipeline run)
        rm -rf ./* ./.* 2>/dev/null || true
        git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git .
        echo "✅ Repo cloned successfully"

    - name: maven-build
      image: maven:3.9.5-eclipse-temurin-17   # Maven + Java 17 — matches our app's Java version
      workingDir: $(workspaces.source.path)/app
      script: |
        #!/bin/sh
        # clean: remove previous build artifacts
        # package: compile + run tests + create JAR
        # -DskipTests: skip unit tests (run separately if needed)
        mvn clean package -DskipTests
        echo "✅ Maven build successful"
```

---

### `tekton/tasks/sonarqube-task.yaml`

**What it does:** Runs `mvn sonar:sonar` to analyze the compiled Java code and upload results to SonarQube for quality reporting.

**Why run after Maven build?** SonarQube needs the compiled `.class` files and any test reports generated by Maven to do a thorough analysis. It can analyze raw source code too, but compiled output provides richer insights.

**Why internal Kubernetes DNS?** `sonarqube-sonarqube.sonarqube.svc.cluster.local` is the internal Kubernetes service DNS name. Since Tekton pods and SonarQube both run inside the same Minikube cluster, they communicate directly through the cluster network — no public internet needed, no port-forward required, faster and more secure.

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: sonarqube-scan
  namespace: tekton-pipelines
spec:
  workspaces:
    - name: source   # Needs access to compiled code from previous maven-build task
  steps:
    - name: sonar-scan
      image: maven:3.9.5-eclipse-temurin-17
      workingDir: $(workspaces.source.path)/app
      script: |
        mvn sonar:sonar \
          -Dsonar.projectKey=gitops-app \
          -Dsonar.projectName=gitops-app \
          # Internal K8s DNS — format: service-name.namespace.svc.cluster.local
          # Tekton pod talks to SonarQube pod directly inside the cluster
          -Dsonar.host.url=http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
          # Token generated from SonarQube UI — authenticates this scan
          -Dsonar.login=YOUR_SONARQUBE_TOKEN
        echo "✅ SonarQube scan complete"
```

---

### `tekton/tasks/docker-task.yaml`

**What it does:** Builds the Docker image from the app's Dockerfile and pushes it to Docker Hub with both a version tag (e.g., `v1.0.0`) and the `latest` tag.

**Why Kaniko instead of Docker?** Building Docker images requires access to the Docker daemon (`/var/run/docker.sock`). Mounting this socket inside a Kubernetes pod is a major security risk — it gives the container root-level access to the host. **Kaniko** is a tool specifically designed to build container images inside Kubernetes **without** needing the Docker daemon. It builds entirely in userspace, reading the Dockerfile and constructing the image layer by layer. This is the industry-standard secure approach for building images in Kubernetes.

**Why push two tags?** The version tag (e.g., `v1.0.0`) is used by Helm for deployment tracking. The `latest` tag is convenient for testing and pulling the most recent build.

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
    - name: source   # Needs access to app source code and Dockerfile
  steps:
    - name: build-and-push
      # Kaniko executor — builds Docker images without Docker daemon
      image: gcr.io/kaniko-project/executor:latest
      workingDir: $(workspaces.source.path)/app
      args:
        - "--dockerfile=Dockerfile"
        - "--context=$(workspaces.source.path)/app"
        # Push with specific version tag for deployment tracking
        - "--destination=YOUR_DOCKERHUB_USERNAME/gitops-app:$(params.IMAGE_TAG)"
        # Also update 'latest' tag for convenience
        - "--destination=YOUR_DOCKERHUB_USERNAME/gitops-app:latest"
        - "--insecure"
        - "--skip-tls-verify"
      volumeMounts:
        # Kaniko reads Docker Hub credentials from /kaniko/.docker/config.json
        - name: kaniko-secret
          mountPath: /kaniko/.docker
  volumes:
    - name: kaniko-secret
      secret:
        secretName: kaniko-secret   # K8s secret containing Docker Hub auth
        items:
          - key: config.json
            path: config.json       # Mounted as /kaniko/.docker/config.json
```

---

### `tekton/tasks/approval-task.yaml`

**What it does:** Pauses the entire pipeline and waits for a human to approve or reject the production deployment by patching a Kubernetes ConfigMap. The task polls every 15 seconds. After 10 minutes (600s) with no decision, it automatically cancels.

**Why use a ConfigMap as the approval flag?** It's the simplest native Kubernetes solution — no extra tools, no webhooks, no external systems. The approver just runs one `kubectl patch` command from any terminal with cluster access. The polling approach means the task stays alive in the pipeline and keeps it paused until a decision is made.

**Why this matters in real organizations?** Production deployments require accountability. Using a manual approval gate ensures that a senior engineer or team lead reviews what's being deployed, when, and takes explicit responsibility for approving it. This is also a compliance requirement in many regulated industries.

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
      default: "600"   # 10 minutes to approve before auto-cancel
  steps:
    - name: wait-for-approval
      image: bitnami/kubectl:latest   # Has kubectl pre-installed
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

        # Reset ConfigMap to "pending" at the start of each approval
        # This prevents a previous "approved" from auto-approving the next run
        kubectl patch configmap pipeline-approval \
          -n tekton-pipelines \
          --type merge \
          -p '{"data":{"status":"pending"}}' 2>/dev/null || \
        kubectl create configmap pipeline-approval \
          -n tekton-pipelines \
          --from-literal=status=pending

        # Poll every 15 seconds until approved, rejected, or timeout
        TIMEOUT=$(params.TIMEOUT)
        ELAPSED=0
        while [ $ELAPSED -lt $TIMEOUT ]; do
          STATUS=$(kubectl get configmap pipeline-approval \
            -n tekton-pipelines \
            -o jsonpath='{.data.status}')

          echo "⏳ Awaiting approval... [$STATUS] (${ELAPSED}s / ${TIMEOUT}s)"

          if [ "$STATUS" = "approved" ]; then
            echo "✅ APPROVED! Proceeding to production deployment..."
            exit 0   # Exit 0 = success, pipeline continues to deploy-prod
          fi

          if [ "$STATUS" = "rejected" ]; then
            echo "❌ REJECTED! Production deployment cancelled."
            exit 1   # Exit 1 = failure, pipeline stops here
          fi

          sleep 15
          ELAPSED=$((ELAPSED + 15))
        done

        echo "⏰ Timed out after ${TIMEOUT}s — deployment cancelled"
        exit 1
```

---

### `tekton/tasks/deploy-to-env-task.yaml`

**What it does:** Updates the correct environment's Helm values file (`values-dev.yaml` or `values-prod.yaml`) with the new Docker image tag, commits the change, and pushes it to GitHub. ArgoCD then detects this Git change and deploys.

**Why update GitHub instead of deploying directly to Kubernetes?** This is the **GitOps principle** in action. Every deployment is a Git commit. This gives you a complete audit trail (who deployed what and when), easy rollback (just `git revert`), and the ability for ArgoCD to reconcile the cluster to Git state at any time. If someone manually changes a K8s resource, ArgoCD will revert it back to what Git says.

**Why SSH for Git push?** GitHub deprecated password-based Git authentication. SSH keys are the secure, recommended approach for server-to-server Git operations. We copy the key to `/tmp/.ssh/` (writable directory) because the Kubernetes secret volume is mounted as read-only.

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
      type: string   # "dev" or "prod" — determines which values file to update
  workspaces:
    - name: source
  steps:
    - name: update-helm-values
      image: alpine/git   # Lightweight — only needs git
      script: |
        #!/bin/sh
        # Copy SSH private key to writable location
        # Secret volumes are mounted read-only, so we must copy first
        mkdir -p /tmp/.ssh
        cp /ssh-key/ssh-privatekey /tmp/.ssh/id_ed25519
        chmod 600 /tmp/.ssh/id_ed25519   # SSH requires strict key file permissions
        ssh-keyscan github.com >> /tmp/.ssh/known_hosts   # Trust GitHub's host key
        export GIT_SSH_COMMAND="ssh -i /tmp/.ssh/id_ed25519 -o UserKnownHostsFile=/tmp/.ssh/known_hosts"

        # Clone a fresh copy of the repo (separate from the build workspace)
        rm -rf /tmp/gitops-repo
        git clone git@github.com:YOUR_USERNAME/YOUR_REPO.git /tmp/gitops-repo
        cd /tmp/gitops-repo

        # Update the image tag in the correct environment values file
        # sed replaces "tag: anything" with "tag: v2.0.0" (or whatever IMAGE_TAG is)
        # This is what triggers ArgoCD to detect a change and redeploy
        sed -i "s|tag: .*|tag: \"$(params.IMAGE_TAG)\"|" \
          helm/gitops-app/values-$(params.ENVIRONMENT).yaml

        echo "✅ Updated values-$(params.ENVIRONMENT).yaml:"
        cat helm/gitops-app/values-$(params.ENVIRONMENT).yaml

        # Commit and push — ArgoCD will detect this within ~3 minutes and deploy
        git config user.email "tekton@gitops.com"
        git config user.name "Tekton Pipeline"
        git add helm/gitops-app/values-$(params.ENVIRONMENT).yaml
        git commit -m "$(params.ENVIRONMENT): Update image tag to $(params.IMAGE_TAG)"
        git push git@github.com:YOUR_USERNAME/YOUR_REPO.git main
        echo "✅ $(params.ENVIRONMENT) deployment triggered via ArgoCD"
      volumeMounts:
        - name: ssh-key
          mountPath: /ssh-key   # Mount SSH secret here (read-only)
  volumes:
    - name: ssh-key
      secret:
        secretName: github-ssh-secret
        defaultMode: 0400   # Read-only — private key should never be world-readable
```

---

### `tekton/pipeline.yaml`

**What it does:** Orchestrates all tasks in the correct order using `runAfter` dependencies. The pipeline won't start the next task until the previous one succeeds. One shared workspace (PVC) is passed between all tasks so they can read each other's output.

**Why this sequence matters:** If Maven build fails (broken code), there's no point scanning broken code with SonarQube. If SonarQube finds critical issues, there's no point building a Docker image. If the image push fails, there's no point updating Helm values. This fail-fast approach saves time and prevents bad code from progressing further in the pipeline.

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
      default: "latest"   # Override this when triggering — e.g. "v2.0.0"
  workspaces:
    # Single shared volume passed between all tasks
    # All tasks read/write to the same cloned repository
    - name: shared-workspace
  tasks:
    # Task 1: Build Java app with Maven
    - name: maven-build
      taskRef:
        name: maven-build
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)

    # Task 2: Scan code quality with SonarQube (runs ONLY after maven-build succeeds)
    - name: sonarqube-scan
      taskRef:
        name: sonarqube-scan
      runAfter:
        - maven-build      # Won't run if maven-build fails
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 3: Build Docker image and push to Docker Hub (ONLY after sonar passes)
    - name: docker-build-push
      taskRef:
        name: docker-build-push
      runAfter:
        - sonarqube-scan   # Won't run if code quality check fails
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)

    # Task 4: Auto-deploy to DEV (no approval needed — developers get immediate feedback)
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

    # Task 5: PAUSE — wait for human approval before touching production
    - name: manual-approval
      taskRef:
        name: manual-approval
      runAfter:
        - deploy-dev       # Approval happens AFTER dev is verified working
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: TIMEOUT
          value: "600"     # 10 minutes to approve

    # Task 6: Deploy to PROD (ONLY after explicit human approval)
    - name: deploy-prod
      taskRef:
        name: deploy-to-env
      runAfter:
        - manual-approval  # Hard dependency — prod never deploys without approval
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE_TAG
          value: $(params.IMAGE_TAG)
        - name: ENVIRONMENT
          value: "prod"
```

---

### `tekton/pipelinerun.yaml`

**What it does:** Creates an actual execution instance of the `gitops-pipeline` with specific parameters (version tag). Each run gets a unique auto-generated name.

**Why `generateName` instead of a fixed `name`?** You run the pipeline many times (once per version). If you used a fixed name, the second run would fail because the first run's name already exists in Kubernetes. `generateName` automatically appends a random suffix (e.g., `gitops-pipeline-run-k69r4`), allowing unlimited runs.

**Why `kubectl create` not `kubectl apply`?** `apply` is idempotent — it updates an existing resource. `create` always creates a new one. Since `generateName` creates a new unique resource every time, you must use `create`. Using `apply` with `generateName` does not work.

**Why `volumeClaimTemplate`?** All tasks in the pipeline share the same workspace (cloned repo, compiled JAR). This needs a Kubernetes PersistentVolumeClaim (PVC) — a disk that persists between task steps. `volumeClaimTemplate` automatically creates a fresh PVC for each pipeline run and deletes it when done. No manual PVC management needed.

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: gitops-pipeline-run-   # Random suffix added: gitops-pipeline-run-abc12
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: gitops-pipeline
  params:
    - name: IMAGE_TAG
      value: "v1.0.0"   # ← Change this for each release: v1.0.0, v2.0.0, v3.0.0...
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce   # One node can read/write (fine for single-node Minikube)
          resources:
            requests:
              storage: 1Gi   # 1GB: enough for cloned repo + Maven cache + compiled JAR
```

---

### `tekton/approval-rbac.yaml`

**What it does:** Grants the Tekton service account permission to read and write ConfigMaps in the `tekton-pipelines` namespace.

**Why needed?** By default, Kubernetes pods run with very minimal permissions (principle of least privilege). The approval task needs to `get`, `create`, and `patch` the `pipeline-approval` ConfigMap. Without this RBAC rule, you'll get `Forbidden` errors and the approval task will fail even though the pipeline logic is correct.

**Why ClusterRoleBinding for lab?** For simplicity in this lab, we grant `cluster-admin` (full access). In production, you'd create a narrower Role that only allows `get/create/patch` on ConfigMaps in the specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: approval-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin   # Full access for lab — use a narrower role in production
subjects:
  - kind: ServiceAccount
    name: default
    namespace: tekton-pipelines   # The service account that Tekton pods run as
```

---

## 🔐 Secrets Configuration

**Why Kubernetes Secrets?** Never hardcode credentials (Docker Hub tokens, SSH keys, API tokens) in YAML files, pipeline code, or environment variables in plain text. Kubernetes Secrets store sensitive data base64-encoded (and can be encrypted at rest), and inject them into pods securely as mounted volumes or environment variables. If your YAML files are committed to a public GitHub repo, no credentials are exposed.

### GitHub SSH Key Setup

**What this does:** Creates an SSH key pair. The public key goes to GitHub (so GitHub trusts requests from your EC2 server). The private key is stored as a Kubernetes Secret (so Tekton tasks can authenticate when pushing to GitHub).

**Why SSH instead of Personal Access Token?** SSH keys are more secure for server automation. They never expire by default, are tied to a specific key pair (easy to revoke), and can't be accidentally leaked in URLs. For Tekton tasks that push to GitHub, SSH is the standard approach.

```bash
# Generate SSH key pair — ed25519 is modern, secure algorithm
# -N "" means no passphrase (required for unattended automation)
ssh-keygen -t ed25519 -C "your-email@gmail.com" -f ~/.ssh/id_ed25519 -N ""

# Display the PUBLIC key — copy this entire line
cat ~/.ssh/id_ed25519.pub

# Add to GitHub:
# GitHub → Settings → SSH and GPG Keys → New SSH Key
# Title: "ec2-tekton-server"
# Paste the public key

# Test the connection — should say "Hi username! You've successfully authenticated"
ssh -T git@github.com

# Switch your repo remote from HTTPS to SSH
git remote set-url origin git@github.com:YOUR_USERNAME/YOUR_REPO.git
```

### Kubernetes Secrets

```bash
# Secret 1: Docker Hub credentials for Kaniko
# Kaniko expects Docker auth at /kaniko/.docker/config.json in this exact JSON format
# We create a K8s secret with the config.json key containing the auth JSON
kubectl create secret generic kaniko-secret \
  --from-literal=config.json="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"$(echo -n 'YOUR_DOCKERHUB_USERNAME:YOUR_DOCKERHUB_TOKEN' | base64 -w 0)\"}}}" \
  -n tekton-pipelines

# Secret 2: GitHub SSH private key for Tekton to push Helm value updates
# type=kubernetes.io/ssh-auth → K8s knows this is an SSH key type secret
kubectl create secret generic github-ssh-secret \
  --from-file=ssh-privatekey=/home/ubuntu/.ssh/id_ed25519 \
  --type=kubernetes.io/ssh-auth \
  -n tekton-pipelines

# Verify both secrets created successfully
kubectl get secrets -n tekton-pipelines
```

> **How to generate Docker Hub token:**
> hub.docker.com → Account Settings → Security → New Access Token
> Name: `tekton-token` | Permissions: **Read & Write**
> Copy the token — it's shown only once

---

## 🔄 ArgoCD Setup

### `argocd/dev-application.yaml` — Auto sync (DEV)

**What it does:** Registers the dev application with ArgoCD. ArgoCD continuously watches the `helm/gitops-app/` path in your GitHub repo. When it detects any change in `values-dev.yaml` (e.g., new image tag pushed by Tekton), it automatically renders the Helm templates with the combined base + dev values and applies the resulting Kubernetes manifests to the `dev` namespace.

**Why `automated` sync for dev?** Dev is the testing environment. Developers want to see their changes deployed immediately — no manual steps, no waiting. Fast feedback is the whole point of dev.

**Why `prune: true`?** If you remove a resource from your Helm chart (e.g., delete a Service), ArgoCD will also delete it from the cluster. Without prune, deleted resources linger forever.

**Why `selfHeal: true`?** If someone manually changes a Kubernetes resource (e.g., `kubectl edit deployment gitops-app`), ArgoCD automatically reverts it back to match what's in Git. Git is the source of truth — manual changes are not allowed.

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
    targetRevision: main   # Watch the main branch
    path: helm/gitops-app  # Location of Helm chart in repo
    helm:
      valueFiles:
        - values.yaml       # First: apply base/common values
        - values-dev.yaml   # Then: apply dev overrides (nodePort, image tag)
  destination:
    server: https://kubernetes.default.svc   # This Minikube cluster
    namespace: dev                           # Deploy into dev namespace
  syncPolicy:
    automated:
      prune: true       # Delete K8s resources removed from Helm chart
      selfHeal: true    # Revert manual K8s changes to match Git state
    syncOptions:
      - CreateNamespace=true   # Auto-create 'dev' namespace if it doesn't exist
```

### `argocd/prod-application.yaml` — Manual sync (PROD)

**What it does:** Same as dev, but without the `automated` sync block. ArgoCD watches the repo and knows when `values-prod.yaml` changes, but it **waits for a human to trigger sync**. Nothing deploys to production automatically.

**Why no auto-sync for prod?** Even with the Tekton approval gate, having a second human decision point (ArgoCD manual sync) adds another layer of safety. In real organizations, the person who approves in Tekton and the person who clicks sync in ArgoCD might be different roles — developer lead approves the code quality, ops team approves the deployment window.

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
        - values.yaml        # Base/common values
        - values-prod.yaml   # Prod overrides (nodePort 30082, prod image tag)
  destination:
    server: https://kubernetes.default.svc
    namespace: prod          # Deploy into prod namespace
  syncPolicy:
    # ← No 'automated' block = manual sync only
    # ArgoCD detects changes but waits for: argocd app sync gitops-app-prod
    syncOptions:
      - CreateNamespace=true
```

### Apply ArgoCD apps:

```bash
kubectl apply -f argocd/dev-application.yaml
kubectl apply -f argocd/prod-application.yaml

# Verify both apps are registered with ArgoCD
kubectl get applications -n argocd
```

---

## 🚀 Running the Pipeline

### Apply all Tekton resources:

```bash
# Step 1: Apply RBAC first — tasks need permission to access ConfigMaps
kubectl apply -f tekton/approval-rbac.yaml

# Step 2: Apply each task (registers the task definition in Kubernetes)
kubectl apply -f tekton/tasks/maven-task.yaml
kubectl apply -f tekton/tasks/sonarqube-task.yaml
kubectl apply -f tekton/tasks/docker-task.yaml
kubectl apply -f tekton/tasks/approval-task.yaml
kubectl apply -f tekton/tasks/deploy-to-env-task.yaml

# Step 3: Apply the pipeline (orchestrates all tasks)
kubectl apply -f tekton/pipeline.yaml

# Step 4: Verify — all 5 tasks and 1 pipeline should be listed
tkn task list -n tekton-pipelines
tkn pipeline list -n tekton-pipelines
```

### Trigger a pipeline run:

```bash
# Edit pipelinerun.yaml first — change IMAGE_TAG to your new version
# Then create the run (must use 'create', not 'apply' — generateName requires it)
kubectl create -f tekton/pipelinerun.yaml
```

### Watch live logs:

```bash
# Stream logs from all tasks in real time
# You'll see Maven output, SonarQube analysis, Kaniko build, etc.
tkn pipelinerun logs --last -f -n tekton-pipelines
```

### Check pipeline status:

```bash
# List all runs and their status (Running/Succeeded/Failed)
tkn pipelinerun list -n tekton-pipelines
```

---

## 🌍 Dev vs Production Workflow

| Feature | DEV | PROD |
|---------|-----|------|
| Namespace | `dev` | `prod` |
| NodePort | 30081 | 30082 |
| ArgoCD Sync | **Automatic** (on every Git change) | **Manual** (`argocd app sync`) |
| Replicas | 1 | 1 (set to 2+ on real multi-node clusters) |
| Pipeline Approval | Not required | **Required** (manual-approval task) |
| Values file updated by Tekton | `values-dev.yaml` | `values-prod.yaml` |
| Who triggers deployment? | ArgoCD (automatic) | Human (argocd sync command) |

```
New Docker image built and pushed to Docker Hub
              │
              ▼
      deploy-dev task runs:
      → updates values-dev.yaml: tag: "v2.0.0"
      → git commit + push to GitHub
      → ArgoCD detects change within ~3 minutes
      → ArgoCD auto-syncs DEV namespace ✅
      → Dev pod is updated with new image
              │
              ▼
      manual-approval task:
      → Pipeline PAUSES ⏸️
      → Team lead reviews dev deployment
      → Runs kubectl patch to approve
              │
         ┌────┴────┐
         │         │
      Approved   Rejected
         │         │
         ▼         ▼
  deploy-prod  Pipeline
     task:      STOPS ❌
  → updates values-prod.yaml
  → git push
  → ArgoCD detects change
  → Team lead runs:
    argocd app sync gitops-app-prod
  → PROD namespace updated ✅
```

---

## ⏸️ Manual Approval Gate

**When the pipeline reaches the approval stage, you'll see this in the Tekton logs:**

```
================================================
⏸️  PRODUCTION DEPLOYMENT - MANUAL APPROVAL REQUIRED
================================================
📦 Image: YOUR_USERNAME/gitops-app:v2.0.0
🌍 Target: PRODUCTION namespace
⏳ Awaiting approval... [pending] (0s / 600s)
```

### Open a SECOND terminal and run one of these:

**To APPROVE production deployment:**
```bash
kubectl patch configmap pipeline-approval \
  -n tekton-pipelines \
  --type merge \
  -p '{"data":{"status":"approved"}}'
```

**To REJECT production deployment:**
```bash
kubectl patch configmap pipeline-approval \
  -n tekton-pipelines \
  --type merge \
  -p '{"data":{"status":"rejected"}}'
```

**If ConfigMap doesn't exist yet (first run), create it directly as approved:**
```bash
# The approval task may not have created it yet if it started very recently
kubectl create configmap pipeline-approval \
  -n tekton-pipelines \
  --from-literal=status=approved
```

### After approval — manually sync ArgoCD prod:

```bash
# Restart port-forward if it died
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0 &

# Login to ArgoCD CLI
argocd login localhost:8080 --username admin --password YOUR_ARGOCD_PASSWORD --insecure

# Sync the prod application — this triggers the actual deployment
argocd app sync gitops-app-prod

# Verify prod pods are running
kubectl get pods -n prod
kubectl get pods -n dev
```

---

## 🔍 Troubleshooting

### Port-forwards stopped working

**Why it happens:** Port-forwards are background processes tied to your terminal session. They die when the session closes or times out.

```bash
# Restart both port-forwards
kubectl port-forward -n sonarqube svc/sonarqube-sonarqube 9000:9000 --address 0.0.0.0 &
kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0 &
```

---

### Git push rejected — "fetch first"

**Why it happens:** Tekton pushed a commit to GitHub (updating values.yaml with new image tag). Your local repo doesn't have that commit, so Git rejects your push to prevent overwriting it.

```bash
git stash                        # Temporarily save your local uncommitted changes
git pull origin main --rebase    # Fetch remote commits, replay your commits on top
git stash pop                    # Restore your local changes
git add .
git commit -m "your message"
git push origin main
```

---

### Tekton pods failing with PodSecurity error

**Why it happens:** Kubernetes 1.25+ enforces Pod Security Standards. The `tekton-pipelines` namespace defaults to `restricted` which blocks Tekton's internal init containers (they need to run as root).

```bash
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite
```

---

### Maven clone fails — "directory not empty"

**Why it happens:** The Tekton shared workspace PVC already contains files from a previous pipeline run.

**Fix:** The `rm -rf ./*` line in `maven-task.yaml` handles this. If you see this error, make sure that line is present in your maven task's clone step.

---

### Docker push UNAUTHORIZED

**Why it happens:** The Kaniko secret format is wrong or the secret isn't mounted at the correct path (`/kaniko/.docker/config.json`).

```bash
# Recreate the secret with the exact correct format
kubectl delete secret kaniko-secret -n tekton-pipelines
kubectl create secret generic kaniko-secret \
  --from-literal=config.json="{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"$(echo -n 'YOUR_USERNAME:YOUR_TOKEN' | base64 -w 0)\"}}}" \
  -n tekton-pipelines
```

---

### Pods Pending — Insufficient CPU or Memory

**Why it happens:** Minikube only has 2 CPUs. When SonarQube, ArgoCD, Tekton, and app pods all run simultaneously, CPU can run out.

```bash
# Check what's consuming resources on the node
kubectl describe node minikube | grep -A 15 "Allocated resources"

# Fix: reduce resource requests in your values files
# Change requests.cpu to "100m" and requests.memory to "128Mi"
# This is safe for a Minikube lab environment
```

---

### Approval ConfigMap not found when trying to patch

**Why it happens:** The approval task creates the ConfigMap when it starts. If it hasn't started yet or failed very early, the ConfigMap doesn't exist.

```bash
# Create it manually with approved status — the task will pick it up within 15 seconds
kubectl create configmap pipeline-approval \
  -n tekton-pipelines \
  --from-literal=status=approved
```

---

### Check pipeline run details and events

```bash
# Get detailed description of the latest pipeline run
tkn pipelinerun describe \
  $(tkn pipelinerun list -n tekton-pipelines -o name | head -1) \
  -n tekton-pipelines

# Check Kubernetes events — shows exactly why a pod failed
kubectl get events -n tekton-pipelines --sort-by='.lastTimestamp' | tail -20
```

---

## 📊 Verify Everything is Running

```bash
# Check all components
kubectl get pods -n tekton-pipelines   # Should show: controller, webhook, dashboard Running
kubectl get pods -n sonarqube          # Should show: sonarqube-sonarqube-0 1/1 Running
kubectl get pods -n argocd             # Should show: 7 pods all Running
kubectl get pods -n dev                # Should show: gitops-app-xxx 1/1 Running
kubectl get pods -n prod               # Should show: gitops-app-xxx 1/1 Running

# Check ArgoCD sync status for all environments
kubectl get applications -n argocd

# Check pipeline run history
tkn pipelinerun list -n tekton-pipelines
```

---

## 🎯 Key Concepts Learned

| Concept | What It Means | Why It Matters |
|---------|--------------|----------------|
| **GitOps** | Git is the single source of truth for deployments | Full audit trail, easy rollback, team collaboration — every deployment is a Git commit |
| **Helm Values Override Pattern** | One chart + `values-dev.yaml` + `values-prod.yaml` | No YAML duplication, consistent structure, easy to manage multiple environments |
| **Tekton Cloud-Native CI** | Every pipeline step runs as a Kubernetes pod | No Jenkins server to maintain, scales with K8s, uses native K8s secrets |
| **Kaniko** | Builds Docker images without Docker daemon inside K8s | Secure image building — no root socket exposure in pods |
| **ArgoCD Auto-sync** | Watches Git, deploys on change | Dev gets instant feedback; no manual deployment steps |
| **Manual Approval Gate** | Pipeline pauses for human review before prod | Safety, compliance, accountability — someone owns production changes |
| **Pod Security Labels** | Kubernetes namespace security policies | Required for Tekton init containers to run — `restricted` blocks them by default |
| **Kubernetes Secrets** | Encrypted credential storage injected into pods | Never hardcode passwords in code or YAML files |
| **Multi-stage Dockerfile** | Build in large image, run in small image | Smaller (~80MB vs ~500MB), faster, more secure production containers |
| **Internal K8s DNS** | `service.namespace.svc.cluster.local` | Secure, fast pod-to-pod communication without going through public internet |
| **readinessProbe** | K8s checks app health before routing traffic | Users never hit a pod that's still starting up |
| **livenessProbe** | K8s restarts pod if app stops responding | Self-healing — crashed apps recover automatically |

---

*Built with ❤️ — GitHub → Tekton → Maven → SonarQube → Docker → DockerHub → ArgoCD → Minikube*
