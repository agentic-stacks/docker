# Known Issues

Version-specific bugs, caveats, and workarounds for Docker Engine. Check this before deploying or upgrading.

---

## How to Use

Each version has its own file with issues in this format:

```
### [Short Description]

**Symptom:** What the operator sees
**Cause:** Why it happens
**Workaround:** Exact steps to fix or avoid it
**Affected versions:** x.y.z through x.y.w
**Status:** Open / Fixed in x.y.w
```

## Version Index

| Version | File | Key Issues |
|---------|------|------------|
| Docker 27.0.x | [docker-27.0.md](docker-27.0.md) | BuildKit rootless cache mounts, overlay2 inode exhaustion |
| Docker 27.1.x | [docker-27.1.md](docker-27.1.md) | Compose watch CPU on macOS, Swarm health check hang |
| All 27.x | [docker-27-general.md](docker-27-general.md) | Live-restore DNS, inode exhaustion |

---

## Reporting New Issues

When encountering a potential Docker bug:

1. **Reproduce** on a clean Docker installation if possible
2. **Collect diagnostics:**
   ```bash
   docker version
   docker info
   docker inspect <affected-container>
   journalctl -u docker --since "1 hour ago"
   ```
3. **Search existing issues** at https://github.com/moby/moby/issues
4. **File a report** with reproduction steps, expected vs actual behavior, and diagnostic output

**Source:** https://github.com/moby/moby/issues
