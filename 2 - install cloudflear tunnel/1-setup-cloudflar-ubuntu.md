# Install cloudflared on Ubuntu (Step by Step)

Based on Cloudflare's official guides: [Cloudflare package repository](https://pkg.cloudflare.com/index.html) and [Downloads – cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/).

`cloudflared` is the small program that connects your server to Cloudflare. It opens an **outbound** connection to Cloudflare, so you can reach the server (SSH, web apps, etc.) **without opening any ports** on your router or firewall.

By the end of this guide, `cloudflared` will be installed from Cloudflare's official repository and will get updates with normal `apt upgrade`.

## Why install it on the host (not in Docker)?

Run `cloudflared` **directly on Ubuntu as a systemd service**, not inside a Docker container.

- Remote access to the server does not depend on Docker being healthy.
- If Docker breaks, stops, or gets reinstalled, you can still SSH in through the tunnel and fix it.
- systemd starts it automatically at boot and restarts it if it crashes.

## Before you start

Log in to the server as your sudo user (not root), and make sure `curl` is installed:

```bash
sudo apt update
sudo apt install -y curl
```

## Step 1: Add Cloudflare's GPG key

Create the folder that stores repository keys (it usually already exists):

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
```

Download Cloudflare's package signing key:

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
```

This key lets `apt` check that the packages really come from Cloudflare.

## Step 2: Add the cloudflared repository

```bash
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
```

`any` works for every supported Ubuntu release, so you don't need to change it when you upgrade Ubuntu.

## Step 3: Install cloudflared

```bash
sudo apt update
sudo apt install -y cloudflared
```

## Step 4: Verify the installation

```bash
cloudflared --version
```

You should see something like:

```text
cloudflared version 2026.x.x (built ...)
```

## Keeping cloudflared up to date

Because it was installed from Cloudflare's repository, normal system updates also update `cloudflared`:

```bash
sudo apt update && sudo apt upgrade -y
```

If `cloudflared` is already running as a service, restart it after an update:

```bash
sudo systemctl restart cloudflared
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `NO_PUBKEY` or `The following signatures couldn't be verified` | Run Step 1 again to download the key, then `sudo apt update`. |
| `Unable to locate package cloudflared` | Check the repository file: `cat /etc/apt/sources.list.d/cloudflared.list`, then run `sudo apt update`. |
| `cloudflared: command not found` after install | Check with `which cloudflared` (should be `/usr/bin/cloudflared`). Log out and back in. |

## Uninstall (if needed)

```bash
sudo systemctl stop cloudflared 2>/dev/null
sudo cloudflared service uninstall 2>/dev/null
sudo apt remove -y cloudflared
sudo rm /etc/apt/sources.list.d/cloudflared.list /usr/share/keyrings/cloudflare-main.gpg
```

## Next step

`cloudflared` is installed, but it is not connected to anything yet. Continue with [2-setup-tunnel.md](2-setup-tunnel.md) to create the tunnel and run it as a systemd service.
