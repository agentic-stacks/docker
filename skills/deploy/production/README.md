# Deploy: Production Hardening

Agent reference for production Docker deployments. Use this when preparing containers for production — reverse proxies, TLS, resource limits, and operational patterns.

---

## Reverse Proxy Patterns

### Traefik (Auto-Discovery)

Traefik automatically discovers containers via Docker labels:

```yaml
services:
  traefik:
    image: traefik:v3.1
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt
    restart: unless-stopped

  app:
    image: myapp:1.0
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      - "traefik.http.services.app.loadbalancer.server.port=8080"
    restart: unless-stopped

volumes:
  letsencrypt:
```

> **Security note:** Traefik needs Docker socket access. Use read-only (`:ro`) and consider a socket proxy (see `skills/operations/security`).

### Caddy (Automatic HTTPS)

```yaml
services:
  caddy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    restart: unless-stopped

  app:
    image: myapp:1.0
    restart: unless-stopped

volumes:
  caddy-data:
  caddy-config:
```

```
# Caddyfile
app.example.com {
    reverse_proxy app:8080
}
```

Caddy obtains and renews TLS certificates automatically via Let's Encrypt.

### Nginx

```yaml
services:
  nginx:
    image: nginx:1.27
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      app:
        condition: service_healthy
    restart: unless-stopped

  app:
    image: myapp:1.0
    restart: unless-stopped
```

```nginx
# nginx.conf
events { worker_connections 1024; }
http {
    upstream app {
        server app:8080;
    }
    server {
        listen 80;
        server_name app.example.com;
        return 301 https://$host$request_uri;
    }
    server {
        listen 443 ssl;
        server_name app.example.com;
        ssl_certificate /etc/nginx/certs/cert.pem;
        ssl_certificate_key /etc/nginx/certs/key.pem;
        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## Resource Limits

**Always set resource limits in production.** Without limits, a single container can consume all host resources.

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 256M
    pids_limit: 200
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

---

## Secrets Management

### Docker Secrets (Swarm Only)

```bash
# Create secret
printf "my-secret-value" | docker secret create api-key -

# Use in service
docker service create --secret api-key --name app myapp:1.0
# Secret available at /run/secrets/api-key
```

### File-Based Secrets (Compose)

```yaml
secrets:
  db-password:
    file: ./secrets/db-password.txt

services:
  db:
    secrets:
      - db-password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
```

### External Secret Managers

For production, integrate with Vault, AWS Secrets Manager, or similar:

```yaml
services:
  app:
    environment:
      # Populated by init container or sidecar
      DATABASE_URL_FILE: /run/secrets/database-url
    volumes:
      - secrets:/run/secrets:ro

  secret-init:
    image: vault-agent:latest
    volumes:
      - secrets:/run/secrets
```

---

## Log Aggregation

### Fluentd/Loki Pattern

```yaml
services:
  app:
    logging:
      driver: fluentd
      options:
        fluentd-address: "localhost:24224"
        tag: "app.{{.Name}}"

  fluentd:
    image: fluent/fluentd:v1.17
    volumes:
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
    ports:
      - "24224:24224"
```

### JSON Structured Logging

Configure applications to output JSON logs:

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
```

---

## Systemd Integration

Run Docker Compose projects as systemd services for auto-start and management:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App Docker Compose
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/docker compose up -d --wait
ExecStop=/usr/bin/docker compose down
ExecReload=/usr/bin/docker compose pull && /usr/bin/docker compose up -d --remove-orphans
TimeoutStartSec=120

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
```

---

## Deployment Patterns

### Blue-Green (Zero Downtime)

```bash
# Deploy new version as separate project
docker compose -p myapp-green -f docker-compose.yml -f docker-compose.green.yml up -d --wait

# Verify health
curl http://localhost:8081/health

# Switch proxy to green
# (update Traefik labels, nginx upstream, or DNS)

# Tear down old version
docker compose -p myapp-blue down
```

### Rolling Update (Compose)

```bash
# Pull new images
docker compose pull

# Recreate containers one at a time (with health check)
docker compose up -d --no-deps --build app
```

### Canary

```yaml
services:
  app-stable:
    image: myapp:1.0
    deploy:
      replicas: 9

  app-canary:
    image: myapp:2.0
    deploy:
      replicas: 1
```

Route 10% of traffic to canary via load balancer weighting.

**Source:** https://docs.docker.com/compose/production/
