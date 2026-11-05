
## 4. Add the SSH route tunnel

4.1. Add Hostname
```text
    # Add your Subdomain
    # Add your Domain
```
4.2. Service
```text
    # Choose Type : SSH
    # Set URL ( ssh://localhost:22)
```

## 5. Install cloudflared on your remote laptop

5.1. On macOS:
```bash
brew install cloudflared
```
5.2. Verify:

```bash
cloudflared --version
```

5.3. Configure the SSH client

```text
Host dev-ssh
    HostName ssh-dev1.mini-homelab.site
    User deploy
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
    ServerAliveInterval 60
    ServerAliveCountMax 10
```
