
## Day 5 — Processes and System Monitoring

---

### Why This is Critical for DevOps

Your application is running on a server but it's extremely slow. Users are complaining. You need to answer: Is the app even running? Is it consuming too much CPU or memory? Is something else eating up all the resources? How do I kill a stuck process?

Today's commands answer all of these.

---

### What You Will Learn Today

- What a process is
- ps — see running processes
- top — live system monitor
- htop — better version of top
- kill — stop a process
- systemctl — manage services
- df — disk space
- du — folder size
- free — memory usage

---

### What is a Process?

Every time you run a command or open an application, Linux creates a **process** — a running instance of that program. Every process has:
- A **PID** (Process ID) — a unique number
- An **owner** — which user started it
- A **status** — running, sleeping, stopped, zombie

Think of it like a kitchen — processes are all the dishes being cooked simultaneously. The PID is the ticket number for each dish.

---

### Command 1 — ps (Process Status)

```bash
ps                         # shows only your current terminal processes
ps aux                     # shows EVERY process on the entire system
ps aux | grep nginx        # find if nginx is running
ps aux | grep python       # find python processes
ps aux | sort -k3 -rn | head -10  # top 10 CPU consuming processes
```

| Column | Meaning |
|--------|---------|
| USER | Who started this process |
| PID | Process ID number |
| %CPU | How much CPU it's using |
| %MEM | How much memory it's using |
| STAT | Status (R=running, S=sleeping, Z=zombie) |
| COMMAND | What command started it |

**Process status codes:**

| Status | Meaning |
|--------|---------|
| R | Running — actively using CPU |
| S | Sleeping — waiting for something |
| D | Uninterruptible sleep — waiting for disk/network |
| Z | Zombie — finished but not cleaned up |
| T | Stopped — paused |

> **Zombie processes** — a finished process whose parent hasn't acknowledged it. Takes no resources but clutters the process table. Usually harmless but worth investigating if many appear.

---

### Command 2 — top (Live System Monitor)

Shows a live, continuously updating view. Refreshes every 3 seconds.

```bash
top
```

**Understanding the top header:**
```
top - 10:30:01 up 2 days, 3:45,  2 users,  load average: 0.52, 0.58, 0.55
Tasks: 213 total,   1 running, 212 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.2 us,  1.3 sy,  92.8 id,  0.4 wa
MiB Mem :   7845.6 total,   2341.2 free,   3210.4 used
```

| Line | What it tells you |
|------|-----------------|
| load average: 0.52, 0.58, 0.55 | System load in last 1, 5, 15 minutes |
| %Cpu id: 92.8 | CPU is 92.8% idle — system is relaxed |
| %Cpu us: 5.2 | 5.2% CPU used by user processes |
| Mem used | How much RAM is being used |

**Load average explained:**
- Load of 1.0 on a single core = 100% busy
- Load of 0.5 = 50% busy — comfortable
- Load of 2.0 on single core = overloaded — processes are waiting
- On a 4-core CPU, load of 4.0 = 100% busy

**Rule of thumb:** If load average consistently exceeds your number of CPU cores — your server is struggling.

| Key inside top | What it does |
|----------------|-------------|
| P | Sort by CPU usage |
| M | Sort by memory usage |
| k | Kill a process (asks for PID) |
| q | Quit |
| 1 | Show individual CPU cores |

> ⚠️ People panic and press Ctrl+C to exit top. Use `q` to quit cleanly.

---

### Command 3 — htop

Like `top` but with colors, mouse support, and visual bars. Much easier to read. Press `q` or F10 to quit.

```bash
htop
sudo apt install htop    # if not installed (Ubuntu)
sudo yum install htop    # if not installed (CentOS)
```

| Feature | top | htop |
|---------|-----|------|
| Color coded | No | Yes |
| Mouse support | No | Yes — can click |
| Kill process | Type k then PID | Select and press F9 |
| Visual CPU/Memory bars | No | Yes |

---

### Command 4 — kill (Stop a Process)

```bash
# First find the PID
ps aux | grep nginx

# Then kill it
kill 1234           # sends SIGTERM (15) — polite request to stop
kill -9 1234        # sends SIGKILL — force kill immediately
kill -15 1234       # explicit SIGTERM
kill -1 1234        # sends SIGHUP — reload config without stopping
killall nginx       # kill ALL processes named nginx
pkill -f "python app"  # kill by pattern match
```

**Kill signals:**

| Signal | Number | Meaning |
|--------|--------|---------|
| SIGTERM | 15 | Please stop gracefully — save your work first. Default. |
| SIGKILL | 9 | Stop immediately — no cleanup, no mercy |
| SIGHUP | 1 | Reload configuration — restart without fully stopping |

**When to use which:**
- Always try `kill` (SIGTERM) first — gives the process chance to clean up
- Only use `kill -9` if SIGTERM is ignored — no chance to clean up
- Think of SIGTERM as asking someone to please leave, SIGKILL as physically removing them

---

### Command 5 — systemctl (Manage Services)

A service is a process that runs continuously in the background — web server, database, SSH daemon. Services start automatically when the system boots.

```bash
systemctl status nginx       # is nginx running? show details
systemctl start nginx        # start nginx
systemctl stop nginx         # stop nginx
systemctl restart nginx      # stop then start again
systemctl reload nginx       # reload config WITHOUT full restart (zero downtime)
systemctl enable nginx       # start nginx automatically on boot
systemctl disable nginx      # don't start on boot
systemctl is-active nginx    # prints "active" or "inactive"
journalctl -u nginx -n 50   # show last 50 log lines for nginx
```

**Reading `systemctl status` output:**

| Part | Meaning |
|------|---------|
| active (running) | Service is running right now |
| enabled | Will start automatically on boot |
| Main PID: 1234 | The process ID of this service |

> ⚠️ **Always prefer `reload` over `restart` on production servers** — reload applies new config with zero downtime. Restart briefly drops all connections.

**If service won't start — check logs immediately:**
```bash
journalctl -u nginx -n 50   # last 50 lines of nginx logs
```

---

### Command 6 — df (Disk Free)

```bash
df -h        # show disk usage — human readable (GB/MB)
```

Example output:
```
Filesystem  Size  Used  Avail  Use%  Mounted on
/dev/sda1    50G   23G    25G   48%  /
/dev/sda2   100G   78G    18G   82%  /var
tmpfs        3.9G    0   3.9G    0%  /dev/shm
```

> ⚠️ If `Use%` reaches **80%** — send an alert. At **90%+** applications start failing. At **100%** the server can crash completely. DevOps engineers set up monitoring alerts at 80%.

---

### Command 7 — du (Disk Usage)

Find what is eating your disk space:

```bash
du -sh /var/log         # total size of /var/log folder
du -sh *                # size of everything in current folder
du -sh /var/*           # which folder inside /var is biggest?
du -sh /home/*          # which user is using most space?
```

**Real scenario:** Disk is 85% full. Find what's taking space:
```bash
du -sh /var/*           # find the biggest folder
du -sh /var/log/*       # drill down into logs
```

---

### Command 8 — free (Memory Usage)

```bash
free -h     # show RAM usage — human readable
```

Example output:
```
              total    used    free  shared  buff/cache  available
Mem:           7.7G    3.1G    2.3G    234M        2.2G       4.1G
Swap:          2.0G    0.0G    2.0G
```

| Column | Meaning |
|--------|---------|
| total | Total RAM installed |
| used | RAM currently in use |
| free | RAM not used at all |
| buff/cache | RAM used by OS for caching — can be freed if needed |
| available | RAM actually available for new processes |

> **Don't panic if `free` is low.** Linux uses spare RAM as cache to speed things up. Look at `available` — that's the real number that matters.

**Swap:** Disk space used as emergency RAM when real RAM fills up. Disk is much slower than RAM so heavy swap usage = slow server. If you see swap being heavily used — the server needs more RAM.

---

### Putting It All Together — Real Scenario

Your manager says: "The application server is slow. Investigate."

```bash
top                              # is any process using 100% CPU?
free -h                          # is RAM full? is swap being used?
df -h                            # is disk full?
ps aux | grep myapp              # is my app even running?
systemctl status myapp           # what does the service say?
du -sh /var/log/*                # are log files eating the disk?
```

Six commands. Complete picture. That's real DevOps work.

---

### Full Command Summary — Day 5

| Command | What it does | Key flag |
|---------|-------------|---------|
| `ps aux` | Snapshot of all processes | `| grep name` to filter |
| `top` | Live process monitor | P=CPU sort, M=mem sort, q=quit |
| `htop` | Better live monitor | Mouse support, color coded |
| `kill PID` | Stop process gracefully | `-9` for force kill |
| `killall name` | Kill all processes by name | |
| `systemctl status` | Check service status | |
| `systemctl start/stop/restart` | Control services | |
| `systemctl reload` | Reload config, zero downtime | |
| `systemctl enable/disable` | Control boot startup | |
| `df -h` | Disk space on partitions | Watch for 80%+ usage |
| `du -sh folder` | Size of specific folder | |
| `free -h` | RAM usage | Look at available column |

---

### Interview Questions — Day 5

**Q1. How do you check if a service like nginx is running?**
`systemctl status nginx` — shows whether it is active or inactive, its PID, and recent log output.

**Q2. What is the difference between `kill` and `kill -9`?**
`kill` sends SIGTERM — a polite request to stop gracefully. `kill -9` sends SIGKILL — forces immediate termination with no cleanup. Always try SIGTERM first.

**Q3. What is a zombie process?**
A process that has finished executing but has not been removed from the process table because its parent process hasn't acknowledged its exit. It consumes no resources but should be investigated if many appear.

**Q4. How do you find which process is using the most CPU?**
Run `top` and press `P` to sort by CPU usage. The highest consumer appears at the top.

**Q5. What is swap memory?**
Disk space used as overflow when RAM is full. Much slower than RAM so heavy swap usage means the server needs more memory.

**Q6. Your disk is at 90% usage. How do you find what is taking up space?**
Use `df -h` to identify which partition is full, then `du -sh /path/*` to drill down into folders and find the largest ones.

**Q7. What does `systemctl enable` do?**
Configures a service to start automatically every time the system boots. It does not start it immediately — use `systemctl start` for that.

---

### Homework — Before Day 6

1. Run `ps aux | grep bash` — how many bash processes are running?
2. Run `top` — press P to sort by CPU. What process is at the top?
3. Run `free -h` — how much RAM is available on your system?
4. Run `df -h` — what percentage full is your main disk?
5. Run `systemctl status ssh` — is SSH running on your system?
6. Run `du -sh ~/linux_practice` — how big is your practice folder?