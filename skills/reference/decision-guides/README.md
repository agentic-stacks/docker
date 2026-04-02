# Decision Guides

Trade-off analyses for choosing between Docker options. Use this when the operator needs to make a technology decision.

---

## Base Image Selection

| Image | Compressed Size | Shell | Package Manager | glibc | Use Case |
|-------|----------------|-------|-----------------|-------|----------|
| `scratch` | 0 MB | No | No | No | Static Go/Rust binaries |
| `distroless/static` | ~2 MB | No | No | No | Static binaries + ca-certs + /etc/passwd |
| `distroless/base` | ~20 MB | No | No | Yes | Dynamic binaries needing glibc |
| `alpine:3.20` | ~3.5 MB | sh | apk | No (musl) | Small images with debugging access |
| `debian:bookworm-slim` | ~30 MB | bash | apt | Yes | Maximum compatibility |
| `ubuntu:22.04` | ~30 MB | bash | apt | Yes | Wide ecosystem, familiar |

### Recommendation by Language

| Language | Recommended Base | Why |
|----------|-----------------|-----|
| Go (static) | `scratch` or `distroless/static` | Smallest possible, no runtime deps |
| Go (CGo) | `distroless/base` or `alpine` | Needs C library |
| Rust | `scratch` or `distroless/static` | Static by default |
| Python | `python:3.12-slim` | Needs Python runtime |
| Node.js | `node:20-slim` | Needs Node runtime |
| Java | `eclipse-temurin:21-jre` | Needs JVM |
| .NET | `mcr.microsoft.com/dotnet/aspnet:8.0` | Needs .NET runtime |

### Migration Path

Changing base image later is straightforward — update the FROM line and test. The main risk is musl vs glibc incompatibilities (alpine uses musl). If you encounter issues with alpine, switch to a debian-slim variant.

---

## Storage Driver Selection

| Driver | When to Use | When NOT to Use |
|--------|-------------|-----------------|
| **overlay2** | Almost always. Default, well-tested, fast. | Never avoid without specific reason. |
| **btrfs** | Host already runs btrfs. Want snapshot features. | Don't adopt btrfs just for Docker. |
| **zfs** | Host already runs ZFS. Want ZFS snapshots and compression. | Don't adopt ZFS just for Docker. |
| **fuse-overlayfs** | Rootless Docker on older kernels. | When native overlay2 is available. |

**Recommendation:** Use overlay2 unless your host filesystem is already btrfs or ZFS.

---

## Networking Mode Selection

| Mode | Performance | Isolation | Port Mapping | Multi-Host | When to Use |
|------|-------------|-----------|-------------|------------|-------------|
| **bridge (user-defined)** | Good | Full | Yes | No | **Default for most apps** |
| **host** | Best | None | Not needed | No | Maximum throughput, single service per port |
| **overlay** | Good | Full | Yes | Yes | Docker Swarm services |
| **macvlan** | Good | Full | Not needed | No | Container needs LAN IP (DHCP, legacy protocols) |
| **ipvlan** | Good | Full | Not needed | No | Like macvlan, but shares host MAC address |
| **none** | N/A | Complete | No | No | Security-sensitive containers with no network need |

### Recommendation

- **Single host, multiple services:** User-defined bridge (default)
- **Multi-host orchestration:** Overlay (Swarm) or external CNI
- **Performance-critical single service:** Host networking
- **Must appear on LAN:** Macvlan

---

## Orchestration: Compose vs Swarm vs Kubernetes

| Feature | Compose | Swarm | Kubernetes |
|---------|---------|-------|------------|
| **Scope** | Single host | Multi-host cluster | Multi-host cluster |
| **Complexity** | Low | Medium | High |
| **Setup time** | Minutes | Minutes | Hours to days |
| **Rolling updates** | Manual | Built-in | Built-in |
| **Auto-scaling** | No | No | Yes (HPA) |
| **Service discovery** | DNS (within Compose network) | DNS (across nodes) | DNS + Service objects |
| **Secrets** | File-based (Compose) | Encrypted at rest (Raft) | Encrypted at rest (etcd) |
| **Health checks** | HEALTHCHECK + depends_on | HEALTHCHECK + update policy | Liveness/Readiness probes |
| **Load balancing** | None (use reverse proxy) | Built-in (routing mesh) | Built-in (Service/Ingress) |
| **Best for** | Dev, small prod, single-server | Small-medium multi-host | Large-scale production |

### Recommendation

- **Single server or dev environment:** Docker Compose
- **Small team, few servers, no K8s expertise:** Docker Swarm
- **Large scale, complex networking, auto-scaling needed:** Kubernetes

### Migration Path

- Compose → Swarm: Compose files work with `docker stack deploy` (minor differences in networking)
- Swarm → Kubernetes: Requires rewriting service definitions as K8s manifests (different model)
- Compose → Kubernetes: Use `kompose convert` as a starting point, then refine

---

## Registry Selection

| Registry | Hosting | Free Tier | Auth | Best For |
|----------|---------|-----------|------|----------|
| **Docker Hub** | SaaS | 1 private repo | Docker ID | Public images, small teams |
| **GHCR** | SaaS | Unlimited private (with GitHub plan) | GitHub token | GitHub-based projects |
| **ECR** | AWS | 500 MB/month (free tier) | AWS IAM | AWS deployments |
| **GCR/Artifact Registry** | GCP | 500 MB/month (free tier) | GCP IAM | GCP deployments |
| **ACR** | Azure | Basic tier (~$5/month) | Azure AD | Azure deployments |
| **Harbor** | Self-hosted | N/A (free software) | LDAP/OIDC | Enterprise, air-gapped, compliance |
| **registry:2** | Self-hosted | N/A (free) | htpasswd/token | Simple private registry |

### Recommendation

- **Open source project:** Docker Hub (public) or GHCR
- **Cloud-native team:** Use your cloud provider's registry (ECR/GCR/ACR)
- **Enterprise with compliance:** Harbor (self-hosted, auditing, vulnerability scanning)
- **Quick private registry:** registry:2 with TLS + htpasswd

---

## Logging Driver Selection

| Driver | Destination | `docker logs` | Performance | When to Use |
|--------|-------------|--------------|-------------|-------------|
| **json-file** | Local disk | Yes | Good | **Default — dev and small prod** |
| **journald** | systemd journal | Yes | Good | systemd-integrated hosts |
| **fluentd** | Fluentd collector | No | Good | Centralized logging (EFK/Loki) |
| **syslog** | Syslog daemon | No | Good | Traditional syslog infrastructure |
| **awslogs** | CloudWatch | No | Good | AWS deployments |
| **gcplogs** | Cloud Logging | No | Good | GCP deployments |
| **none** | Nowhere | No | Best | App handles its own logging |

### Recommendation

- **Development:** json-file (default) with rotation configured
- **Production single-host:** json-file with rotation + Promtail/Loki sidecar
- **Production multi-host:** fluentd or application-level logging to centralized system
- **Cloud:** Use cloud-native driver (awslogs, gcplogs)

**Always configure log rotation** regardless of driver. Without `max-size`, json-file logs grow unbounded.

**Source:** https://docs.docker.com/engine/
