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