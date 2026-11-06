# Publish the Nginx Proxy Manager Admin Panel Through Cloudflare Tunnel

- **Date:** 2026-11-05 (last change in git), restructured 2026-11-06
- **Author:** Siem Polen
- **Version:** [UNCLEAR: no version number is tracked. Git history is the only record.]
- **Related files:**
  - [1-setup-nginx.md](1-setup-nginx.md): install Nginx Proxy Manager
  - [2-setup-tunnel.md](../2%20-%20install%20cloudflear%20tunnel/2-setup-tunnel.md): create the Cloudflare Tunnel
  - [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md): protect hostnames with Cloudflare Access
  - [4-publish-ssh-tunnel.md](../3%20-%20ssh/4-publish-ssh-tunnel.md): the same process, used for SSH

## Summary

This guide makes the Nginx Proxy Manager (NPM) admin panel reachable from anywhere, at an address like `https://npm.YOUR-DOMAIN.com`.

It works by adding a route to your existing Cloudflare Tunnel. The route points to the admin panel on port 81 of the server.

You do not need to open any port on your router or firewall.

The process is the same as publishing SSH. The only difference is the service type (HTTP) and the port (81).

**Status:** the steps are written. [UNCLEAR: it is not clear whether this hostname is meant to be protected by a Cloudflare Access application. See "Before you start".]

## Before you start

- Nginx Proxy Manager is running. You can open `http://<your-server-ip>:81` and you have created the admin account: [1-setup-nginx.md](1-setup-nginx.md)
- The tunnel is **Healthy**: [2-setup-tunnel.md](../2%20-%20install%20cloudflear%20tunnel/2-setup-tunnel.md)
- [UNCLEAR: Access application for this hostname.] The original notes say to "use exactly the same hostname as in your Access application". But [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md) only creates an Access application for `ssh.YOUR-DOMAIN.com`. No step creates one for `npm.YOUR-DOMAIN.com`. Also, [zero-trust.md](../2%20-%20install%20cloudflear%20tunnel/zero-trust.md) says to set up Access **before** publishing any hostname. Without it, the NPM login page is open to anyone on the internet. [REASON MISSING: if leaving this hostname without Access is intentional, write down why.]

## How it works

```text
Your browser
   ↓  https://npm.YOUR-DOMAIN.com
Cloudflare  ← handles the SSL certificate
   ↓
Tunnel  ←── outbound connection ──  cloudflared service on the Ubuntu server
                                          ↓
                                     http://localhost:81
                                          ↓
                                     NPM admin panel (login page)
```

Why it is set up this way:

- **No firewall changes.** The `cloudflared` service on the server connects **out** to Cloudflare. Traffic comes back through that connection, so no port has to be opened. (Same as [4-publish-ssh-tunnel.md](../3%20-%20ssh/4-publish-ssh-tunnel.md).)
- **`localhost`.** `cloudflared` runs on the same server as NPM, so it forwards traffic to its own port 81.
- **Port 81, not 80 or 443.** Port 81 is the NPM admin web port. Ports 80 and 443 are the public ports NPM uses for the sites it proxies. (From the `docker-compose.yml` in [1-setup-nginx.md](1-setup-nginx.md).)
- **Service type `http`, not `https`.** The admin panel is served over plain HTTP on port 81 (the setup guide opens it with `http://`). Cloudflare provides HTTPS to visitors with its own certificate.

## Steps

### 1. Open your tunnel's routes

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

### 2. Enter the hostname

```text
Subdomain:  npm   (or nginx-proxy-manager, or any name you want)
Domain:     YOUR-DOMAIN.com
Path:       (leave empty)
```

This guide uses `npm` in the examples below.

### 3. Point the route to the NPM admin panel

```text
Service Type:  http
URL:           localhost:81
```

### 4. Save

Click **Save application**.

Cloudflare then does these things automatically:

- Creates the public hostname.
- Generates the SSL certificate.
- Routes traffic through your tunnel.

### 5. Open the admin panel

In your browser, go to:

```text
https://npm.YOUR-DOMAIN.com
```

Use the hostname you chose in Step 2. You should see the Nginx Proxy Manager login page.

### 6. Log in

[UNCLEAR: which login to use.] The original notes say: "Since you already created an Access application for SSH, you can log in with the same credentials." But:

- The NPM login page uses the NPM admin account you created in [1-setup-nginx.md](1-setup-nginx.md) Step 4. That is not a Cloudflare login.
- A Cloudflare Access login page only appears if an Access application covers this hostname. The SSH application covers only `ssh.YOUR-DOMAIN.com`.

