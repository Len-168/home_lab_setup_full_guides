# 1. Install SSH Server

SSH (Secure Shell) lets you log in to the server and run commands from your own computer, so you don't need a monitor and keyboard attached to it. Ubuntu uses **OpenSSH** for this. The `openssh-server` package runs a background service (`ssh`) that listens on **port 22** for incoming connections.

**Before you start:**
- Run these steps on the server itself, through its console or a monitor and keyboard.
- You should already have a sudo user (see [1 - initialize server](../1%20-%20initialize%20server/1-Security%20&%20System%20Fundamentals.md), step 1.3).

---

1.1. Switch to your sudo user

Don't do daily work as `root`. Switch to the sudo user you created earlier:

```bash
su - your_username

# Confirm who you are and that you have sudo rights
whoami          # should print: your_username
sudo -v         # asks for your password; no error = you have sudo
```

1.2. Update the package list

This makes sure `apt` installs the latest version of OpenSSH:

```bash
sudo apt update
```

1.3. Install the SSH server

```bash
sudo apt install -y openssh-server
```

> Ubuntu Server may already include OpenSSH if you ticked "Install OpenSSH server" during setup. Running the command again does no harm. It reports that the package is already the newest version.

1.4. Enable and start the SSH service

`enable` starts SSH automatically on every boot. `--now` also starts it right away:

```bash
sudo systemctl enable --now ssh
```

1.5. Verify the SSH service

```bash
c
```

You should see `Active: active (running)` in green. Press `q` to exit.

Check that SSH is listening on port 22:

```bash
sudo ss -tlnp | grep ssh
# Expected: LISTEN ... 0.0.0.0:22 ... users:(("sshd",...))
```

> On Ubuntu 22.10 and later, SSH can be started by `ssh.socket` (socket activation). If `ssh.service` shows `inactive` but `sudo systemctl status ssh.socket` shows `active (listening)`, SSH still works. It starts as soon as someone connects.

1.6. Allow SSH through the firewall (UFW)

If UFW is enabled, or you plan to enable it, allow SSH **first**. Otherwise you lock yourself out of remote access.

```bash
sudo ufw allow OpenSSH
sudo ufw enable        # only if UFW is not enabled yet
sudo ufw status        # should list: OpenSSH  ALLOW  Anywhere
```

1.7. Find the server's IP address

You need this to connect from your local machine:

```bash
hostname -I            # e.g. 192.168.1.50
# or
ip addr                # look for: inet 192.168.x.xxx/24 (not 127.0.0.1)
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Unit ssh.service could not be found` | `openssh-server` isn't installed. Run step 1.3 again. |
| Status shows `failed` | Check the config with `sudo sshd -t`, then read the logs with `sudo journalctl -u ssh -n 50`. |
| `Connection refused` from the client | The service isn't running (step 1.4) or the firewall blocks port 22 (step 1.6). |
| `Connection timed out` | Wrong IP address, or the client and server are on different networks. |

## Next step

➡️ [2-setup-ssh-password.md](2-setup-ssh-password.md): connect from your local machine using a password.
