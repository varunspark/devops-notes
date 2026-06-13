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
