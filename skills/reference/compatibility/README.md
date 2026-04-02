# Compatibility Matrices

Version and platform compatibility reference. Check this before deploying or changing infrastructure.

---

## OS and Kernel Requirements

### Docker Engine 27.x

| OS | Minimum Version | Kernel | Architecture | Status |
|----|----------------|--------|--------------|--------|
| Ubuntu 24.04 (Noble) | GA | 6.8+ | amd64, arm64 | Supported |
| Ubuntu 22.04 (Jammy) | GA | 5.15+ | amd64, arm64 | Supported |
| Debian 12 (Bookworm) | GA | 6.1+ | amd64, arm64 | Supported |
| RHEL 9 | 9.0+ | 5.14+ | amd64, arm64 | Supported |
| Fedora 39/40 | GA | 6.5+ | amd64, arm64 | Supported |
| Arch Linux | Rolling | Latest | amd64 | Community |
| macOS | 12.0+ (Monterey) | N/A (VM) | amd64, arm64 | Docker Desktop |
| Windows | 10/11 Pro/Enterprise | WSL2 / Hyper-V | amd64, arm64 | Docker Desktop |

### Minimum Kernel Features

| Feature | Minimum Kernel | Required By |
|---------|---------------|-------------|
| overlay2 | 4.0 | Default storage driver |
| cgroup v2 | 5.2 | Modern resource management |
| user namespaces | 3.8 | Rootless Docker, userns-remap |
| seccomp | 3.5 | Default security profile |

---

## Storage Driver Compatibility

| Driver | Backing Filesystem | Kernel | Docker Version | Notes |
|--------|-------------------|--------|----------------|-------|
| overlay2 | ext4 | 4.0+ | 17.06+ | **Default and recommended** |
| overlay2 | xfs (ftype=1) | 4.0+ | 17.06+ | Must have d_type support |
| btrfs | btrfs | 3.18+ | All | Only if already using btrfs |
| zfs | zfs | zfs 0.8+ | All | Only if already using ZFS |
| fuse-overlayfs | Any | 4.18+ | 20.10+ | Rootless Docker fallback |

### Check Storage Driver Compatibility

```bash
# Current driver
docker info --format '{{.Driver}}'

# Backing filesystem
df -T /var/lib/docker

# XFS d_type support (must be ftype=1)
xfs_info /var/lib/docker | grep ftype
```

---

## Docker Compose Compatibility

| Docker Compose Version | Docker Engine | Compose File Spec | Notes |
|----------------------|---------------|-------------------|-------|
| V2.24+ | 24.0+ | Compose Specification | Current — no `version:` field needed |
| V2.20-2.23 | 23.0+ | Compose Specification | Stable |
| V1 (docker-compose) | Any | version: "2"/"3" | **Deprecated — do not use** |

### Compose Specification Notes

- The `version:` field is **optional and ignored** in Compose V2. Omit it.
- The `deploy:` section works in both Swarm and `docker compose up` (for resource limits)
- `docker-compose` (hyphenated, V1) is deprecated. Use `docker compose` (space, V2 plugin).

---

## Docker Desktop vs Docker Engine

| Feature | Docker Engine (Linux) | Docker Desktop (Mac/Win/Linux) |
|---------|----------------------|-------------------------------|
| **Runtime** | Native Linux containers | Linux VM + containers |
| **Performance** | Native | Overhead from VM layer |
| **Networking** | Full Linux networking | VM networking (some limitations) |
| **host network mode** | Works (shares host network) | Shares VM network, not actual host |
| **File sharing** | Native bind mounts | VirtioFS/gRPC-FUSE (slower for large trees) |
| **GPU access** | Native (nvidia-container-toolkit) | Supported on some platforms |
| **Kubernetes** | Not included | Built-in single-node K8s |
| **License** | Free (Apache 2.0) | Free for personal/small business, paid for enterprise |
| **Swarm mode** | Full support | Full support |
| **BuildKit** | Included | Included |

---

## Architecture Support

| Architecture | Docker Engine | Docker Desktop | Common Platforms |
|-------------|---------------|----------------|------------------|
| `linux/amd64` | Yes | Yes | Intel/AMD x86_64 servers |
| `linux/arm64` | Yes | Yes | AWS Graviton, Apple Silicon (in VM), Ampere |
| `linux/arm/v7` | Yes | No | Raspberry Pi 3/4 (32-bit) |
| `linux/arm/v6` | Community | No | Raspberry Pi Zero/1 |
| `linux/s390x` | Yes | No | IBM Z mainframes |
| `linux/ppc64le` | Yes | No | IBM POWER |

### Check Platform

```bash
# Host architecture
uname -m

# Docker's platform
docker info --format '{{.Architecture}}'

# Image platform
docker inspect --format '{{.Os}}/{{.Architecture}}' nginx:1.27
```

---

## containerd Compatibility

| Docker Engine | Bundled containerd | Notes |
|--------------|-------------------|-------|
| 27.x | 1.7.x | Stable pairing |
| 26.x | 1.7.x | Stable pairing |
| 25.x | 1.6.x - 1.7.x | Transition period |

```bash
# Check containerd version
containerd --version
docker info --format '{{.ServerVersion}}' # Docker version
```

**Source:** https://docs.docker.com/engine/install/
