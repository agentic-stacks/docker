# Operations: Storage

Agent reference for Docker storage — volumes, bind mounts, tmpfs, and storage drivers. Use this when managing persistent data.

---

## Storage Types

| Type | Managed by Docker | Persistent | Performance | Use Case |
|------|-------------------|------------|-------------|----------|
| Named volume | Yes | Yes | Best | **Database data, app state** |
| Anonymous volume | Yes | Until removed | Best | Temporary scratch data |
| Bind mount | No (host path) | Yes | Host-dependent | Config files, development code |
| tmpfs | No (memory) | No (RAM only) | Fastest | Secrets, temp files, caches |

---

## Named Volumes

```bash
# Create a named volume
docker volume create mydata

# Use in docker run
docker run -d -v mydata:/var/lib/postgresql/data postgres:16

# Use in Compose
# volumes:
#   mydata:
# services:
#   db:
#     volumes:
#       - mydata:/var/lib/postgresql/data
```

### Volume Management

```bash
# List volumes
docker volume ls

# Inspect volume (shows mount point on host)
docker volume inspect mydata

# Remove a volume (data is permanently deleted!)
docker volume rm mydata

# Remove all unused volumes
docker volume prune

# Remove all unused volumes including named ones
docker volume prune -a
```

> **Warning:** `docker volume rm` and `docker volume prune` permanently delete data. Always back up before removing.

---

## Bind Mounts

Mount a host directory into a container:

```bash
# Bind mount (host path must be absolute)
docker run -v /host/path:/container/path myapp:1.0

# Read-only bind mount
docker run -v /host/configs:/etc/app/configs:ro myapp:1.0
```

In Compose:

```yaml
services:
  app:
    volumes:
      - ./src:/app/src                    # relative path (to compose file)
      - /etc/ssl/certs:/etc/ssl/certs:ro  # absolute path, read-only
```

### When to Use Bind Mounts vs Volumes

| Scenario | Use |
|----------|-----|
| Database storage | Named volume |
| Application state | Named volume |
| Config files (read-only) | Bind mount with `:ro` |
| Development source code | Bind mount |
| Sharing data between host and container | Bind mount |
| Data that must survive `docker system prune` | Named volume |

---

## tmpfs Mounts

Memory-backed filesystem — data is never written to disk:

```bash
docker run --tmpfs /tmp:size=100m myapp:1.0
```

In Compose:

```yaml
services:
  app:
    tmpfs:
      - /tmp:size=100m
    # Or long syntax:
    volumes:
      - type: tmpfs
        target: /app/cache
        tmpfs:
          size: 50m
```

Use for: secrets at runtime, temporary caches, scratch space that must not persist.

---

## Volume Drivers

### Local Driver (Default)

```bash
# Standard local volume
docker volume create mydata

# NFS mount
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/data \
  nfs-data
```

In Compose:

```yaml
volumes:
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw,nfsvers=4
      device: ":/exports/data"
```

### CIFS/SMB Mount

```yaml
volumes:
  smb-data:
    driver: local
    driver_opts:
      type: cifs
      o: username=user,password=pass,addr=192.168.1.100
      device: "//192.168.1.100/share"
```

---

## Volume Backup and Restore

### Backup a Named Volume

```bash
# Create a backup using a temporary container
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata-backup-$(date +%Y%m%d).tar.gz -C /source .
```

### Restore a Volume

```bash
# Create the target volume
docker volume create mydata-restored

# Restore from backup
docker run --rm \
  -v mydata-restored:/target \
  -v $(pwd):/backup:ro \
  alpine sh -c "cd /target && tar xzf /backup/mydata-backup-20260329.tar.gz"
```

### Clone a Volume

```bash
docker volume create mydata-clone
docker run --rm \
  -v mydata:/source:ro \
  -v mydata-clone:/target \
  alpine sh -c "cd /source && tar cf - . | (cd /target && tar xf -)"
```

---

## Disk Usage Analysis

```bash
# Summary: images, containers, volumes, build cache
docker system df

# Detailed breakdown
docker system df -v

# Find large volumes
docker system df -v 2>/dev/null | grep -A1 "VOLUME NAME"

# Find the host path of a volume
docker volume inspect --format '{{.Mountpoint}}' mydata
# Usually: /var/lib/docker/volumes/mydata/_data

# Check actual disk usage of volume data
sudo du -sh /var/lib/docker/volumes/mydata/_data
```

---

## Storage Driver Internals

### overlay2 (Default)

```
Image layers (read-only):
  Layer 1: base OS (ubuntu:22.04)
  Layer 2: apt-get install python3
  Layer 3: pip install flask
  Layer 4: COPY app.py

Container layer (read-write):
  Layer 5: runtime changes (logs, temp files, state)
```

overlay2 uses the Linux kernel's OverlayFS. Each layer is a directory on disk. The kernel merges them into a single view.

**Copy-on-write:** When a container modifies a file from an image layer, the file is copied to the container's writable layer before modification. The original image layer is unchanged.

### Check Storage Driver

```bash
docker info --format '{{.Driver}}'
# Expected: overlay2

# Data directory
docker info --format '{{.DockerRootDir}}'
# Default: /var/lib/docker
```

---

## Sharing Volumes Between Containers

```yaml
services:
  writer:
    image: myapp:1.0
    volumes:
      - shared-data:/data

  reader:
    image: nginx:1.27
    volumes:
      - shared-data:/usr/share/nginx/html:ro

volumes:
  shared-data:
```

> **Caution:** Multiple containers writing to the same volume can cause data corruption unless the application handles concurrent access (e.g., database with locking).

**Source:** https://docs.docker.com/engine/storage/
