# ============================================================================
# CordysCRM Frontend Dockerfile - Multi-stage Production Build
# ============================================================================
# Stage 1: Build Vue 3 frontend with pnpm
# Stage 2: Serve with Nginx (production-grade static file server)
# ============================================================================

# ---------------------------------------------------------------------------
# Stage 1: Build
# ---------------------------------------------------------------------------
FROM node:22-slim AS builder

WORKDIR /build

# Install pnpm globally
RUN npm install -g pnpm@10.4.1

# Copy package files first for better layer caching
COPY frontend/pnpm-workspace.yaml frontend/pnpm-lock.yaml* ./
COPY frontend/packages/lib-shared/package.json ./packages/lib-shared/
COPY frontend/packages/web/package.json ./packages/web/
COPY frontend/packages/mobile/package.json ./packages/mobile/
COPY frontend/package.json ./

# Install dependencies (resolve fresh to avoid stale workspace references)
RUN rm -f pnpm-lock.yaml && pnpm install --no-frozen-lockfile

# Copy source code
COPY frontend/ ./

# Build all packages (lib-shared → web + mobile)
RUN pnpm run build

# ---------------------------------------------------------------------------
# Stage 2: Production Nginx
# ---------------------------------------------------------------------------
FROM nginx:1.27-alpine AS production

LABEL maintainer="CordysCRM Docker <support@cordyscrm.com>"
LABEL description="CordysCRM Frontend - Nginx Production Image"

# Install curl for health checks
RUN apk add --no-cache curl

# Remove default nginx config
RUN rm -f /etc/nginx/conf.d/default.conf

# Copy custom nginx configuration
COPY docker/frontend/nginx.conf /etc/nginx/nginx.conf
COPY docker/frontend/default.conf /etc/nginx/conf.d/default.conf

# Copy built static assets
# Web (desktop) → root
COPY --from=builder /build/packages/web/dist/ /usr/share/nginx/html/
# Mobile → /mobile
COPY --from=builder /build/packages/mobile/dist/ /usr/share/nginx/html/mobile/

# Set correct permissions
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

# Create cache directories
RUN mkdir -p /var/cache/nginx/client_temp /var/cache/nginx/proxy_temp && \
    chown -R nginx:nginx /var/cache/nginx

EXPOSE 80

# Health check: verify nginx is serving and responding
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
