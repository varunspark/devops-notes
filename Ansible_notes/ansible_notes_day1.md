# Ansible Notes

---

## Ansible Day 1 — Introduction & Core Concepts

---

### Why Ansible is Non-Negotiable for DevOps

Here is the most common problem in infrastructure management:

**"I manually configured 50 servers and now I don't remember what I did on which one."**

You set up one server manually — works perfectly. You need to do the same on 20 more servers. You SSH into each one, run commands, make typos, forget steps. Every server ends up slightly different. Chaos.

Ansible solves this completely. You write exactly what you want done in a simple file — and Ansible does it identically on 1 server or 1000 servers in one command.

**Ansible is in every DevOps job description. No exceptions.**

---

### What You Will Learn Today

- What Ansible is and why it exists
- Ansible vs other tools
- Core concepts — inventory, playbook, task, module, handler
- Installing Ansible on WSL
- Your first playbook
- Variables and handlers
- Ansible roles
- Best practices

---

### What is Ansible?

Ansible is an **automation tool** that lets you configure servers, install software, deploy applications, and manage infrastructure — all from your local machine, without touching each server manually.

```
Normal way (painful):               Ansible way (smart):

SSH into server 1 → run commands    Write one playbook
SSH into server 2 → run commands    Run: ansible-playbook site.yml
SSH into server 3 → run commands    Done. All 3 servers configured.
...repeat 47 more times
```

**Three things that make Ansible special:**

| Feature | What it means |
|---|---|
| Agentless | No software needed on target servers — uses SSH only |
| YAML based | Instructions written in simple, readable YAML files |
| Idempotent | Run the same playbook 10 times — same result, no duplicates |

---

### Ansible vs Other Tools

This is one of the most asked interview questions:

```
Tool          What it does                    When to use
----------    ----------------------------    ----------------------
Ansible       Configure servers               After servers exist
Terraform     Create servers (infra)          Before servers exist
Docker        Package apps in containers      App deployment
Kubernetes    Manage containers at scale      Large container fleets
```

**Key point:** Ansible and Terraform complement each other.
Terraform creates the servers → Ansible configures them.

---

### Core Concepts

Five concepts. Understand these and you understand Ansible.

| Concept | What it is | Real world analogy |
|---|---|---|
| Inventory | List of your servers | Contacts list on your phone |
| Playbook | The instruction file | Recipe — full list of steps |
| Task | One single action | One step in a recipe |
| Module | Built-in tool that does the task | App on your phone |
| Handler | Task that runs only when something changes | Fire alarm — only rings when there's fire |

**How they connect:**

```
Inventory  →  WHERE to go    (which servers)
Playbook   →  WHAT to do     (full plan)
Task       →  one step       (single action)
Module     →  the tool       (apt, copy, service...)
Handler    →  only if changed (restart nginx if config changed)
```

---

### Installing Ansible

**On WSL (Windows) or Ubuntu:**

```bash
sudo apt update
sudo apt install ansible -y

# Verify installation:
ansible --version
```

**Test it works:**

```bash
ansible localhost -m ping
```

Expected output:
```
localhost | SUCCESS => {
    "ping": "pong"
}
```

If you see **pong** — you're set up correctly. That's your first Ansible win.

---

### Your First Inventory File

An inventory file tells Ansible which servers to manage.

```ini
# inventory.ini

[webservers]
192.168.1.10
192.168.1.11

[dbservers]
192.168.1.20

[local]
localhost ansible_connection=local
```

**Group servers by role** — webservers, dbservers, appservers etc. Then your playbook can target specific groups.

---

### Your First Playbook

A playbook is a YAML file that tells Ansible what to do on which servers.

```yaml
# playbook.yml
---
- name: Setup web server
  hosts: webservers        # run on [webservers] group
  become: yes              # use sudo

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present     # present = install it

    - name: Start nginx
      service:
        name: nginx
        state: started     # started = make sure it's running
```

**Run it:**

```bash
ansible-playbook -i inventory.ini playbook.yml
```

**State values explained:**

| State | Meaning |
|---|---|
| present | Make sure it's installed |
| absent | Make sure it's removed |
| started | Make sure it's running |
| stopped | Make sure it's not running |
| latest | Install and keep updated |

---

### Variables

Instead of hardcoding values, use variables. Cleaner and reusable.

```yaml
# playbook.yml
---
- name: Setup web server
  hosts: webservers
  become: yes

  vars:
    package_name: nginx
    service_port: 80

  tasks:
    - name: Install {{ package_name }}
      apt:
        name: "{{ package_name }}"
        state: present
```

**Where to define variables:**

```
1. Inside playbook (vars:)           → for simple, playbook-specific values
2. Separate file (vars/main.yml)     → for organised, reusable values
3. Inventory file                    → for per-server values
4. Command line (--extra-vars)       → for one-time overrides
```

---

### Handlers

A handler is a task that only runs when notified — when something actually changed.

**Without handler (bad):** nginx restarts every single time you run the playbook, even when nothing changed.

**With handler (good):** nginx only restarts when the config file was actually modified.

```yaml
---
- name: Setup web server
  hosts: webservers
  become: yes

  tasks:
    - name: Copy nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx        # triggers the handler below

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted           # only runs if config was changed
```

**Key point:** If the config file was already the same — handler never runs. That's idempotency in action.

---

### Common Modules

These are the modules you'll use 90% of the time:

| Module | What it does | Example use |
|---|---|---|
| apt | Install/remove packages (Ubuntu) | Install nginx, python |
| yum | Install/remove packages (CentOS) | Same but for RedHat |
| copy | Copy files to server | Copy config files |
| template | Copy file with variables | Dynamic config files |
| service | Start/stop/restart services | Manage nginx, mysql |
| file | Create/delete files & folders | Create directories |
| user | Manage users | Create deploy user |
| git | Clone git repositories | Deploy app code |
| command | Run any shell command | One-off commands |
| shell | Run shell commands with pipes | Complex commands |

---

### Roles — Organising Your Playbooks

When your playbook grows big, break it into **roles**. A role is a structured folder for one responsibility — like "nginx" or "database".

**Generate a role:**

```bash
ansible-galaxy init nginx
```

**This creates:**

```
nginx/
├── tasks/
│   └── main.yml        ← your tasks go here
├── handlers/
│   └── main.yml        ← handlers go here
├── vars/
│   └── main.yml        ← variables go here
├── templates/
│   └── nginx.conf.j2   ← config file templates
├── files/              ← static files to copy
└── defaults/
    └── main.yml        ← default variable values
```

**Using a role in your playbook:**

```yaml
---
- name: Configure servers
  hosts: webservers
  become: yes

  roles:
    - nginx
    - common
```

Clean. No giant playbook file. Each role handles one thing.

---

### SSH Keys — Where to Store Them

Ansible connects to servers via SSH. You need keys set up.

```
Your machine                   Remote server
────────────                   ─────────────
~/.ssh/id_rsa                  ~/.ssh/authorized_keys
(private key — never share)    (your public key goes here)
```

**Tell Ansible where your key is — in ansible.cfg:**

```ini
# ansible.cfg
[defaults]
private_key_file = ~/.ssh/id_rsa
remote_user = ubuntu
inventory = ./inventory.ini
```

**Best practice — project folder structure:**

```
my-project/
├── ansible.cfg          ← SSH key location, settings
├── inventory.ini        ← server list
├── playbook.yml         ← tasks
├── roles/               ← organised roles
│   ├── nginx/
│   └── database/
└── .gitignore           ← never commit keys to GitHub!
```

**Golden rules:**
- Private key stays in `~/.ssh/` — never move it
- Never paste keys inside playbooks
- Always add `.gitignore` before pushing to GitHub

---

### Ansible Vault — Protecting Secrets

Never store passwords, API keys, or tokens as plain text in playbooks. Use Ansible Vault to encrypt them.

```bash
# Encrypt a value:
ansible-vault encrypt_string 'mypassword123' --name 'db_password'

# Create an encrypted file:
ansible-vault create secrets.yml

# Edit an encrypted file:
ansible-vault edit secrets.yml

# Run playbook with vault password:
ansible-playbook playbook.yml --ask-vault-pass
```

**In your playbook:**

```yaml
vars:
  db_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    61383538...encrypted...
```

---

### Project Folder — Full Real-World Setup

```
project/
├── ansible.cfg
├── inventory.ini
├── site.yml             ← master playbook
├── .gitignore
├── vars/
│   └── main.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   ├── nginx/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/nginx.conf.j2
│   └── database/
│       └── tasks/main.yml
└── group_vars/
    ├── webservers.yml   ← vars only for webservers
    └── dbservers.yml    ← vars only for dbservers
```

---

### Core Commands Cheat Sheet

```bash
# Test connection to all servers
ansible all -m ping -i inventory.ini

# Run a playbook
ansible-playbook -i inventory.ini playbook.yml

# Dry run — see what would change without doing it
ansible-playbook -i inventory.ini playbook.yml --check

# Run only specific tasks (by tag)
ansible-playbook -i inventory.ini playbook.yml --tags "nginx"

# Show verbose output for debugging
ansible-playbook -i inventory.ini playbook.yml -v

# Create a new role
ansible-galaxy init rolename

# Encrypt a secret
ansible-vault encrypt_string 'secret' --name 'var_name'
```

---

### Interview Questions — Ansible

**Q: What is idempotency in Ansible?**
Running a playbook multiple times produces the same result. If nginx is already installed, Ansible skips that task instead of installing again.

**Q: What is the difference between a task and a handler?**
A task always runs. A handler only runs when notified by a task that made a change.

**Q: What is an Ansible role?**
A way to organise playbooks into reusable, structured folders — with separate files for tasks, handlers, variables, and templates.

**Q: Ansible vs Terraform — what's the difference?**
Terraform provisions infrastructure (creates servers, networks). Ansible configures that infrastructure (installs software, manages files). They work together.

**Q: What is ansible-vault?**
A tool to encrypt sensitive data like passwords and API keys inside playbooks so they can be safely stored in version control.

**Q: What is the difference between `command` and `shell` module?**
`command` runs a command directly — no shell features like pipes or redirects. `shell` runs through a shell so you can use `|`, `&&`, `>` etc.

**Q: What is a dynamic inventory?**
Instead of a static `inventory.ini` file, a dynamic inventory script fetches the server list automatically from AWS, GCP, Azure etc. Useful when servers are created and destroyed frequently.

---

### End of Day 1 Checklist

- [ ] Ansible installed — `ansible --version` works
- [ ] `ansible localhost -m ping` returns pong
- [ ] inventory.ini created with server groups
- [ ] First playbook written and run
- [ ] Variables used in a playbook
- [ ] Handler added to playbook
- [ ] Role created with `ansible-galaxy init`
- [ ] ansible.cfg configured with SSH key path
- [ ] 6 interview questions answered in your own words

---

*Next — Ansible Day 2: Dynamic inventory, Jinja2 templates, ansible-vault deep dive, CI/CD integration with GitHub Actions*
