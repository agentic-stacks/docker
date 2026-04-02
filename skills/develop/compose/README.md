# Docker Compose File Reference

Agent reference for authoring Docker Compose files. Use this when defining multi-container applications.

---

## Compose File Structure

```yaml
# All Compose files use the Compose Specification (no version: field needed)
name: my-project  # optional — defaults to directory name

services:
  app:
    image: myapp:1.0
    # ... service config

  db:
    image: postgres:16
    # ... service config

networks:
  frontend:
  backend:

volumes:
  db-data:

configs:
  app-config:
    file: ./configs/app.conf

secrets:
  db-password:
    file: ./secrets/db-password.txt
```

---

## Services

### Image or Build

```yaml
services:
  # Use a pre-built image
  nginx:
    image: nginx:1.27

  # Build from Dockerfile
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
      target: runtime            # multi-stage build target
      cache_from:
        - myapp:cache
    image: myapp:1.0             # tag the built image
```

### Ports

```yaml
services:
  app:
    ports:
      - "8080:80"                # host:container
      - "127.0.0.1:9090:9090"   # bind to specific interface
      - "8443:443/udp"          # UDP
      - target: 80              # long syntax
        published: 8080
        protocol: tcp
        mode: host              # or ingress (Swarm)
```

### Environment Variables

```yaml
services:
  app:
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
      # Reference host environment variable
      API_KEY: ${API_KEY}
      # Default value if unset
      LOG_LEVEL: ${LOG_LEVEL:-info}

    # Or from a file
    env_file:
      - .env
      - .env.local
```

### Volumes

```yaml
services:
  db:
    volumes:
      # Named volume
      - db-data:/var/lib/postgresql/data
      # Bind mount
      - ./configs/pg.conf:/etc/postgresql/postgresql.conf:ro
      # tmpfs
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100m

volumes:
  db-data:
    driver: local
```

### Networks

```yaml
services:
  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true   # no external access
```

Services on the same network can reach each other by service name (DNS).

### Restart Policies

```yaml
services:
  app:
    restart: unless-stopped
```

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart, including on daemon start |
| `unless-stopped` | Restart unless explicitly stopped by operator |
| `on-failure[:max-retries]` | Restart only on non-zero exit, optional limit |

### Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
```

### Dependency Ordering

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
```

`condition: service_healthy` waits for the dependency's health check to pass before starting the dependent service.

### Health Checks

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
      start_interval: 5s
```

---

## Profiles

Selectively start services based on profiles:

```yaml
services:
  app:
    image: myapp:1.0
    # No profile — always starts

  debug:
    image: myapp:1.0
    command: ["--debug"]
    profiles:
      - debug

  test:
    image: myapp:1.0
    command: ["npm", "test"]
    profiles:
      - test
```

```bash
# Start only default services
docker compose up -d

# Start default + debug services
docker compose --profile debug up -d

# Start default + test services
docker compose --profile test up -d
```

---

## Extends

Reuse service configuration:

```yaml
# common.yml
services:
  base-app:
    image: myapp:1.0
    environment:
      NODE_ENV: production
    restart: unless-stopped
```

```yaml
# docker-compose.yml
services:
  web:
    extends:
      file: common.yml
      service: base-app
    ports:
      - "8080:80"

  worker:
    extends:
      file: common.yml
      service: base-app
    command: ["worker"]
```

---

## Variable Interpolation

### .env File

```
# .env (loaded automatically)
POSTGRES_VERSION=16
APP_PORT=8080
```

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION}
  app:
    ports:
      - "${APP_PORT}:80"
```

### Substitution Syntax

| Syntax | Behavior |
|--------|----------|
| `${VAR}` | Value of VAR, error if unset |
| `${VAR:-default}` | Value of VAR, or "default" if unset or empty |
| `${VAR-default}` | Value of VAR, or "default" if unset (empty is valid) |
| `${VAR:?error msg}` | Value of VAR, or error with message if unset or empty |
| `${VAR?error msg}` | Value of VAR, or error with message if unset |

### Validate Interpolation

```bash
# Show the resolved Compose file with all variables expanded
docker compose config
```

---

## Override Files

Compose merges multiple files in order:

```bash
# Default merge order (automatic):
# docker-compose.yml + docker-compose.override.yml

# Explicit files:
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Common pattern:**
- `docker-compose.yml` — base service definitions
- `docker-compose.override.yml` — local dev overrides (bind mounts, debug ports) — gitignored
- `docker-compose.prod.yml` — production overrides (resource limits, logging, replicas)

### Merge Rules

| Field | Merge Behavior |
|-------|----------------|
| `image` | Replaced by override |
| `command` | Replaced by override |
| `environment` | Merged (override wins on conflict) |
| `ports` | Appended |
| `volumes` | Appended |
| `labels` | Merged |
| `networks` | Merged |

---

## Compose Watch (Dev Hot-Reload)

```yaml
services:
  app:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./package.json
```

| Action | Behavior |
|--------|----------|
| `sync` | Syncs changed files into the running container (no rebuild) |
| `rebuild` | Triggers a full image rebuild and container restart |
| `sync+restart` | Syncs files and restarts the container |

```bash
docker compose watch
```

---

## Configs and Secrets (Compose)

### Configs (Non-Sensitive)

```yaml
services:
  nginx:
    image: nginx:1.27
    configs:
      - source: nginx-conf
        target: /etc/nginx/nginx.conf
        mode: 0444

configs:
  nginx-conf:
    file: ./configs/nginx.conf
```

### Secrets (Sensitive)

```yaml
services:
  db:
    image: postgres:16
    secrets:
      - db-password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password

secrets:
  db-password:
    file: ./secrets/db-password.txt
```

Secrets are mounted as files under `/run/secrets/` in the container.

---

## Essential Commands

| Task | Command |
|------|---------|
| Start all services | `docker compose up -d` |
| Start and wait for healthy | `docker compose up -d --wait` |
| Stop all services | `docker compose down` |
| Stop and remove volumes | `docker compose down -v` |
| Rebuild and restart | `docker compose up -d --build` |
| View logs | `docker compose logs -f app` |
| Run one-off command | `docker compose exec app sh` |
| Scale a service | `docker compose up -d --scale worker=3` |
| Validate Compose file | `docker compose config` |
| Pull latest images | `docker compose pull` |
| List running services | `docker compose ps` |

**Source:** https://docs.docker.com/compose/compose-file/
