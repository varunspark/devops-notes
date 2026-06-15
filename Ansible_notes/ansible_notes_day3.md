# Ansible Notes

---

## Ansible Day 3 — Galaxy, Testing, AWX & Advanced Automation

---

### What Day 2 Covered (Quick Recap)

Before Day 3 — make sure these are solid:

```
Dynamic inventory   →  auto-fetch servers from AWS/GCP/Azure using plugins
Jinja2 templates    →  config files with variables, conditions, loops (.j2 files)
Ansible Vault       →  encrypt secrets, use --vault-password-file in CI/CD
Tags                →  run only parts of a playbook with --tags
Conditionals        →  when: ansible_os_family == "Debian"
Loops               →  loop: over packages, users, directories
Register            →  save task output to a variable for later use
Block/rescue/always →  try/catch/finally for error handling
GitHub Actions      →  run Ansible automatically on git push
```

If any of these feel fuzzy — re-read Day 2 before continuing.

---

### What You Will Learn Today

- Ansible Galaxy — use community roles instead of writing everything yourself
- Molecule — test your Ansible roles properly before running on real servers
- Ansible Tower / AWX — web UI and API for Ansible at scale
- Windows automation — manage Windows servers with Ansible
- Network device automation — configure routers and switches
- Performance tuning — make Ansible fast on large fleets
- Ansible best practices — how real teams use Ansible professionally

---

### Ansible Galaxy — Don't Write Everything Yourself

**What is Ansible Galaxy?**

Galaxy is the community hub for Ansible roles. Think of it like npm for Node.js or pip for Python — but for Ansible roles. Someone has already written a role to install MySQL, configure nginx, set up Docker, harden a server, and thousands of other things. You just download and use it.

```
Without Galaxy:                  With Galaxy:
  Write 200 lines to install        ansible-galaxy install geerlingguy.mysql
  and configure MySQL               Done. Tested, production-grade role.
  Debug for 3 hours
  Still have edge cases
```

**Search and install roles:**

```bash
# Search for a role
ansible-galaxy search nginx

# Install a role
ansible-galaxy install geerlingguy.nginx

# Install a specific version
ansible-galaxy install geerlingguy.nginx,3.2.0

# Install multiple roles at once from a file
ansible-galaxy install -r requirements.yml
```

**`requirements.yml` — the right way to manage Galaxy roles:**

```yaml
# requirements.yml
---
roles:

  # From Ansible Galaxy
  - name: geerlingguy.nginx
    version: "3.2.0"

  - name: geerlingguy.mysql
    version: "4.3.2"

  - name: geerlingguy.docker
    version: "6.1.0"

  # From a Git repo directly
  - name: my-company-hardening
    src: https://github.com/mycompany/ansible-hardening.git
    version: main

collections:

  # Collections are like bundles of multiple roles + modules
  - name: community.general
    version: "8.0.0"

  - name: amazon.aws
    version: "7.0.0"
```

**Install everything at once:**

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

**Where roles get installed:**

```
~/.ansible/roles/           ← default location
./roles/                    ← project-local (recommended)
```

**Tell Ansible to look in your project's roles folder — `ansible.cfg`:**

```ini
[defaults]
roles_path = ./roles:~/.ansible/roles
```

**Use a Galaxy role in your playbook — same as your own roles:**

```yaml
---
- name: Setup web server
  hosts: webservers
  become: yes

  roles:
    - geerlingguy.nginx      # Galaxy role
    - geerlingguy.mysql      # Galaxy role
    - myapp                  # your own role
```

**Passing variables to a Galaxy role:**

```yaml
- name: Setup database server
  hosts: dbservers
  become: yes

  vars:
    mysql_root_password: "{{ vault_mysql_root_password }}"
    mysql_databases:
      - name: myapp_db
        encoding: utf8mb4
    mysql_users:
      - name: myapp
        password: "{{ vault_mysql_app_password }}"
        priv: "myapp_db.*:ALL"

  roles:
    - geerlingguy.mysql
```

**Most popular Galaxy roles:**

```
geerlingguy.nginx        →  Nginx web server
geerlingguy.mysql        →  MySQL database
geerlingguy.docker       →  Docker engine
geerlingguy.java         →  Java JDK
geerlingguy.nodejs       →  Node.js
geerlingguy.postgresql   →  PostgreSQL
geerlingguy.redis        →  Redis cache
dev-sec.os-hardening     →  Security hardening
dev-sec.ssh-hardening    →  SSH hardening
```

**Always pin versions in `requirements.yml`.** A Galaxy role update can break your playbook. Pin to a known good version and update consciously.

---

### Molecule — Test Your Ansible Roles

**The problem without testing:**

```
You write a role → run it on your server → it works
Teammate runs it on a different OS → it breaks
You make a small change → run on prod → something breaks
You have no idea if your change broke anything until it's too late
```

**Molecule is a testing framework for Ansible roles.** It spins up a fresh container or VM, runs your role against it, runs tests to verify it worked, and destroys the container — all automatically.

```
molecule test
  ↓
Spin up fresh Docker container
  ↓
Run your role against it
  ↓
Run tests (verify the role did what it should)
  ↓
Destroy container
  ↓
Pass or Fail
```

**Install Molecule:**

```bash
pip install molecule molecule-docker --break-system-packages

# Verify
molecule --version
```

**Add Molecule to an existing role:**

```bash
cd roles/nginx
molecule init scenario   # creates the molecule/ folder
```

**Or create a new role with Molecule already set up:**

```bash
molecule init role my-new-role --driver-name docker
```

**What Molecule creates:**

```
roles/nginx/
└── molecule/
    └── default/
        ├── molecule.yml        ← which driver (Docker/Vagrant/EC2)
        ├── converge.yml        ← playbook that runs your role
        ├── verify.yml          ← tests to check role worked
        └── prepare.yml         ← optional setup before role runs
```

**`molecule/default/molecule.yml`:**

```yaml
---
dependency:
  name: galaxy

driver:
  name: docker               # use Docker containers for testing

platforms:
  - name: ubuntu22           # test on Ubuntu 22.04
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true

  - name: ubuntu20           # also test on Ubuntu 20.04
    image: geerlingguy/docker-ubuntu2004-ansible
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible              # use Ansible tasks to verify
```

**`molecule/default/converge.yml` — runs your role:**

```yaml
---
- name: Converge
  hosts: all
  become: yes

  vars:
    http_port: 80

  roles:
    - role: nginx              # the role being tested
```

**`molecule/default/verify.yml` — check the role actually worked:**

```yaml
---
- name: Verify
  hosts: all
  become: yes

  tasks:

    - name: Check nginx is installed
      command: nginx -v
      changed_when: false

    - name: Check nginx is running
      service_facts:

    - name: Assert nginx is running
      assert:
        that:
          - "'nginx' in services"
          - "services['nginx'].state == 'running'"
        fail_msg: "nginx is NOT running — role failed"
        success_msg: "nginx is running — role passed"

    - name: Check port 80 is listening
      wait_for:
        port: 80
        timeout: 5
        msg: "Port 80 is not open — role failed"
```

**Run Molecule:**

```bash
# Full test cycle (create → converge → verify → destroy)
molecule test

# Just run the role without destroying (for debugging)
molecule converge

# Run only the verify tests
molecule verify

# SSH into the test container to investigate
molecule login

# Destroy test containers
molecule destroy

# Test against all platforms in molecule.yml
molecule test --all
```

**Molecule in GitHub Actions:**

```yaml
- name: Test Ansible role with Molecule
  run: |
    pip install molecule molecule-docker
    cd roles/nginx
    molecule test
  env:
    PY_COLORS: '1'
    ANSIBLE_FORCE_COLOR: '1'
```

**The golden rule:** Write a Molecule test for every role you write. If you skip testing, you will break production eventually.

---

### Ansible Tower / AWX — Ansible with a Web UI

**What is AWX / Tower?**

AWX is the open-source version. Ansible Tower is the paid Red Hat enterprise version. They are the same product — AWX is free, Tower is supported.

They give you:

```
Without AWX/Tower:              With AWX/Tower:
  Run playbooks from terminal     Web UI to run playbooks
  Everyone needs Ansible          Non-technical people can trigger runs
  installed locally               Scheduled runs (like cron)
  No audit trail                  Full history — who ran what, when, result
  Secrets in files                Credential vault built in
  No RBAC                         Role-based access — dev team can't touch prod
```

**Run AWX locally with Docker:**

```bash
git clone https://github.com/ansible/awx.git
cd awx
docker compose up -d
```

Access at `http://localhost` — default: admin / password

**Key AWX concepts — in simple English:**

```
Inventory       →  your server list (static file or dynamic from AWS/Azure)
Credentials     →  SSH keys, vault passwords, cloud API keys (stored securely)
Project         →  your Git repo containing playbooks
Job Template    →  one playbook + one inventory + one credential = ready to run
Job             →  one run of a Job Template (has logs, status, output)
Workflow        →  chain multiple Job Templates together in sequence
Schedule        →  run a Job Template automatically at a time (like cron)
```

**Typical AWX setup for a team:**

```
1. Connect Git repo as a Project  →  AWX pulls your playbooks automatically
2. Add SSH key as a Credential    →  no one sees the raw key
3. Add Vault password as a Credential → encrypted in AWX database
4. Create Inventory from repo     →  inventory.ini from your Git
5. Create Job Template            →  link project + inventory + credential
6. Set RBAC                       →  dev team can run staging, ops team runs prod
7. Schedule nightly run           →  servers stay in desired state automatically
```

**AWX REST API — trigger a playbook from any system:**

```bash
# Trigger a job template via API (for integration with other tools)
curl -X POST \
  https://awx.mycompany.com/api/v2/job_templates/42/launch/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"env": "prod", "version": "1.2.0"}}'
```

This means Jenkins, GitHub Actions, or any CI tool can trigger Ansible runs via API — without having Ansible installed on the CI runner.

---

### Windows Automation

Ansible can manage Windows servers too — not just Linux. Instead of SSH, it uses WinRM (Windows Remote Management).

**How Windows connection works:**

```
Linux servers:    Ansible → SSH → server
Windows servers:  Ansible → WinRM → server
```

**Set up WinRM on the Windows server (run in PowerShell as Admin):**

```powershell
# Enable WinRM
winrm quickconfig -q

# Allow Ansible to connect
Set-Item WSMan:\localhost\Service\Auth\Basic -Value $true
Set-Item WSMan:\localhost\Service\AllowUnencrypted -Value $true

# Or use the Ansible-provided script (easier)
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"
(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
powershell.exe -ExecutionPolicy ByPass -File $file
```

**`inventory.ini` for Windows:**

```ini
[windows]
192.168.1.30
192.168.1.31

[windows:vars]
ansible_user=Administrator
ansible_password={{ vault_windows_password }}
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
ansible_port=5985
```

**Windows-specific modules:**

```yaml
---
- name: Configure Windows servers
  hosts: windows
  gather_facts: yes

  tasks:

    # Install software using Chocolatey (Windows package manager)
    - name: Install Chrome
      win_chocolatey:
        name: googlechrome
        state: present

    # Install Windows features
    - name: Install IIS web server
      win_feature:
        name: Web-Server
        state: present
        include_management_tools: yes

    # Manage Windows services
    - name: Start IIS service
      win_service:
        name: W3SVC
        state: started
        start_mode: auto

    # Copy files to Windows
    - name: Copy app config
      win_copy:
        src: app.config
        dest: C:\inetpub\wwwroot\app.config

    # Run PowerShell commands
    - name: Set environment variable
      win_environment:
        name: APP_ENV
        value: production
        level: machine

    # Create Windows user
    - name: Create service account
      win_user:
        name: svc_myapp
        password: "{{ vault_svc_password }}"
        password_never_expires: yes
        user_cannot_change_password: yes
        groups:
          - IIS_IUSRS

    # Manage Windows registry
    - name: Set registry key
      win_regedit:
        path: HKLM:\Software\MyApp
        name: Environment
        data: production
        type: string

    # Reboot if needed
    - name: Reboot Windows server
      win_reboot:
        reboot_timeout: 300
      when: reboot_required
```

**Windows vs Linux modules:**

```
Linux module    →  Windows equivalent
────────────       ──────────────────
apt / yum       →  win_chocolatey
copy            →  win_copy
file            →  win_file
service         →  win_service
user            →  win_user
command         →  win_command
shell           →  win_shell / win_powershell
```

---

### Network Device Automation

Ansible can configure routers, switches, and firewalls — not just servers. This is huge for network teams who used to manually SSH into each device.

**Supported network vendors:**

```
Cisco IOS        →  ios_*  modules
Cisco NX-OS      →  nxos_* modules
Juniper Junos    →  junos_* modules
Arista EOS       →  eos_*  modules
F5 BIG-IP        →  bigip_* modules
Palo Alto        →  panos_* modules
```

**How network connections work:**

```
Servers:          Ansible connects → runs Python on the server
Network devices:  Ansible connects → sends CLI commands or API calls
                  No Python needed on the device
```

**`inventory.ini` for network devices:**

```ini
[routers]
router1 ansible_host=192.168.1.1
router2 ansible_host=192.168.1.2

[routers:vars]
ansible_network_os=ios
ansible_connection=network_cli
ansible_user=admin
ansible_password={{ vault_network_password }}
ansible_become=yes
ansible_become_method=enable
ansible_become_password={{ vault_enable_password }}

[switches]
switch1 ansible_host=192.168.1.10

[switches:vars]
ansible_network_os=nxos
ansible_connection=network_cli
ansible_user=admin
ansible_password={{ vault_network_password }}
```

**Network automation playbook:**

```yaml
---
- name: Configure network devices
  hosts: routers
  gather_facts: no       # IMPORTANT — no gather_facts for network devices

  tasks:

    # Get device facts
    - name: Gather IOS facts
      ios_facts:
        gather_subset: all

    - name: Show device info
      debug:
        msg: "Device: {{ ansible_net_hostname }}, IOS: {{ ansible_net_version }}"

    # Configure interfaces
    - name: Configure Gigabit interface
      ios_interface:
        name: GigabitEthernet0/1
        description: "Uplink to core switch"
        enabled: yes

    # Configure VLANs
    - name: Configure VLANs
      ios_vlan:
        vlan_id: 100
        name: SERVERS
        state: present

    # Push full config from a template
    - name: Push interface config
      ios_config:
        src: interface_config.j2    # Jinja2 template with interface settings

    # Save config so it survives a reboot
    - name: Save running config
      ios_config:
        save_when: always

    # Backup current config
    - name: Backup device config
      ios_config:
        backup: yes
        backup_options:
          filename: "{{ ansible_net_hostname }}_backup.cfg"
          dir_path: ./backups/
```

**Why network automation matters in DevOps:**

```
Before:  Network engineer SSHs into 50 switches, types commands manually
         Takes hours, typos happen, no audit trail

After:   ansible-playbook network-config.yml
         All 50 devices configured identically in 2 minutes
         Full audit trail in Git — who changed what, when
```

---

### Performance Tuning — Fast Ansible on Large Fleets

When you manage 500 servers, a slow Ansible run that takes 30 minutes blocks everything. Here's how to make it fast.

**1. Parallelism — run on many servers at once:**

```ini
# ansible.cfg
[defaults]
forks = 50      # run on 50 servers at the same time (default is 5)
```

```bash
# Override at command line
ansible-playbook playbook.yml -f 50
```

**2. SSH multiplexing — reuse SSH connections:**

```ini
# ansible.cfg
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True    # send multiple tasks over one SSH connection (big speedup)
```

`pipelining = True` alone can make playbooks 2–5x faster. It reduces the number of SSH connections by sending multiple commands over one connection.

**3. Fact caching — don't re-collect facts every run:**

```ini
# ansible.cfg
[defaults]
gathering = smart           # only gather facts if not cached
fact_caching = jsonfile     # cache to disk as JSON files
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 3600 # cache for 1 hour
```

Gathering facts takes 1–3 seconds per server. On 200 servers that's 200–600 seconds saved.

**4. Async tasks — don't wait for slow tasks:**

```yaml
# Normal — Ansible waits for each task to finish before moving on
- name: Run long backup job
  command: /usr/bin/backup.sh

# Async — start the task and move on, check later
- name: Start backup job (async)
  command: /usr/bin/backup.sh
  async: 3600       # let it run for up to 1 hour
  poll: 0           # don't wait — move on immediately
  register: backup_job

# Later — check if it finished
- name: Check backup job status
  async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30         # check every 30 seconds
```

**5. Only run what changed — `--check` and `--diff`:**

```bash
# See what WOULD change without actually changing anything
ansible-playbook playbook.yml --check

# See the exact diff of file changes
ansible-playbook playbook.yml --check --diff
```

**6. Limit scope — don't run on everything every time:**

```bash
# Run only on specific host
ansible-playbook playbook.yml --limit "web01.mycompany.com"

# Run only on specific group
ansible-playbook playbook.yml --limit "webservers"

# Run only on first 10 servers in a group
ansible-playbook playbook.yml --limit "webservers[0:9]"
```

**7. Strategy — free strategy runs tasks independently per host:**

```yaml
---
- name: Fast deployment
  hosts: webservers
  strategy: free      # each host runs as fast as it can, no waiting for others
  # default strategy = linear (all hosts do task 1, then all do task 2...)
  # free strategy = host1 might be on task 5 while host2 is still on task 2
```

**Performance checklist:**

```
forks = 50              →  2–10x speedup on large fleets
pipelining = True       →  2–5x speedup (biggest single win)
fact caching            →  save 1–3s per server
async for slow tasks    →  don't block on long-running jobs
strategy: free          →  better parallelism
--limit                 →  only touch what you need
--tags                  →  only run relevant tasks
```

---

### Ansible Best Practices — How Real Teams Use It

**1. Always use roles — never write giant single playbooks:**

```
Bad:  one 500-line playbook.yml that does everything
Good: site.yml calling roles: common, nginx, app, monitoring
```

**2. Use group_vars and host_vars — not vars inside playbooks:**

```
group_vars/
├── all/
│   ├── vars.yml        ← variables for every server
│   └── secrets.yml     ← encrypted vault file
├── webservers/
│   └── vars.yml        ← variables only for webservers
└── dbservers/
    └── vars.yml        ← variables only for DB servers

host_vars/
└── web01.mycompany.com/
    └── vars.yml        ← variables only for this one server
```

**3. Name every single task clearly:**

```yaml
# Bad
- apt:
    name: nginx
    state: present

# Good
- name: Install nginx web server
  apt:
    name: nginx
    state: present
```

**4. Use `ansible.cfg` in every project:**

```ini
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
roles_path = ./roles
forks = 20
retry_files_enabled = False     # don't create annoying .retry files

[ssh_connection]
pipelining = True
```

**5. Always test with `--check` before running on production:**

```bash
# Dry run on staging first
ansible-playbook -i staging.ini site.yml --check

# Then real run on staging
ansible-playbook -i staging.ini site.yml

# Then real run on production
ansible-playbook -i production.ini site.yml
```

**6. Use `assert` to validate before making changes:**

```yaml
- name: Check we are on the right server before making changes
  assert:
    that:
      - ansible_hostname != "prod-db-01"    # never run this on the prod DB
      - ansible_memtotal_mb >= 2048          # server needs at least 2GB RAM
    fail_msg: "Safety check failed — aborting"
```

**7. Tag everything — at minimum tag by component:**

```yaml
- name: Install nginx
  apt:
    name: nginx
  tags: [nginx, install, webserver]

- name: Deploy app code
  git:
    repo: ...
  tags: [app, deploy]
```

**8. Never store secrets in plaintext — vault everything:**

```yaml
# Bad — plaintext in vars.yml
db_password: mypassword

# Good — reference a vault-encrypted variable
db_password: "{{ vault_db_password }}"
```

**9. Version control your inventory:**

```
inventory/
├── production/
│   ├── hosts           ← production servers
│   └── group_vars/
└── staging/
    ├── hosts           ← staging servers
    └── group_vars/
```

**10. Use `ansible-lint` — catch mistakes before running:**

```bash
pip install ansible-lint --break-system-packages

# Check a playbook
ansible-lint playbook.yml

# Check a role
ansible-lint roles/nginx/
```

Common things ansible-lint catches:
- Tasks missing `name:`
- Using `shell` when `command` would do
- Deprecated module names
- YAML formatting issues

---

### Full Commands Cheat Sheet — Day 3

```bash
# Galaxy
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install amazon.aws
ansible-galaxy list                              # list installed roles

# Molecule
molecule test                  # full test cycle
molecule converge              # run role without destroying
molecule verify                # run tests only
molecule login                 # SSH into test container
molecule destroy               # remove test containers

# Performance
ansible-playbook playbook.yml -f 50             # 50 parallel forks
ansible-playbook playbook.yml --check --diff    # dry run with diff

# Limiting scope
ansible-playbook playbook.yml --limit "web01"
ansible-playbook playbook.yml --limit "webservers[0:4]"  # first 5

# Debugging
ansible-playbook playbook.yml -v     # verbose
ansible-playbook playbook.yml -vv    # more verbose
ansible-playbook playbook.yml -vvv   # connection debugging

# Linting
ansible-lint playbook.yml
ansible-lint roles/

# Syntax check (no run)
ansible-playbook playbook.yml --syntax-check

# Step through tasks one by one
ansible-playbook playbook.yml --step

# Ad-hoc commands (quick one-liners without a playbook)
ansible webservers -m command -a "uptime" -i inventory.ini
ansible all -m shell -a "df -h" -i inventory.ini
ansible webservers -m service -a "name=nginx state=restarted" -i inventory.ini
```

---

### Interview Questions — Ansible Day 3

**Q: What is Ansible Galaxy?**
Ansible Galaxy is the community repository for Ansible roles and collections. You can download pre-written, tested roles for common tasks like installing MySQL or configuring nginx using `ansible-galaxy install`. It saves time and you get roles that are already tested across multiple OS versions.

**Q: What is Molecule and why should you use it?**
Molecule is a testing framework for Ansible roles. It spins up a fresh Docker container, runs your role against it, then runs verification tests to confirm the role did what it should. It destroys the container after. Using Molecule means you catch bugs in your roles before they hit real servers.

**Q: What is the difference between AWX and Ansible Tower?**
AWX is the open-source upstream project. Ansible Tower is the paid Red Hat enterprise product built from AWX. Both give you a web UI, REST API, scheduling, RBAC, and credential management for Ansible. AWX is free but unsupported. Tower comes with Red Hat support.

**Q: How does Ansible connect to Windows servers?**
Ansible connects to Windows using WinRM (Windows Remote Management) instead of SSH. You set `ansible_connection=winrm` in the inventory. Windows-specific modules like `win_chocolatey`, `win_service`, and `win_copy` are used instead of the Linux equivalents.

**Q: What is `pipelining` in Ansible and why does it improve performance?**
Pipelining sends multiple Ansible tasks over a single SSH connection instead of opening a new connection for each task. This dramatically reduces SSH overhead. Setting `pipelining = True` in `ansible.cfg` can make playbooks 2–5x faster on large fleets.

**Q: What is `strategy: free` in Ansible?**
The default strategy is `linear` — all hosts complete task 1 before anyone moves to task 2. The `free` strategy lets each host run as fast as it can independently. If one server is slow, others don't wait for it. This gives better overall throughput on large fleets.

**Q: What is `async` in Ansible?**
`async` lets you start a long-running task without blocking the playbook. You set a maximum runtime with `async:` and set `poll: 0` to not wait. You can check the job status later using `async_status`. Useful for backups, long compilations, or database migrations.

**Q: What is `ansible-lint`?**
A static analysis tool that checks your playbooks and roles for common mistakes, bad practices, and deprecated syntax before you run them. It catches things like missing task names, using `shell` when `command` is safer, and YAML formatting issues.

---

### End of Day 3 Checklist

- [ ] `ansible-galaxy install -r requirements.yml` works with a requirements file
- [ ] A Galaxy role (e.g. `geerlingguy.nginx`) used in a playbook with custom vars
- [ ] Molecule installed — `molecule --version` works
- [ ] `molecule test` runs and passes for one of your roles
- [ ] `molecule verify` tests written for a role
- [ ] AWX running locally via Docker — job template created and run
- [ ] `forks` and `pipelining` set in `ansible.cfg`
- [ ] `--check --diff` used on a playbook to preview changes
- [ ] `ansible-lint` installed and run on a playbook
- [ ] group_vars structure used instead of vars inside playbooks
- [ ] All 8 interview questions answered in your own words

---

*Next — Ansible Day 4 (Final): Ansible with Terraform (full infra + config pipeline), Kubernetes management with Ansible, secret management with HashiCorp Vault, Ansible in GitLab CI, real-world production project — full stack deployment from zero*