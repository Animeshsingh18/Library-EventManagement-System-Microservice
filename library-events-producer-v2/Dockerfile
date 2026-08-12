# ─────────────────────────────────────────────────────────
# Stage 1: Builder
# Extract the JAR layers for optimized caching in runtime image
# ─────────────────────────────────────────────────────────
FROM eclipse-temurin:25-jre AS builder

WORKDIR /builder

ARG JAR_FILE=build/libs/*-SNAPSHOT.jar

COPY ${JAR_FILE} app.jar

# Extract Spring Boot layers for better Docker caching
RUN java -Djarmode=layertools -jar app.jar extract

# ─────────────────────────────────────────────────────────
# Stage 2: Runtime
# Minimal image with only what's needed to run the app
# ─────────────────────────────────────────────────────────
FROM eclipse-temurin:25-jre

WORKDIR /app

# Copy layers from builder in the correct order
# This maximizes layer caching when dependencies don't change
COPY --from=builder /builder/dependencies/ ./
COPY --from=builder /builder/spring-boot-loader/ ./
COPY --from=builder /builder/snapshot-dependencies/ ./
COPY --from=builder /builder/application/ ./

# Application port
EXPOSE 8080

# Health check endpoint for orchestration tools (Docker Compose, K8s)
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Start the application
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]

