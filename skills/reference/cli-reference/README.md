# CLI Reference

Essential docker and docker compose command patterns. Use this for quick command lookup.

---

## Build Commands

```bash
# Build image
docker build -t myapp:1.0 .

# Build with BuildKit (default in Docker 23+)
DOCKER_BUILDKIT=1 docker build -t myapp:1.0 .

# Build with build args
docker build --build-arg NODE_ENV=production -t myapp:1.0 .

# Build specific stage
docker build --target builder -t myapp:builder .

# Build with no cache
docker build --no-cache -t myapp:1.0 .

# Build with progress output
docker build --progress=plain -t myapp:1.0 .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .
```

---

## Run Commands

```bash
# Run detached with port mapping
docker run -d --name myapp -p 8080:80 nginx:1.27

# Run interactive shell
docker run --rm -it ubuntu:22.04 /bin/bash

# Run with environment
docker run -d -e KEY=value --env-file .env myapp:1.0

# Run with volume
docker run -d -v mydata:/data myapp:1.0

# Run with resource limits
docker run -d --memory 512m --cpus 1.0 --pids-limit 100 myapp:1.0

# Run with security options
docker run -d --read-only --user 1000:1000 --cap-drop ALL myapp:1.0

# Run on specific network
docker run -d --network mynet --name app myapp:1.0

# Run with healthcheck
docker run -d --health-cmd "curl -f http://localhost/ || exit 1" --health-interval 30s myapp:1.0
```

---

## Inspect Commands

```bash
# Container status
docker inspect --format '{{.State.Status}}' myapp

# Container IP address
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# Container environment variables
docker inspect --format '{{json .Config.Env}}' myapp | jq .

# Container mounts
docker inspect --format '{{json .Mounts}}' myapp | jq .

# Container health
docker inspect --format '{{.State.Health.Status}}' myapp

# Container exit code
docker inspect --format '{{.State.ExitCode}}' myapp

# Container restart count
docker inspect --format '{{.RestartCount}}' myapp

# Image layers
docker inspect --format '{{json .RootFS.Layers}}' myapp:1.0 | jq .

# Image creation date
docker inspect --format '{{.Created}}' myapp:1.0

# Image size
docker inspect --format '{{.Size}}' myapp:1.0
```

---

## Cleanup Commands

```bash
# Remove stopped containers
docker container prune -f

# Remove unused images (dangling only)
docker image prune -f

# Remove ALL unused images (not just dangling)
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Remove build cache
docker builder prune -f

# Remove everything unused
docker system prune -f

# Nuclear: everything including named volumes
docker system prune -a --volumes -f
```

---

## Debug Commands

```bash
# Exec into running container
docker exec -it myapp /bin/sh

# Exec as root (even if USER is set)
docker exec -u root -it myapp /bin/sh

# Copy file from container
docker cp myapp:/app/log.txt ./log.txt

# Copy file to container
docker cp ./config.json myapp:/app/config.json

# View real-time events
docker events --filter type=container

# Follow logs with timestamps
docker logs -f -t --tail 100 myapp

# View processes inside container
docker top myapp

# View resource usage
docker stats myapp --no-stream
```

---

## Compose Commands

```bash
# Start services
docker compose up -d

# Start and wait for healthy
docker compose up -d --wait

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# Rebuild and restart
docker compose up -d --build

# Pull new images
docker compose pull

# Update with new images
docker compose pull && docker compose up -d --remove-orphans

# View status
docker compose ps

# View logs
docker compose logs -f app

# Run one-off command
docker compose run --rm app npm test

# Execute in running service
docker compose exec app sh

# Validate Compose file
docker compose config

# Scale a service
docker compose up -d --scale worker=3
```

---

## Go Template Formatting

Docker `--format` uses Go templates:

```bash
# Table format
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# JSON output
docker ps --format '{{json .}}' | jq .

# Conditional
docker ps --format '{{if eq .State "running"}}UP{{else}}DOWN{{end}}: {{.Names}}'

# Range (iterate arrays)
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' myapp

# Index (access map)
docker inspect --format '{{index .Config.Labels "com.example.version"}}' myapp
```

### Useful One-Liners

```bash
# Find containers using more than 500MB memory
docker stats --no-stream --format "{{.Name}}\t{{.MemUsage}}" | sort -k2 -h -r

# List all container IPs
docker ps -q | xargs -I{} docker inspect --format '{{.Name}}: {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' {}

# Find large images
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h -r | head -10

# List dangling volumes
docker volume ls -f dangling=true

# Show containers with restart count > 0
docker ps -q | xargs -I{} docker inspect --format '{{if gt .RestartCount 0}}{{.Name}}: {{.RestartCount}} restarts{{end}}' {} | grep -v "^$"

# Export all container logs to files
docker ps --format '{{.Names}}' | xargs -I{} sh -c 'docker logs {} > {}.log 2>&1'
```

**Source:** https://docs.docker.com/reference/cli/docker/
