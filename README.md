# Retail Store Microservices

Source code microservices cho dự án **DevSecOps E-commerce** (NT114 - UIT). Ứng dụng mô phỏng một cửa hàng bán lẻ trực tuyến với kiến trúc microservices, phục vụ làm workload cho pipeline CI/CD và triển khai lên Kubernetes (EKS).

> Source code gốc: [aws-containers/retail-store-sample-app](https://github.com/aws-containers/retail-store-sample-app) (MIT-0 License). Đã được tinh gọn, chỉ giữ lại phần microservice code.

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

## Microservices

| Service | Ngôn ngữ | Framework | Port | Database | Mô tả |
|---------|----------|-----------|------|----------|-------|
| **UI** | Java 21 | Spring Boot 3.5 + Thymeleaf | 8080 | - | Giao diện web, gọi tới các service khác |
| **Catalog** | Go | Gin | 8080 | MariaDB | API danh mục sản phẩm |
| **Cart** | Java 21 | Spring Boot 3.5 | 8080 | DynamoDB (local) | API giỏ hàng |
| **Orders** | Java 21 | Spring Boot 3.5 | 8080 | PostgreSQL + RabbitMQ | API quản lý đơn hàng |
| **Checkout** | TypeScript | NestJS 11 | 8080 | Redis | API xử lý thanh toán |

## Cấu trúc thư mục

```
retail-store-microservices/
├── src/
│   ├── ui/                    # Web UI (Java/Spring Boot)
│   │   ├── Dockerfile
│   │   ├── Jenkinsfile        # Pipeline CI/CD cho Jenkins
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

## Liên kết

| Repo | Mô tả |
|------|-------|
| [infrastructure](https://github.com/<your-org>/infrastructure) | Terraform + Ansible: VPC, EKS, Jenkins, ArgoCD |
| [gitops-config](https://github.com/<your-org>/gitops-config) | Helm values cho ArgoCD deployment |

> *Đồ án môn NT114 - Đại học Công nghệ Thông tin (UIT)*

## License

Dựa trên [AWS Containers Retail Sample](https://github.com/aws-containers/retail-store-sample-app) — MIT-0 License.
