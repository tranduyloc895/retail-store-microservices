# ============================================================
# Stage 1: Build
# ============================================================
FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS build-env

# Chỉ cài gói cần thiết, bỏ qua weak deps cho image gọn nhất
RUN dnf --setopt=install_weak_deps=False install -q -y \
    git \
    golang \
    && dnf clean all

WORKDIR /appsrc

ENV GOPROXY=https://goproxy.io,direct

# Cache dependencies layer: chỉ invalidate khi go.mod/go.sum đổi
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build binary tĩnh, strip debug info để image nhỏ hơn
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o main main.go

# ============================================================
# Stage 2: Runtime (minimal image)
# ============================================================
FROM public.ecr.aws/amazonlinux/amazonlinux:2023

# curl-minimal đã có sẵn; chỉ cần shadow-utils để tạo user
RUN dnf --setopt=install_weak_deps=False install -q -y \
    shadow-utils \
    && dnf clean all

# Tạo non-root user với UID 10001 (tránh xung đột UID 1000 mặc định của 1 số base image)
RUN groupadd --gid 10001 appuser \
    && useradd --uid 10001 --gid 10001 --home /app --create-home appuser

WORKDIR /app

COPY --chown=appuser:appuser --from=build-env /appsrc/main /app/

ENV GIN_MODE=release

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["/app/main"]
