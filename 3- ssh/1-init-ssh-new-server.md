# 1. Install SSH Server

1.1. Switch to sudo user
    
```bash 
# Make sure you are Switch to sudo user
su - your_username
```

1.2. Install SSH server

```bash
# Install SSH server
sudo apt install openssh-server
```

1.3. Verify SSH Server Status

```bash
# Check SSH server status
sudo systemctl status ssh # you should see active (running) in green color

# If not, enable and start SSH server:
sudo systemctl enable ssh
sudo systemctl start ssh
```
