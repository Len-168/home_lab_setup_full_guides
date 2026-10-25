# 1. Security & System Fundamentals

1.1. Update and Upgrade: Ensure you have the latest security patches before continuing.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

1.2. Time synchronization
Set the correct time zone:
```bash
sudo timedatectl set-timezone Asia/Phnom_Penh
```
Check status:
```bash
date
# Or
timedatectl
```
Typical output:
```text
Local time: Wed 2026-10-07 21:05:00 +07
Universal time: Wed 2026-10-07 14:05:00 UTC
Time zone: Asia/Phnom_Penh (+07, +0700)
System clock synchronized: yes
NTP service: active
```

1.3. Create a New Sudo User: Avoid using the root account for daily tasks.

```bash
# Create a new user
sudo adduser your_username

# Add user to the sudo group
sudo usermod -aG sudo your_username

# (Optional) Check user has sudo or not:
groups your_username # should return [your_username: your_username sudo]
```
