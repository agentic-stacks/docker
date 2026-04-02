# Dockerfile Reference

Agent reference for writing Dockerfiles. Use this when creating new images or improving existing Dockerfiles.

---

## Instruction Reference

### FROM — Base Image

```dockerfile
# Single-stage
FROM ubuntu:22.04

# Multi-stage with named stages
FROM golang:1.22 AS builder
FROM alpine:3.20 AS runtime

# From scratch (empty filesystem)
FROM scratch
```

Every Dockerfile must start with FROM (or ARG before FROM for build-time variables).

### RUN — Execute Commands

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (no shell processing)
RUN ["apt-get", "install", "-y", "curl"]

# Heredoc syntax (Docker Engine 23.0+, BuildKit required)
RUN <<EOF
apt-get update
apt-get install -y \
  curl \
  wget \
  jq
rm -rf /var/lib/apt/lists/*
EOF
```

**Best practice:** Combine related commands in a single RUN to reduce layers. Always clean up package manager caches in the same layer.

### COPY — Copy Files from Build Context

```dockerfile
# Copy single file
COPY app.py /app/

# Copy directory
COPY src/ /app/src/

# Copy with ownership
COPY --chown=appuser:appgroup app.py /app/

# Copy from another build stage
COPY --from=builder /app/binary /usr/local/bin/

# Copy with chmod (BuildKit required)
COPY --chmod=755 entrypoint.sh /usr/local/bin/
```

### ADD — Copy with Extras

```dockerfile
# Auto-extract tar archives
ADD archive.tar.gz /app/

# Download from URL (prefer curl in RUN instead)
ADD https://example.com/file.txt /app/
```

**Use COPY instead of ADD** unless you need tar auto-extraction. ADD's URL download does not use cache efficiently and cannot handle authentication.

### WORKDIR — Set Working Directory

```dockerfile
WORKDIR /app
# All subsequent RUN, CMD, ENTRYPOINT, COPY, ADD use /app as base
```

Always use WORKDIR instead of `RUN cd /app`. WORKDIR creates the directory if it doesn't exist.

### ENV — Environment Variables

```dockerfile
ENV NODE_ENV=production
ENV APP_HOME=/app \
    APP_PORT=8080
```

ENV variables persist in the running container. For build-only variables, use ARG instead.

### ARG — Build Arguments

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}

ARG BUILD_DATE
LABEL build-date=${BUILD_DATE}
```

ARG values do not persist in the running container. Use `docker build --build-arg NODE_VERSION=22` to override.

**ARG before FROM** is the only instruction allowed before FROM. It's scoped to the FROM instruction.

### ENTRYPOINT + CMD — Container Startup

```dockerfile
# ENTRYPOINT: the executable
ENTRYPOINT ["python3"]
# CMD: default arguments
CMD ["app.py"]
```

#### Interaction Matrix

| ENTRYPOINT | CMD | `docker run myimage` | `docker run myimage foo` |
|------------|-----|---------------------|-------------------------|
| Not set | `["python3", "app.py"]` | `python3 app.py` | `foo` |
| `["python3"]` | `["app.py"]` | `python3 app.py` | `python3 foo` |
| `["python3"]` | Not set | `python3` | `python3 foo` |
| `["/entrypoint.sh"]` | `["start"]` | `/entrypoint.sh start` | `/entrypoint.sh foo` |

**Rule:** Use exec form (`["binary", "arg"]`) not shell form (`binary arg`). Shell form wraps in `/bin/sh -c`, which prevents signal propagation (SIGTERM won't reach your app).

### EXPOSE — Document Ports

```dockerfile
EXPOSE 8080
EXPOSE 8080/tcp 9090/udp
```

EXPOSE does not publish the port. It documents which ports the container listens on. Use `-p` at runtime to publish.

### VOLUME — Declare Mount Points

```dockerfile
VOLUME /data
VOLUME ["/data", "/logs"]
```

VOLUME creates an anonymous volume at the specified path. **Prefer named volumes in Compose** over VOLUME in Dockerfile — VOLUME creates anonymous volumes that are hard to track and easy to lose.

### USER — Set Runtime User

```dockerfile
# Create user and switch to it
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
```

All subsequent RUN, CMD, ENTRYPOINT run as this user. **Always set USER in production images.**

### HEALTHCHECK — Container Health

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
  CMD curl -f http://localhost:8080/health || exit 1

# Disable health check inherited from base image
HEALTHCHECK NONE
```

| Option | Default | Description |
|--------|---------|-------------|
| `--interval` | 30s | Time between checks |
| `--timeout` | 30s | Max time for a single check |
| `--retries` | 3 | Consecutive failures before unhealthy |
| `--start-period` | 0s | Grace period at startup (failures don't count) |
| `--start-interval` | 5s | Check interval during start-period (Docker 25+) |

### LABEL — Metadata

```dockerfile
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/example/myapp"
```

### STOPSIGNAL — Stop Signal

```dockerfile
STOPSIGNAL SIGQUIT
```

Default is SIGTERM. Change if your application needs a different signal for graceful shutdown (e.g., nginx uses SIGQUIT).

### SHELL — Default Shell

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -fsSL https://example.com | tar xz
```

Changes the default shell for shell-form RUN instructions. Use this to enable bash features like `pipefail`.

---

## Multi-Stage Builds

Multi-stage builds let you use separate stages for building and running, keeping final images small.

### Basic Pattern

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/server

# Stage 2: Runtime
FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /usr/local/bin/server
USER nobody:nobody
EXPOSE 8080
ENTRYPOINT ["server"]
```

Result: final image contains only the compiled binary + ca-certificates, not the entire Go toolchain.

### Language-Specific Examples

#### Python

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
RUN groupadd -r app && useradd -r -g app app
USER app
EXPOSE 8000
CMD ["gunicorn", "app:create_app()", "--bind", "0.0.0.0:8000"]
```

#### Node.js

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
RUN npm run build

FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json .
RUN groupadd -r app && useradd -r -g app app
USER app
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

#### Rust

```dockerfile
FROM rust:1.78 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs && cargo build --release && rm -rf src
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapp /usr/local/bin/
USER nobody:nobody
ENTRYPOINT ["myapp"]
```

#### Java (Maven)

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/target/myapp-*.jar app.jar
RUN groupadd -r app && useradd -r -g app app
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## .dockerignore

Exclude files from the build context. Same syntax as .gitignore.

```
# Version control
.git
.gitignore

# Dependencies (rebuilt in container)
node_modules
vendor
__pycache__
*.pyc
.venv

# Build artifacts
dist
build
target
*.jar
*.war

# IDE and editor
.vscode
.idea
*.swp
*.swo

# Docker files (not needed in build)
Dockerfile
docker-compose*.yml
.dockerignore

# Environment and secrets
.env
.env.*
*.pem
*.key

# Documentation
README.md
LICENSE
docs/

# Test files (not needed in production image)
tests/
test/
*.test.*
coverage/
```

**Always create .dockerignore** — without it, the entire directory (including .git, node_modules, etc.) is sent to the build daemon, making builds slow.

**Source:** https://docs.docker.com/reference/dockerfile/
