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