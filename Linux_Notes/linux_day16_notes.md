## Day 16 — Interview Mega Revision

---

### How 10–12 LPA DevOps Interviews Work

Most interviews have 3 rounds:

| Round | What happens |
|-------|-------------|
| Round 1 | HR screening — background, salary, availability |
| Round 2 | Technical — Linux, Git, Docker, AWS questions |
| Round 3 | Practical/scenario based — "what would you do if..." |

**Interview Strategy:**
- Explain what a command does before giving it — shows understanding, not memorization
- Give real examples — "I would use this when the server disk is full"
- Admit what you don't know: "I haven't worked with that specifically but I know the concept..."
- Never panic — interviewers expect juniors not to know everything. Calmness and logical thinking impress more than perfect answers

---

## Section 1 — Linux Fundamentals

**Q1. What is Linux and why do DevOps engineers use it?**
Linux is an open source operating system used on approximately 95% of all servers worldwide. DevOps engineers use it because it is stable, secure, highly customizable and free. It provides full control through the terminal — you can automate anything, schedule tasks, manage users, configure networking — all through commands and scripts.

**Q2. What is the difference between Linux and Unix?**
Unix is the original OS developed at Bell Labs in the 1960s–70s. Linux was created by Linus Torvalds in 1991 as a free open-source alternative inspired by Unix. Linux follows Unix principles and commands are nearly identical — but Linux is not derived from Unix code.

**Q3. What is the kernel?**
The kernel is the core of the operating system. It sits between hardware and software — managing CPU, memory, devices and system calls. Users never interact with the kernel directly — the shell provides the interface.

**Q4. What is the difference between absolute path and relative path?**
```bash
# Absolute path — starts from root /
/home/varun/linux_practice/day1/notes.txt

# Relative path — starts from current location
./day1/notes.txt       # from linux_practice folder
../linux_practice/     # going up one level then down
```
Absolute path works from anywhere. Relative path depends on where you currently are.

**Q5. Explain the Linux boot process.**
1. BIOS/UEFI — hardware self-test, finds bootable device
2. Bootloader (GRUB) — loads the Linux kernel into memory
3. Kernel — initializes hardware, mounts root filesystem
4. Init/Systemd — starts all services and processes
5. Login prompt — system ready for user

---

## Section 2 — File System and Permissions

**Q6. Explain Linux file permissions completely.**
Every file has three groups — owner, group, others. Each group has three permissions — read (r=4), write (w=2), execute (x=1).

Breaking down `-rw-r--r--`:
- `-` = regular file (d=directory, l=symlink)
- `rw-` = owner can read and write (6)
- `r--` = group can only read (4)
- `r--` = others can only read (4)
- Octal = 644

**Q7. What is the difference between hard link and soft link?**

| Feature | Hard Link | Soft Link (Symlink) |
|---------|-----------|---------------------|
| Inode | Same inode as original | Different inode |
| If original deleted | Still works | Breaks (dangling link) |
| Cross filesystems | Cannot | Can |
| Create with | `ln file link` | `ln -s file link` |

```bash
ln file.txt hardlink         # hard link
ln -s file.txt softlink      # soft link (symlink)
ls -l                        # symlinks show -> pointing to original
```

---

## Section 3 — Processes and System Monitoring

**Q8. How do you find which process is consuming the most memory?**
```bash
top                      # press M to sort by memory
ps aux --sort=-%mem | head -10   # top 10 by memory
```

**Q9. What happens when you run `kill -9` vs `kill -15`?**
`kill -15` (SIGTERM) politely asks the process to stop — it can save state and clean up. `kill -9` (SIGKILL) immediately terminates — no cleanup, no saving. Always try `-15` first.

**Q10. What is the difference between `systemctl restart` and `systemctl reload`?**
`restart` fully stops and starts the service — brief interruption, drops connections. `reload` reloads only the configuration without stopping the service — zero downtime. Always prefer `reload` on production servers when just changing config.

---

## Section 4 — Networking

**Q11. How do you check which process is listening on port 80?**
```bash
ss -tulnp | grep :80
netstat -tulnp | grep :80
```

**Q12. What is the difference between `0.0.0.0` and `127.0.0.1` for a listening service?**
`0.0.0.0` — listens on all network interfaces, accessible from outside. `127.0.0.1` — listens only on loopback, accessible only from the same machine.

**Q13. A deployment went fine but application is unreachable. Systematic diagnosis:**
```bash
systemctl status myapp              # is it running?
ss -tulnp | grep 8080               # is it listening on the right port?
ss -tulnp | grep "0.0.0.0:8080"    # is it publicly accessible?
sudo ufw status                     # is firewall blocking it?
curl -I http://localhost:8080       # does it respond locally?
nslookup yourdomain.com             # is DNS resolving correctly?
```

---

## Section 5 — Shell Scripting

**Q14. What does `set -euo pipefail` do?**
`-e` exits immediately if any command fails. `-u` treats undefined variables as errors. `-o pipefail` catches failures in pipelines. Together they make scripts fail fast and predictably — essential for safe automation.

**Q15. What is the difference between `$*` and `$@`?**
```bash
# With "$*" — all args treated as one string
for arg in "$*"; do echo $arg; done
# If args are "hello world" — prints: hello world (one item)

# With "$@" — each arg treated separately
for arg in "$@"; do echo $arg; done
# If args are "hello world" — prints:
# hello
# world (two items)
```
Always use `"$@"` when you want to preserve individual arguments.

**Q16. What is the difference between `[ ]` and `[[ ]]`?**
```bash
[ "$var" == "value" ]    # Single bracket — POSIX compatible — older
[[ "$var" == "value" ]]  # Double bracket — bash specific — more features
[[ "$var" =~ ^[0-9]+$ ]] # supports regex — only in [[ ]]
[[ -f file && -r file ]] # supports && directly — only in [[ ]]
```
Double brackets are safer — no word splitting issues. Use `[[ ]]` in bash scripts.

**Q17. Write a script to check if a service is running and restart if not.**
```bash
#!/bin/bash
SERVICE="nginx"
LOG="/var/log/service_monitor.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG
}

if systemctl is-active --quiet $SERVICE; then
    log "$SERVICE is running — OK"
else
    log "WARNING: $SERVICE is not running. Attempting restart..."
    sudo systemctl start $SERVICE
    if systemctl is-active --quiet $SERVICE; then
        log "$SERVICE restarted successfully"
    else
        log "ERROR: $SERVICE failed to restart. Manual intervention needed."
        exit 1
    fi
fi
```

**Q18. How do you handle errors in shell scripts professionally?**
```bash
#!/bin/bash
set -euo pipefail

# Manual error handling
command || { echo "Command failed"; exit 1; }

# Trap — run cleanup on error
cleanup() {
    rm -f /tmp/tempfile
    echo "Cleanup done"
}
trap 'echo "Script failed on line $LINENO"; cleanup' ERR
```

---

## Section 6 — User and Package Management

**Q19. How do you add a user with sudo access?**
```bash
sudo useradd -m -s /bin/bash -c "Developer Name" username
sudo passwd username
sudo passwd -e username           # force password change on first login
sudo usermod -aG sudo username    # Ubuntu
sudo usermod -aG wheel username   # CentOS
id username                       # verify groups
```

**Q20. What is the difference between `apt update`, `apt upgrade` and `apt full-upgrade`?**
```bash
apt update        # refreshes package list — no installs, no upgrades
apt upgrade       # upgrades packages — never removes existing packages
apt full-upgrade  # upgrades and removes packages if required for dependencies
```
Safe production sequence:
```bash
sudo apt update && sudo apt upgrade   # safe — no package removal
```

---

## Section 7 — Scenario Based Questions

**Q21. Your application server is slow. How do you diagnose it?**
```bash
# Step 1 — CPU and memory
top                          # is any process using 100% CPU?
free -h                      # is RAM full? is swap being used heavily?

# Step 2 — Disk
df -h                        # is disk full?
du -sh /var/log/*            # are log files eating the disk?

# Step 3 — Network
ss -tulnp                    # are ports open and listening?

# Step 4 — Application logs
tail -f /var/log/app/error.log
journalctl -u myapp -n 100

# Step 5 — Process specific
ps aux | grep myapp          # find the process
lsof -p PID                  # what files does it have open?
```

**Q22. How do you recover from running `rm -rf` on the wrong directory?**
Honest answer — you likely cannot recover without backups. This is why:
- Always maintain automated backups (cron jobs from Day 9)
- Test destructive commands with `echo rm -rf /path` first
- Use `rm -i` flag which asks confirmation
- Restore from tar backup: `tar -xzf backup_20240320.tar.gz -C /restore/`

**This is why backups from Day 9 cron jobs and Day 12 tar scripts are so critical.**

**Q23. Server is not starting after reboot. What do you do?**
```bash
journalctl -b -1   # logs from previous boot
journalctl -b 0    # logs from current boot
journalctl -xe     # recent logs with explanations
systemctl --failed # which services failed
systemctl status nginx        # check specific failed service
df -h              # is disk full — full disk prevents services from starting
dmesg | grep error # hardware errors
```

**Q24. How do you secure a fresh Linux server?**
```bash
# 1. Update all packages first
sudo apt update && sudo apt upgrade -y

# 2. Create non-root user with sudo
sudo useradd -m -s /bin/bash devops
sudo passwd devops
sudo usermod -aG sudo devops

# 3. Set up SSH key authentication
ssh-copy-id devops@server

# 4. Disable root SSH and password authentication
sudo vim /etc/ssh/sshd_config
# PermitRootLogin no
# PasswordAuthentication no
# Port 2222 (change default port)
sudo systemctl restart ssh

# 5. Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222    # new SSH port
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable

# 6. Install fail2ban — blocks brute force
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
```

---

## Section 8 — Text Processing Questions

**Q25. Given a log file — find the top 5 IPs with most requests.**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -5
```
Step by step: extract IP column → sort IPs → count consecutive duplicates → sort by count descending → show top 5.

**Q26. How do you find all files larger than 100MB?**
```bash
find / -size +100M -type f 2>/dev/null
find / -size +100M -type f -ls 2>/dev/null        # with details
find /var -size +100M -exec ls -lh {} \;          # human readable size
```

**Q27. How do you count occurrences of a word in a file?**
```bash
grep -o "ERROR" logfile | wc -l     # count occurrences
grep -c "ERROR" logfile             # count matching lines
```

---

## Section 9 — Quick Fire Round

**Q. How do you check Linux version?**
```bash
cat /etc/os-release
lsb_release -a
uname -r          # kernel version only
```

**Q. How do you check how long server has been running?**
```bash
uptime
```

**Q. How do you check open file handles?**
```bash
lsof              # list all open files
lsof -u varun     # files opened by user
lsof -p 1234      # files opened by process ID
```

**Q. What is `/dev/null`?**
A black hole — anything written to it disappears. Used to discard output:
```bash
command > /dev/null 2>&1    # discard all output
```

**Q. What does `2>&1` mean?**
Redirects stderr (file descriptor 2) to the same place as stdout (file descriptor 1). Ensures error messages go to the same destination as regular output.

**Q. What is umask?**
```bash
umask    # shows default permission mask for new files
# umask 022 means new files get 644, new folders get 755
```

**Q. How do you run a process in the background?**
```bash
command &         # run in background
nohup command &   # run in background — survives logout
screen            # terminal multiplexer — keeps sessions alive
tmux              # better terminal multiplexer
jobs              # list background jobs
fg                # bring job to foreground
bg                # send stopped job to background
```

**Q. What is the nice value?**
```bash
nice -n 10 command    # run with lower priority (19=lowest, -20=highest)
renice 10 -p PID      # change priority of running process
```

**Q. What is `/proc`?**
A virtual filesystem showing kernel and process information:
```bash
cat /proc/cpuinfo         # CPU info
cat /proc/meminfo         # memory info
cat /proc/1234/status     # status of process 1234
cat /proc/PID/environ | tr '\0' '\n'  # environment of running process
```

**Q. How do you see environment of a running process?**
```bash
cat /proc/PID/environ | tr '\0' '\n'
```

---

### Phrases That Impress Interviewers

- "In production I would always test with `--dry-run` first"
- "Before deleting I would create a backup"
- "I would check the logs in `/var/log` first"
- "I would use `-i` flag to confirm before overwriting"
- "On production servers I prefer `reload` over `restart` to avoid downtime"
- "I would check disk usage with `df -h` to rule out space issues"
- "I would use `set -euo pipefail` at the top of any production script"

---

## Section 10 — Your Complete Journey Summary

### Everything You Have Mastered

```
DAY 1  ✅  pwd, ls, cd — navigating Linux like a professional
DAY 2  ✅  mkdir, touch, cp, mv, rm — creating and managing files
DAY 3  ✅  cat, less, head, tail, grep — reading and searching files
DAY 4  ✅  chmod, chown, sudo — permissions and ownership
DAY 5  ✅  ps, top, kill, systemctl, df, free — processes and monitoring
DAY 6  ✅  ping, curl, wget, ss, dig — networking and troubleshooting
DAY 7  ✅  awk, sed, cut, sort, uniq, pipes — text processing
DAY 8  ✅  shell scripting — variables, loops, functions, automation
DAY 9  ✅  cron jobs — scheduling automation
DAY 10 ✅  useradd, passwd, groups, sudo — user management
DAY 11 ✅  apt, yum — package management
DAY 12 ✅  tar, gzip, zip — compression and archives
DAY 13 ✅  SSH, scp, rsync — remote access and file transfer
DAY 14 ✅  Vim — terminal text editing
DAY 15 ✅  environment variables, .bashrc, aliases, .env files
DAY 16 ✅  interview mega revision — scenario questions, quick fire
```

---

### What Interviewers at 10–12 LPA Companies Test

```
Linux fundamentals     — 20% of questions
File permissions       — 15% of questions
Networking commands    — 20% of questions
Shell scripting        — 20% of questions
Scenario based         — 25% of questions
```

---

### Your Next Steps After Linux

```
Linux ✅ COMPLETE
↓
Git & GitHub (2 weeks)
↓
Docker & Containers (2 weeks)
↓
AWS Basics (3 weeks) ← doing this in parallel
↓
CI/CD — Jenkins or GitHub Actions (1 week)
↓
Kubernetes basics (1 week)
↓
First DevOps job application
```
