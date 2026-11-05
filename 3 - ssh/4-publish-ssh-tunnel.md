# Publish SSH Through Cloudflare Tunnel (Access From Anywhere)

Based on Cloudflare's official guide: [Connect to SSH with client-side cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/ssh-cloudflared-authentication/).

So far you can only SSH into the server from your home network (`ssh your_username@192.168.x.x`). This guide lets you reach it **from anywhere on the internet**, without opening any port on your router.

## How it works

There are two sides:

| Side | What you do | Where |
|---|---|---|
| **Server side (public access)** | Publish a hostname like `ssh.YOUR-DOMAIN.com` in the tunnel and point it to SSH on `localhost:22` | Cloudflare dashboard |
| **Client side** | Install `cloudflared` on your Mac and tell `ssh` to connect through it | Your Mac |

```text
Your Mac                                   Cloudflare                         Ubuntu server
─────────                                  ──────────                         ─────────────
ssh dev-ssh
   ↓
cloudflared access ssh  ── HTTPS ──→  Access login (email + MFA)
                                           ↓ allowed?
                                      Tunnel  ←── outbound connection ──  cloudflared service
                                                                               ↓
                                                                          sshd on localhost:22
                                                                               ↓
                                                                          checks your SSH key
                                                                               ↓
                                                                          Ubuntu shell ✅
```

- The server **never opens a port**. Its `cloudflared` service connects **out** to Cloudflare, and traffic comes back through that connection.
- Plain `ssh` can't talk to a Cloudflare hostname directly. On your Mac, `cloudflared` wraps SSH traffic in HTTPS, and that's why the client needs it too.
- A visitor has to pass **two locks**: the Cloudflare Access login, then your SSH key.

## Before you start

- [ ] The tunnel is **Healthy**: [2-setup-tunnel.md](../2%20-%20install%20cloudflear%20tunnel/2-setup-tunnel.md)
- [ ] The Access application for `ssh.YOUR-DOMAIN.com` exists: [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md)
  > **Do this first.** Without Access, anyone on the internet can reach your SSH login.
- [ ] Key login works and password login is off: [3-setup-ssh-key.md](3-setup-ssh-key.md)

> Replace `ssh.YOUR-DOMAIN.com` with your own hostname, and `your_username` with your Ubuntu user.

---

# Part 1: Server side, publishing SSH

## Step 1: Open your tunnel

```text
Cloudflare Dashboard
   ↓
Networking
   ↓
Tunnels
   ↓
Click your tunnel (e.g. ubuntu-home-lab)
   ↓
Published application routes (Public Hostnames)
   ↓
Add a published application route
```

## Step 2: Add the hostname

Use **exactly the same** hostname as in your Access application:

```text
Subdomain:  ssh
Domain:     YOUR-DOMAIN.com
Path:       (leave empty)
```

## Step 3: Point it to SSH

```text
Service Type:  SSH
URL:           localhost:22
```

`localhost:22` means "SSH on this same server". `cloudflared` runs on the Ubuntu server, so it forwards traffic to its own port 22.

Click **Save**. Cloudflare creates the DNS record (a CNAME to your tunnel) automatically.

## Step 4: Check it on the server

```bash
# SSH is listening on port 22
sudo ss -tlnp | grep ssh

# The tunnel is running
sudo systemctl status cloudflared
```

The server side is done. Next, set up your Mac.

---

# Part 2: Client side, connecting from your Mac

## Step 5: Install cloudflared

```bash
brew install cloudflared
```

Check it and note the path; you need it in Step 6:

```bash
cloudflared --version
which cloudflared
# Apple Silicon (M1/M2/M3...): /opt/homebrew/bin/cloudflared
# Intel Mac:                   /usr/local/bin/cloudflared
```

## Step 6: Configure the SSH client

Edit `~/.ssh/config` on your Mac:

```bash
nano ~/.ssh/config
```

Add:

```text
Host dev-ssh
    HostName ssh.YOUR-DOMAIN.com
    User your_username
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
    ServerAliveInterval 60
    ServerAliveCountMax 10
```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

What each line does:

| Line | Meaning |
|---|---|
| `Host dev-ssh` | A short name. You type `ssh dev-ssh` instead of the full command. |
| `HostName` | The public hostname you published in Step 2. |
| `User` | Your Ubuntu username on the server. |
| `IdentityFile` | The private key from [3-setup-ssh-key.md](3-setup-ssh-key.md). |
| `IdentitiesOnly yes` | Offer only this key, not every key on your Mac. This avoids "Too many authentication failures". |
| `ProxyCommand` | Sends SSH through `cloudflared` instead of a direct connection. `%h` is replaced with `HostName`. **Use the path from `which cloudflared`.** |
| `ServerAliveInterval 60` | Sends a keep-alive every 60 seconds, so idle sessions don't drop. |
| `ServerAliveCountMax 10` | Disconnects after 10 missed keep-alives (about 10 minutes). |

## Step 7: Connect

```bash
ssh dev-ssh
```

The first time:

1. Your browser opens the **Cloudflare Access** login page.
2. Log in (email code, or your login with MFA).
3. The browser says you can close the tab. Back in Terminal, the SSH session opens.

Until the Access session expires (the **Session duration** you set in [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md)), `ssh dev-ssh` connects straight away without the browser.

> **Without `~/.ssh/config`:** for a one-off connection, run
> ```bash
> ssh -o ProxyCommand="cloudflared access ssh --hostname %h" your_username@ssh.YOUR-DOMAIN.com
> ```

## Step 8: Check the security

Each lock should block you on its own:

```bash
# 1. Password login must be refused (key only)
ssh -o PubkeyAuthentication=no dev-ssh
# Expected: Permission denied (publickey).

# 2. Plain SSH without cloudflared must NOT connect (no open port)
ssh your_username@ssh.YOUR-DOMAIN.com
# Expected: hangs or "Connection refused/timed out"
```

Also check **Zero Trust → Logs → Access**: your login should be listed.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `ProxyCommand ... No such file or directory` | Wrong `cloudflared` path in `~/.ssh/config`. Use the output of `which cloudflared`. |
| `websocket: bad handshake` | The hostname isn't published in the tunnel, or it's spelled differently from the Access application. Check Step 2. |
| Browser never opens / login loops | Run `cloudflared access login ssh.YOUR-DOMAIN.com` to log in by hand, then try `ssh dev-ssh` again. |
| Connects with no Cloudflare login | The hostname isn't protected by Access. Fix this right away: [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md). |
| `Connection closed by UNKNOWN port 65535` | The tunnel reached the server but SSH didn't answer. Check that the Service Type is **SSH** with `localhost:22`, and run `sudo systemctl status ssh` on the server. |
| `Permission denied (publickey)` | The tunnel works; the key is the problem. Check `IdentityFile` and `User`, and see [3-setup-ssh-key.md](3-setup-ssh-key.md). |
| Tunnel shows **Down** in the dashboard | On the server, run `sudo systemctl restart cloudflared` and check `sudo journalctl -u cloudflared -n 50`. |
