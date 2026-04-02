# Operations: Upgrades

Agent reference for upgrading Docker Engine and container images. Use this for planned upgrade operations.

---

## Pre-Upgrade Checklist

Before any upgrade:

1. **Back up named volumes** — see `skills/operations/backup-restore`
2. **Check known issues** for the target version — see reference/known-issues
3. **Check compatibility** — see reference/compatibility
4. **Record current state:**

```bash
docker version
docker info
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
docker volume ls
```

---

## Upgrade Docker Engine

### Ubuntu/Debian (apt)

```bash
# 1. Check available versions
apt-cache madison docker-ce

# 2. Stop running containers (or use live-restore)
docker compose down  # for each project

# 3. Upgrade
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. Verify
docker version
docker info

# 5. Restart containers
docker compose up -d  # for each project
```

### Pin to Specific Version

```bash
# Install specific version
VERSION=5:27.1.2-1~ubuntu.24.04~noble
sudo apt-get install -y docker-ce=$VERSION docker-ce-cli=$VERSION

# Hold version to prevent auto-upgrade
sudo apt-mark hold docker-ce docker-ce-cli
```

### RHEL/CentOS/Fedora (dnf)

```bash
# 1. Check available versions
dnf list docker-ce --showduplicates

# 2. Upgrade
sudo dnf update docker-ce docker-ce-cli containerd.io

# 3. Verify
docker version
```

---

## Live Restore (Upgrade Without Stopping Containers)

With `live-restore` enabled in daemon.json, containers continue running during daemon restart:

```json
{
  "live-restore": true
}
```

```bash
# Upgrade Docker (containers keep running)
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli

# Verify containers are still running
docker ps
```

> **Limitation:** live-restore does not work with Swarm mode.

---

## Upgrade Container Images

### Manual Pull and Recreate

```bash
# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d --remove-orphans

# Verify health
docker compose ps
```

### Pin Image Versions

```yaml
services:
  db:
    image: postgres:16.3          # Specific patch version
  redis:
    image: redis:7.2.5            # Specific patch version
  app:
    image: myapp:v1.2.3           # Semantic version
```

### Update Single Service

```bash
# Update just one service without touching others
docker compose pull app
docker compose up -d --no-deps app
```

---

## Zero-Downtime Redeployment

### With Health Checks

```yaml
services:
  app:
    image: myapp:2.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
```

### Compose Rolling Restart

```bash
# Scale up with new image, then scale down old
docker compose up -d --scale app=2 --no-recreate
# Wait for new container to be healthy
docker compose up -d --scale app=1
```

### Swarm Rolling Updates

```bash
docker service update \
  --image myapp:2.0 \
  --update-parallelism 1 \
  --update-delay 30s \
  --update-failure-action rollback \
  web
```

---

## Rollback

### Compose Rollback

```bash
# Revert to previous image
docker compose pull  # if image tag changed
docker compose up -d

# Or specify the old image explicitly
docker compose -f docker-compose.yml up -d
```

### Swarm Rollback

```bash
docker service rollback web
```

---

## Downgrade Docker Engine

```bash
# Install specific older version
VERSION=5:27.0.3-1~ubuntu.24.04~noble
sudo apt-get install -y --allow-downgrades docker-ce=$VERSION docker-ce-cli=$VERSION

# Verify
docker version
```

> **Warning:** Downgrading may cause issues if the newer version changed the storage format. Always back up `/var/lib/docker` before downgrading.

**Source:** https://docs.docker.com/engine/install/
