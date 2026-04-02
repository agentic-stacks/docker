# Operations: Security

Agent reference for Docker security hardening. Use this to reduce attack surface and meet compliance requirements.

---

## Security Checklist (Quick Assessment)

Run these checks to assess current security posture:

```bash
# 1. Are containers running as root?
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: User={{.Config.User}}' {}
# Empty User = running as root

# 2. Are containers using read-only root filesystem?
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: ReadOnly={{.HostConfig.ReadonlyRootfs}}' {}

# 3. Are there privileged containers?
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: Privileged={{.HostConfig.Privileged}}' {}

# 4. Docker socket exposed to any container?
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: {{range .Mounts}}{{if eq .Source "/var/run/docker.sock"}}SOCKET EXPOSED{{end}}{{end}}' {}

# 5. Any containers without resource limits?
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: Memory={{.HostConfig.Memory}} CPU={{.HostConfig.NanoCpus}}' {}
```

---

## Run as Non-Root

### In Dockerfile

```dockerfile
RUN groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app
WORKDIR /app
COPY --chown=app:app . .
USER app
```

### At Runtime

```bash
docker run --user 1000:1000 myapp:1.0
```

### In Compose

```yaml
services:
  app:
    user: "1000:1000"
```

---

## Rootless Docker

Run the entire Docker daemon as a non-root user:

```bash
# Install prerequisites
sudo apt-get install -y uidmap dbus-user-session

# Install rootless Docker (run as your normal user)
dockerd-rootless-setuptool.sh install

# Set socket path
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

Rootless Docker maps container UID 0 to your user's UID on the host. Even if a container process escapes, it runs as your unprivileged user.

---

## User Namespace Remapping

Map container root (UID 0) to an unprivileged host UID:

```json
// /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

```bash
# Restart Docker
sudo systemctl restart docker

# Verify — container root maps to high UID on host
docker run --rm alpine id
# uid=0(root) gid=0(root) — but on host, this is UID 100000+
```

### Check Mapping

```bash
cat /etc/subuid
# dockremap:100000:65536

cat /etc/subgid
# dockremap:100000:65536
```

---

## Read-Only Root Filesystem

Prevent containers from writing to the root filesystem:

```bash
docker run --read-only --tmpfs /tmp:size=50m --tmpfs /run myapp:1.0
```

In Compose:

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp:size=50m
      - /run
    volumes:
      - app-data:/data  # writable data directory
```

---

## Capability Dropping

Linux capabilities grant specific root privileges. Drop all and add only what's needed:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:1.0
```

In Compose:

```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # bind to ports < 1024
```

### Common Capabilities

| Capability | Purpose | Commonly Needed? |
|-----------|---------|------------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | Only if binding low ports |
| `CHOWN` | Change file ownership | Rarely |
| `SETUID`/`SETGID` | Change UID/GID | Rarely (init processes) |
| `SYS_PTRACE` | Debug processes | Only for debugging containers |
| `NET_RAW` | Raw sockets (ping) | Only if using ping/traceroute |
| `ALL` | All capabilities | **Never in production** |

---

## No New Privileges

Prevent privilege escalation via setuid/setgid binaries:

```bash
docker run --security-opt=no-new-privileges myapp:1.0
```

In Compose:

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

---

## Seccomp Profiles

Seccomp restricts which system calls a container can make. Docker applies a default profile that blocks ~44 dangerous syscalls.

### Use Default Profile (Recommended)

The default profile is applied automatically. To verify:

```bash
docker inspect --format '{{.HostConfig.SecurityOpt}}' myapp
# If empty, default seccomp is active
```

### Custom Seccomp Profile

```bash
# Download Docker's default profile as starting point
curl -o default-seccomp.json https://raw.githubusercontent.com/moby/moby/master/profiles/seccomp/default.json

# Run with custom profile
docker run --security-opt seccomp=custom-seccomp.json myapp:1.0
```

> **Warning:** `--security-opt seccomp=unconfined` disables seccomp entirely. Never do this in production.

---

## AppArmor (Ubuntu/Debian)

Docker applies a default AppArmor profile (`docker-default`) to all containers.

```bash
# Check AppArmor status
sudo aa-status | grep docker

# Run with custom AppArmor profile
docker run --security-opt apparmor=my-custom-profile myapp:1.0
```

## SELinux (RHEL/CentOS/Fedora)

```bash
# Check SELinux status
getenforce

# Run with SELinux labels
docker run --security-opt label=type:container_t myapp:1.0

# Disable SELinux labeling for a container (not recommended)
docker run --security-opt label=disable myapp:1.0
```

For bind mounts with SELinux:
```bash
# :z = shared label (multiple containers can access)
docker run -v /host/path:/container/path:z myapp:1.0

# :Z = private label (only this container can access)
docker run -v /host/path:/container/path:Z myapp:1.0
```

---

## Image Scanning

### Docker Scout (Built-in)

```bash
# Quick vulnerability overview
docker scout quickview myapp:1.0

# Detailed CVE list
docker scout cves myapp:1.0

# Compare with previous version
docker scout compare myapp:2.0 --to myapp:1.0
```

### Trivy

```bash
# Scan a local image
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:1.0

# Scan with severity filter
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity HIGH,CRITICAL myapp:1.0
```

### Grype

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/grype myapp:1.0
```

---

## Docker Bench for Security

Automated CIS Docker Benchmark audit:

```bash
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

---

## Docker Socket Protection

The Docker socket (`/var/run/docker.sock`) grants **full root access** to the host. If a container needs socket access (e.g., Traefik, Portainer):

### Socket Proxy (Recommended)

```yaml
services:
  socket-proxy:
    image: tecnativa/docker-socket-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      CONTAINERS: 1
      SERVICES: 1
      TASKS: 1
      NETWORKS: 0
      VOLUMES: 0
      IMAGES: 0
      EXEC: 0
      POST: 0  # read-only

  traefik:
    image: traefik:v3.1
    environment:
      DOCKER_HOST: tcp://socket-proxy:2375
    depends_on:
      - socket-proxy
    # No socket mount needed!
```

The proxy allows only whitelisted API endpoints.

**Source:** https://docs.docker.com/engine/security/
