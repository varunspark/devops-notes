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
