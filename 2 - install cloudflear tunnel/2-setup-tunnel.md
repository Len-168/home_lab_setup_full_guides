# Create a Cloudflare Tunnel on Ubuntu (Step by Step)

Based on Cloudflare's official guide: [Create a remotely-managed tunnel (dashboard)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/).

In this guide you create a tunnel in the Cloudflare dashboard and connect your Ubuntu server to it. `cloudflared` runs as a **systemd service**, so it starts at boot and reconnects on its own.

This tunnel is **remotely managed**: its settings (hostnames, routes) live in the Cloudflare dashboard, not in a config file on the server.

## Before you start

- `cloudflared` is installed on the server. If not, follow [1-setup-cloudflar-ubuntu.md](1-setup-cloudflar-ubuntu.md).
- Your domain is added to Cloudflare and uses Cloudflare's nameservers.
- You are logged in to the server as your sudo user.

## Step 1: Create the tunnel in the dashboard

In the Cloudflare dashboard, go to:

```text
Cloudflare Dashboard
   ↓
Networking
   ↓
Tunnels
   ↓
Create Tunnel
   ↓
Select "Cloudflared" as the connector type
```

> In the Zero Trust dashboard, the same page is under **Networks → Connectors → Cloudflare Tunnels → Create a tunnel**.

## Step 2: Name the tunnel

Use a name that tells you which machine it is, for example:

```text
ubuntu-home-lab
```

Click **Save tunnel**.

## Step 3: Install the tunnel as a service

Cloudflare now shows install commands. Choose:

```text
Operating system:  Debian
Architecture:      64-bit
```

`cloudflared` is already installed, so **skip the install part** and only copy the last command. It looks like this:

```bash
sudo cloudflared service install eyJhIjoi...
```

Run it on the server.

> **The token is a password.** Anyone who has it can run your tunnel. Don't commit it to git, paste it in chat, or keep it in shell history on a shared machine.

## Step 4: Check that the service is running

```bash
sudo systemctl status cloudflared
```

You should see:

```text
Active: active (running)
```

Make sure it starts on boot:

```bash
sudo systemctl enable cloudflared
```

Back in the dashboard, the tunnel status should be **Healthy**. Click **Next**, but **do not add a public hostname yet**. You will add the SSH route after Cloudflare Access is set up.

## Useful commands

| Task | Command |
|---|---|
| Show service status | `sudo systemctl status cloudflared` |
| Follow the live logs | `sudo journalctl -u cloudflared -f` |
| Restart the tunnel | `sudo systemctl restart cloudflared` |
| Stop the tunnel | `sudo systemctl stop cloudflared` |

## Troubleshooting

| Problem | Fix |
|---|---|
| `Active: failed` | Read the logs: `sudo journalctl -u cloudflared -n 50`. A bad or incomplete token is the usual cause. |
| `cloudflared service is already installed` | Remove the old one first: `sudo cloudflared service uninstall`, then run Step 3 again. |
| Dashboard shows **Inactive** or **Down** | Check the server's internet connection, and that outbound port **7844** (TCP/UDP) is not blocked. |
| Token was leaked | Dashboard → your tunnel → **Refresh token**, then run `sudo cloudflared service uninstall` and Step 3 again with the new token. |

## Next step

The tunnel is connected, but nothing is published through it yet.

1. Protect your hostnames with Cloudflare Access: [zero-trust.md](zero-trust.md)
2. Publish SSH through the tunnel: [4-publish-ssh-tunnel.md](../3%20-%20ssh/4-publish-ssh-tunnel.md)
