# ============================================================================
# CordysCRM Backend Dockerfile - Multi-stage Production Build
# ============================================================================
# Stage 1: Build Java application with Maven
# Stage 2: Production runtime with JRE 21 (minimal footprint)
# ============================================================================

# ---------------------------------------------------------------------------
# Stage 1: Build
# ---------------------------------------------------------------------------
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

# Copy Maven wrapper and POM first for dependency caching
COPY mvnw ./
COPY .mvn .mvn
COPY pom.xml ./
COPY backend/pom.xml backend/pom.xml
COPY backend/framework/pom.xml backend/framework/
COPY backend/crm/pom.xml backend/crm/
COPY backend/app/pom.xml backend/app/

# Download dependencies (cached layer)
RUN ./mvnw dependency:resolve -pl backend/app -am -DskipTests -Dmaven.javadoc.skip=true || true

# Copy source code
COPY backend backend

# Build the application
RUN ./mvnw clean package -DskipTests -pl '!frontend' && \
    mkdir -p backend/app/target/dependency && \
    cd backend/app/target/dependency && \
    jar -xf ../*.jar

# ---------------------------------------------------------------------------
# Stage 2: Production Runtime
# ---------------------------------------------------------------------------
FROM eclipse-temurin:21-jre-alpine AS production

LABEL maintainer="CordysCRM Docker <support@cordyscrm.com>"
LABEL description="CordysCRM Backend - Spring Boot Production Image"

# Install curl for health checks and tzdata for timezone
RUN apk add --no-cache curl tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

# Create non-root user for security
RUN addgroup -g 1001 -S cordys && \
    adduser -u 1001 -S cordys -G cordys

# Create application directories
RUN mkdir -p /app /opt/cordys/conf /opt/cordys/data/files /opt/cordys/logs/cordys-crm && \
    chown -R cordys:cordys /app /opt/cordys

# Copy built artifacts from builder
COPY --from=builder /build/backend/app/target/dependency/BOOT-INF/lib /app/lib
COPY --from=builder /build/backend/app/target/dependency/META-INF /app/META-INF
COPY --from=builder /build/backend/app/target/dependency/BOOT-INF/classes /app

# Copy application configuration templates
COPY installer/conf/cordys-crm.properties /installer/conf/cordys-crm.properties
COPY installer/conf/mysql/my.cnf /installer/conf/mysql/my.cnf
COPY installer/conf/redis/redis.conf /installer/conf/redis/redis.conf

# Copy shell scripts
COPY installer/shells /shells
RUN chmod +x /shells/*.sh

# Set environment variables
ENV JAVA_CLASSPATH=/app:/app/lib/* \
    JAVA_MAIN_CLASS=cn.cordys.Application \
    JAVA_OPTIONS="-Dfile.encoding=utf-8 \
    -Djava.awt.headless=true \
    -XX:+UseG1GC \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:+UseStringDeduplication \
    -XX:+ParallelRefProcEnabled \
    --add-opens java.base/jdk.internal.loader=ALL-UNNAMED \
    --add-opens java.base/java.util=ALL-UNNAMED" \
    SPRING_PROFILES_ACTIVE=docker \
    CRM_VERSION=latest

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=5 \
    CMD curl -f http://localhost:8081/front/account/info || exit 1

# Switch to non-root user
USER cordys

EXPOSE 8081

# Default entrypoint
ENTRYPOINT ["sh", "/shells/start-cordys.sh"]
