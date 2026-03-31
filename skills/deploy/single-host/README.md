# Deploy: Single Host

Agent reference for running containers on a single Docker host. Use this for development environments, small deployments, or single-server production.

---

## docker run Essential Flags

```bash
docker run -d \
  --name myapp \
  --hostname myapp \
  -p 8080:80 \
  -v app-data:/data \
  -e NODE_ENV=production \
  --restart unless-stopped \
  --memory 512m \
  --cpus 1.0 \
  --read-only \
  --user 1000:1000 \
  --health-cmd "curl -f http://localhost:80/health || exit 1" \
  --health-interval 30s \
  myapp:1.0
```

### Flag Reference

| Flag | Purpose | Example |
|------|---------|---------|
| `-d` | Detached mode (background) | `-d` |
| `--name` | Container name | `--name myapp` |
| `-p` | Port mapping (host:container) | `-p 8080:80` |
| `-v` | Volume mount | `-v data:/app/data` |
| `-e` | Environment variable | `-e KEY=value` |
| `--env-file` | Load env from file | `--env-file .env` |
| `--restart` | Restart policy | `--restart unless-stopped` |
| `--memory` | Memory limit | `--memory 512m` |
| `--cpus` | CPU limit | `--cpus 1.5` |
| `--read-only` | Read-only root filesystem | `--read-only` |
| `--user` | Run as UID:GID | `--user 1000:1000` |
| `--network` | Connect to network | `--network mynet` |
| `--hostname` | Set container hostname | `--hostname myapp` |
| `--tmpfs` | Mount tmpfs | `--tmpfs /tmp:size=100m` |
| `--init` | Use tini as PID 1 | `--init` |
| `--rm` | Remove container on exit | `--rm` |
| `-it` | Interactive + TTY (for shell) | `-it` |

---

## Restart Policies

| Policy | Behavior | When to Use |
|--------|----------|-------------|
| `no` | Never restart (default) | One-off tasks, debugging |
| `always` | Always restart, including on daemon start | Critical services that must always run |
| `unless-stopped` | Like always, but not if explicitly stopped | **Recommended default for services** |
| `on-failure[:N]` | Restart on non-zero exit, optional max count | Services where crash loops should be limited |

```bash
# Update restart policy on existing container
docker update --restart unless-stopped myapp
```

---

## Docker Compose Lifecycle

### Start Services

```bash
# Start all services in background
docker compose up -d

# Start and wait for health checks to pass
docker compose up -d --wait

# Start specific services
docker compose up -d app db

# Start with rebuild
docker compose up -d --build

# Start with fresh containers (remove orphans)
docker compose up -d --force-recreate --remove-orphans
```

### Stop Services

```bash
# Stop containers (preserve volumes)
docker compose down

# Stop and remove named volumes
docker compose down -v

# Stop and remove images
docker compose down --rmi all
```

### Restart and Recreate

```bash
# Restart a single service
docker compose restart app

# Recreate a single service (applies config changes)
docker compose up -d --no-deps app

# Pull new images and recreate
docker compose pull && docker compose up -d --remove-orphans
```

### View Status

```bash
# List running services
docker compose ps

# View logs
docker compose logs -f app

# View logs with timestamps
docker compose logs -f -t app

# View resource usage
docker compose top
```

---

## Resource Constraints

### Memory

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M        # Hard limit — OOM killed if exceeded
        reservations:
          memory: 256M        # Guaranteed minimum
```

CLI equivalent:
```bash
docker run --memory 512m --memory-reservation 256m myapp:1.0
```

### CPU

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "1.5"         # Max 1.5 cores
        reservations:
          cpus: "0.25"        # Guaranteed 0.25 cores
```

### PIDs

```yaml
services:
  app:
    pids_limit: 100            # Prevent fork bombs
```

---

## Logging Configuration

### Per-Service in Compose

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
        compress: "true"
```

### View Logs

```bash
# Follow logs
docker logs -f myapp

# Last 100 lines
docker logs --tail 100 myapp

# Since a timestamp
docker logs --since 2026-03-29T10:00:00 myapp

# Filter logs (pipe to grep)
docker logs myapp 2>&1 | grep ERROR
```

---

## Container Management

### Execute Commands in Running Container

```bash
# Interactive shell
docker exec -it myapp /bin/sh

# Run a command
docker exec myapp cat /etc/hosts

# Run as specific user
docker exec -u root myapp whoami
```

### Copy Files

```bash
# Copy from container to host
docker cp myapp:/app/data/export.csv ./export.csv

# Copy from host to container
docker cp ./config.json myapp:/app/config.json
```

### Inspect Container

```bash
# Full inspection
docker inspect myapp

# Specific fields with Go templates
docker inspect --format '{{.State.Status}}' myapp
docker inspect --format '{{.NetworkSettings.IPAddress}}' myapp
docker inspect --format '{{json .Config.Env}}' myapp | jq .
docker inspect --format '{{.State.Health.Status}}' myapp
```

---

## Cleanup

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove dangling images only
docker image prune -f

# Remove unused volumes (careful — data loss!)
docker volume prune

# Remove everything unused (containers, images, networks, build cache)
docker system prune

# Nuclear option — remove ALL unused including named volumes
docker system prune -a --volumes
```

> **Warning:** `docker system prune -a --volumes` removes all unused named volumes. Back up data first.

**Source:** https://docs.docker.com/reference/cli/docker/container/run/
