# Registry Management

Agent reference for pushing, pulling, and managing container images across registries. Use this when setting up image distribution workflows.

---

## Image Naming Convention

```
[registry/][namespace/]repository[:tag|@digest]
```

| Example | Registry | Namespace | Repository | Tag |
|---------|----------|-----------|------------|-----|
| `nginx:1.27` | Docker Hub (implicit) | library (implicit) | nginx | 1.27 |
| `myorg/myapp:1.0` | Docker Hub (implicit) | myorg | myapp | 1.0 |
| `ghcr.io/myorg/myapp:1.0` | ghcr.io | myorg | myapp | 1.0 |
| `123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0` | ECR | (account) | myapp | 1.0 |

---

## Push/Pull Workflows

```bash
# Tag a local image for a registry
docker tag myapp:1.0 ghcr.io/myorg/myapp:1.0

# Push to registry
docker push ghcr.io/myorg/myapp:1.0

# Pull from registry
docker pull ghcr.io/myorg/myapp:1.0

# Pull by digest (immutable)
docker pull ghcr.io/myorg/myapp@sha256:abc123...
```

### Tagging Strategy

| Strategy | Format | Example | Use Case |
|----------|--------|---------|----------|
| Semantic version | `vMAJOR.MINOR.PATCH` | `myapp:v1.2.3` | Releases with clear versioning |
| Git SHA | `sha-<short-hash>` | `myapp:sha-a1b2c3d` | CI builds, exact commit traceability |
| Date-based | `YYYYMMDD` | `myapp:20260329` | Nightly or scheduled builds |
| Branch + SHA | `branch-sha` | `myapp:main-a1b2c3d` | CI builds with branch context |
| Latest | `latest` | `myapp:latest` | **Never in production** — mutable, no traceability |

**Best practice:** Push multiple tags per build:
```bash
docker tag myapp:build ghcr.io/myorg/myapp:v1.2.3
docker tag myapp:build ghcr.io/myorg/myapp:sha-a1b2c3d
docker push ghcr.io/myorg/myapp:v1.2.3
docker push ghcr.io/myorg/myapp:sha-a1b2c3d
```

---

## Private Registry (registry:2)

Run a self-hosted Docker registry:

### Basic (No Auth, No TLS — Dev Only)

```bash
docker run -d -p 5000:5000 --name registry --restart unless-stopped \
  -v registry-data:/var/lib/registry \
  registry:2

# Push to it
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
```

### With TLS and Basic Auth

```bash
# 1. Generate htpasswd file
mkdir -p /opt/registry/auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn admin secretpassword > /opt/registry/auth/htpasswd

# 2. Place TLS certs
mkdir -p /opt/registry/certs
# Copy your cert.pem and key.pem to /opt/registry/certs/

# 3. Run registry with TLS + auth
docker run -d -p 443:5000 --name registry --restart unless-stopped \
  -v registry-data:/var/lib/registry \
  -v /opt/registry/certs:/certs \
  -v /opt/registry/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cert.pem \
  -e REGISTRY_HTTP_TLS_KEY=/certs/key.pem \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2

# 4. Login
docker login myregistry.example.com
```

### Garbage Collection

Unreferenced blobs accumulate over time. Clean up:

```bash
# Dry run — see what would be deleted
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run

# Actually delete
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

---

## Cloud Registries

### GitHub Container Registry (GHCR)

```bash
# Login with personal access token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push
docker tag myapp:1.0 ghcr.io/myorg/myapp:1.0
docker push ghcr.io/myorg/myapp:1.0
```

### AWS Elastic Container Registry (ECR)

```bash
# Login (requires AWS CLI configured)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository (first time only)
aws ecr create-repository --repository-name myapp

# Push
docker tag myapp:1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

### Google Container Registry / Artifact Registry (GCR)

```bash
# Login
gcloud auth configure-docker us-docker.pkg.dev

# Push (Artifact Registry — preferred over gcr.io)
docker tag myapp:1.0 us-docker.pkg.dev/myproject/myrepo/myapp:1.0
docker push us-docker.pkg.dev/myproject/myrepo/myapp:1.0
```

### Azure Container Registry (ACR)

```bash
# Login
az acr login --name myregistry

# Push
docker tag myapp:1.0 myregistry.azurecr.io/myapp:1.0
docker push myregistry.azurecr.io/myapp:1.0
```

---

## Image Signing with Cosign

Verify image integrity and provenance:

```bash
# Generate key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key ghcr.io/myorg/myapp:1.0

# Verify signature
cosign verify --key cosign.pub ghcr.io/myorg/myapp:1.0
```

### Keyless Signing (with OIDC)

```bash
# Sign using identity token (e.g., GitHub Actions OIDC)
cosign sign ghcr.io/myorg/myapp:1.0

# Verify
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'https://github.com/myorg/myapp' \
  ghcr.io/myorg/myapp:1.0
```

---

## Docker Content Trust (DCT)

Built-in image signing using Notary:

```bash
# Enable content trust
export DOCKER_CONTENT_TRUST=1

# Push (will sign automatically)
docker push myregistry.example.com/myapp:1.0

# Pull (rejects unsigned images)
docker pull myregistry.example.com/myapp:1.0
```

> **Note:** DCT is being superseded by cosign/Sigstore in most modern workflows. Prefer cosign for new deployments.

**Source:** https://docs.docker.com/docker-hub/
