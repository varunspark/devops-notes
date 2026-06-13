## CI/CD Day 1 — Introduction & Core Concepts

---

### Why CI/CD is the Heart of DevOps

You have learned Linux, Git, Docker, and AWS separately. CI/CD is the glue that connects all of them into one automated pipeline.

**Without CI/CD:**

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

**With CI/CD:**

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

> This is what DevOps companies pay 10-12 LPA for.

---

### What You Will Learn Today

- What CI/CD is and why it exists
- CI vs Continuous Delivery vs Continuous Deployment — the three terms
- Pipeline stages explained
- Git branching and CI/CD connection
- GitHub Actions — introduction
- Jenkins — introduction
- Choosing between tools
- Key CI/CD concepts — artifacts, environments, rollback
- Real DevOps CI/CD workflow

---

### What is CI/CD?

CI/CD stands for:

```
CI = Continuous Integration
CD = Continuous Delivery (or)
CD = Continuous Deployment
```

These are **practices** — a way of working — not just tools.

---

### Continuous Integration (CI)

The practice of merging code changes frequently and automatically verifying each change.

```
Developer A pushes code → CI runs automatically:
✓ Code compiles
✓ Unit tests pass
✓ Integration tests pass
✓ Code quality checks pass
✓ Security scan passes
→ "This code is safe to merge"

Developer B pushes code → same checks run automatically
Developer C pushes code → same checks run automatically
```

- Everyone's code is always verified
- No "it works on my machine" — same environment every time
- Problems caught immediately — not weeks later

---

### Continuous Delivery vs Continuous Deployment

These two are different, and interviewers test this:

**Continuous Delivery:**

```
Code → CI passes → Artifact ready → MANUAL approval → Deploy
                                          ↑
                              Human decides when to deploy
                              Code is READY but not auto-deployed
                              Used when: regulated industries,
                                         need business sign-off
```

**Continuous Deployment:**

```
Code → CI passes → Artifact ready → AUTO deploy → Production
                                          ↑
                              No human intervention
                              Every passing commit goes to prod
                              Used when: fast-moving products,
                                         high test confidence
```

> Most companies use **Continuous Delivery** — automated pipeline up to a staging environment, with manual approval for production.

---

### The Four Stages of a CI/CD Pipeline

```
Stage 1 — SOURCE
Trigger: developer pushes to Git
Action: checkout code, prepare workspace

Stage 2 — BUILD
Action: compile code, run unit tests,
        build Docker image,
        scan image for vulnerabilities
Output: Docker image tagged with commit SHA

Stage 3 — TEST
Action: deploy to staging environment,
        run integration tests,
        run end-to-end tests,
        performance tests
Output: test report — pass or fail

Stage 4 — DEPLOY
Action: push image to registry (ECR),
        deploy to production,
        run health checks,
        notify team
Output: new version live in production
```

---

### Pipeline as Code

Modern CI/CD defines pipelines as code files stored in the repository itself:

```
GitHub Actions → .github/workflows/pipeline.yml
Jenkins         → Jenkinsfile
GitLab CI       → .gitlab-ci.yml
CircleCI        → .circleci/config.yml
```

**Benefits of pipeline as code:**

- Version controlled — see history of pipeline changes
- Reviewed in pull requests — pipeline changes need approval
- Reproducible — same pipeline runs every time
- Developer-owned — teams manage their own pipelines

---

### The Two Main Tools We Cover

#### GitHub Actions

```
What it is: CI/CD built directly into GitHub
Where it runs: GitHub's cloud infrastructure
Pricing: Free for public repos
         2000 minutes/month free for private repos
Config file: .github/workflows/name.yml
Trigger: push, pull request, schedule, manual
Best for: Projects hosted on GitHub
          Open source
          Teams already using GitHub
```

#### Jenkins

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

### GitHub Actions vs Jenkins

| Feature | GitHub Actions | Jenkins |
|---------|-----------------|---------|
| Hosting | GitHub cloud | Your server |
| Setup | Zero — instant | Install + configure |
| Maintenance | None — GitHub manages | You manage updates |
| Cost | Free tier generous | Free + infra cost |
| Plugins | Marketplace actions | 1800+ plugins |
| Customisation | Good | Excellent |
| Enterprise | Good | Industry standard |
| Learning curve | Low | Medium |

> For interviews — know both. Most companies use one or the other, or both.

---

### Key CI/CD Concepts

#### Artifact

```
Source code → BUILD → Artifact
```

The output of a build step — something that gets stored and passed to the next stage.

**Types of artifacts:**
- Docker image (pushed to ECR/Docker Hub)
- JAR/WAR file (Java applications)
- ZIP file (Lambda deployment package)
- tar.gz (compiled binary)
- npm package

#### Environment

A separate deployment target with its own infrastructure:

```
dev        → developer testing, any branch
staging    → pre-production, mirrors production, automated tests
production → real users, manual or auto deploy
```

Each environment:
- Has its own AWS account or VPC
- Has its own database (separate data)
- Has its own configuration (.env values)
- May have different instance sizes

#### Pipeline Trigger

What starts a pipeline:

```yaml
# Push to any branch:
on: push

# Push to main only:
on:
  push:
    branches: [main]

# Pull request:
on:
  pull_request:
    branches: [main]

# Schedule (like cron):
on:
  schedule:
    - cron: '0 2 * * *'   # every day at 2am

# Manual trigger:
on:
  workflow_dispatch:   # adds Run Workflow button in GitHub
```

#### Build Status Badge

A small image showing pipeline status — added to README:

```markdown
![CI](https://github.com/varun/myapp/actions/workflows/ci.yml/badge.svg)
```

> Shows green ✅ or red ❌ — visible to anyone viewing the repo.

#### Rollback

Reverting production to a previous working version:

```bash
# With Docker — instantly rollback to previous image:
docker pull registry/myapp:v1.2.2   # previous version
docker stop myapp-current
docker run -d --name myapp registry/myapp:v1.2.2
```

```bash
# With Auto Scaling instance refresh — rollback to previous AMI:
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name myapp-asg \
  --preferences '{"MinHealthyPercentage": 80}'
# After updating Launch Template back to v1.2.2
```

> Why tagging with git SHA matters: you always know exactly which code is running, and you can roll back to any previous commit instantly.

---

### CI/CD and Git Branching — How They Connect

Different branches trigger different pipeline stages:

```
feature/* branches:
→ Run unit tests only (fast feedback)
→ Do NOT deploy anywhere

develop branch:
→ Run all tests
→ Deploy to dev environment automatically

main branch:
→ Run all tests
→ Build production Docker image
→ Deploy to staging automatically
→ Manual approval gate
→ Deploy to production

Tags (v1.0.0, v1.1.0):
→ Full pipeline
→ Deploy to production automatically
→ Create GitHub Release
```

---

### The DevOps Feedback Loop

CI/CD creates a tight feedback loop:

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

> The faster the feedback, the cheaper the bug fix. Finding a bug in CI takes 5 minutes to fix. Finding it in production takes hours.

---

### What a Real Pipeline Does Step by Step

```
Trigger: developer pushes to main branch

Step 1 — Checkout:
  git clone the repository
  checkout the specific commit

Step 2 — Environment setup:
  install dependencies (npm install, pip install)
  set up test database
  load environment variables

Step 3 — Lint and code quality:
  run ESLint / flake8 / pylint
  check code style
  fail fast if quality below threshold

Step 4 — Unit tests:
  run pytest / jest / junit
  generate coverage report
  fail if coverage below 80%

Step 5 — Build:
  docker build -t myapp:$GIT_SHA .
  docker scan myapp:$GIT_SHA (security scan)

Step 6 — Integration tests:
  docker-compose up -d (app + dependencies)
  run integration test suite
  docker-compose down

Step 7 — Push to registry:
  docker tag myapp:$GIT_SHA registry/myapp:$GIT_SHA
  docker push registry/myapp:$GIT_SHA

Step 8 — Deploy to staging:
  update EC2/ECS/EKS with new image
  wait for health checks to pass

Step 9 — Smoke tests on staging:
  curl staging.myapp.com/health
  run critical path tests

Step 10 — Notify:
  Slack message: "v1.2.3 deployed to staging"
  Link to test report
  Manual approval button for production
```

---

### Pipeline Failure — What Happens

```
Any step fails → Pipeline stops immediately
              → Developer gets email/Slack notification
              → Production is NOT affected
              → Fix the code → push again → pipeline reruns
```

> This is "fail fast" — stop at the first problem. Better to catch it in CI than in production.

---

### Secrets in CI/CD

Never put passwords or API keys in pipeline files:

```yaml
# WRONG — secret visible in repository:
- run: docker push registry/myapp
  env:
    PASSWORD: supersecret

# CORRECT — stored in GitHub Secrets or Jenkins Credentials:
- run: docker push registry/myapp
  env:
    PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

> GitHub Secrets are stored encrypted, only injected at pipeline runtime, and never visible in logs.

---

### Pipeline Best Practices

1. **Fast feedback** — unit tests should run in < 5 minutes
2. **Fail fast** — stop at first failure
3. **Idempotent** — running the pipeline twice gives the same result
4. **Reproducible** — same code always produces same artifact
5. **Secure** — secrets in vault, not in code
6. **Visible** — team can see pipeline status easily
7. **Tested** — the pipeline itself should be tested
8. **Tagged artifacts** — every build tagged with git SHA
9. **Immutable artifacts** — once built, never modified
10. **One pipeline per environment** — not one pipeline doing everything

---

### Real World CI/CD at 10-12 LPA Companies

```
Small startup (10-50 engineers):
→ GitHub Actions → ECR → ECS or EC2
→ Simple — push to main → auto deploy staging
→ Manual deploy to production

Medium company (50-500 engineers):
→ Jenkins → ECR → EKS
→ Feature branches → dev → staging → production
→ Multiple approval gates
→ Automated rollback on health check failure

Enterprise (500+ engineers):
→ Jenkins or TeamCity → Artifactory → Multiple regions
→ Strict change control
→ Blue/green deployments
→ Canary releases (5% → 25% → 100% traffic)
```

---

### Full Summary — CI/CD Day 1

| Concept | Key point |
|---------|-----------|
| CI | Automatically test every code change |
| Continuous Delivery | Auto deploy to staging — manual to production |
| Continuous Deployment | Auto deploy to production — no manual step |
| Pipeline stages | Source → Build → Test → Deploy |
| Pipeline as code | YAML/Groovy file in repository |
| GitHub Actions | Cloud CI/CD — built into GitHub |
| Jenkins | Self-hosted — more control |
| Artifact | Build output — Docker image, JAR, ZIP |
| Environment | dev, staging, production — separate infrastructure |
| Fail fast | Stop at first failure — protect production |
| Secrets | Never in code — use GitHub Secrets or Jenkins Credentials |
| Rollback | Previous Docker image tag — instant recovery |

---

### Interview Questions — CI/CD Day 1

**Q1. What is the difference between Continuous Delivery and Continuous Deployment?**
Continuous Delivery means every code change is automatically built, tested, and made ready to deploy — but a human approves when it goes to production. Continuous Deployment goes one step further — every passing change is automatically deployed to production with no manual step. Most companies use Continuous Delivery — an automated pipeline up to staging with manual production approval.

**Q2. What is a CI/CD pipeline and what stages does it have?**
A CI/CD pipeline is an automated sequence of steps that takes code from a developer's commit to running in production. Typical stages: Source (code checkout), Build (compile, unit tests, Docker build), Test (integration tests, security scan, deploy to staging), Deploy (push to registry, deploy to production, health check). Each stage must pass before the next begins.

**Q3. What is pipeline as code and why is it important?**
Defining the CI/CD pipeline in a file stored in the repository — like `.github/workflows/pipeline.yml` for GitHub Actions or `Jenkinsfile` for Jenkins. It's important because the pipeline is version controlled, reviewed in pull requests, reproducible, and developer-owned. Changes to the pipeline go through the same review process as application code.

**Q4. How do you handle secrets in a CI/CD pipeline?**
Never put secrets in pipeline files — they get committed to Git and are visible to everyone. Store secrets in GitHub Secrets (for GitHub Actions) or Jenkins Credentials Manager (for Jenkins). Reference them as environment variables in pipeline steps — they are injected at runtime and never appear in logs. For AWS credentials, use IAM roles with OIDC — no static keys at all.

**Q5. What is an artifact in CI/CD?**
The output of a build stage that gets stored and passed to subsequent stages. For containerised applications, the artifact is the Docker image tagged with the git commit SHA. The artifact is built once and deployed to multiple environments — dev, staging, production — ensuring the same exact code runs everywhere. Immutable — once built it is never modified.

**Q6. What happens when a CI/CD pipeline fails?**
The pipeline stops immediately at the failing step — fail fast principle. The developer receives a notification (email, Slack) with details about which step failed and why. Production is not affected — the failing code never reaches it. The developer fixes the issue and pushes again — the pipeline reruns from the beginning. This catches bugs in minutes rather than finding them in production.

---

### Homework — Before CI/CD Day 2

```bash
# 1. Create a GitHub repository if you don't have one:
git init myapp-cicd
cd myapp-cicd

# 2. Create a simple Python Flask app:
cat > app.py << 'EOF'
from flask import Flask, jsonify
app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/')
def hello():
    return jsonify({'message': 'Hello from CI/CD!'})
EOF

# 3. Create a simple test:
cat > test_app.py << 'EOF'
import pytest
from app import app

def test_health():
    client = app.test_client()
    response = client.get('/health')
    assert response.status_code == 200
    assert b'healthy' in response.data
EOF

# 4. Create requirements.txt:
echo "flask==3.0.0
pytest==7.4.0
gunicorn==21.2.0" > requirements.txt

# 5. Create Dockerfile:
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF

# 6. Push to GitHub:
git add .
git commit -m "Initial Flask app with tests"
git push origin main

# 7. Go to github.com and look at the Actions tab
# Nothing there yet — we add a pipeline in Day 2
```

> CI/CD Day 2 builds your first real GitHub Actions pipeline — automated testing, Docker build, and push to Docker Hub on every commit. You will see your code go from push to deployed automatically for the first time.
