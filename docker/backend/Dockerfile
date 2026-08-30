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

# Copy full project context (Maven parent POM requires frontend module to exist)
# .dockerignore excludes node_modules, dist, .idea etc.
COPY . .

# Build backend only (exclude frontend module via -pl)
RUN chmod +x mvnw && \
    ./mvnw clean package -DskipTests -pl '!frontend' -Dmaven.javadoc.skip=true && \
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
RUN apk add --no-cache curl tzdata bash netcat-openbsd && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

# Create non-root user for security
RUN addgroup -g 1001 -S cordys && \
    adduser -u 1001 -S cordys -G cordys

# Create application directories (all writable by cordys user)
RUN mkdir -p /app/lib \
             /app/META-INF \
             /opt/cordys/conf \
             /opt/cordys/conf/mysql \
             /opt/cordys/conf/redis \
             /opt/cordys/data/files \
             /opt/cordys/data/mysql \
             /opt/cordys/data/redis \
             /opt/cordys/logs/cordys-crm \
             /opt/cordys/logs/mcp-server \
             /run/cordys && \
    chown -R cordys:cordys /app /opt/cordys /run/cordys

# Copy built artifacts from builder
COPY --from=builder --chown=cordys:cordys /build/backend/app/target/dependency/BOOT-INF/lib /app/lib
COPY --from=builder --chown=cordys:cordys /build/backend/app/target/dependency/META-INF /app/META-INF
COPY --from=builder --chown=cordys:cordys /build/backend/app/target/dependency/BOOT-INF/classes /app

# Copy application configuration templates
COPY --chown=cordys:cordys installer/conf/cordys-crm.properties /installer/conf/cordys-crm.properties
COPY --chown=cordys:cordys installer/conf/mysql/my.cnf /installer/conf/mysql/my.cnf
COPY --chown=cordys:cordys installer/conf/redis/redis.conf /installer/conf/redis/redis.conf

# Copy shell scripts and make executable
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

WORKDIR /app

EXPOSE 8081

# Direct Java startup with MySQL/Redis readiness check
ENTRYPOINT ["sh", "-c", "\
  echo '[INFO] Initializing directories...' && \
  bash /shells/init-directories.sh && \
  echo '[INFO] Patching config with Docker environment...' && \
  CONF=/opt/cordys/conf/cordys-crm.properties && \
  if [ -f \"$CONF\" ]; then \
    sed -i 's|mysql.embedded.enabled=true|mysql.embedded.enabled=false|g' \"$CONF\" && \
    sed -i 's|redis.embedded.enabled=true|redis.embedded.enabled=false|g' \"$CONF\" && \
    sed -i 's|mcp.embedded.enabled=true|mcp.embedded.enabled=false|g' \"$CONF\" && \
    sed -i 's|cockpit.embedded.enabled=true|cockpit.embedded.enabled=false|g' \"$CONF\" && \
    sed -i 's|jdbc:mysql://127.0.0.1:3306|jdbc:mysql://${MYSQL_HOST:-cordys-mysql}:${MYSQL_PORT:-3306}|g' \"$CONF\" && \
    sed -i \"s|spring.datasource.password=.*|spring.datasource.password=${MYSQL_PASSWORD:-CordysCRM@mysql}|g\" \"$CONF\" && \
    sed -i \"s|spring.data.redis.host=.*|spring.data.redis.host=${REDIS_HOST:-cordys-redis}|g\" \"$CONF\" && \
    sed -i \"s|spring.data.redis.password=.*|spring.data.redis.password=${REDIS_PASSWORD:-CordysCRM@redis}|g\" \"$CONF\" && \
    echo '[INFO] Config patched successfully.'; \
  fi && \
  echo '[INFO] Waiting for MySQL at ${MYSQL_HOST:-cordys-mysql}:${MYSQL_PORT:-3306}...' && \
  until nc -z ${MYSQL_HOST:-cordys-mysql} ${MYSQL_PORT:-3306} 2>/dev/null; do \
    echo '[INFO] MySQL not ready, retrying in 3s...'; \
    sleep 3; \
  done && \
  echo '[INFO] MySQL is reachable. Waiting 5s for full initialization...' && \
  sleep 5 && \
  echo '[INFO] Waiting for Redis at ${REDIS_HOST:-cordys-redis}:${REDIS_PORT:-6379}...' && \
  until nc -z ${REDIS_HOST:-cordys-redis} ${REDIS_PORT:-6379} 2>/dev/null; do \
    echo '[INFO] Redis not ready, retrying in 3s...'; \
    sleep 3; \
  done && \
  echo '[INFO] Redis is reachable.' && \
  echo '[INFO] Starting CordysCRM Backend (version: ${CRM_VERSION})...' && \
  exec java \
    -cp ${JAVA_CLASSPATH} \
    ${JAVA_OPTIONS} \
    ${JAVA_MAIN_CLASS} \
    \"$@\"", "--"]
