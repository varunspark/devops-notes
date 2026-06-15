## CI/CD Day 4 — Complete End-to-End Pipeline

---

### What You Will Build Today

A complete production pipeline that:

```
Developer pushes code
   ↓
GitHub Actions triggers automatically
   ↓
Job 1 (test):     Lint + unit tests + coverage
   ↓
Job 2 (build):    Authenticate via OIDC → Docker build → push to ECR
   ↓
Job 3 (deploy-staging):   SSH to staging EC2 → deploy → health check
   ↓
Job 4 (integration-test): Real HTTP tests against staging
   ↓
Job 5 (deploy-production): Manual approval → ASG instance refresh
   ↓
Job 6 (notify):   Slack notification — success or failure
```

---

### What You Will Learn Today

- OIDC authentication — AWS with no static keys
- AWS ECR — private Docker registry
- Staging deploy via SSH
- Deploy script and health check script
- CI pipeline for pull requests
- Complete deploy pipeline with 6 jobs
- Rollback pipeline — specify any previous tag
- Scheduled maintenance pipeline
- End-to-end pipeline visualization

---

### Step 1 — OIDC Authentication Setup (No Static AWS Keys)

OIDC (OpenID Connect) lets GitHub Actions get temporary AWS credentials — no stored access keys.

```bash
# Create the OIDC Identity Provider in AWS:
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create trust policy:
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPO:*"
        }
      }
    }
  ]
}
EOF

# Create the IAM role:
aws iam create-role \
  --role-name github-actions-role \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions to the role:
aws iam attach-role-policy \
  --role-name github-actions-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess

aws iam attach-role-policy \
  --role-name github-actions-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# Get the role ARN — add this to GitHub Secrets as AWS_ROLE_ARN:
aws iam get-role \
  --role-name github-actions-role \
  --query 'Role.Arn' \
  --output text
```

> The trust policy uses `StringLike` with `*` to allow all branches and tags from your specific repo. Only your repo can assume this role — nobody else.

---

### Step 2 — Create ECR Repository

```bash
# Create the repository:
aws ecr create-repository \
  --repository-name myapp \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Add lifecycle policy to auto-clean old images:
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 20 images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["sha-"],
          "countType": "imageCountMoreThan",
          "countNumber": 20
        },
        "action": { "type": "expire" }
      },
      {
        "rulePriority": 2,
        "description": "Delete untagged images after 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": { "type": "expire" }
      }
    ]
  }'
```

---

### Step 3 — GitHub Secrets and Variables

**Secrets** (sensitive — encrypted):

```
AWS_ROLE_ARN       ← IAM role ARN for OIDC
SSH_PRIVATE_KEY    ← .pem file contents for staging EC2
SLACK_WEBHOOK      ← Slack incoming webhook URL
```

**Variables** (non-sensitive — readable):

```
AWS_REGION         ← ap-south-1
ECR_REPOSITORY     ← myapp
STAGING_HOST       ← staging EC2 public IP
APP_PORT           ← 5000
ASG_NAME           ← myapp-asg
```

```
Repository → Settings → Secrets and variables → Actions
→ New repository secret
→ New repository variable
```

---

### Step 4 — Deploy and Health Check Scripts

```bash
# scripts/deploy.sh — runs on EC2 server
cat > scripts/deploy.sh << 'EOF'
#!/bin/bash
set -euo pipefail

IMAGE=$1
APP_NAME=${2:-myapp}
PORT=${3:-5000}

log() { echo "[$(date '+%H:%M:%S')] $1"; }

log "Deploying: $IMAGE"

# Login to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login \
    --username AWS \
    --password-stdin \
    $(echo $IMAGE | cut -d'/' -f1)

# Pull new image
log "Pulling image..."
docker pull $IMAGE

# Stop and remove old container
log "Stopping old container..."
docker stop $APP_NAME 2>/dev/null || true
docker rm   $APP_NAME 2>/dev/null || true

# Start new container
log "Starting new container..."
docker run -d \
  --name $APP_NAME \
  --restart unless-stopped \
  -p $PORT:$PORT \
  -e ENVIRONMENT=production \
  --health-cmd="curl -f http://localhost:$PORT/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  $IMAGE

# Wait for health check
log "Waiting for container health..."
MAX_WAIT=60
WAITED=0

while [ $WAITED -lt $MAX_WAIT ]; do
  STATUS=$(docker inspect $APP_NAME \
    --format='{{.State.Health.Status}}' 2>/dev/null || echo "starting")

  if [ "$STATUS" = "healthy" ]; then
    log "✅ Container is healthy!"
    break
  elif [ "$STATUS" = "unhealthy" ]; then
    log "❌ Container is unhealthy!"
    docker logs $APP_NAME --tail 20
    exit 1
  fi

  log "Status: $STATUS — waiting..."
  sleep 5
  WAITED=$((WAITED + 5))
done

# Clean up old images
docker image prune -f

log "Deployment complete: $IMAGE"
EOF

chmod +x scripts/deploy.sh
```

```bash
# scripts/health-check.sh — verify deployment
cat > scripts/health-check.sh << 'EOF'
#!/bin/bash
set -euo pipefail

HOST=${1:-localhost}
PORT=${2:-5000}
MAX_RETRIES=${3:-10}
WAIT_SECONDS=${4:-5}

URL="http://${HOST}:${PORT}/health"
ATTEMPT=0

echo "Checking health: $URL"

while [ $ATTEMPT -lt $MAX_RETRIES ]; do
  ATTEMPT=$((ATTEMPT + 1))
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    --max-time 5 "$URL" 2>/dev/null || echo "000")

  if [ "$HTTP_CODE" = "200" ]; then
    echo "✅ Health check passed (attempt $ATTEMPT)"
    exit 0
  fi

  echo "Attempt $ATTEMPT/$MAX_RETRIES — HTTP $HTTP_CODE"
  sleep $WAIT_SECONDS
done

echo "❌ Health check failed after $MAX_RETRIES attempts"
exit 1
EOF

chmod +x scripts/health-check.sh
```

---

### Step 5 — CI Pipeline (Pull Request Checks)

```yaml
# .github/workflows/ci.yml
name: CI — Pull Request Checks

on:
  pull_request:
    branches: [main, develop]

env:
  PYTHON_VERSION: "3.11"

jobs:

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Cache pip
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install flake8 pytest-cov

      - name: Lint
        run: flake8 . --max-line-length=100 --exclude=.git,__pycache__

      - name: Test
        run: |
          pytest test_app.py -v \
            --cov=app \
            --cov-report=xml \
            --junitxml=test-results.xml

      - name: Upload results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results-${{ github.sha }}
          path: |
            test-results.xml
            coverage.xml

  docker-build-check:
    name: Docker Build Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image (no push)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: myapp:pr-${{ github.event.pull_request.number }}

      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL'
          exit-code: '1'
```

---

### Step 6 — Complete Deploy Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      force_deploy:
        description: 'Force deploy even if tests fail?'
        required: false
        default: false
        type: boolean

permissions:
  id-token: write     # required for OIDC
  contents: read
  packages: write

env:
  AWS_REGION:       ${{ vars.AWS_REGION }}
  ECR_REPOSITORY:   ${{ vars.ECR_REPOSITORY }}
  APP_NAME:         myapp

jobs:

  # ════════════════════════════════════════
  # JOB 1 — TEST
  # ════════════════════════════════════════
  test:
    name: Test
    runs-on: ubuntu-latest
    if: ${{ !inputs.force_deploy }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

      - name: Test
        run: |
          pip install -r requirements.txt pytest-cov flake8
          flake8 . --max-line-length=100 --exclude=.git,__pycache__
          pytest test_app.py -v --cov=app --cov-report=xml

      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: coverage-${{ github.sha }}
          path: coverage.xml

  # ════════════════════════════════════════
  # JOB 2 — BUILD AND PUSH TO ECR
  # ════════════════════════════════════════
  build:
    name: Build and Push to ECR
    runs-on: ubuntu-latest
    needs: test
    if: always() && (needs.test.result == 'success' || inputs.force_deploy)

    outputs:
      image-uri: ${{ steps.build.outputs.image-uri }}
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=sha-,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        id: build-push
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
            VERSION=${{ steps.meta.outputs.version }}

      - name: Set image URI output
        id: build
        run: |
          IMAGE_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:sha-$(echo ${{ github.sha }} | cut -c1-7)"
          echo "image-uri=$IMAGE_URI" >> $GITHUB_OUTPUT
          echo "Built and pushed: $IMAGE_URI"

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.build.outputs.image-uri }}
          format: 'table'
          exit-code: '0'
          severity: 'CRITICAL,HIGH'

  # ════════════════════════════════════════
  # JOB 3 — DEPLOY TO STAGING
  # ════════════════════════════════════════
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: http://${{ vars.STAGING_HOST }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Set up SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ vars.STAGING_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy to staging EC2
        run: |
          IMAGE="${{ needs.build.outputs.image-uri }}"
          HOST="${{ vars.STAGING_HOST }}"

          echo "Deploying $IMAGE to staging: $HOST"

          # Copy deploy script to server
          scp -i ~/.ssh/deploy_key \
            scripts/deploy.sh \
            ubuntu@$HOST:/tmp/deploy.sh

          # Run deployment
          ssh -i ~/.ssh/deploy_key ubuntu@$HOST \
            "chmod +x /tmp/deploy.sh && \
             /tmp/deploy.sh $IMAGE ${{ env.APP_NAME }} ${{ vars.APP_PORT }}"

      - name: Staging health check
        run: |
          bash scripts/health-check.sh \
            ${{ vars.STAGING_HOST }} \
            ${{ vars.APP_PORT }} \
            12 \
            10

      - name: Verify deployment
        run: |
          RESPONSE=$(curl -s http://${{ vars.STAGING_HOST }}:${{ vars.APP_PORT }}/api/info)
          echo "App info: $RESPONSE"

          echo "$RESPONSE" | python3 -c "
          import sys, json
          info = json.load(sys.stdin)
          print(f'Version: {info.get(\"version\", \"unknown\")}')
          print(f'Environment: {info.get(\"environment\", \"unknown\")}')
          "

  # ════════════════════════════════════════
  # JOB 4 — INTEGRATION TESTS ON STAGING
  # ════════════════════════════════════════
  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: deploy-staging

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Run integration tests against staging
        env:
          BASE_URL: http://${{ vars.STAGING_HOST }}:${{ vars.APP_PORT }}
        run: |
          pip install requests pytest

          # Write integration tests inline:
          cat > test_integration.py << 'TESTEOF'
          import requests
          import os
          import pytest

          BASE_URL = os.environ.get('BASE_URL', 'http://localhost:5000')

          def test_health_endpoint():
              response = requests.get(f"{BASE_URL}/health", timeout=5)
              assert response.status_code == 200
              data = response.json()
              assert data['status'] == 'healthy'

          def test_info_endpoint():
              response = requests.get(f"{BASE_URL}/api/info", timeout=5)
              assert response.status_code == 200
              data = response.json()
              assert 'version' in data

          def test_response_time():
              import time
              start = time.time()
              requests.get(f"{BASE_URL}/health", timeout=5)
              elapsed = time.time() - start
              assert elapsed < 2.0, f"Response too slow: {elapsed:.2f}s"
          TESTEOF

          pytest test_integration.py -v

  # ════════════════════════════════════════
  # JOB 5 — DEPLOY TO PRODUCTION
  # ════════════════════════════════════════
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, integration-test]
    if: |
      github.ref == 'refs/heads/main' &&
      (inputs.environment == 'production' || startsWith(github.ref, 'refs/tags/v'))
    environment:
      name: production
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update Launch Template
        run: |
          IMAGE="${{ needs.build.outputs.image-uri }}"
          echo "Deploying to production: $IMAGE"

          # Get current launch template version
          CURRENT_VERSION=$(aws ec2 describe-launch-templates \
            --launch-template-names myapp-lt \
            --query 'LaunchTemplates[0].LatestVersionNumber' \
            --output text)

          echo "Current LT version: $CURRENT_VERSION"

          # Create new version with updated user data
          USER_DATA=$(base64 -w 0 << USEREOF
          #!/bin/bash
          IMAGE=$IMAGE
          APP_NAME=myapp

          aws ecr get-login-password --region ap-south-1 | \
            docker login --username AWS --password-stdin \
            $(echo $IMAGE | cut -d'/' -f1)

          docker pull $IMAGE
          docker stop $APP_NAME || true
          docker rm   $APP_NAME || true
          docker run -d \
            --name $APP_NAME \
            --restart unless-stopped \
            -p 80:5000 \
            -e ENVIRONMENT=production \
            $IMAGE
          USEREOF
          )

          aws ec2 create-launch-template-version \
            --launch-template-name myapp-lt \
            --source-version '$Latest' \
            --launch-template-data "{\"UserData\":\"$USER_DATA\"}"

          echo "Launch Template updated"

      - name: Start instance refresh (rolling deployment)
        id: refresh
        run: |
          REFRESH_ID=$(aws autoscaling start-instance-refresh \
            --auto-scaling-group-name ${{ vars.ASG_NAME }} \
            --preferences '{
              "MinHealthyPercentage": 80,
              "InstanceWarmup": 120
            }' \
            --query 'InstanceRefreshId' \
            --output text)

          echo "refresh-id=$REFRESH_ID" >> $GITHUB_OUTPUT
          echo "Instance refresh started: $REFRESH_ID"

      - name: Wait for refresh to complete
        run: |
          REFRESH_ID="${{ steps.refresh.outputs.refresh-id }}"
          ASG="${{ vars.ASG_NAME }}"
          MAX_WAIT=1200   # 20 minutes
          WAITED=0

          echo "Waiting for instance refresh to complete..."

          while [ $WAITED -lt $MAX_WAIT ]; do
            STATUS=$(aws autoscaling describe-instance-refreshes \
              --auto-scaling-group-name $ASG \
              --instance-refresh-ids $REFRESH_ID \
              --query 'InstanceRefreshes[0].Status' \
              --output text)

            PROGRESS=$(aws autoscaling describe-instance-refreshes \
              --auto-scaling-group-name $ASG \
              --instance-refresh-ids $REFRESH_ID \
              --query 'InstanceRefreshes[0].PercentageComplete' \
              --output text)

            echo "Status: $STATUS — Progress: ${PROGRESS}%"

            case $STATUS in
              Successful)
                echo "✅ Instance refresh complete!"
                break
                ;;
              Failed|Cancelled)
                echo "❌ Instance refresh $STATUS!"
                exit 1
                ;;
              *)
                sleep 30
                WAITED=$((WAITED + 30))
                ;;
            esac
          done

      - name: Production health check
        run: |
          bash scripts/health-check.sh \
            myapp.com \
            443 \
            12 \
            15

  # ════════════════════════════════════════
  # JOB 6 — NOTIFY
  # ════════════════════════════════════════
  notify:
    name: Notify Team
    runs-on: ubuntu-latest
    needs: [test, build, deploy-staging, integration-test]
    if: always()

    steps:
      - name: Determine status
        id: status
        run: |
          if [[ "${{ needs.deploy-staging.result }}" == "success" ]]; then
            echo "status=success" >> $GITHUB_OUTPUT
            echo "emoji=✅"       >> $GITHUB_OUTPUT
            echo "color=good"     >> $GITHUB_OUTPUT
          else
            echo "status=failure" >> $GITHUB_OUTPUT
            echo "emoji=❌"       >> $GITHUB_OUTPUT
            echo "color=danger"   >> $GITHUB_OUTPUT
          fi

      - name: Send Slack notification
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
          IMAGE="${{ needs.build.outputs.image-uri }}"

          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            -d "{
              \"attachments\": [{
                \"color\": \"${{ steps.status.outputs.color }}\",
                \"title\": \"${{ steps.status.outputs.emoji }} Deployment: myapp\",
                \"fields\": [
                  {\"title\": \"Status\",  \"value\": \"${{ steps.status.outputs.status }}\", \"short\": true},
                  {\"title\": \"Branch\",  \"value\": \"${{ github.ref_name }}\",             \"short\": true},
                  {\"title\": \"Commit\",  \"value\": \"$SHORT_SHA\",                         \"short\": true},
                  {\"title\": \"By\",      \"value\": \"${{ github.actor }}\",                \"short\": true},
                  {\"title\": \"Image\",   \"value\": \"$IMAGE\"}
                ],
                \"footer\": \"<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Pipeline>\"
              }]
            }" || true
```

---

### Step 7 — Rollback Pipeline

```yaml
# .github/workflows/rollback.yml
name: Emergency Rollback

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to roll back to (e.g. sha-abc1234)'
        required: true
      environment:
        description: 'Environment to roll back'
        required: true
        default: 'production'
        type: choice
        options: [staging, production]
      reason:
        description: 'Reason for rollback'
        required: true

permissions:
  id-token: write
  contents: read

jobs:
  rollback:
    name: Rollback ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Verify rollback image exists
        run: |
          IMAGE_TAG="${{ inputs.image_tag }}"
          REPO="${{ vars.ECR_REPOSITORY }}"

          echo "Verifying image exists: $REPO:$IMAGE_TAG"

          aws ecr describe-images \
            --repository-name $REPO \
            --image-ids imageTag=$IMAGE_TAG \
            --region ${{ vars.AWS_REGION }} || {
            echo "❌ Image not found: $REPO:$IMAGE_TAG"
            echo "Available tags:"
            aws ecr list-images \
              --repository-name $REPO \
              --query 'imageIds[*].imageTag' \
              --output table
            exit 1
          }

          echo "✅ Image verified: $REPO:$IMAGE_TAG"

      - name: Set up SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ vars.STAGING_HOST }} >> ~/.ssh/known_hosts

      - name: Execute rollback
        run: |
          ACCOUNT_ID=$(aws sts get-caller-identity \
            --query Account --output text)
          IMAGE="${ACCOUNT_ID}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com/${{ vars.ECR_REPOSITORY }}:${{ inputs.image_tag }}"
          HOST="${{ vars.STAGING_HOST }}"

          echo "Rolling back to: $IMAGE"
          echo "Reason: ${{ inputs.reason }}"

          scp -i ~/.ssh/deploy_key \
            scripts/deploy.sh \
            ubuntu@$HOST:/tmp/deploy.sh

          ssh -i ~/.ssh/deploy_key ubuntu@$HOST \
            "chmod +x /tmp/deploy.sh && \
             /tmp/deploy.sh $IMAGE ${{ env.APP_NAME }} ${{ vars.APP_PORT }}"

      - name: Verify rollback health
        run: |
          bash scripts/health-check.sh \
            ${{ vars.STAGING_HOST }} \
            ${{ vars.APP_PORT }} \
            10 \
            10

      - name: Notify rollback
        if: always()
        run: |
          STATUS=$([[ "${{ job.status }}" == "success" ]] && \
            echo "✅ Rollback succeeded" || echo "❌ Rollback FAILED")

          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            -d "{
              \"text\": \"🔄 *ROLLBACK* $STATUS\nEnv: ${{ inputs.environment }}\nTag: ${{ inputs.image_tag }}\nReason: ${{ inputs.reason }}\nBy: ${{ github.actor }}\"
            }" || true
```

---

### Step 8 — Scheduled Maintenance Pipeline

```yaml
# .github/workflows/maintenance.yml
name: Scheduled Maintenance

on:
  schedule:
    - cron: '0 2 * * 0'   # Every Sunday at 2 AM
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  maintenance:
    name: Weekly Maintenance
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Clean ECR old images
        run: |
          REPO="${{ vars.ECR_REPOSITORY }}"
          echo "Cleaning ECR repository: $REPO"

          aws ecr describe-images \
            --repository-name $REPO \
            --query "imageDetails[?imagePushedAt < \`$(date -d '30 days ago' --iso-8601)\`].imageDigest" \
            --output text | while read digest; do
            if [ -n "$digest" ]; then
              echo "Deleting old image: $digest"
              aws ecr batch-delete-image \
                --repository-name $REPO \
                --image-ids imageDigest=$digest || true
            fi
          done

          echo "ECR cleanup complete"

      - name: Check CloudWatch alarms
        run: |
          echo "=== Alarms in ALARM state ==="
          aws cloudwatch describe-alarms \
            --state-value ALARM \
            --query 'MetricAlarms[*].[AlarmName,StateReason]' \
            --output table

          echo ""
          echo "=== All alarm statuses ==="
          aws cloudwatch describe-alarms \
            --alarm-name-prefix myapp \
            --query 'MetricAlarms[*].[AlarmName,StateValue]' \
            --output table

      - name: Generate weekly report
        run: |
          echo "# Weekly Infrastructure Report" > report.md
          echo "Date: $(date)" >> report.md
          echo "" >> report.md

          echo "## EC2 Instances" >> report.md
          aws ec2 describe-instances \
            --filters Name=tag:Project,Values=myapp \
            --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name,PublicIpAddress]' \
            --output table >> report.md

          echo "" >> report.md
          echo "## RDS Status" >> report.md
          aws rds describe-db-instances \
            --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,DBInstanceClass]' \
            --output table >> report.md

          cat report.md

      - uses: actions/upload-artifact@v3
        with:
          name: weekly-report-${{ github.run_number }}
          path: report.md
```

---

### Step 9 — Complete Pipeline Flow Summary

```
Push to feature/* branch:
→ ci.yml triggers
→ Tests + Lint only
→ No deploy

Push to develop branch:
→ deploy.yml triggers
→ Tests → Build → Push ECR → Deploy staging → Integration tests
→ Notify team

Push to main branch:
→ deploy.yml triggers
→ Tests → Build → Push ECR
→ Deploy staging → Integration tests
→ Wait for approval (production environment)
→ Deploy production via instance refresh
→ Notify team

Tag v* (release):
→ deploy.yml triggers
→ Full pipeline
→ Auto deploys to production
→ Creates GitHub Release

Manual rollback:
→ rollback.yml
→ Specify tag to roll back to
→ Verify image exists in ECR
→ Deploy old image → Health check

Sunday 2 AM:
→ maintenance.yml
→ Clean old ECR images
→ Check CloudWatch alarms
→ Generate weekly report
```

---

### Full Summary — CI/CD Day 4

| Concept | Key point |
|---------|-----------|
| OIDC | No static AWS keys — temporary tokens via GitHub |
| ECR | AWS private registry — push/pull via IAM role |
| Staging deploy | SSH to EC2 — run deploy script |
| Production deploy | Instance refresh on ASG — zero downtime rolling update |
| Integration tests | Run against staging after deploy — verify before production |
| Rollback | Manual workflow — specify tag — deploy old image |
| Scheduled | Maintenance, cleanup, reporting via cron triggers |
| Environment protection | Manual approval gate for production |
| OIDC trust policy | Only this specific repo can assume the role |
| Image tagging | SHA tag — always know what is running |

---

### Interview Questions — CI/CD Day 4

**Q1. How do you authenticate GitHub Actions with AWS without storing access keys?**
Using OIDC (OpenID Connect). Create an IAM OIDC identity provider pointing to GitHub's token endpoint. Create an IAM role with a trust policy that allows only your specific repository to assume it. In the workflow use `aws-actions/configure-aws-credentials` with `role-to-assume` — GitHub gets a temporary token from AWS that expires after the pipeline. No static credentials stored anywhere.

**Q2. What is the difference between deploying to EC2 directly versus Auto Scaling Group?**
Deploying directly to EC2 via SSH is simple but affects only that one instance and causes a brief restart. Deploying via ASG instance refresh is more complex but performs a rolling replacement — launches new instances with updated configuration, waits for health checks, terminates old ones. Maintains minimum healthy percentage throughout — zero downtime. Production always uses ASG for this reason.

**Q3. How do you implement a rollback in your pipeline?**
Every Docker image is tagged with the git commit SHA and pushed to ECR. To rollback — run the manual rollback workflow, provide the previous image tag, the pipeline pulls that specific image from ECR and deploys it. Because every version is preserved in ECR you can rollback to any previous commit. The rollback takes exactly as long as a normal deployment — about 5 minutes.

**Q4. How do you handle deployment to multiple environments in GitHub Actions?**
Use GitHub Environments — create staging and production environments in repository settings. Assign different variables (server IPs, ports) and protection rules to each. The staging environment deploys automatically on every main branch push. The production environment has required reviewers — pipeline pauses and sends approval request. After approval production deployment proceeds. Different secrets and variables are scoped to each environment.

**Q5. What is an integration test in the context of CI/CD?**
Tests that run against a deployed application — not just unit tests that test code in isolation. After deploying to staging the pipeline sends real HTTP requests to the running application — checking health endpoint returns 200, API endpoints return correct data, response times are acceptable. Integration tests catch issues that unit tests miss — wrong configuration, networking problems, database connectivity. Only if integration tests pass does production deployment proceed.

**Q6. How do you clean up old Docker images in ECR automatically?**
ECR lifecycle policies automatically expire images based on rules — for example keep only the last 10 images or delete images older than 30 days. Apply with `aws ecr put-lifecycle-policy`. Also run periodic cleanup in a scheduled maintenance pipeline. Without cleanup ECR fills up with hundreds of old images costing money and making it hard to find recent versions.

---

### Homework — Before CI/CD Day 5

```bash
# 1. Set up OIDC in your AWS account:
#    Follow Step 1 above — create OIDC provider and IAM role

# 2. Create ECR repository:
aws ecr create-repository \
  --repository-name myapp \
  --region ap-south-1

# 3. Add all secrets to GitHub:
#    AWS_ROLE_ARN, SSH_PRIVATE_KEY, SLACK_WEBHOOK

# 4. Add all variables:
#    AWS_REGION, ECR_REPOSITORY, STAGING_HOST, APP_PORT

# 5. Push ci.yml and deploy.yml to your repository

# 6. Create a pull request — verify ci.yml runs

# 7. Merge to main — verify deploy.yml runs

# 8. Check ECR — verify your image is there:
aws ecr list-images \
  --repository-name myapp \
  --query 'imageIds[*].imageTag' \
  --output table

# 9. Try the rollback workflow manually:
#    Go to Actions → Emergency Rollback → Run workflow
#    Specify a previous image tag
```

> CI/CD Day 5 is the Interview Mega Revision — every CI/CD concept in interview format, scenario questions, architecture questions and the exact questions asked at 10–12 LPA companies.
