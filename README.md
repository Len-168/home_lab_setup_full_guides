# Ubuntu Server Setup

Step-by-step guides for turning a fresh Ubuntu server into a secure, self-hosted home server: user and system basics, Cloudflare Tunnel with Zero Trust, key-only SSH reachable from anywhere, Docker, and Nginx Proxy Manager.

Follow the sections in order. Each guide ends with a **Next step** link where it applies.

## Table of Contents

| # | Section | Guide | What you'll do |
|---|---------|-------|----------------|
| 1 | **Initialize Server** | [Security & System Fundamentals](1%20-%20initialize%20server/1-Security%20%26%20System%20Fundamentals.md) | Create a sudo user and set up the system basics |
| 2 | **Cloudflare Tunnel** | [Install cloudflared](2%20-%20install%20cloudflear%20tunnel/1-setup-cloudflar-ubuntu.md) | Add Cloudflare's repository and install `cloudflared` on the host |
|   |  | [Create a Tunnel](2%20-%20install%20cloudflear%20tunnel/2-setup-tunnel.md) | Create a tunnel in the dashboard and run it as a service |
|   |  | [Zero Trust (Access)](2%20-%20install%20cloudflear%20tunnel/zero-trust.md) | Put a login (identity provider + Allow policy) in front of your services |
| 3 | **SSH** | [Install SSH Server](3%20-%20ssh/1-init-ssh-new-server.md) | Install and verify OpenSSH on the new server |
|   |  | [Password Login](3%20-%20ssh/2-setup-ssh-password.md) | Connect from your local machine with a password |
|   |  | [SSH Key Login](3%20-%20ssh/3-setup-ssh-key.md) | Generate a key on macOS, copy it over, and disable password login |
|   |  | [Publish SSH via Tunnel](3%20-%20ssh/4-publish-ssh-tunnel.md) | Reach SSH from anywhere through Cloudflare, with no open ports |
| 4 | **Docker** | [Install Docker](4%20-%20install%20docker/docker.md) | Install Docker Engine, run without sudo, limit logs, keep ports private |
| 5 | **Nginx Proxy Manager** | [Set up Nginx Proxy Manager](5%20-%20nginx%20proxcy%20manager/1-setup-nginx.md) | Run NPM with Docker Compose and open the admin panel |

## Setup Flow

```text
1. Initialize Server ─► 2. Cloudflare Tunnel ─► 3. SSH ─► 4. Docker ─► 5. Nginx Proxy Manager
   (sudo user)            (cloudflared, tunnel,     (key-only,   (engine,     (reverse proxy
                           Zero Trust)               via tunnel)  compose)     + admin UI)
```

## Requirements

- A machine running Ubuntu Server with internet access
- A domain managed in Cloudflare (for the Tunnel and Zero Trust sections)
- A local machine to connect from (the SSH guides use macOS)
