# Build Optimization

Agent reference for optimizing Docker image builds — speed, size, and caching. Use this when builds are slow or images are too large.

---

## Layer Ordering Strategy

Docker caches layers. If a layer's input hasn't changed, Docker reuses the cached version. **Order instructions from least-frequently-changed to most-frequently-changed.**

### Bad: Cache busted on every code change

```dockerfile
COPY . /app/
RUN pip install -r requirements.txt   # Reinstalls every time code changes
```

### Good: Dependencies cached separately from code

```dockerfile
COPY requirements.txt /app/
RUN pip install -r requirements.txt   # Only reinstalls when requirements change
COPY . /app/                          # Code changes don't bust dependency cache
```

This pattern applies to all languages:
- **Python:** `COPY requirements.txt` before `COPY .`
- **Node.js:** `COPY package.json package-lock.json` before `COPY .`
- **Go:** `COPY go.mod go.sum` before `COPY .`
- **Rust:** `COPY Cargo.toml Cargo.lock` before `COPY .`
- **Java:** `COPY pom.xml` before `COPY .`

---

## BuildKit Features

BuildKit is Docker's modern build engine. Enabled by default in Docker 23.0+.

### Verify BuildKit is Active

```bash
docker buildx version
DOCKER_BUILDKIT=1 docker build .   # force-enable if needed
```

### Cache Mounts

Persist package manager caches across builds:

```dockerfile
# apt cache (Debian/Ubuntu)
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl

# pip cache (Python)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# npm cache (Node.js)
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Go module cache
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Maven/Gradle cache (Java)
RUN --mount=type=cache,target=/root/.m2 \
    mvn package -DskipTests
```

Cache mounts survive across builds but are **not included in the final image**. They dramatically speed up repeated builds.

### Build Secrets

Pass secrets to build without baking them into layers:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

The secret is available only during the RUN instruction and does not appear in any image layer.

### SSH Forwarding

Use host SSH keys during build (e.g., for private git repos):

```dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

```bash
docker build --ssh default .
```

---

## Image Size Reduction

### Base Image Comparison

| Base Image | Compressed Size | Has Shell | Has Package Manager | Use Case |
|-----------|-----------------|-----------|--------------------:|----------|
| `scratch` | 0 MB | No | No | Static binaries (Go, Rust) |
| `gcr.io/distroless/static` | ~2 MB | No | No | Static binaries + ca-certs |
| `gcr.io/distroless/base` | ~20 MB | No | No | Dynamically linked + glibc |
| `alpine:3.20` | ~3.5 MB | Yes (sh) | Yes (apk) | Small images that need shell |
| `ubuntu:22.04` | ~30 MB | Yes (bash) | Yes (apt) | Compatibility, debugging |
| `debian:bookworm-slim` | ~30 MB | Yes (bash) | Yes (apt) | Compatibility, debugging |

**Decision guide:**
- **Go/Rust (static binary):** `scratch` or `distroless/static`
- **Python/Node/Java:** `*-slim` variant (e.g., `python:3.12-slim`)
- **Need debugging shell in production:** `alpine`
- **glibc compatibility required:** `debian:*-slim` or `distroless/base`

### Minimize Layer Content

```dockerfile
# Bad: leaves apt cache in layer
RUN apt-get update && apt-get install -y curl

# Good: clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### Remove Unnecessary Files

```dockerfile
# Remove docs, man pages, locales
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/* /usr/share/doc /usr/share/man /usr/share/locale
```

---

## Multi-Platform Builds

Build images for multiple architectures from a single Dockerfile:

```bash
# Create a builder that supports multi-platform
docker buildx create --name multiplatform --driver docker-container --use

# Build for linux/amd64 and linux/arm64, push to registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myapp:1.0 \
  --push .

# Build for local platform only (load into local docker)
docker buildx build --load -t myapp:1.0 .
```

### Check Available Platforms

```bash
docker buildx ls
```

---

## Analyze Image Size

### docker scout (built-in)

```bash
# Quick overview
docker scout quickview myapp:1.0

# Detailed CVE scan
docker scout cves myapp:1.0
```

### dive (third-party)

```bash
# Install dive
docker pull wagoodman/dive

# Analyze image layers
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive myapp:1.0
```

dive shows each layer's content and wasted space (files added then deleted in later layers).

### Docker System Disk Usage

```bash
# Summary
docker system df

# Detailed breakdown
docker system df -v

# Clean build cache
docker builder prune

# Clean everything (careful!)
docker system prune -a --volumes
```

---

## .dockerignore Best Practices

```
# Always exclude version control
.git

# Exclude dependency directories (rebuilt in container)
node_modules
vendor
__pycache__
.venv
target

# Exclude Docker and CI files
Dockerfile*
docker-compose*
.dockerignore
.github
.gitlab-ci.yml

# Exclude IDE and OS files
.vscode
.idea
.DS_Store

# Exclude secrets and env files
.env
.env.*
*.pem
*.key
*.p12

# Exclude test and doc files
tests/
docs/
README.md
LICENSE
coverage/
```

Without .dockerignore, the entire build context is sent to the daemon. A `.git` directory alone can be hundreds of MB.

### Measure Build Context Size

```bash
# See what gets sent to the build daemon
docker build --no-cache --progress=plain . 2>&1 | head -5
# Look for "transferring context" line
```

---

## Build Cache Strategies

### Inline Cache (for CI/CD)

```bash
# Build with inline cache metadata
docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t myapp:cache .

# Push the cache image
docker push myapp:cache

# Use cached image in next build
docker build --cache-from myapp:cache -t myapp:1.0 .
```

### Registry Cache (BuildKit)

```bash
# Export cache to registry
docker buildx build \
  --cache-to type=registry,ref=myapp:buildcache \
  --cache-from type=registry,ref=myapp:buildcache \
  -t myapp:1.0 --push .
```

### Local Cache (BuildKit)

```bash
# Export cache to local directory
docker buildx build \
  --cache-to type=local,dest=/tmp/buildcache \
  --cache-from type=local,src=/tmp/buildcache \
  -t myapp:1.0 .
```

**Source:** https://docs.docker.com/build/buildkit/
