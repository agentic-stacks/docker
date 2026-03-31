# Configure Docker Daemon

Agent reference for Docker daemon configuration. Use this to tune Docker behavior, select drivers, configure networking defaults, and manage logging.

---

## daemon.json

The primary configuration file for dockerd. Location: `/etc/docker/daemon.json`

Changes to daemon.json require either a restart or a live reload (depending on the field).

### Live Reload vs Restart

```bash
# Live reload (some fields only — see table below)
sudo kill -SIGHUP $(pidof dockerd)

# Full restart (all fields)
sudo systemctl restart docker
```

> **Warning:** Restarting Docker stops all running containers. Use live reload when possible.

| Field | Live Reload | Restart Required |
|-------|-------------|------------------|
| `debug` | Yes | No |
| `log-level` | Yes | No |
| `tls`, `tlscacert`, `tlscert`, `tlskey` | Yes | No |
| `default-runtime` | Yes | No |
| `authorization-plugins` | Yes | No |
| `storage-driver` | No | Yes |
| `data-root` | No | Yes |
| `dns`, `dns-search`, `dns-opts` | No | Yes |
| `default-address-pools` | No | Yes |
| `insecure-registries` | No | Yes |
| `registry-mirrors` | No | Yes |
| `log-driver`, `log-opts` | No | Yes |
| `userns-remap` | No | Yes |

### Complete Reference

```json
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {
      "base": "172.17.0.0/12",
      "size": 24
    }
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "dns-search": ["example.com"],
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["myregistry.local:5000"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "features": {
    "buildkit": true
  },
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB"
    }
  },
  "live-restore": true,
  "userland-proxy": false,
  "experimental": false,
  "debug": false,
  "log-level": "warn",
  "iptables": true,
  "ip-forward": true,
  "ip-masq": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "userns-remap": "default"
}
```

---

## Storage Drivers

The storage driver controls how image layers and container writable layers are stored on disk.

### Driver Comparison

| Driver | Backing Filesystem | Copy-on-Write | Performance | Recommended |
|--------|--------------------|---------------|-------------|-------------|
| `overlay2` | ext4, xfs | File-level | Good for most workloads | **Default — use this** |
| `btrfs` | btrfs | Block-level (snapshots) | Good for snapshot-heavy | Only if btrfs is already in use |
| `zfs` | zfs | Block-level (clones) | Good for data-intensive | Only if zfs is already in use |
| `fuse-overlayfs` | Any (FUSE) | File-level | Slower (userspace) | Rootless Docker fallback |

### Check Current Driver

```bash
docker info --format '{{.Driver}}'
# Expected: overlay2
```

### overlay2 Requirements

- Linux kernel 4.0+ (Ubuntu 22.04/Debian 12/RHEL 9 all qualify)
- Backing filesystem: ext4 (recommended) or xfs with `d_type=true`
- Check xfs d_type: `xfs_info /var/lib/docker | grep ftype`
  - `ftype=1` means d_type is enabled (good)
  - `ftype=0` means overlay2 will not work — reformat with `mkfs.xfs -n ftype=1`

### Change Data Root

Move Docker data to a different partition:

```bash
# 1. Stop Docker
sudo systemctl stop docker

# 2. Configure new path
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "data-root": "/mnt/docker-data"
}
EOF

# 3. Move existing data (optional)
sudo rsync -aP /var/lib/docker/ /mnt/docker-data/

# 4. Start Docker
sudo systemctl start docker

# 5. Verify
docker info --format '{{.DockerRootDir}}'
```

---

## Logging Drivers

Docker captures stdout and stderr from containers and routes them through a logging driver.

### Driver Comparison

| Driver | Destination | `docker logs` support | When to Use |
|--------|-------------|----------------------|-------------|
| `json-file` | Local JSON files | Yes | **Default — development and small deployments** |
| `journald` | systemd journal | Yes | systemd-based hosts, centralized via journalctl |
| `syslog` | Syslog daemon | No | Traditional syslog infrastructure |
| `fluentd` | Fluentd collector | No | Centralized logging (EFK/Loki stack) |
| `gelf` | Graylog (GELF) | No | Graylog deployments |
| `awslogs` | CloudWatch Logs | No | AWS deployments |
| `gcplogs` | Google Cloud Logging | No | GCP deployments |
| `none` | Nowhere | No | When logging is handled inside the container |

### Configure json-file with Rotation

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

Without `max-size`, log files grow unbounded and will eventually fill the disk. **Always configure log rotation.**

### Configure journald

```json
{
  "log-driver": "journald",
  "log-opts": {
    "tag": "{{.Name}}"
  }
}
```

Read logs: `journalctl CONTAINER_NAME=myapp --follow`

### Per-Container Override

Override the default logging driver for a specific container:

```bash
docker run -d --log-driver=json-file --log-opt max-size=50m --name myapp nginx:1.27
```

In Compose:

```yaml
services:
  app:
    image: nginx:1.27
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
```

---

## Network Configuration

### Default Address Pools

Docker assigns IP ranges to bridge networks from a default pool. To avoid conflicts with your LAN:

```json
{
  "default-address-pools": [
    {
      "base": "10.10.0.0/16",
      "size": 24
    }
  ]
}
```

Each new `docker network create` gets a /24 from the 10.10.0.0/16 range.

### DNS Configuration

```json
{
  "dns": ["10.0.0.2", "8.8.8.8"],
  "dns-search": ["corp.example.com"],
  "dns-opts": ["ndots:2"]
}
```

These apply to all containers that do not specify `--dns` explicitly.

### Disable iptables Management

If you manage iptables externally (e.g., firewalld, nftables):

```json
{
  "iptables": false
}
```

> **Warning:** Disabling iptables means Docker cannot set up port mappings or inter-container networking automatically. You must configure rules manually.

---

## Proxy Configuration

### For Docker Daemon (Image Pulls)

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.local"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

### For Containers (Build and Run)

In `~/.docker/config.json`:

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy.example.com:3128",
      "httpsProxy": "http://proxy.example.com:3128",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
```

Docker injects these as environment variables into `docker build` and `docker run` automatically.

---

## Live Restore

Keep containers running when the Docker daemon stops (for daemon upgrades):

```json
{
  "live-restore": true
}
```

With live-restore enabled:
- Containers continue running during `systemctl restart docker`
- Containers reconnect to the daemon after restart
- Does **not** work with Swarm mode

---

## Security-Related Configuration

### Userland Proxy

Docker uses a userland proxy for port mapping by default. Disable for better performance:

```json
{
  "userland-proxy": false
}
```

With `userland-proxy: false`, Docker uses iptables NAT rules directly (faster, less overhead).

### User Namespace Remapping

Map container root (UID 0) to an unprivileged host UID:

```json
{
  "userns-remap": "default"
}
```

This creates a `dockremap` user and maps container UIDs to a high-number range on the host. See `skills/operations/security` for detailed setup.

---

## Validate Configuration

```bash
# Check daemon.json syntax before restarting
python3 -c "import json; json.load(open('/etc/docker/daemon.json'))"

# Check effective configuration
docker info

# Check for configuration warnings
docker info 2>&1 | grep -i warning
```

**Source:** https://docs.docker.com/reference/cli/dockerd/
