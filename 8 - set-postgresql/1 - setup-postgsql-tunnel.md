# Publish PostgreSQL Through Cloudflare Tunnel

Based on Cloudflare's official guide: [Connect to arbitrary TCP with client-side cloudflared](https://developers.cloudflare.com/cloudflare-one/applications/non-http/cloudflared-authentication/arbitrary-tcp/).

This guide runs a PostgreSQL container for your apps and lets you reach it from your Mac through the Cloudflare Tunnel, with no open ports. It works the same way as [Publish SSH via Tunnel](../3%20-%20ssh/4-publish-ssh-tunnel.md): `cloudflared` runs on both the server and the client.

## Why not Nginx Proxy Manager?

| NPM option | Why it doesn't fit |
|---|---|
| **Proxy Hosts** | Handle HTTP/HTTPS only. PostgreSQL speaks its own TCP protocol. |
| **Streams** | Forward raw TCP, but only through a published host port like `5432:5432`. Docker bypasses UFW ([Docker guide, Step 9](../4%20-%20install%20docker/docker.md)), so the database would be open to your whole LAN. |

Cloudflare needs `cloudflared` on the client for raw TCP anyway, so NPM in the middle adds nothing. The tunnel connects straight to PostgreSQL.

> [!IMPORTANT]
> Don't reuse the `db` container from NPM's compose file. That is NPM's own internal database. Run a separate PostgreSQL for your apps, as below.

## How it works

```text
Your Mac                                  Cloudflare                     Ubuntu server
────────                                  ──────────                     ─────────────
psql / DBeaver → localhost:15432
   ↓
cloudflared access tcp ── HTTPS ──→  Access login (email + MFA)
                                          ↓ allowed?
                                     Tunnel ←── outbound ──  cloudflared service
                                                                  ↓
                                                             127.0.0.1:5432
                                                                  ↓
                                                             postgres container
                                                             (checks DB password)
```

A visitor has to pass **two locks**: the Cloudflare Access login, then the database password.

## Before you start

- [ ] The tunnel is **Healthy**: [2-setup-tunnel.md](../2%20-%20install%20cloudflear%20tunnel/2-setup-tunnel.md)
- [ ] An Access application for `db.YOUR-DOMAIN.com` exists: [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md)
  > **Do this first.** Without Access, anyone with `cloudflared` can reach your database login.
- [ ] Docker and Compose are installed: [docker.md](../4%20-%20install%20docker/docker.md)

> Replace `db.YOUR-DOMAIN.com` with your own hostname.

---

# Part 1: Server side

## Step 1: Run PostgreSQL bound to localhost

```bash
mkdir -p ~/postgres && cd ~/postgres
nano compose.yaml
```

```yaml
services:
  db:
    image: postgres:17
    container_name: app-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: change-me-to-a-long-random-password
      POSTGRES_DB: appdb
    ports:
      - "127.0.0.1:5432:5432"   # only the host (cloudflared) can reach it
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
```

> Generate a strong password with `openssl rand -base64 32`.

Start it and check the port binding:

```bash
docker compose up -d
docker ps --format 'table {{.Names}}\t{{.Ports}}'
# Must show 127.0.0.1:5432->5432/tcp, NOT 0.0.0.0:5432
```

## Step 2: Publish the hostname in the tunnel

```text
Cloudflare Dashboard → Networking → Tunnels → your tunnel
   ↓
Published application routes → Add a published application route
```

```text
Subdomain:     db
Domain:        YOUR-DOMAIN.com
Path:          (leave empty)
Service Type:  TCP
URL:           localhost:5432
```

Use **exactly the same** hostname as in your Access application. Click **Save**; Cloudflare creates the DNS record automatically.

---

# Part 2: Client side (your Mac)

## Step 3: Install cloudflared

```bash
brew install cloudflared
```

## Step 4: Open a local tunnel to the database

```bash
cloudflared access tcp --hostname db.YOUR-DOMAIN.com --url localhost:15432
```

Leave this running. The first connection opens the Cloudflare Access login in your browser.

> Port `15432` avoids clashing with a PostgreSQL you may run locally on `5432`.

## Step 5: Connect

In a second terminal:

```bash
psql -h localhost -p 15432 -U appuser appdb
```

In DBeaver or another GUI client, use host `localhost`, port `15432`, database `appdb`, user `appuser`.

## Step 6: Check the security

```bash
# From another machine on your LAN: the port must NOT be reachable
nc -zv SERVER_LAN_IP 5432
# Expected: Connection refused
```

Also check **Zero Trust → Logs → Access**: your login should be listed.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `websocket: bad handshake` | The hostname isn't published in the tunnel, or it's spelled differently from the Access application. Check Step 2. |
| Connects with no Cloudflare login | The hostname isn't protected by Access. Fix this right away: [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md). |
| `connection refused` on `localhost:15432` | The `cloudflared access tcp` command from Step 4 isn't running. |
| `password authentication failed` | The tunnel works; check the user and password in `compose.yaml`. Changing `POSTGRES_PASSWORD` later has no effect once `postgres_data` exists. |
| Port shows `0.0.0.0:5432` | The `127.0.0.1:` prefix is missing in `ports:`. Fix it and run `docker compose up -d`. |
