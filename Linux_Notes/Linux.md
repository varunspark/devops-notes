bash
# Linux for DevOps — Complete Day-Wise Notes


---

## Progress Tracker

| Day | Topic | Status |
|-----|-------|--------|
| Day 1 | What is Linux, Terminal, pwd / ls / cd | ✅ |
| Day 2 | mkdir, touch, cp, mv, rm — File Management | ✅ |
| Day 3 | cat, less, head, tail, grep — Viewing Files | ✅ |
| Day 4 | chmod, chown, sudo — Permissions & Users | ✅ |
| Day 5 | ps, top, kill, systemctl, df, free — Processes | ✅ |
| Day 6 | ping, curl, wget, ss, dig — Networking | ✅ |
| Day 7 | awk, sed, cut, sort, uniq, pipes — Text Processing | ✅ |
| Day 8 | Shell Scripting — variables, loops, functions | ✅ |
| Day 9 | Cron Jobs — scheduling automation | ✅ |
| Day 10 | useradd, passwd, groups, sudo — User Management | ✅ |
| Day 11 | apt, yum — Package Management | ✅ |
| Day 12 | tar, gzip, zip — Compression & Archives | ✅ |
| Day 13 | SSH, scp, rsync — Remote Access | ✅ |
| Day 14 | Vim — Terminal Text Editing | ✅ |
| Day 15 | Environment Variables, .bashrc, aliases | ✅ |
| Day 16 | Interview Mega Revision — Scenario Questions | ✅ |

---

## Day 1 — What is Linux & Basic Navigation

---

### What is Linux?

Linux is an Operating System — just like Windows. It is used on almost all servers, cloud platforms, phones (Android), and supercomputers. As a DevOps engineer, 95% of servers you manage will run Linux.

Windows gives you a graphical interface — you click icons. Linux gives you a **terminal** — you type commands. This feels scary at first but gives you full power and control over the system.

The terminal is just a text box where you type instructions and the computer responds. Think of it like texting your computer instead of clicking on it. The `$` sign means Linux is ready and waiting for your command.

---

### What You Will Learn Today

- What Linux is and why DevOps engineers use it
- Terminal basics
- pwd, ls, cd — navigating the file system
- Linux file system structure

---

### Command 1 — pwd (Print Working Directory)

**What it means:** "Where am I right now?" Think of your computer like a building with many floors and rooms. `pwd` tells you which room you are in.

```bash
pwd
```

Output: `/home/varun` — means: root → home floor → your room.

---

### Command 2 — ls (List)

**What it means:** "Show me what's inside this room."

```bash
ls          # basic list
ls -l       # detailed — permissions, owner, size, date
ls -a       # show hidden files (files starting with a dot)
ls -la      # detailed AND hidden — most used by professionals
ls -ltr     # detailed, sorted by time, newest at bottom
```

**Understanding `ls -l` output:**

```
Permissions  Links  Owner  Group  Size   Date        Name
drwxr-xr-x  2      varun  varun  4096   Feb 5 19:16  Desktop
-rw-rw-r--  1      varun  varun  344017 Feb 28       DevOps.pdf
```

| Column | Meaning |
|--------|---------|
| Permissions | Who can read/write/execute |
| Links | Number of hard links |
| Owner | Who owns this file |
| Group | Which group it belongs to |
| Size | File size in bytes |
| Date | Last modification date |
| Name | File or folder name |

**Memory trick:** "People Love Our Software, Delivering More Notably" → Permissions, Links, Owner, Software(Group), Size, Date, Name

**Why `ls -ltr`?** Newest log file appears at the very bottom — exactly what you need when debugging a crashed server. `-t` sorts by time, `-r` reverses so newest is last.

> ⚠️ No output from `ls` just means the folder is empty — totally normal.

---

### Command 3 — cd (Change Directory)

```bash
cd Desktop      # move into Desktop folder
cd ..           # go one level UP
cd ~            # go to home folder from anywhere
cd /            # go to root (very top of the system)
cd -            # go back to where you just were
cd /etc         # go to a specific path directly
```

> ⚠️ Folder names with spaces need quotes or backslash:
> ```bash
> cd my\ folder    # RIGHT — escape the space
> cd "my folder"   # RIGHT — use quotes
> cd my folder     # WRONG — space confuses Linux
> ```

---

### Linux File System Structure

```
/home     → Your personal files (like C:\Users on Windows)
/etc      → All configuration files — very important for DevOps
/var      → Variable data — log files live here
/var/log  → Application and system logs
/tmp      → Temporary files — wiped on reboot
/bin      → Basic commands like ls, cd, pwd
/usr      → Installed software
/root     → Home folder of the superuser (admin)
```

As a DevOps engineer you will spend most time in `/etc` and `/var/log`.

---

### Common Mistakes — Day 1

| Mistake | What happens | Fix |
|---------|-------------|-----|
| Typing `PWD` in capitals | "command not found" | Linux is case-sensitive — always lowercase |
| Pressing Enter on empty line | Nothing — new prompt | Normal, just type again |
| Wrong command spelling | "command not found" | Check spelling |

---

### Interview Questions — Day 1

**Q1. What does `pwd` do?**
Prints the current working directory — shows exactly where you are in the file system.

**Q2. What is the difference between `ls -l` and `ls -la`?**
`ls -l` shows a detailed listing. `ls -la` shows detailed listing including hidden files (files starting with a dot).

**Q3. What is the root directory in Linux?**
`/` — the top-most directory. Everything in Linux starts from `/`.

**Q4. What is the difference between `/root` and `/home`?**
`/home` contains home directories for all regular users. `/root` is the home directory specifically for the root (superuser/admin) user.

**Q5. How do you go to your home directory from anywhere?**
`cd ~` or just `cd` with no argument.

---

### Homework — Before Day 2

1. Start at home: `cd ~`
2. Go into `/etc`: `cd /etc` then `ls`
3. Go back home: `cd ~`
4. Try `cd /var/log` and run `ls`

Just explore. Don't change or delete anything.

---

## Day 2 — Creating, Copying, Moving and Deleting Files

---

### What You Will Learn Today

- mkdir — create folders
- touch — create files
- cp — copy files
- mv — move and rename files
- rm — delete files (carefully!)

---

### Setup — Create a Practice Folder First

Always create a safe practice area. Never experiment on real system files.

```bash
cd ~
mkdir linux_practice
cd linux_practice
```

---

### Command 1 — mkdir (Make Directory)

```bash
mkdir myfolder                      # create one folder
mkdir folder1 folder2 folder3       # create multiple at once
mkdir -p projects/devops/scripts    # create nested folders in one shot
```

**What is `-p`?**
- Without `-p`: `mkdir projects/devops/scripts` → ERROR (parent doesn't exist)
- With `-p`: creates all three levels automatically — no error

> ⚠️ If folder already exists Linux throws an error — will NOT silently overwrite.

---

### Command 2 — touch (Create Empty File)

```bash
touch myfile.txt                          # create empty file
touch file1.txt file2.txt file3.txt       # create multiple at once
touch deploy.sh config.yml .env           # create DevOps files
```

**Why called "touch"?** Originally designed to update a file's timestamp. If the file doesn't exist, it creates it. That side effect became its most common use.

---

### Command 3 — cp (Copy)

```bash
cp file1.txt file2.txt                    # copy with new name
cp file1.txt /home/varun/Desktop/         # copy to different folder
cp -r myfolder/ myfolder_backup/          # copy entire folder (MUST use -r)
cp -i file1.txt file2.txt                 # ask before overwriting
```

> ⚠️ If destination file already exists — Linux **silently overwrites**. No warning. No recycle bin. Use `-i` to be safe.

---

### Command 4 — mv (Move / Rename)

Linux has no separate rename command. `mv` does both.

```bash
mv file1.txt /home/varun/Desktop/         # MOVE to different folder
mv file1.txt newname.txt                  # RENAME — same folder, new name
mv file1.txt /Desktop/newname.txt         # MOVE and RENAME at same time
mv -i file1.txt file2.txt                 # ask before overwriting
```

**Real DevOps situation:**
```bash
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup   # backup old config
```
Rename old config as backup before deploying the new one. Done constantly on real servers.

---

### Command 5 — rm (Remove / Delete)

> ⚠️ **MOST DANGEROUS COMMAND IN LINUX** — deletes permanently. No recycle bin. No undo.

```bash
rm myfile.txt               # delete one file
rm file1.txt file2.txt      # delete multiple files
rm -r myfolder/             # delete folder and everything inside
rm -i file.txt              # ask confirmation before deleting
rm -v file.txt              # verbose — tells you what was deleted
```

**Never type this:**
```bash
rm -rf /      # DESTROYS ENTIRE OPERATING SYSTEM — no recovery
```

**Safe habits:** Always use `-i` when not 100% sure. Double check path before pressing Enter.

---

### Interview Questions — Day 2

**Q1. What does `touch` do?**
Updates the last modified timestamp of a file. If the file doesn't exist, it creates an empty file.

**Q2. What is the difference between `cp` and `cp -r`?**
`cp` copies individual files. `cp -r` copies directories recursively — the folder and all its contents including subfolders.

**Q3. How do you delete a folder with all its contents?**
`rm -r foldername` — the `-r` flag makes it recursive.

**Q4. What happens if you `cp` or `mv` to a file that already exists?**
Linux silently overwrites it with no warning. Use `-i` flag to prompt for confirmation.

**Q5. What does `mkdir -p` do?**
Creates the entire directory path including all parent directories that don't exist yet. Without `-p`, mkdir fails if the parent directory is missing.

---

### Homework — Before Day 3

Inside `linux_practice`, create this structure:
```
project/
├── src/
├── logs/
└── backup/
```
1. Create 2 files inside `src/`
2. Copy one file from `src/` to `backup/`
3. Rename the other file in `src/`
4. Delete the original from `src/` after copying
5. Run `ls -ltr project/` to verify everything

---

## Day 3 — Viewing File Contents

---

### Why This is Critical for DevOps

Imagine your application crashed on a live server at 2am. Your manager calls. You SSH in. You cannot open a file manager. You only have the terminal. You need to read log files, find the error, fix it — fast. That is exactly what today's commands are for.

---

### What You Will Learn Today

- cat — print file contents
- less — read large files page by page
- head — see top of file
- tail — see bottom of file
- tail -f — watch live updates in real time
- grep — search inside files

---

### Setup — Create a Practice Log File

```bash
cd ~/linux_practice && mkdir day3 && cd day3
cat > server.log << EOF
INFO  2024-01-01 10:00:01 Server started
INFO  2024-01-01 10:00:02 Database connected
INFO  2024-01-01 10:00:05 App running on port 8080
WARNING 2024-01-01 10:05:00 High memory usage detected
INFO  2024-01-01 10:10:00 User varun logged in
ERROR 2024-01-01 10:15:00 Database connection lost
INFO  2024-01-01 10:15:30 Retrying database connection
ERROR 2024-01-01 10:16:00 Retry failed - connection timeout
INFO  2024-01-01 10:16:30 Switching to backup database
INFO  2024-01-01 10:17:00 Backup database connected
WARNING 2024-01-01 10:20:00 Disk usage above 80 percent
INFO  2024-01-01 10:25:00 Scheduled backup started
INFO  2024-01-01 10:30:00 Backup completed successfully
ERROR 2024-01-01 10:35:00 API rate limit exceeded
INFO  2024-01-01 10:40:00 Server running normally
EOF
```

---

### Command 1 — cat (Concatenate)

```bash
cat server.log                        # print entire file
cat -n server.log                     # show with line numbers
cat file1.txt file2.txt               # print both files one after another
cat file1.txt file2.txt > combined.txt  # join into one file
cat -A server.log                     # show hidden characters (debug config files)
```

> ⚠️ For very large files — cat dumps everything at once and your terminal floods. Use `less` for large files.

---

### Command 2 — less (Read Large Files)

Opens a file in a scrollable viewer. Stay inside until you press `q` to quit.

```bash
less server.log
less /var/log/syslog
```

| Key | What it does |
|-----|-------------|
| Space or f | Forward one page |
| b | Back one page |
| Up/Down arrows | Scroll one line |
| g | Jump to beginning |
| G | Jump to end |
| /searchword | Search for a word |
| n | Next search result |
| N | Previous search result |
| q | Quit |

> ⚠️ Beginners panic when less looks different. Don't press Ctrl+C. Just press `q` to quit cleanly. Once inside, press `/ERROR` and hit Enter — it jumps to every error in the file.

---

### Command 3 — head (See Top of File)

```bash
head server.log          # first 10 lines (default)
head -n 5 server.log     # first 5 lines
head -n 20 server.log    # first 20 lines
head -n 1 data.csv       # see just the header row of a CSV
```

**When to use:** Checking what format a log file is in, seeing CSV headers, confirming a file has the right content at the top.

---

### Command 4 — tail (See Bottom of File)

Log files are written top to bottom chronologically. The newest entries are always at the bottom. `tail` shows the most recent activity.

```bash
tail server.log          # last 10 lines (default)
tail -n 5 server.log     # last 5 lines
tail -n 20 server.log    # last 20 lines
```

---

### Command 5 — tail -f (Follow Live Updates)

**This is one of the most used commands in real DevOps work.**

`-f` means follow — keeps the file open and prints new lines as they are written in real time.

```bash
tail -f server.log
tail -f /var/log/nginx/error.log
```

**Real DevOps situation:** You deployed a new version and want to watch for errors:
```bash
tail -f /var/log/nginx/error.log
```
You sit and watch. If an error appears, you see it immediately. Press `Ctrl+C` to stop following.

---

### Command 6 — grep (Search Inside Files)

Searches for a word or pattern and prints only matching lines. Out of 15 lines, grep finds exactly the ones you need. On a million-line log file — this saves hours.

```bash
grep "ERROR" server.log           # case sensitive search
grep -i "error" server.log        # case insensitive — finds ERROR, Error, error
grep -n "ERROR" server.log        # show line numbers
grep -c "ERROR" server.log        # count matching lines (just prints a number)
grep -v "INFO" server.log         # show everything EXCEPT lines with INFO
grep -r "ERROR" ~/linux_practice/ # search inside ALL files recursively
```

**Combining grep with pipe:**
```bash
cat server.log | grep "ERROR"
tail -n 50 server.log | grep "ERROR"   # search only last 50 lines
grep -i "error" server.log | grep -v "retry"  # errors but exclude retry lines
```

> ⚠️ **Most common grep mistake:** `grep error` finds nothing if log says `ERROR`. Grep is case sensitive by default. Always use `-i` if unsure about case.

**Real DevOps situation:** Your manager says "find all errors and warnings in the log":
```bash
grep -i "error" server.log
grep -i "warning" server.log
grep -c "error" server.log    # how many errors total?
grep -n "error" server.log    # which line numbers?
tail -n 5 server.log          # what was the last thing that happened?
```

---

### Full Command Summary — Day 3

| Command | What it does | Key flag |
|---------|-------------|---------|
| `cat file` | Print entire file | `-n` for line numbers |
| `less file` | Scroll through large file | `q` to quit, `/` to search |
| `head file` | First 10 lines | `-n 5` for first 5 |
| `tail file` | Last 10 lines | `-n 5` for last 5 |
| `tail -f file` | Watch file live in real time | Ctrl+C to stop |
| `grep "word" file` | Find matching lines | `-i` case insensitive, `-n` line numbers, `-v` exclude |

---

### Interview Questions — Day 3

**Q1. How do you monitor a log file in real time on a Linux server?**
Using `tail -f filename` — it continuously prints new lines added to the file.

**Q2. How do you search for "error" in a log ignoring case?**
`grep -i "error" filename` — the `-i` flag makes the search case insensitive.

**Q3. What is the difference between `head` and `tail`?**
`head` shows the first 10 lines. `tail` shows the last 10. Since logs are written chronologically, `tail` is more useful for seeing recent activity.

**Q4. You have a 2 million line log file. How do you open it without crashing the terminal?**
Use `less filename` — opens in a scrollable viewer without loading everything into memory at once.

**Q5. What does the `-v` flag do in grep?**
It inverts the match — shows all lines that do NOT contain the search word. Useful for filtering out noise from log files.

---

### Homework — Before Day 4

1. Run `grep "WARNING" server.log` — how many warnings?
2. Run `grep -c "INFO" server.log` — how many info lines?
3. Run `tail -n 3 server.log` — what are the last 3 events?
4. Run `head -n 3 server.log` — what are the first 3 events?
5. Run `grep -v "INFO" server.log` — what do you see and why?
6. Open `server.log` with `less` and search for ERROR using `/ERROR`

---

## Day 4 — File Permissions, Users and Groups

---

### Why This is Critical for DevOps

Most common interview topic for Linux in DevOps roles. Almost every interview has 2–3 questions on this. Permissions answer three questions for every file:
- What can YOU (the owner) do?
- What can your GROUP do?
- What can EVERYONE ELSE do?

---

### What You Will Learn Today

- Reading the permission string (rwxr-xr-x)
- Users and Groups
- chmod — changing permissions (numbers and letters)
- chown — changing ownership
- sudo — running as superuser

---

### Reading the Permission String

```
d  rwx  r-x  r-x
|   |    |    |
|   |    |    └── Others (everyone else on the system)
|   |    └─────── Group (members of the file's group)
|   └──────────── Owner (the person who owns the file)
└──────────────── Type: d=directory, -=file, l=symlink
```

| Letter | On a FILE means | On a DIRECTORY means |
|--------|----------------|---------------------|
| r (read) | Can open and read | Can run ls to see contents |
| w (write) | Can modify | Can create/delete files inside |
| x (execute) | Can run as a program | Can cd into it |
| - (none) | Permission not given | Permission not given |

> ⚠️ **Most confusing part:** Execute on a directory means permission to **enter** it with `cd`. Without `x` on a directory you cannot `cd` into it even if you can see it with `ls`.

---

### The Octal (Number) Permission System

Each permission has a number:

| Permission | Value |
|-----------|-------|
| r (read) | 4 |
| w (write) | 2 |
| x (execute) | 1 |
| - (none) | 0 |

Add them up for each group (owner, group, others):

| Combo | Calculation | Number |
|-------|------------|--------|
| rwx | 4+2+1 | 7 |
| rw- | 4+2+0 | 6 |
| r-x | 4+0+1 | 5 |
| r-- | 4+0+0 | 4 |
| --- | 0+0+0 | 0 |

**Common permission numbers:**

| Number | Permissions | Use case |
|--------|------------|---------|
| 755 | rwxr-xr-x | Scripts, directories |
| 644 | rw-r--r-- | Config files — others can read only |
| 600 | rw------- | SSH keys, secrets |
| 400 | r-------- | AWS .pem key files |
| 777 | rwxrwxrwx | Everyone full access — NEVER use in production |

---

### Command 1 — chmod (Change Mode)

```bash
chmod 755 script.sh              # set using numbers
chmod 644 config.yml             # config file — others read only
chmod 600 ~/.ssh/id_rsa          # private key — owner only
chmod u+x script.sh              # add execute for owner only
chmod g-w file.txt               # remove write from group
chmod o-r secret.txt             # remove read from others
chmod a+r file.txt               # add read for ALL (owner+group+others)
chmod -R 755 myfolder/           # change permissions recursively
```

**Symbolic method:**
```
u = user (owner)    + = add permission
g = group           - = remove permission
o = others          = = set exact permission
a = all (u+g+o)
```

---

### Command 2 — chown (Change Owner)

```bash
chown varun file.txt              # change owner to varun
chown varun:developers file.txt   # change owner AND group
chown -R varun myfolder/          # change owner recursively
```

> ⚠️ `chown` requires `sudo` — only root can change ownership.

---

### sudo — Superuser Do

```bash
sudo command                      # run command as root
sudo systemctl restart nginx      # restart nginx as root
whoami                            # show current username
id                                # show user ID and group memberships
sudo -i                           # start a root shell (be careful)
```

---

### Interview Questions — Day 4

**Q1. What does `chmod 755` mean?**
Sets permissions to `rwxr-xr-x` — owner has full read, write, execute. Group and others have read and execute only. Standard for scripts and directories.

**Q2. What is the difference between `chmod` and `chown`?**
`chmod` changes the permissions of a file. `chown` changes who owns the file.

**Q3. What does `sudo` do?**
Allows a permitted user to run a command with root privileges temporarily. Requires the user's own password.

**Q4. What permissions would you set on an SSH private key and why?**
`chmod 400` — read only for owner, no access for anyone else. SSH refuses to use the key if permissions are too open.

**Q5. What is execute permission on a directory?**
The ability to enter it using `cd`. Without execute permission you cannot navigate into the directory even if you can see it with `ls`.

**Q6. A file has permissions `rw-r--r--`. What is its octal value?**
644 — owner is rw (4+2=6), group is r (4), others is r (4).

---

### Homework — Before Day 5

1. Create `secret.txt` and set permissions so **only you can read it**
2. Create `shared/` and set permissions so **everyone can read and enter but only you can write inside**
3. Run `ls -l` and verify the permissions look correct
4. Run `id` and write down what groups you belong to
5. Decode this permission string: `rwxr-x---` — what is the octal number?

---

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

---

## Day 6 — Networking Commands

---

### Why This is Critical for DevOps

You deployed your application but users cannot access it. You check the app — it's running fine. So what's wrong? The problem could be: server not reachable, wrong port, firewall blocking, DNS not resolving, or service not listening on the right interface. Today's commands diagnose every one of these.

---

### Quick Concept — IP Addresses and Ports

**IP Address** — the address of a server on a network. Like a house address. Example: `192.168.1.10` or `13.234.56.78`

**Port** — a specific door on that house. Different services listen on different ports.

| Port | Service |
|------|---------|
| 22 | SSH — remote login |
| 80 | HTTP — websites |
| 443 | HTTPS — secure websites |
| 3306 | MySQL database |
| 5432 | PostgreSQL database |
| 6379 | Redis |
| 8080 | Common alternate web port |
| 27017 | MongoDB |

When you say `http://192.168.1.10:8080` — you mean "go to this IP and knock on door number 8080."

---

### What You Will Learn Today

- ping — is a server reachable?
- curl — make HTTP requests
- wget — download files
- ip / ifconfig — network interfaces
- ss / netstat — open ports
- nslookup / dig — DNS troubleshooting
- traceroute — trace network path

---

### Command 1 — ping

```bash
ping google.com          # ping continuously — press Ctrl+C to stop
ping -c 4 google.com    # send exactly 4 packets then stop
ping -i 2 google.com    # send one packet every 2 seconds
```

**Understanding ping output:**
```
PING google.com (142.250.67.46) 56(84) bytes of data.
64 bytes from 142.250.67.46: icmp_seq=1 ttl=118 time=12.3 ms
```

| Part | Meaning |
|------|---------|
| 142.250.67.46 | IP address Linux resolved for google.com |
| icmp_seq=1 | Packet number 1 |
| ttl=118 | How many routers this packet passed through |
| time=12.3 ms | Round trip time — how long the response took |

| Situation | What it means |
|-----------|--------------|
| ping works | Server is reachable at network level |
| ping fails | Server is down, unreachable, OR blocking ping |
| High response time 500ms+ | Network is slow or congested |
| Packet loss shown | Unstable network connection |

> ⚠️ Some servers block ping for security. Ping failing does NOT always mean the server is down.

**Real DevOps situation:** User says "I can't reach the website." First check:
```bash
ping yourserver.com
```
If ping fails — network level issue. If ping works — problem is at application level (wrong port, app crashed, firewall).

---

### Command 2 — curl (Make HTTP Requests)

Makes HTTP requests from the terminal. Like a browser but in text form. One of the most used tools in DevOps.

```bash
curl http://google.com               # fetch a webpage
curl -I http://google.com           # fetch ONLY headers (faster)
curl -o myfile.html http://google.com  # save output to file
curl -L http://google.com           # follow redirects automatically
curl -v http://google.com           # verbose — full request and response
curl -I http://localhost:8080/health  # check if app is responding
```

**HTTP status codes:**

| Code | Meaning |
|------|---------|
| 200 | OK — success |
| 301/302 | Redirect |
| 400 | Bad request |
| 401 | Unauthorized |
| 403 | Forbidden — permission denied |
| 404 | Not found |
| 500 | Internal server error |
| 502 | Bad gateway — upstream server issue |
| 503 | Service unavailable |

**Testing an API with curl:**
```bash
curl -X GET "https://api.example.com/users"
curl -X POST "https://api.example.com/users" \
     -H "Content-Type: application/json" \
     -d '{"name": "varun", "email": "varun@example.com"}'
```

**Check only the HTTP response code:**
```bash
curl -o /dev/null -s -w "%{http_code}" http://google.com
# Output: 200
```

---

### Command 3 — wget (Download Files)

```bash
wget https://example.com/file.zip              # download file
wget -O myname.zip https://example.com/file.zip  # save with custom name
wget -c https://example.com/largefile.zip     # resume interrupted download
wget -q https://example.com/file.zip          # quiet — no progress output
```

**curl vs wget:**

| Feature | curl | wget |
|---------|------|------|
| Primary use | Making HTTP requests, testing APIs | Downloading files |
| Resuming downloads | Manual | Built in with `-c` |
| Recursive download | No | Yes with `-r` |
| Output | Prints to screen by default | Saves to file by default |

**Real DevOps situation:** Install a tool on a server:
```bash
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform --version
```

---

### Command 4 — ip / ifconfig (Network Interfaces)

```bash
ip addr show           # show all network interfaces and IPs
ip addr show eth0      # show specific interface
ip route show          # show routing table
ifconfig               # older command — same purpose
```

Example output:
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 192.168.1.100  netmask 255.255.255.0  broadcast 192.168.1.255
lo:   flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
      inet 127.0.0.1  netmask 255.255.0.0
```

**Important interfaces:**

| Interface | Meaning |
|-----------|---------|
| eth0 or ens3 | Your main ethernet network card |
| lo | Loopback — always 127.0.0.1 — your machine talking to itself |
| wlan0 | Wireless interface |
| docker0 | Docker's virtual network interface |

`127.0.0.1` (localhost) — always refers to your own machine. When an app says "listening on localhost:8080" — only accessible from the same machine, not from outside.

---

### Command 5 — ss / netstat (Open Ports and Connections)

```bash
ss -tulnp                    # show all open ports and listeners
ss -tulnp | grep :8080       # check if port 8080 is in use
ss -tulnp | grep :80         # what is listening on port 80?
netstat -tulnp               # older command — same purpose
netstat -an                  # all connections with numbers
```

| Flag | Meaning |
|------|---------|
| -t | TCP connections |
| -u | UDP connections |
| -l | Show only listening ports |
| -n | Numbers instead of service names |
| -p | Show which process uses each port |

Example output:
```
Netid  State   Local Address:Port   Process
tcp    LISTEN  0.0.0.0:22           sshd
tcp    LISTEN  0.0.0.0:80           nginx
tcp    LISTEN  127.0.0.1:3306       mysqld
```

Reading this:
- SSH is listening on port 22 — accessible from anywhere (0.0.0.0)
- Nginx is on port 80 — accessible from anywhere
- MySQL is on port 3306 — only accessible locally (127.0.0.1)

**Real DevOps situation:** You deployed your app on port 8080 but users can't connect:
```bash
ss -tulnp | grep 8080
```
If nothing shows — app is not listening on that port (crashed or wrong port). If it shows `127.0.0.1:8080` — it's listening only locally. Change app config to listen on `0.0.0.0:8080`.

---

### Command 6 — nslookup / dig (DNS Lookup)

DNS converts domain names (google.com) to IP addresses (142.250.67.46). When DNS is broken, your domain stops working even if the server is perfectly fine.

```bash
nslookup google.com                # basic DNS lookup
dig google.com                     # detailed DNS lookup
dig google.com A                   # get IPv4 address record
dig google.com MX                  # get mail server records
dig @8.8.8.8 google.com           # query Google's DNS server specifically
```

**Real DevOps situation:** You updated DNS records but the site still shows old content:
```bash
dig yourwebsite.com
```
Check the IP in response. If still old IP — DNS hasn't propagated yet. Wait and check again. If shows new IP but site still broken — problem is on your server, not DNS.

---

### Command 7 — traceroute

Shows every router (hop) your packet passes through to reach a destination. Like GPS tracking for your network packet.

```bash
traceroute google.com
```

Example output:
```
1  192.168.1.1 (router)       1.234 ms
2  10.0.0.1 (ISP gateway)     5.678 ms
3  ...
15 142.250.67.46 (google)     12.3 ms
```

If traceroute stops at hop 5 — that router is where the problem is. You can tell your network team exactly where the issue is.

---

### Putting It All Together — Real Scenario

Your manager says: "Users in Bangalore can't access our website. Investigate."

```bash
ping yourwebsite.com             # is server reachable at all?
curl -I http://yourwebsite.com   # is web server responding?
nslookup yourwebsite.com         # is DNS resolving correctly?
traceroute yourwebsite.com       # where is the connection failing?
ss -tulnp | grep :80             # is nginx actually listening on port 80?
systemctl status nginx           # is nginx running?
```

Six commands. Complete picture. That's professional DevOps troubleshooting.

---

### Full Command Summary — Day 6

| Command | What it does | Key flag |
|---------|-------------|---------|
| `ping host` | Check if server is reachable | `-c 4` for 4 packets |
| `curl URL` | Make HTTP request | `-I` headers only, `-v` verbose |
| `wget URL` | Download file | `-c` resume, `-O` custom name |
| `ip addr show` | Show IP addresses | |
| `ifconfig` | Show network interfaces (older) | |
| `ss -tulnp` | Show open ports and listeners | `| grep port` to filter |
| `netstat -tulnp` | Same as ss (older) | |
| `nslookup domain` | Basic DNS lookup | |
| `dig domain` | Detailed DNS lookup | |
| `traceroute host` | Trace network path | |

---

### Interview Questions — Day 6

**Q1. How do you check if a remote server is reachable?**
Using `ping servername` — sends ICMP packets and shows if the server responds. Note: some servers block ping so a failed ping doesn't always mean the server is down.

**Q2. What is the difference between `curl` and `wget`?**
`curl` is primarily for making HTTP requests and testing APIs — outputs to screen by default. `wget` is primarily for downloading files — saves to disk by default and supports resuming interrupted downloads.

**Q3. How do you find which process is using port 3306?**
`ss -tulnp | grep 3306` — shows the process listening on that port.

**Q4. What is DNS and how do you troubleshoot a DNS issue?**
DNS converts domain names to IP addresses. To troubleshoot, use `nslookup` or `dig` to check what IP a domain resolves to and verify it matches the expected server IP.

**Q5. What is the difference between `0.0.0.0` and `127.0.0.1` when a service is listening?**
`0.0.0.0` means the service accepts connections from any network interface — publicly accessible. `127.0.0.1` means only from the same machine — not accessible from outside.

**Q6. A deployment went fine but the application is unreachable from outside. What do you check?**
Check if the app is listening on the right port with `ss -tulnp`. Check if it's on `0.0.0.0` not just `127.0.0.1`. Check firewall. Check DNS. Check service status with `systemctl status`.

---

### Homework — Before Day 7

1. Run `ping -c 4 google.com` — what is the average response time?
2. Run `curl -I http://google.com` — what HTTP status code do you get?
3. Run `ip addr show` — what is your machine's IP address?
4. Run `ss -tulnp` — list 3 services currently listening on your machine
5. Run `nslookup github.com` — what IP does it resolve to?

---

## Day 7 — Text Processing and Piping

---

### Why This is Critical for DevOps

You have a log file with 1 million lines and need to extract only the IP addresses. You have a CSV with 10,000 users and need to find duplicates. You need to replace a config value in 50 files at once. You cannot do this manually. These commands automate it in seconds.

**Interviews at 10–12 LPA often give you a log file and say "extract this data using the command line."** Today prepares you exactly for that.

---

### What You Will Learn Today

- The `|` pipe — connecting commands together
- wc — count lines, words, characters
- cut — extract specific columns
- sort — sort lines
- uniq — remove duplicates
- awk — column-based text processing
- sed — find and replace in files
- Chaining multiple commands for real tasks

---

### Setup — Create Practice Files

```bash
cd ~/linux_practice && mkdir day7 && cd day7

cat > access.log << EOF
192.168.1.10 GET /home 200 0.234
192.168.1.20 GET /about 200 0.123
192.168.1.10 POST /login 401 0.456
192.168.1.30 GET /home 200 0.321
192.168.1.20 GET /products 200 0.234
192.168.1.10 GET /home 200 0.111
192.168.1.40 GET /home 404 0.543
192.168.1.30 POST /login 200 0.678
192.168.1.20 GET /home 500 0.890
192.168.1.10 GET /products 200 0.234
192.168.1.50 GET /about 200 0.123
192.168.1.40 POST /login 200 0.456
EOF

cat > users.csv << EOF
name,department,salary
Varun,DevOps,85000
Priya,Development,90000
Ravi,DevOps,82000
Sneha,Testing,75000
Amit,Development,88000
Priya,Development,90000
Ravi,DevOps,82000
Kiran,Testing,76000
EOF
```

---

### The Pipe | — The Most Powerful Concept

A pipe `|` takes the **output of one command** and feeds it as **input to the next command.**

```bash
command1 | command2 | command3
```

Without pipe — you'd save output to a file, then read it, then process it. With pipe — it all happens in one line.

Think of it like a factory assembly line — each machine takes what the previous one produced.

```bash
cat access.log | grep "200"     # reads file → filters 200 lines
# Same as:
grep "200" access.log
```

Piping becomes powerful when you chain 3, 4, 5 commands together.

---

### Command 1 — wc (Word Count)

```bash
wc access.log           # lines, words, characters
wc -l access.log        # count lines only — most used
wc -w access.log        # count words only
wc -c access.log        # count characters/bytes only
```

**Real DevOps use with pipe:**
```bash
grep "ERROR" app.log | wc -l    # how many error lines?
ps aux | wc -l                   # how many processes are running?
```

---

### Command 2 — cut (Extract Columns)

```bash
cut -d',' -f1 users.csv         # first column of CSV
cut -d',' -f2 users.csv         # second column
cut -d',' -f1,3 users.csv       # first and third column
cut -d' ' -f1 access.log        # first column of space-separated log (IP)
```

| Flag | Meaning |
|------|---------|
| `-d','` | delimiter is comma |
| `-d' '` | delimiter is space |
| `-f1` | take field number 1 (first column) |
| `-f1,3` | take fields 1 and 3 |

**Extract all IP addresses from log:**
```bash
cut -d' ' -f1 access.log
# Output:
192.168.1.10
192.168.1.20
192.168.1.10
...
```

---

### Command 3 — sort

```bash
sort users.csv              # alphabetical sort
sort -r users.csv           # reverse alphabetical
sort -n numbers.txt         # numerical sort (important for numbers)
sort -rn numbers.txt        # reverse numerical — largest first
sort -k2 users.csv          # sort by column 2
sort -t',' -k3 -n users.csv # sort CSV by column 3 numerically
sort -u users.csv           # sort and remove duplicates
```

**Real DevOps use:** Sort IP addresses from log so same IPs are grouped together:
```bash
cut -d' ' -f1 access.log | sort
```

---

### Command 4 — uniq (Remove Duplicates)

> ⚠️ `uniq` only removes **consecutive** duplicates. **Always use after `sort`.**

```bash
sort users.csv | uniq           # unique list
uniq -c file.txt                # count how many times each line appears
uniq -d file.txt                # show only duplicate lines
uniq -u file.txt                # show only NON-duplicate lines
```

**Classic combo — find unique IPs:**
```bash
cut -d' ' -f1 access.log | sort | uniq
```

**Count how many requests each IP made:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
# Output:
      4 192.168.1.10
      3 192.168.1.20
      2 192.168.1.30
      2 192.168.1.40
      1 192.168.1.50
```

---

### Command 5 — awk (Column Processing)

awk processes text **column by column** — like a mini programming language built into Linux.

```
$1 = first column    $NF = last column (NF = Number of Fields)
$2 = second column   $0  = entire line
```

```bash
awk '{print $1}' access.log           # print first column (IP addresses)
awk '{print $1, $3}' access.log       # print columns 1 and 3
awk '{print $NF}' access.log          # print last column (response time)
awk '$4 == "200"' access.log          # show lines where column 4 = 200
awk '$4 == "404"' access.log          # show only 404 errors
awk '$4 >= "500"' access.log          # show 5xx server errors
awk -F',' '{print $1}' users.csv      # use comma as delimiter
awk -F',' '{print $1, $3}' users.csv  # print name and salary
```

**awk doing calculations:**
```bash
awk -F',' 'NR>1 {sum += $3} END {print "Total salary:", sum}' users.csv
# NR>1 = skip the first row (header)
# sum += $3 = add column 3 (salary) to running total
# END = do this after processing all lines
# Output: Total salary: 668000
```

**awk vs cut — when to use which:**

| Situation | Use |
|-----------|-----|
| Simple column extraction | `cut` — faster to type |
| Need conditions or math | `awk` — more powerful |
| Multiple columns with processing | `awk` |
| Just splitting by delimiter | `cut` |

---

### Command 6 — sed (Stream Editor)

Finds and replaces text in files or streams. Like Ctrl+H in a text editor but from the command line and much more powerful.

```bash
sed 's/old/new/' file.txt           # replace FIRST occurrence per line
sed 's/old/new/g' file.txt          # replace ALL (global)
sed 's/old/new/gi' file.txt         # replace all, case insensitive
sed 's/GET/HTTP-GET/g' access.log   # replace all GET with HTTP-GET
```

**sed does NOT change the file by default** — it prints modified output to screen. To actually modify the file:
```bash
sed -i 's/old/new/g' file.txt        # edit file IN PLACE (modifies file directly)
sed -i.bak 's/old/new/g' file.txt    # creates file.bak backup FIRST then modifies
```

> ⚠️ **Always use `-i.bak` on important files** — it creates a backup before changing. Never run `sed -i` on a production config without a backup.

**Deleting lines:**
```bash
sed '/ERROR/d' app.log              # delete all lines containing ERROR
sed '1d' users.csv                  # delete line 1 (header row)
sed '1,5d' file.txt                 # delete lines 1 through 5
```

**Real DevOps situation:** Update database host in config across servers:
```bash
sed -i.bak 's/db-old.company.com/db-new.company.com/g' /etc/app/config.yml
```
One command updates the config. The `.bak` backup means you can restore if something goes wrong.

---

### Chaining Everything Together — Real Pipeline Challenges

**Challenge 1 — Find unique IPs that got 404 errors:**
```bash
grep "404" access.log | cut -d' ' -f1 | sort | uniq
```
Step by step:
1. `grep "404"` — filter only 404 lines
2. `cut -d' ' -f1` — extract IP column
3. `sort` — group same IPs together
4. `uniq` — remove duplicates

**Challenge 2 — Count requests per IP, highest first:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
```

**Challenge 3 — Top 5 IPs with most requests:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -5
```

**Challenge 4 — Count total errors vs successes:**
```bash
grep "200" access.log | wc -l              # count successes
grep -E " [45][0-9][0-9] " access.log | wc -l  # count errors
```

**Challenge 5 — Replace all 500 errors with CRITICAL in a new file:**
```bash
sed 's/ 500 / CRITICAL /g' access.log > access_flagged.log
```

---

### Full Command Summary — Day 7

| Command | What it does | Key flag |
|---------|-------------|---------|
| `wc -l file` | Count lines | `-w` words, `-c` chars |
| `cut -d' ' -f1` | Extract column 1 | `-d` delimiter, `-f` field |
| `sort file` | Sort alphabetically | `-n` numeric, `-r` reverse, `-k` by column |
| `uniq file` | Remove consecutive duplicates | `-c` count, `-d` show duplicates |
| `awk '{print $1}'` | Print column 1 | `-F` delimiter, conditions |
| `sed 's/old/new/g'` | Find and replace | `-i` in-place, `-i.bak` with backup |
| `cmd1 | cmd2` | Pipe output to next command | Chain as many as needed |

---

### Interview Questions — Day 7

**Q1. How do you count the number of lines in a file?**
`wc -l filename`

**Q2. How do you extract the third column from a space-separated file?**
`cut -d' ' -f3 filename` or `awk '{print $3}' filename`

**Q3. Why must you sort before using uniq?**
Because `uniq` only removes **consecutive** duplicate lines. If duplicates are not adjacent it won't detect them. Sorting first brings all identical lines together.

**Q4. How do you find the top 3 most frequent values in a column?**
`cut -d' ' -f1 file | sort | uniq -c | sort -rn | head -3`

**Q5. How do you replace text in a file without opening an editor and create a backup?**
`sed -i.bak 's/oldtext/newtext/g' filename` — modifies in place and creates `.bak` backup automatically.

**Q6. What is the difference between `awk` and `cut`?**
`cut` is simpler — extracts columns by a fixed delimiter. `awk` is more powerful — can apply conditions, do calculations, handle variable whitespace, and process data like a programming language.

**Q7. How do you count how many times each unique value appears in a column?**
`cut -d' ' -f1 file | sort | uniq -c | sort -rn` — extracts the column, sorts it, counts unique values, and sorts by frequency.

---

### Homework — Before Day 8

Using `access.log` from today's practice:
1. How many total requests were made? (`wc -l`)
2. How many requests returned status 200? (`grep | wc -l`)
3. Which IP made the most requests? (`cut | sort | uniq -c | sort -rn | head -1`)
4. How many unique IPs visited? (`cut | sort | uniq | wc -l`)
5. What is the most requested URL? (column 3 — `cut | sort | uniq -c | sort -rn`)

---

## Day 8 — Shell Scripting

---

### Why This is Critical for DevOps

Until now you have been typing commands one at a time. Every day as a DevOps engineer you do the same tasks repeatedly: check if a service is running and restart if not, back up files every night, deploy new code, monitor disk space and alert if above 80%. Typing these manually every time is not DevOps. Writing a script that does it automatically — that is DevOps.

Shell scripting is where everything from Days 1–7 comes together into real automation.

---

### What You Will Learn Today

- Creating and running your first script
- Variables and command substitution
- Taking user input
- if-else conditions
- for loops
- while loops
- Functions
- Real deployment scripts
- exit 0 and exit 1 — why they matter in CI/CD

---

### Your First Shell Script

```bash
cd ~/linux_practice && mkdir day8 && cd day8
touch myfirst.sh
nano myfirst.sh
```

Type inside:
```bash
#!/bin/bash
echo "Hello Varun!"
echo "Today you are learning Shell Scripting"
echo "This is your path to 10-12 LPA"
```

Make executable and run:
```bash
chmod +x myfirst.sh
./myfirst.sh
```

**The Shebang `#!/bin/bash`** — tells Linux which shell to use to run the script. Always the first line of every script. Without it Linux may not know how to execute the file.

**Why `./` before the script name?** Linux searches system folders (PATH) for commands. Your script is not there — `./` means "look in the current folder."

---

### Variables

```bash
name="Varun"                       # NO spaces around = sign
city="Bangalore"
target="10-12 LPA"
echo "Hello $name from $city!"

# Command substitution — run command and store output
current_date=$(date)
current_user=$(whoami)
disk_usage=$(df -h / | tail -1 | awk '{print $5}')

echo "Date: $current_date"
echo "User: $current_user"
echo "Disk: $disk_usage"
```

> ⚠️ **No spaces around `=` sign ever** — this is the #1 beginner mistake.
> ```bash
> name="Varun"   # CORRECT
> name = "Varun" # WRONG — spaces break it
> ```

**Special variables:**
```bash
$0    # name of the script itself
$1    # first argument passed to the script
$2    # second argument
$@    # all arguments
$#    # number of arguments passed
$?    # exit status of last command (0=success, non-zero=failure)
$$    # PID of current script
```

**Example script using arguments:**
```bash
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Total arguments: $#"
```
Run: `./myscript.sh hello world`

**`$?` — Exit status:**
```bash
ls /home
echo $?       # prints 0 — command succeeded

ls /fakefolder
echo $?       # prints 2 — command failed (folder doesn't exist)
```

---

### Taking User Input

```bash
#!/bin/bash
read -p "Enter your name: " name
read -p "Enter your city: " city
echo "Hello $name from $city!"

# For passwords — hides what you type
read -sp "Enter password: " password
echo ""   # new line after hidden input
```

---

### if-else Conditions

```bash
if [ condition ]; then
    # commands if TRUE
elif [ condition ]; then
    # commands if this condition is TRUE
else
    # commands if FALSE
fi
```

**Number comparisons:** `-eq` (equal) `-ne` (not equal) `-gt` (greater than) `-lt` (less than) `-ge` (greater or equal) `-le` (less or equal)

**String comparisons:** `==` `!=` `-z` (is empty) `-n` (is not empty)

**File checks:** `-f` (file exists) `-d` (directory exists) `-r` (readable) `-w` (writable) `-x` (executable)

```bash
# Always spaces inside [ ] and quote variables
if [ "$name" == "varun" ]; then    # CORRECT
if [$name == "varun"]; then        # WRONG — no spaces inside brackets
if [ $name == "varun" ]; then      # RISKY — breaks if name is empty
```

**Real DevOps script — disk space alert:**
```bash
#!/bin/bash
threshold=80
disk_usage=$(df -h / | tail -1 | awk '{print $5}' | tr -d '%')

if [ $disk_usage -ge $threshold ]; then
    echo "WARNING: Disk at ${disk_usage}% — above threshold!"
    echo "Please clean up disk space immediately"
else
    echo "Disk at ${disk_usage}% — OK"
fi
```

**Grade checker with elif:**
```bash
#!/bin/bash
read -p "Enter your score: " score
if [ $score -ge 90 ]; then
    echo "Grade: A — Excellent!"
elif [ $score -ge 75 ]; then
    echo "Grade: B — Good"
elif [ $score -ge 60 ]; then
    echo "Grade: C — Average"
else
    echo "Grade: F — Need improvement"
fi
```

---

### for Loops

```bash
# Loop through a list
for fruit in apple banana mango orange; do
    echo "Fruit: $fruit"
done

# Loop through numbers
for i in {1..5}; do
    echo "Count: $i"
done

# Loop through files
for file in /var/log/*.log; do
    echo "Log file: $file"
done

# C-style loop
for ((i=1; i<=5; i++)); do
    echo "Count: $i"
done
```

**Real DevOps script — check multiple services:**
```bash
#!/bin/bash
services=("nginx" "mysql" "ssh" "docker")

for service in "${services[@]}"; do
    status=$(systemctl is-active $service)
    if [ "$status" == "active" ]; then
        echo "$service is RUNNING"
    else
        echo "$service is STOPPED — restarting..."
        sudo systemctl start $service
    fi
done
```

---

### while Loops

```bash
#!/bin/bash
counter=1
while [ $counter -le 5 ]; do
    echo "Count: $counter"
    counter=$((counter + 1))   # arithmetic in bash
done
```

**Arithmetic in bash — `$((expression))`:**
```bash
a=10
b=3
echo $((a + b))   # 13
echo $((a - b))   # 7
echo $((a * b))   # 30
echo $((a / b))   # 3 (integer division)
echo $((a % b))   # 1 (remainder)
```

**Real DevOps script — wait for service to start:**
```bash
#!/bin/bash
service="nginx"
max_attempts=10
attempt=1

echo "Waiting for $service to start..."

while [ $attempt -le $max_attempts ]; do
    status=$(systemctl is-active $service)
    if [ "$status" == "active" ]; then
        echo "$service is now running after $attempt attempt(s)"
        exit 0
    fi
    echo "Attempt $attempt/$max_attempts — $service not ready yet. Waiting..."
    sleep 5
    attempt=$((attempt + 1))
done

echo "ERROR: $service failed to start after $max_attempts attempts"
exit 1
```

This script is used in deployment pipelines constantly.

---

### Functions

A function is a named block of commands you can call multiple times. Instead of writing the same 5 commands in 3 places — write them once and call the function name.

```bash
function_name() {
    # commands
    # $1 = first argument passed to this function
}

# Call it:
function_name
function_name "argument"
```

**Real DevOps function — logging with timestamps:**
```bash
#!/bin/bash
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Script started"
log "Checking disk space..."
disk=$(df -h / | tail -1 | awk '{print $5}')
log "Disk usage: $disk"
log "Script completed"
```

Output:
```
[2024-03-20 10:30:01] Script started
[2024-03-20 10:30:01] Checking disk space...
[2024-03-20 10:30:01] Disk usage: 45%
[2024-03-20 10:30:01] Script completed
```

---

### exit 0 and exit 1 — Critical for CI/CD

| Command | Meaning |
|---------|---------|
| `exit 0` | Script finished successfully |
| `exit 1` | Script finished with an error |

**This matters in CI/CD pipelines.** If your deploy script exits with `1`, the pipeline knows deployment failed and stops. If it exits with `0`, it proceeds to the next step.

---

### Professional Error Handling

```bash
#!/bin/bash

# Exit immediately on any error
set -e

# Treat undefined variables as errors
set -u

# Pipe failures are caught
set -o pipefail

# All three together (most common in professional scripts)
set -euo pipefail

# Manual error handling
command || { echo "Command failed"; exit 1; }

# Trap — run cleanup on exit or error
cleanup() {
    rm -f /tmp/tempfile
    echo "Cleanup done"
}
trap 'echo "Script failed on line $LINENO"; cleanup' ERR
```

---

### Real Deployment Script

```bash
#!/bin/bash
set -euo pipefail

# Configuration
APP_NAME="mywebapp"
APP_DIR="/var/www/mywebapp"
BACKUP_DIR="/var/backups/mywebapp"
LOG_FILE="/var/log/deploy.log"

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

check_service() {
    if systemctl is-active --quiet $1; then
        log "$1 is running"
        return 0
    else
        log "ERROR: $1 is NOT running"
        return 1
    fi
}

# Main Script
log "===== Deployment started for $APP_NAME ====="

# Step 1: Check required services
log "Checking required services..."
for service in nginx mysql; do
    if ! check_service $service; then
        log "Starting $service..."
        sudo systemctl start $service
    fi
done

# Step 2: Backup current app
log "Creating backup..."
mkdir -p $BACKUP_DIR
cp -r $APP_DIR $BACKUP_DIR/backup_$(date '+%Y%m%d_%H%M%S')
log "Backup created successfully"

# Step 3: Check disk space before deploying
disk_usage=$(df -h / | tail -1 | awk '{print $5}' | tr -d '%')
if [ $disk_usage -ge 85 ]; then
    log "ERROR: Disk usage is ${disk_usage}%. Aborting deployment."
    exit 1
fi
log "Disk usage is ${disk_usage}% — OK to proceed"

# Step 4: Reload nginx
log "Reloading nginx..."
sudo systemctl reload nginx
log "Nginx reloaded successfully"

log "===== Deployment completed successfully ====="
exit 0
```

---

### Full Summary — Day 8

| Concept | Syntax | Example |
|---------|--------|---------|
| Shebang | `#!/bin/bash` | First line of every script |
| Variable | `name="value"` | `city="Bangalore"` |
| Use variable | `$name` | `echo $name` |
| Command substitution | `$(command)` | `date=$(date)` |
| User input | `read -p "prompt" var` | `read -p "Name: " name` |
| if condition | `if [ cond ]; then ... fi` | `if [ -f "file" ]; then` |
| for loop | `for x in list; do ... done` | loop through services |
| while loop | `while [ cond ]; do ... done` | wait for service |
| Function | `name() { ... }` | reusable log function |
| Arithmetic | `$((a + b))` | `counter=$((counter+1))` |
| Exit success | `exit 0` | script completed OK |
| Exit failure | `exit 1` | script failed |

---

### Interview Questions — Day 8

**Q1. What is the shebang line and why is it needed?**
`#!/bin/bash` — tells the OS which interpreter to use to run the script. Without it Linux may not know how to execute the file or may use the wrong shell.

**Q2. What is `$?` in shell scripting?**
It holds the exit status of the last executed command. 0 means success, any non-zero value means failure. Used to check if a command succeeded before proceeding.

**Q3. What is the difference between `$@` and `$#`?**
`$@` contains all arguments passed to the script. `$#` contains the count of how many arguments were passed.

**Q4. Why do you need spaces inside `[ ]` in if conditions?**
In bash, `[` is actually a command. It requires spaces to separate it from its arguments. Without spaces bash cannot parse the condition and you get a syntax error.

**Q5. What is command substitution?**
Using `$(command)` to capture the output of a command into a variable. Example: `disk=$(df -h / | tail -1 | awk '{print $5}')` stores the disk usage in the variable `disk`.

**Q6. What is the difference between `exit 0` and `exit 1`?**
`exit 0` means the script completed successfully. `exit 1` (or any non-zero) means it failed. CI/CD pipelines check the exit code to decide whether to continue or abort deployment.

**Q7. How do you pass arguments to a shell script?**
Pass them after the script name — `./script.sh arg1 arg2`. Inside the script, `$1` is the first argument, `$2` is the second, `$@` is all of them.

**Q8. What does `set -euo pipefail` do?**
`-e` exits immediately on any error. `-u` treats undefined variables as errors. `-o pipefail` catches pipe failures. Together they make scripts safer and easier to debug.

---

### Homework — Before Day 9

1. Write a script that asks for name and age, prints "Hello NAME, you are AGE years old"
2. Write a script that checks if `/etc/passwd` exists and prints whether it does or not
3. Write a script that prints numbers 1 to 10 using a for loop
4. Write a script that checks if nginx is running — if yes print "Running" — if no print "Stopped"
5. Write a script with a function called `check_disk` that prints current disk usage

---

## Day 9 — Cron Jobs

---

### Why This is Critical for DevOps

Your manager says "back up the database every night at midnight" and "send a disk usage report every morning at 8am". You cannot sit at your computer at midnight every day. Cron jobs do it automatically while you sleep. Every DevOps engineer sets up and manages cron jobs daily.

---

### What You Will Learn Today

- What cron is
- Crontab file — where jobs are defined
- Cron syntax — the 5 star format
- Creating, listing and removing cron jobs
- Special cron shortcuts (@daily, @reboot)
- Cron environment gotchas
- Real DevOps cron examples

---

### What is Cron?

Cron is a time-based job scheduler built into Linux. It runs in the background as a service (called `crond` or `cron`) and checks every minute — "is any job scheduled for right now?" If yes — it runs it automatically.

**Crontab = Cron Table** — a file where you write all your scheduled jobs. Each user has their own crontab file.

---

### Crontab Commands

```bash
crontab -e          # edit your crontab (opens in default editor)
crontab -l          # list all current cron jobs
crontab -u varun -l # list cron jobs for a specific user (needs sudo)
```

> ⚠️ **WARNING:** `crontab -r` DELETES ALL your cron jobs instantly — no confirmation. Many engineers have accidentally typed `-r` instead of `-e` and wiped all their scheduled tasks.

---

### The Cron Syntax — 5 Star Format

```
* * * * * command_to_run
| | | | |
| | | | └── Day of week (0-7, 0 and 7 = Sunday)
| | | └───── Month (1-12)
| | └──────── Day of month (1-31)
| └─────────── Hour (0-23)
└────────────── Minute (0-59)
```

**Memory trick:** "Monkeys Have Dreams Making Coffee" → Minute, Hour, Day(month), Month, Confidence(weekday)

**Special characters:**

| Character | Meaning | Example |
|-----------|---------|---------|
| `*` | Every / any value | `*` in hour = every hour |
| `,` | Multiple specific values | `1,15,30` = at minute 1, 15 and 30 |
| `-` | Range of values | `9-17` in hour = 9am to 5pm |
| `/` | Every N units | `*/15` in minute = every 15 minutes |

---

### Reading Cron Examples

```bash
* * * * *       # every minute
0 0 * * *       # every day at midnight (00:00)
0 8 * * *       # every day at 8am
30 8 * * *      # every day at 8:30am
0 3 * * 0       # every Sunday at 3am
0 18 * * 1,5    # every Monday and Friday at 6pm
*/15 * * * *    # every 15 minutes
0 * * * *       # every hour
0 */6 * * *     # every 6 hours
0 0 1 * *       # 1st of every month at midnight
0 9 * * 1-5     # every weekday (Mon-Fri) at 9am
*/30 * * * *    # every 30 minutes
0 9-17 * * 1-5  # every hour from 9am to 5pm, Monday to Friday
```

---

### Creating Cron Jobs

```bash
crontab -e
```

First time may ask which editor — choose nano (easiest for beginners).

At the bottom of the file add your job:
```bash
# This backs up home folder every day at midnight
0 0 * * * tar -czf /tmp/home_backup_$(date +\%Y\%m\%d).tar.gz /home/varun
```

Save and exit (nano: `Ctrl+X` → `Y` → `Enter`). Cron immediately activates your new job.

**Verify it was saved:**
```bash
crontab -l
```

---

### Real DevOps Cron Examples

```bash
# Daily disk space report at 8am
0 8 * * * df -h / >> /var/log/disk_report.log

# Delete logs older than 30 days every Sunday at 2am
0 2 * * 0 find /var/log -name "*.log" -mtime +30 -delete

# Database backup every night at 1am
0 1 * * * mysqldump -u root -pPASSWORD mydb > /backup/db_$(date +\%Y\%m\%d).sql

# Restart app every Sunday at 3am for maintenance (low traffic)
0 3 * * 0 sudo systemctl restart myapp

# Check nginx every 5 min — auto-restart if down
# || means "if left command fails, run right command"
*/5 * * * * systemctl is-active nginx || sudo systemctl start nginx

# Run shell script every hour — capture output and errors
0 * * * * /home/varun/scripts/disk_alert.sh >> /var/log/disk_alerts.log 2>&1
```

---

### Cron Environment Gotcha

**The Problem:** Cron does NOT load your normal shell environment. It runs with a very minimal environment — different PATH, no aliases, no user settings.

A command that works perfectly when you type it manually might **fail silently in a cron job.**

```bash
# Works manually:
python3 /home/varun/scripts/backup.py

# WRONG in cron — cron can't find python3:
0 8 * * * python3 /home/varun/scripts/backup.py

# CORRECT in cron — always use full paths:
0 8 * * * /usr/bin/python3 /home/varun/scripts/backup.py
```

**Find the full path of any command:**
```bash
which python3    # shows: /usr/bin/python3
which nginx      # shows: /usr/sbin/nginx
which mysqldump  # shows: /usr/bin/mysqldump
```

**Fix — set PATH at top of crontab:**
```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Always redirect output to a log file** (by default cron tries to email output — on most servers email isn't set up so output disappears silently):
```bash
# Save normal output only
0 8 * * * /home/varun/script.sh >> /var/log/myscript.log

# Save both output AND errors (recommended)
0 8 * * * /home/varun/script.sh >> /var/log/myscript.log 2>&1

# Discard all output (silent mode)
0 8 * * * /home/varun/script.sh > /dev/null 2>&1
```

`>>` appends to file (keeps history). `>` overwrites each time (only keeps latest).

---

### Special Cron Shortcuts

```bash
@reboot     # run once when system starts up
@hourly     # same as 0 * * * *
@daily      # same as 0 0 * * *
@weekly     # same as 0 0 * * 0
@monthly    # same as 0 0 1 * *
```

**@reboot — very useful for DevOps:**
```bash
@reboot sleep 30 && /home/varun/scripts/startup.sh
```
`sleep 30` waits 30 seconds after reboot — gives the system time to fully come up first.

---

### Interview Questions — Day 9

**Q1. How do you schedule a task to run every day at 3am?**
Add `0 3 * * * /path/to/script.sh` to crontab with `crontab -e`.

**Q2. Why do cron jobs sometimes fail even when commands work manually?**
Cron runs with a minimal environment — different PATH, no aliases, no user settings. Always use full paths to commands in cron jobs.

**Q3. What is `2>&1` in a cron job?**
Redirects stderr (file descriptor 2) to the same place as stdout (file descriptor 1). Ensures both regular output and error messages are captured in the log file.

**Q4. What is the difference between `cron` and `systemd timers`?**

| Feature | Cron | Systemd Timer |
|---------|------|--------------|
| Configuration | Crontab file | .timer and .service files |
| Logging | Manual with `>>` | Automatic via journald |
| Dependency handling | None | Can depend on other services |
| Missed job handling | Skips missed jobs | Can catch up missed jobs |
| Complexity | Simple | More complex but more powerful |

For most DevOps tasks, cron is sufficient and simpler.

**Q5. What does `*/15 * * * *` mean?**
Run every 15 minutes.

---

## Day 10 — User Management

---

### What You Will Learn Today

- Where users are stored (`/etc/passwd`, `/etc/shadow`, `/etc/group`)
- useradd / adduser — create users
- passwd — set passwords
- usermod — modify users
- userdel — delete users
- Group management
- su — switch users
- sudo and the sudoers file

---

### Where Users Are Stored

```bash
cat /etc/passwd          # user database — one line per user
sudo cat /etc/shadow     # encrypted passwords — only root can read
cat /etc/group           # group database
```

**/etc/passwd format:**
```
varun:x:1000:1000:Varun,,,:/home/varun:/bin/bash
  |   |  |    |      |          |          |
  |   |  |    |      |          |          └── Default shell
  |   |  |    |      |          └── Home directory
  |   |  |    |      └── Comment/Full name
  |   |  |    └── Primary group ID (GID)
  |   |  └── User ID (UID)
  |   └── Password (x = stored in /etc/shadow)
  └── Username
```

**Three types of users in Linux:**
1. Root user — UID 0, full system access
2. System users — UID 1–999, created by software (nginx, mysql, www-data)
3. Regular users — UID 1000+, actual humans

---

### Command 1 — useradd vs adduser

| Command | Type | What it does |
|---------|------|-------------|
| `useradd` | Low level | Creates user — minimal, needs flags for home folder |
| `adduser` | High level | Interactive, creates home folder, asks for password — friendlier |

`adduser` is actually a script that calls `useradd` behind the scenes with sensible defaults.

**In interviews and scripts — `useradd` is asked about more. In real daily work — `adduser` is easier.**

```bash
sudo useradd -m john                         # -m creates home folder
sudo useradd -m -s /bin/bash john            # set default shell to bash
sudo useradd -m -s /bin/bash -c "John Smith" john   # add full name
sudo useradd -m -s /bin/bash -G sudo john    # add to sudo group on creation
sudo adduser john                            # friendlier interactive way
```

> ⚠️ Without `-m` the user exists but has no home directory — they cannot log in properly. Always use `-m`.

---

### Command 2 — passwd (Set Password)

```bash
sudo passwd john        # set or change john's password
passwd                  # change YOUR OWN password (no sudo needed)
sudo passwd -l john     # lock account — john cannot login
sudo passwd -u john     # unlock account
sudo passwd -e john     # expire password — force change on next login
sudo passwd -d john     # delete password — account has no password
```

**Real DevOps situation:** New developer joins — force them to set their own password:
```bash
sudo useradd -m -s /bin/bash -c "New Developer" devuser
sudo passwd devuser          # set temporary password
sudo passwd -e devuser       # force change on first login
```

---

### Command 3 — usermod (Modify User)

```bash
sudo usermod -aG sudo john           # add john to sudo group
sudo usermod -aG docker john         # add john to docker group
sudo usermod -aG developers,docker varun  # add to multiple groups at once
sudo usermod -s /bin/zsh john        # change shell to zsh
sudo usermod -l newname john         # rename user
sudo usermod -L john                 # lock the account
sudo usermod -U john                 # unlock the account
sudo usermod -e 2024-12-31 john     # set account expiry date
```

> ⚠️ **Critical — always use `-aG` not `-G`**
> ```bash
> sudo usermod -G sudo john    # DANGEROUS — REPLACES all existing groups with just sudo
> sudo usermod -aG sudo john   # CORRECT — APPENDS sudo to existing groups
> ```
> Without `-a` (append), usermod removes the user from ALL other groups.

---

### Command 4 — userdel (Delete User)

```bash
sudo userdel john       # delete user but KEEP home folder
sudo userdel -r john    # delete user AND home folder and mail spool
find / -user john 2>/dev/null  # find all files owned by john FIRST
```

**Best practice — don't just delete immediately:**
```bash
sudo passwd -l john     # lock first
# wait a few weeks and confirm nothing breaks
sudo userdel -r john    # then delete
```

---

### Command 5 — Group Management

```bash
sudo groupadd developers          # create a new group
sudo groupadd -g 1500 developers  # create group with specific GID
sudo groupdel developers          # delete a group
groups                            # show YOUR groups
groups varun                      # show varun's groups
id varun                          # show UID, GID and all groups
```

**Real DevOps situation:** Developers need access to `/var/www/html` but not system configs:
```bash
sudo groupadd developers
sudo groupadd devops

sudo usermod -aG developers priya
sudo usermod -aG developers ravi
sudo usermod -aG devops varun

# Set ownership of web folder
sudo chown -R root:developers /var/www/html
sudo chmod -R 775 /var/www/html
# Now developers can read and write, others can only read
```

---

### su — Switch User

```bash
su john         # switch to john (keeps current environment)
su - john       # switch to john WITH their full environment (preferred)
sudo su         # become root using YOUR sudo password
sudo su - john  # switch to john as root (no password needed)
exit            # go back to previous user (or Ctrl+D)
```

**Always use `su -` when you want to fully become that user.** Without `-` you might have wrong paths and settings.

---

### sudo — Deep Dive

```bash
sudo visudo     # ALWAYS use visudo to edit — validates before saving
```

**Never edit `/etc/sudoers` directly** — a syntax error locks everyone out of sudo.

**sudoers file format:**
```
# Format: user host=(run_as_user) commands
varun ALL=(ALL:ALL) ALL             # varun can run any command as any user
john ALL=(ALL) /usr/bin/nginx       # john can ONLY run nginx with sudo
priya ALL=(ALL) NOPASSWD: ALL      # priya can sudo without password
```

**Easiest way to give sudo access:**
```bash
sudo usermod -aG sudo username      # Ubuntu
sudo usermod -aG wheel username     # CentOS
```

---

### Interview Questions — Day 10

**Q1. What is the difference between `useradd` and `adduser`?**
`useradd` is a low-level command requiring explicit flags. `adduser` is a friendlier script that calls `useradd` with sensible defaults and prompts interactively.

**Q2. Why must you use `-aG` instead of `-G` with usermod?**
`-G` alone replaces ALL existing group memberships. `-aG` appends — adds to existing groups without removing others.

**Q3. How do you create a user who must change their password on first login?**
`sudo passwd -e username` — expires the password immediately, forcing a change on next login.

**Q4. How do you see who is logged in to the server?**
```bash
who        # who is currently logged in
w          # who is logged in and what they are doing
last       # login history
last varun # login history for specific user
lastlog    # last login for all users
```

**Q5. What is the sudoers file and why must you use visudo to edit it?**
`/etc/sudoers` controls who can use sudo and what they can run. `visudo` validates syntax before saving — a syntax error in the file would lock everyone out of sudo.

---

## Day 11 — Package Management

---

### What is Package Management?

Installing, updating, and removing software on Linux. Package managers handle downloading, installing, and resolving dependencies automatically.

---

### What You Will Learn Today

- APT — Ubuntu and Debian
- YUM / DNF — CentOS and RHEL
- APT vs YUM comparison
- Adding third-party repositories
- Installing from other sources (.deb, snap)
- Real DevOps scenarios

---

### APT — Ubuntu and Debian

> ⚠️ Always run `sudo apt update` **before** installing anything — refreshes the package list without it you might install an old version or get "package not found" errors.

```bash
sudo apt update                     # fetch latest package list (ALWAYS do first)
sudo apt upgrade                    # upgrade all installed packages
sudo apt install nginx              # install a package
sudo apt install nginx curl git     # install multiple at once
sudo apt install -y nginx           # auto-confirm without typing Y
sudo apt remove nginx               # remove package (keeps config files)
sudo apt purge nginx                # remove package AND all config files
sudo apt autoremove                 # remove unused dependency packages
apt search nginx                    # search for a package
apt show nginx                      # show package details — version, size, deps
apt list --installed                # list all installed packages
apt list --installed | grep nginx   # check if specific package is installed
dpkg -l nginx                       # another way to check — shows version too
```

**`apt remove` vs `apt purge`:**
- `remove` — keeps config files (use if you might reinstall later)
- `purge` — clean uninstall, removes everything including config (fresh start)

**`apt upgrade` vs `apt full-upgrade`:**
- `upgrade` — upgrades packages but never removes existing ones (safer)
- `full-upgrade` — upgrades and removes packages if needed for dependencies
- On production always use `apt upgrade` — safer.

---

### YUM / DNF — CentOS and RHEL

```bash
sudo yum update                 # update all packages
sudo yum install nginx          # install package
sudo yum remove nginx           # remove package
sudo yum search nginx           # search for package
yum info nginx                  # show package details
yum list installed              # list installed packages
yum list installed | grep nginx # check if installed

sudo dnf install nginx          # dnf = modern replacement for yum
sudo dnf remove nginx
sudo dnf search nginx
```

---

### APT vs YUM Side by Side

| Task | APT (Ubuntu) | YUM (CentOS) |
|------|-------------|-------------|
| Refresh list | `apt update` | `yum check-update` |
| Install | `apt install pkg` | `yum install pkg` |
| Remove | `apt remove pkg` | `yum remove pkg` |
| Search | `apt search pkg` | `yum search pkg` |
| Show info | `apt show pkg` | `yum info pkg` |
| List installed | `apt list --installed` | `yum list installed` |
| Update all | `apt upgrade` | `yum update` |

---

### Adding a Third-Party Repository (Docker Example)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker GPG key (verifies packages are genuine)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

**What is a GPG key?** A digital signature that proves packages from a repository are genuine and not tampered with. Ubuntu refuses to install packages from a repo without a valid GPG key.

---

### Real DevOps Scenarios

**Scenario 1 — Setting Up a Fresh Server:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
    git curl wget vim htop net-tools unzip tree jq
echo "Server setup complete"
```

**Scenario 2 — Install and Verify Nginx:**
```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
systemctl status nginx
curl -I http://localhost     # verify it is responding
```

**Scenario 3 — Hold a Package Version (prevent upgrades):**
```bash
sudo apt-mark hold mysql-server   # prevent upgrades
sudo apt-mark unhold mysql-server # allow upgrades again
```

Used when a new version might break your application.

---

### Interview Questions — Day 11

**Q1. What does `sudo apt update` do?**
Downloads the latest list of available packages from repositories. Does NOT install anything — just refreshes the list.

**Q2. What is the difference between `apt remove` and `apt purge`?**
`remove` uninstalls but keeps config files. `purge` removes everything including config — use for a clean uninstall.

**Q3. Why do you always run `apt update` before `apt install`?**
Without it you might install an old version or get "package not found" errors because the local package list is outdated.

**Q4. How do you install a package without interactive confirmation in a script?**
Using the `-y` flag: `sudo apt install -y packagename`. Essential when writing automation scripts that cannot wait for interactive input.

**Q5. What is a repository in Linux package management?**
A remote server that stores packages. When you run `apt install`, Linux downloads the package from the configured repositories. Third-party software like Docker and Kubernetes provide their own repositories.

**Q6. How do you prevent a specific package from being upgraded?**
`sudo apt-mark hold packagename` — locks the package at its current version.

**Q7. What is the difference between `apt` and `yum`?**
Both are package managers but for different Linux distributions. `apt` is used on Ubuntu and Debian-based systems. `yum` (and its modern replacement `dnf`) is used on CentOS and RHEL-based systems.

---

## Day 12 — File Compression & Archives

---

### Archiving vs Compression

| Concept | What it does | Example |
|---------|-------------|---------|
| Archiving | Bundles multiple files into ONE | tar — combines 100 files into 1 |
| Compression | Reduces file size | gzip — makes a file smaller |
| Both together | Bundle AND shrink | tar.gz — combine and compress |

Think of archiving like putting files into a box. Compression is vacuum-sealing that box to make it smaller.

---

### tar — The Most Important Command

**Three commands you will use 90% of the time:**
```bash
tar -czf archive.tar.gz foldername/   # CREATE
tar -xzf archive.tar.gz              # EXTRACT
tar -tzf archive.tar.gz              # LIST contents without extracting
```

**Memory tricks:**
- Create: **c**reate g**z**ip **f**ile = `czf`
- Extract: e**x**tract g**z**ip **f**ile = `xzf`

**All tar flags:**

| Flag | Meaning |
|------|---------|
| c | Create new archive |
| x | Extract files |
| t | List contents |
| f | Specify filename (always needed) |
| v | Verbose — show files being processed |
| z | Compress with gzip (.tar.gz) |
| j | Compress with bzip2 (.tar.bz2) |
| J | Compress with xz (.tar.xz) |
| C | Extract to specific directory |

**More tar examples:**
```bash
tar -czf backup_$(date +%Y%m%d).tar.gz /home/varun/   # date in filename
tar -czvf backup.tar.gz folder/                        # verbose — see progress
tar -czf backup.tar.gz /home/ --exclude="*.log"        # exclude log files
tar -czf backup.tar.gz /home/ --exclude=".git"         # exclude .git
tar -xzf backup.tar.gz -C /tmp/restore/               # extract to specific folder
tar -xzf backup.tar.gz home/varun/file.txt             # extract ONE file only
tar -tzvf backup.tar.gz                                # list with details
```

> ⚠️ **Common tar mistakes:**
> ```bash
> tar -czv backup.tar.gz folder/    # WRONG — forgot -f flag
> tar -czvf backup.tar.gz folder/   # CORRECT
>
> tar -xzf backup.tar.gz /tmp/restore/   # WRONG — /tmp/ interpreted as file
> tar -xzf backup.tar.gz -C /tmp/restore/  # CORRECT — -C specifies destination
> ```

---

### Different Compression Formats

```bash
# .tar.gz — most common, good speed and compression
tar -czf archive.tar.gz folder/   # create
tar -xzf archive.tar.gz           # extract

# .tar.bz2 — better compression, slower speed
tar -cjf archive.tar.bz2 folder/  # create
tar -xjf archive.tar.bz2          # extract

# .tar.xz — best compression, slowest speed (used for software releases)
tar -cJf archive.tar.xz folder/   # create
tar -xJf archive.tar.xz           # extract
```

**Which format to use:**

| Format | Use when |
|--------|---------|
| .tar.gz | Default choice — good balance of speed and size |
| .tar.bz2 | Need smaller file, have time to wait |
| .tar.xz | Maximum compression — software releases |
| .zip | Sharing with Windows users |

---

### gzip — Compress Single Files

```bash
gzip file.txt          # compress — ORIGINAL IS DELETED
gzip -k file.txt       # compress but KEEP original (-k = keep)
gzip -d file.txt.gz    # decompress — same as gunzip
gunzip file.txt.gz     # decompress
gzip -l file.txt.gz    # show compression ratio and sizes
gzip -9 file.txt       # maximum compression (slower)
gzip -1 file.txt       # fastest compression (larger file)
```

> ⚠️ `gzip` deletes the original by default. Use `-k` to keep both original and compressed.

**Read gzipped files without extracting:**
```bash
zcat file.txt.gz       # like cat but for gzip files
zless file.txt.gz      # scroll through without extracting
zgrep "ERROR" file.gz  # search inside without extracting
```

Very useful — you can read compressed log files without extracting them first. Saves time and disk space.

---

### zip — Windows Compatible

```bash
zip -r archive.zip folder/     # create zip
unzip archive.zip              # extract zip
unzip -l archive.zip           # list contents
unzip -d /tmp/ archive.zip     # extract to specific directory
```

---

### Real DevOps Scenarios

**Transfer files between servers:**
```bash
# On source server — pack everything
tar -czf /tmp/transfer.tar.gz /var/www/myapp/

# Transfer to another server
scp /tmp/transfer.tar.gz user@192.168.1.20:/tmp/

# On destination server — extract
tar -xzf /tmp/transfer.tar.gz -C /var/www/
```

**Compress old logs to save disk space:**
```bash
find /var/log/nginx/ -name "*.log" -mtime +7 -exec gzip {} \;

# Or as a cron job (combine with Day 9):
0 3 * * 0 find /var/log/nginx/ -name "*.log" -mtime +7 -exec gzip {} \;
```

---

### Interview Questions — Day 12

**Q1. What is the difference between archiving and compression?**
Archiving combines multiple files into one without reducing size. Compression reduces file size. `tar.gz` does both — tar archives and gzip compresses.

**Q2. What does `tar -czf` mean?**
`c` = create new archive, `z` = compress with gzip, `f` = the next argument is the filename.

**Q3. How do you extract a tar.gz to a specific directory?**
`tar -xzf archive.tar.gz -C /path/to/directory/` — the `-C` flag specifies destination.

**Q4. What is the difference between `.tar.gz` and `.zip`?**
Both archive and compress. `tar.gz` is the Linux standard, preserves file permissions fully and has slightly better compression. `zip` is Windows compatible and more universal for cross-platform sharing. On Linux servers always use `tar.gz`.

**Q5. How do you read a gzipped log file without extracting it?**
Using `zcat file.gz`, `zless file.gz`, or `zgrep "pattern" file.gz` — all work directly on gzip files without extracting.

**Q6. What does `gzip -k` do?**
The `-k` flag keeps the original file after compression. Without `-k`, gzip deletes the original.

**Q7. How do you list the contents of a tar.gz without extracting?**
`tar -tzf archive.tar.gz` — `t` = list contents, `z` = gzip, `f` = filename.

---

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

## Day 14 — Vim Text Editor

---

### Why Vim

Every server has Vim pre-installed. When you SSH into a server at 3am to fix an issue, Vim is what is available. Nano is simpler but Vim is faster once learned. Also — "How do you exit Vim?" is a classic interview question.

---

### What You Will Learn Today

- Understanding Vim modes — the key concept
- Opening and closing files
- Navigation without arrow keys
- Inserting and editing text
- Saving files
- Searching and replacing
- Copy, cut and paste
- Vim in real DevOps config editing

---

### The Single Most Important Concept — Vim Modes

Vim has modes. When you open it you **cannot just start typing.**

| Mode | What it is | How to enter |
|------|-----------|-------------|
| Normal mode | Default — for navigation and commands | Press `Esc` |
| Insert mode | For typing and editing text | Press `i`, `a`, or `o` |
| Command mode | For saving, quitting, searching | Press `:` from Normal |
| Visual mode | For selecting text | Press `v` |

**The most common beginner mistake:** You open Vim and start typing. Text appears in random places. You panic. **Why:** You are in Normal mode. Vim thinks your keystrokes are commands.

**Golden rule:** When confused — press `Esc` multiple times to get back to Normal mode. From Normal mode you can always recover.

---

### Opening and Closing Vim

```bash
vim filename.txt          # open existing file or create new
vim /etc/nginx/nginx.conf # open a system config file
sudo vim /etc/hosts       # open system file with root permissions
```

**Closing Vim — the classic interview question:**
```bash
:q          # quit — only works if no changes were made
:q!         # quit WITHOUT saving — force quit, discard all changes
:w          # save but stay in vim
:wq         # save AND quit
:wq!        # save and quit forcefully
:x          # same as :wq — save and quit
ZZ          # save and quit (no colon needed, in Normal mode)
ZQ          # quit without saving (no colon needed)
```

---

### Navigation in Normal Mode

```bash
h j k l         # left down up right (arrow keys also work)
w / b           # jump to next / previous word
e               # jump to END of current word
0 / $           # jump to start / end of line
^               # jump to first non-blank character of line
gg / G          # go to FIRST / LAST line of file
50G             # go to line 50
:50             # go to line 50 (command mode)
Ctrl+f / b      # page forward / backward
Ctrl+d / u      # half page down / up
```

**Memory trick for hjkl:** h is leftmost, l is rightmost, j has a tail going down, k goes up.

**Real DevOps navigation:** Open huge config file and go straight to what you need:
```bash
G          # jump to bottom
gg         # jump to top
/searchterm  # search for what you need
```

---

### Entering Insert Mode

```bash
i    # insert BEFORE cursor
a    # insert AFTER cursor (append)
I    # insert at BEGINNING of line
A    # insert at END of line (very common for editing config lines)
o    # open new line BELOW and enter insert mode
O    # open new line ABOVE and enter insert mode
```

**Workflow for editing a config file:**
1. Navigate to the line (hjkl or search with `/`)
2. Press `i` to enter Insert mode
3. Make your changes
4. Press `Esc` to return to Normal mode
5. Type `:wq` to save and quit

---

### Editing — Delete, Undo, Redo

```bash
x         # delete ONE character under cursor
dd        # delete ENTIRE current line
5dd       # delete 5 lines from cursor
dw        # delete from cursor to end of word
d$        # delete from cursor to end of line
d0        # delete from cursor to beginning of line
dG        # delete from current line to end of file
u         # undo last change
Ctrl+r    # redo — undo the undo
5u        # undo last 5 changes
r         # replace ONE character (stay in normal mode)
cw        # change word (delete word + enter insert mode)
cc        # change entire line
```

---

### Copy, Cut, Paste

In Vim: Copy = Yank, Cut = Delete (deleted text goes to clipboard), Paste = Put

```bash
yy        # copy (yank) current line
5yy       # copy 5 lines
yw        # yank from cursor to end of word
y$        # yank from cursor to end of line
p         # paste AFTER cursor / BELOW current line
P         # paste BEFORE cursor / ABOVE current line
dd        # cut current line
5dd       # cut 5 lines
```

**Practical copy-paste:** Duplicate a config block in nginx:
1. Navigate to first line of block
2. Count lines — say it is 5 lines
3. Press `5yy` — copies 5 lines
4. Navigate to where you want to paste
5. Press `p` — pastes below current line
6. Edit the pasted block as needed

---

### Searching and Replacing

```bash
/searchterm          # search FORWARD — press Enter
?searchterm          # search BACKWARD — press Enter
n                    # jump to NEXT match
N                    # jump to PREVIOUS match

# Replace in current line:
:s/old/new/          # replace FIRST occurrence on current line
:s/old/new/g         # replace ALL on current line

# Replace in entire file:
:%s/old/new/         # replace first occurrence on each line
:%s/old/new/g        # replace ALL in entire file
:%s/old/new/gc       # replace all but ASK confirmation each time
:%s/old/new/gi       # replace all, case insensitive

# Replace in a range of lines:
:5,20s/old/new/g     # replace between lines 5 and 20
```

---

### Full Vim Quick Reference

```bash
# Modes
Esc           # Normal mode — press when confused
i             # Insert mode before cursor
a             # Insert mode after cursor
o             # Insert mode, new line below
:             # Command mode

# Save/Quit
:w            # save
:q!           # quit without saving
:wq or :x     # save and quit
ZZ            # save and quit (no colon)

# Navigation
h j k l       # left down up right
gg / G        # top / bottom of file
0 / $         # line start / end
:50           # go to line 50

# Edit
dd            # delete line
yy            # copy line
p / P         # paste below / above
u / Ctrl+r    # undo / redo
r             # replace one character
cw            # change word
.             # repeat last change

# Search/Replace
/term         # search forward
n / N         # next / previous
:%s/old/new/g # replace all in file
:%s/old/new/gc # replace all with confirmation
```

---

### Real Scenario

Your manager says: "Change nginx port from 8080 to 9090 everywhere in the config."

```bash
sudo vim /etc/nginx/nginx.conf
# Inside Vim:
/8080                       # verify it exists — press Enter
n                           # check all occurrences
:%s/8080/9090/gc            # replace all with confirmation
:wq                         # save and quit
sudo systemctl reload nginx  # apply changes — no downtime
```

---

### Interview Questions — Day 14

**Q1. How do you exit Vim without saving?**
Press `Esc` to ensure Normal mode, then `:q!` and Enter.

**Q2. What are the main modes in Vim?**
Normal (navigation/commands), Insert (typing), Command (save/quit), Visual (selecting text).

**Q3. How do you search and replace all occurrences in a file?**
`:%s/oldword/newword/g` — `%` = entire file, `s` = substitute, `g` = global (all occurrences).

**Q4. How do you go to a specific line number?**
Type `:50` in command mode, or `50G` in Normal mode.

**Q5. What is the difference between `dd` and `yy`?**
`dd` deletes the current line and puts it in the clipboard (cut). `yy` copies the current line without deleting it. Both can be pasted with `p`.

**Q6. How do you save a file you opened without sudo but need root to save?**
`:w !sudo tee %` — writes the file using sudo through tee. Common trick when you forget to open with sudo.

**Q7. What does the `.` command do in Vim?**
Repeats the last change. Extremely useful for applying the same edit to multiple locations — navigate and press `.` each time.

**Q8. How do you comment out multiple lines at once?**
Press `Ctrl+v`, select lines with `j`, press `I` (capital i), type `#`, press `Esc`. The `#` appears on all selected lines simultaneously.

---

### Homework — Before Day 15

1. Open `vim day14_practice.txt` in `~/linux_practice`
2. Enter Insert mode and type 10 lines of any text
3. Press Esc and navigate using hjkl only — no arrow keys
4. Go to line 5 using `:5`
5. Delete line 3 with `dd`
6. Copy line 2 with `yy` and paste below line 7 with `p`
7. Search for a word using `/word`
8. Replace all occurrences of a word using `:%s/old/new/g`
9. Save and quit with `:wq`
10. Open again and quit WITHOUT saving using `:q!`

---

## Day 15 — Environment Variables & .bashrc

---

### Why This is Critical for DevOps

Your application needs a database password — you cannot hardcode it in code because it goes into Git and everyone can see it. The same script needs to work differently on dev vs production. Docker containers need configuration passed to them. CI/CD pipelines like Jenkins and GitHub Actions use environment variables for secrets. Environment variables solve all of these.

---

### What You Will Learn Today

- What environment variables are
- Viewing, setting and unsetting variables
- Local vs exported variables
- Important system variables (PATH, HOME, etc.)
- Making variables permanent with `.bashrc`
- Aliases — command shortcuts
- `.env` files — used in every real project
- Environment variables in Docker and CI/CD

---

### What is an Environment Variable?

A named value stored in the shell that programs and scripts can read. Think of it as a global setting any program on your system can access.

Examples: your home directory, username, where commands are installed, app passwords, which environment you are running in (dev/prod).

---

### Viewing Environment Variables

```bash
env                  # show all environment variables
printenv             # same as env
printenv HOME        # show specific variable
echo $HOME           # another way — using $
echo $USER           # your username
echo $PATH           # where Linux looks for commands
echo $SHELL          # which shell you are using
echo $PWD            # current directory — same as pwd command
echo $HOSTNAME       # machine hostname
echo $LANG           # system language setting
```

---

### Setting Variables

```bash
# Local variable — current shell only, child processes cannot see
name="Varun"
city="Bangalore"
echo $name

# Exported variable — available to child processes and scripts
export name="Varun"
export DB_HOST="localhost"
export DB_PORT="5432"

# Set for ONE command only — then it disappears
DB_HOST="production.db.com" python3 app.py
ENVIRONMENT="production" ./deploy.sh

# Unset a variable
unset name
echo $name    # prints nothing
```

**The difference — local vs exported:**
```bash
greeting="Hello"          # local variable
export language="Python"  # exported variable

# Run a script — the script can only see 'language', not 'greeting'
./myscript.sh
```

---

### The PATH Variable — Most Important

PATH is a list of directories separated by `:` where Linux looks for commands when you type them.

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/varun/.local/bin
```

When you type `nginx`, Linux searches each PATH directory left to right until it finds the executable. Not found anywhere = "command not found" error.

**Adding to PATH temporarily:**
```bash
export PATH=$PATH:/home/varun/scripts
# Now scripts in /home/varun/scripts work without full path
```

> ⚠️ **Critical mistake:**
> ```bash
> export PATH=/home/varun/scripts   # WRONG — REPLACES entire PATH
> # Now ls, cd, everything breaks — commands not found!
>
> export PATH=$PATH:/home/varun/scripts  # CORRECT — APPENDS
> ```

**Why cron jobs fail because of PATH:** Cron runs with a minimal PATH. Commands you installed in `/usr/local/bin` may not be in cron's PATH — now you understand exactly why (from Day 9).

---

### Making Variables Permanent — .bashrc

Every variable you set with `export` disappears when you close the terminal. To make variables permanent, add them to shell startup files.

| File | When it runs | Use for |
|------|-------------|---------|
| `~/.bashrc` | Every new terminal | Aliases, variables, customization |
| `~/.bash_profile` | Login sessions (SSH) | Variables for login sessions |
| `~/.profile` | Login for any shell | Universal variables |

**Simple rule:** Put everything in `~/.bashrc` — it covers most cases.

```bash
nano ~/.bashrc
```

Add at the bottom:
```bash
# Environment variables
export DB_HOST="localhost"
export DB_PORT="5432"
export JAVA_HOME="/usr/lib/jvm/java-11"
export EDITOR="vim"

# Add scripts folder to PATH
export PATH=$PATH:/home/varun/scripts

# Aliases — shortcuts for long commands
alias ll='ls -la'
alias la='ls -ltr'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'

# System monitoring shortcuts
alias ports='ss -tulnp'
alias meminfo='free -h'
alias diskinfo='df -h'
alias myip='ip addr show | grep inet'

# Service management
alias sstart='sudo systemctl start'
alias sstop='sudo systemctl stop'
alias srestart='sudo systemctl restart'
alias sstatus='sudo systemctl status'

# Log watching
alias syslog='sudo tail -f /var/log/syslog'
alias nginxlog='sudo tail -f /var/log/nginx/error.log'

# Safety nets — always ask before destructive operations
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
```

**Apply changes without restarting terminal:**
```bash
source ~/.bashrc
# or
. ~/.bashrc      # shorthand — . is the same as source
```

> ⚠️ **Common mistake:** Edit `.bashrc`, forget to source it, wonder why aliases don't work yet. Always `source ~/.bashrc` after editing.

---

### Aliases — Creating and Managing

```bash
alias ll='ls -la'              # create alias temporarily (current session)
alias gs='git status'
alias dc='docker-compose'
alias                          # view all current aliases
unalias ll                     # remove an alias
```

Aliases in `~/.bashrc` are permanent — available in every new terminal.

---

### .env Files — Used in Every Real Project

```bash
# /home/varun/myproject/.env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=admin
DB_PASSWORD=supersecret123
API_KEY=abc123xyz789
ENVIRONMENT=development
DEBUG=true
```

> ⚠️ **NEVER commit .env to Git.** Passwords and API keys in a public repo are exposed to the entire internet. This happens to developers constantly.

```bash
echo ".env" >> .gitignore    # add to gitignore BEFORE creating the file
```

**Loading .env in scripts:**
```bash
source .env                  # load all variables from .env
export $(cat .env | xargs)   # alternative — export all at once
```

---

### Checking if Variable is Set — Professional Scripts

```bash
#!/bin/bash
if [ -z "$DB_PASSWORD" ]; then
    echo "ERROR: DB_PASSWORD environment variable is not set"
    echo "Run: export DB_PASSWORD=yourpassword"
    exit 1
fi

echo "Connecting to database..."
```

**Default value if variable is not set:**
```bash
ENVIRONMENT=${ENVIRONMENT:-"development"}   # default to development if not set
```

---

### Environment Variables in Docker

```bash
docker run -e DB_HOST=localhost myapp
docker run -e DB_HOST=localhost -e DB_PORT=5432 myapp
docker run --env-file .env myapp     # pass entire .env file
```

---

### Environment Variables in CI/CD

**GitHub Actions:**
```yaml
env:
  DB_HOST: ${{ secrets.DB_HOST }}
  API_KEY: ${{ secrets.API_KEY }}
```

**Jenkins:**
```groovy
environment {
    DB_PASSWORD = credentials('db-password-secret')
    ENVIRONMENT = 'production'
}
```

Secrets are stored in the CI/CD tool settings — not in code. The pipeline injects them as environment variables at runtime.

---

### PS1 — Customize Your Terminal Prompt

```bash
echo $PS1            # see current prompt setting

# Add to ~/.bashrc:
export PS1='\u@\h:\w\$ '    # username@hostname:directory$

# With colors:
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '

# With time:
export PS1='[\t] \u@\h:\w\$ '
```

| Code | Meaning |
|------|---------|
| `\u` | Username |
| `\h` | Hostname |
| `\w` | Current directory |
| `\t` | Current time |
| `\$` | $ for regular user, # for root |

---

### Full Summary — Day 15

| Command | What it does |
|---------|-------------|
| `env` or `printenv` | Show all environment variables |
| `echo $VARIABLE` | Show value of specific variable |
| `export VAR=value` | Set exported variable — visible to child processes |
| `VAR=value ./script.sh` | Set variable for one command only |
| `unset VAR` | Remove a variable |
| `source ~/.bashrc` | Reload .bashrc in current shell |
| `. ~/.bashrc` | Same as source — shorthand |
| `alias name='command'` | Create command shortcut |
| `unalias name` | Remove alias |
| `alias` | List all current aliases |

| File | Purpose |
|------|---------|
| `~/.bashrc` | Runs every new terminal — put aliases and exports here |
| `~/.bash_profile` | Runs on login sessions (SSH) |
| `~/.profile` | Login for any shell |
| `.env` | Project-specific variables — NEVER commit to Git |

---

### Interview Questions — Day 15

**Q1. What is an environment variable?**
A named value stored in the shell that programs and scripts can read. Used to pass configuration and secrets to applications without hardcoding them in code.

**Q2. What is the difference between a local variable and an exported variable?**
A local variable exists only in the current shell — child processes and scripts cannot see it. An exported variable is available to any program or script launched from that shell.

**Q3. What is the PATH variable?**
A colon-separated list of directories where Linux searches for executable commands. When you type a command, Linux looks through each PATH directory until it finds the executable.

**Q4. How do you add a directory to PATH without breaking existing commands?**
`export PATH=$PATH:/new/directory` — always append using `$PATH:` before the new directory. Never replace PATH.

**Q5. What is a .env file and why should it never be committed to Git?**
A `.env` file stores environment variables for a project — typically database passwords, API keys, and config values. Committing it to Git exposes secrets to anyone who can access the repository.

**Q6. What is the difference between `~/.bashrc` and `~/.bash_profile`?**
`.bashrc` runs every time a new interactive terminal is opened. `.bash_profile` runs only on login sessions (like SSH). For most purposes on Ubuntu, putting variables in `.bashrc` is sufficient.

**Q7. How do you make environment variables permanent?**
Add `export VARIABLE=value` to `~/.bashrc`. Then run `source ~/.bashrc` to apply immediately.

**Q8. What does `source` do?**
`source` (or `.`) executes a file in the current shell session — not in a subprocess. This means variables set in the sourced file persist in your current shell. Regular `./script.sh` runs in a subprocess so its variables don't persist.

**Q9. How does `${VARIABLE:-default}` work?**
Uses the variable's value if set, otherwise uses the default value. Example: `ENV=${ENVIRONMENT:-"development"}` — if ENVIRONMENT is not set, ENV becomes "development".

---

### Homework — Before Day 16

1. Run `env | sort` — look through all variables on your system
2. Run `echo $PATH` — count how many directories are in it
3. Add 3 aliases to `~/.bashrc` that you will actually use daily
4. Add `export EDITOR=vim` to `~/.bashrc` and source it
5. Create a `.env` file in `~/linux_practice` with 3 fake variables
6. Write a script that reads from `.env` and prints the values

---

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