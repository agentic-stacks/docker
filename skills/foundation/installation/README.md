# Install Docker Engine

Agent reference for installing Docker Engine on supported platforms. Use this for fresh installations or reinstallation.

---

## Choose Your Platform

| Platform | Method | Notes |
|----------|--------|-------|
| Ubuntu 22.04/24.04 | apt repository | Recommended for servers |
| Debian 12 (Bookworm) | apt repository | Recommended for servers |
| RHEL 9 / CentOS Stream 9 | dnf repository | Requires removing Podman first |
| Fedora 39/40 | dnf repository | Requires removing Podman first |
| Arch Linux | pacman | `docker` package in community repo |
| macOS | Docker Desktop | VM-based, includes Compose and BuildKit |
| Windows | Docker Desktop | WSL2 or Hyper-V backend |

---

## Ubuntu (apt)

### Prerequisites

- Ubuntu 22.04 (Jammy) or 24.04 (Noble) — 64-bit
- Remove unofficial packages if present:

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y $pkg 2>/dev/null
done
```

### Install from Docker Repository

```bash
# 1. Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# 2. Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Verify installation
sudo docker run hello-world
```

---

## Debian (apt)

### Prerequisites

- Debian 12 (Bookworm) — 64-bit
- Remove unofficial packages:

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y $pkg 2>/dev/null
done
```

### Install from Docker Repository

```bash
# 1. Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# 2. Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Verify installation
sudo docker run hello-world
```

---

## RHEL 9 / CentOS Stream 9 (dnf)

### Prerequisites

```bash
# Remove Podman and conflicting packages
sudo dnf remove -y podman buildah containers-common
```

### Install from Docker Repository

```bash
# 1. Add Docker repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# 2. Install Docker Engine
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 3. Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# 4. Verify installation
sudo docker run hello-world
```

---

## Fedora (dnf)

### Prerequisites

```bash
# Remove Podman and conflicting packages
sudo dnf remove -y podman buildah containers-common
```

### Install from Docker Repository

```bash
# 1. Add Docker repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo

# 2. Install Docker Engine
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 3. Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# 4. Verify installation
sudo docker run hello-world
```

---

## Arch Linux (pacman)

```bash
# 1. Install Docker
sudo pacman -S docker docker-compose docker-buildx

# 2. Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# 3. Verify installation
sudo docker run hello-world
```

---

## macOS (Docker Desktop)

1. Download Docker Desktop from https://docs.docker.com/desktop/setup/install/mac-install/
2. Choose the correct architecture:
   - **Apple Silicon (M1/M2/M3/M4)**: `Docker Desktop for Mac with Apple Silicon`
   - **Intel**: `Docker Desktop for Mac with Intel chip`
3. Open the `.dmg` and drag Docker to Applications
4. Launch Docker from Applications
5. Verify: `docker run hello-world`

Docker Desktop on macOS runs a Linux VM (using Apple's Virtualization framework). Containers run inside this VM, not natively on macOS.

---

## Windows (Docker Desktop)

1. **Enable WSL2** (recommended) or Hyper-V:
   ```powershell
   wsl --install
   ```
2. Download Docker Desktop from https://docs.docker.com/desktop/setup/install/windows-install/
3. Run the installer
4. In Settings, ensure "Use WSL 2 based engine" is checked
5. Verify: `docker run hello-world`

---

## Post-Install Configuration (Linux)

### Add User to Docker Group

Running `docker` without `sudo`:

```bash
# Create the docker group (may already exist)
sudo groupadd docker

# Add your user
sudo usermod -aG docker $USER

# Apply group change (log out and back in, or run:)
newgrp docker

# Verify — should work without sudo
docker run hello-world
```

> **Security note:** The `docker` group grants root-equivalent privileges. Any user in this group can mount the host filesystem, access the Docker socket, and escalate to root. Only add trusted users.

### Enable Docker at Boot

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### Configure Default Logging

Add log rotation to prevent disk fill:

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

## Rootless Docker (Linux)

Run Docker without root privileges. The daemon and containers run in user namespace.

### Install Rootless

```bash
# Prerequisites
sudo apt-get install -y uidmap dbus-user-session

# Install rootless Docker (run as non-root user, NOT sudo)
dockerd-rootless-setuptool.sh install

# Add to shell profile
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock' >> ~/.bashrc
source ~/.bashrc

# Enable lingering (keeps user services running after logout)
sudo loginctl enable-linger $(whoami)

# Verify
docker run hello-world
```

### Rootless Limitations

| Feature | Rootless Support |
|---------|-----------------|
| Port binding < 1024 | No (use `--publish 8080:80` instead of `--publish 80:80`) |
| `--net=host` | Limited (uses slirp4netns or pasta) |
| Overlay network (Swarm) | No |
| AppArmor | No |
| cgroup v2 resource limits | Yes (if cgroup v2 enabled) |
| `--privileged` | Limited |

---

## Verify Installation

After any installation method, run the full verification:

```bash
# Docker version
docker version

# Docker system info (storage driver, cgroup version, kernel)
docker info

# Run test container
docker run --rm hello-world

# Verify Compose
docker compose version

# Verify BuildKit
docker buildx version
```

Expected output for `docker version`:
```
Client: Docker Engine - Community
 Version:           27.x.x
 ...
Server: Docker Engine - Community
 Version:           27.x.x
 ...
```

**Source:** https://docs.docker.com/engine/install/
