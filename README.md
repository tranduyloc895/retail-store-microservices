# Retail Store Microservices

Source code for the microservices of the **DevSecOps E-commerce** project (NT114 — UIT). The application models an online retail store with a microservices architecture and serves as the workload for the CI/CD pipeline and GitOps deployment on Kubernetes (EKS).

> Upstream source: [aws-containers/retail-store-sample-app](https://github.com/aws-containers/retail-store-sample-app) (MIT-0 License). Trimmed to keep only the microservice code plus the Jenkinsfiles.

---

## Table of Contents

- [Application Architecture](#application-architecture)
- [Microservices](#microservices)
- [Directory Structure](#directory-structure)
- [Run Locally with Docker Compose](#run-locally-with-docker-compose)
- [CI/CD Pipeline (Jenkins + GitOps)](#cicd-pipeline-jenkins--gitops)
- [Build Each Service](#build-each-service)
- [Pipeline Onboarding Status](#pipeline-onboarding-status)
- [Cleanup After Each Lab](#cleanup-after-each-lab)
- [Related Repos](#related-repos)

---

## Application Architecture

```
                    ┌──────────────────────────────┐
                    │           UI (Java)          │
                    │       Spring Boot :8080      │
                    │    Web frontend, Thymeleaf   │
                    └──┬──────┬──────┬──────┬──────┘
                       │      │      │      │
            ┌──────────▼┐ ┌───▼────┐ ▼      ▼
            │  Catalog  │ │  Cart  │ Orders  Checkout
            │   (Go)    │ │ (Java) │ (Java)  (TypeScript)
            │  Gin :8080│ │  :8080 │ :8080    NestJS :8080
            └──────┬────┘ └───┬────┘ ┌┴───┐   ┌──┴──┐
                   │          │      │    │   │     │
               MariaDB    DynamoDB  PG   RMQ Redis  │
               (catalog)  (carts) (orders)  (checkout)
```

## Microservices

| Service | Language | Framework | Port | Database | Description |
|---------|----------|-----------|------|----------|-------------|
| **UI** | Java 21 | Spring Boot 3.5 + Thymeleaf | 8080 | — | Web frontend; calls every other service |
| **Catalog** | Go | Gin | 8080 | MariaDB | Product catalog API |
| **Cart** | Java 21 | Spring Boot 3.5 | 8080 | DynamoDB (local) | Shopping cart API |
| **Orders** | Java 21 | Spring Boot 3.5 | 8080 | PostgreSQL + RabbitMQ | Orders management API |
| **Checkout** | TypeScript | NestJS 11 | 8080 | Redis | Checkout / payment API |

---

## Directory Structure

```
retail-store-microservices/
├── src/
│   ├── ui/                    # Web UI (Java/Spring Boot)
│   │   ├── Dockerfile         #   Multi-stage build, non-root user UID 10001
│   │   ├── Jenkinsfile        #   Pipeline (Build → Push → Update GitOps)
│   │   ├── pom.xml
│   │   └── src/
│   │
│   ├── catalog/               # Product Catalog (Go/Gin)
│   │   ├── Dockerfile
│   │   ├── Jenkinsfile
│   │   ├── main.go
│   │   ├── go.mod
│   │   └── (controller, model, repository, ...)
│   │
│   ├── cart/                  # Shopping Cart (Java/Spring Boot)
│   │   ├── Dockerfile
│   │   ├── Jenkinsfile
│   │   ├── pom.xml
│   │   └── src/
│   │
│   ├── checkout/              # Checkout (TypeScript/NestJS)
│   │   ├── Dockerfile
│   │   ├── Jenkinsfile
│   │   ├── package.json
│   │   └── src/
│   │
│   ├── orders/                # Orders (Java/Spring Boot)
│   │   ├── Dockerfile
│   │   ├── Jenkinsfile
│   │   ├── pom.xml
│   │   └── src/
│   │
│   └── app/
│       └── docker-compose.yml # Orchestration for local dev
│
├── .gitignore
├── LICENSE
└── README.md
```

---

## Run Locally with Docker Compose

### Requirements

- [Docker](https://docs.docker.com/get-docker/) >= 24.0
- [Docker Compose](https://docs.docker.com/compose/) >= 2.20

### Start all services

```bash
# Set the password for the databases
export DB_PASSWORD=your_password_here

# Start
docker compose --project-directory src/app up --build -d

# Check status
docker compose --project-directory src/app ps
```

Open: `http://localhost:8888`

### Port mapping

| Service | URL |
|---------|-----|
| UI | `http://localhost:8888` |
| Catalog API | `http://localhost:8081` |
| Cart API | `http://localhost:8082` |
| Orders API | `http://localhost:8083` |
| Checkout API | `http://localhost:8085` |

### Stop services

```bash
docker compose --project-directory src/app down
```

---

## CI/CD Pipeline (Jenkins + GitOps)

The pipeline follows the **GitOps** model: Jenkins builds the image, pushes to ECR, and updates the manifest in the `retail-store-gitops` repo. Applying manifests to the cluster is **ArgoCD**'s job.

### Full flow

```
Developer pushes code (this repo)
        │
        ▼
Jenkins Agent triggers the pipeline
        │
        ├─── Stage 1: Build Docker Image
        │       • git rev-parse --short=7 HEAD → IMAGE_TAG
        │       • docker build -t <ECR>/<repo>:<tag>
        │
        ├─── Stage 2: Push to ECR
        │       • aws ecr get-login-password | docker login
        │       • docker push
        │
        └─── Stage 3: Update GitOps
                • git clone retail-store-gitops
                • sed replaces the image tag in apps/<service>/deployment.yml
                • git commit + push to main
                        │
                        ▼
                ArgoCD polls (every 3 min) / webhook
                        │
                        ▼
                ArgoCD sync → kubectl apply into EKS
                        │
                        ▼
                Rolling pod update (K8s)
                        │
                        ▼
                New version live behind the ELB
```

### Jenkinsfile (used by every service, parameterized)

Located at `src/<service>/Jenkinsfile`. Three stages:

```groovy
pipeline {
    agent { label 'docker-agent' }

    environment {
        AWS_REGION    = 'ap-southeast-1'
        ECR_REPO_NAME = 'retail-store/<service>'
        EKS_CLUSTER   = 'ecommerce-cluster'
    }

    stages {
        stage('Build Docker Image') { /* ... */ }
        stage('Push to ECR')        { /* ... */ }
        stage('Update GitOps')      { /* ... */ }
    }
}
```

**Key technical decisions:**

| Concern | Solution |
|---------|----------|
| Image tag must trace back to a commit | `git rev-parse --short=7 HEAD` as the tag |
| AWS Account ID must not be hard-coded | Stored in the Jenkins Credential `aws-account-id` (Secret text) |
| GitHub token must not leak in logs | `withCredentials` + single-quoted shell strings |
| Dockerfile base image has UID 1000 in use | Create the app user with UID 10001 to avoid conflicts |
| `sed` can silent-fail | After `sed`, `grep` verifies the new tag is present |
| Pipeline commits even when nothing changed | `git diff --staged --quiet \|\| git commit` (idempotent) |

### Credentials to create in Jenkins

| Credential ID | Kind | Purpose |
|---------------|------|---------|
| `aws-account-id` | Secret text | AWS Account ID (used during ECR login) |
| `jenkins-agent-ssh` | SSH Username with private key | Master connects to the Agent over SSH |
| `github-gitops-token` | Username with password | Jenkins pushes manifest changes to `retail-store-gitops` |

**GitHub token permissions** (least-privilege):
- Repository: only `retail-store-gitops`
- Permissions: **Contents: Read and write** + **Metadata: Read** (required automatically)
- Do not grant Administration, Secrets, Webhooks, Actions, or any other permission.

### Pipeline job setup in Jenkins

- **Type:** Pipeline
- **Pipeline definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** `<URL of this repo>`
- **Branch:** `*/main`
- **Script Path:** `src/<service>/Jenkinsfile` (one job per service)

### Automatic trigger (roadmap)

Currently the pipeline is triggered manually (click "Build Now"). To automate:

**Option A — GitHub webhook → Jenkins:**
1. Jenkins needs a public URL (the current port-forward is local only).
2. In the GitHub repo → Settings → Webhooks → add `https://<jenkins-url>/github-webhook/`.
3. Configure the pipeline: `triggers { githubPush() }`.

**Option B — SCM polling** (simpler, no need to expose Jenkins):
```groovy
triggers {
    pollSCM('H/5 * * * *')  // check the repo every 5 minutes
}
```

---

## Build Each Service

```bash
# UI
docker build -t retail-store/ui:latest src/ui/

# Catalog
docker build -t retail-store/catalog:latest src/catalog/

# Cart
docker build -t retail-store/cart:latest src/cart/

# Checkout
docker build -t retail-store/checkout:latest src/checkout/

# Orders
docker build -t retail-store/orders:latest src/orders/
```

---

## Pipeline Onboarding Status

| Service | Dockerfile | Jenkinsfile | GitOps manifests | ArgoCD App | Status |
|---------|-----------|-------------|------------------|------------|--------|
| UI | Yes | Yes | Yes | Yes | **Live end-to-end** |
| Catalog | Yes | Yes | Yes | Yes | **Live end-to-end** |
| Cart | Yes | Yes | Yes | Yes | **Live end-to-end** |
| Orders | Yes | Yes | Yes | Yes | **Live end-to-end** |
| Checkout | Yes | Yes | Yes | Yes | **Live end-to-end** |

All 5 services have been onboarded. To add a 6th service, follow the **"Adding a New Service"** guide in the [retail-store-gitops](https://github.com/tranduyloc895/retail-store-gitops) README.

---

## Cleanup After Each Lab

Source code itself costs nothing, but leaving the local Docker Compose stack running consumes memory and disk. After a local dev session:

```bash
# Stop and remove containers, networks, and images created by compose
docker compose --project-directory src/app down -v

# (Optional) Prune dangling images to reclaim disk
docker image prune -f
```

For the full cloud-side teardown (EKS, Jenkins, NAT GW, ELB), see `Cleanup After Each Lab` in the `infrastructure` and `retail-store-gitops` READMEs.

---

## Related Repos

| Repo | Role |
|------|------|
| [infrastructure](https://github.com/tranduyloc895/infrastructure) | Terraform + Ansible: VPC, EKS, Jenkins, ECR |
| **retail-store-microservices** (this repo) | Source code for the 5 microservices + Jenkinsfile |
| [retail-store-gitops](https://github.com/tranduyloc895/retail-store-gitops) | K8s manifests + ArgoCD Applications |

---

> *NT114 course project — University of Information Technology (UIT)*

## License

Based on [AWS Containers Retail Sample](https://github.com/aws-containers/retail-store-sample-app) — MIT-0 License.
