# Setup SSH Key Login (macOS → Ubuntu Server)

An SSH key lets you log in without typing the Ubuntu password. You create a **key pair** on your Mac:

- **Private key** (`~/.ssh/id_ed25519`): stays on your Mac. Never copy or share it.
- **Public key** (`~/.ssh/id_ed25519.pub`): goes on the server, in `~/.ssh/authorized_keys`.

When you connect, the server checks that your Mac holds the private key that matches the public key. Once that works, you turn off password login, so nobody can guess or brute-force their way in.

**Before you start:**
- SSH is installed and running on the server ([1-init-ssh-new-server.md](1-init-ssh-new-server.md)).
- You can log in with a password ([2-setup-ssh-password.md](2-setup-ssh-password.md)).

> In the commands below, replace `your_username` with your Ubuntu user and `your_server_ip` with the server's IP address.

---

# 1. Generate an SSH key on your Mac

Open **Terminal** on your Mac:

```bash
ssh-keygen -t ed25519 -C "your_mac_name"
```

`-C` adds a label so you can tell which machine a key belongs to on the server.

You will see:

```text
Enter file in which to save the key (/Users/you/.ssh/id_ed25519):   ← press Enter
Enter passphrase (empty for no passphrase):                         ← type a passphrase (recommended)
Enter same passphrase again:
```

> **Passphrase:** it protects the key if someone gets hold of your Mac. macOS Keychain can remember it (step 2), so you won't have to type it every time. To skip it, just press Enter twice.

Check the two files:

```bash
ls -l ~/.ssh/id_ed25519*
# ~/.ssh/id_ed25519      ← private key (keep secret)
# ~/.ssh/id_ed25519.pub  ← public key (copy to server)
```

# 2. Save the passphrase in macOS Keychain

Skip this step if you didn't set a passphrase.

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Then create or edit `~/.ssh/config` so macOS loads the key automatically:

```bash
nano ~/.ssh/config
```

```text
Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519
```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

# 3. Copy the public key to the server

## Method 1: `ssh-copy-id` (recommended)

macOS includes `ssh-copy-id`:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub your_username@your_server_ip
```

Enter the Ubuntu password one last time. The command creates `~/.ssh/authorized_keys` on the server and sets the correct permissions for you.

## Method 2: Manual copy-paste

**On your Mac**, copy the public key to the clipboard:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

It looks like this:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... your_mac_name
```

**On the Ubuntu server**, logged in as `your_username`:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the key on its own line (`Cmd + V`), then save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

Fix the permissions. SSH ignores the file if they are too open:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

# 4. Test key login

Keep password login turned on for now.

Open a **new** Terminal tab on your Mac (`Cmd + T`):

```bash
ssh your_username@your_server_ip
```

You should log in **without** the Ubuntu password. You may be asked for the key passphrase once, if you didn't save it in Keychain.

# 5. (Optional) Add a host shortcut

Add a block to `~/.ssh/config` on your Mac:

```text
Host myserver
    HostName your_server_ip
    User your_username
    Port 22
```

Now you can connect with just:

```bash
ssh myserver
```

# 6. Disable password login on the server

**Only do this after step 4 works.** Otherwise you lock yourself out.

On Ubuntu, create a config file with your own settings:

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

Add:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

> **Why a separate file?** Ubuntu reads the files in `/etc/ssh/sshd_config.d/` before the main `sshd_config`, and the first value it finds wins. Some installs ship `50-cloud-init.conf` with `PasswordAuthentication yes`, which overrides edits to `sshd_config`. Naming your file `00-...` makes sure it's read first.
>
> Leave `UsePAM yes` in `/etc/ssh/sshd_config` as it is. Ubuntu needs it for login sessions.

# 7. Validate the config and restart SSH

Check the config for errors **before** you restart:

```bash
sudo sshd -t
```

No output means the config is valid.

Confirm the settings that SSH will actually use:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|permitrootlogin|kbdinteractive'
# Expected:
# pubkeyauthentication yes
# passwordauthentication no
# kbdinteractiveauthentication no
# permitrootlogin no
```

Restart SSH and check its status:

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

# 8. Test again before you close your current session

**Keep your current SSH session open.** If something is wrong, you can use it to fix the config.

**1. In a new Mac Terminal tab, test key login:**

```bash
ssh your_username@your_server_ip
```

You should log in without the Ubuntu password.

**2. Check that password login is now refused:**

```bash
ssh -o PubkeyAuthentication=no your_username@your_server_ip
```

Expected:

```text
your_username@your_server_ip: Permission denied (publickey).
```

That means password login is disabled. ✅

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Still asks for the Ubuntu password | Check permissions on the server: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`. Also check that the key sits on one line in `authorized_keys`. |
| `Permission denied (publickey)` with the key | Run `ssh -v your_username@your_server_ip` to see which key is offered, then check `IdentityFile` in `~/.ssh/config`. |
| Password login still works after step 6 | Run `sudo grep -ri passwordauthentication /etc/ssh/sshd_config.d/`. Another file sets `yes` and is read before yours. |
| Locked out | Log in on the server's console and fix `/etc/ssh/sshd_config.d/00-hardening.conf`, then run `sudo systemctl restart ssh`. |

## Next step

➡️ [4-publish-ssh-tunnel.md](4-publish-ssh-tunnel.md)
