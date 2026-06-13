## Day 13 — SSH & Remote Access

---

### Why This is Critical for DevOps

Every server you will ever manage in your career will be accessed using SSH. You will never physically sit in front of those servers — they are in data centers thousands of kilometers away. SSH is how you reach them. This is also one of the most heavily tested topics in 10–12 LPA DevOps interviews.

---

### What is SSH?

SSH (Secure Shell) — lets you log into a remote server securely over a network, run commands, and transfer files. Everything is encrypted. Before SSH, people used Telnet which sent everything including passwords as plain text. Default port is **22**.

---

### What You Will Learn Today

- Basic SSH connection
- SSH key pairs (public and private keys)
- Generating and copying keys
- SSH config file
- SCP — copying files between servers
- rsync — syncing files efficiently
- Common SSH errors and fixes

---

### Basic SSH Connection

```bash
ssh username@server_ip
ssh varun@192.168.1.100
ssh varun@myserver.company.com
ssh -p 2222 varun@192.168.1.100     # non-standard port
ssh -i ~/.ssh/mykey.pem varun@server  # specific key file
ssh -v varun@server                  # verbose — debug connection
ssh -vv varun@server                 # more verbose — deep debugging
ssh varun@192.168.1.100 "df -h"      # run single command and disconnect
```

**Run command on multiple servers:**
```bash
#!/bin/bash
for server in 192.168.1.10 192.168.1.11 192.168.1.12; do
    echo "=== $server ==="
    ssh varun@$server "df -h / | tail -1"
done
```
Checks disk usage on 3 servers in 3 seconds.

---

### SSH Authentication Methods

| Method | How it works | Security |
|--------|-------------|---------|
| Password | Type username and password | Less secure — brute force risk |
| SSH Key Pair | Cryptographic key files | Much more secure — industry standard |

**In real DevOps work — always use SSH keys. Never use passwords for server access.**

---

### SSH Key Pairs

**Private key** = stays on your laptop, NEVER share  
**Public key** = goes on the remote server, safe to share  

Think of it as: public key = the padlock (give to anyone), private key = the key to that padlock (keep secret).

**Generating SSH Keys:**
```bash
ssh-keygen -t rsa -b 4096 -C "varun@company.com"
ssh-keygen -t ed25519 -C "varun@company.com"    # modern, preferred
```

| Flag | Meaning |
|------|---------|
| `-t rsa` | Key type — RSA algorithm |
| `-t ed25519` | Modern algorithm — smaller and faster |
| `-b 4096` | Key size — 4096 bits (stronger) |
| `-C` | Comment — usually your email |

Creates:
- `~/.ssh/id_rsa` — private key (NEVER share)
- `~/.ssh/id_rsa.pub` — public key (share this with servers)

**Fix permissions — SSH refuses key if too open:**
```bash
chmod 600 ~/.ssh/id_rsa          # private key
chmod 644 ~/.ssh/id_rsa.pub      # public key
chmod 700 ~/.ssh/                # .ssh directory
```

---

### Copy Public Key to Server

```bash
# Easiest way
ssh-copy-id varun@192.168.1.100

# Manual method — on your machine, copy public key
cat ~/.ssh/id_rsa.pub
# On the server, paste it
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys    # paste public key here
chmod 600 ~/.ssh/authorized_keys

# One liner
cat ~/.ssh/id_rsa.pub | ssh varun@192.168.1.100 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

After this — passwordless login works:
```bash
ssh varun@192.168.1.100    # no password asked!
```

---

### SSH Config File — Making Life Easy

Instead of typing `ssh -p 2222 -i ~/.ssh/key.pem varun@ec2-13-234.amazonaws.com` every time:

```bash
nano ~/.ssh/config
chmod 600 ~/.ssh/config    # required — SSH refuses to use config if too open
```

Add your servers:
```
Host myserver
    HostName 192.168.1.100
    User varun
    Port 22
    IdentityFile ~/.ssh/id_rsa

Host aws-prod
    HostName ec2-13-234-56-78.compute-1.amazonaws.com
    User ubuntu
    IdentityFile ~/.ssh/aws_key.pem

Host aws-dev
    HostName 54.123.45.67
    User ubuntu
    IdentityFile ~/.ssh/aws_key.pem

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

Now connect with just:
```bash
ssh myserver      # instead of full command
ssh aws-prod      # production server
ssh aws-dev       # dev server
```

---

### SCP — Secure Copy Between Servers

```bash
# Local to remote
scp file.txt varun@192.168.1.100:/home/varun/       # copy file
scp -r myfolder/ varun@192.168.1.100:/home/varun/   # copy folder

# Remote to local
scp varun@192.168.1.100:/var/log/app.log ./          # download log
scp -r varun@192.168.1.100:/var/www/html/ ./backup/  # download folder

# Using SSH config aliases
scp file.txt myserver:/home/varun/
scp aws-prod:/var/log/app.log ./

# Custom port — NOTE: capital P for scp (different from ssh)
scp -P 2222 file.txt varun@server:/home/varun/
scp -i ~/.ssh/mykey.pem file.txt varun@server:/home/varun/
```

> ⚠️ SCP uses capital `-P` for port. `ssh` uses lowercase `-p`. This trips many people up.

---

### rsync — Smart File Syncing

SCP always copies the entire file even if 1 byte changed. rsync only transfers the parts that changed — much faster for large files or repeated transfers.

```bash
rsync -avz source/ varun@192.168.1.100:/destination/
rsync -avz varun@192.168.1.100:/var/www/ ./backup/    # sync to local
rsync -avzn /var/www/ server:/var/www/                # dry run first!
rsync -avz --delete /var/www/ server:/var/www/        # delete files removed from source
rsync -avz --exclude="node_modules" --exclude=".git" /app/ server:/app/
```

| Flag | Meaning |
|------|---------|
| -a | Archive mode — preserves permissions, timestamps, symlinks |
| -v | Verbose — shows what is being transferred |
| -z | Compress data during transfer |
| -n or --dry-run | Show what WOULD happen without doing it |
| --delete | Delete files on destination not in source |
| --exclude | Skip certain files or folders |

> **Always do a dry run first** with `-n` before using `--delete` — you might accidentally delete something important.

**rsync vs scp — when to use which:**

| Situation | Use |
|-----------|-----|
| One-time file copy | `scp` — simpler |
| Repeated sync of large folders | `rsync` — only transfers changes |
| Deployment — push code to server | `rsync` — fast incremental |
| Mirror directories | `rsync --delete` |

---

### Common SSH Errors and Fixes

**Error 1 — Connection Refused:**
```
ssh: connect to host 192.168.1.100 port 22: Connection refused
```
```bash
sudo systemctl status ssh    # is SSH service running?
sudo systemctl start ssh
sudo ufw status              # is port 22 blocked by firewall?
sudo ufw allow 22
ssh -p 2222 varun@server    # maybe non-standard port is used
```

**Error 2 — Permission Denied:**
```
varun@192.168.1.100: Permission denied (publickey)
```
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ssh -vv varun@192.168.1.100    # verbose — shows exactly what's happening
ssh -i ~/.ssh/correct_key.pem varun@server  # are you using the right key?
```

**Error 3 — Host key changed (MITM warning):**
```
REMOTE HOST IDENTIFICATION HAS CHANGED!
```
This appears when the server was rebuilt or IP reassigned. If you are sure it is safe:
```bash
ssh-keygen -R 192.168.1.100   # remove old key from known_hosts
```

---

### Interview Questions — Day 13

**Q1. What is the difference between SCP and rsync?**
SCP always copies entire files. rsync only transfers changed parts — much faster for large files or repeated syncing.

**Q2. Where is the public key stored on a remote server?**
In `~/.ssh/authorized_keys` — one public key per line.

**Q3. What permissions should SSH private key have and why?**
`chmod 600` — readable only by owner. SSH refuses to use the key if permissions are too open with the error "UNPROTECTED PRIVATE KEY FILE".

**Q4. What does the SSH config file (`~/.ssh/config`) do?**
Stores connection shortcuts — hostname, user, port, key file per server. Lets you type `ssh myserver` instead of the full command.

**Q5. What is the difference between SCP's `-P` flag and SSH's `-p` flag?**
Both specify port number but the case differs. `scp` uses capital `-P`, `ssh` uses lowercase `-p`. Common source of mistakes.

**Q6. What is SSH key-based authentication and why is it better than passwords?**
Public/private key pair — private key stays on your machine, public key on the server. Better than passwords because keys can't be guessed by brute force and can be revoked per key without changing passwords.

---
