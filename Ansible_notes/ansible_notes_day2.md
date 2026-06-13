# Ansible Notes

---

## Ansible Day 2 — Dynamic Inventory, Jinja2, Vault & CI/CD

---

### What Day 1 Covered (Quick Recap)

Before Day 2 — make sure these are solid:

```
Ansible           →  automation tool, agentless, YAML based, idempotent
Inventory         →  list of servers (inventory.ini)
Playbook          →  instruction file (what to do, where to do it)
Task              →  one single action
Module            →  the tool that does the task (apt, copy, service...)
Handler           →  task that runs only when something changed
Roles             →  organised folder structure for big playbooks
ansible-vault     →  encrypts secrets so they're safe in Git
```

If any of these feel fuzzy — re-read Day 1 before continuing.

---

### What You Will Learn Today

- Dynamic inventory — auto-fetch servers from AWS, GCP, Azure
- Jinja2 templates — config files with variables baked in
- Ansible Vault deep dive — encrypting files, using vault in CI/CD
- Tags — run only parts of a playbook
- Conditionals — run tasks only when a condition is true
- Loops — do the same task for a list of things
- Error handling — when tasks fail, what to do
- CI/CD integration — run Ansible from GitHub Actions
- Real-world project — deploy a web app end to end

---

### Dynamic Inventory — Auto-Fetch Your Servers

**The problem with static inventory:**

```
# inventory.ini — static, you write it manually
[webservers]
192.168.1.10
192.168.1.11
192.168.1.12
```

What happens in real DevOps? You use AWS, Azure, or GCP. Servers get created and destroyed automatically. Their IP addresses change. Updating `inventory.ini` manually every time is not realistic.

**Dynamic inventory solves this.** Instead of a file, you give Ansible a script or plugin that calls your cloud provider's API and returns the current server list automatically.

```
Static inventory:    You write the server list → gets outdated
Dynamic inventory:   Ansible asks AWS "give me all servers" → always fresh
```

**AWS dynamic inventory — the modern way (inventory plugin):**

```bash
# Install required collection
ansible-galaxy collection install amazon.aws
pip install boto3 botocore --break-system-packages
```

**Create `aws_ec2.yml` — your dynamic inventory file:**

```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2

regions:
  - ap-south-1        # Mumbai — change to your region

filters:
  instance-state-name: running    # only running instances

keyed_groups:
  - key: tags.Role           # group by the "Role" tag on your EC2 instances
    prefix: role
  - key: tags.Environment    # group by "Environment" tag
    prefix: env

hostnames:
  - dns-name               # use public DNS as hostname
  - ip-address             # fallback to IP

compose:
  ansible_host: public_ip_address   # connect using public IP
```

**Use it:**

```bash
# Test — list all servers Ansible can see
ansible-inventory -i aws_ec2.yml --list

# Ping all servers
ansible all -i aws_ec2.yml -m ping

# Run playbook using dynamic inventory
ansible-playbook -i aws_ec2.yml playbook.yml
```

**Your EC2 instances must have tags** — that's how dynamic inventory groups them:

```
EC2 Tag: Role = webserver    → group: role_webserver
EC2 Tag: Role = database     → group: role_database
EC2 Tag: Environment = prod  → group: env_prod
```

Then in your playbook:

```yaml
- name: Setup web servers
  hosts: role_webserver     # targets all EC2 with Role=webserver tag
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

**GCP and Azure dynamic inventory:**

```yaml
# gcp_compute.yml — for Google Cloud
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project
auth_kind: application
keyed_groups:
  - key: labels.role
    prefix: role

# azure_rm.yml — for Azure
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - my-resource-group
keyed_groups:
  - key: tags.role
    prefix: role
```

---

### Jinja2 Templates — Dynamic Config Files

**What is Jinja2?**

Jinja2 is a templating language. It lets you write config files with variables, conditions, and loops inside them. Ansible uses Jinja2 to generate config files dynamically per server.

```
Normal config file:           Jinja2 template:
  worker_processes 4;           worker_processes {{ ansible_processor_vcpus }};
  listen 80;                    listen {{ http_port }};
  server_name myapp.com;        server_name {{ server_name }};
```

The template has variables. Ansible fills them in differently for each server.

**Create a template file — `nginx.conf.j2`:**

```nginx
# nginx.conf.j2
# The .j2 extension means "this is a Jinja2 template"

user www-data;
worker_processes {{ ansible_processor_vcpus }};   {# uses server's actual CPU count #}

events {
    worker_connections 1024;
}

http {

    server {
        listen {{ http_port | default(80) }};      {# default to 80 if not set #}
        server_name {{ server_name }};

        location / {
            proxy_pass http://{{ app_host }}:{{ app_port }};
            proxy_set_header Host $host;
        }

        {# only add SSL block if ssl_enabled is true #}
        {% if ssl_enabled %}
        listen 443 ssl;
        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};
        {% endif %}

    }

    {# loop over all upstream app servers #}
    upstream app_servers {
        {% for server in app_servers_list %}
        server {{ server }}:{{ app_port }};
        {% endfor %}
    }

}
```

**Use the template in your playbook:**

```yaml
- name: Deploy nginx config
  hosts: webservers
  become: yes

  vars:
    http_port: 80
    server_name: myapp.com
    app_host: localhost
    app_port: 8080
    ssl_enabled: false
    app_servers_list:
      - 10.0.0.1
      - 10.0.0.2

  tasks:
    - name: Generate nginx config from template
      template:
        src: nginx.conf.j2        # template file on your machine
        dest: /etc/nginx/nginx.conf   # where it goes on the server
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

**What Ansible does:**
1. Takes `nginx.conf.j2`
2. Replaces all `{{ variable }}` with real values
3. Copies the filled-in file to the server
4. If the file changed → handler restarts nginx

**Jinja2 — useful filters:**

```yaml
# default — use this value if variable is undefined
{{ port | default(8080) }}

# upper / lower — change case
{{ env | upper }}              # "prod" → "PROD"

# join — join a list into a string
{{ servers | join(', ') }}     # ["a","b","c"] → "a, b, c"

# length — count items
{{ servers | length }}         # 3

# replace — substitute text
{{ path | replace('/tmp', '/var') }}

# int / string — convert types
{{ "42" | int + 1 }}           # 43
```

---

### Ansible Vault — Deep Dive

Day 1 introduced Vault basics. Now let's use it properly in real projects.

**Why vault matters:**

```
Without vault:                  With vault:
  db_password: mypassword123      db_password: !vault |
  → visible in Git                  $ANSIBLE_VAULT;1.1;AES256
  → anyone can read it              61626361343430...
  → security disaster             → safe to commit to Git
```

**Encrypting a whole file:**

```bash
# Create a new encrypted file
ansible-vault create group_vars/all/secrets.yml

# Encrypt an existing file
ansible-vault encrypt group_vars/all/secrets.yml

# View the contents (decrypts temporarily for reading)
ansible-vault view group_vars/all/secrets.yml

# Edit the contents
ansible-vault edit group_vars/all/secrets.yml

# Decrypt a file permanently (use with caution)
ansible-vault decrypt group_vars/all/secrets.yml

# Re-encrypt with a new password
ansible-vault rekey group_vars/all/secrets.yml
```

**Encrypting a single value (not a whole file):**

```bash
ansible-vault encrypt_string 'MyS3cr3tP@ssword' --name 'db_password'
```

Output you paste directly into your playbook:

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  30303664613561653061306466303335336230653365386431
  3162366165333738623735333835653532643032643836350a
  ...
```

**Running a playbook with vault — three ways:**

```bash
# Way 1: Type password manually (local use)
ansible-playbook playbook.yml --ask-vault-pass

# Way 2: Password stored in a file (safer for scripts)
echo "myvaultpassword" > ~/.vault_pass
chmod 600 ~/.vault_pass   # only you can read it
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Way 3: Environment variable (best for CI/CD)
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook playbook.yml
```

**Project structure — where vault files go:**

```
project/
├── group_vars/
│   ├── all/
│   │   ├── vars.yml        ← plain variables (safe to commit)
│   │   └── secrets.yml     ← vault encrypted (safe to commit, can't be read)
│   ├── webservers/
│   │   └── vars.yml
│   └── dbservers/
│       └── secrets.yml     ← DB passwords, only for DB servers
```

**`.gitignore` — what to never commit:**

```
.vault_pass           # vault password file — NEVER in Git
~/.ssh/id_rsa         # private SSH keys
*.pem                 # AWS key pairs
```

---

### Tags — Run Only Parts of a Playbook

In a large playbook with 50 tasks, you don't always want to run everything. Tags let you run only specific tasks.

```yaml
---
- name: Full server setup
  hosts: webservers
  become: yes

  tasks:

    - name: Install nginx
      apt:
        name: nginx
        state: present
      tags:
        - nginx
        - install

    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx
        - config

    - name: Install MySQL
      apt:
        name: mysql-server
        state: present
      tags:
        - mysql
        - install

    - name: Create app directory
      file:
        path: /var/www/myapp
        state: directory
      tags:
        - app
        - setup
```

**Run only tagged tasks:**

```bash
# Run only nginx tasks
ansible-playbook playbook.yml --tags "nginx"

# Run only install tasks (nginx + mysql install)
ansible-playbook playbook.yml --tags "install"

# Run everything EXCEPT mysql
ansible-playbook playbook.yml --skip-tags "mysql"

# Run multiple tags
ansible-playbook playbook.yml --tags "nginx,app"

# List all available tags without running anything
ansible-playbook playbook.yml --list-tags
```

**Special built-in tags:**

```bash
ansible-playbook playbook.yml --tags "always"    # always runs, even if tags filter
ansible-playbook playbook.yml --tags "never"     # never runs unless explicitly called
```

---

### Conditionals — Run Tasks Only When Needed

A conditional makes a task run only when a condition is true. Use `when:` for this.

```yaml
tasks:

  # Install on Ubuntu only
  - name: Install nginx (Ubuntu)
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"    # only on Ubuntu/Debian

  # Install on CentOS only
  - name: Install nginx (CentOS)
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"   # only on CentOS/RHEL

  # Run only in production
  - name: Enable strict firewall rules
    ufw:
      rule: deny
      port: all
    when: env == "prod"

  # Run only when variable is set
  - name: Configure SSL
    template:
      src: ssl.conf.j2
      dest: /etc/nginx/ssl.conf
    when: ssl_enabled is defined and ssl_enabled == true

  # Run only if a file doesn't exist (don't overwrite)
  - name: Generate app secret key
    command: openssl rand -hex 32
    when: not app_secret_file.stat.exists
```

**Check facts about the server — `ansible_facts`:**

```yaml
# ansible_facts are variables Ansible collects about every server automatically
# They tell you: OS, CPU, memory, IP, hostname, etc.

- name: Print server info
  debug:
    msg: "Server: {{ ansible_hostname }}, OS: {{ ansible_distribution }}, RAM: {{ ansible_memtotal_mb }}MB"

# Common facts:
# ansible_hostname         → server hostname
# ansible_os_family        → "Debian" or "RedHat"
# ansible_distribution     → "Ubuntu", "CentOS" etc.
# ansible_processor_vcpus  → number of CPUs
# ansible_memtotal_mb      → total RAM in MB
# ansible_default_ipv4.address  → primary IP address
```

**Collect facts manually:**

```bash
ansible webservers -m setup -i inventory.ini             # all facts
ansible webservers -m setup -a "filter=ansible_memory*"  # just memory facts
```

---

### Loops — Do the Same Task for a List

Instead of writing the same task 5 times for 5 packages, use a loop.

```yaml
tasks:

  # Install multiple packages with a loop
  - name: Install required packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - python3
      - git
      - curl
      - unzip

  # Create multiple users
  - name: Create deployment users
    user:
      name: "{{ item.name }}"
      groups: "{{ item.group }}"
      shell: /bin/bash
    loop:
      - { name: "deploy", group: "www-data" }
      - { name: "monitor", group: "adm" }
      - { name: "backup", group: "sudo" }

  # Create multiple directories
  - name: Create app directories
    file:
      path: "{{ item }}"
      state: directory
      owner: deploy
      mode: '0755'
    loop:
      - /var/www/myapp
      - /var/www/myapp/logs
      - /var/www/myapp/uploads
      - /var/log/myapp

  # Loop with index — when you need the position
  - name: Create numbered config files
    template:
      src: worker.conf.j2
      dest: "/etc/myapp/worker-{{ item.0 }}.conf"
    loop: "{{ range(1, 5) | list | zip(worker_configs) | list }}"
```

**Loop over a dictionary:**

```yaml
vars:
  nginx_headers:
    X-Frame-Options: DENY
    X-Content-Type-Options: nosniff
    X-XSS-Protection: "1; mode=block"

tasks:
  - name: Set security headers
    lineinfile:
      path: /etc/nginx/nginx.conf
      line: "add_header {{ item.key }} {{ item.value }};"
    loop: "{{ nginx_headers | dict2items }}"
```

---

### Error Handling — When Tasks Fail

By default, Ansible stops the entire playbook if any task fails. In real DevOps you often need more control.

**Ignore errors on a task:**

```yaml
- name: Stop app (might not be running yet — that's okay)
  service:
    name: myapp
    state: stopped
  ignore_errors: yes      # if this fails, keep going
```

**Define what "failure" means:**

```yaml
# By default, a non-zero exit code = failure
# But some commands return non-zero even when they succeed

- name: Check if user exists
  command: id myuser
  register: user_check
  failed_when: user_check.rc != 0 and user_check.rc != 1
  # rc=0 = user exists, rc=1 = user not found (not a failure), anything else = real error
```

**Register — save the output of a task:**

```yaml
- name: Get current app version
  command: cat /var/www/myapp/VERSION
  register: current_version     # save output to variable

- name: Show current version
  debug:
    msg: "Current version: {{ current_version.stdout }}"

- name: Deploy only if version changed
  git:
    repo: https://github.com/myorg/myapp.git
    dest: /var/www/myapp
  when: current_version.stdout != new_version
```

**Block, rescue, always — like try/catch/finally:**

```yaml
tasks:

  - name: Deploy application
    block:                          # try — run these tasks

      - name: Pull latest code
        git:
          repo: https://github.com/myorg/myapp.git
          dest: /var/www/myapp

      - name: Install dependencies
        command: npm install
        args:
          chdir: /var/www/myapp

      - name: Restart app
        service:
          name: myapp
          state: restarted

    rescue:                         # catch — runs if block fails

      - name: Rollback to previous version
        command: git checkout HEAD~1
        args:
          chdir: /var/www/myapp

      - name: Send alert
        mail:
          to: ops@mycompany.com
          subject: "Deploy failed on {{ ansible_hostname }}"
          body: "Rolled back to previous version."

    always:                         # finally — always runs

      - name: Write deploy log
        lineinfile:
          path: /var/log/deploy.log
          line: "{{ ansible_date_time.iso8601 }} — deploy attempted on {{ ansible_hostname }}"
          create: yes
```

---

### CI/CD Integration — GitHub Actions

In real DevOps, Ansible runs automatically when code is pushed — not manually from your laptop.

**How it works:**

```
Developer pushes infra change to Git
    ↓
GitHub Actions triggers
    ↓
Runner: installs Ansible
    ↓
Runner: decrypts vault using secret from GitHub Secrets
    ↓
Runner: runs ansible-playbook against your servers
    ↓
Servers are configured automatically
```

**Full GitHub Actions workflow:**

```yaml
# .github/workflows/ansible-deploy.yml
name: Ansible Deploy

on:
  push:
    branches: [ main ]
    paths:
      - 'ansible/**'        # only trigger if ansible files changed
  workflow_dispatch:        # allow manual trigger from GitHub UI

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Ansible
        run: |
          sudo apt update
          sudo apt install -y ansible
          ansible --version

      - name: Install required collections
        run: ansible-galaxy collection install amazon.aws community.general

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      - name: Write vault password
        run: |
          echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > ~/.vault_pass
          chmod 600 ~/.vault_pass

      - name: Run Ansible playbook
        run: |
          ansible-playbook \
            -i inventory.ini \
            --vault-password-file ~/.vault_pass \
            --private-key ~/.ssh/id_rsa \
            site.yml
        env:
          ANSIBLE_HOST_KEY_CHECKING: False

      - name: Clean up secrets
        if: always()          # runs even if previous steps failed
        run: |
          rm -f ~/.ssh/id_rsa
          rm -f ~/.vault_pass
```

**GitHub Secrets you need to set:**

```
SSH_PRIVATE_KEY          → your server's private SSH key (copy from ~/.ssh/id_rsa)
ANSIBLE_VAULT_PASSWORD   → your vault password
SERVER_IP                → IP of the target server (for ssh-keyscan)
AWS_ACCESS_KEY_ID        → if using dynamic inventory on AWS
AWS_SECRET_ACCESS_KEY    → if using dynamic inventory on AWS
```

**Go to:** GitHub repo → Settings → Secrets and variables → Actions → New repository secret

---

### Real-World Project — Deploy a Web App

Let's put everything together. Deploy a Node.js app to an Ubuntu server — automatically.

**Project structure:**

```
ansible-deploy/
├── ansible.cfg
├── inventory.ini
├── site.yml                    ← master playbook
├── .github/
│   └── workflows/
│       └── deploy.yml
├── group_vars/
│   ├── all/
│   │   ├── vars.yml
│   │   └── secrets.yml         ← vault encrypted
│   └── webservers/
│       └── vars.yml
└── roles/
    ├── common/
    │   └── tasks/main.yml      ← base packages, users, firewall
    ├── nodejs/
    │   ├── tasks/main.yml      ← install Node.js
    │   └── vars/main.yml
    └── myapp/
        ├── tasks/main.yml      ← deploy app code
        ├── handlers/main.yml   ← restart app
        └── templates/
            ├── app.service.j2  ← systemd service file
            └── nginx.conf.j2   ← nginx config
```

**`group_vars/all/vars.yml`:**

```yaml
app_name: myapp
app_user: deploy
app_dir: /var/www/myapp
app_port: 3000
node_version: "20"
git_repo: https://github.com/myorg/myapp.git
git_branch: main
http_port: 80
server_name: myapp.com
```

**`group_vars/all/secrets.yml` (vault encrypted):**

```yaml
db_password: MySuperSecretPassword
app_secret_key: anotherSecretKey
```

**`roles/common/tasks/main.yml`:**

```yaml
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600   # only update if cache is older than 1 hour

- name: Install base packages
  apt:
    name:
      - git
      - curl
      - ufw
      - fail2ban
    state: present

- name: Create deploy user
  user:
    name: "{{ app_user }}"
    shell: /bin/bash
    create_home: yes

- name: Allow SSH through firewall
  ufw:
    rule: allow
    port: "22"

- name: Allow HTTP through firewall
  ufw:
    rule: allow
    port: "{{ http_port }}"

- name: Enable firewall
  ufw:
    state: enabled
    policy: deny
```

**`roles/myapp/templates/app.service.j2`:**

```ini
[Unit]
Description={{ app_name }} Node.js application
After=network.target

[Service]
Type=simple
User={{ app_user }}
WorkingDirectory={{ app_dir }}
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT={{ app_port }}
Environment=DB_PASSWORD={{ db_password }}

[Install]
WantedBy=multi-user.target
```

**`roles/myapp/tasks/main.yml`:**

```yaml
---
- name: Clone/update app code
  git:
    repo: "{{ git_repo }}"
    dest: "{{ app_dir }}"
    version: "{{ git_branch }}"
    force: yes
  become_user: "{{ app_user }}"
  notify: Restart app

- name: Install npm dependencies
  command: npm ci --production
  args:
    chdir: "{{ app_dir }}"
  become_user: "{{ app_user }}"
  notify: Restart app

- name: Copy systemd service
  template:
    src: app.service.j2
    dest: "/etc/systemd/system/{{ app_name }}.service"
  notify: Restart app

- name: Enable and start app
  systemd:
    name: "{{ app_name }}"
    enabled: yes
    state: started
    daemon_reload: yes

- name: Copy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/{{ app_name }}
  notify: Reload nginx

- name: Enable nginx site
  file:
    src: /etc/nginx/sites-available/{{ app_name }}
    dest: /etc/nginx/sites-enabled/{{ app_name }}
    state: link
  notify: Reload nginx
```

**`roles/myapp/handlers/main.yml`:**

```yaml
---
- name: Restart app
  systemd:
    name: "{{ app_name }}"
    state: restarted
    daemon_reload: yes

- name: Reload nginx
  service:
    name: nginx
    state: reloaded
```

**`site.yml` — master playbook:**

```yaml
---
- name: Configure all servers
  hosts: webservers
  become: yes

  roles:
    - common
    - nodejs
    - myapp
```

**Run it:**

```bash
# Dry run first — see what would change
ansible-playbook -i inventory.ini site.yml --check

# Real run
ansible-playbook -i inventory.ini site.yml --vault-password-file ~/.vault_pass

# Run only app deployment (not common setup)
ansible-playbook -i inventory.ini site.yml --tags "app" --vault-password-file ~/.vault_pass
```

---

### Core Commands Cheat Sheet — Day 2

```bash
# Dynamic inventory — list all discovered servers
ansible-inventory -i aws_ec2.yml --list

# Dynamic inventory — run playbook
ansible-playbook -i aws_ec2.yml playbook.yml

# Collect all facts about servers
ansible all -m setup -i inventory.ini

# Collect specific facts
ansible all -m setup -a "filter=ansible_distribution*" -i inventory.ini

# Run only tagged tasks
ansible-playbook playbook.yml --tags "nginx"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "mysql"

# List all tags in a playbook
ansible-playbook playbook.yml --list-tags

# Run with vault password from file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Encrypt a single value
ansible-vault encrypt_string 'mysecret' --name 'var_name'

# Run with extra variables (override anything)
ansible-playbook playbook.yml --extra-vars "env=prod version=1.2.0"

# Limit to specific host
ansible-playbook playbook.yml --limit "192.168.1.10"

# Step through tasks one at a time (interactive)
ansible-playbook playbook.yml --step

# Check syntax without running
ansible-playbook playbook.yml --syntax-check
```

---

### Interview Questions — Ansible Day 2

**Q: What is dynamic inventory in Ansible?**
Instead of a static `inventory.ini` file, dynamic inventory uses a plugin or script to fetch the server list automatically from a cloud provider like AWS, GCP, or Azure. When servers are created or destroyed automatically, the inventory stays up to date without manual changes.

**Q: What is a Jinja2 template in Ansible?**
A template is a config file with variables and logic inside it — using the Jinja2 templating language. The `template` module fills in the variables and copies the result to the server. It lets you generate different config files for each server from one template file.

**Q: What is `register` in Ansible?**
`register` saves the output of a task into a variable so you can use it in later tasks. For example, you can register the result of a version check and only run the deploy task if the version has changed.

**Q: What is the difference between `ignore_errors` and `failed_when`?**
`ignore_errors: yes` tells Ansible to continue running even if the task fails — regardless of why. `failed_when` lets you define exactly what counts as a failure — you can make a task pass even if the exit code is non-zero, or fail even if exit code is zero.

**Q: What is block/rescue/always in Ansible?**
It's Ansible's version of try/catch/finally. `block` contains the tasks to try. `rescue` runs if any task in the block fails — useful for rollback. `always` runs regardless of success or failure — useful for cleanup and logging.

**Q: How do you use Ansible in a CI/CD pipeline?**
Store the SSH private key and vault password in CI secrets (e.g. GitHub Secrets). In the workflow, write the SSH key to `~/.ssh/id_rsa` and the vault password to a file, then run `ansible-playbook` with the vault password file. Clean up the secrets at the end using an `always` step.

**Q: What is `ansible_facts` and why is it useful?**
Facts are variables Ansible automatically collects about each server before running tasks — OS type, CPU count, memory, IP address, hostname etc. You use them in conditionals like `when: ansible_os_family == "Debian"` to make playbooks work on different server types without changing the code.

**Q: What is the `when` keyword and give an example?**
`when` is a conditional that makes a task run only if the condition is true. For example: `when: ansible_os_family == "RedHat"` makes a task run only on CentOS/RHEL servers, and skip on Ubuntu. It's how you make one playbook work across different server types.

---

### End of Day 2 Checklist

- [ ] Dynamic inventory configured — `ansible-inventory --list` shows servers from AWS/GCP or localhost
- [ ] Jinja2 template created and deployed with the `template` module
- [ ] Vault file encrypted — `ansible-vault create secrets.yml` works
- [ ] Playbook run with `--vault-password-file`
- [ ] Tags added to a playbook — `--tags` used to run subset of tasks
- [ ] `when:` conditional used in at least one task
- [ ] `loop:` used to install multiple packages in one task
- [ ] `register:` used to capture output and use it in another task
- [ ] `block/rescue/always` used for error handling
- [ ] GitHub Actions workflow written that runs a playbook
- [ ] All 8 interview questions answered in your own words

---

*Next — Ansible Day 3: Ansible Galaxy (community roles), testing with Molecule, Ansible Tower/AWX (web UI for Ansible), Windows automation, network device automation, performance tuning for large fleets*
