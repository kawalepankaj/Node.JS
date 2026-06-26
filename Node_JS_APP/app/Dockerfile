# ─────────────────────────────────────────────────────────────────────────────
# Stage 1 – Dependencies
#   Uses the full node image to install npm packages (including native addons).
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS deps

WORKDIR /app

# Copy only the manifest files first for better layer-cache utilisation
COPY package*.json ./

# Install production + dev dependencies (dev needed for test stage)
RUN npm ci --include=dev


# ─────────────────────────────────────────────────────────────────────────────
# Stage 2 – Test
#   Run linting and unit tests in CI so a failing test breaks the build.
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS test

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm test


# ─────────────────────────────────────────────────────────────────────────────
# Stage 3 – Production
#   Minimal image: only production deps + app source, no dev tooling.
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS production

# Security best-practice: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Re-install prod-only dependencies
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy application source
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 3000

# Metadata labels (useful for container registries & auditing)
LABEL maintainer="devops-team@example.com" \
      org.opencontainers.image.title="devops-poc-app" \
      org.opencontainers.image.description="DevOps POC Node.js Application" \
      org.opencontainers.image.version="1.0.0"

# Health check so Docker/K8s knows when the container is ready
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "app.js"]
