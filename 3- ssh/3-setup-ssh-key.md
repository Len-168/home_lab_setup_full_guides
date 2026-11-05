# 1. Access from Local Machine to SSH Server with SSH Key (Access without password):
On Windows PowerShell, Linux, or macOS:
```bash
ssh-keygen -t ed25519
```
You will see:
```text
Enter file in which to save the key: ~/.ssh/id_ed25519
```
Press Enter twice to finish.

Default location:
```text
# Windows
C:\Users\YOUR_USERNAME\.ssh\id_ed25519

# Linux/Mac
~/.ssh/id_ed25519
```
You will have two files:
```text
# Private key (keep secret) (Never copy or share)
~/.ssh/id_ed25519

# Public key (share this one to server)
~/.ssh/id_ed25519.pub
```

# 2. Copy the SSH public key to Server (Two methods)

## Method 1: The Automated One-Liner

**1. If your local machine is Linux/macOS:**
```bash
ssh-copy-id dev@[IP_ADDRESS]
```
Then enter the Ubuntu password one final time.


**2. On Windows, `ssh-copy-id` often isn't installed. You can run:**
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh dev@[IP_ADDRESS] "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
Enter your Ubuntu password.


## Method 2: The Manual Copy-Paste

**1. Local machine (Linux/macOS):**

```bash
cat ~/.ssh/id_ed25519.pub
```

**2. Local machine (Windows):**
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

It will look similar to:
```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... yourpc
```


On Ubuntu: 
```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the public key.

Then save and exit. (`Ctrl + O, Enter, Ctrl + X`)

Then fix permissions:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

# 3. Test key authentication
Do not disable passwords yet.

Open a new terminal on your local computer:

```bash
ssh dev@[IP_ADDRESS]
```

You should now log in without entering the Ubuntu account password.

# 4. Disable SSH password login

Only do this after key authentication works successfully.

On Ubuntu:

```bash
sudo nano /etc/ssh/sshd_config
```

Find or add:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Do not disable:
```text
UsePAM yes
```

Save and exit (`Ctrl + O, Enter, Ctrl + X`). Then restart SSH:

# 5. Validate the SSH configuration

Before restarting SSH:

```bash
sudo sshd -t
```

If it returns nothing, the configuration is valid.

Then restart SSH:

```bash
sudo systemctl restart ssh
```

Check ssh status:

```bash
sudo systemctl status ssh
```

# 6. Test again before closing your current SSH session
Keep your existing SSH terminal open.

**1. Open a second terminal and run:**

```bash
ssh dev@[IP_ADDRESS]
```

Should log in without password.

**2. Then explicitly test password authentication:**
```bash
ssh -o PubkeyAuthentication=no dev@[IP_ADDRESS]
```

You should get something similar to:

```text
Permission denied (publickey).
```

That means password SSH is disabled correctly.
