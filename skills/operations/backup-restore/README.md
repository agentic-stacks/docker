# Operations: Backup and Restore

Agent reference for backing up and restoring Docker data. Use this before destructive operations or for disaster recovery planning.

---

## What to Back Up

| Data | Location | Method |
|------|----------|--------|
| Named volumes | Docker-managed (`/var/lib/docker/volumes/`) | Volume backup via helper container |
| Compose files | Project directory | git / file backup |
| .env files | Project directory | Secure file backup (contains secrets) |
| Docker configs | `/etc/docker/daemon.json` | File backup |
| Swarm state | `/var/lib/docker/swarm/` | Directory backup |
| TLS certificates | Varies | File backup |
| Registry data | Registry volume | Volume backup |

---

## Volume Backup

### Backup a Single Volume

```bash
# Create compressed backup
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/mydata-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
```

### Backup All Named Volumes

```bash
#!/bin/bash
# backup-all-volumes.sh
BACKUP_DIR="$(pwd)/backups/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

for vol in $(docker volume ls -q); do
  echo "Backing up volume: $vol"
  docker run --rm \
    -v "$vol":/source:ro \
    -v "$BACKUP_DIR":/backup \
    alpine tar czf "/backup/${vol}.tar.gz" -C /source .
done

echo "Backups saved to $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
```

### Backup with Container Stopped (Consistent State)

For databases, stop the container before backup to ensure consistency:

```bash
# Stop the database
docker compose stop db

# Backup the volume
docker run --rm \
  -v myapp_db-data:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/db-data-$(date +%Y%m%d).tar.gz -C /source .

# Restart the database
docker compose start db
```

### PostgreSQL Logical Backup (Preferred for Databases)

```bash
# pg_dump while container is running
docker exec myapp-db pg_dump -U postgres mydb > backups/mydb-$(date +%Y%m%d).sql

# Compressed
docker exec myapp-db pg_dump -U postgres -Fc mydb > backups/mydb-$(date +%Y%m%d).dump
```

---

## Volume Restore

### Restore a Volume

```bash
# Create the target volume (or use existing)
docker volume create mydata

# Restore from backup
docker run --rm \
  -v mydata:/target \
  -v $(pwd)/backups:/backup:ro \
  alpine sh -c "cd /target && tar xzf /backup/mydata-20260329.tar.gz"
```

### Restore PostgreSQL

```bash
# SQL format
docker exec -i myapp-db psql -U postgres mydb < backups/mydb-20260329.sql

# Custom format
docker exec -i myapp-db pg_restore -U postgres -d mydb < backups/mydb-20260329.dump
```

---

## docker save / docker load (Images)

Backup and restore images with all layers and metadata:

```bash
# Save image to tar archive
docker save -o myapp-1.0.tar myapp:1.0

# Save multiple images
docker save -o all-images.tar myapp:1.0 postgres:16 redis:7

# Load image from tar archive
docker load -i myapp-1.0.tar
```

Use case: transferring images to air-gapped environments without registry access.

## docker export / docker import (Containers)

Backup a container's filesystem (flattened, no layers):

```bash
# Export container filesystem
docker export myapp > myapp-filesystem.tar

# Import as a new image (loses CMD, ENV, EXPOSE, etc.)
docker import myapp-filesystem.tar myapp-flat:1.0
```

> **Prefer `docker save`/`docker load`** over export/import. Export flattens all layers and loses image metadata.

---

## Swarm State Backup

```bash
# Stop Docker on the manager (careful — affects cluster!)
sudo systemctl stop docker

# Backup Swarm state
sudo tar czf swarm-backup-$(date +%Y%m%d).tar.gz -C /var/lib/docker/swarm .

# Restart Docker
sudo systemctl start docker
```

### Swarm Disaster Recovery

If all managers are lost:

```bash
# On a node with the backup:
sudo systemctl stop docker
sudo rm -rf /var/lib/docker/swarm
sudo mkdir -p /var/lib/docker/swarm
sudo tar xzf swarm-backup-20260329.tar.gz -C /var/lib/docker/swarm
sudo systemctl start docker

# Force new cluster from this node's state
docker swarm init --force-new-cluster --advertise-addr 192.168.1.10
```

---

## Automated Backup Script

```bash
#!/bin/bash
# /opt/scripts/docker-backup.sh
set -euo pipefail

BACKUP_ROOT="/backups/docker"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$DATE"
RETAIN_DAYS=7

mkdir -p "$BACKUP_DIR"

# Backup all named volumes
for vol in $(docker volume ls -q --filter "dangling=false"); do
  docker run --rm \
    -v "$vol":/source:ro \
    -v "$BACKUP_DIR":/backup \
    alpine tar czf "/backup/vol-${vol}.tar.gz" -C /source . 2>/dev/null || \
    echo "WARNING: Failed to backup volume $vol"
done

# Backup daemon config
cp /etc/docker/daemon.json "$BACKUP_DIR/" 2>/dev/null || true

# Cleanup old backups
find "$BACKUP_ROOT" -maxdepth 1 -type d -mtime +$RETAIN_DAYS -exec rm -rf {} \;

echo "Backup completed: $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
```

### Cron Schedule

```bash
# Run daily at 2 AM
echo "0 2 * * * /opt/scripts/docker-backup.sh >> /var/log/docker-backup.log 2>&1" | sudo crontab -
```

**Source:** https://docs.docker.com/engine/storage/volumes/
