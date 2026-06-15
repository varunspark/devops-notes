## CI/CD Day 3 — Jenkins

---

### Why Jenkins After GitHub Actions?

You might wonder — we already have GitHub Actions, why learn Jenkins?

```
Real job market breakdown:

Startups (< 50 engineers):    GitHub Actions — 70%,  Jenkins — 30%
Mid-size (50–500 engineers):  Jenkins — 60%,         GitHub Actions — 40%
Enterprise (500+):            Jenkins — 80%,         GitHub Actions — 20%

At 10–12 LPA level — you will interview at mid-size companies
where Jenkins is the most common CI/CD tool.

Interview reality:
"Do you know Jenkins?"     → asked in 70% of DevOps interviews
"Show me a Jenkinsfile"    → asked in 50% of technical rounds
```

Both tools do the same job. Jenkins gives you more control. GitHub Actions is easier to set up. You need to know both.

---

### What You Will Learn Today

- Installing Jenkins on EC2
- Jenkins architecture — master and agents
- Jenkinsfile — pipeline as code
- Declarative vs Scripted pipeline
- Stages and steps
- Environment variables and credentials
- Jenkins plugins
- Webhook — auto trigger on Git push
- Parallel stages
- Post actions — always, success, failure
- Complete pipeline — test, build, push, deploy

---

### Jenkins Architecture

```
Jenkins Master (EC2 — controller)
├── Web UI (port 8080)
├── Pipeline scheduler
├── Plugin manager
└── Credential store

Jenkins Agents (workers — where builds run)
├── Agent 1 (same EC2 or separate)
├── Agent 2
└── Agent 3

Master distributes work to agents
Agents run the actual build steps
Master just coordinates — never runs builds directly in production
```

> For learning — master and agent on the same EC2 is fine. In production — separate agents for scalability.

---

### Step 1 — Install Jenkins on EC2

Launch a **t2.medium** EC2 (Jenkins needs more RAM than t2.micro):

```bash
# User Data for Jenkins EC2:
#!/bin/bash
set -e
exec > /var/log/jenkins-setup.log 2>&1

echo "=== Installing Jenkins ==="

# Update system
apt-get update -y

# Install Java (Jenkins requires Java 11 or 17):
apt-get install -y openjdk-17-jdk

# Verify Java:
java -version

# Add Jenkins repository:
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins:
apt-get update -y
apt-get install -y jenkins

# Start Jenkins:
systemctl start jenkins
systemctl enable jenkins

# Install Docker (for building images):
curl -fsSL https://get.docker.com | bash
usermod -aG docker jenkins    # allow Jenkins to use Docker
usermod -aG docker ubuntu

# Install AWS CLI:
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
/tmp/aws/install

# Wait for Jenkins to start:
sleep 30

echo "=== Jenkins setup complete ==="
echo "Initial admin password:"
cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Security group for Jenkins EC2:**

```bash
# Allow port 8080 from your IP:
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 8080 \
  --cidr $(curl -s ifconfig.me)/32

# Allow SSH:
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $(curl -s ifconfig.me)/32
```

---

### Step 2 — Initial Jenkins Setup

```bash
# Get initial admin password:
ssh -i mykey.pem ubuntu@<jenkins-ec2-ip>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Open browser:
http://<jenkins-ec2-ip>:8080

# Paste the initial password
# Click: Install suggested plugins (wait 3–5 minutes)
# Create admin user: admin / YourPassword123
# Jenkins URL: http://<jenkins-ec2-ip>:8080
# Save and finish
```

---

### Step 3 — Install Plugins

```
Jenkins → Manage Jenkins → Plugins → Available

Install these plugins:
✅ Pipeline               (already installed)
✅ Git                    (already installed)
✅ Docker Pipeline        search and install
✅ GitHub Integration     search and install
✅ Blue Ocean             search and install (better UI)
✅ Credentials Binding    search and install
✅ Workspace Cleanup      search and install
✅ Timestamper            search and install
✅ AnsiColor              search and install (colored output)

After installing → Restart Jenkins
```

---

### Step 4 — Add Credentials to Jenkins

```
Jenkins → Manage Jenkins → Credentials
→ System → Global credentials
→ Add Credentials
```

Add these:

1. **Docker Hub:**
   - Kind: Username with password
   - Username: your-dockerhub-username
   - Password: your-dockerhub-token
   - ID: `dockerhub-credentials`
   - Description: Docker Hub Access

2. **GitHub:**
   - Kind: Secret text
   - Secret: your-github-token
   - ID: `github-token`
   - Description: GitHub Personal Access Token

3. **SSH Key for deployment:**
   - Kind: SSH Username with private key
   - Username: ubuntu
   - Private key: paste your .pem file contents
   - ID: `deployment-ssh-key`
   - Description: Production Server SSH Key

4. **AWS Credentials:**
   - Kind: AWS Credentials (needs AWS Credentials plugin)
   - Access Key ID: your-access-key
   - Secret Access Key: your-secret-key
   - ID: `aws-credentials`
   - Description: AWS Access

---

### Jenkinsfile — Pipeline as Code

#### Declarative vs Scripted Pipeline

Always use **Declarative**. It is cleaner, easier to read, has better error messages and is what interviewers expect.

```groovy
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

// Scripted — full Groovy, more flexible:
node {
  stage('Test') {
    sh 'pytest'
  }
}
```

#### Declarative Pipeline Structure

```groovy
pipeline {
  agent any              // where to run

  options { }            // pipeline-level options
  environment { }        // environment variables
  parameters { }         // input parameters
  triggers { }           // what triggers the build

  stages {
    stage('Stage Name') {
      when { }           // conditional execution
      steps {
        sh 'command'     // shell command
        script { }       // Groovy script block
      }
      post { }           // after stage actions
    }
  }

  post { }               // after all stages
}
```

---

### Step 5 — Your First Jenkinsfile

Create `Jenkinsfile` in your repository root:

```groovy
pipeline {
  agent any

  environment {
    APP_NAME = 'myapp'
    PYTHON_VERSION = '3.11'
  }

  stages {

    stage('Checkout') {
      steps {
        // Jenkins automatically checks out code
        // when triggered by webhook
        echo "Building branch: ${env.BRANCH_NAME}"
        echo "Commit: ${env.GIT_COMMIT}"
        sh 'git log --oneline -5'
      }
    }

    stage('Setup Python') {
      steps {
        sh '''
          python3 --version
          python3 -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 pytest-cov
        '''
      }
    }

    stage('Lint') {
      steps {
        sh 'flake8 . --max-line-length=100 --exclude=.git,__pycache__'
      }
    }

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

---

### Step 6 — Create Jenkins Pipeline Job

```
Jenkins → New Item
→ Name: myapp-pipeline
→ Type: Pipeline
→ OK

Configuration:
  General:
    ✅ Discard old builds → Max: 10

  Pipeline:
    Definition: Pipeline script from SCM
    SCM: Git
    Repository URL: https://github.com/varun/myapp
    Credentials: github-token
    Branch: */main
    Script Path: Jenkinsfile

→ Save
→ Build Now
```

Watch the build run in the console output.

---

### Step 7 — Complete Production Jenkinsfile

```groovy
pipeline {
  agent any

  // ── Options ────────────────────────────────────
  options {
    timestamps()                                    // add timestamps to console
    ansiColor('xterm')                              // colored output
    timeout(time: 30, unit: 'MINUTES')              // fail if takes > 30 min
    buildDiscarder(
      logRotator(numToKeepStr: '10')                // keep last 10 builds
    )
    disableConcurrentBuilds()                       // one build at a time
  }

  // ── Environment ────────────────────────────────
  environment {
    APP_NAME    = 'myapp'
    IMAGE_NAME  = "varun/${APP_NAME}"
    DOCKER_CREDS = credentials('dockerhub-credentials')
    AWS_CREDS    = credentials('aws-credentials')
    GIT_SHA_SHORT = sh(
      script: 'git rev-parse --short HEAD',
      returnStdout: true
    ).trim()
  }

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
    githubPush()  // trigger on GitHub webhook
  }

  stages {

    // ════════════════════════════════════════
    // STAGE 1 — CHECKOUT
    // ════════════════════════════════════════
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_COMMIT_MSG = sh(
            script: 'git log -1 --pretty=%B',
            returnStdout: true
          ).trim()
          echo "Commit:  ${env.GIT_SHA_SHORT}"
          echo "Message: ${env.GIT_COMMIT_MSG}"
          echo "Branch:  ${env.BRANCH_NAME}"
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
          pip install flake8 pytest-cov --quiet
          echo "Dependencies installed"
        '''
      }
    }

    // ════════════════════════════════════════
    // STAGE 3 — QUALITY CHECKS (parallel)
    // ════════════════════════════════════════
    stage('Code Quality') {
      when {
        not { expression { params.SKIP_TESTS } }
      }
      parallel {
        stage('Lint') {
          steps {
            sh '''
              echo "Running flake8..."
              flake8 . \
                --max-line-length=100 \
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

    // ════════════════════════════════════════
    // STAGE 4 — TEST
    // ════════════════════════════════════════
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
            reportName:  'Coverage Report',
            reportDir:   '.',
            reportFiles: 'coverage.xml',
            keepAll:     true
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

    // ════════════════════════════════════════
    // STAGE 7 — PUSH TO REGISTRY
    // ════════════════════════════════════════
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
            def image = docker.image("${IMAGE_NAME}:${GIT_SHA_SHORT}")

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
      }
      steps {
        script {
          def image     = "${IMAGE_NAME}:${GIT_SHA_SHORT}"
          def stagingIp = env.STAGING_SERVER_IP

          sshagent(['deployment-ssh-key']) {
            sh """
              ssh -o StrictHostKeyChecking=no ubuntu@${stagingIp} \
                "docker pull ${image} && \
                 docker stop ${APP_NAME} || true && \
                 docker rm   ${APP_NAME} || true && \
                 docker run -d \
                   --name ${APP_NAME} \
                   --restart unless-stopped \
                   -p 80:5000 \
                   -e ENVIRONMENT=staging \
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
          sleep(15)   // wait for container to start

          def stagingUrl  = "http://${env.STAGING_SERVER_IP}"
          def maxAttempts = 5
          def attempt     = 0

          while (attempt < maxAttempts) {
            attempt++
            try {
              def response = sh(
                script: "curl -sf ${stagingUrl}/health",
                returnStatus: true
              )
              if (response == 0) {
                echo "✅ Health check passed on attempt ${attempt}"
                break
              }
            } catch (Exception e) {
              echo "Attempt ${attempt}/${maxAttempts} failed"
            }

            if (attempt >= maxAttempts) {
              error("Health check failed after ${maxAttempts} attempts")
            }
            sleep(10)
          }
        }
      }
    }

    // ════════════════════════════════════════
    // STAGE 10 — DEPLOY TO PRODUCTION
    // ════════════════════════════════════════
    stage('Deploy to Production') {
      when {
        allOf {
          branch 'main'
          expression { params.ENVIRONMENT == 'production' }
        }
      }
      input {
        message   "Deploy to PRODUCTION?"
        ok        "Yes, deploy!"
        submitter "admin,varun"   // only these users can approve
        parameters {
          string(
            name:        'REASON',
            description: 'Reason for deployment'
          )
        }
      }
      steps {
        withAWS(credentials: 'aws-credentials', region: 'ap-south-1') {
          sh """
            # Update Launch Template with new image
            aws ec2 create-launch-template-version \
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
      sh """
        curl -X POST '${env.SLACK_WEBHOOK}' \
          -H 'Content-type: application/json' \
          -d '{
            "text": "✅ *${APP_NAME}* build ${BUILD_NUMBER} passed!\\nBranch: ${BRANCH_NAME}\\nCommit: ${GIT_SHA_SHORT}"
          }' || true
      """
    }
    failure {
      echo "❌ Pipeline failed!"
      sh """
        curl -X POST '${env.SLACK_WEBHOOK}' \
          -H 'Content-type: application/json' \
          -d '{
            "text": "❌ *${APP_NAME}* build ${BUILD_NUMBER} FAILED!\\nBranch: ${BRANCH_NAME}\\nCommit: ${GIT_SHA_SHORT}"
          }' || true
      """
    }
    unstable {
      echo "⚠ Pipeline unstable — tests have failures"
    }
    always {
      // Clean workspace to free disk space:
      cleanWs(
        cleanWhenSuccess: true,
        cleanWhenFailure: true,
        cleanWhenAborted: true
      )
    }
  }
}
```

---

### Jenkins Parallel Stages

```groovy
stage('Run Tests in Parallel') {
  parallel {
    stage('Unit Tests') {
      steps {
        sh 'pytest tests/unit/ -v'
      }
    }
    stage('Integration Tests') {
      steps {
        sh 'pytest tests/integration/ -v'
      }
    }
    stage('Security Scan') {
      steps {
        sh 'trivy fs --severity HIGH,CRITICAL .'
      }
    }
  }
  // All three run simultaneously
  // Stage fails if ANY parallel stage fails
}
```

---

### When Conditions

```groovy
// Run only on main branch:
when { branch 'main' }

// Run on specific branches:
when {
  anyOf {
    branch 'main'
    branch 'develop'
  }
}

// Run only on tags:
when { tag "v*" }

// Run based on parameter:
when {
  expression { params.ENVIRONMENT == 'production' }
}

// Run only if previous stage succeeded:
when { expression { currentBuild.result == null } }

// Run on all branches EXCEPT main:
when {
  not { branch 'main' }
}

// Combine conditions:
when {
  allOf {
    branch 'main'
    not { expression { params.SKIP_TESTS } }
  }
}
```

---

### Environment Variables in Jenkins

```groovy
// Built-in Jenkins variables:
env.BUILD_NUMBER   // 42
env.BUILD_URL      // http://jenkins:8080/job/myapp/42/
env.JOB_NAME       // myapp-pipeline
env.BRANCH_NAME    // main
env.GIT_COMMIT     // full commit SHA
env.GIT_BRANCH     // origin/main
env.WORKSPACE      // /var/lib/jenkins/workspace/myapp

// From credentials:
environment {
  DOCKER_CREDS = credentials('dockerhub-credentials')
  // Creates: DOCKER_CREDS_USR and DOCKER_CREDS_PSW
}

// Dynamic variable from command:
script {
  env.GIT_SHA_SHORT = sh(
    script: 'git rev-parse --short HEAD',
    returnStdout: true
  ).trim()
}
```

---

### GitHub Webhook — Auto Trigger Jenkins

**GitHub configuration:**

```
GitHub → Repository → Settings → Webhooks → Add webhook

Payload URL:  http://<jenkins-ip>:8080/github-webhook/
Content type: application/json
Events:       Just the push event
Active:       ✅
```

**Jenkins configuration:**

```
Pipeline job → Configure
Build Triggers:
  ✅ GitHub hook trigger for GITScm polling
```

> Now every `git push` automatically triggers the Jenkins pipeline.

---

### Jenkins Pipeline — Useful Step Reference

```groovy
// sh — run shell command
sh 'echo hello'

// sh — capture output
def result = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

// sh — capture exit code (don't fail on error)
def exitCode = sh(script: 'pytest tests/', returnStatus: true)
if (exitCode != 0) {
  echo "Tests failed but continuing..."
}

// echo — print message
echo "Building version ${env.BUILD_NUMBER}"

// error — fail pipeline with message
error("Tests failed — aborting deployment")

// input — manual approval gate
input message: "Deploy to production?", ok: "Deploy"

// sleep — wait (in seconds)
sleep(30)

// timeout — fail if takes too long
timeout(time: 5, unit: 'MINUTES') {
  sh './slow-test.sh'
}

// retry — retry failed steps
retry(3) {
  sh 'curl https://flaky-service.com/api'
}

// withEnv — temporary environment
withEnv(['FLASK_ENV=testing']) {
  sh 'pytest'
}
```

---

### Jenkins vs GitHub Actions — Side by Side

```groovy
// GitHub Actions:
- name: Run tests
  run: pytest test_app.py -v
  env:
    FLASK_ENV: testing

// Jenkins equivalent:
stage('Run tests') {
  steps {
    withEnv(['FLASK_ENV=testing']) {
      sh 'pytest test_app.py -v'
    }
  }
}
```

```groovy
// GitHub Actions:
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

// Jenkins equivalent:
docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
  // Docker commands here
}
```

---

### Blue Ocean — Better Jenkins UI

```
Jenkins → Open Blue Ocean (top menu)

Blue Ocean shows:
  - Visual pipeline graph
  - Parallel stages side by side
  - Test results inline
  - Much cleaner than classic UI
```

> For interviews — mention Blue Ocean as the modern Jenkins UI.

---

### Full Summary — CI/CD Day 3

| Concept | Key point |
|---------|-----------|
| Jenkins | Self-hosted CI/CD — most used in enterprise |
| Jenkinsfile | Pipeline as code — Groovy — stored in repo root |
| Declarative | Structured pipeline — always use this |
| Agent | Where build runs — `agent any` for local |
| Stage | Logical grouping of steps |
| Steps | `sh`, `echo`, `script`, `input` |
| `parallel` | Run stages simultaneously |
| `when` | Conditional stage execution |
| `post` | Run after stages — `success`, `failure`, `always` |
| `credentials` | Access stored secrets securely |
| `input` | Manual approval gate |
| Webhook | Auto trigger on GitHub push |
| Blue Ocean | Modern Jenkins UI |

---

### Interview Questions — CI/CD Day 3

**Q1. What is Jenkins and how does it work?**
Jenkins is an open-source self-hosted CI/CD automation server. You install it on your own infrastructure — EC2, VM or bare metal. Developers define pipelines as code in a Jenkinsfile using Groovy syntax. Jenkins monitors Git repositories via webhooks or polling. When changes are detected it runs the pipeline on build agents — virtual machines or containers that execute the actual build steps. The master coordinates scheduling, the agents do the work.

**Q2. What is a Jenkinsfile and why store it in the repository?**
A Jenkinsfile defines the entire CI/CD pipeline as code — stages, steps, conditions and post actions — using Groovy syntax. Storing it in the repository means the pipeline is version controlled alongside the application code. Pipeline changes go through code review. Any team member can see exactly how the build and deployment works. Different branches can have different pipeline behaviors.

**Q3. What is the difference between Declarative and Scripted Jenkins pipeline?**
Declarative pipeline uses a structured `pipeline { }` block with predefined sections — agent, stages, post. It is opinionated, easier to read and write, and has better error messages. Scripted pipeline uses raw Groovy inside `node { }` — more flexible but more complex and error-prone. Always use Declarative unless you need functionality only available in Scripted.

**Q4. How do you implement parallel stages in Jenkins?**
Using the `parallel` block inside a stage. Each parallel branch runs simultaneously on the same or different agents. All parallel branches must complete before the pipeline continues. If any branch fails the stage fails. Used for running unit tests, integration tests and security scans simultaneously to reduce total pipeline time.

**Q5. How do you manage secrets in Jenkins?**
Store secrets in Jenkins Credentials Manager — Manage Jenkins → Credentials. Different types — username/password, secret text, SSH key, AWS credentials. Reference in pipeline with `credentials('credential-id')` in the environment block or `withCredentials` step. Credentials are injected as environment variables at runtime and masked in logs — never printed even if echoed.

**Q6. What is the input step and when do you use it?**
The `input` step pauses pipeline execution and waits for a human to approve or reject. Used as a manual approval gate before deploying to production. You can specify which users can approve with `submitter`, add a reason field and set a timeout. Pipeline holds the build in waiting state until approved — approved builds continue, rejected builds are aborted.

**Q7. What does `post` do in a Jenkinsfile?**
The `post` section defines actions that run after stages complete regardless of outcome. `success` runs only when pipeline passes. `failure` runs only when it fails. `always` runs every time — used for cleanup like `cleanWs()`. `unstable` runs when tests have failures but pipeline continues. Essential for notifications and cleanup that must happen regardless of build result.

---

### Homework — Before CI/CD Day 4

```groovy
// Create this Jenkinsfile in your repository and get it working in Jenkins:

pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 20, unit: 'MINUTES')
  }

  environment {
    APP_NAME = 'myapp'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        echo "Building: ${env.BRANCH_NAME}"
      }
    }

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
            sh 'flake8 app.py --max-line-length=100'
          }
        }
      }
    }

    stage('Build') {
      steps {
        sh "docker build -t ${APP_NAME}:\$(git rev-parse --short HEAD) ."
        echo "Image built successfully"
      }
    }

    stage('Approval') {
      when { branch 'main' }
      input {
        message "Deploy to staging?"
        ok      "Deploy"
      }
      steps {
        echo "Deployment approved!"
      }
    }

  }

  post {
    success { echo "✅ Build passed!" }
    failure { echo "❌ Build failed!" }
    always  { cleanWs() }
  }
}
```

> CI/CD Day 4 builds a complete end-to-end pipeline — GitHub Actions that automatically tests, builds a Docker image, pushes to AWS ECR and deploys to EC2 using all AWS knowledge from Days 1–10. Everything connected together.
