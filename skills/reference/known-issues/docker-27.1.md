# Docker 27.1.x Known Issues

---

### docker compose watch high CPU on macOS

**Symptom:** `docker compose watch` consumes excessive CPU (50-100%) on macOS with large project directories.

**Cause:** File watching via the virtualization framework has overhead proportional to the number of watched files.

**Workaround:**
```yaml
develop:
  watch:
    - action: sync
      path: ./src           # Watch only specific directories, not .
      target: /app/src
      ignore:
        - node_modules/
        - .git/
```

Narrow the watch path and add ignore patterns.

**Affected versions:** All 27.x on macOS
**Status:** Ongoing optimization

---

### Swarm service update hangs with health check

**Symptom:** `docker service update --image` hangs indefinitely. The new container starts but the old one is never drained.

**Cause:** Health check start-period not configured. The new container is stuck in "starting" health state, and Swarm waits for it to become healthy before draining the old task.

**Workaround:**
```bash
docker service update \
  --health-start-period 30s \
  --image myapp:2.0 \
  web
```

Always set `--health-start-period` (or `start_period` in Compose) to give the application time to initialize.

**Affected versions:** All Swarm versions
**Status:** Expected behavior (misconfiguration)
