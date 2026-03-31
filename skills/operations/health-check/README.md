# Operations: Health Check

Procedures for verifying container and system health. Use before and after any operational change.

---

## Quick Health Check (30 seconds)

Run these commands in sequence. If all pass, the system is healthy.

```bash
# 1. Docker daemon is running
docker info > /dev/null 2>&1 && echo "OK: daemon running" || echo "FAIL: daemon not running"

# 2. All containers with restart policies are running
docker ps --filter "status=restarting" --format "{{.Names}}: RESTARTING"
docker ps --filter "status=exited" --filter "label=com.docker.compose.project" --format "{{.Names}}: EXITED ({{.Status}})"

# 3. No unhealthy containers
docker ps --filter "health=unhealthy" --format "{{.Names}}: UNHEALTHY"

# 4. Disk usage is acceptable
docker system df

# 5. No OOM kills in recent logs
journalctl -k --since "1 hour ago" | grep -i "oom\|out of memory" || echo "OK: no OOM kills"
```

---

## HEALTHCHECK in Dockerfiles

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
  CMD curl -f http://localhost:8080/health || exit 1
```

| Option | Default | Purpose |
|--------|---------|---------|
| `--interval` | 30s | Time between health checks |
| `--timeout` | 30s | Max time for a single check to complete |
| `--retries` | 3 | Consecutive failures before marking unhealthy |
| `--start-period` | 0s | Grace period — failures don't count during startup |
| `--start-interval` | 5s | Interval during start-period (Docker 25+) |

### Health Check Patterns by Application Type

| App Type | Health Check Command |
|----------|---------------------|
| HTTP API | `curl -f http://localhost:PORT/health \|\| exit 1` |
| TCP service | `nc -z localhost PORT \|\| exit 1` |
| PostgreSQL | `pg_isready -U postgres \|\| exit 1` |
| MySQL | `mysqladmin ping -h localhost \|\| exit 1` |
| Redis | `redis-cli ping \|\| exit 1` |
| MongoDB | `mongosh --eval "db.adminCommand('ping')" \|\| exit 1` |
| gRPC | `grpc_health_probe -addr=localhost:PORT \|\| exit 1` |

### Compose Health Check

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

## Inspect Health State

```bash
# Health status
docker inspect --format '{{.State.Health.Status}}' myapp
# Returns: starting, healthy, unhealthy, or none

# Last 5 health check results
docker inspect --format '{{json .State.Health}}' myapp | jq '.Log[-5:]'

# Watch health changes in real time
docker events --filter type=container --filter event=health_status
```

---

## docker events

Real-time event stream from the Docker daemon:

```bash
# All events
docker events

# Container events only
docker events --filter type=container

# Specific events
docker events --filter event=start --filter event=stop --filter event=die --filter event=oom

# Since a timestamp
docker events --since "2026-03-29T10:00:00"

# JSON format for parsing
docker events --format '{{json .}}' | jq .
```

Key events to monitor:

| Event | Meaning |
|-------|---------|
| `start` | Container started |
| `stop` | Container stopped (graceful) |
| `die` | Container exited (check exit code) |
| `kill` | Container received signal |
| `oom` | Container hit memory limit |
| `health_status` | Health check state changed |
| `restart` | Container restarted by policy |

---

## docker stats

Real-time resource usage:

```bash
# Live stats for all containers
docker stats

# Specific containers
docker stats myapp db redis

# One-shot (no streaming)
docker stats --no-stream

# Custom format
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

---

## Monitoring with Prometheus + cAdvisor + Grafana

### cAdvisor (Container Metrics)

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8081:8080"
    restart: unless-stopped
```

### Prometheus Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']  # Docker daemon metrics
```

Enable daemon metrics in `/etc/docker/daemon.json`:

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

---

## Alerting Patterns

### Container Crash Loop

A container that restarts more than 3 times in 5 minutes:

```bash
# Check restart count
docker inspect --format '{{.RestartCount}}' myapp

# Watch for restart events
docker events --filter event=restart --filter container=myapp
```

### Disk Space Alert

```bash
# Check Docker disk usage
docker system df --format '{{.Type}}\t{{.Size}}\t{{.Reclaimable}}'

# Check host disk
df -h /var/lib/docker
```

**Source:** https://docs.docker.com/reference/dockerfile/#healthcheck
