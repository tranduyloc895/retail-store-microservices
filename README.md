# Retail Store Microservices

Source code microservices cho dự án **DevSecOps E-commerce** (NT114 - UIT). Ứng dụng mô phỏng một cửa hàng bán lẻ trực tuyến với kiến trúc microservices, phục vụ làm workload cho pipeline CI/CD và triển khai lên Kubernetes (EKS) qua GitOps.

> Source code gốc: [aws-containers/retail-store-sample-app](https://github.com/aws-containers/retail-store-sample-app) (MIT-0 License). Đã được tinh gọn, chỉ giữ lại phần microservice code + thêm Jenkinsfile.

---

## Mục lục

- [Kiến trúc ứng dụng](#kiến-trúc-ứng-dụng)
- [Danh sách microservices](#danh-sách-microservices)
- [Cấu trúc thư mục](#cấu-trúc-thư-mục)
- [Chạy local bằng Docker Compose](#chạy-local-bằng-docker-compose)
- [Pipeline CI/CD (Jenkins + GitOps)](#pipeline-cicd-jenkins--gitops)
- [Build từng service](#build-từng-service)
- [Trạng thái onboarding pipeline](#trạng-thái-onboarding-pipeline)
- [Liên kết repo](#liên-kết-repo)

---

## Kiến trúc ứng dụng

```
                    ┌──────────────────────────────┐
                    │           UI (Java)           │
                    │        Spring Boot :8080      │
                    │   Giao diện web, Thymeleaf    │
                    └──┬──────┬──────┬──────┬──────┘
                       │      │      │      │
            ┌──────────▼┐ ┌───▼────┐ ▼      ▼
            │  Catalog   │ │  Cart  │ Orders  Checkout
            │   (Go)     │ │ (Java) │ (Java)  (TypeScript)
            │  Gin :8080 │ │  :8080 │ :8080    NestJS :8080
            └──────┬─────┘ └───┬───┘ ┌┴───┐    ┌──┴──┐
                   │           │     │    │    │     │
               MariaDB     DynamoDB  PG  RMQ  Redis  │
               (catalog)   (carts) (orders)  (checkout)
```

## Danh sách microservices

| Service | Ngôn ngữ | Framework | Port | Database | Mô tả |
|---------|----------|-----------|------|----------|-------|
| **UI** | Java 21 | Spring Boot 3.5 + Thymeleaf | 8080 | - | Giao diện web, gọi tới các service khác |
| **Catalog** | Go | Gin | 8080 | MariaDB | API danh mục sản phẩm |
| **Cart** | Java 21 | Spring Boot 3.5 | 8080 | DynamoDB (local) | API giỏ hàng |
| **Orders** | Java 21 | Spring Boot 3.5 | 8080 | PostgreSQL + RabbitMQ | API quản lý đơn hàng |
| **Checkout** | TypeScript | NestJS 11 | 8080 | Redis | API xử lý thanh toán |

---

## Cấu trúc thư mục

```
retail-store-microservices/
├── src/
│   ├── ui/                    # Web UI (Java/Spring Boot)
│   │   ├── Dockerfile         #   Multi-stage build, non-root user UID 10001
│   │   ├── Jenkinsfile        #   Pipeline CI/CD (Build → Push → Update GitOps)
│   │   ├── pom.xml
│   │   └── src/
│   │
│   ├── catalog/               # Product Catalog (Go/Gin)
│   │   ├── Dockerfile
│   │   ├── main.go
│   │   ├── go.mod
│   │   └── (controller, model, repository...)
│   │
│   ├── cart/                  # Shopping Cart (Java/Spring Boot)
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │
│   ├── checkout/              # Checkout (TypeScript/NestJS)
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │
│   ├── orders/                # Orders (Java/Spring Boot)
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │
│   └── app/
│       └── docker-compose.yml # Orchestration cho local dev
│
├── .gitignore
├── LICENSE
└── README.md
```

---

## Chạy local bằng Docker Compose

### Yêu cầu

- [Docker](https://docs.docker.com/get-docker/) >= 24.0
- [Docker Compose](https://docs.docker.com/compose/) >= 2.20

### Khởi chạy toàn bộ services

```bash
# Set password cho databases
export DB_PASSWORD=your_password_here

# Khởi chạy
docker compose --project-directory src/app up --build -d

# Kiểm tra trạng thái
docker compose --project-directory src/app ps
```

Truy cập: `http://localhost:8888`

### Port mapping

| Service | URL |
|---------|-----|
| UI | `http://localhost:8888` |
| Catalog API | `http://localhost:8081` |
| Cart API | `http://localhost:8082` |
| Orders API | `http://localhost:8083` |
| Checkout API | `http://localhost:8085` |

### Dừng services

```bash
docker compose --project-directory src/app down
```

---

## Pipeline CI/CD (Jenkins + GitOps)

Pipeline áp dụng mô hình **GitOps**: Jenkins build image + push ECR + cập nhật manifest trong repo `retail-store-gitops`, còn việc apply lên cluster do **ArgoCD** đảm nhận.

### Luồng hoàn chỉnh

```
Developer push code (repo này)
        │
        ▼
Jenkins Agent trigger pipeline
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
                • sed thay image tag trong apps/<service>/deployment.yml
                • git commit + push lên main
                        │
                        ▼
                ArgoCD poll (mỗi 3 phút) / webhook
                        │
                        ▼
                ArgoCD sync → kubectl apply vào EKS
                        │
                        ▼
                Rolling update pod (K8s)
                        │
                        ▼
                Version mới live trên ELB
```

### Jenkinsfile của UI (tham khảo khi viết cho service khác)

File: `src/ui/Jenkinsfile`. Cấu trúc 3 stage:

```groovy
pipeline {
    agent { label 'docker-agent' }

    environment {
        AWS_REGION    = 'ap-southeast-1'
        ECR_REPO_NAME = 'retail-store/ui'
        EKS_CLUSTER   = 'ecommerce-cluster'
    }

    stages {
        stage('Build Docker Image') { /* ... */ }
        stage('Push to ECR')         { /* ... */ }
        stage('Update GitOps')       { /* ... */ }
    }
}
```

**Điểm kỹ thuật quan trọng:**

| Vấn đề | Giải pháp |
|--------|-----------|
| Image tag phải trace được về commit | `git rev-parse --short=7 HEAD` làm tag |
| AWS Account ID không được hardcode | Lưu trong Jenkins Credential `aws-account-id` (Secret text) |
| GitHub token không được leak trong log | Dùng `withCredentials` + single-quoted shell string |
| Dockerfile base image có sẵn user UID 1000 | Tạo app user với UID 10001 để tránh conflict |
| `sed` thay image tag có thể silent-fail | Sau sed phải `grep` verify tag mới xuất hiện |
| Pipeline commit Git nhưng không có thay đổi | `git diff --staged --quiet \|\| git commit` (idempotent) |

### Credentials cần tạo trong Jenkins

| Credential ID | Kind | Mục đích |
|---------------|------|----------|
| `aws-account-id` | Secret text | AWS Account ID (dùng khi login ECR) |
| `jenkins-agent-ssh` | SSH Username with private key | Master kết nối SSH vào Agent |
| `github-gitops-token` | Username with password | Jenkins push manifest vào repo `retail-store-gitops` |

**Quyền token GitHub** (nguyên tắc least-privilege):
- Repository: chỉ `retail-store-gitops`
- Permissions: **Contents: Read and write** + **Metadata: Read** (bắt buộc auto)
- Không cấp Administration, Secrets, Webhooks, Actions, hay bất kỳ quyền nào khác.

### Pipeline Job setup trong Jenkins

- **Type**: Pipeline
- **Pipeline definition**: Pipeline script from SCM
- **SCM**: Git
- **Repository URL**: `<URL repo này>`
- **Branch**: `*/main`
- **Script Path**: `src/ui/Jenkinsfile` (hoặc `src/<service>/Jenkinsfile`)

### Trigger tự động (roadmap)

Hiện tại pipeline được trigger thủ công (click "Build Now"). Để tự động:

**Option A — Webhook GitHub → Jenkins**:
1. Jenkins cần URL public (port-forward hiện tại chỉ local)
2. Tại repo GitHub → Settings → Webhooks → thêm URL `https://<jenkins-url>/github-webhook/`
3. Cấu hình pipeline: `triggers { githubPush() }`

**Option B — Polling SCM** (đơn giản hơn, không cần expose Jenkins):
```groovy
triggers {
    pollSCM('H/5 * * * *')  // check repo mỗi 5 phút
}
```

---

## Build từng service

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

## Trạng thái onboarding pipeline

| Service | Dockerfile | Jenkinsfile | Manifest GitOps | ArgoCD App | Trạng thái |
|---------|-----------|-------------|-----------------|------------|-----------|
| UI | ✅ | ✅ | ✅ | ✅ | **Hoạt động end-to-end** |
| Catalog | ✅ | ⏳ | ⏳ | ⏳ | Chưa onboard |
| Cart | ✅ | ⏳ | ⏳ | ⏳ | Chưa onboard |
| Orders | ✅ | ⏳ | ⏳ | ⏳ | Chưa onboard |
| Checkout | ✅ | ⏳ | ⏳ | ⏳ | Chưa onboard |

Để onboard service mới, xem hướng dẫn **"Thêm service mới"** trong README của repo [retail-store-gitops](https://github.com/tranduyloc895/retail-store-gitops).

---

## Liên kết repo

| Repo | Vai trò |
|------|---------|
| [infrastructure](https://github.com/tranduyloc895/infrastructure) | Terraform + Ansible: VPC, EKS, Jenkins, ECR |
| **retail-store-microservices** (this repo) | Source code 5 microservices + Jenkinsfile |
| [retail-store-gitops](https://github.com/tranduyloc895/retail-store-gitops) | Manifests K8s + ArgoCD Application |

---

> *Đồ án môn NT114 - Đại học Công nghệ Thông tin (UIT)*

## License

Dựa trên [AWS Containers Retail Sample](https://github.com/aws-containers/retail-store-sample-app) — MIT-0 License.
