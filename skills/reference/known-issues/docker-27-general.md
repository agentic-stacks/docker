# Docker 27.x General Known Issues

Issues that affect all Docker 27.x minor versions.

---

### Container DNS fails after daemon restart with live-restore

**Symptom:** Containers lose DNS resolution after Docker daemon restart, even with `live-restore: true`. Containers can still reach IPs but not hostnames.

**Cause:** The embedded DNS server (127.0.0.11) is part of the daemon process. During restart, DNS requests have no handler. After restart, some containers' DNS config may not reconnect.

**Workaround:**
```bash
# Restart affected containers
docker restart <container-name>

# Or for Compose projects
docker compose restart
```

**Affected versions:** All 27.x (inherited behavior)
**Status:** Known limitation of live-restore
