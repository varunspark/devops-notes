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

### Both tools do the same job. Jenkins gives you more control. GitHub Actions is easier to set

### up. You need to know both.

## What You Will Learn Today

### Installing Jenkins on EC 2

### Jenkins architecture — master and agents

### Jenkinsfile — pipeline as code

### Declarative vs Scripted pipeline

### Stages and steps

### Environment variables and credentials

### Jenkins plugins

### Webhook — auto trigger on Git push

### Parallel stages

### Post actions — always, success, failure

### Complete pipeline — test, build, push, deploy

## Jenkins Architecture

```
Enterprise (500+): Jenkins — 80 %, GitHub Actions — 20 %
```
```
At 10-12 LPA level — you will interview at mid-size companies
where Jenkins is the most common CI/CD tool.
```
```
Interview reality:
"Do you know Jenkins?" — asked in 70 % of DevOps interviews
"Show me a Jenkinsfile" — asked in 50 % of technical rounds
```
```
Jenkins Master (EC 2 — controller)
├── Web UI (port 8080)
├── Pipeline scheduler
├── Plugin manager
└── Credential store
```
```
Jenkins Agents (workers — where builds run)
├── Agent 1 (same EC 2 or separate)
├── Agent 2
└── Agent 3
```
```
Master distributes work to agents
Agents run the actual build steps
Master just coordinates — never runs builds directly in production
```

### For learning — master and agent on the same EC 2 is fine. In production — separate agents

### for scalability.

## Step 1 — Install Jenkins on EC 2

### Launch a t2.medium EC 2 (Jenkins needs more RAM than t2.micro):

```
bash
```
```
# User Data for Jenkins EC2:
#!/bin/bash
set - e
exec > /var/log/jenkins-setup.log 2 >& 1
```
```
echo "=== Installing Jenkins ==="
```
```
# Update system
apt-get update -y
```
```
# Install Java (Jenkins requires Java 11 or 17):
apt-get install -y openjdk-17-jdk
```
```
# Verify Java:
java -version
```
```
# Add Jenkins repository:
curl - fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```
```
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | \
tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
```
# Install Jenkins:
apt-get update -y
apt-get install -y jenkins
```
```
# Start Jenkins:
systemctl start jenkins
systemctl enable jenkins
```
```
# Install Docker (for building images):
curl - fsSL https://get.docker.com | bash
usermod - aG docker jenkins # allow Jenkins to use Docker
usermod - aG docker ubuntu
```
```
# Install AWS CLI:
curl - fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
```

### Security group for Jenkins EC2:

## Step 2 — Initial Jenkins Setup

```
-o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip - d /tmp
/tmp/aws/install
```
```
# Wait for Jenkins to start:
sleep 30
```
```
echo "=== Jenkins setup complete ==="
echo "Initial admin password:"
cat /var/lib/jenkins/secrets/initialAdminPassword
```
```
bash
# Allow port 8080 from your IP:
aws ec 2 authorize-security-group-ingress \
--group-id $SG_ID \
--protocol tcp \
--port 8080 \
--cidr $(curl -s ifconfig.me)/
```
```
# Allow SSH:
aws ec 2 authorize-security-group-ingress \
--group-id $SG_ID \
--protocol tcp \
--port 22 \
--cidr $(curl -s ifconfig.me)/
```
```
bash
```
```
# Get initial admin password:
ssh - i mykey.pem ubuntu@<jenkins-ec2-ip>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
```
# Open browser:
http://<jenkins-ec2-ip>:
```
```
# Paste the initial password
# Click: Install suggested plugins (wait 3-5 minutes)
# Create admin user: admin / YourPassword 123
# Jenkins URL: http://<jenkins-ec2-ip>:
# Save and finish
```

## Step 4 — Add Credentials to Jenkins

```
Jenkins → Manage Jenkins → Plugins → Available
```
```
Install these plugins:
✅ Pipeline (already installed)
✅ Git (already installed)
✅ Docker Pipeline search and install
✅ GitHub Integration search and install
✅ Blue Ocean search and install (better UI)
✅ Credentials Binding search and install
✅ Workspace Cleanup search and install
✅ Timestamper search and install
✅ AnsiColor search and install (colored output)
```
```
After installing → Restart Jenkins
```
```
Jenkins → Manage Jenkins → Credentials
→ System → Global credentials
→ Add Credentials
```
```
Add these:
```
1. Docker Hub:
    Kind: Username with password
    Username: your-dockerhub-username
    Password: your-dockerhub-token
    ID: dockerhub-credentials
    Description: Docker Hub Access
2. GitHub:
    Kind: Secret text
    Secret: your-github-token
    ID: github-token
    Description: GitHub Personal Access Token
3. SSH Key for deployment:
    Kind: SSH Username with private key
    Username: ubuntu
    Private key: paste your .pem file contents
    ID: deployment-ssh-key
    Description: Production Server SSH Key
4. AWS Credentials:
    Kind: AWS Credentials (needs AWS Credentials plugin)
    Access Key ID: your-access-key


# Jenkinsfile — Pipeline as Code

## Declarative vs Scripted Pipeline

## Always use Declarative. It is cleaner, easier to read, has better error messages and is what

## interviewers expect.

## Declarative Pipeline Structure

```
Secret Access Key: your-secret-key
ID: aws-credentials
Description: AWS Access
```
```
groovy
```
```
// Declarative — structured, opinionated, recommended:
pipeline {
agent any
stages {
stage('Test') {
steps {
sh 'pytest'
}
}
}
}
```
```
// Scripted — full Groovy, more flexible:
node {
stage('Test') {
sh 'pytest'
}
}
```
```
groovy
```
```
pipeline {
```
```
agent any // where to run
```
```
options { } // pipeline-level options
```
```
environment { } // environment variables
```
```
parameters { } // input parameters
```

## Step 5 — Your First Jenkinsfile

### Create Jenkinsfile in your repository root:

```
triggers { } // what triggers the build
```
```
stages {
stage('Stage Name') {
when { } // conditional execution
steps {
sh 'command' // shell command
script { } // Groovy script block
}
post { } // after stage actions
}
}
```
```
post { } // after all stages
}
```
```
groovy
```
```
pipeline {
agent any
```
```
environment {
APP_NAME = 'myapp'
PYTHON_VERSION = '3.11'
}
```
```
stages {
```
```
stage('Checkout') {
steps {
// Jenkins automatically checks out code
// when triggered by webhook
echo "Building branch: ${env.BRANCH_NAME}"
echo "Commit: ${env.GIT_COMMIT}"
sh 'git log --oneline -5'
}
}
```
```
stage('Setup Python') {
steps {
sh '''
python3 --version
python3 -m pip install --upgrade pip
```

## Step 6 — Create Jenkins Pipeline Job

```
pip install -r requirements.txt
pip install flake 8 pytest-cov
'''
}
}
```
```
stage('Lint') {
steps {
sh 'flake 8. --max-line-length= 100 --exclude=.git,__pycache__'
}
}
```
```
stage('Test') {
steps {
sh '''
pytest test_app.py -v \
--cov=app \
--cov-report=xml \
--junitxml=test-results.xml
'''
}
post {
always {
// Publish test results even if tests fail
junit 'test-results.xml'
}
}
}
}
```
```
post {
success {
echo '✅ Pipeline passed!'
}
failure {
echo '❌ Pipeline failed!'
}
always {
cleanWs() // clean workspace after every build
}
}
}
```
```
Jenkins → New Item
→ Name: myapp-pipeline
```

### Watch the build run in the console output.

## Step 7 — Complete Production Jenkinsfile

```
→ Type: Pipeline
→ OK
```
```
Configuration:
General:
✅ Discard old builds → Max: 10
```
```
Pipeline:
Definition: Pipeline script from SCM
SCM: Git
Repository URL: https://github.com/varun/myapp
Credentials: github-token
Branch: */main
Script Path: Jenkinsfile
```
```
→ Save
→ Build Now
```
```
groovy
```
```
pipeline {
agent any
```
```
// ── Options ────────────────────────────────────
options {
timestamps() // add timestamps to console
ansiColor('xterm') // colored output
timeout(time: 30 , unit: 'MINUTES') // fail if takes > 30 min
buildDiscarder(
logRotator(numToKeepStr: '10') // keep last 10 builds
)
disableConcurrentBuilds() // one build at a time
}
```
```
// ── Environment ────────────────────────────────
environment {
APP_NAME = 'myapp'
IMAGE_NAME = "varun/${APP_NAME}"
DOCKER_CREDS = credentials('dockerhub-credentials')
AWS_CREDS = credentials('aws-credentials')
GIT_SHA_SHORT = sh(script: 'git rev-parse --short HEAD',
returnStdout: true).trim()
}
```

// ── Parameters ─────────────────────────────────
parameters {
choice(
name: 'ENVIRONMENT',
choices: ['staging', 'production'],
description: 'Deploy to which environment?'
)
booleanParam(
name: 'SKIP_TESTS',
defaultValue: false,
description: 'Skip tests? (use only in emergency)'
)
}

// ── Trigger ────────────────────────────────────
triggers {
githubPush() // trigger on GitHub webhook
}

stages {

##### // ════════════════════════════════════════

##### // STAGE 1 — CHECKOUT

##### // ════════════════════════════════════════

stage('Checkout') {
steps {
checkout scm
script {
env.GIT_COMMIT_MSG = sh(
script: 'git log -1 --pretty=%B',
returnStdout: true
).trim()
echo "Commit: ${env.GIT_SHA_SHORT}"
echo "Message: ${env.GIT_COMMIT_MSG}"
echo "Branch: ${env.BRANCH_NAME}"
}
}
}

// ════════════════════════════════════════
// STAGE 2 — INSTALL
// ════════════════════════════════════════
stage('Install Dependencies') {
steps {
sh '''
python3 -m pip install --upgrade pip --quiet
pip install -r requirements.txt --quiet
pip install flake 8 pytest-cov --quiet
echo "Dependencies installed"


##### '''

##### }

##### }

##### // ════════════════════════════════════════

##### // STAGE 3 — QUALITY CHECKS

##### // ════════════════════════════════════════

stage('Code Quality') {
when {
not { expression { params.SKIP_TESTS } }
}
parallel {
stage('Lint') {
steps {
sh '''
echo "Running flake8..."
flake 8. \
--max-line-length= 100 \
--exclude=.git,__pycache__,.venv \
--statistics
'''
}
}
stage('Security Check') {
steps {
sh '''
pip install safety --quiet
safety check -r requirements.txt || true
'''
}
}
}
}

##### // ════════════════════════════════════════

##### // STAGE 4 — TEST

##### // ════════════════════════════════════════

stage('Test') {
when {
not { expression { params.SKIP_TESTS } }
}
steps {
sh '''
echo "Running tests..."
pytest test_app.py \
--verbose \
--cov=app \
--cov-report=xml:coverage.xml \
--cov-report=term-missing \


--junitxml=test-results.xml
'''
}
post {
always {
junit 'test-results.xml'
publishHTML(target: [
reportName: 'Coverage Report',
reportDir: '.',
reportFiles: 'coverage.xml',
keepAll: true
])
}
}
}

// ════════════════════════════════════════
// STAGE 5 — BUILD DOCKER IMAGE
// ════════════════════════════════════════
stage('Build Docker Image') {
steps {
script {
docker.build(
"${IMAGE_NAME}:${GIT_SHA_SHORT}",
"--build-arg BUILD_DATE=${new Date().format('yyyy-MM-dd')} \
--build-arg VCS_REF=${GIT_SHA_SHORT} \
."
)
echo "Image built: ${IMAGE_NAME}:${GIT_SHA_SHORT}"
}
}
}

// ════════════════════════════════════════
// STAGE 6 — SECURITY SCAN IMAGE
// ════════════════════════════════════════
stage('Scan Image') {
steps {
sh """
docker run --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
aquasec/trivy:latest image \
--severity HIGH,CRITICAL \
--exit-code 0 \
${IMAGE_NAME}:${GIT_SHA_SHORT}
"""
}
}


##### // ════════════════════════════════════════

##### // STAGE 7 — PUSH TO REGISTRY

##### // ════════════════════════════════════════

stage('Push to Registry') {
when {
anyOf {
branch 'main'
branch 'develop'
tag pattern: 'v.*', comparator: 'REGEXP'
}
}
steps {
script {
docker.withRegistry(
'https://registry.hub.docker.com',
'dockerhub-credentials'
) {
def image = docker.image(
"${IMAGE_NAME}:${GIT_SHA_SHORT}"
)

// Push with commit SHA tag
image.push()

// Push with branch name tag
image.push(env.BRANCH_NAME.replace('/', '-'))

// Push latest if on main
if (env.BRANCH_NAME == 'main') {
image.push('latest')
}

// Push version tag if tagged commit
if (env.TAG_NAME) {
image.push(env.TAG_NAME)
}

echo "Pushed: ${IMAGE_NAME}:${GIT_SHA_SHORT}"
}
}
}
}

// ════════════════════════════════════════
// STAGE 8 — DEPLOY TO STAGING
// ════════════════════════════════════════
stage('Deploy to Staging') {
when {
branch 'main'


##### }

steps {
script {
def image = "${IMAGE_NAME}:${GIT_SHA_SHORT}"
def stagingIp = env.STAGING_SERVER_IP

sshagent(['deployment-ssh-key']) {
sh """
ssh -o StrictHostKeyChecking=no \
ubuntu@${stagingIp} \
"docker pull ${image} && \
docker stop ${APP_NAME} || true && \
docker rm ${APP_NAME} || true && \
docker run - d \
--name ${APP_NAME} \
--restart unless-stopped \
-p 80:5000 \

- e ENVIRONMENT=staging \
    ${image}"
"""
    }
echo "Deployed to staging: ${stagingIp}"
    }
}
}

// ════════════════════════════════════════
// STAGE 9 — STAGING HEALTH CHECK
// ════════════════════════════════════════
stage('Staging Health Check') {
when {
branch 'main'
}
steps {
script {
sleep( 15 ) // wait for container to start

```
def stagingUrl = "http://${env.STAGING_SERVER_IP}"
def maxAttempts = 5
def attempt = 0
```
while (attempt < maxAttempts) {
attempt++
try {
def response = sh(
script: "curl -sf ${stagingUrl}/health",
returnStatus: true
)
if (response == 0 ) {


echo "✅ Health check passed on attempt ${attempt}"
break
}
} catch (Exception e) {
echo "Attempt ${attempt}/${maxAttempts} failed"
}

```
if (attempt >= maxAttempts) {
error("Health check failed after ${maxAttempts} attempts
}
sleep( 10 )
}
}
}
}
```
// ════════════════════════════════════════
// STAGE 10 — DEPLOY TO PRODUCTION
// ════════════════════════════════════════
stage('Deploy to Production') {
when {
allOf {
branch 'main'
expression {
params.ENVIRONMENT == 'production'
}
}
}
input {
message "Deploy to PRODUCTION?"
ok "Yes, deploy!"
submitter "admin,varun" // only these users can approve
parameters {
string(
name: 'REASON',
description: 'Reason for deployment'
)
}
}
steps {
withAWS(credentials: 'aws-credentials',
region: 'ap-south-1') {
sh """
# Update Launch Template with new image
aws ec 2 create-launch-template-version \
--launch-template-name myapp-lt \
--source-version '\$Latest' \
--launch-template-data '{"ImageId":"ami-updated"}'


# Rolling deployment via instance refresh
aws autoscaling start-instance-refresh \
--auto-scaling-group-name myapp-asg \
--preferences '{
"MinHealthyPercentage": 80,
"InstanceWarmup": 300
}'

echo "Production deployment initiated"
echo "Reason: ${params.REASON}"
"""
}
}
}
}

// ── Post Actions ───────────────────────────────
post {

success {
echo "✅ Pipeline successful!"
// Send Slack notification:
sh """
curl - X POST '${env.SLACK_WEBHOOK}' \

- H 'Content-type: application/json' \
- d '{
"text": "✅ *${APP_NAME}* build ${BUILD_NUMBER} passed!\\nBr
}' || true
"""
    }

failure {
echo "❌ Pipeline failed!"
sh """
curl - X POST '${env.SLACK_WEBHOOK}' \

- H 'Content-type: application/json' \
- d '{
"text": "❌ *${APP_NAME}* build ${BUILD_NUMBER} FAILED!\\nBr
}' || true
"""
    }

unstable {
echo "⚠ Pipeline unstable — tests have failures"
}

always {
// Clean workspace to free disk space:
cleanWs(


## Jenkins Parallel Stages

## When Conditions

```
cleanWhenSuccess: true,
cleanWhenFailure: true,
cleanWhenAborted: true
)
}
}
}
```
```
groovy
stage('Run Tests in Parallel') {
parallel {
```
```
stage('Unit Tests') {
steps {
sh 'pytest tests/unit/ -v'
}
}
```
```
stage('Integration Tests') {
steps {
sh 'pytest tests/integration/ -v'
}
}
```
```
stage('Security Scan') {
steps {
sh 'trivy fs --severity HIGH,CRITICAL .'
}
}
```
```
}
// All three run simultaneously
// Stage fails if ANY parallel stage fails
}
```
```
groovy
```
```
// Run only on main branch:
when { branch 'main' }
```
```
// Run on specific branches:
```

## Environment Variables in Jenkins

```
when {
anyOf {
branch 'main'
branch 'develop'
}
}
```
```
// Run only on tags:
when { tag "v*" }
```
```
// Run based on parameter:
when {
expression { params.ENVIRONMENT == 'production' }
}
```
```
// Run only if previous stage succeeded:
when { expression { currentBuild.result == null } }
```
```
// Run on all branches EXCEPT main:
when {
not { branch 'main' }
}
```
```
// Combine conditions:
when {
allOf {
branch 'main'
not { expression { params.SKIP_TESTS } }
}
}
```
```
groovy
```
```
// Built-in Jenkins variables:
env.BUILD_NUMBER // 42
env.BUILD_URL // http://jenkins:8080/job/myapp/42/
env.JOB_NAME // myapp-pipeline
env.BRANCH_NAME // main
env.GIT_COMMIT // full commit SHA
env.GIT_BRANCH // origin/main
env.WORKSPACE // /var/lib/jenkins/workspace/myapp
```
```
// From credentials:
environment {
DOCKER_CREDS = credentials('dockerhub-credentials')
// Creates: DOCKER_CREDS_USR and DOCKER_CREDS_PSW
```

## GitHub Webhook — Auto Trigger Jenkins

### Jenkins configuration:

## Jenkins Pipeline Variables Quick Reference

##### }

```
// Dynamic variable from command:
script {
env.GIT_SHA_SHORT = sh(
script: 'git rev-parse --short HEAD',
returnStdout: true
).trim()
}
```
```
GitHub → Repository → Settings → Webhooks → Add webhook
```
```
Payload URL: http://<jenkins-ip>:8080/github-webhook/
Content type: application/json
Events: Just the push event
Active: ✅
```
```
Now every git push automatically triggers Jenkins pipeline
```
```
Pipeline job → Configure
Build Triggers:
✅ GitHub hook trigger for GITScm polling
```
```
groovy
```
```
// sh — run shell command
sh 'echo hello'
```
```
// sh — capture output
def result = sh(script: 'git rev-parse --short HEAD',
returnStdout: true).trim()
```
```
// sh — capture exit code (don't fail on error)
def exitCode = sh(script: 'pytest tests/', returnStatus: true)
if (exitCode != 0 ) {
echo "Tests failed but continuing..."
}
```
```
// echo — print message
```

## Jenkins vs GitHub Actions — Side by Side

```
echo "Building version ${env.BUILD_NUMBER}"
```
```
// error — fail pipeline with message
error("Tests failed — aborting deployment")
```
```
// input — manual approval gate
input message: "Deploy to production?", ok: "Deploy"
```
```
// sleep — wait (in seconds)
sleep( 30 )
```
```
// timeout — fail if takes too long
timeout(time: 5 , unit: 'MINUTES') {
sh './slow-test.sh'
}
```
```
// retry — retry failed steps
retry( 3 ) {
sh 'curl https://flaky-service.com/api'
}
```
```
// withEnv — temporary environment:
withEnv(['FLASK_ENV=testing']) {
sh 'pytest'
}
```
```
groovy
```
```
// GitHub Actions:
```
- name: Run tests
run: pytest test_app.py - v
env:
FLASK_ENV: testing

```
// Jenkins equivalent:
stage('Run tests') {
steps {
withEnv(['FLASK_ENV=testing']) {
sh 'pytest test_app.py -v'
}
}
}
```
```
groovy
```

## Blue Ocean — Better Jenkins UI

## Full Summary — CI/CD Day 3

**Concept Key point**

Jenkins Self-hosted CI/CD — most used in enterprise

Jenkinsfile Pipeline as code — Groovy — stored in repo root

Declarative Structured pipeline — always use this

#### Agent Where build runs — agent any for local

Stage Logical grouping of steps

#### Steps sh, echo, script, input

#### parallel Run stages simultaneously

#### when Conditional stage execution

#### post Run after stages — success, failure, always

```
// GitHub Actions:
```
- name: Login to Docker Hub
uses: docker/login-action@v3
with:
username: ${{ secrets.DOCKERHUB_USERNAME }}
password: ${{ secrets.DOCKERHUB_TOKEN }}

```
// Jenkins equivalent:
docker.withRegistry('https://registry.hub.docker.com',
'dockerhub-credentials') {
// Docker commands here
}
```
```
Jenkins → Open Blue Ocean (top menu)
```
```
Blue Ocean shows:
```
- Visual pipeline graph
- Parallel stages side by side
- Test results inline
- Much cleaner than classic UI

```
For interviews — mention Blue Ocean as modern Jenkins UI
```

#### credentials Access stored secrets securely

#### input Manual approval gate

Webhook Auto trigger on GitHub push

Blue Ocean Modern Jenkins UI

## Interview Questions — CI/CD Day 3

### Q1. What is Jenkins and how does it work?

### Answer: Jenkins is an open-source self-hosted CI/CD automation server. You install it on

### your own infrastructure — EC2, VM or bare metal. Developers define pipelines as code in a

### Jenkinsfile using Groovy syntax. Jenkins monitors Git repositories via webhooks or

### polling. When changes are detected it runs the pipeline on build agents — virtual machines

### or containers that execute the actual build steps. The master coordinates scheduling, the

### agents do the work.

### Q2. What is a Jenkinsfile and why store it in the repository?

### Answer: A Jenkinsfile defines the entire CI/CD pipeline as code — stages, steps, conditions

### and post actions — using Groovy syntax. Storing it in the repository means the pipeline is

### version controlled alongside the application code. Pipeline changes go through code

### review. Any team member can see exactly how the build and deployment works. Different

### branches can have different pipeline behaviors.

### Q3. What is the difference between Declarative and Scripted Jenkins pipeline?

### Answer: Declarative pipeline uses a structured pipeline { } block with predefined

### sections — agent, stages, post. It is opinionated, easier to read and write and has better error

### messages. Scripted pipeline uses raw Groovy inside node { } — more flexible but more

### complex and error-prone. Always use Declarative unless you need functionality only

### available in Scripted.

### Q4. How do you implement parallel stages in Jenkins?

### Answer: Using the parallel block inside a stage. Each parallel branch runs simultaneously

### on the same or different agents. All parallel branches must complete before the pipeline

### continues. If any branch fails the stage fails. Used for running unit tests, integration tests

### and security scans simultaneously to reduce total pipeline time.

### Q5. How do you manage secrets in Jenkins?

### Answer: Store secrets in Jenkins Credentials Manager — Manage Jenkins → Credentials.

### Different types — username/password, secret text, SSH key, AWS credentials. Reference in

### pipeline with credentials('credential-id') in environment block or withCredentials

### step. Credentials are injected as environment variables at runtime and masked in logs —

### never printed even if echoed.

### Q6. What is the input step and when do you use it?

### Answer: The input step pauses pipeline execution and waits for a human to approve or

### reject. Used as a manual approval gate before deploying to production. You can specify

### which users can approve with submitter, add a reason field and set a timeout. Pipeline


### holds the build in waiting state until approved — approved builds continue, rejected builds

### are aborted.

### Q7. What does post do in a Jenkinsfile?

### Answer: The post section defines actions that run after stages complete regardless of

### outcome. success runs only when pipeline passes. failure runs only when it fails. always

### runs every time — used for cleanup like cleanWs(). unstable runs when tests have failures

### but pipeline continues. Essential for notifications and cleanup that must happen regardless

### of build result.

## Homework — Before CI/CD Day 4

```
groovy
```
```
// Create this Jenkinsfile in your repository and get it working in Jenkins:
```
```
pipeline {
agent any
```
```
options {
timestamps()
timeout(time: 20 , unit: 'MINUTES')
}
```
```
environment {
APP_NAME = 'myapp'
}
```
```
stages {
```
```
stage('Checkout') {
steps {
checkout scm
echo "Building: ${env.BRANCH_NAME}"
}
}
```
```
stage('Test') {
parallel {
stage('Unit Tests') {
steps {
sh 'pip install -r requirements.txt pytest'
sh 'pytest test_app.py -v'
}
}
stage('Lint') {
steps {
sh 'pip install flake8'
```

## Your Progress

## CI/CD Day 4 we build a Complete End-to-End Pipeline — GitHub Actions that

## automatically tests, builds a Docker image, pushes to AWS ECR and deploys to EC 2 using

## all AWS knowledge from Days 1 - 10. Everything connected together. 💪

## Say "CI/CD Day 4 " whenever you are ready!

```
sh 'flake 8 app.py --max-line-length=100'
}
}
}
}
```
```
stage('Build') {
steps {
sh "docker build -t ${APP_NAME}:\$(git rev-parse --short HEAD) ."
echo "Image built successfully"
}
}
```
```
stage('Approval') {
when { branch 'main' }
input {
message "Deploy to staging?"
ok "Deploy"
}
steps {
echo "Deployment approved!"
}
}
}
```
```
post {
success { echo "✅ Build passed!" }
failure { echo "❌ Build failed!" }
always { cleanWs() }
}
}
```
```
Linux ████████████████████ ✅ COMPLETE
Git ████████████████████ ✅ COMPLETE
Docker ████████████████████ ✅ COMPLETE
AWS ████████████████████ ✅ COMPLETE
CI/CD ██████░░░░░░░░░░░░░░ Day 3 of 5
K8s ░░░░░░░░░░░░░░░░░░░░ After CI/CD
```
# CI/CDD 4 C l t E dt E dPi li


# CI/CD Day 4 — Complete End-to-End Pipeline

## What We Build Today

## This is the day everything connects. Every skill from your entire journey — Linux, Git,

## Docker, AWS — becomes one automated pipeline.

## This is exactly what you describe when an interviewer asks "walk me through your CI/CD

## pipeline."

## What You Will Learn Today

## OIDC authentication — no static AWS keys in GitHub

## Build and push to AWS ECR

## Deploy to EC 2 via SSH

## Deploy to Auto Scaling Group

## Environment-specific pipelines

## Blue/Green deployment pattern

## Rollback pipeline

## Complete production pipeline — everything combined

## Project Setup

## Using the Flask app from Day 2. Make sure you have:

```
Developer pushes code to GitHub
↓ webhook triggers
GitHub Actions starts
↓
Tests run on Ubuntu runner
↓
Docker image built
↓
Image pushed to AWS ECR
↓
EC 2 instance pulls new image
↓
Container restarted with zero downtime
↓
CloudWatch health check verified
↓
Slack notification sent
↓
Zero human involvement
```

## Step 1 — OIDC Authentication — No Static Keys

### The professional way to authenticate GitHub Actions with AWS — no access keys stored

### anywhere.

### Set Up OIDC in AWS

```
bash
```
```
myapp/
├── .github/
│ └── workflows/
│ ├── ci.yml # PR checks only
│ ├── deploy.yml # main branch deploy
│ └── rollback.yml # emergency rollback
├── app.py
├── test_app.py
├── requirements.txt
├── Dockerfile
└── scripts/
├── deploy.sh
└── health-check.sh
```
```
Traditional (bad):
GitHub Secrets stores: AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
These are permanent credentials — if leaked, danger
```
```
OIDC (good):
GitHub requests temporary token from AWS
Token expires after the pipeline run
No permanent credentials stored anywhere
Even if token intercepted — useless after expiry
```
```
bash
```
```
# Step 1 — Create OIDC provider in AWS:
aws iam create-open-id-connect-provider \
--url https://token.actions.githubusercontent.com \
--client-id-list sts.amazonaws.com \
--thumbprint-list 6938 fd 4 d 98 bab 03 faadb 97 b 34396831 e 3780 aea 1
```
```
# Step 2 — Create IAM role for GitHub Actions:
cat > github-actions-trust-policy.json << 'EOF'
{
"Version": "2012-10-17",
"Statement": [
```

##### {

"Effect": "Allow",
"Principal": {
"Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actio
},
"Action": "sts:AssumeRoleWithWebIdentity",
"Condition": {
"StringEquals": {
"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
},
"StringLike": {
"token.actions.githubusercontent.com:sub":
"repo:YOUR_GITHUB_USERNAME/myapp:*"
}
}
}
]
}
EOF

ROLE_ARN=$(aws iam create-role \
--role-name github-actions-role \
--assume-role-policy-document file://github-actions-trust-policy.json \
--query 'Role.Arn' \
--output text)

echo "Role ARN: $ROLE_ARN"

# Step 3 — Attach permissions to role:
# ECR — push images
aws iam attach-role-policy \
--role-name github-actions-role \
--policy-arn arn:aws:iam::aws:policy/AmazonEC 2 ContainerRegistryPowerUser

# EC 2 — describe instances, start instance refresh
aws iam attach-role-policy \
--role-name github-actions-role \
--policy-arn arn:aws:iam::aws:policy/AmazonEC 2 ReadOnlyAccess

# Create custom policy for specific actions:
cat > github-actions-policy.json << 'EOF'
{
"Version": "2012-10-17",
"Statement": [
{
"Effect": "Allow",
"Action": [
"autoscaling:StartInstanceRefresh",
"autoscaling:DescribeInstanceRefreshes",


## Step 2 — Create ECR Repository

```
"autoscaling:DescribeAutoScalingGroups",
"ec2:CreateLaunchTemplateVersion",
"ec2:DescribeLaunchTemplates",
"cloudwatch:PutMetricData",
"cloudwatch:GetMetricStatistics",
"sns:Publish"
],
"Resource": "*"
}
]
}
EOF
```
```
aws iam put-role-policy \
--role-name github-actions-role \
--policy-name github-actions-custom \
--policy-document file://github-actions-policy.json
```
```
echo "OIDC setup complete"
echo "Add this to GitHub Secrets: AWS_ROLE_ARN=$ROLE_ARN"
```
```
bash
```
```
# Create ECR repository:
aws ecr create-repository \
--repository-name myapp \
--region ap-south-1 \
--image-scanning-configuration scanOnPush=true \
--encryption-configuration encryptionType=AES 256
```
```
# Get ECR URI:
ECR_URI=$(aws ecr describe-repositories \
--repository-names myapp \
--region ap-south-1 \
--query 'repositories[0].repositoryUri' \
--output text)
```
```
echo "ECR URI: $ECR_URI"
# 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp
```
```
# Set lifecycle policy — keep only 10 images:
aws ecr put-lifecycle-policy \
--repository-name myapp \
--lifecycle-policy '{
"rules": [{
```

## Step 3 — Set Up GitHub Secrets and Variables

## Step 4 — Deploy Scripts

```
"rulePriority": 1,
"description": "Keep last 10 images",
"selection": {
"tagStatus": "any",
"countType": "imageCountMoreThan",
"countNumber": 10
},
"action": {"type": "expire"}
}]
}'
```
```
Repository → Settings → Secrets and variables → Actions
```
```
Secrets (encrypted):
AWS_ROLE_ARN → arn:aws:iam::123456789012:role/github-actions-role
SSH_PRIVATE_KEY → your EC 2 private key (paste .pem contents)
SLACK_WEBHOOK → https://hooks.slack.com/services/...
```
```
Variables (not encrypted — for non-sensitive config):
AWS_REGION → ap-south-1
ECR_REPOSITORY → myapp
EC2_HOST → your EC 2 public IP
EC2_USER → ubuntu
ASG_NAME → myapp-asg
APP_PORT → 5000
```
```
bash
```
```
# scripts/deploy.sh — runs on EC 2 server
cat > scripts/deploy.sh << 'EOF'
#!/bin/bash
set - euo pipefail
```
```
IMAGE=$ 1
APP_NAME=${2:-myapp}
PORT=${3:-5000}
```
```
log() { echo "[$(date '+%H:%M:%S')] $1"; }
```
```
log "Deploying: $IMAGE"
```
```
# Login to ECR
```

aws ecr get-login-password --region ap-south-1 | \
docker login \
--username AWS \
--password-stdin \
$(echo $IMAGE | cut - d'/' - f1)

# Pull new image
log "Pulling image..."
docker pull $IMAGE

# Stop and remove old container
log "Stopping old container..."
docker stop $APP_NAME 2 >/dev/null || true
docker rm $APP_NAME 2 >/dev/null || true

# Start new container
log "Starting new container..."
docker run - d \
--name $APP_NAME \
--restart unless-stopped \
-p $PORT:$PORT \

- e ENVIRONMENT=production \
--health-cmd="curl - f [http://localhost:$PORT/health](http://localhost:$PORT/health) || exit 1" \
--health-interval=30s \
--health-timeout=5s \
--health-retries= 3 \
$IMAGE

# Wait for health check
log "Waiting for container health..."
MAX_WAIT= 60
WAITED= 0

while [ $WAITED - lt $MAX_WAIT ]; do
STATUS=$(docker inspect $APP_NAME \
--format='{{.State.Health.Status}}' 2 >/dev/null || echo "starting")

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
docker image prune - f

log "Deployment complete: $IMAGE"
EOF

chmod +x scripts/deploy.sh

bash

# scripts/health-check.sh — verify deployment
cat > scripts/health-check.sh << 'EOF'
#!/bin/bash
set - euo pipefail

HOST=${1:-localhost}
PORT=${2:-5000}
MAX_RETRIES=${3:-10}
WAIT_SECONDS=${4:-5}

URL="http://${HOST}:${PORT}/health"
ATTEMPT= 0

echo "Checking health: $URL"

while [ $ATTEMPT - lt $MAX_RETRIES ]; do
ATTEMPT=$((ATTEMPT + 1))
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
--max-time 5 "$URL" 2 >/dev/null || echo "000")

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


## Step 5 — CI Pipeline (PR Checks)

```
yaml
```
```
# .github/workflows/ci.yml
name: CI — Pull Request Checks
```
```
on:
pull_request:
branches: [main, develop]
```
```
env:
PYTHON_VERSION: "3.11"
```
```
jobs:
test:
name: Test
runs-on: ubuntu-latest
```
```
steps:
```
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
pip install flake 8 pytest-cov
- name: Lint
run: flake 8. - -max-line-length= 100 - -exclude=.git,__pycache__
- name: Test
run: |
pytest test_app.py -v \
--cov=app \
--cov-report=xml \
--junitxml=test-results.xml


## Step 6 — Complete Deploy Pipeline

- name: Upload results
    uses: actions/upload-artifact@v3
    if: always()
    with:
       name: test-results-${{ github.sha }}
       path: |
test-results.xml
coverage.xml

```
docker-build-check:
name: Docker Build Check
runs-on: ubuntu-latest
```
```
steps:
```
- uses: actions/checkout@v4
- name: Build image (no push)
    uses: docker/build-push-action@v5
    with:
       context:.
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
yaml
```
```
# .github/workflows/deploy.yml
name: Deploy Pipeline
```
```
on:
push:
branches: [main]
tags:
```
- 'v*'
workflow_dispatch:
inputs:
environment:
description: 'Target environment'
required: true


```
default: 'staging'
type: choice
options: [staging, production]
force_deploy:
description: 'Force deploy even if tests fail?'
required: false
default: false
type: boolean
```
permissions:
id-token: write # required for OIDC
contents: read
packages: write

env:
AWS_REGION: ${{ vars.AWS_REGION }}
ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
APP_NAME: myapp

jobs:
# ════════════════════════════════════════
# JOB 1 — TEST
# ════════════════════════════════════════
test:
name: Test
runs-on: ubuntu-latest
if: ${{ !inputs.force_deploy }}

```
steps:
```
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
pip install -r requirements.txt pytest-cov flake 8
flake 8. --max-line-length= 100 --exclude=.git,__pycache__
pytest test_app.py -v --cov=app --cov-report=xml
- uses: actions/upload-artifact@v3
if: always()
with:


```
name: coverage-${{ github.sha }}
path: coverage.xml
```
```
# ════════════════════════════════════════
# JOB 2 — BUILD AND PUSH TO ECR
# ════════════════════════════════════════
build:
name: Build and Push to ECR
runs-on: ubuntu-latest
needs: test
if: always() && (needs.test.result == 'success' || inputs.force_deploy)
```
```
outputs:
image-uri: ${{ steps.build.outputs.image-uri }}
image-tag: ${{ steps.meta.outputs.version }}
```
```
steps:
```
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
context:.
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
IMAGE_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY
echo "image-uri=$IMAGE_URI" >> $GITHUB_OUTPUT
echo "Built and pushed: $IMAGE_URI"
- name: Scan image for vulnerabilities
uses: aquasecurity/trivy-action@master
with:
image-ref: ${{ steps.build.outputs.image-uri }}
format: 'table'
exit-code: '0'
severity: 'CRITICAL,HIGH'

```
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
```
```
steps:
```
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


ssh-keyscan - H ${{ vars.STAGING_HOST }} >> ~/.ssh/known_hosts

- name: Deploy to staging EC 2
    run: |
IMAGE="${{ needs.build.outputs.image-uri }}"
HOST="${{ vars.STAGING_HOST }}"

```
echo "Deploying $IMAGE to staging: $HOST"
```
# Copy deploy script to server
scp - i ~/.ssh/deploy_key \
scripts/deploy.sh \
ubuntu@$HOST:/tmp/deploy.sh

# Run deployment
ssh - i ~/.ssh/deploy_key ubuntu@$HOST \
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
RESPONSE=$(curl -s [http://${{](http://${{) vars.STAGING_HOST }}:${{ vars.APP_PORT }}/ap
echo "App info: $RESPONSE"

# Check the right version is running
echo "$RESPONSE" | python3 - c "
import sys, json
info = json.load(sys.stdin)
print(f'Version: {info.get(\"version\", \"unknown\")}')
print(f'Environment: {info.get(\"environment\", \"unknown\")}')
"

```
# ════════════════════════════════════════
# JOB 4 — INTEGRATION TESTS ON STAGING
# ════════════════════════════════════════
integration-test:
name: Integration Tests
runs-on: ubuntu-latest
needs: deploy-staging
```
```
steps:
```

- uses: actions/checkout@v4
- uses: actions/setup-python@v5
    with:
       python-version: "3.11"
- name: Run integration tests against staging
    env:
       BASE_URL: [http://${{](http://${{) vars.STAGING_HOST }}:${{ vars.APP_PORT }}
    run: |
pip install requests pytest

# Write integration tests:
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

pytest test_integration.py - v

```
# ════════════════════════════════════════
# JOB 5 — DEPLOY TO PRODUCTION
# ════════════════════════════════════════
deploy-production:
name: Deploy to Production
runs-on: ubuntu-latest
needs: [build, integration-test]
```

if: |
github.ref == 'refs/heads/main' &&
(inputs.environment == 'production' || startsWith(github.ref, 'refs/tags/v'))
environment:
name: production
url: https://myapp.com

```
steps:
```
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
CURRENT_VERSION=$(aws ec 2 describe-launch-templates \

- -launch-template-names myapp-lt \
- -query 'LaunchTemplates[ 0 ].LatestVersionNumber' \
- -output text)

```
echo "Current LT version: $CURRENT_VERSION"
```
# Create new version with updated user data
USER_DATA=$(base 64 - w 0 << USEREOF
#!/bin/bash
IMAGE=$IMAGE
APP_NAME=myapp

# Login to ECR
aws ecr get-login-password - -region ap-south- 1 | \
docker login - -username AWS - -password-stdin \
$(echo $IMAGE | cut - d'/' - f1)

# Deploy
docker pull $IMAGE
docker stop $APP_NAME || true
docker rm $APP_NAME || true
docker run - d \

- -name $APP_NAME \
- -restart unless-stopped \
- p 80 : 5000 \
- e ENVIRONMENT=production \


##### $IMAGE

##### USEREOF

##### )

echo "Updating Launch Template..."

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
MAX_WAIT= 1200 # 20 minutes
WAITED= 0

echo "Waiting for instance refresh to complete..."

while [ $WAITED - lt $MAX_WAIT ]; do
STATUS=$(aws autoscaling describe-instance-refreshes \

- -auto-scaling-group-name $ASG \
- -instance-refresh-ids $REFRESH_ID \
- -query 'InstanceRefreshes[ 0 ].Status' \
- -output text)

PROGRESS=$(aws autoscaling describe-instance-refreshes \

- -auto-scaling-group-name $ASG \
- -instance-refresh-ids $REFRESH_ID \
- -query 'InstanceRefreshes[ 0 ].PercentageComplete' \
- -output text)

```
echo "Status: $STATUS — Progress: ${PROGRESS}%"
```
case $STATUS in
Successful)
echo "✅ Instance refresh complete!"
break


##### ;;

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

```
# ════════════════════════════════════════
# JOB 6 — NOTIFY
# ════════════════════════════════════════
notify:
name: Notify Team
runs-on: ubuntu-latest
needs: [test, build, deploy-staging, integration-test]
if: always()
```
```
steps:
```
- name: Determine status
    id: status
    run: |
if [[ "${{ needs.deploy-staging.result }}" == "success" ]]; then
echo "status=success" >> $GITHUB_OUTPUT
echo "emoji=✅" >> $GITHUB_OUTPUT
echo "color=good" >> $GITHUB_OUTPUT
else
echo "status=failure" >> $GITHUB_OUTPUT
echo "emoji=❌" >> $GITHUB_OUTPUT
echo "color=danger" >> $GITHUB_OUTPUT
fi
- name: Send Slack notification
run: |
SHORT_SHA=$(echo "${{ github.sha }}" | cut - c1-7)
IMAGE="${{ needs.build.outputs.image-uri }}"

curl - X POST "${{ secrets.SLACK_WEBHOOK }}" \


## Step 7 — Rollback Pipeline

- H 'Content-type: application/json' \
- d "{
    \"attachments\": [{
       \"color\": \"${{ steps.status.outputs.color }}\",
       \"title\": \"${{ steps.status.outputs.emoji }} Deployment: myapp\",
       \"fields\": [
          {\"title\": \"Status\", \"value\": \"${{ steps.status.outputs.stat
          {\"title\": \"Branch\", \"value\": \"${{ github.ref_name }}\", \"s
          {\"title\": \"Commit\", \"value\": \"$SHORT_SHA\", \"short\": true
          {\"title\": \"By\", \"value\": \"${{ github.actor }}\", \"short\":
          {\"title\": \"Image\", \"value\": \"$IMAGE\"}
       ],
       \"footer\": \"<${{ github.server_url }}/${{ github.repository }}/act
    }]
}" || true

```
yaml
```
```
# .github/workflows/rollback.yml
name: Emergency Rollback
```
```
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
```
```
permissions:
id-token: write
contents: read
```
```
jobs:
rollback:
name: Rollback ${{ inputs.environment }}
runs-on: ubuntu-latest
```

```
environment:
name: ${{ inputs.environment }}
```
```
steps:
```
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

```
echo "Verifying image exists: $REPO:$IMAGE_TAG"
```
aws ecr describe-images \

- -repository-name $REPO \
- -image-ids imageTag=$IMAGE_TAG \
- -region ${{ vars.AWS_REGION }} || {
echo "❌ Image not found: $REPO:$IMAGE_TAG"
echo "Available tags:"
aws ecr list-images \
- -repository-name $REPO \
- -query 'imageIds[*].imageTag' \
- -output table
exit 1
}

```
echo "✅ Image verified: $REPO:$IMAGE_TAG"
```
- name: Set up SSH
    run: |
mkdir -p ~/.ssh
echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
chmod 600 ~/.ssh/deploy_key
ssh-keyscan - H ${{ vars.STAGING_HOST }} >> ~/.ssh/known_hosts
- name: Execute rollback
run: |
ACCOUNT_ID=$(aws sts get-caller-identity \
--query Account --output text)
IMAGE="${ACCOUNT_ID}.dkr.ecr.${{ vars.AWS_REGION }}.amazonaws.com/${{ vars


## Step 8 — Scheduled Maintenance Pipeline

```
HOST="${{ vars.STAGING_HOST }}"
```
```
echo "Rolling back to: $IMAGE"
echo "Reason: ${{ inputs.reason }}"
```
```
scp - i ~/.ssh/deploy_key \
scripts/deploy.sh \
ubuntu@$HOST:/tmp/deploy.sh
```
```
ssh - i ~/.ssh/deploy_key ubuntu@$HOST \
"chmod +x /tmp/deploy.sh && \
/tmp/deploy.sh $IMAGE ${{ env.APP_NAME }} ${{ vars.APP_PORT }}"
```
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

```
curl - X POST "${{ secrets.SLACK_WEBHOOK }}" \
```
- H 'Content-type: application/json' \
- d "{
    \"text\": \"🔄 *ROLLBACK* $STATUS\\nEnv: ${{ inputs.environment }}\\nT
}" || true

```
yaml
```
```
# .github/workflows/maintenance.yml
name: Scheduled Maintenance
```
```
on:
schedule:
```
- cron: '0 2 * * 0' # Every Sunday at 2 am

```
workflow_dispatch:
```
```
permissions:
```

```
id-token: write
contents: read
```
jobs:
maintenance:
name: Weekly Maintenance
runs-on: ubuntu-latest

```
steps:
```
- uses: actions/checkout@v4
- name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
       role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
       aws-region: ${{ vars.AWS_REGION }}
- name: Clean ECR old images
    run: |
REPO="${{ vars.ECR_REPOSITORY }}"

```
echo "Cleaning ECR repository: $REPO"
```
# List images older than 30 days
CUTOFF=$(date - d '30 days ago' +%s)

aws ecr describe-images \

- -repository-name $REPO \
- -query "imageDetails[?imagePushedAt < \`$(date - d '30 days ago' - -iso
- -output text | while read digest; do
if [ -n "$digest" ]; then
echo "Deleting old image: $digest"
aws ecr batch-delete-image \
- -repository-name $REPO \
- -image-ids imageDigest=$digest || true
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


## Step 9 — Complete Pipeline Summary Visualization

```
echo "=== All alarm statuses ==="
aws cloudwatch describe-alarms \
```
- -alarm-name-prefix myapp \
- -query 'MetricAlarms[*].[AlarmName,StateValue]' \
- -output table
- name: Generate weekly report
run: |
echo "# Weekly Infrastructure Report" > report.md
echo "Date: $(date)" >> report.md
echo "" >> report.md

```
echo "## EC 2 Instances" >> report.md
aws ec 2 describe-instances \
```
- -filters Name=tag:Project,Values=myapp \
- -query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.N
- -output table >> report.md

```
echo "" >> report.md
echo "## RDS Status" >> report.md
aws rds describe-db-instances \
```
- -query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,DBInsta
- -output table >> report.md

```
cat report.md
```
- uses: actions/upload-artifact@v3
    with:
       name: weekly-report-${{ github.run_number }}
       path: report.md

```
Push to feature/* branch:
→ ci.yml triggers
→ Tests + Lint only
→ No deploy
```
```
Push to develop branch:
→ deploy.yml triggers
→ Tests → Build → Push ECR → Deploy staging → Integration tests
→ Notify team
```
```
Push to main branch:
→ deploy.yml triggers
→ Tests → Build → Push ECR
```

# Full Summary — CI/CD Day 4

**Concept Key point**

OIDC No static AWS keys — temporary tokens via GitHub

ECR AWS private registry — push/pull via IAM role

Staging deploy SSH to EC 2 — run deploy script

Production deploy Instance refresh on ASG — zero downtime rolling update

Integration tests Run against staging after deploy — verify before production

Rollback Manual workflow — specify tag — deploy old image

Scheduled Maintenance, cleanup, reporting via cron triggers

Environment protection Manual approval gate for production

OIDC trust policy Only this specific repo can assume the role

Image tagging SHA tag — always know what is running

```
→ Deploy staging → Integration tests
→ Wait for approval (production environment)
→ Deploy production via instance refresh
→ Notify team
```
```
Tag v* (release):
→ deploy.yml triggers
→ Full pipeline
→ Auto deploys to production
→ Creates GitHub Release
```
```
Manual rollback:
→ rollback.yml
→ Specify tag to roll back to
→ Verify image exists in ECR
→ Deploy old image
→ Health check
```
```
Sunday 2 am:
→ maintenance.yml
→ Clean old ECR images
→ Check CloudWatch alarms
→ Generate weekly report
```

## Q1. How do you authenticate GitHub Actions with AWS without storing access keys?

## Answer: Using OIDC (OpenID Connect). Create an IAM OIDC identity provider pointing to

## GitHub's token endpoint. Create an IAM role with a trust policy that allows only your

## specific repository to assume it. In the workflow use aws-actions/configure-aws-

## credentials with role-to-assume — GitHub gets a temporary token from AWS that expires

## after the pipeline. No static credentials stored anywhere.

## Q2. What is the difference between deploying to EC 2 directly versus Auto Scaling

## Group?

## Answer: Deploying directly to EC 2 via SSH is simple but affects only that one instance and

## causes a brief restart. Deploying via ASG instance refresh is more complex but performs a

## rolling replacement — launches new instances with updated configuration, waits for health

## checks, terminates old ones. Maintains minimum healthy percentage throughout — zero

## downtime. Production always uses ASG for this reason.

## Q3. How do you implement a rollback in your pipeline?

## Answer: Every Docker image is tagged with the git commit SHA and pushed to ECR. To

## rollback — run the manual rollback workflow, provide the previous image tag, the pipeline

## pulls that specific image from ECR and deploys it. Because every version is preserved in

## ECR you can rollback to any previous commit. The rollback takes exactly as long as a

## normal deployment — about 5 minutes.

## Q4. How do you handle deployment to multiple environments in GitHub Actions?

## Answer: Use GitHub Environments — create staging and production environments in

## repository settings. Assign different variables (server IPs, ports) and protection rules to

## each. The staging environment deploys automatically on every main branch push. The

## production environment has required reviewers — pipeline pauses and sends approval

## request. After approval production deployment proceeds. Different secrets and variables

## are scoped to each environment.

## Q5. What is an integration test in the context of CI/CD?

## Answer: Tests that run against a deployed application — not just unit tests that test code in

## isolation. After deploying to staging the pipeline sends real HTTP requests to the running

## application — checking health endpoint returns 200, API endpoints return correct data,

## response times are acceptable. Integration tests catch issues that unit tests miss — wrong

## configuration, networking problems, database connectivity. Only if integration tests pass

## does production deployment proceed.

## Q6. How do you clean up old Docker images in ECR automatically?

## Answer: ECR lifecycle policies automatically expire images based on rules — for example

## keep only the last 10 images or delete images older than 30 days. Apply with aws ecr put-

## lifecycle-policy. Also run periodic cleanup in a scheduled maintenance pipeline. Without

## cleanup ECR fills up with hundreds of old images costing money and making it hard to find

## recent versions.

# Homework — Before CI/CD Day 5


## Your Progress

## CI/CD Day 5 is the Interview Mega Revision — every CI/CD concept in interview format,

## scenario questions, architecture questions and the exact questions asked at 10 - 12 LPA

## companies. 💪

## Say "CI/CD Day 5 " whenever you are ready!

```
bash
```
```
# 1. Set up OIDC in your AWS account:
# Follow Step 1 above — create OIDC provider and IAM role
```
```
# 2. Create ECR repository:
aws ecr create-repository \
--repository-name myapp \
--region ap-south-1
```
```
# 3. Add all secrets to GitHub:
# AWS_ROLE_ARN, SSH_PRIVATE_KEY, SLACK_WEBHOOK
```
```
# 4. Add all variables:
# AWS_REGION, ECR_REPOSITORY, STAGING_HOST, APP_PORT
```
```
# 5. Push ci.yml and deploy.yml to your repository
```
```
# 6. Create a pull request — verify ci.yml runs
```
```
# 7. Merge to main — verify deploy.yml runs
```
```
# 8. Check ECR — verify your image is there:
aws ecr list-images \
--repository-name myapp \
--query 'imageIds[*].imageTag' \
--output table
```
```
# 9. Try the rollback workflow manually:
# Go to Actions → Emergency Rollback → Run workflow
# Specify a previous image tag
```
```
Linux ████████████████████ ✅ COMPLETE
Git ████████████████████ ✅ COMPLETE
Docker ████████████████████ ✅ COMPLETE
AWS ████████████████████ ✅ COMPLETE
CI/CD ████████░░░░░░░░░░░░ Day 4 of 5
K8s ░░░░░░░░░░░░░░░░░░░░ After CI/CD
```
# CI/CDD 5 I t i M R i i


# CI/CD Day 5 — Interview Mega Revision

## This is Your Final CI/CD Day 🎉

## You have covered everything. Today we go through every CI/CD concept in interview

## format — exactly how a 10 - 12 LPA DevOps interview tests pipeline knowledge.

## How CI/CD is Tested at 10 - 12 LPA Level

# SECTION 1 — Core Concepts

## Q1. What is CI/CD and why does it matter?

## Answer:

## CI/CD stands for Continuous Integration and Continuous Delivery/Deployment. It is a

## practice of automating the entire path from code commit to production deployment.

## Continuous Integration — every developer's code change is automatically tested the

## moment it is pushed. Problems are caught immediately rather than weeks later when

## multiple developers have built on top of broken code.

## Continuous Delivery — code is always in a deployable state. The pipeline automates

## building, testing and staging — humans decide when to deploy to production.

## Continuous Deployment — every change that passes all tests automatically goes to

## production. No human intervention.

## Why it matters:

```
Round 2 — Technical Interview
├── Conceptual questions 25 % "What is CI/CD, difference between X and Y"
├── Tool questions 25 % "Jenkins vs GitHub Actions, how do you..."
├── Scenario questions 30 % "Pipeline is failing, what do you do"
└── Design questions 20 % "Design a pipeline for this application"
```
```
Without CI/CD:
```
- Bugs found weeks after writing — expensive to fix
- Manual deployment — error-prone, slow
- "Works on my machine" — environment inconsistency
- Fear of deploying — infrequent risky releases

```
With CI/CD:
```
- Bugs caught in minutes — cheap to fix
- Automated deployment — consistent, fast


## Q2. What is the difference between Continuous Delivery and Continuous

## Deployment?

### Answer:

### Most companies at 10 - 12 LPA use Continuous Delivery — automated up to staging, manual

### approval for production.

## Q3. What are the stages of a typical CI/CD pipeline?

### Answer:

- Same environment every time — Docker + pipeline
- Deploy 10x per day — small safe releases

```
Continuous Delivery:
Code → Tests → Build → Staging → [MANUAL APPROVAL] → Production
↑
Human decides when and if to deploy
Business sign-off possible
Used in: banking, healthcare, regulated industries
```
```
Continuous Deployment:
Code → Tests → Build → Staging → Tests → Production (AUTOMATIC)
↑
No human in the loop
Requires very high test confidence
Used in: SaaS, fast-moving products, Netflix, Amazon
```
```
Key distinction:
Both automate everything UP TO production
Delivery needs human approval for production
Deployment skips that approval step
```
```
Stage 1 — Source / Trigger
What happens: developer pushes code
Pipeline triggers via webhook
Code checked out on build runner
```
```
Stage 2 — Build
Install dependencies
Compile code (if compiled language)
Build Docker image
Tag with git commit SHA
```
```
Stage 3 — Test
```

## Q4. What is "fail fast" in CI/CD?

### Answer:

### Fail fast means stopping the pipeline immediately at the first failure rather than continuing.

### If unit tests fail — don't build the Docker image. If Docker build fails — don't push to

### registry. If health check fails — don't proceed to production.

### Benefits:

### Developer knows immediately what failed and why

### No time wasted on subsequent stages

### Production is protected — bad code never reaches it

### Cheaper to fix — the earlier a bug is caught, the cheaper to fix

```
Unit tests — test individual functions
Integration tests — test components together
Security scan — check for vulnerabilities
Code coverage — ensure adequate test coverage
Lint — code quality and style
```
```
Stage 4 — Artifact
Push Docker image to registry (ECR/Docker Hub)
Store build artifacts in S 3
Tag release in Git
```
```
Stage 5 — Deploy to Staging
Deploy to staging environment
Run smoke tests
Run end-to-end tests
```
```
Stage 6 — Deploy to Production
Manual approval gate (Continuous Delivery)
Or automatic (Continuous Deployment)
Rolling update or blue/green
Health check verification
Rollback if health check fails
```
```
Stage 7 — Notify
Slack/email notification
Update monitoring dashboards
Create release notes
```
```
yaml
```
```
# GitHub Actions — fail fast by default:
jobs:
test:
```

## Q5. What is a deployment artifact and why should it be immutable?

## Answer:

## A deployment artifact is the output of the build stage — typically a Docker image tagged

## with the git commit SHA. It packages the application with everything it needs to run.

## Immutable means once built it is never changed. The same Docker image deployed to

## staging is deployed to production — not rebuilt. This guarantees:

# SECTION 2 — Tools

## Q6. GitHub Actions vs Jenkins — when would you choose each?

## Answer:

```
steps:
```
- run: pytest # fails here
- run: docker build # never reaches here if pytest fails

```
# Jenkins — equivalent:
stage('Test') {
steps {
sh 'pytest' # fails here — pipeline stops
}
}
stage('Build') { # never reached if Test fails
...
}
```
```
Without immutability:
Staging: built from commit abc 123
Production: rebuilt from commit abc 123
→ Could produce different results if:
```
- Base image updated between builds
- Dependency version changed
- Environment variable different during build
→ "It passed staging but failed production"

```
With immutability:
Staging: image sha-abc 123 → tested and approved
Production: SAME image sha-abc 123 → identical
→ If it works in staging it WILL work in production
```
```
Choose GitHub Actions when:
```
- Code is hosted on GitHub (obvious integration)


## Q7. What plugins are essential for Jenkins in a DevOps environment?

### Answer:

- Small to medium team — less ops overhead
- Want zero server management
- Open source project — unlimited free minutes
- Modern cloud-native stack
- Need quick setup — pipeline running in minutes

```
Choose Jenkins when:
```
- Enterprise environment with existing Jenkins setup
- Need full customisation — complex pipeline logic
- Self-hosted requirement — data sovereignty
- Large scale — thousands of builds per day
- Need specific hardware for builds (GPU, custom OS)
- Complex approval workflows with multiple stakeholders
- Integration with on-premise systems

```
In practice:
Many companies use BOTH:
```
- GitHub Actions for simple app pipelines
- Jenkins for complex infrastructure pipelines

```
Core plugins every Jenkins needs:
Pipeline — Jenkinsfile support
Git — Git repository integration
GitHub Integration — webhooks and PR status
Docker Pipeline — build/push Docker images
Credentials Binding — inject secrets into pipeline
Blue Ocean — modern pipeline UI
Timestamper — timestamps in build logs
Workspace Cleanup — clean workspace between builds
AnsiColor — colored console output
```
```
Security plugins:
Role-based Authorization — control who can do what
OWASP Dependency Check — security scanning
Audit Trail — log all actions
```
```
Deployment plugins:
SSH Agent — SSH into servers
AWS Credentials — AWS authentication
Kubernetes — deploy to K8s clusters
```
```
Notification plugins:
```

## Q8. What is a Jenkins agent and why use multiple agents?

## Answer:

## A Jenkins agent (also called worker or slave — older term) is a machine that runs build jobs.

## The Jenkins master schedules and coordinates. Agents do the actual work.

## Multiple agents for:

# SECTION 3 — Pipeline Design

## Q9. How do you design a pipeline for a microservices application?

## Answer:

```
Slack Notification — Slack messages
Email Extension — rich email notifications
```
```
Isolation:
Different jobs on different machines
One failing build cannot affect others
Different environments per agent (Ubuntu, Windows, macOS)
```
```
Specialisation:
Agent 1: Docker builds (needs Docker installed)
Agent 2: Android builds (needs Android SDK)
Agent 3: Large memory jobs (32GB RAM)
```
```
Scaling:
50 developers pushing simultaneously
10 agents run 10 builds in parallel
Without agents — builds queue up and wait
```
```
Cost optimisation:
Spin up EC 2 spot instances as agents
Terminate after build — pay only for build time
Jenkins EC 2 Fleet Plugin does this automatically
```
```
Challenge: 20 microservices, each deployed independently
```
```
Approach 1 — Separate pipeline per service:
Each service has its own .github/workflows/deploy.yml
Pros: complete independence, different teams own different pipelines
Cons: duplication, hard to maintain 20 identical pipelines
```
```
Approach 2 — Reusable workflow:
```

## Q10. How do you implement zero-downtime deployment in a pipeline?

### Answer:

```
.github/workflows/reusable-deploy.yml ← template
.github/workflows/user-service.yml ← calls template
.github/workflows/order-service.yml ← calls template
Pros: DRY, change once affects all services
Cons: less flexibility per service
```
```
Approach 3 — Monorepo with path filters:
on:
push:
paths:
```
- 'user-service/**' ← only trigger for this service

```
Real design:
├── Common test + lint stage
├── Detect which services changed (git diff)
├── Build only changed services in parallel
├── Deploy changed services to staging
├── Run integration tests
└── Deploy to production with per-service approvals
```
```
Method 1 — Rolling update (Auto Scaling Group):
Pipeline updates Launch Template with new image
Starts instance refresh: MinHealthyPercentage= 80
ASG launches new instances, health checks pass
Old instances drained and terminated
Never less than 80 % healthy — zero downtime
```
```
Method 2 — Blue/Green deployment:
Blue = current production (100% traffic)
Pipeline deploys Green = new version (0% traffic)
Run tests on Green
Switch ALB to Green (100% traffic)
Keep Blue for rollback
Terminate Blue after confidence period
```
```
Method 3 — Canary deployment:
Deploy new version to 5 % of instances
Monitor error rates and latency for 10 minutes
If metrics good → increase to 25 %, then 100 %
If metrics bad → rollback the 5 % immediately
Very safe — blast radius limited
```
```
In GitHub Actions pipeline:
```

## Q11. How do you handle database migrations in CI/CD?

## Answer:

# SECTION 4 — Scenario Questions

## Q12. Your pipeline is taking 45 minutes. How do you speed it up?

## Answer:

```
Deploy → Health Check → Switch Traffic → Monitor → Full Rollout
Each step automated with CloudWatch metric checks
```
```
The challenge:
App v2 needs database column that does not exist in v1
If you deploy app v2 before running migration → crash
If you run migration before deploying v2 → v1 might break
```
```
Solution — Expand and Contract pattern:
```
```
Phase 1 (expand):
Deploy migration that ADDS new column (nullable)
Both v1 and v2 work with the schema
Run: python manage.py migrate
```
```
Phase 2:
Deploy app v2 that uses new column
Old column still exists for safety
```
```
Phase 3 (contract):
After confidence → deploy migration removing old column
Only after verified v1 is completely gone
```
```
In pipeline:
stage('Migrate') {
steps {
// Run on one instance before rolling update
sh 'python manage.py migrate --check' // verify safe
sh 'python manage.py migrate' // apply
}
}
stage('Deploy') { ... } // deploy app after migration
```
```
bash
```

## Q13. A deployment succeeded but application is not working. What do you do?

### Answer:

```
# Diagnose: find the slow stages
# Look at pipeline timing — which stage takes longest?
```
```
Common culprits and fixes:
```
```
1. Dependencies installation ( 10 -15 minutes):
Fix: Cache pip/npm packages
GitHub Actions: actions/cache
Jenkins: Global tool cache
Result: 30 seconds instead of 10 minutes
```
```
2. Docker build ( 5 -10 minutes):
Fix: Layer caching — copy package files before code
GitHub Actions: cache-from/cache-to: type=gha
Fix: Multi-stage build — smaller final image
Result: 30 seconds for unchanged dependencies
```
```
3. Tests running sequentially:
Fix: Run in parallel
GitHub Actions: matrix strategy or parallel jobs
Jenkins: parallel {} block
Result: 3 parallel jobs = 3x faster
```
```
4. Large test suite ( 20 minutes):
Fix: Split into fast unit tests (< 5 min) and
slow integration tests (run less frequently)
Unit tests: every commit
Integration tests: only on main branch
```
```
5. Large Docker image:
Fix: Multi-stage build, alpine base image
Before: 900 MB, After: 150 MB
Faster: push, pull, deploy
```
```
Target metrics:
Unit tests + lint: < 5 minutes
Docker build + push: < 5 minutes
Deploy to staging: < 3 minutes
Total pipeline: < 15 minutes
```
```
bash
```

## Q14. Your pipeline works locally but fails in CI. How do you debug?

### Answer:

```
# Step 1 — Check immediately:
curl http://staging.myapp.com/health
# HTTP 200 = app responds but something else is wrong
# HTTP 500 = app is up but erroring
# Connection refused = app not running at all
```
```
# Step 2 — Check container logs:
docker logs myapp --tail 50
# or in GitHub Actions pipeline output
# Look for startup errors, missing env vars, DB connection failed
```
```
# Step 3 — Check what changed:
git diff HEAD~ 1 HEAD --stat
# What files changed in this commit?
```
```
# Step 4 — Check environment variables:
docker inspect myapp | grep - A 20 '"Env"'
# Is DB_HOST correct? Are all required vars present?
```
```
# Step 5 — Check dependencies:
docker exec myapp python3 - c "import flask; print(flask.__version__)"
# Are all packages installed correctly?
```
```
# Step 6 — Check database connectivity:
docker exec myapp python3 - c "
import os
import mysql.connector
conn = mysql.connector.connect(
host=os.environ['DB_HOST'],
user=os.environ['DB_USER'],
password=os.environ['DB_PASSWORD']
)
print('DB connected')
"
```
```
# Step 7 — Roll back immediately if users affected:
# Trigger rollback workflow with previous image tag
# Investigate on rolled-back version with no user impact
```
```
# Step 8 — After rollback — reproduce in local environment
# Fix the issue
# Add a test that would have caught this
# Re-deploy
```

## Q15. How do you prevent secrets from being exposed in CI/CD logs?

```
bash
```
```
# Classic causes:
```
```
# 1. Environment variables missing:
# Local: .env file loaded
# CI: secrets not configured or wrong name
# Check: echo all env vars at start of failing step
```
```
# 2. Different Python/Node version:
# Local: Python 3.11
# CI runner: Python 3.9
# Fix: pin version in pipeline: python-version: "3.11"
# Fix: use same version in Dockerfile and pipeline
```
```
# 3. Missing system dependencies:
# Local: already installed
# CI: fresh Ubuntu — nothing extra installed
# Fix: add apt-get install to pipeline
```
```
# 4. File path differences:
# Local: relative paths work
# CI: different working directory
# Fix: use absolute paths or $GITHUB_WORKSPACE
```
```
# 5. Test order dependency:
# Local: always run in same order, state persists
# CI: clean environment, random order
# Fix: each test must be independent
```
```
# Debugging approach:
# 1. Add verbose output to failing step
# 2. Print working directory: pwd
# 3. Print file listing: ls - la
# 4. Print environment: env | sort
# 5. Run intermediate commands to isolate failure
# 6. Use actions/upload-artifact to save debug files
# 7. Use tmate action for interactive SSH into runner
```
```
# tmate — SSH into GitHub Actions runner:
```
- name: Debug session
if: failure()
uses: mxschmitt/action-tmate@v3
with:
limit-access-to-actor: true


### Answer:

## Q16. How do you implement different configurations for dev, staging and

## production?

### Answer:

```
bash
```
```
# GitHub Actions — automatic masking:
# Any value stored in secrets is automatically masked in logs
# Even if you echo ${{ secrets.PASSWORD }} → shows ***
```
```
# Jenkins — use credentials binding:
withCredentials([
usernamePassword(
credentialsId: 'db-creds',
usernameVariable: 'DB_USER',
passwordVariable: 'DB_PASS'
)
]) {
sh 'mysql -u $DB_USER -p$DB_PASS database'
// $DB_PASS is masked in logs
}
```
```
# Additional practices:
# 1. Never echo secrets explicitly
# 2. Use set +x before secret commands (bash — disables xtrace)
# 3. Redirect command output: command > /dev/null
# 4. Use env vars not command args (visible in ps output)
# 5. Rotate secrets regularly
# 6. Audit pipeline logs periodically
# 7. Use OIDC — no static secrets to leak
```
```
# If secret accidentally logged:
# 1. Immediately rotate the secret
# 2. Check if logs are publicly visible
# 3. Delete the pipeline run logs
# 4. Audit what accessed the secret after exposure
```
```
yaml
```
```
# GitHub Actions — environments with different variables:
# Repository Settings → Environments → staging
# Add variable: DB_HOST = staging-db.company.com
```
```
# Repository Settings → Environments → production
# Add variable: DB_HOST = prod-db.company.com
```

# SECTION 5 — Advanced Concepts

## Q17. What is a blue/green deployment and how does it work in a pipeline?

## Answer:

```
# In pipeline:
jobs:
deploy-staging:
environment: staging # uses staging variables
steps:
```
- run: echo ${{ vars.DB_HOST }} # staging-db.company.com

```
deploy-production:
environment: production # uses production variables
steps:
```
- run: echo ${{ vars.DB_HOST }} # prod-db.company.com

```
# Jenkins — parameterized builds:
parameters {
choice(name: 'ENV', choices: ['staging', 'production'])
}
environment {
DB_HOST = "${params.ENV == 'production'?
'prod-db.company.com' :
'staging-db.company.com'}"
}
```
```
# Best practice — no code changes between environments:
# Same Docker image, same code
# Only configuration differs
# Configuration from environment variables at runtime
# Secrets from Secrets Manager by environment path
# prod/myapp/db vs staging/myapp/db
```
```
Blue/Green deployment:
```
```
Current state:
Blue stack: v1.0 running (100% traffic)
Green stack: empty
```
```
Pipeline triggered:
```
```
Step 1 — Deploy new version to Green:
Launch new EC 2 instances in Green Auto Scaling Group
```

## Q18. What is a canary deployment?

### Answer:

```
Deploy v1.1 Docker image
Green stack: v1.1 running (0% traffic)
```
```
Step 2 — Test Green:
Run smoke tests against Green directly (internal URL)
No user traffic to Green yet
```
```
Step 3 — Switch traffic:
Update ALB weighted routing:
Blue: 100 % → 0 %
Green: 0 % → 100 %
Happens in seconds — no downtime
```
```
Step 4 — Monitor:
Watch CloudWatch metrics for 15 minutes
Error rate, latency, 5xx count
```
```
Step 5 a — Success:
Terminate Blue stack (save cost)
Green becomes new Blue for next deployment
```
```
Step 5 b — Problem detected:
Switch ALB back to Blue (< 30 seconds)
Investigate Green in isolation
No user impact
```
```
Advantages:
```
- Instant rollback — just flip the ALB
- Green tested in production environment before traffic
- Zero downtime guaranteed
- Easy to verify before full cutover

```
Disadvantage:
```
- Double infrastructure cost during deployment
- Database schema must support both versions

```
Canary: gradually route traffic to new version
```
```
Start:
v1.0 instances: 10 (100% traffic)
v1.1 instances: 0 (0% traffic)
```
```
Step 1 — Deploy one canary:
```

## Q19. How do you test infrastructure changes in a pipeline?

### Answer:

```
v1.0 instances: 9 (90% traffic)
v1.1 instances: 1 (10% traffic)
```
```
Step 2 — Monitor for 15 minutes:
Error rate on v1.1 vs v1.0
Latency on v1.1 vs v1.0
If metrics OK → continue
```
```
Step 3 — Expand:
v1.0 instances: 5 (50% traffic)
v1.1 instances: 5 (50% traffic)
```
```
Step 4 — Monitor again
```
```
Step 5 — Full rollout:
v1.0 instances: 0 (0% traffic)
v1.1 instances: 10 (100% traffic)
```
```
If problem at ANY step:
Roll back canary instances immediately
Maximum 10 % of users affected at first step
```
```
AWS implementation:
ALB weighted target groups
Target Group 1: v1.0 instances (weight 9)
Target Group 2: v1.1 instances (weight 1)
```
```
In pipeline:
```
1. Deploy 1 new instance to Green TG
2. Set ALB weight: Blue=9, Green= 1
3. Monitor CloudWatch 15 min
4. If pass: Blue=5, Green= 5
5. Monitor 15 min
6. If pass: Blue=0, Green= 10

```
bash
```
```
# Infrastructure as Code testing pipeline:
```
```
Stage 1 — Validate:
terraform validate # syntax check
terraform fmt --check # formatting check
tflint # linting for Terraform
```

# SECTION 6 — Quick Fire Round

## Know These Instantly

```
Stage 2 — Security scan:
tfsec. # security issues in Terraform
checkov - d. # compliance checks
snyk iac test # vulnerability scan
```
```
Stage 3 — Plan:
terraform plan -out=tfplan
# Save plan as artifact
# Review plan in PR — what will change?
```
```
Stage 4 — Apply to staging:
terraform apply tfplan # apply the saved plan
# Never plan and apply separately — plan may differ
```
```
Stage 5 — Test staging infrastructure:
python test_infra.py # verify resources exist
bash smoke-test.sh # verify connectivity
```
```
Stage 6 — Manual approval for production
```
```
Stage 7 — Apply to production:
Same tfplan applied to production
```
```
Testing infrastructure (Terratest — Go):
func TestVPC(t *testing.T) {
vpc := terraform.Output(t, opts, "vpc_id")
assert.NotEmpty(t, vpc)
// verify subnets, security groups etc
}
```
```
bash
# Q: What is a webhook in CI/CD?
# An HTTP callback from GitHub/GitLab to Jenkins/other CI
# Triggered when you push code
# Jenkins URL: http://jenkins:8080/github-webhook/
```
```
# Q: What is the difference between a pipeline trigger and a schedule?
# Trigger: event-based — push, PR, tag
# Schedule: time-based — cron "run every day at 2 am"
```
```
# Q: What is a pipeline artifact?
```

# Output of build step stored for use in later stages
# Docker image, JAR file, test report, coverage XML

# Q: What is a code coverage report?
# Shows what percentage of code is covered by tests
# Common tools: pytest-cov, Istanbul (JS), JaCoCo (Java)
# Common threshold: 80 % minimum

# Q: What is a linter?
# Tool that checks code style and potential bugs
# Python: flake8, pylint
# JavaScript: ESLint
# Dockerfile: hadolint

# Q: What is SAST?
# Static Application Security Testing
# Scans source code for vulnerabilities without running it
# Tools: Bandit (Python), Semgrep, SonarQube

# Q: What is DAST?
# Dynamic Application Security Testing
# Tests running application for vulnerabilities
# Tools: OWASP ZAP, Burp Suite

# Q: What is a Dockerfile best practice scanner?
# hadolint — checks Dockerfile for issues
# docker run --rm - i hadolint/hadolint < Dockerfile

# Q: What is build caching?
# Storing dependencies between builds
# pip install, npm install take 5 minutes
# With cache: 30 seconds
# Invalidates when requirements.txt changes

# Q: What is the difference between push and pull deployment?
# Push: CI server SSH into target server and deploys
# Pull: target server polls registry and pulls updates
# Push: simpler, used for EC 2
# Pull: more secure (server initiates), used for K8s

# Q: What is GitOps?
# Git is the single source of truth for infrastructure
# All changes via pull requests — not manual commands
# Automated reconciliation — system matches what is in Git
# Tools: ArgoCD, Flux (for Kubernetes)

# Q: What is a feature flag?
# Toggle features on/off without deploying
# Deploy code to production but keep feature hidden


# SECTION 7 — Complete CI/CD Summary

## Everything You Have Mastered

## CI/CD Architecture You Can Describe in Any Interview

```
# Enable for 5 % of users, then gradually more
# Tools: LaunchDarkly, AWS AppConfig
```
```
# Q: What is trunk-based development?
# All developers commit to main (trunk) frequently
# Short-lived feature branches (< 1 day)
# Feature flags hide incomplete work
# Requires very good CI/CD and test coverage
# Used by Google, Facebook
```
```
# Q: What is a quality gate?
# Threshold that build must meet to proceed
# Example: tests must pass, coverage > 80 %, no CRITICAL CVEs
# Enforced in pipeline — stops deployment if not met
```
```
Day 1 ✅ CI/CD concepts, pipelines, stages, tools overview
Artifacts, environments, rollback, branching strategies
```
```
Day 2 ✅ GitHub Actions — workflows, jobs, steps, runners
Caching, matrix builds, secrets, environments
Complete pipeline — test, build, push, deploy
```
```
Day 3 ✅ Jenkins — installation, plugins, credentials
Declarative Jenkinsfile, parallel stages, when conditions
Post actions, webhooks, Blue Ocean
```
```
Day 4 ✅ End-to-end pipeline — OIDC, ECR, EC2, ASG
Staging deploy, integration tests, production deploy
Rollback pipeline, scheduled maintenance
```
```
Day 5 ✅ Interview mega revision — all scenarios covered
```
```
Developer pushes code
↓
GitHub webhook triggers GitHub Actions
↓
Job 1 (test):
```
- Checkout code


## Phrases That Impress CI/CD Interviewers

**Situation Say this**

Asked about secrets "We use OIDC for AWS — no static keys anywhere in the pipeline"

Asked about Docker "Every image tagged with git SHA — fully traceable to the commit"

Asked about deployments "Instance refresh on ASG with 80 % healthy minimum — zero downtime"

Asked about testing "Integration tests run against staging before any production deploy"

Asked about rollback "Every SHA tag preserved in ECR — rollback is just deploying a previous
tag"

Asked about speed "Dependencies cached — pipeline runs in under 10 minutes"

Asked about safety "Manual approval gate before production with required reviewers"

- Install dependencies (cached)
- Lint and unit tests
- Upload coverage report
    ↓ (needs: test)
Job 2 (build):
- Authenticate to AWS via OIDC (no static keys)
- Login to ECR
- Docker build (layer cached)
- Trivy security scan
- Push image: registry/myapp:sha-abc 1234
↓ (needs: build)
Job 3 (deploy-staging):
- SSH to staging EC 2
- Pull new image from ECR
- Stop old container, start new
- Health check: curl /health → 200
- Integration tests against staging
↓ (needs: deploy-staging, environment: production)
Job 4 (deploy-production):
- Manual approval gate (required reviewers)
- Update ASG Launch Template
- Start instance refresh: MinHealthyPercentage= 80
- Wait for refresh completion
- Production health check
↓ (always)
Job 5 (notify):
- Slack notification: success or failure
- Link to pipeline run


Asked about monitoring "CloudWatch health check must pass before instance refresh completes"

Asked about failures "Fail fast — pipeline stops at first failure, production untouched"

Asked about
environments

```
"Same Docker image deployed to staging and production — only config
differs"
```
## Your Complete DevOps Roadmap — Almost There

## What is Coming — Kubernetes

## 🎉 Congratulations — CI/CD Complete!

### You now understand how modern software delivery works end to end. From a developer

### typing code on their laptop to that code running in production serving real users — you

### understand every step, every tool, every decision.

```
Linux ████████████████████ ✅ COMPLETE — 16 days
Git ████████████████████ ✅ COMPLETE — 4 days
Docker ████████████████████ ✅ COMPLETE — 8 days
AWS ████████████████████ ✅ COMPLETE — 10 days
CI/CD ████████████████████ ✅ COMPLETE — 5 days
K8s ░░░░░░░░░░░░░░░░░░░░ ⬅ NEXT — 6 days
Final ░░░░░░░░░░░░░░░░░░░░ Interview prep — 2 days
```
```
CI/CD deploys Docker containers to EC 2 directly
That works for simple applications
```
```
But what if you have:
```
- 50 microservices each in Docker containers
- Need to scale individual services independently
- Container crashes — must restart automatically
- Rolling updates across 100 containers
- Efficient resource usage across many servers

```
That is what Kubernetes solves.
It is the container orchestration platform.
```
```
K8s + CI/CD = complete modern DevOps stack
Your pipeline builds Docker image → pushes to ECR
→ Kubernetes pulls image → manages containers automatically
```
```
This is exactly how companies like Flipkart, Swiggy,
Zomato run their production infrastructure.
```

