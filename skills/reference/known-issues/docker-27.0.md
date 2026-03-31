# Docker 27.0.x Known Issues

---

### BuildKit cache mount permissions with rootless Docker

**Symptom:** `RUN --mount=type=cache,target=/path` fails with permission denied in rootless Docker.

**Cause:** Cache mount directories are created with root ownership, but rootless Docker maps root to a non-root UID.

**Workaround:**
```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip,uid=0,gid=0 \
    pip install -r requirements.txt
```

Explicitly set `uid` and `gid` on the cache mount.

**Affected versions:** 27.0.x through 27.1.x
**Status:** Improved in 27.2.0

---

### overlay2 "no space left on device" with sufficient disk

**Symptom:** Docker operations fail with "no space left on device" even though `df` shows available space.

**Cause:** Inode exhaustion on the Docker data partition. overlay2 creates many small files for layers.

**Workaround:**
```bash
# Check inode usage
df -i /var/lib/docker

# If inodes are exhausted, clean up
docker system prune -a
docker builder prune -a

# Long-term: reformat with more inodes or use XFS (dynamic inodes)
```

**Affected versions:** All Docker versions on ext4 with small inode count
**Status:** Not a Docker bug — filesystem limitation
