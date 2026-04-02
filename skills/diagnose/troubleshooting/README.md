# Diagnose — Troubleshooting Guide

Agent reference for diagnosing Docker issues. Use this before attempting any remediation.

---

## Safety Rules

1. **Diagnose first, then propose a fix** — never remediate without understanding the root cause.
2. **Capture state before changing anything** — run `docker inspect` and save logs before restarting.
3. **Use the least destructive option** — prefer restart over recreate, recreate over remove.
4. **Never run `docker system prune` without understanding what will be removed.**
5. **Back up volumes before any destructive operation.**

---

## Diagnostic Toolkit

| # | Command | Purpose | When to Use |
|---|---------|---------|-------------|
| 1 | `docker ps -a` | List all containers with status | First command for any issue |
| 2 | `docker logs <container> --tail 100` | View recent container logs | Container crashed, error messages |
| 3 | `docker inspect <container>` | Full container configuration and state | Exit codes, health, mounts, network |
| 4 | `docker events --since 1h` | Recent Docker daemon events | Unexpected restarts, OOM kills |
| 5 | `docker stats --no-stream` | Current resource usage | Performance issues, resource exhaustion |
| 6 | `docker system df` | Disk usage by type | Disk space issues |
| 7 | `docker network inspect <net>` | Network configuration and connected containers | Connectivity issues |
| 8 | `docker volume inspect <vol>` | Volume mount point and driver | Volume/permission issues |
| 9 | `journalctl -u docker --since "1 hour ago"` | Docker daemon logs | Daemon crashes, startup failures |
| 10 | `docker exec <container> sh` | Interactive shell inside container | Deep debugging |

---

## Symptom Decision Trees

### 1. Container Won't Start

```
docker ps -a shows container "Created" or "Exited" immediately
|
+-> Check exit code:
|   docker inspect --format '{{.State.ExitCode}}' <container>
|   |
|   +-> Exit 0: Container ran and completed normally
|   |   → Is this expected? If one-shot task, this is correct.
|   |
|   +-> Exit 1: Application error
|   |   → docker logs <container> — check application error output
|   |
|   +-> Exit 126: Command not executable
|   |   → Check ENTRYPOINT/CMD — binary exists but not executable
|   |   → Check: docker inspect --format '{{json .Config.Entrypoint}}' <container>
|   |
|   +-> Exit 127: Command not found
|   |   → Binary missing in image. Wrong base image? Missing install step?
|   |   → docker run --rm --entrypoint sh <image> -c "which <binary>"
|   |
|   +-> Exit 137: OOM killed (SIGKILL)
|   |   → Check memory limit: docker inspect --format '{{.HostConfig.Memory}}' <container>
|   |   → Check kernel OOM: journalctl -k | grep -i oom
|   |   → Increase --memory or fix memory leak
|   |
|   +-> Exit 139: Segmentation fault
|   |   → Application bug or architecture mismatch (amd64 image on arm64?)
|   |   → docker inspect --format '{{.Platform}}' <image>
|   |
|   +-> Exit 143: SIGTERM received
|       → Container was stopped. Check docker events for who stopped it.
|
+-> Check OCI runtime error:
    docker logs <container> 2>&1 | grep -i "oci\|runtime"
    → "executable file not found" → wrong ENTRYPOINT/CMD
    → "permission denied" → entrypoint not executable (chmod +x)
    → "no such file or directory" → binary missing or wrong path
```

### 2. Container OOM Killed

```
Container exits with code 137 or docker events shows "oom" event
|
+-> Check container memory limit
|   docker inspect --format '{{.HostConfig.Memory}}' <container>
|   (0 = no limit)
|
+-> Check actual memory usage at time of kill
|   docker stats --no-stream <container>  (if still running)
|   journalctl -k | grep oom             (kernel OOM messages)
|
+-> Is the limit too low?
|   +-> Yes → Increase --memory (Compose: deploy.resources.limits.memory)
|   +-> No → Application is leaking memory
|       → Profile the application
|       → Check for known memory leaks in the application framework
```

### 3. Networking Between Containers Fails

```
Container A cannot reach Container B by name
|
+-> Are they on the same network?
|   docker inspect --format '{{json .NetworkSettings.Networks}}' containerA | jq 'keys'
|   docker inspect --format '{{json .NetworkSettings.Networks}}' containerB | jq 'keys'
|   |
|   +-> Different networks → Connect to same network:
|       docker network connect <network> <container>
|
+-> Are they on the default bridge?
|   +-> Yes → Default bridge has NO DNS resolution
|       → Switch to a user-defined network
|
+-> Can they ping each other by IP?
|   docker exec containerA ping <containerB-IP>
|   |
|   +-> No → Network driver issue, firewall, or iptables
|       → Check: sudo iptables -L -n | grep -A5 DOCKER
|   +-> Yes → DNS issue
|       → docker exec containerA nslookup containerB
|       → Check DNS is 127.0.0.11 inside container
```

### 4. DNS Resolution Broken

```
Container cannot resolve hostnames
|
+-> Can it resolve container names on same network?
|   docker exec <container> nslookup <other-container>
|   |
|   +-> No → Check network type (default bridge has no DNS)
|       docker inspect --format '{{json .NetworkSettings.Networks}}' <container>
|
+-> Can it resolve external names?
|   docker exec <container> nslookup google.com
|   |
|   +-> No → Check DNS configuration:
|       docker exec <container> cat /etc/resolv.conf
|       → nameserver should be 127.0.0.11 on user-defined networks
|       → Check host DNS: cat /etc/resolv.conf
|       → Check daemon.json dns settings
```

### 5. Volume Permission Errors

```
Application cannot read/write to mounted volume
|
+-> Check file ownership inside container
|   docker exec <container> ls -la /data/
|   docker exec <container> id
|
+-> Does the container user's UID match the volume file owner?
|   |
|   +-> No → Fix ownership:
|       # Option 1: Change ownership in entrypoint
|       # Option 2: Run container as matching UID
|       docker run --user $(stat -c '%u:%g' /host/path) ...
|       # Option 3: Use init container to fix permissions
|
+-> Is SELinux blocking access?
|   getenforce   # If Enforcing:
|   → Use :z or :Z suffix on bind mount
|   → docker run -v /host/path:/data:z ...
|
+-> Is the volume read-only?
    docker inspect --format '{{json .Mounts}}' <container> | jq '.[] | {Source, Destination, RW}'
```

### 6. Build Cache Invalidated Unexpectedly

```
Docker build rebuilds layers that shouldn't have changed
|
+-> Is the build context sending unexpected files?
|   → Check .dockerignore — missing .git, node_modules, etc.
|   → Look at "transferring context" size in build output
|
+-> Are COPY/ADD instructions ordered correctly?
|   → Dependency files (package.json, go.mod) should be copied
|     BEFORE the rest of the source code
|   → See `skills/develop/build-optimization`
|
+-> Did timestamps change? (ADD/COPY check file metadata)
|   → git checkout can change timestamps
|   → Use --no-cache if needed for clean build
```

### 7. Image Pull Failures

```
docker pull fails
|
+-> "unauthorized" or "denied"
|   → docker login <registry>
|   → Check credentials: cat ~/.docker/config.json
|   → Check image name/tag spelling
|
+-> "manifest not found" or "not found"
|   → Image or tag doesn't exist
|   → Check exact tag: docker manifest inspect <image>
|   → Check architecture: --platform linux/amd64 vs linux/arm64
|
+-> "connection refused" or "timeout"
|   → Check network/proxy: curl -v https://registry-1.docker.io/v2/
|   → Check daemon.json for proxy settings
|   → Check firewall rules
|
+-> "x509: certificate signed by unknown authority"
    → Private registry with self-signed cert
    → Add to insecure-registries in daemon.json
    → Or install the CA cert on the host
```

### 8. Disk Space Exhaustion

```
"no space left on device" errors
|
+-> Check Docker disk usage
|   docker system df
|   docker system df -v
|
+-> Identify the largest consumer:
|   |
|   +-> Images → Remove unused images
|       docker image prune -a
|   |
|   +-> Build cache → Clear builder cache
|       docker builder prune
|   |
|   +-> Containers → Remove stopped containers
|       docker container prune
|   |
|   +-> Volumes → Identify and clean large volumes
|       docker volume ls
|       for v in $(docker volume ls -q); do
|         echo "$v: $(docker run --rm -v $v:/data alpine du -sh /data 2>/dev/null | cut -f1)"
|       done
|
+-> Configure log rotation (prevent future disk fill)
    → See `skills/foundation/configuration` — log-opts max-size
```

### 9. Docker Daemon Won't Start

```
systemctl start docker fails
|
+-> Check daemon logs
|   journalctl -u docker --no-pager -n 50
|
+-> Common causes:
|   |
|   +-> "failed to start daemon: error initializing graphdriver"
|       → Storage driver issue. Check /etc/docker/daemon.json
|       → Try removing driver-specific state (CAREFUL — data loss)
|   |
|   +-> "address already in use"
|       → Another process using port 2375/2376
|       → ss -tlnp | grep 2375
|   |
|   +-> JSON parse error in daemon.json
|       → Validate: python3 -c "import json; json.load(open('/etc/docker/daemon.json'))"
|   |
|   +-> "permission denied" on socket
|       → Check socket ownership: ls -la /var/run/docker.sock
|       → systemctl restart docker
```

### 10. Container Crash Loop

```
Container keeps restarting (restart count increasing)
|
+-> Check restart count and last exit
|   docker inspect --format '{{.RestartCount}} exits: {{.State.ExitCode}}' <container>
|
+-> Check logs from previous run
|   docker logs <container> --tail 50
|   docker logs <container> --previous  (if using Swarm)
|
+-> Is it OOM? (exit 137)
|   → See "Container OOM Killed" tree above
|
+-> Is it a dependency issue?
|   → Database not ready? API not reachable?
|   → Add health check + depends_on condition: service_healthy
|
+-> Is the config wrong?
    → docker inspect --format '{{json .Config.Env}}' <container> | jq .
    → Check mounted config files
```

**Source:** https://docs.docker.com/config/daemon/troubleshoot/
