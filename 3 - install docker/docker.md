# How to Install Docker on Ubuntu (Step by Step) (Offical guide) [https://docs.docker.com/engine/install/ubuntu/]

I'll teach you from beginner level. By the end, you'll have Docker Engine, Docker CLI, and Docker Compose installed and be able to run your first container.

## Step 1 — Update package Ubuntu

```bash
sudo apt update
```
Note: If you previously installed docker.io or another Docker package, you may need to remove conflicting packages first. See the official installation guide linked below.

## Step 2: Install required packages

```bash
sudo apt install ca-certificates curl gnupg
```

Create a directory to store Docker's official GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download the Docker GPG key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

Set the correct permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
This allows Ubuntu to verify packages from Docker's official repository.

## Step 3: Add Docker's official repository

Copy and paste this entire block into your terminal:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Then update your package list:

```bash
sudo apt update
```

## Step 4: Install Docker Engine

Install Docker and Docker Compose:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

These packages provide Docker Engine, the Docker command-line tool, container runtime, Buildx, and Docker Compose.

Check the Docker version:
```bash
docker --version
```

Check Docker Compose:
```bash
docker compose version
```   

## Step 5: Start Docker