## Git Day 1 — Introduction & Core Concepts

---

### Why Git is Non-Negotiable for DevOps

Every company that writes code uses Git — no exceptions. As soon as you join a DevOps team, your first task will involve Git: cloning a repository, making changes, pushing code, reviewing pull requests.

Git is also how DevOps connects to everything else:

- CI/CD pipelines trigger when you push to Git
- Docker images are built from code stored in Git
- Infrastructure as Code (Terraform, Ansible) lives in Git
- Kubernetes manifests live in Git

> No Git knowledge = cannot work in any modern tech company.

---

### What You Will Learn Today

- What Git is and why it exists
- Git vs GitHub — the difference
- Installing and configuring Git
- The three areas of Git
- Your first repository
- Core workflow — init, add, commit, status, log
- Writing good commit messages
- The `.gitignore` file

---

### What is Git?

**Git is a version control system** — it tracks every change made to your files over time.

Imagine you are writing a deployment script. **Without Git:**
- You save `deploy.sh`
- You make changes — something breaks
- You cannot go back to the working version
- You have no idea what changed

**With Git:**
- Every version is saved with a message explaining what changed
- You can go back to any previous version instantly
- You can see exactly what changed between versions
- Multiple people can work on the same files without overwriting each other

> Think of Git like a time machine for your code.

---

### What is GitHub?

| Git | GitHub |
|-----|--------|
| Runs locally on your machine | Runs in the cloud — website |
| Tracks changes in files | Stores repositories online |
| Created by Linus Torvalds, 2005 | Created by GitHub Inc, 2008 |
| Free and open source | Free tier + paid plans |
| Works without internet | Requires internet |

**Git ≠ GitHub**
- **Git** = the tool installed on your computer that tracks changes
- **GitHub** = a website that hosts your Git repositories online

Other alternatives to GitHub: **GitLab** (self-hostable, popular in enterprises), **Bitbucket** (Atlassian — common in corporate environments).

---

### Installing Git

```bash
# Ubuntu/Debian:
sudo apt update
sudo apt install -y git

# Verify installation:
git --version
# Output: git version 2.39.x
```

---

### Configuring Git — Do This First

Before using Git you must tell it who you are. This information is attached to every commit you make.

```bash
git config --global user.name "Varun"
git config --global user.email "varun@email.com"
git config --global core.editor "vim"
git config --global init.defaultBranch "main"

# Verify your config:
git config --list
```

> `--global` means this applies to all repositories on your machine. Without `--global`, the setting only applies to the current repository.

---

### The Three Areas of Git — Most Important Concept

This is what most beginners do not understand, and it causes all the confusion.

```
Working Directory  →  Staging Area  →  Repository
   (your files)        (git add)       (git commit)

 where you edit     where you prepare   where history
   your files          your changes      is saved forever
```

**Think of it like packing for a trip:**

| Area | Analogy |
|------|---------|
| Working Directory | Your room with all your stuff |
| Staging Area | Your suitcase — where you decide what to pack |
| Repository | The suitcase is closed and ready — history saved |

You choose exactly what goes into each commit using the staging area. You might change 10 files but only commit 3 of them together.

---

### Your First Repository

```bash
cd ~/linux_practice
mkdir git_practice
cd git_practice

# Initialize git repository
git init
```

Output:
```
Initialized empty Git repository in /home/varun/git_practice/.git/
```

This creates a hidden `.git` folder — Git stores everything here. **Never manually edit or delete this folder.**

```bash
ls -la              # see the .git folder
ls .git/
```

```
HEAD      # points to current branch
config    # repository-specific config
objects/  # all your file content stored here
refs/     # branches and tags
hooks/    # scripts that run on git events
```

> You never need to touch these directly — Git manages them. But knowing they exist helps when debugging.

---

### Core Git Commands

#### `git status` — Your Most Used Command

Shows:
- Which branch you are on
- Which files are modified
- Which files are staged (ready to commit)
- Which files are untracked (Git does not know about them)

```bash
git status
```

> Run `git status` constantly — before every add, after every add, before every commit. It tells you exactly what is happening.

#### `git add` — Moving to Staging Area

```bash
# Create a file first
echo "My first DevOps script" > deploy.sh
echo "Server configuration" > config.yml

git status            # shows both files as "untracked"

git add deploy.sh     # add one specific file
git add config.yml    # add another file
git add .             # add ALL changed files in current folder
git add -p            # interactive — choose which parts to stage
```

> ⚠️ **`git add .` warning:** adds *everything*, including files you did not mean to add. Better habit — add files individually, or use `.gitignore` first.

> Always run `git status` after `git add .` to see exactly what was staged.

#### `git commit` — Saving to History

```bash
git commit -m "Add deployment script and config file"

# Stage and commit in one command (only for tracked files):
git commit -am "Fix typo in deploy script"
# -a stages all MODIFIED tracked files automatically
# Does NOT add new untracked files — use git add for those
```

> The `-m` flag adds the message inline. Without `-m`, Git opens your configured editor.

**Format:** Start with a verb. Be specific. Keep under 72 characters. Explain WHAT and WHY — not HOW.

> Interviewers often ask to see your Git history. Bad commit messages show unprofessional habits.

---

### Writing Good Commit Messages — Critical

**Bad commit messages:**
```
"fix"
"changes"
"update"
"asdfgh"
"WIP"
```

**Good commit messages:**
```
"Add nginx configuration for production server"
"Fix disk space check in monitoring script"
"Update database backup to run at 1am daily"
"Remove hardcoded password from deploy script"
```

---

### `git log` — Viewing History

```bash
git log                       # full history — press q to quit
git log --oneline             # one line per commit — cleaner
git log --oneline -10         # last 10 commits only
git log --oneline --graph     # shows branch graph
git log --author="Varun"      # commits by specific person
git log --since="7 days ago"  # commits in last 7 days
git log -p                    # shows actual changes in each commit
```

Each commit has:
```
commit a1b2c3d4e5f6...           # SHA — unique identifier for this commit
Author: Varun <varun@email.com>
Date:   Thu Mar 20 10:30:00 2024

    Add deployment script and config file
```

> The SHA is how you reference any commit. You only need the first 7 characters.

---

### `git diff` — See What Changed

```bash
git diff               # changes in working directory not yet staged
git diff --staged      # changes staged but not yet committed
git diff HEAD          # all changes since last commit
git diff a1b2c3d       # difference between two commits
```

---

### The `.gitignore` File

**What is `.gitignore`?**

A file that tells Git which files and folders to completely ignore — never track, never add, never show in `git status`.

**What to put in `.gitignore`:**

```bash
# Secrets and environment files — NEVER commit these
.env
.env.local
*.pem
*.key
secrets.yml

# Logs — no need to track
*.log
logs/

# OS generated files
.DS_Store     # Mac
Thumbs.db     # Windows
desktop.ini

# Dependencies — too large, regenerated from package files
node_modules/
vendor/
__pycache__/
*.pyc

# Build output
dist/
build/
target/
*.class
*.jar

# IDE files
.vscode/
.idea/
*.swp         # Vim swap files

# Terraform
*.tfstate
*.tfstate.backup
.terraform/
```

**Creating `.gitignore` for your practice:**

```bash
cd ~/linux_practice/git_practice

cat > .gitignore << 'EOF'
*.log
.env
*.pem
*.key
.DS_Store
__pycache__/
*.pyc
EOF

git add .gitignore
git commit -m "Add gitignore for logs, secrets and OS files"
```

**Global `.gitignore` — applies to all repos:**

```bash
vim ~/.gitignore_global
```
```
# Add OS files:
.DS_Store
Thumbs.db
*.swp
```
```bash
git config --global core.excludesfile ~/.gitignore_global
```

> Now these are ignored in every repository automatically.

---

### Complete First Day Workflow — Memorize This

```bash
# 1. Initialize repository
git init

# 2. Check status — always first
git status

# 3. Create .gitignore before anything else
vim .gitignore

# 4. Add files to staging
git add .

# 5. Check what was staged
git status

# 6. Commit with clear message
git commit -m "Initial commit — add project files"

# 7. Verify commit was saved
git log --oneline
```

---

### Practice — Do This Right Now

```bash
cd ~/linux_practice/git_practice

# Create some files
echo "#!/bin/bash" > deploy.sh
echo "echo 'Deploying application...'" >> deploy.sh
echo "DB_HOST=localhost" > .env
echo "server_name localhost;" > nginx.conf

# Initialize and make first commit
git init
cat > .gitignore << 'EOF'
.env
*.log
*.pem
EOF

git add .gitignore
git commit -m "Add gitignore"

git add deploy.sh nginx.conf
git commit -m "Add deployment script and nginx config"

git log --oneline
git status
```

> Notice `.env` does not appear in `git status` — `.gitignore` is working.

---

### Summary — Git Day 1

| Command | What it does |
|---------|---------------|
| `git config --global user.name` | Set your name |
| `git init` | Initialize new repository |
| `git status` | See current state — use constantly |
| `git add filename` | Stage a specific file |
| `git add .` | Stage all changes |
| `git commit -m "message"` | Save staged changes to history |
| `git log --oneline` | View commit history |
| `git diff` | See unstaged changes |
| `git diff --staged` | See staged changes |

---

### Interview Questions — Git Day 1

**Q1. What is Git and why is it used?**
Git is a distributed version control system that tracks changes to files over time. Used to collaborate on code, maintain history of all changes, revert to previous versions, and work on features in parallel using branches. Every DevOps team uses Git.

**Q2. What is the difference between Git and GitHub?**
Git is the version control tool installed locally on your machine. GitHub is a cloud platform that hosts Git repositories online for collaboration and backup. GitHub also provides features like pull requests, issues, and CI/CD pipelines.

**Q3. What are the three areas of Git?**
Working Directory — where you edit files; Staging Area — where you prepare changes with `git add`; Repository — where history is permanently saved with `git commit`.

**Q4. What is the difference between `git add .` and `git add -p`?**
`git add .` stages all changed files at once. `git add -p` goes through each change interactively, asking which parts to stage — useful when you want to commit only specific changes from a file.

**Q5. What is a commit SHA?**
A unique 40-character hash identifier automatically generated for every commit. Used to reference specific commits — you only need the first 7 characters. Example: `git diff a1b2c3d e4f5g6h` compares two commits.

**Q6. Why should `.env` files never be committed to Git?**
`.env` files contain secrets — database passwords, API keys, access tokens. Committing them exposes these secrets to anyone with repository access. On public repositories this is a serious security incident. Always add `.env` to `.gitignore`.

---

### Homework — Before Git Day 2

1. Configure Git with your name and email
2. Create a new folder, initialize a repository
3. Create 3 files — a script, a config, and a readme
4. Add a proper `.gitignore`
5. Make 3 separate commits with clear messages
6. Run `git log --oneline` and see your history
7. Edit a file, then run `git diff` before staging

> Git Day 2 covers Branches, Merging, and Resolving Conflicts — the most important Git concept for team collaboration.
