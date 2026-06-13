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
