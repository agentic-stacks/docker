# Docker Architecture: Core Concepts

Agent reference for understanding Docker Engine internals. Use this before executing any container deployment, networking, or troubleshooting task.

---

## What Is Docker

Docker is a platform for building, shipping, and running applications in containers. Containers are lightweight, isolated processes that share the host kernel but have their own filesystem, networking, and process space.

Docker is **not** a virtual machine. Containers do not include a guest OS kernel — they use the host's kernel directly. This makes them faster to start (milliseconds vs minutes) and more resource-efficient (no hypervisor overhead).

**What Docker provides:**
- Image build system (Dockerfiles + BuildKit)
- Container runtime (containerd + runc)
- Image distribution (registries)
- Container orchestration (Compose for single-host, Swarm for multi-host)
- Networking, storage, and logging subsystems

**What Docker does not provide:** kernel-level isolation equivalent to VMs. Containers share the host kernel. For strong isolation, use gVisor, Kata Containers, or VMs.

---

## Architecture Overview

Docker uses a client-server architecture. The Docker client (`docker` CLI) communicates with the Docker daemon (`dockerd`) via a REST API over a Unix socket or TCP.

### Component Stack

```
docker CLI (client)
    │
    ▼ REST API (Unix socket /var/run/docker.sock or TCP)
dockerd (Docker daemon)
    │
    ▼ gRPC
containerd (container runtime manager)
    │
    ▼ OCI runtime spec
runc (OCI runtime — creates containers)
    │
    ▼ Linux kernel
namespaces + cgroups + union filesystem
```

### dockerd (Docker Daemon)

The Docker daemon manages Docker objects: images, containers, networks, volumes. It:

- Listens on `/var/run/docker.sock` (Unix) or TCP port 2375/2376
- Handles image builds, container lifecycle, network creation, volume management
- Delegates container execution to containerd
- Provides the Docker API (RESTful)

Configuration: `/etc/docker/daemon.json` — see `skills/foundation/configuration`.

### containerd

containerd is the container runtime that dockerd delegates to. It manages:

- Image pull and push (registry interactions)
- Container execution lifecycle (create, start, stop, delete)
- Snapshot management (filesystem layers)
- Task management (running processes in containers)

containerd is an independent CNCF project. Docker uses it, but it can also be used standalone (without dockerd) — this is how Kubernetes uses containerd directly.

### runc

runc is the low-level OCI runtime that creates and runs containers. It:

- Sets up Linux namespaces (PID, network, mount, UTS, IPC, user)
- Configures cgroups for resource limits
- Executes the container's entrypoint process
- Implements the OCI runtime specification

runc is a standalone binary. It receives a container spec (config.json) and a root filesystem, and creates a running container. Alternative runtimes (crun, gVisor/runsc, Kata) can replace runc.

---

## Image Layer Model

Docker images are built from ordered layers. Each layer represents a filesystem change (files added, modified, or deleted).

### How Layers Work

```
Layer 4: COPY app.py /app/          (adds app.py)
Layer 3: RUN pip install flask       (adds pip packages)
Layer 2: RUN apt-get install python3 (adds python3 binaries)
Layer 1: FROM ubuntu:22.04           (base OS filesystem)
```

Each layer is:
- **Immutable** — once created, a layer never changes
- **Content-addressable** — identified by SHA256 hash of its content
- **Shared** — multiple images can reference the same layer (deduplication)
- **Stacked** — union filesystem presents all layers as a single coherent tree

### Union Filesystem (overlay2)

overlay2 is the default storage driver on modern Linux. It merges layers into a unified view:

| Component | Path | Purpose |
|-----------|------|---------|
| `lowerdir` | Read-only image layers | Base filesystem from all image layers, stacked |
| `upperdir` | Read-write container layer | Changes made by the running container |
| `workdir` | Internal overlay2 workdir | Required by the kernel for atomic operations |
| `merged` | Union mount point | What the container sees — combined lower + upper |

When a container modifies a file from a lower layer, the file is **copied up** to the upper layer before modification (copy-on-write). The original layer remains unchanged.

### Image Manifests

An image is identified by:

| Identifier | Format | Example | Mutable? |
|-----------|--------|---------|----------|
| Tag | `name:tag` | `nginx:1.27` | Yes — can be repointed |
| Digest | `name@sha256:...` | `nginx@sha256:abc123...` | No — content-addressed |

**Always use digests or specific version tags in production.** The `:latest` tag is mutable — it changes whenever a new image is pushed without a tag.

---

## Linux Kernel Primitives

Containers rely on kernel features for isolation and resource control. Docker sets these up automatically, but understanding them helps with troubleshooting and security decisions.

### Namespaces (Isolation)

| Namespace | Isolates | Effect |
|-----------|----------|--------|
| `pid` | Process IDs | Container sees only its own processes. PID 1 inside = entrypoint. |
| `net` | Network stack | Container has its own interfaces, routing table, iptables rules. |
| `mnt` | Mount points | Container has its own filesystem tree (from image layers + volumes). |
| `uts` | Hostname | Container has its own hostname (set via `--hostname`). |
| `ipc` | IPC resources | Shared memory and semaphores are isolated between containers. |
| `user` | User/group IDs | Maps container UIDs to different host UIDs (rootless mode). |
| `cgroup` | Cgroup view | Container sees only its own cgroup hierarchy. |

### Cgroups (Resource Limits)

| Resource | Docker Flag | Example |
|----------|------------|---------|
| Memory limit | `--memory` | `--memory 512m` — kill if exceeds 512 MB |
| Memory reservation | `--memory-reservation` | `--memory-reservation 256m` — soft limit |
| CPU shares | `--cpu-shares` | `--cpu-shares 512` — relative weight (default 1024) |
| CPU count | `--cpus` | `--cpus 1.5` — max 1.5 CPU cores |
| CPU pinning | `--cpuset-cpus` | `--cpuset-cpus 0,1` — pin to cores 0 and 1 |
| PIDs limit | `--pids-limit` | `--pids-limit 100` — max 100 processes |
| Block I/O weight | `--blkio-weight` | `--blkio-weight 300` — relative I/O priority |

---

## Docker vs Alternatives

| Feature | Docker Engine | Podman | containerd (standalone) | LXC/LXD |
|---------|--------------|--------|------------------------|---------|
| **Daemon** | Yes (dockerd) | Daemonless | Yes (containerd) | Yes (lxd) |
| **Root required** | By default (rootless available) | Rootless by default | By default | By default |
| **CLI compatibility** | `docker` | `podman` (docker-compatible) | `ctr`, `nerdctl` | `lxc` |
| **Compose support** | `docker compose` | `podman-compose` or `docker compose` | `nerdctl compose` | N/A |
| **Swarm mode** | Yes | No | No | No |
| **BuildKit** | Built-in | Via `buildah` | Separate `buildctl` | N/A |
| **Kubernetes** | Uses containerd underneath | CRI-O or containerd | Direct CRI | N/A |
| **Image format** | OCI / Docker v2 | OCI | OCI | System containers |
| **Use case** | Development + production | Security-focused, rootless | Kubernetes runtime | System containers |

### When to Use Docker vs Alternatives

- **Docker Engine**: best for development workflows, Compose-based deployments, Swarm orchestration
- **Podman**: when daemonless/rootless is required by policy, or in systemd-managed environments
- **containerd standalone**: when running Kubernetes (no need for dockerd layer)
- **LXC/LXD**: when you need persistent system containers (closer to VMs than app containers)

---

## Container Lifecycle

```
Image (stored) → Created → Running → Paused → Running → Stopped → Removed
                    │                                        │
                    └─ docker create ─────────── docker rm ──┘
                            │                        ▲
                            ▼                        │
                       docker start ──── docker stop/kill
                            │
                            ▼
                       docker pause/unpause
```

| State | Description | Persists Data? |
|-------|-------------|----------------|
| Created | Container exists but process has not started | Yes (writable layer exists) |
| Running | Process is active | Yes |
| Paused | Process frozen (SIGSTOP via cgroups freezer) | Yes |
| Stopped | Process exited (exit code available) | Yes (writable layer preserved) |
| Removed | Container deleted | No (writable layer deleted, volumes preserved if named) |

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success — process exited normally |
| 1 | Application error — generic failure |
| 126 | Command not executable — permission denied |
| 127 | Command not found — binary missing in image |
| 137 | SIGKILL (128 + 9) — typically OOM killed or `docker kill` |
| 139 | SIGSEGV (128 + 11) — segmentation fault |
| 143 | SIGTERM (128 + 15) — `docker stop` sent SIGTERM |

---

## Quick Reference: Essential Commands

| Task | Command |
|------|---------|
| Run a container | `docker run -d --name myapp -p 8080:80 nginx:1.27` |
| List running containers | `docker ps` |
| List all containers | `docker ps -a` |
| View logs | `docker logs myapp --follow --tail 100` |
| Execute command in container | `docker exec -it myapp /bin/sh` |
| Stop container | `docker stop myapp` |
| Remove container | `docker rm myapp` |
| Build image | `docker build -t myapp:1.0 .` |
| List images | `docker images` |
| Pull image | `docker pull nginx:1.27` |
| System info | `docker info` |
| Disk usage | `docker system df` |
| Clean up | `docker system prune` |

**Source:** https://docs.docker.com/get-started/docker-overview/
