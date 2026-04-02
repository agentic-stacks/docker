# docker

Complete operational knowledge for building, deploying, managing, and troubleshooting production containerized applications using [Docker Engine](https://docs.docker.com/engine/). An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that turns AI agents into Docker experts.

## What This Is

This repository is a structured knowledge base — not code. It teaches AI agents (Claude, Gemini, GPT, etc.) how to operate Docker across the full container lifecycle: from Dockerfile authoring and image optimization through production deployment, day-two operations, and disaster recovery.

When an agent reads the `CLAUDE.md` entry point, it gains:
- Deep understanding of Docker's layered architecture (dockerd, containerd, runc)
- Step-by-step procedures with exact `docker` and `docker compose` commands
- Decision frameworks for choosing storage drivers, networking modes, base images, and registries
- Safety guardrails that prevent destructive operations without approval
- Symptom-based troubleshooting decision trees

## Coverage

| Area | What's Covered |
|---|---|
| **Dockerfile** | Every instruction, multi-stage builds, heredoc syntax, language-specific examples |
| **Compose** | Full Compose V2 spec, profiles, extends, watch, dependency ordering |
| **Build** | BuildKit, cache mounts, multi-platform builds, image size analysis |
| **Registries** | Docker Hub, GHCR, ECR, GCR, ACR, Harbor, private registry:2, cosign |
| **Deployment** | Single-host, Docker Swarm, production hardening, blue-green/canary |
| **Networking** | Bridge, host, overlay, macvlan, ipvlan, DNS, IPv6 |
| **Storage** | Volumes, bind mounts, tmpfs, NFS, storage drivers |
| **Security** | Rootless, user namespaces, seccomp, AppArmor, CIS benchmarks, image scanning |
| **Operations** | Health checks, upgrades, backup/restore, monitoring |
| **Docker version** | 27.x |

## Quick Start

### For AI Agent Users

Pull this stack into your project:

```bash
agentic-stacks init ./my-app --name my-app --namespace my-org --from docker
```

Or clone directly:

```bash
git clone https://github.com/agentic-stacks/docker.git .stacks/docker
```

Then point your agent to `CLAUDE.md` (or `.stacks/docker/CLAUDE.md` if using the stacks workflow). The agent will use the routing table to navigate to the right skill for any task.

### For Humans

Browse the skills directly:

- **New to Docker?** Start with [`skills/foundation/concepts`](skills/foundation/concepts/README.md)
- **Writing Dockerfiles?** See [`skills/develop/dockerfile`](skills/develop/dockerfile/README.md)
- **Setting up Compose?** See [`skills/develop/compose`](skills/develop/compose/README.md)
- **Going to production?** Follow the [production deployment workflow](#workflows)
- **Something broken?** Jump to [`skills/diagnose/troubleshooting`](skills/diagnose/troubleshooting/README.md)

## Skills

### Foundation
| Skill | Description |
|---|---|
| [`foundation/concepts`](skills/foundation/concepts/README.md) | Docker architecture, containerd, OCI runtimes, image layers, namespaces/cgroups |
| [`foundation/installation`](skills/foundation/installation/README.md) | Installing Docker Engine on Linux, macOS, Windows, post-install configuration |
| [`foundation/configuration`](skills/foundation/configuration/README.md) | daemon.json settings, storage drivers, logging drivers, runtime configuration |

### Develop
| Skill | Description |
|---|---|
| [`develop/dockerfile`](skills/develop/dockerfile/README.md) | Dockerfile syntax, instructions, multi-stage builds, best practices |
| [`develop/compose`](skills/develop/compose/README.md) | Compose file authoring, services, profiles, extends, variable interpolation |
| [`develop/build-optimization`](skills/develop/build-optimization/README.md) | BuildKit, layer caching, .dockerignore, image size reduction, build arguments |
| [`develop/registry`](skills/develop/registry/README.md) | Push/pull workflows, private registries, Harbor, ECR/GCR/ACR, image signing |

### Deploy
| Skill | Description |
|---|---|
| [`deploy/single-host`](skills/deploy/single-host/README.md) | docker run, compose up, restart policies, resource constraints |
| [`deploy/swarm`](skills/deploy/swarm/README.md) | Swarm init, services, stacks, overlay networking, rolling updates, secrets |
| [`deploy/production`](skills/deploy/production/README.md) | Reverse proxies, TLS termination, secrets management, resource limits, logging |

### Operations
| Skill | Description |
|---|---|
| [`operations/health-check`](skills/operations/health-check/README.md) | Container health checks, system health, docker events, monitoring integration |
| [`operations/networking`](skills/operations/networking/README.md) | Bridge, host, overlay, macvlan, DNS resolution, port mapping, network troubleshooting |
| [`operations/storage`](skills/operations/storage/README.md) | Volumes, bind mounts, tmpfs, storage drivers, NFS, shared storage |
| [`operations/security`](skills/operations/security/README.md) | Rootless mode, user namespaces, seccomp, AppArmor, image scanning, CIS benchmarks |
| [`operations/upgrades`](skills/operations/upgrades/README.md) | Docker Engine upgrades, image update strategies, zero-downtime redeployment |
| [`operations/backup-restore`](skills/operations/backup-restore/README.md) | Volume backup, container export/import, Swarm state backup, disaster recovery |

### Diagnose
| Skill | Description |
|---|---|
| [`diagnose/troubleshooting`](skills/diagnose/troubleshooting/README.md) | Symptom-based diagnostic decision trees for common Docker issues |

### Reference
| Skill | Description |
|---|---|
| [`reference/cli-reference`](skills/reference/cli-reference/README.md) | Essential docker and docker compose CLI patterns and one-liners |
| [`reference/known-issues`](skills/reference/known-issues/README.md) | Docker 27.x specific bugs, caveats, and workarounds |
| [`reference/compatibility`](skills/reference/compatibility/README.md) | OS, kernel, storage driver, and runtime compatibility matrices |
| [`reference/decision-guides`](skills/reference/decision-guides/README.md) | Trade-off analyses for storage drivers, networking modes, orchestration, registries |

## Workflows

### New Project

```
foundation/concepts → foundation/installation → foundation/configuration
→ develop/dockerfile → develop/compose → develop/build-optimization
→ deploy/single-host → operations/health-check
```

### Production Deployment

```
develop/registry → deploy/production → operations/networking
→ operations/security → operations/health-check
```

### Swarm Cluster

```
foundation/concepts → deploy/swarm → operations/networking
→ operations/security → operations/health-check
```

### Existing Deployment

Jump directly to the relevant `operations/`, `diagnose/`, or `develop/` skill.

## Required Tools

| Tool | Purpose |
|---|---|
| `docker` | Docker CLI (version must match target Docker Engine) |
| `docker compose` | Docker Compose V2 (bundled as docker compose plugin) |
| `docker buildx` | Docker Buildx for advanced build features (bundled with Docker Desktop) |

## Project Structure

When using this stack, your operator project should look like:

```
my-app/
├── CLAUDE.md                    # Points to .stacks/docker/
├── stacks.lock
├── .stacks/
│   └── docker/                  # This stack
├── Dockerfile                   # Application image definition
├── docker-compose.yml           # Service definitions
├── docker-compose.override.yml  # Local dev overrides (gitignored)
├── docker-compose.prod.yml      # Production overrides
├── .env.example                 # Template for environment variables
├── .env                         # Actual env vars (gitignored)
├── .dockerignore                # Build context exclusions
├── configs/                     # Mounted config files
├── scripts/                     # Operational scripts
└── data/                        # Volume mount points (gitignored)
```

## Contributing

This stack follows the [agentic-stacks](https://github.com/agentic-stacks/agentic-stacks) format. Each skill is a directory under `skills/` with a `README.md` entry point.

To add or update content:
1. Follow the existing writing style (imperative headings, exact commands, decision trees)
2. Verify commands against the [official Docker docs](https://docs.docker.com)
3. Add version-specific notes where behavior differs between Docker releases
4. Update `stack.yaml` if adding new skills

## License

MIT
