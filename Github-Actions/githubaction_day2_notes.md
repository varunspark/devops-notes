## CI/CD Day 2 — GitHub Actions
 
---
 
### What We Build Today
 
By the end of this day your repository will have a pipeline that:
 
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
 
> This is a real production-grade GitHub Actions pipeline.
 
---
 
### What You Will Learn Today
 
- GitHub Actions core concepts
- YAML syntax for workflows
- Jobs and steps
- Runners — where pipelines run
- Environment variables and secrets
- Actions Marketplace — reusing pre-built steps
- Matrix builds — test on multiple versions
- Caching — speed up builds
- Artifacts — save build outputs
- Real pipeline — test, build, push, deploy
- Pull Request checks
---
 
### GitHub Actions Core Concepts
 
```
Workflow → the entire pipeline file (.github/workflows/name.yml)
Event    → what triggers the workflow (push, PR, schedule)
Job      → a group of steps that run on one runner
Step     → individual task (run command or use action)
Runner   → virtual machine that runs the job
Action   → reusable step from marketplace
Secret   → encrypted variable injected at runtime
Artifact → file saved from pipeline for download or later use
```
 
---
 
### Where Pipelines Run — Runners
 
**GitHub-hosted runners (free):**
 
```
ubuntu-latest  → Ubuntu 22.04
windows-latest → Windows Server 2022
macos-latest   → macOS 13
 
Specs: 2-core CPU, 7 GB RAM, 14 GB SSD
Free tier: 2000 minutes/month for private repos
Public repos: unlimited free minutes
```
 
**Self-hosted runners:**
 
```
Your own EC2, VM, or laptop
Install GitHub Actions runner agent
Useful for: private networks, specific hardware, cost at scale
```
 
---
 
### Workflow YAML Structure
 
```yaml
name: My Pipeline   # displayed in GitHub Actions tab
 
on:                  # what triggers this workflow
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
 
env:                 # global environment variables
  APP_NAME: myapp
  PYTHON_VERSION: "3.11"
 
jobs:                # one or more jobs
  my-job:            # job name (you choose)
    runs-on: ubuntu-latest   # which runner to use
    steps:                   # steps run in sequence
      - name: Step name
        uses: actions/checkout@v4   # use a marketplace action
      - name: Another step
        run: echo "Hello World"     # run shell command
```
 
---
 
### Step 1 — Create Your First Workflow
 
In your repository from yesterday's homework:
 
```bash
mkdir -p .github/workflows
vim .github/workflows/ci.yml
```
 
```yaml
name: CI Pipeline
 
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
 
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Show Python version
        run: python --version
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run tests
        run: pytest test_app.py -v
```
 
Push it:
 
```bash
git add .github/
git commit -m "Add CI pipeline"
git push origin main
```
 
> Go to GitHub → your repo → **Actions** tab. Watch your pipeline run!
 
### Understanding the Checkout Action
 
```yaml
- name: Checkout code
  uses: actions/checkout@v4
```
 
`actions/checkout@v4` is a marketplace action — a pre-built reusable step. This one clones your repository into the runner. Without this, the runner has no code to work with.
 
**Format:** `owner/repo@version`
- `actions/` = official GitHub actions
- `checkout` = action name
- `@v4` = version — always pin to a version, never `@latest`
---
 
### Step 2 — Add Caching for Speed
 
Without caching, `pip install` downloads packages fresh every run. With caching, packages are cached between runs — saves 1-2 minutes per pipeline.
 
```yaml
name: CI Pipeline
 
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
 
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Lint with flake8
        run: |
          pip install flake8
          flake8 app.py --max-line-length=100
      - name: Run tests with coverage
        run: |
          pip install pytest-cov
          pytest test_app.py -v --cov=app --cov-report=term-missing
      - name: Upload coverage report
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: .coverage
          retention-days: 7
```
 
### Cache Key Explained
 
```yaml
key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
```
 
| Part | Meaning |
|------|---------|
| `runner.os` | Linux/Windows/macOS — different cache per OS |
| `pip-` | Prefix — distinguish from other caches |
| `hashFiles('requirements.txt')` | Hash of requirements file — cache invalidates when deps change |
 
> When `requirements.txt` changes, the hash changes, so a new cache is created. When it stays the same, the existing cache is restored instantly.
 
---
 
### Step 3 — Environment Variables and Secrets
 
```yaml
name: CI Pipeline
 
on:
  push:
    branches: [main]
 
env:
  APP_ENV: production   # available to all jobs
 
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      DB_HOST: localhost   # available to this job only
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Show environment
        run: |
          echo "App env: $APP_ENV"
          echo "DB host: $DB_HOST"
      - name: Use secret
        run: echo "Token length: ${#MY_SECRET}"
        env:
          MY_SECRET: ${{ secrets.MY_SECRET }}   # per-step secret
```
 
### Adding Secrets to GitHub
 
```
GitHub Repository
  → Settings
  → Secrets and variables
  → Actions
  → New repository secret
 
Add:
DOCKERHUB_USERNAME → your Docker Hub username
DOCKERHUB_TOKEN    → Docker Hub access token (not password)
```
 
To create a Docker Hub access token:
 
```
hub.docker.com → Account Settings → Security
→ New Access Token → name it "github-actions"
→ Copy the token — save it now, shown only once
```
 
---
 
### Step 4 — Build and Push Docker Image
 
Now add a second job that builds and pushes the Docker image:
 
```yaml
name: CI/CD Pipeline
 
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
 
env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
 
jobs:
  # ── Job 1 — Test ──────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Cache pip
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest test_app.py -v
 
  # ── Job 2 — Build and Push ────────────────────────
  build:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: test                            # only runs if test job passes
    if: github.ref == 'refs/heads/main'    # only on main branch
 
    steps:
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
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
 
### `needs` — Job Dependencies
 
Jobs run in parallel by default. `needs` makes them sequential and conditional.
 
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps: [...]
 
  build:
    needs: test            # build only runs if test passes
    runs-on: ubuntu-latest
 
  deploy:
    needs: [test, build]   # deploy only if BOTH pass
    runs-on: ubuntu-latest
```
 
### `if` — Conditional Execution
 
```yaml
# Only run on main branch:
if: github.ref == 'refs/heads/main'
 
# Only run on pull requests:
if: github.event_name == 'pull_request'
 
# Only run if previous step failed:
if: failure()
 
# Only run if previous step succeeded:
if: success()
 
# Always run (even if previous failed):
if: always()
 
# Run on specific tag:
if: startsWith(github.ref, 'refs/tags/v')
```
 
---
 
### Step 5 — Matrix Builds
 
Test on multiple Python versions simultaneously:
 
```yaml
jobs:
  test:
    name: Test Python ${{ matrix.python-version }}
    runs-on: ${{ matrix.os }}
 
    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest]
 
    steps:
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
 
> This creates 8 parallel jobs — all 4 Python versions on both Ubuntu and Windows. All must pass for the pipeline to succeed.
 
---
 
### Step 6 — Complete Production Pipeline
 
Here is the full production-ready pipeline:
 
```yaml
name: Production CI/CD
 
on:
  push:
    branches: [main, develop]
    tags:
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
 
    steps:
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
          pip install flake8 pytest-cov
      - name: Lint with flake8
        run: flake8 . --max-line-length=100 --exclude=.git,__pycache__
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
 
  # ════════════════════════════════════════
  # JOB 2 — SECURITY SCAN
  # ════════════════════════════════════════
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Run Trivy vulnerability scan on filesystem
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'table'
          exit-code: '0'   # don't fail — just report
          severity: 'CRITICAL,HIGH'
 
  # ════════════════════════════════════════
  # JOB 3 — BUILD AND PUSH
  # ════════════════════════════════════════
  build:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name != 'pull_request'
 
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-digest: ${{ steps.build.outputs.digest }}
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
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
          context: .
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
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Deploy to staging EC2
        run: |
          IMAGE="${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"
          ssh -i ${{ secrets.SSH_KEY }} \
            -o StrictHostKeyChecking=no \
            ubuntu@${{ secrets.STAGING_IP }} \
            "docker pull $IMAGE && \
             docker stop myapp || true && \
             docker rm myapp || true && \
             docker run -d \
               --name myapp \
               --restart unless-stopped \
               -p 80:5000 \
               -e ENVIRONMENT=staging \
               $IMAGE"
      - name: Health check staging
        run: |
          sleep 15
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            http://${{ secrets.STAGING_IP }}/health)
          echo "Health check status: $HTTP_CODE"
          [ "$HTTP_CODE" = "200" ] || exit 1
 
  # ════════════════════════════════════════
  # JOB 5 — DEPLOY TO PRODUCTION
  # ════════════════════════════════════════
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://myapp.com
 
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Deploy to production (Rolling update)
        run: |
          IMAGE="${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"
 
          # Update Auto Scaling Group launch template
          aws ec2 create-launch-template-version \
            --launch-template-name myapp-lt \
            --source-version '$Latest' \
            --launch-template-data "{\"ImageId\": \"ami-prod\"}"
 
          # Start instance refresh (zero downtime)
          aws autoscaling start-instance-refresh \
            --auto-scaling-group-name myapp-asg \
            --preferences '{
              "MinHealthyPercentage": 80,
              "InstanceWarmup": 300
            }'
 
          echo "Production deployment initiated"
          echo "Monitor at: AWS Console → EC2 → Auto Scaling Groups"
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
 
  # ════════════════════════════════════════
  # JOB 6 — NOTIFY
  # ════════════════════════════════════════
  notify:
    name: Notify Team
    runs-on: ubuntu-latest
    needs: [test, build, deploy-staging]
    if: always()
 
    steps:
      - name: Notify on success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{
              "text": "✅ *${{ github.repository }}* pipeline passed!\nBranch: `${{ github.ref_name }}`"
            }'
      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{
              "text": "❌ *${{ github.repository }}* pipeline FAILED!\nBranch: `${{ github.ref_name }}`"
            }'
```
 
---
 
### GitHub Actions Contexts — Variables Available
 
```yaml
# GitHub context:
${{ github.sha }}          # full commit SHA
${{ github.ref }}          # refs/heads/main or refs/tags/v1.0
${{ github.ref_name }}     # main or v1.0 (just the name)
${{ github.actor }}        # who triggered the pipeline
${{ github.repository }}   # owner/repo-name
${{ github.event_name }}   # push, pull_request, schedule
${{ github.run_id }}       # unique ID for this run
${{ github.run_number }}   # sequential build number
${{ github.server_url }}   # https://github.com
 
# Runner context:
${{ runner.os }}     # Linux, Windows, macOS
${{ runner.temp }}   # temp directory path
 
# Secrets:
${{ secrets.MY_SECRET }}   # encrypted secret value
 
# Env:
${{ env.MY_VAR }}   # environment variable
```
 
---
 
### GitHub Environments — Deployment Protection
 
GitHub Environments add protection rules to deployments:
 
```
Repository → Settings → Environments → New environment
 
Production environment settings:
Required reviewers: varun, team-lead  ← manual approval needed
Wait timer: 10 minutes                ← cooling off period
Branch restrictions: main only        ← only from main branch
```
 
```yaml
deploy-production:
  environment:
    name: production       # references the environment
    url: https://myapp.com # shown in GitHub UI
```
 
> When the pipeline reaches this job, it pauses and sends an email to required reviewers. They approve or reject in the GitHub UI.
 
---
 
### Pull Request Checks
 
When someone opens a PR, the pipeline runs automatically:
 
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
 
> The PR cannot be merged until all checks pass. This enforces code quality across the team.
 
---
 
### Useful Actions from Marketplace
 
```yaml
# Checkout code:
- uses: actions/checkout@v4
 
# Set up Python:
- uses: actions/setup-python@v5
  with:
    python-version: "3.11"
 
# Set up Node.js:
- uses: actions/setup-node@v4
  with:
    node-version: "18"
 
# Cache dependencies:
- uses: actions/cache@v3
 
# Upload artifact:
- uses: actions/upload-artifact@v3
 
# Download artifact:
- uses: actions/download-artifact@v3
 
# Docker login:
- uses: docker/login-action@v3
 
# Docker build and push:
- uses: docker/build-push-action@v5
 
# Configure AWS credentials:
- uses: aws-actions/configure-aws-credentials@v4
 
# Login to ECR:
- uses: aws-actions/amazon-ecr-login@v2
 
# Trivy security scan:
- uses: aquasecurity/trivy-action@master
 
# Create GitHub Release:
- uses: actions/create-release@v1
 
# Comment on PR:
- uses: actions/github-script@v7
```
 
---
 
### Reusable Workflows — DRY Principle
 
```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test
 
on:
  workflow_call:        # can be called by other workflows
    inputs:
      python-version:
        required: true
        type: string
    secrets:
      CODECOV_TOKEN:
        required: true
 
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: pytest
```
 
```yaml
# .github/workflows/main.yml
jobs:
  run-tests:
    uses: ./.github/workflows/reusable-test.yml
    with:
      python-version: "3.11"
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```
 
---
 
### GitHub Actions for AWS ECR
 
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-south-1
 
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
 
- name: Build and push to ECR
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
    docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
    echo "image=$ECR_REGISTRY/myapp:$IMAGE_TAG" >> $GITHUB_OUTPUT
```
 
---
 
### Full Summary — CI/CD Day 2
 
| Concept | Key point |
|---------|-----------|
| Workflow | YAML file in `.github/workflows/` |
| Event | `on:` push, pull_request, schedule |
| Job | Group of steps — runs on one runner |
| Step | Single task — run command or uses action |
| Runner | VM that runs the job — `ubuntu-latest` |
| `needs` | Job dependency — sequential execution |
| `if` | Conditional — only run in certain conditions |
| Secrets | Encrypted — reference with `${{ secrets.NAME }}` |
| Cache | Speed up builds — cache pip, npm etc |
| Artifact | Save files between jobs or for download |
| Matrix | Test on multiple versions in parallel |
| Environment | staging, production — with approval gates |
| Contexts | `github.sha`, `github.ref`, `runner.os` |
 
---
 
### Interview Questions — CI/CD Day 2
 
**Q1. What is GitHub Actions and how does it work?**
GitHub Actions is a CI/CD platform built directly into GitHub. You define workflows as YAML files in `.github/workflows/`. When events occur — push, pull request, schedule — GitHub automatically runs the workflows on managed virtual machines called runners. Each workflow has jobs, each job has steps that either run shell commands or use pre-built marketplace actions.
 
**Q2. What is the difference between a job and a step in GitHub Actions?**
A job is a group of steps that run on the same runner virtual machine. Steps within a job run sequentially and share the same filesystem. Multiple jobs run in parallel by default — unless you use `needs` to create dependencies. Steps are individual tasks within a job — either `run` (shell command) or `uses` (marketplace action).
 
**Q3. How do you pass data between jobs in GitHub Actions?**
Three ways — job outputs (for small values), artifacts (for files), and environment files. Job outputs use `echo "key=value" >> $GITHUB_OUTPUT` in one job and `${{ needs.job-name.outputs.key }}` in the next. Artifacts use `upload-artifact` and `download-artifact` actions to pass files between jobs. The artifact is temporarily stored on GitHub servers.
 
**Q4. How do you secure secrets in GitHub Actions?**
Store secrets in repository Settings → Secrets and variables → Actions. Reference them in the workflow with `${{ secrets.SECRET_NAME }}`. Secrets are encrypted at rest, only available to workflows in the repository, and masked in logs — never printed even if you try to echo them. For AWS, use OIDC (OpenID Connect) with `aws-actions/configure-aws-credentials` — eliminates the need for static access keys.
 
**Q5. What is a matrix build and when do you use it?**
Matrix builds run the same job with different combinations of variables simultaneously. Used to test on multiple Python/Node/Java versions at once, multiple operating systems, or multiple dependency versions. All matrix combinations run in parallel — faster than sequential. If any combination fails, the job fails.
 
**Q6. How do you implement manual approval in GitHub Actions?**
Create a GitHub Environment (Settings → Environments) with required reviewers configured. Reference the environment in a deploy job with `environment: name: production`. When the pipeline reaches that job, it pauses and sends an email to reviewers. They approve or reject in the GitHub UI. The job only proceeds on approval.
 
---
 
### Homework — Before CI/CD Day 3
 
```bash
# 1. Add .github/workflows/ci.yml to your repo with:
#    - checkout, setup-python, install deps, run tests
 
# 2. Push and watch it run in GitHub Actions tab
 
# 3. Add Docker Hub secrets to your repository:
#    DOCKERHUB_USERNAME and DOCKERHUB_TOKEN
 
# 4. Add a build job that:
#    - Only runs on main branch
#    - Only runs if test job passes
#    - Builds and pushes a Docker image
 
# 5. Break a test intentionally — push — watch pipeline fail
 
# 6. Fix the test — push — watch pipeline pass
 
# 7. Check Docker Hub — verify your image is there
 
# 8. Try this workflow trigger — add to your workflow:
on:
  workflow_dispatch:   # adds manual trigger button
    inputs:
      environment:
        description: 'Deploy to which environment?'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```