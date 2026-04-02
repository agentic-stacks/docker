# Deploy: Docker Swarm

Agent reference for Docker Swarm mode — multi-host container orchestration built into Docker Engine. Use this for multi-node deployments without Kubernetes.

---

## Initialize a Swarm

```bash
# Initialize on the first manager node
docker swarm init --advertise-addr 192.168.1.10

# Output includes join tokens for workers and managers
```

### Get Join Tokens

```bash
# Worker join token
docker swarm join-token worker

# Manager join token
docker swarm join-token manager
```

### Join Nodes

```bash
# Join as worker (run on worker node)
docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# Join as manager (run on new manager node)
docker swarm join --token SWMTKN-1-yyy 192.168.1.10:2377
```

---

## Manager Topology

Swarm uses Raft consensus. Managers must be an odd number to maintain quorum.

| Managers | Quorum | Tolerates Failures | Recommendation |
|----------|--------|-------------------|----------------|
| 1 | 1 | 0 | Dev/test only |
| 3 | 2 | 1 | **Minimum for production** |
| 5 | 3 | 2 | Large deployments |
| 7 | 4 | 3 | Maximum recommended |

> **Never use an even number of managers.** With 2 managers, losing 1 loses quorum (both must agree). With 3 managers, you can lose 1 and still have quorum (2 of 3).

### Node Management

```bash
# List nodes
docker node ls

# Inspect a node
docker node inspect node-hostname

# Promote worker to manager
docker node promote node-hostname

# Demote manager to worker
docker node demote node-hostname

# Drain node (stop scheduling, migrate tasks)
docker node update --availability drain node-hostname

# Reactivate node
docker node update --availability active node-hostname

# Remove a node (run on node being removed)
docker swarm leave        # worker
docker swarm leave --force  # manager
```

---

## Services

A service defines the desired state for a set of container replicas.

### Create a Service

```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  --network app-net \
  --env NODE_ENV=production \
  --mount type=volume,source=web-data,target=/data \
  --limit-cpu 1 \
  --limit-memory 512m \
  --restart-condition on-failure \
  --restart-max-attempts 3 \
  --update-parallelism 1 \
  --update-delay 30s \
  --update-failure-action rollback \
  --health-cmd "curl -f http://localhost:80/health || exit 1" \
  --health-interval 30s \
  nginx:1.27
```

### Service Types

| Type | Flag | Behavior |
|------|------|----------|
| Replicated | `--mode replicated --replicas N` | N copies distributed across nodes (default) |
| Global | `--mode global` | Exactly one instance on every node |

### Manage Services

```bash
# List services
docker service ls

# Service details
docker service ps web

# Scale
docker service scale web=5

# Update image
docker service update --image nginx:1.28 web

# Update environment variable
docker service update --env-add NEW_VAR=value web

# Remove service
docker service rm web

# View logs
docker service logs -f web
```

---

## Stack Deploys

Deploy a multi-service application from a Compose file:

```yaml
# stack.yml
services:
  web:
    image: myapp:1.0
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    ports:
      - "8080:80"
    networks:
      - app-net
    secrets:
      - db-password

  db:
    image: postgres:16
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net
    secrets:
      - db-password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password

networks:
  app-net:
    driver: overlay
    encrypted: true

volumes:
  db-data:

secrets:
  db-password:
    external: true
```

```bash
# Deploy the stack
docker stack deploy -c stack.yml myapp

# List stacks
docker stack ls

# List stack services
docker stack services myapp

# List stack tasks
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

---

## Secrets and Configs

### Secrets

```bash
# Create a secret from a file
echo "mysecretpassword" | docker secret create db-password -

# Create from file
docker secret create tls-cert /path/to/cert.pem

# List secrets
docker secret ls

# Use in a service
docker service create --secret db-password --name web myapp:1.0
```

Secrets are mounted at `/run/secrets/<secret-name>` inside containers as files.

### Configs

```bash
# Create a config
docker config create nginx-conf /path/to/nginx.conf

# Use in a service
docker service create \
  --config source=nginx-conf,target=/etc/nginx/nginx.conf \
  --name web nginx:1.27
```

---

## Rolling Updates

```bash
# Update with controlled rollout
docker service update \
  --image myapp:2.0 \
  --update-parallelism 1 \
  --update-delay 30s \
  --update-failure-action rollback \
  --update-max-failure-ratio 0.25 \
  --update-monitor 15s \
  web
```

| Parameter | Meaning |
|-----------|---------|
| `--update-parallelism` | How many tasks to update at once |
| `--update-delay` | Pause between batches |
| `--update-failure-action` | `pause`, `continue`, or `rollback` on failure |
| `--update-max-failure-ratio` | Max fraction of failed tasks before triggering failure action |
| `--update-monitor` | How long to monitor a task after update before considering it healthy |

### Rollback

```bash
# Rollback to previous version
docker service rollback web

# Automatic rollback (configured in update-failure-action)
```

---

## Overlay Networking

```bash
# Create an overlay network
docker network create --driver overlay app-net

# Encrypted overlay (all traffic encrypted with AES-GCM)
docker network create --driver overlay --opt encrypted app-net

# Attachable overlay (allows standalone containers to connect)
docker network create --driver overlay --attachable app-net
```

Services on the same overlay network can reach each other by service name across nodes.

---

## Placement Constraints

```yaml
deploy:
  placement:
    constraints:
      - node.role == manager
      - node.labels.zone == us-east-1a
      - node.hostname == node-01
    preferences:
      - spread: node.labels.zone
```

### Label Nodes

```bash
docker node update --label-add zone=us-east-1a node-01
docker node update --label-add ssd=true node-02
```

**Source:** https://docs.docker.com/engine/swarm/
