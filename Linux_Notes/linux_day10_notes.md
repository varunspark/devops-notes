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
