# ============================================================
# Stage 1: Build
# ============================================================
FROM eclipse-temurin:21-jdk AS build

WORKDIR /workspace

# Copy Maven wrapper & pom.xml first (cache dependencies layer)
COPY .mvn .mvn
COPY mvnw .
COPY pom.xml .

# Download dependencies (cached unless pom.xml changes)
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B -q

# Copy source code & build
COPY ./src ./src
RUN ./mvnw -DskipTests package -q \
    && mv target/ui-0.0.1-SNAPSHOT.jar /app.jar

# ============================================================
# Stage 2: Runtime (minimal JRE image)
# ============================================================
FROM eclipse-temurin:21-jre

# Create non-root user for security (use -f to ignore if GID/UID already exists)
RUN groupadd --gid 1000 -f appuser \
    && (useradd --uid 1000 --gid 1000 --create-home appuser 2>/dev/null || true)

ENV SPRING_PROFILES_ACTIVE=prod

WORKDIR /app

# Copy artifact from build stage
COPY --from=build --chown=appuser:appuser /app.jar .

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health/readiness || exit 1

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-Djava.security.egd=file:/dev/urandom", "-jar", "/app/app.jar"]
