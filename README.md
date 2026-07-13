# SSH Remote Server Setup

A remote Linux server (AWS EC2, Ubuntu) configured to allow secure, key-based SSH access using two separate SSH key pairs and with password authentication disabled and fail2ban protecting against brute-force attempts.

## Overview

- **Provider:** AWS EC2
- **OS:** Ubuntu
- **Instance type:** t3.micro
- **Region:** eu-west-1 (Ireland)

## Steps Taken

### 1. Provisioned the server
Created an EC2 instance on AWS using the Ubuntu AMI. AWS generated a default key pair (`.pem` file) at launch time, which was used for the initial connection.

### 2. Generated two SSH key pairs locally
On my local machine (macOS), generated two Ed25519 key pairs:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_server1 -C "key1"
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_server2 -C "key2"
```

This produced two private/public key pairs. Private keys were kept local only and never committed to this repository.

### 3. Added both public keys to the server
Connected to the server using the AWS-issued `.pem` key:

```bash
ssh -i ~/Downloads/ubuntuwebserverkeypair.pem ubuntu@<server-ip>
```

Then appended both generated public keys to the server's `~/.ssh/authorized_keys` file:

```bash
echo "<public-key-contents>" >> ~/.ssh/authorized_keys
```

### 4. Verified both keys work independently
Confirmed each key could authenticate on its own:

```bash
ssh -i ~/.ssh/id_ed25519_server1 ubuntu@<server-ip>
ssh -i ~/.ssh/id_ed25519_server2 ubuntu@<server-ip>
```

Both connected successfully without a password prompt.

### 5. Configured SSH aliases
Added entries to `~/.ssh/config` on the local machine to allow shorthand connections:

```
Host awsserver
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519_server2

Host awsserver-key1
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519_server1
```

Verified both aliases connect correctly:

```bash
ssh awsserver
ssh awsserver-key1
```

### 6. Hardened SSH access
Edited `/etc/ssh/sshd_config` on the server to disable password-based login and root login, forcing key-only authentication:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Validated the config and restarted the SSH service:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

Confirmed both keys could still connect after the change before closing the original session.

### 7. Installed and configured fail2ban (stretch goal)
Installed fail2ban to automatically ban IPs after repeated failed SSH login attempts:

```bash
sudo apt update
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Configured the `[sshd]` jail in `/etc/fail2ban/jail.local`:

```
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600
findtime = 600
```

Enabled and started the service:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verified the jail was active and monitoring SSH:

```bash
sudo fail2ban-client status sshd
```

## Outcome

- Server accessible via two independently-generated SSH key pairs
- Both keys work via full command and via short config aliases
- Password authentication disabled — key-only access enforced
- Root login disabled
- fail2ban actively monitoring and banning brute-force attempts on port 22

## Notes

- No private keys, `.pem` files, or the `~/.ssh/config` file (which contains the server IP) are included in this repository.
- Server IP addresses in this README are placeholders (`<server-ip>`) — replace with actual values if adapting these steps, but avoid publishing your own real IP alongside identifying details if security is a concern.
