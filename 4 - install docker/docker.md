# Install Docker on Ubuntu (Step by Step)

Based on Docker's official guides: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/) and [Linux post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/).

This guide is for beginners. By the end, you will have Docker Engine, the Docker CLI and Docker Compose installed, set up safely for a server, and tested with a real container.

## Before you start

Log in to the server as your sudo user (not root), then check your Ubuntu version:

```bash
lsb_release -a
```

Docker only supports certain Ubuntu releases. Compare yours with the [supported versions](https://docs.docker.com/engine/install/ubuntu/#os-requirements). For a server, use an LTS release such as 24.04 or 26.04.

## Step 1: Remove old or conflicting packages

Ubuntu's own repositories have unofficial Docker packages (such as `docker.io`) that conflict with the official ones. This command removes any that are installed:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

On a fresh server you will see `dpkg: no packages found matching ...`. That is fine: there was nothing to remove.

If you selected **docker** on the "Featured server snaps" screen while installing Ubuntu Server, you also have the snap version of Docker. Check, and remove it if it is there:

```bash
snap list docker                  # "error: no matching snaps installed" means you don't have it
sudo snap remove --purge docker   # only if the command above listed docker
```

## Step 2: Install required packages and Docker's GPG key

Update the package list and install the tools needed to download Docker's key:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
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

This key lets Ubuntu check that the packages really come from Docker.

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

`tee` prints what it wrote. Check that `Suites:` shows your Ubuntu codename (for example `noble`) and `Architectures:` shows your CPU type (for example `amd64`). Neither should be empty.

Then update your package list:

```bash
sudo apt update
```

Check that apt can now find Docker's packages:

```bash
apt-cache policy docker-ce
```

You should see a line with `download.docker.com` (your version numbers and codename may be different):

```text
docker-ce:
  Installed: (none)
  Candidate: 5:<version>~ubuntu.24.04~noble
  ...
        500 https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
```

If you don't see `download.docker.com`, the repository was not added correctly. Repeat this step.

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

These packages provide Docker Engine, the Docker command-line tool, the container runtime, Buildx and Docker Compose.

Check the Docker version:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Both commands should print a version number.

Note: Compose is now a plugin, so the command is `docker compose` (with a space). Older tutorials use `docker-compose` (with a hyphen); type `docker compose` instead.

## Step 5: Check that Docker is running

Docker starts automatically after installation. Check its status:

```bash
sudo systemctl status docker
```

You should see `Active: active (running)` in green. Press `q` to exit.

Check that Docker also starts when the server reboots:

```bash
systemctl is-enabled docker   # should print: enabled
```

If Docker is not running or not enabled:

```bash
sudo systemctl enable --now docker containerd
```

## Step 6: Run your first container

```bash
sudo docker run hello-world
```

Docker downloads a small test image, starts a container from it, and the container prints a message and exits. You should see:

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

## Step 7: Use Docker without sudo

Add your user to the `docker` group:

```bash
# 1. Create the docker group (may already exist — that's fine)
sudo groupadd docker

# 2. Add yourself to it
sudo usermod -aG docker $USER

# 3. Apply the change: log out and back in (or reboot if in a VM).
#    For the current terminal only, you can use:
newgrp docker

# 4. Test
docker run hello-world
```

Group changes only apply when you log in again. Log out (`exit`) and SSH back in, then check:

```bash
groups                    # the list should include: docker
docker run hello-world    # works without sudo now
```

> [!WARNING]
> Being in the `docker` group is the same as having root access: a member can start a container that mounts the server's whole filesystem, with no sudo password. Only add users you would also give sudo to. If you prefer to keep the password prompt, skip this step and put `sudo` in front of the `docker` commands in the rest of this guide.

## Step 8: Limit container log size

By default, Docker keeps every line a container logs, forever. On a server, one noisy container can slowly fill the whole disk. Set a size limit for all containers.

Create Docker's settings file:

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

Paste this (if the file already has content, merge these settings into it):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Save and exit (`Ctrl + O, Enter, Ctrl + X`). Each container now keeps at most 3 log files of 10 MB each. Keep the quotes around `"3"`: Docker won't start if `log-opts` values are written without quotes.

Check the file before restarting Docker (like `sshd -t` for SSH):

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
```

It should print `configuration OK`. Then restart Docker:

```bash
sudo systemctl restart docker
```

This only applies to containers created from now on, so it is best done now, before you run real apps. To apply it to a Compose app that already exists, recreate it with `docker compose up -d --force-recreate`.

## Step 9: Keep container ports private (Docker bypasses UFW)

> [!WARNING]
> When you publish a port (`-p 8080:80`, or `ports:` in Compose), Docker adds its own firewall rules that send traffic straight to the container, skipping UFW. So `sudo ufw deny 8080` does **not** protect that port: any device that can reach your server can reach the container.

Only publish a port to the network when other machines really need it. Otherwise, bind it to `127.0.0.1` so only the server itself can reach it:

| Port setting | Who can reach the container |
| --- | --- |
| `-p 8080:80` | Any device that can reach the server (UFW won't stop it) |
| `-p 127.0.0.1:8080:80` | Only programs running on the server itself |

- **Cloudflare Tunnel:** if `cloudflared` runs on the host (as in the SSH tunnel guide), it can still reach `http://localhost:8080`. Apps behind the tunnel don't need a public port.
- **Container to container** (for example Nginx Proxy Manager → your app): no published port is needed at all. Put both containers on the same Docker network and use the container name, like `http://my-app:80`.

To see which ports are exposed at any time:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

`0.0.0.0:8080->80/tcp` or `[::]:8080->80/tcp` means open to the network. `127.0.0.1:8080->80/tcp` means private.

## Step 10: Test everything with Docker Compose

Most self-hosted apps, including Nginx Proxy Manager, run with Docker Compose. Try it with a test web server.

Create a test folder and a Compose file:

```bash
mkdir -p ~/docker-test && cd ~/docker-test
nano compose.yaml
```

Paste:

```yaml
services:
  web:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"
```

- `restart: unless-stopped` starts the container again after a reboot or a Docker update, unless you stopped it yourself.
- `127.0.0.1:8080:80` keeps the site private to the server (Step 9).

Start it in the background:

```bash
docker compose up -d
docker compose ps    # STATUS should say "Up"
```

Test from the server:

```bash
curl -I http://localhost:8080    # should show: HTTP/1.1 200 OK
```

Test from your laptop: open `http://your_server_ip:8080` in a browser. It should **not** load, because the port is private, as intended.

Check that the log limit from Step 8 is applied:

```bash
docker inspect --format '{{.HostConfig.LogConfig}}' docker-test-web-1
```

```text
{json-file map[max-file:3 max-size:10m]}
```

Clean up:

```bash
docker compose down
cd ~ && rm -rf ~/docker-test
```

Docker is now installed and ready for your apps.

## Updating Docker

Docker updates come through apt, like any other package:

```bash
sudo apt update && sudo apt upgrade -y
```

- Ubuntu's automatic security updates don't include Docker's repository by default, so Docker only updates when you run this yourself.
- Updating Docker restarts the Docker service, which briefly stops all containers. Containers with `restart: unless-stopped` start again by themselves; containers without a restart policy stay stopped until you start them.

## Troubleshooting

**`permission denied while trying to connect to the Docker daemon socket`**

You are not in the `docker` group yet, or you haven't logged out and back in since Step 7. Run `groups` to check, or use `sudo docker ...`.

**Docker won't start after editing `daemon.json`**

Usually a typo in the JSON (a missing or extra comma, or a `log-opts` value without quotes). Find the problem with:

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
sudo journalctl -u docker -n 50 --no-pager
```

**`apt update` says `Conflicting values set for option Signed-By`**

An old Docker repository file from a previous install is still there, usually `docker.list`. Find it and remove it:

```bash
ls /etc/apt/sources.list.d/
sudo rm /etc/apt/sources.list.d/docker.list
```

**`apt update` says the Docker repository `does not have a Release file`**

Docker doesn't support your Ubuntu version (yet). Check the [supported versions](https://docs.docker.com/engine/install/ubuntu/#os-requirements).

## Uninstall Docker (start over)

> [!CAUTION]
> This deletes **all** images, containers and volumes, including any app data stored in Docker volumes.

```bash
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker /var/lib/containerd
sudo rm /etc/apt/sources.list.d/docker.sources /etc/apt/keyrings/docker.asc
```
