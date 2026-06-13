# Ansible Notes

---

## Ansible Day 4 — Terraform, Kubernetes, HashiCorp Vault & Production

---

### What Day 3 Covered (Quick Recap)

Before Day 4 — make sure these are solid:

```
Ansible Galaxy      →  download community roles, requirements.yml, pin versions
Molecule            →  test roles with Docker containers, verify tasks
AWX / Tower         →  web UI, REST API, RBAC, scheduling for Ansible at scale
Windows automation  →  WinRM connection, win_* modules
Network automation  →  configure routers/switches with ios_*, nxos_* modules
Performance tuning  →  forks, pipelining, fact caching, async, strategy: free
Best practices      →  roles, group_vars, tags, ansible-lint, assert safety checks
```

If any of these feel fuzzy — re-read Day 3 before continuing.

---

### What You Will Learn Today

- Ansible + Terraform — the complete infra + config pipeline
- Kubernetes management with Ansible
- HashiCorp Vault — external secret management (beyond ansible-vault)
- Ansible in GitLab CI — full enterprise pipeline
- Real-world production project — full stack from zero to running app
- Complete DevOps mental model — how Ansible fits everything together

---

### Ansible + Terraform — The Complete Pipeline

**The most important concept in infrastructure DevOps:**

```
Terraform   →  CREATE the infrastructure  (servers, networks, databases)
Ansible     →  CONFIGURE the infrastructure (install software, deploy apps)

They are not competitors. They work together perfectly.
```

**The full flow:**

```
1. terraform apply        →  creates EC2 instances, RDS, VPC, load balancer
2. Terraform outputs IPs  →  writes server IPs to a file or state
3. ansible-playbook       →  reads those IPs, configures every server
4. App is live            →  fully automated, zero manual steps
```

**Method 1 — Terraform writes inventory file, Ansible reads it:**

```hcl
# main.tf — Terraform creates servers and writes inventory

resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"

  tags = {
    Name        = "web-${count.index + 1}"
    Role        = "webserver"
    Environment = "prod"
  }
}

# Write Ansible inventory file after servers are created
resource "local_file" "ansible_inventory" {
  content = templatefile("inventory.tpl", {
    web_servers = aws_instance.web[*].public_ip
    db_host     = aws_db_instance.main.address
  })
  filename = "../ansible/inventory/production.ini"
}

output "web_server_ips" {
  value = aws_instance.web[*].public_ip
}
```

**`inventory.tpl` — template for the inventory file:**

```ini
[webservers]
%{ for ip in web_servers ~}
${ip}
%{ endfor ~}

[dbservers]
${db_host}

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/prod_key.pem
```

**After `terraform apply`, Ansible has a fresh inventory automatically:**

```bash
# Full pipeline in one script
terraform apply -auto-approve
sleep 30   # wait for servers to boot

ansible-playbook \
  -i ansible/inventory/production.ini \
  ansible/site.yml \
  --vault-password-file ~/.vault_pass
```

**Method 2 — Use Terraform dynamic inventory plugin (cleanest way):**

```bash
pip install terraform-inventory --break-system-packages
```

```bash
# Ansible reads Terraform state directly — no intermediate file needed
ansible-playbook \
  -i /usr/local/bin/terraform-inventory \
  site.yml
```

**Method 3 — Use AWS dynamic inventory (after Terraform creates tagged resources):**

Terraform creates EC2 with tags → AWS dynamic inventory reads those tags → Ansible groups servers by tag automatically. This is the most production-grade approach covered in Day 2.

**Full Terraform + Ansible pipeline script:**

```bash
#!/bin/bash
# deploy.sh — full infrastructure + configuration pipeline

set -e   # stop on any error

echo "=== Step 1: Create infrastructure with Terraform ==="
cd terraform/
terraform init
terraform plan -out=tfplan
terraform apply tfplan

echo "=== Step 2: Wait for servers to be ready ==="
sleep 60

echo "=== Step 3: Configure servers with Ansible ==="
cd ../ansible/
ansible-playbook \
  -i inventory/aws_ec2.yml \
  --vault-password-file ~/.vault_pass \
  site.yml

echo "=== Deployment complete ==="
```

---

### Kubernetes Management with Ansible

Ansible can manage Kubernetes clusters — create namespaces, apply manifests, manage Helm charts, and more. Useful when you want one tool to manage both your servers AND your Kubernetes workloads.

**Install the Kubernetes collection:**

```bash
ansible-galaxy collection install kubernetes.core
pip install kubernetes --break-system-packages
```

**Connect to your Kubernetes cluster:**

```yaml
# The kubernetes.core modules use your kubeconfig automatically
# Make sure KUBECONFIG env variable points to the right config
# or specify it in the playbook
```

**Ansible playbook to manage Kubernetes:**

```yaml
---
- name: Manage Kubernetes resources
  hosts: localhost          # runs from your local machine, not on servers
  connection: local         # no SSH needed — talks to K8s API directly
  gather_facts: no

  vars:
    kubeconfig: ~/.kube/config
    namespace: myapp

  tasks:

    # Create a namespace
    - name: Create application namespace
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig }}"
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"

    # Create a secret in Kubernetes from vault variable
    - name: Create database credentials secret
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig }}"
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: db-credentials
            namespace: "{{ namespace }}"
          type: Opaque
          stringData:
            DB_PASSWORD: "{{ vault_db_password }}"
            DB_USERNAME: "{{ db_username }}"

    # Apply a deployment manifest
    - name: Deploy application
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig }}"
        state: present
        src: manifests/deployment.yml    # apply a YAML file

    # Or apply inline definition
    - name: Create service
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig }}"
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: myapp-service
            namespace: "{{ namespace }}"
          spec:
            selector:
              app: myapp
            ports:
              - port: 80
                targetPort: 8080

    # Wait for deployment to be ready
    - name: Wait for deployment to be ready
      kubernetes.core.k8s_info:
        kubeconfig: "{{ kubeconfig }}"
        kind: Deployment
        name: myapp
        namespace: "{{ namespace }}"
        wait: yes
        wait_condition:
          type: Available
          status: "True"
        wait_timeout: 120

    # Get pod info
    - name: Get running pods
      kubernetes.core.k8s_info:
        kubeconfig: "{{ kubeconfig }}"
        kind: Pod
        namespace: "{{ namespace }}"
        label_selectors:
          - app=myapp
      register: pods

    - name: Show pod names
      debug:
        msg: "Running pod: {{ item.metadata.name }}"
      loop: "{{ pods.resources }}"
```

**Manage Helm charts with Ansible:**

```yaml
    # Install a Helm chart
    - name: Install nginx ingress controller
      kubernetes.core.helm:
        kubeconfig: "{{ kubeconfig }}"
        name: nginx-ingress
        chart_ref: ingress-nginx/ingress-nginx
        release_namespace: ingress-nginx
        create_namespace: true
        values:
          controller:
            replicaCount: 2
            service:
              type: LoadBalancer

    # Upgrade a Helm release
    - name: Upgrade myapp chart
      kubernetes.core.helm:
        kubeconfig: "{{ kubeconfig }}"
        name: myapp
        chart_ref: ./charts/myapp
        release_namespace: "{{ namespace }}"
        values:
          image:
            tag: "{{ app_version }}"
          replicas: 3
```

---

### HashiCorp Vault — Enterprise Secret Management

**Ansible Vault vs HashiCorp Vault — what's the difference?**

```
Ansible Vault          →  encrypts secrets inside your playbooks/files
                          Simple, built-in, good for small teams

HashiCorp Vault        →  a dedicated secret server
                          Secrets stored centrally, accessed via API
                          Fine-grained access control, secret rotation
                          Audit log of every secret access
                          Used by large enterprises
```

```
Small team (5 people):     Ansible Vault is enough
Large team (50+ people):   HashiCorp Vault is the right choice
```

**Run HashiCorp Vault locally:**

```bash
docker run -d \
  --name vault \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=myroot \
  hashicorp/vault:latest

export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=myroot
```

**Store secrets in HashiCorp Vault:**

```bash
# Enable the KV secrets engine
vault secrets enable -path=secret kv-v2

# Store secrets
vault kv put secret/myapp \
  db_password=SuperSecret123 \
  api_key=abc123xyz \
  jwt_secret=myjwtsecret

# Read them back
vault kv get secret/myapp
```

**Pull secrets from HashiCorp Vault in Ansible:**

```bash
pip install hvac --break-system-packages   # HashiCorp Vault Python client
ansible-galaxy collection install community.hashi_vault
```

```yaml
---
- name: Deploy with HashiCorp Vault secrets
  hosts: webservers
  become: yes

  vars:
    vault_addr: "http://vault.mycompany.com:8200"

  tasks:

    # Read secrets from HashiCorp Vault
    - name: Get database password from Vault
      community.hashi_vault.hashi_vault_kv2_get:
        url: "{{ vault_addr }}"
        path: secret/myapp
        token: "{{ lookup('env', 'VAULT_TOKEN') }}"
      register: vault_secrets

    # Use the secret
    - name: Write app config with DB password
      template:
        src: app.config.j2
        dest: /etc/myapp/config.yml
      vars:
        db_password: "{{ vault_secrets.secret.db_password }}"
        api_key: "{{ vault_secrets.secret.api_key }}"

    # Or use lookup directly in a task
    - name: Create env file with secrets
      copy:
        content: |
          DB_PASSWORD={{ lookup('community.hashi_vault.hashi_vault',
            'secret/data/myapp:db_password',
            url=vault_addr,
            token=lookup('env', 'VAULT_TOKEN')) }}
        dest: /etc/myapp/.env
        mode: '0600'
```

**In CI/CD — authenticate to Vault using AppRole (no static token):**

```yaml
# In GitHub Actions:
- name: Get Vault token via AppRole
  run: |
    VAULT_TOKEN=$(curl -s \
      --request POST \
      --data '{"role_id":"${{ secrets.VAULT_ROLE_ID }}","secret_id":"${{ secrets.VAULT_SECRET_ID }}"}' \
      $VAULT_ADDR/v1/auth/approle/login | jq -r '.auth.client_token')
    echo "VAULT_TOKEN=$VAULT_TOKEN" >> $GITHUB_ENV

- name: Run Ansible with Vault secrets
  run: ansible-playbook site.yml
  env:
    VAULT_TOKEN: ${{ env.VAULT_TOKEN }}
    VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
```

---

### Ansible in GitLab CI — Full Enterprise Pipeline

```yaml
# .gitlab-ci.yml — complete Ansible pipeline

image: python:3.11-slim

variables:
  ANSIBLE_HOST_KEY_CHECKING: "False"
  ANSIBLE_FORCE_COLOR: "1"
  PY_COLORS: "1"

# Cache pip packages between jobs
cache:
  paths:
    - .pip_cache/

before_script:
  - pip install --cache-dir .pip_cache ansible ansible-lint molecule molecule-docker
  - ansible --version
  - ansible-galaxy install -r requirements.yml

stages:
  - lint
  - test
  - deploy-staging
  - approve
  - deploy-production

# ─── Stage 1: Lint ───────────────────────────────────────────
lint:
  stage: lint
  script:
    - ansible-lint site.yml
    - ansible-playbook site.yml --syntax-check -i inventory/staging/
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Stage 2: Test Roles with Molecule ───────────────────────
molecule-test:
  stage: test
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - cd roles/nginx && molecule test
    - cd ../../roles/myapp && molecule test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Stage 3: Deploy to Staging ──────────────────────────────
deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.myapp.com
  before_script:
    - pip install ansible
    - ansible-galaxy install -r requirements.yml
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "$ANSIBLE_VAULT_PASSWORD" > ~/.vault_pass
    - chmod 600 ~/.vault_pass
  script:
    - ansible-playbook
        -i inventory/staging/
        --vault-password-file ~/.vault_pass
        --private-key ~/.ssh/id_rsa
        site.yml
  after_script:
    - rm -f ~/.ssh/id_rsa ~/.vault_pass
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Stage 4: Manual approval before production ──────────────
approve-production:
  stage: approve
  script:
    - echo "Waiting for manual approval to deploy to production"
  when: manual           # someone must click Approve in GitLab UI
  allow_failure: false
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─── Stage 5: Deploy to Production ───────────────────────────
deploy-production:
  stage: deploy-production
  environment:
    name: production
    url: https://myapp.com
  before_script:
    - pip install ansible
    - ansible-galaxy install -r requirements.yml
    - mkdir -p ~/.ssh
    - echo "$PROD_SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "$PROD_VAULT_PASSWORD" > ~/.vault_pass
    - chmod 600 ~/.vault_pass
  script:
    - ansible-playbook
        -i inventory/production/
        --vault-password-file ~/.vault_pass
        --private-key ~/.ssh/id_rsa
        site.yml
  after_script:
    - rm -f ~/.ssh/id_rsa ~/.vault_pass
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      needs: ["approve-production"]
```

---

### Real-World Production Project — Full Stack from Zero

Let's build the complete thing. Terraform creates the infrastructure. Ansible configures it. A Node.js app runs behind Nginx. Fully automated. Zero manual steps.

**What we're building:**

```
Internet
    ↓
Load Balancer (AWS ALB)
    ↓
2x Web Servers (EC2 — Ubuntu 22.04)
    → Nginx (reverse proxy on port 80)
    → Node.js app (port 3000)
    ↓
Database (RDS PostgreSQL)
```

**Complete project structure:**

```
myproject/
├── terraform/
│   ├── main.tf             ← EC2, RDS, VPC, security groups
│   ├── outputs.tf          ← output IPs for Ansible
│   └── variables.tf
├── ansible/
│   ├── ansible.cfg
│   ├── site.yml            ← master playbook
│   ├── requirements.yml    ← Galaxy roles
│   ├── inventory/
│   │   └── aws_ec2.yml     ← dynamic inventory
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── vars.yml
│   │   │   └── secrets.yml  ← vault encrypted
│   │   └── webservers/
│   │       └── vars.yml
│   └── roles/
│       ├── common/
│       ├── nodejs/
│       ├── myapp/
│       └── nginx/
├── deploy.sh               ← one command to deploy everything
└── .github/
    └── workflows/
        └── deploy.yml
```

**`ansible/group_vars/all/vars.yml`:**

```yaml
app_name: myapp
app_user: deploy
app_dir: /var/www/myapp
app_port: 3000
node_version: "20"
git_repo: "https://github.com/myorg/myapp.git"
git_branch: "{{ lookup('env', 'DEPLOY_BRANCH') | default('main') }}"
http_port: 80
server_name: myapp.com
deploy_timestamp: "{{ ansible_date_time.iso8601 }}"
```

**`ansible/group_vars/all/secrets.yml` (vault encrypted):**

```yaml
vault_db_password: SuperSecretDBPassword
vault_app_secret: MyAppJWTSecret
vault_newrelic_key: abc123monitoring
```

**`ansible/site.yml` — master playbook:**

```yaml
---
- name: Configure all web servers
  hosts: role_webserver    # from dynamic inventory EC2 tag
  become: yes
  serial: 1                # deploy one server at a time (rolling update)
                           # at any time, only 1 server is out of rotation

  pre_tasks:
    - name: Remove server from load balancer before update
      command: >
        aws elbv2 deregister-targets
        --target-group-arn {{ lb_target_group_arn }}
        --targets Id={{ instance_id }}
      delegate_to: localhost
      when: rolling_update | default(false)

  roles:
    - common
    - nodejs
    - myapp
    - nginx

  post_tasks:
    - name: Wait for app to be healthy
      uri:
        url: "http://{{ ansible_host }}:{{ app_port }}/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: Add server back to load balancer
      command: >
        aws elbv2 register-targets
        --target-group-arn {{ lb_target_group_arn }}
        --targets Id={{ instance_id }}
      delegate_to: localhost
      when: rolling_update | default(false)
```

**`ansible/roles/myapp/tasks/main.yml`:**

```yaml
---
- name: Pull latest app code
  git:
    repo: "{{ git_repo }}"
    dest: "{{ app_dir }}"
    version: "{{ git_branch }}"
    force: yes
  become_user: "{{ app_user }}"
  register: git_result
  notify: Restart app

- name: Install npm dependencies
  command: npm ci --production
  args:
    chdir: "{{ app_dir }}"
  become_user: "{{ app_user }}"
  when: git_result.changed
  notify: Restart app

- name: Run database migrations
  command: node scripts/migrate.js
  args:
    chdir: "{{ app_dir }}"
  become_user: "{{ app_user }}"
  environment:
    DATABASE_URL: "postgresql://myapp:{{ vault_db_password }}@{{ db_host }}/myapp"
  when: git_result.changed

- name: Write app environment file
  template:
    src: env.j2
    dest: "{{ app_dir }}/.env"
    owner: "{{ app_user }}"
    mode: '0600'
  notify: Restart app

- name: Install systemd service
  template:
    src: app.service.j2
    dest: "/etc/systemd/system/{{ app_name }}.service"
  notify: Restart app

- name: Enable and start app service
  systemd:
    name: "{{ app_name }}"
    enabled: yes
    state: started
    daemon_reload: yes

- name: Write deployment record
  lineinfile:
    path: /var/log/deploy.log
    line: "{{ deploy_timestamp }} — deployed {{ git_result.after[:7] }} by {{ lookup('env', 'USER') }}"
    create: yes
```

**`ansible/roles/myapp/templates/env.j2`:**

```
NODE_ENV=production
PORT={{ app_port }}
DATABASE_URL=postgresql://myapp:{{ vault_db_password }}@{{ db_host }}/myapp
JWT_SECRET={{ vault_app_secret }}
NEW_RELIC_LICENSE_KEY={{ vault_newrelic_key }}
```

**`deploy.sh` — one command to rule them all:**

```bash
#!/bin/bash
set -e

BRANCH=${1:-main}
ENV=${2:-staging}

echo "======================================"
echo " Deploying branch: $BRANCH"
echo " Environment:      $ENV"
echo "======================================"

# Step 1: Infrastructure
echo ">>> Creating infrastructure..."
cd terraform/
terraform workspace select $ENV || terraform workspace new $ENV
terraform apply -auto-approve -var="environment=$ENV"
cd ..

# Step 2: Wait for instances to boot
echo ">>> Waiting for instances to be ready..."
sleep 45

# Step 3: Configure servers
echo ">>> Configuring servers with Ansible..."
cd ansible/
ansible-playbook \
  -i inventory/aws_ec2.yml \
  --vault-password-file ~/.vault_pass \
  --extra-vars "env=$ENV git_branch=$BRANCH rolling_update=true" \
  site.yml

echo "======================================"
echo " Deployment complete!"
echo " URL: https://$ENV.myapp.com"
echo "======================================"
```

**Run it:**

```bash
./deploy.sh main staging      # deploy main branch to staging
./deploy.sh main prod         # deploy main branch to production
./deploy.sh feature/xyz staging  # test a feature branch
```

---

### The Complete DevOps Mental Model — Ansible's Place

```
Developer writes code
        ↓
Git push → PR created
        ↓
CI Pipeline (GitHub Actions / GitLab CI)
  ├── ansible-lint            (catch mistakes early)
  ├── molecule test           (test roles in Docker)
  └── Build & test app code
        ↓
Merge to main
        ↓
CD Pipeline triggered
  ├── terraform apply         (create/update infrastructure)
  └── ansible-playbook        (configure servers, deploy app)
        ↓
Staging deployed → smoke tests → manual approval
        ↓
Production deployed (rolling update — zero downtime)
        ↓
Monitoring & alerting (New Relic, Datadog, Prometheus)
        ↓
If alert fires → Ansible playbook to auto-remediate
```

**Every Ansible concept — in one final table:**

| Concept | Simple explanation | Real use |
|---|---|---|
| Inventory | Server list | Which servers to touch |
| Playbook | Instruction file | What to do |
| Task | One action | Install nginx |
| Module | The tool | apt, copy, service |
| Handler | Runs on change | Restart nginx if config changed |
| Role | Organised folder | Reusable nginx setup |
| Variable | Reusable value | Port number, app name |
| Vault | Encrypted secrets | DB passwords |
| Template | Dynamic config file | nginx.conf per server |
| Tag | Label on tasks | Run only nginx tasks |
| When | Conditional | Only on Ubuntu |
| Loop | Repeat for a list | Install 10 packages |
| Register | Save task output | Use result later |
| Block/rescue | Try/catch | Rollback on failure |
| Galaxy | Community roles | Don't write everything |
| Molecule | Role testing | Catch bugs before prod |
| AWX/Tower | Web UI | Non-devs can run playbooks |
| Dynamic inventory | Auto server list | Always fresh from AWS |
| Forks | Parallel servers | Run on 50 at once |
| Pipelining | Fast SSH | 2–5x speed boost |
| HashiCorp Vault | Central secrets | Enterprise secret store |

---

### Interview Questions — Ansible Day 4

**Q: How do Terraform and Ansible work together?**
Terraform provisions the infrastructure — creates EC2 instances, databases, networks. Ansible then configures that infrastructure — installs software, deploys apps, manages files. Terraform handles "what servers exist", Ansible handles "what runs on those servers". They are used together, not as alternatives.

**Q: How do you do a rolling deployment with Ansible?**
Use `serial: 1` (or a number/percentage) in the play. Ansible processes servers one at a time instead of all at once. You use `pre_tasks` to remove the server from the load balancer before updating, and `post_tasks` to verify it's healthy and add it back. This ensures zero downtime.

**Q: What is the difference between Ansible Vault and HashiCorp Vault?**
Ansible Vault encrypts secrets inside your playbook files — simple, built-in, no extra server needed. HashiCorp Vault is a dedicated secret management server with fine-grained access control, secret rotation, audit logging, and dynamic secrets. Small teams use Ansible Vault. Large enterprises use HashiCorp Vault.

**Q: How do you manage Kubernetes with Ansible?**
Using the `kubernetes.core` collection. Ansible talks directly to the Kubernetes API from localhost — no SSH needed. You can create namespaces, apply manifests, manage Helm charts, create secrets, and wait for deployments to be ready, all from Ansible playbooks.

**Q: What is `serial` in Ansible?**
`serial` controls how many hosts Ansible processes at once. `serial: 1` means one host at a time. `serial: 2` means two at a time. `serial: "30%"` means 30% of the fleet at a time. Used for rolling updates so you never take down all servers simultaneously.

**Q: What is `delegate_to` in Ansible?**
`delegate_to` runs a specific task on a different host than the current play target. For example, when deploying to a web server, you can use `delegate_to: localhost` to run a command locally — like calling the AWS API to deregister a server from a load balancer.

**Q: What is `uri` module used for?**
The `uri` module makes HTTP requests from Ansible. Used in post-deploy steps to check that a health endpoint returns 200 OK before declaring the deployment successful and adding the server back to the load balancer.

**Q: How do you handle database migrations in an Ansible deployment?**
Run the migration command as a task — typically with `command` or `shell` — only when the code actually changed (using `when: git_result.changed`). Run it before restarting the app service so the new code always starts against an already-migrated database.

---

### End of Day 4 Checklist

- [ ] Terraform + Ansible pipeline working — `./deploy.sh` provisions and configures servers
- [ ] Dynamic inventory reads EC2 instances created by Terraform
- [ ] Kubernetes playbook applies a deployment and waits for it to be ready
- [ ] HashiCorp Vault running locally — secret stored and read in a playbook
- [ ] GitLab CI pipeline written with lint → test → staging → approve → production
- [ ] Rolling update implemented with `serial: 1` and load balancer tasks
- [ ] `delegate_to: localhost` used for a cloud API call inside a server play
- [ ] Health check with `uri` module in post_tasks
- [ ] Complete DevOps mental model drawn from memory — where Ansible fits
- [ ] All 8 interview questions answered in your own words

---

### Complete Ansible Series — Final Summary

```
Day 1  →  Core concepts: inventory, playbook, task, module, handler, roles, vault basics
Day 2  →  Dynamic inventory, Jinja2 templates, vault deep dive, tags, conditionals,
           loops, error handling, GitHub Actions CI/CD
Day 3  →  Galaxy, Molecule testing, AWX/Tower, Windows, network devices, performance tuning
Day 4  →  Terraform integration, Kubernetes, HashiCorp Vault, GitLab CI,
           full production deployment project, complete DevOps mental model
```

**You now know everything needed to use Ansible professionally in a DevOps role.**

---

*Suggested next series: Terraform — infrastructure as code from scratch. Covers providers, state management, modules, workspaces, remote backends, and full AWS/Azure infrastructure automation.*
