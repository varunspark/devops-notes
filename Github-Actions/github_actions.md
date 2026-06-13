## 🎉 Congratulations — AWS Complete!

## You started AWS as a complete beginner. 10 days later you understand:

## How to design production VPC architectures

## How to secure AWS accounts with IAM

## How to build highly available applications

## How to automate everything with CLI and boto

## How to monitor and alert on everything

## How to optimise costs

## That knowledge puts you in the top tier of DevOps candidates at the 10 - 12 LPA level.

## Say "make AWS document" to get the complete AWS notes packed into a document, or say

## "CI/CD Day 1 " to start the next and second-to-last module! 💪

# CI/CD Day 1 — Introduction & Core Concepts

## Why CI/CD is the Heart of DevOps

## You have learned Linux, Git, Docker and AWS separately. CI/CD is the glue that connects

## all of them into one automated pipeline.

## Without CI/CD:

## With CI/CD:

```
Developer writes code
↓ manually
Runs tests on laptop
↓ manually
Builds Docker image
↓ manually
Pushes to registry
↓ manually
SSHs into server
↓ manually
Pulls image and restarts
↓ manually
Checks if it works
```
```
Total: 30-45 minutes of manual work
Frequency: multiple times per day
Error rate: high — humans make mistakes
```

### This is what DevOps companies pay 10 - 12 LPA for.

## What You Will Learn Today

### What CI/CD is and why it exists

### CI vs CD vs CD — the three terms

### Pipeline stages explained

### Git branching and CI/CD connection

### GitHub Actions — introduction

### Jenkins — introduction

### Choosing between tools

### Key CI/CD concepts — artifacts, environments, rollback

### Real DevOps CI/CD workflow

## What is CI/CD?

### CI/CD stands for:

### These are practices — a way of working — not just tools.

## Continuous Integration (CI)

```
Developer pushes code to Git
↓ automatically
Tests run
↓ automatically
Docker image built
↓ automatically
Image pushed to ECR
↓ automatically
Deployed to production
↓ automatically
Health check verified
```
```
Total: 5-10 minutes, zero human intervention
Frequency: every single commit
Error rate: near zero — same process every time
```
```
CI = Continuous Integration
CD = Continuous Delivery (or)
CD = Continuous Deployment
```

### The practice of merging code changes frequently and automatically verifying each

### change.

## Continuous Delivery vs Continuous Deployment

### These two are different and interviewers test this:

### Most companies use Continuous Delivery — automated pipeline up to a staging

### environment, manual approval for production.

## The Four Stages of a CI/CD Pipeline

```
Developer A pushes code → CI runs automatically:
✓ Code compiles
✓ Unit tests pass
✓ Integration tests pass
✓ Code quality checks pass
✓ Security scan passes
→ "This code is safe to merge"
```
```
Developer B pushes code → same checks run automatically
Developer C pushes code → same checks run automatically
```
```
Everyone's code is always verified
No "it works on my machine" — same environment every time
Problems caught immediately — not weeks later
```
```
Continuous Delivery:
Code → CI passes → Artifact ready → MANUAL approval → Deploy
↑
Human decides when to deploy
Code is READY but not auto-deployed
Used when: regulated industries,
need business sign-off
```
```
Continuous Deployment:
Code → CI passes → Artifact ready → AUTO deploy → Production
↑
No human intervention
Every passing commit goes to prod
Used when: fast-moving products,
high test confidence
```
```
Stage 1 — SOURCE
Trigger: developer pushes to Git
```

## Pipeline as Code

### Modern CI/CD defines pipelines as code files stored in the repository itself:

### Benefits of pipeline as code:

### Version controlled — see history of pipeline changes

### Reviewed in pull requests — pipeline changes need approval

### Reproducible — same pipeline runs every time

### Developer-owned — teams manage their own pipelines

## The Two Main Tools We Cover

### GitHub Actions

```
Action: checkout code, prepare workspace
```
```
Stage 2 — BUILD
Action: compile code, run unit tests
build Docker image
scan image for vulnerabilities
Output: Docker image tagged with commit SHA
```
```
Stage 3 — TEST
Action: deploy to staging environment
run integration tests
run end-to-end tests
performance tests
Output: test report — pass or fail
```
```
Stage 4 — DEPLOY
Action: push image to registry (ECR)
deploy to production
run health checks
notify team
Output: new version live in production
```
```
GitHub Actions → .github/workflows/pipeline.yml
Jenkins → Jenkinsfile
GitLab CI → .gitlab-ci.yml
CircleCI → .circleci/config.yml
```
```
What it is: CI/CD built directly into GitHub
Where it runs: GitHub's cloud infrastructure
```

### Jenkins

## GitHub Actions vs Jenkins

**Feature GitHub Actions Jenkins**

Hosting GitHub cloud Your server

Setup Zero — instant Install + configure

Maintenance None — GitHub manages You manage updates

Cost Free tier generous Free + infra cost

Plugins Marketplace actions 1800 + plugins

Customisation Good Excellent

Enterprise Good Industry standard

Learning curve Low Medium

### For interviews — know both. Most companies use one or the other or both.

## Key CI/CD Concepts

```
Pricing: Free for public repos
2000 minutes/month free for private repos
Config file: .github/workflows/name.yml
Trigger: push, pull request, schedule, manual
Best for: Projects hosted on GitHub
Open source
Teams already using GitHub
```
```
What it is: Self-hosted open source CI/CD server
Where it runs: Your own server (EC2, VM)
Pricing: Free (you pay for infrastructure)
Config file: Jenkinsfile (Groovy)
Trigger: Push, poll, manual, schedule, webhook
Best for: Enterprise environments
Full control and customisation
Complex multi-step pipelines
Teams with existing Jenkins setup
```

### The output of a build step — something that gets stored and passed to the next stage.

### Environment

### A separate deployment target with its own infrastructure:

### Pipeline Trigger

### What starts a pipeline:

```
Source code → BUILD → Artifact
```
```
Types of artifacts:
```
- Docker image (pushed to ECR/Docker Hub)
- JAR/WAR file (Java applications)
- ZIP file (Lambda deployment package)
- tar.gz (compiled binary)
- npm package

```
dev → developer testing, any branch
staging → pre-production, mirrors production, automated tests
production → real users, manual or auto deploy
```
```
Each environment:
```
- Has its own AWS account or VPC
- Has its own database (separate data)
- Has its own configuration (.env values)
- May have different instance sizes

```
yaml
# Push to any branch:
on: push
```
```
# Push to main only:
on:
push:
branches: [main]
```
```
# Pull request:
on:
pull_request:
branches: [main]
```
```
# Schedule (like cron):
```

### Build Status Badge

### A small image showing pipeline status — added to README:

### Shows green ✅ or red ❌ — visible to anyone viewing the repo.

### Rollback

### Reverting production to a previous working version:

## CI/CD and Git Branching — How They Connect

### Different branches trigger different pipeline stages:

```
on:
schedule:
```
- cron: '0 2 * * *' # every day at 2 am

```
# Manual trigger:
on:
workflow_dispatch: # adds Run Workflow button in GitHub
```
```
markdown
```
```
![CI](https://github.com/varun/myapp/actions/workflows/ci.yml/badge.svg)
```
```
bash
```
```
# With Docker — instantly rollback to previous image:
docker pull registry/myapp:v1.2.2 # previous version
docker stop myapp-current
docker run - d --name myapp registry/myapp:v1.2.
```
```
# With Auto Scaling instance refresh — rollback to previous AMI:
aws autoscaling start-instance-refresh \
--auto-scaling-group-name myapp-asg \
--preferences '{"MinHealthyPercentage": 80}'
# After updating Launch Template back to v1.2.
```
```
# Why tagging with git SHA matters:
# You always know exactly which code is running
# You can roll back to any previous commit instantly
```
```
feature/* branches:
→ Run unit tests only (fast feedback)
→ Do NOT deploy anywhere
```

## The DevOps Feedback Loop

### CI/CD creates a tight feedback loop:

### The faster the feedback — the cheaper the bug fix. Finding a bug in CI takes 5 minutes to

### fix. Finding it in production takes hours.

## What a Real Pipeline Does Step by Step

```
develop branch:
→ Run all tests
→ Deploy to dev environment automatically
```
```
main branch:
→ Run all tests
→ Build production Docker image
→ Deploy to staging automatically
→ Manual approval gate
→ Deploy to production
```
```
Tags (v1.0.0, v1.1.0):
→ Full pipeline
→ Deploy to production automatically
→ Create GitHub Release
```
```
Write code (10 minutes)
↓
Push to Git (30 seconds)
↓
Pipeline runs (5 minutes)
↓
Either:
✅ Tests pass → code is in staging in 10 minutes
❌ Tests fail → developer notified in 5 minutes
fix the issue while still in context
not discovering the bug weeks later
```
```
Trigger: developer pushes to main branch
```
```
Step 1 — Checkout:
git clone the repository
checkout the specific commit
```
```
Step 2 — Environment setup:
install dependencies (npm install, pip install)
```

## Pipeline Failure — What Happens

```
set up test database
load environment variables
```
```
Step 3 — Lint and code quality:
run ESLint / flake 8 / pylint
check code style
fail fast if quality below threshold
```
```
Step 4 — Unit tests:
run pytest / jest / junit
generate coverage report
fail if coverage below 80 %
```
```
Step 5 — Build:
docker build -t myapp:$GIT_SHA.
docker scan myapp:$GIT_SHA (security scan)
```
```
Step 6 — Integration tests:
docker-compose up - d (app + dependencies)
run integration test suite
docker-compose down
```
```
Step 7 — Push to registry:
docker tag myapp:$GIT_SHA registry/myapp:$GIT_SHA
docker push registry/myapp:$GIT_SHA
```
```
Step 8 — Deploy to staging:
update EC2/ECS/EKS with new image
wait for health checks to pass
```
```
Step 9 — Smoke tests on staging:
curl staging.myapp.com/health
run critical path tests
```
```
Step 10 — Notify:
Slack message: "v1.2.3 deployed to staging"
Link to test report
Manual approval button for production
```
```
Any step fails → Pipeline stops immediately
→ Developer gets email/Slack notification
→ Production is NOT affected
→ Fix the code → push again → pipeline reruns
```

## Secrets in CI/CD

### Never put passwords or API keys in pipeline files:

### GitHub Secrets are stored encrypted, only injected at pipeline runtime, never visible in logs.

## Pipeline Best Practices

## Real World CI/CD at 10 - 12 LPA Companies

```
This is "fail fast" — stop at the first problem
Better to catch it in CI than in production
```
```
yaml
```
```
# WRONG — secret visible in repository:
```
- run: docker push registry/myapp
    env:
       PASSWORD: supersecret

```
# CORRECT — stored in GitHub Secrets or Jenkins Credentials:
```
- run: docker push registry/myapp
    env:
       PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
1. Fast feedback — unit tests should run in < 5 minutes
2. Fail fast — stop at first failure
3. Idempotent — running pipeline twice gives same result
4. Reproducible — same code always produces same artifact
5. Secure — secrets in vault not in code
6. Visible — team can see pipeline status easily
7. Tested — pipeline itself should be tested
8. Tagged artifacts — every build tagged with git SHA
9. Immutable artifacts — once built never modified
10. One pipeline per environment — not one pipeline doing everything

```
Small startup (10-50 engineers):
→ GitHub Actions → ECR → ECS or EC 2
→ Simple — push to main → auto deploy staging
→ Manual deploy to production
```
```
Medium company (50-500 engineers):
→ Jenkins → ECR → EKS
→ Feature branches → dev → staging → production
```

## Full Summary — CI/CD Day 1

**Concept Key point**

CI Automatically test every code change

Continuous Delivery Auto deploy to staging — manual to production

Continuous Deployment Auto deploy to production — no manual step

Pipeline stages Source → Build → Test → Deploy

Pipeline as code YAML/Groovy file in repository

GitHub Actions Cloud CI/CD — built into GitHub

Jenkins Self-hosted — more control

Artifact Build output — Docker image, JAR, ZIP

Environment dev, staging, production — separate infrastructure

Fail fast Stop at first failure — protect production

Secrets Never in code — use GitHub Secrets or Jenkins Credentials

Rollback Previous Docker image tag — instant recovery

## Interview Questions — CI/CD Day 1

### Q1. What is the difference between Continuous Delivery and Continuous Deployment?

### Answer: Continuous Delivery means every code change is automatically built, tested and

### made ready to deploy — but a human approves when it goes to production. Continuous

### Deployment goes one step further — every passing change is automatically deployed to

### production with no manual step. Most companies use Continuous Delivery — automated

### pipeline up to staging with manual production approval.

### Q2. What is a CI/CD pipeline and what stages does it have?

### Answer: A CI/CD pipeline is an automated sequence of steps that takes code from a

### developer's commit to running in production. Typical stages: Source (code checkout), Build

```
→ Multiple approval gates
→ Automated rollback on health check failure
```
```
Enterprise (500+ engineers):
→ Jenkins or TeamCity → Artifactory → Multiple regions
→ Strict change control
→ Blue/green deployments
→ Canary releases (5% → 25 % → 100 % traffic)
```

### (compile, unit tests, Docker build), Test (integration tests, security scan, deploy to staging),

### Deploy (push to registry, deploy to production, health check). Each stage must pass before

### the next begins.

### Q3. What is pipeline as code and why is it important?

### Answer: Defining the CI/CD pipeline in a file stored in the repository — like

### .github/workflows/pipeline.yml for GitHub Actions or Jenkinsfile for Jenkins.

### Important because the pipeline is version controlled, reviewed in pull requests,

### reproducible and developer-owned. Changes to the pipeline go through the same review

### process as application code.

### Q4. How do you handle secrets in a CI/CD pipeline?

### Answer: Never put secrets in pipeline files — they are committed to Git and visible to

### everyone. Store secrets in GitHub Secrets (for GitHub Actions) or Jenkins Credentials

### Manager (for Jenkins). Reference them as environment variables in pipeline steps — they

### are injected at runtime and never appear in logs. For AWS credentials use IAM roles with

### OIDC — no static keys at all.

### Q5. What is an artifact in CI/CD?

### Answer: The output of a build stage that gets stored and passed to subsequent stages. For

### containerised applications the artifact is the Docker image tagged with the git commit SHA.

### The artifact is built once and deployed to multiple environments — dev, staging, production

### — ensuring the same exact code runs everywhere. Immutable — once built it is never

### modified.

### Q6. What happens when a CI/CD pipeline fails?

### Answer: The pipeline stops immediately at the failing step — fail fast principle. The

### developer receives a notification (email, Slack) with details about which step failed and

### why. Production is not affected — the failing code never reaches it. The developer fixes the

### issue and pushes again — the pipeline reruns from the beginning. This catches bugs in

### minutes rather than finding them in production.

## Homework — Before CI/CD Day 2

```
bash
```
```
# 1. Create a GitHub repository if you don't have one:
git init myapp-cicd
cd myapp-cicd
```
```
# 2. Create a simple Python Flask app:
cat > app.py << 'EOF'
from flask import Flask, jsonify
app = Flask(__name__)
```
```
@app.route('/health')
def health():
return jsonify({'status': 'healthy'})
```

## Your Progress

```
@app.route('/')
def hello():
return jsonify({'message': 'Hello from CI/CD!'})
EOF
```
```
# 3. Create a simple test:
cat > test_app.py << 'EOF'
import pytest
from app import app
```
```
def test_health():
client = app.test_client()
response = client.get('/health')
assert response.status_code == 200
assert b'healthy' in response.data
EOF
```
```
# 4. Create requirements.txt:
echo "flask==3.0.
pytest==7.4.
gunicorn==21.2.0" > requirements.txt
```
```
# 5. Create Dockerfile:
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt.
RUN pip install -r requirements.txt
COPY..
EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF
```
```
# 6. Push to GitHub:
git add.
git commit -m "Initial Flask app with tests"
git push origin main
```
```
# 7. Go to github.com and look at the Actions tab
# Nothing there yet — we add pipeline in Day 2
```
```
Linux ████████████████████ ✅ COMPLETE
Git ████████████████████ ✅ COMPLETE
Docker ████████████████████ ✅ COMPLETE
AWS ████████████████████ ✅ COMPLETE
```

## CI/CD Day 2 we build your first real GitHub Actions pipeline — automated testing, Docker

## build and push to Docker Hub on every commit. You will see your code go from push to

## deployed automatically for the first time. 💪

## Say "CI/CD Day 2 " whenever you are ready!

```
CI/CD ██░░░░░░░░░░░░░░░░░░ Day 1 of 5
K8s ░░░░░░░░░░░░░░░░░░░░ After CI/CD
```
# CI/CD Day 2 — GitHub Actions

## What We Build Today

## By the end of this day your repository will have a pipeline that:

## This is a real production-grade GitHub Actions pipeline.

## What You Will Learn Today

## GitHub Actions core concepts

## YAML syntax for workflows

## Jobs and steps

## Runners — where pipelines run

## Environment variables and secrets

## Actions Marketplace — reusing pre-built steps

## Matrix builds — test on multiple versions

## Caching — speed up builds

## Artifacts — save build outputs

## Real pipeline — test, build, push, deploy

## Pull Request checks

```
You push code to GitHub
↓ automatically in 5 minutes
Tests run
↓
Docker image built
↓
Image pushed to Docker Hub
↓
Slack/email notification sent
↓
Zero human intervention
```

## GitHub Actions Core Concepts

## Where Pipelines Run — Runners

## Workflow YAML Structure

```
Workflow → the entire pipeline file (.github/workflows/name.yml)
Event → what triggers the workflow (push, PR, schedule)
Job → a group of steps that run on one runner
Step → individual task (run command or use action)
Runner → virtual machine that runs the job
Action → reusable step from marketplace
Secret → encrypted variable injected at runtime
Artifact → file saved from pipeline for download or later use
```
```
GitHub-hosted runners (free):
ubuntu-latest → Ubuntu 22.
windows-latest → Windows Server 2022
macos-latest → macOS 13
```
```
Specs: 2-core CPU, 7 GB RAM, 14 GB SSD
Free tier: 2000 minutes/month for private repos
Public repos: unlimited free minutes
```
```
Self-hosted runners:
Your own EC2, VM or laptop
Install GitHub Actions runner agent
Useful for: private networks, specific hardware, cost at scale
```
```
yaml
```
```
name: My Pipeline # displayed in GitHub Actions tab
```
```
on: # what triggers this workflow
push:
branches: [main, develop]
pull_request:
branches: [main]
```
```
env: # global environment variables
APP_NAME: myapp
PYTHON_VERSION: "3.11"
```
```
jobs: # one or more jobs
my-job: # job name (you choose)
```

## Step 1 — Create Your First Workflow

### In your repository from yesterday's homework:

```
runs-on: ubuntu-latest # which runner to use
steps: # steps run in sequence
```
- name: Step name
    uses: actions/checkout@v4 # use a marketplace action
- name: Another step
    run: echo "Hello World" # run shell command

```
bash
```
```
mkdir -p .github/workflows
vim .github/workflows/ci.yml
```
```
yaml
```
```
name: CI Pipeline
```
```
on:
push:
branches: [main, develop]
pull_request:
branches: [main]
```
```
jobs:
test:
name: Run Tests
runs-on: ubuntu-latest
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v
- name: Set up Python
    uses: actions/setup-python@v
    with:
       python-version: "3.11"
- name: Install dependencies
    run: |
python -m pip install --upgrade pip
pip install -r requirements.txt
- name: Run tests
run: pytest test_app.py - v


### Push it:

### Go to GitHub → your repo → Actions tab. Watch your pipeline run!

## Understanding the Checkout Action

### uses: actions/checkout@v4 is a marketplace action — pre-built reusable step. This one

### clones your repository into the runner. Without this — the runner has no code to work with.

### Format: owner/repo@version

### actions/ = official GitHub actions

### checkout = action name

### @v4 = version — always pin to a version, never @latest

## Step 2 — Add Caching for Speed

### Without caching — pip install downloads packages fresh every run. With caching —

### packages are cached between runs. Saves 1 - 2 minutes per pipeline.

- name: Show Python version
    run: python - -version

```
bash
```
```
git add .github/
git commit -m "Add CI pipeline"
git push origin main
```
```
yaml
```
- name: Checkout code
    uses: actions/checkout@v

```
yaml
```
```
name: CI Pipeline
```
```
on:
push:
branches: [main, develop]
pull_request:
branches: [main]
```
```
jobs:
test:
name: Run Tests
```

## Cache Key Explained

```
runs-on: ubuntu-latest
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v
- name: Set up Python
    uses: actions/setup-python@v
    with:
       python-version: "3.11"
- name: Cache pip packages
    uses: actions/cache@v
    with:
       path: ~/.cache/pip
       key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
       restore-keys: |
${{ runner.os }}-pip-
- name: Install dependencies
run: |
python -m pip install --upgrade pip
pip install -r requirements.txt
- name: Lint with flake 8
run: |
pip install flake 8
flake 8 app.py --max-line-length= 100
- name: Run tests with coverage
run: |
pip install pytest-cov
pytest test_app.py -v --cov=app --cov-report=term-missing
- name: Upload coverage report
uses: actions/upload-artifact@v
with:
name: coverage-report
path: .coverage
retention-days: 7

```
yaml
```
```
key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
```

**Part Meaning**

#### runner.os Linux/Windows/macOS — different cache per OS

#### pip- Prefix — distinguish from other caches

#### hashFiles('requirements.txt') Hash of requirements file — cache invalidates when deps change

### When requirements.txt changes — hash changes — new cache created. When it stays the

### same — existing cache restored instantly.

## Step 3 — Environment Variables and Secrets

## Adding Secrets to GitHub

```
yaml
```
```
name: CI Pipeline
```
```
on:
push:
branches: [main]
```
```
env:
APP_ENV: production # available to all jobs
```
```
jobs:
test:
runs-on: ubuntu-latest
env:
DB_HOST: localhost # available to this job only
steps:
```
- name: Checkout
    uses: actions/checkout@v
- name: Show environment
    run: |
echo "App env: $APP_ENV"
echo "DB host: $DB_HOST"
- name: Use secret
run: echo "Token length: ${#MY_SECRET}"
env:
MY_SECRET: ${{ secrets.MY_SECRET }} # per-step secret


### To create Docker Hub access token:

## Step 4 — Build and Push Docker Image

### Now add a second job that builds and pushes Docker image:

```
GitHub Repository
→ Settings
→ Secrets and variables
→ Actions
→ New repository secret
```
```
Add:
DOCKERHUB_USERNAME → your Docker Hub username
DOCKERHUB_TOKEN → Docker Hub access token (not password)
```
```
hub.docker.com → Account Settings → Security
→ New Access Token → name it "github-actions"
→ Copy the token — save it now, shown only once
```
```
yaml
```
```
name: CI/CD Pipeline
```
```
on:
push:
branches: [main]
pull_request:
branches: [main]
```
```
env:
IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
```
```
jobs:
# ── Job 1 — Test ──────────────────────────────────
test:
name: Test
runs-on: ubuntu-latest
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v
- name: Set up Python
    uses: actions/setup-python@v
    with:
       python-version: "3.11"


- name: Cache pip
    uses: actions/cache@v3
    with:
       path: ~/.cache/pip
       key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
- name: Install dependencies
    run: pip install - r requirements.txt
- name: Run tests
    run: pytest test_app.py - v

```
# ── Job 2 — Build and Push ────────────────────────
build:
name: Build and Push
runs-on: ubuntu-latest
needs: test # only runs if test job passes
if: github.ref == 'refs/heads/main' # only on main branch
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v4
- name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3
- name: Login to Docker Hub
    uses: docker/login-action@v3
    with:
       username: ${{ secrets.DOCKERHUB_USERNAME }}
       password: ${{ secrets.DOCKERHUB_TOKEN }}
- name: Extract metadata for Docker
    id: meta
    uses: docker/metadata-action@v5
    with:
       images: ${{ env.IMAGE_NAME }}
       tags: |
type=sha,prefix=sha-
type=ref,event=branch
type=semver,pattern={{version}}
latest
- name: Build and push Docker image
uses: docker/build-push-action@v5
with:
context:.
push: true


## needs — Job Dependencies

### Jobs run in parallel by default. needs makes them sequential and conditional.

## if — Conditional Execution

```
tags: ${{ steps.meta.outputs.tags }}
labels: ${{ steps.meta.outputs.labels }}
cache-from: type=gha
cache-to: type=gha,mode=max
```
```
yaml
```
```
jobs:
test:
runs-on: ubuntu-latest
steps: [...]
```
```
build:
needs: test # build only runs if test passes
runs-on: ubuntu-latest
```
```
deploy:
needs: [test, build] # deploy only if BOTH pass
runs-on: ubuntu-latest
```
```
yaml
```
```
# Only run on main branch:
if: github.ref == 'refs/heads/main'
```
```
# Only run on pull requests:
if: github.event_name == 'pull_request'
```
```
# Only run if previous step failed:
if: failure()
```
```
# Only run if previous step succeeded:
if: success()
```
```
# Always run (even if previous failed):
if: always()
```
```
# Run on specific tag:
if: startsWith(github.ref, 'refs/tags/v')
```

## Step 5 — Matrix Builds

### Test on multiple Python versions simultaneously:

### This creates 8 parallel jobs — all 4 Python versions on both Ubuntu and Windows. All must

### pass for the pipeline to succeed.

## Step 6 — Complete Production Pipeline

### Here is the full production-ready pipeline:

```
yaml
```
```
jobs:
test:
name: Test Python ${{ matrix.python-version }}
runs-on: ubuntu-latest
```
```
strategy:
matrix:
python-version: ["3.9", "3.10", "3.11", "3.12"]
os: [ubuntu-latest, windows-latest]
```
```
steps:
```
- uses: actions/checkout@v4
- name: Set up Python ${{ matrix.python-version }}
    uses: actions/setup-python@v5
    with:
       python-version: ${{ matrix.python-version }}
- name: Install and test
    run: |
pip install -r requirements.txt
pytest test_app.py -v

```
yaml
```
```
name: Production CI/CD
```
```
on:
push:
branches: [main, develop]
tags:
```
- 'v*'
pull_request:
branches: [main]


env:
REGISTRY: docker.io
IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
PYTHON_VERSION: "3.11"

jobs:
# ════════════════════════════════════════
# JOB 1 — LINT AND TEST
# ════════════════════════════════════════
test:
name: Lint and Test
runs-on: ubuntu-latest

```
steps:
```
- name: Checkout code
    uses: actions/checkout@v4
- name: Set up Python
    uses: actions/setup-python@v5
    with:
       python-version: ${{ env.PYTHON_VERSION }}
- name: Cache pip packages
    uses: actions/cache@v3
    with:
       path: ~/.cache/pip
       key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
       restore-keys: ${{ runner.os }}-pip-
- name: Install dependencies
    run: |
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install flake 8 pytest-cov
- name: Lint with flake 8
run: flake 8. - -max-line-length= 100 - -exclude=.git,__pycache__
- name: Run tests with coverage
run: |
pytest test_app.py -v \
--cov=app \
--cov-report=xml \
--cov-report=term-missing
- name: Upload test results
uses: actions/upload-artifact@v3
if: always()
with:


name: test-results
path: |
coverage.xml
pytest-report.xml
retention-days: 30

##### # ════════════════════════════════════════

##### # JOB 2 — SECURITY SCAN

##### # ════════════════════════════════════════

```
security:
name: Security Scan
runs-on: ubuntu-latest
needs: test
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v4
- name: Run Trivy vulnerability scan on filesystem
    uses: aquasecurity/trivy-action@master
    with:
       scan-type: 'fs'
       scan-ref: '.'
       format: 'table'
       exit-code: '0' # don't fail — just report
       severity: 'CRITICAL,HIGH'

```
# ════════════════════════════════════════
# JOB 3 — BUILD AND PUSH
# ════════════════════════════════════════
build:
name: Build and Push
runs-on: ubuntu-latest
needs: [test, security]
if: github.event_name != 'pull_request'
```
```
outputs:
image-tag: ${{ steps.meta.outputs.version }}
image-digest: ${{ steps.build.outputs.digest }}
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v4
- name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3
- name: Login to Docker Hub
    uses: docker/login-action@v3


```
with:
username: ${{ secrets.DOCKERHUB_USERNAME }}
password: ${{ secrets.DOCKERHUB_TOKEN }}
```
- name: Extract Docker metadata
    id: meta
    uses: docker/metadata-action@v5
    with:
       images: ${{ env.IMAGE_NAME }}
       tags: |
type=sha,prefix=sha-,format=short
type=ref,event=branch
type=semver,pattern={{version}}
type=semver,pattern={{major}}.{{minor}}
type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
- name: Build and push image
id: build
uses: docker/build-push-action@v5
with:
context:.
push: true
tags: ${{ steps.meta.outputs.tags }}
labels: ${{ steps.meta.outputs.labels }}
cache-from: type=gha
cache-to: type=gha,mode=max
build-args: |
BUILD_DATE=${{ github.event.head_commit.timestamp }}
VCS_REF=${{ github.sha }}
- name: Output image info
run: |
echo "Image tags: ${{ steps.meta.outputs.tags }}"
echo "Image digest: ${{ steps.build.outputs.digest }}"

```
# ════════════════════════════════════════
# JOB 4 — DEPLOY TO STAGING
# ════════════════════════════════════════
deploy-staging:
name: Deploy to Staging
runs-on: ubuntu-latest
needs: build
if: github.ref == 'refs/heads/main'
environment:
name: staging
url: https://staging.myapp.com
```
```
steps:
```
- name: Checkout code


```
uses: actions/checkout@v4
```
- name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ap-south- 1
- name: Deploy to staging EC 2
    run: |
IMAGE="${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"

ssh - i ${{ secrets.SSH_KEY }} \

- o StrictHostKeyChecking=no \
ubuntu@${{ secrets.STAGING_IP }} \
"docker pull $IMAGE && \
docker stop myapp || true && \
docker rm myapp || true && \
docker run - d \
- -name myapp \
- -restart unless-stopped \
- p 80 : 5000 \
- e ENVIRONMENT=staging \
$IMAGE"
- name: Health check staging
run: |
sleep 15
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
[http://${{](http://${{) secrets.STAGING_IP }}/health)
echo "Health check status: $HTTP_CODE"
[ "$HTTP_CODE" = "200" ] || exit 1

##### # ════════════════════════════════════════

##### # JOB 5 — DEPLOY TO PRODUCTION

##### # ════════════════════════════════════════

```
deploy-production:
name: Deploy to Production
runs-on: ubuntu-latest
needs: deploy-staging
if: startsWith(github.ref, 'refs/tags/v')
environment:
name: production
url: https://myapp.com
```
```
steps:
```
- name: Checkout code
    uses: actions/checkout@v4


- name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ap-south- 1
- name: Deploy to production (Rolling update)
    run: |
IMAGE="${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"

# Update Auto Scaling Group launch template
aws ec 2 create-launch-template-version \

- -launch-template-name myapp-lt \
- -source-version '$Latest' \
- -launch-template-data "{\"ImageId\": \"ami-prod\"}"

# Start instance refresh (zero downtime)
aws autoscaling start-instance-refresh \

- -auto-scaling-group-name myapp-asg \
- -preferences '{
    "MinHealthyPercentage": 80 ,
    "InstanceWarmup": 300
}'

echo "Production deployment initiated"
echo "Monitor at: AWS Console → EC 2 → Auto Scaling Groups"

- name: Create GitHub Release
    uses: actions/create-release@v1
    env:
       GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    with:
       tag_name: ${{ github.ref }}
       release_name: Release ${{ github.ref }}
       body: |
## Changes in this release
${{ github.event.head_commit.message }}

## Docker Image
`${{ env.IMAGE_NAME }}:sha-${{ github.sha }}`
draft: false
prerelease: false

```
# ════════════════════════════════════════
# JOB 6 — NOTIFY
# ════════════════════════════════════════
notify:
```

## GitHub Actions Contexts — Variables Available

```
name: Notify Team
runs-on: ubuntu-latest
needs: [test, build, deploy-staging]
if: always()
```
```
steps:
```
- name: Notify on success
    if: success()
    run: |
curl - X POST ${{ secrets.SLACK_WEBHOOK }} \
- H 'Content-type: application/json' \
- d '{
"text": "✅ *${{ github.repository }}* pipeline passed!\nBranch: `${{
}'
- name: Notify on failure
if: failure()
run: |
curl - X POST ${{ secrets.SLACK_WEBHOOK }} \
- H 'Content-type: application/json' \
- d '{
"text": "❌ *${{ github.repository }}* pipeline FAILED!\nBranch: `${{
}'

```
yaml
# GitHub context:
${{ github.sha }} # full commit SHA
${{ github.ref }} # refs/heads/main or refs/tags/v1.0
${{ github.ref_name }} # main or v1.0 (just the name)
${{ github.actor }} # who triggered the pipeline
${{ github.repository }} # owner/repo-name
${{ github.event_name }} # push, pull_request, schedule
${{ github.run_id }} # unique ID for this run
${{ github.run_number }} # sequential build number
${{ github.server_url }} # https://github.com
```
```
# Runner context:
${{ runner.os }} # Linux, Windows, macOS
${{ runner.temp }} # temp directory path
```
```
# Secrets:
${{ secrets.MY_SECRET }} # encrypted secret value
```

## GitHub Environments — Deployment Protection

### GitHub Environments add protection rules to deployments:

### When pipeline reaches this job — it pauses and sends email to required reviewers. They

### approve or reject in GitHub UI.

## Pull Request Checks

### When someone opens a PR — the pipeline runs automatically:

```
# Env:
${{ env.MY_VAR }} # environment variable
```
```
Repository → Settings → Environments → New environment
```
```
Production environment settings:
Required reviewers: varun, team-lead ← manual approval needed
Wait timer: 10 minutes ← cooling off period
Branch restrictions: main only ← only from main branch
```
```
yaml
```
```
deploy-production:
environment:
name: production # references the environment
url: https://myapp.com # shown in GitHub UI
```
```
PR opened: "Add user authentication"
↓
Pipeline starts automatically
↓
Tests run: 45 seconds
↓
Security scan: 30 seconds
↓
Status shown on PR:
✅ test (ubuntu-latest)
✅ security-scan
✗ lint — 2 errors found
```
```
PR cannot be merged until all checks pass
This enforces code quality across the team
```

## Useful Actions from Marketplace

```
yaml
```
```
# Checkout code:
```
- uses: actions/checkout@v4

```
# Set up Python:
```
- uses: actions/setup-python@v5
    with:
       python-version: "3.11"

```
# Set up Node.js:
```
- uses: actions/setup-node@v4
    with:
       node-version: "18"

```
# Cache dependencies:
```
- uses: actions/cache@v3

```
# Upload artifact:
```
- uses: actions/upload-artifact@v3

```
# Download artifact:
```
- uses: actions/download-artifact@v3

```
# Docker login:
```
- uses: docker/login-action@v3

```
# Docker build and push:
```
- uses: docker/build-push-action@v5

```
# Configure AWS credentials:
```
- uses: aws-actions/configure-aws-credentials@v4

```
# Login to ECR:
```
- uses: aws-actions/amazon-ecr-login@v2

```
# Trivy security scan:
```
- uses: aquasecurity/trivy-action@master

```
# Create GitHub Release:
```
- uses: actions/create-release@v1

```
# Comment on PR:
```
- uses: actions/github-script@v7


## Reusable Workflows — DRY Principle

## GitHub Actions for AWS ECR

```
yaml
```
```
# .github/workflows/reusable-test.yml
name: Reusable Test
```
```
on:
workflow_call: # can be called by other workflows
inputs:
python-version:
required: true
type: string
secrets:
CODECOV_TOKEN:
required: true
```
```
jobs:
test:
runs-on: ubuntu-latest
steps:
```
- uses: actions/checkout@v4
- uses: actions/setup-python@v5
    with:
       python-version: ${{ inputs.python-version }}
- run: pytest

```
# .github/workflows/main.yml
jobs:
run-tests:
uses: ./.github/workflows/reusable-test.yml
with:
python-version: "3.11"
secrets:
CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```
```
yaml
```
- name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ap-south- 1
- name: Login to Amazon ECR


## Full Summary — CI/CD Day 2

**Concept Key point**

#### Workflow YAML file in .github/workflows/

#### Event on: push, pull_request, schedule

Job Group of steps — runs on one runner

#### Step Single task — run command or uses action

#### Runner VM that runs the job — ubuntu-latest

#### needs Job dependency — sequential execution

#### if Conditional — only run in certain conditions

#### Secrets Encrypted — reference with ${{ secrets.NAME }}

Cache Speed up builds — cache pip, npm etc

Artifact Save files between jobs or for download

Matrix Test on multiple versions in parallel

Environment staging, production — with approval gates

#### Contexts github.sha, github.ref, runner.os

## Interview Questions — CI/CD Day 2

### Q1. What is GitHub Actions and how does it work?

### Answer: GitHub Actions is a CI/CD platform built directly into GitHub. You define

### workflows as YAML files in .github/workflows/. When events occur — push, pull request,

### schedule — GitHub automatically runs the workflows on managed virtual machines called

```
id: login-ecr
uses: aws-actions/amazon-ecr-login@v2
```
- name: Build and push to ECR
    env:
       ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
       IMAGE_TAG: ${{ github.sha }}
    run: |
docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG.
docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
echo "image=$ECR_REGISTRY/myapp:$IMAGE_TAG" >> $GITHUB_OUTPUT


### runners. Each workflow has jobs, each job has steps that either run shell commands or use

### pre-built marketplace actions.

### Q2. What is the difference between a job and a step in GitHub Actions?

### Answer: A job is a group of steps that run on the same runner virtual machine. Steps within

### a job run sequentially and share the same filesystem. Multiple jobs run in parallel by default

### — unless you use needs to create dependencies. Steps are individual tasks within a job —

### either run (shell command) or uses (marketplace action).

### Q3. How do you pass data between jobs in GitHub Actions?

### Answer: Three ways — job outputs (for small values), artifacts (for files) and environment

### files. Job outputs use echo "key=value" >> $GITHUB_OUTPUT in one job and ${{ needs.job-

### name.outputs.key }} in the next. Artifacts use upload-artifact and download-artifact

### actions to pass files between jobs. The artifact is temporarily stored on GitHub servers.

### Q4. How do you secure secrets in GitHub Actions?

### Answer: Store secrets in repository Settings → Secrets and variables → Actions. Reference

### in workflow with ${{ secrets.SECRET_NAME }}. Secrets are encrypted at rest, only available

### to workflows in the repository, masked in logs — never printed even if you try to echo them.

### For AWS use OIDC (OpenID Connect) with aws-actions/configure-aws-credentials —

### eliminates need for static access keys.

### Q5. What is a matrix build and when do you use it?

### Answer: Matrix builds run the same job with different combinations of variables

### simultaneously. Used to test on multiple Python/Node/Java versions at once, multiple

### operating systems, multiple dependency versions. All matrix combinations run in parallel

### — faster than sequential. If any combination fails the job fails.

### Q6. How do you implement manual approval in GitHub Actions?

### Answer: Create a GitHub Environment (Settings → Environments) with required reviewers

### configured. Reference the environment in a deploy job with environment: name:

### production. When the pipeline reaches that job it pauses and sends email to reviewers.

### They approve or reject in the GitHub UI. The job only proceeds on approval.

## Homework — Before CI/CD Day 3

```
bash
```
```
# 1. Add .github/workflows/ci.yml to your repo with:
# - checkout, setup-python, install deps, run tests
```
```
# 2. Push and watch it run in GitHub Actions tab
```
```
# 3. Add Docker Hub secrets to your repository:
# DOCKERHUB_USERNAME and DOCKERHUB_TOKEN
```
```
# 4. Add the build job that:
# - Only runs on main branch
# - Only runs if test job passes
```

## Your Progress

## CI/CD Day 3 we learn Jenkins — the industry-standard self-hosted CI/CD tool used in most

## enterprise environments. You will install Jenkins on EC2, create a Jenkinsfile and build the

## same pipeline we built today but in Jenkins. 💪

## Say "CI/CD Day 3 " whenever you are ready!

```
# - Builds and pushes Docker image
```
```
# 5. Break a test intentionally — push — watch pipeline fail
```
```
# 6. Fix the test — push — watch pipeline pass
```
```
# 7. Check Docker Hub — verify your image is there
```
```
# 8. Try this workflow trigger — add to your workflow:
on:
workflow_dispatch: # adds manual trigger button
inputs:
environment:
description: 'Deploy to which environment?'
required: true
default: 'staging'
type: choice
options:
```
- staging
- production

```
Linux ████████████████████ ✅ COMPLETE
Git ████████████████████ ✅ COMPLETE
Docker ████████████████████ ✅ COMPLETE
AWS ████████████████████ ✅ COMPLETE
CI/CD ████░░░░░░░░░░░░░░░░ Day 2 of 5
K8s ░░░░░░░░░░░░░░░░░░░░ After CI/CD
```
# CI/CD Day 3 — Jenkins

## Why Jenkins After GitHub Actions?

## You might wonder — we already have GitHub Actions, why learn Jenkins?

```
Real job market breakdown:
```
```
Startups (< 50 engineers): GitHub Actions — 70 %, Jenkins — 30 %
Mid-size (50-500 engineers): Jenkins — 60 %, GitHub Actions — 40 %
```

