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
