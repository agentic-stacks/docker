# Operations: Networking

Agent reference for Docker networking — drivers, DNS, port mapping, and troubleshooting. Use this when configuring container connectivity.

---

## Network Drivers

| Driver | Scope | Use Case | Container-to-Container | External Access |
|--------|-------|----------|----------------------|-----------------|
| `bridge` | Single host | **Default for standalone containers** | Via user-defined bridge | Port mapping (-p) |
| `host` | Single host | Maximum network performance | Shares host network stack | Direct (no mapping) |
| `none` | Single host | Complete isolation | No networking | No networking |
| `overlay` | Multi-host | **Swarm services** | Across nodes via VXLAN | Port mapping |
| `macvlan` | Single host | Direct LAN integration | Each container gets LAN IP | Direct (no NAT) |
| `ipvlan` | Single host | Similar to macvlan, L3 mode | Each container gets LAN IP | Direct (no NAT) |

---

## Bridge Networks

### Default Bridge vs User-Defined Bridge

| Feature | Default Bridge (`bridge`) | User-Defined Bridge |
|---------|--------------------------|---------------------|
| DNS resolution | No (use `--link`, deprecated) | **Yes — by container name** |
| Automatic isolation | No (all containers share) | Yes (per-network isolation) |
| Connect/disconnect live | No | Yes |
| Recommended | No | **Yes — always use user-defined** |

### Create and Use a Bridge Network

```bash
# Create network
docker network create mynet

# Run container on network
docker run -d --name app --network mynet myapp:1.0

# Connect existing container to network
docker network connect mynet existing-container

# Disconnect
docker network disconnect mynet existing-container
```

In Compose, user-defined networks are created automatically:

```yaml
services:
  app:
    networks:
      - backend
  db:
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

`app` can reach `db` at hostname `db` (DNS name = service name).

---

## DNS Resolution

On user-defined networks, Docker runs an embedded DNS server at `127.0.0.11`:

- **Container name** → container IP (e.g., `ping db` resolves to db container's IP)
- **Service name** → VIP or container IPs (in Swarm, load-balanced across replicas)
- **Network alias** → container IP (multiple containers can share an alias)

### Network Aliases

```bash
docker run -d --name app1 --network mynet --network-alias app myapp:1.0
docker run -d --name app2 --network mynet --network-alias app myapp:1.0
# Both respond to DNS name "app" (round-robin)
```

### DNS Configuration per Container

```bash
docker run --dns 10.0.0.2 --dns-search example.com --dns-opt ndots:2 myapp:1.0
```

---

## Port Mapping

```bash
# Map host port to container port
docker run -p 8080:80 nginx:1.27

# Map to specific interface
docker run -p 127.0.0.1:8080:80 nginx:1.27

# Map random host port
docker run -p 80 nginx:1.27

# UDP port
docker run -p 53:53/udp dns-server:1.0

# Multiple ports
docker run -p 8080:80 -p 8443:443 nginx:1.27
```

### Check Published Ports

```bash
docker port myapp
# 80/tcp -> 0.0.0.0:8080
```

---

## Host Networking

Container shares the host's network stack directly:

```bash
docker run --network host myapp:1.0
```

- No port mapping needed — container binds directly to host ports
- No network isolation
- Best performance (no NAT overhead)
- **Linux only** (on macOS/Windows, host networking uses the VM's network, not the actual host)

---

## Macvlan

Give containers their own MAC address and IP on the physical network:

```bash
# Create macvlan network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# Run container with LAN IP
docker run -d --network my-macvlan --ip 192.168.1.50 myapp:1.0
```

The container appears as a separate device on the LAN. Useful for services that need a dedicated IP (e.g., DHCP servers, network appliances).

> **Limitation:** The host cannot directly communicate with macvlan containers on the same interface. Use a macvlan sub-interface for host-to-container communication.

---

## Overlay Networks (Swarm)

```bash
# Create overlay network
docker network create --driver overlay --attachable my-overlay

# Encrypted overlay
docker network create --driver overlay --opt encrypted my-secure-overlay
```

Overlay networks use VXLAN to tunnel traffic between Swarm nodes. Services on the same overlay can communicate by service name across hosts.

---

## IPv6

```json
// daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80"
}
```

Per-network IPv6:

```bash
docker network create --ipv6 --subnet fd00:1::/80 mynet6
```

---

## Network Inspection and Debugging

```bash
# List networks
docker network ls

# Inspect network (shows connected containers and IPs)
docker network inspect mynet

# Find container's IP
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# Find container's networks
docker inspect --format '{{json .NetworkSettings.Networks}}' myapp | jq 'keys'

# Test connectivity from inside a container
docker exec myapp ping -c 3 db
docker exec myapp nslookup db
docker exec myapp curl -v http://db:5432

# Use a debug container on the same network
docker run --rm -it --network mynet nicolaka/netshoot
# Inside netshoot: dig, nslookup, ping, curl, tcpdump, iperf, etc.
```

### iptables Rules

```bash
# View Docker's iptables rules
sudo iptables -L -n -t nat | grep -A5 DOCKER
sudo iptables -L -n -t filter | grep -A5 DOCKER
```

---

## Common Networking Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Container can't reach another by name | Using default bridge (no DNS) | Switch to user-defined network |
| Port mapping not working | Firewall blocking, port already in use | Check `ss -tlnp`, firewalld/ufw rules |
| Container can't reach internet | DNS misconfigured, proxy needed | Check `--dns`, daemon.json dns settings |
| Containers on different networks can't talk | Expected — networks are isolated | Connect both to a shared network |
| macvlan container unreachable from host | macvlan limitation | Use macvlan sub-interface |

**Source:** https://docs.docker.com/engine/network/
