## Git Day 3 — GitHub, Remote Repositories, Push, Pull & Pull Requests

---

### Why This is Critical for DevOps

Everything you have done so far in Git has been local — only on your machine. In real DevOps work:

- Your code lives on GitHub — not just your laptop
- CI/CD pipelines trigger when you push to GitHub
- Your team reviews your code through Pull Requests before it goes to production
- Other engineers pull your changes to stay in sync
- Docker images are built from code hosted on GitHub

> GitHub is where Git meets the real world of team collaboration and automation.

---

### What You Will Learn Today

- Creating a GitHub account and repository
- Connecting local repo to GitHub (remote)
- `git push` — sending your commits to GitHub
- `git pull` — getting changes from GitHub
- `git fetch` — checking changes without merging
- `git clone` — downloading a repository
- Pull Requests — the heart of team collaboration
- Forking repositories
- Real DevOps GitHub workflows

---

### Quick Concept — Local vs Remote

```
LOCAL (your machine)                REMOTE (GitHub)
git_practice/                       github.com/varun/git_practice
├── .git/                           ├── main branch
├── deploy.sh                       ├── feature branches
└── nginx.conf                      └── pull requests

git push  ──────────────────>  sends your commits up
git pull  <──────────────────  brings their commits down
```

---

### Setting Up GitHub

**Step 1 — Create GitHub Account**

Go to github.com and create a free account. Use your professional email — this account will be on your resume.

**Step 2 — Set Up SSH Authentication with GitHub**

GitHub no longer accepts passwords for git operations. You must use SSH keys or Personal Access Tokens. SSH keys are cleaner.

```bash
# Generate key (if you don't have one already):
ssh-keygen -t ed25519 -C "varun@email.com"

# Copy your PUBLIC key:
cat ~/.ssh/id_ed25519.pub
```

Go to **GitHub → Settings → SSH and GPG Keys → New SSH Key** → paste your public key → Save.

Test the connection:

```bash
ssh -T git@github.com
# Output: Hi varun! You've successfully authenticated
```

**Step 3 — Create Repository on GitHub**

1. Click **+ → New repository**
2. Repository name: `devops-scripts`
3. Description: "Linux and DevOps automation scripts"
4. Choose Public or Private
5. Do NOT initialize with README — you have local code already
6. Click **Create repository**

> GitHub shows you the commands to connect — we will use the SSH URL format.

---

### Connecting Local Repo to GitHub

**What is a Remote?**

A remote is a named link to a repository on another server. By convention, the main remote is called `origin`.

```bash
# Add remote to your local repo:
git remote add origin git@github.com:varun/devops-scripts.git

# Verify remote was added:
git remote -v
# Output:
# origin  git@github.com:varun/devops-scripts.git (fetch)
# origin  git@github.com:varun/devops-scripts.git (push)
```

**Remote Commands:**

```bash
git remote -v                       # list all remotes
git remote add origin URL           # add new remote
git remote remove origin            # remove remote
git remote rename origin upstream   # rename remote
git remote set-url origin NEW_URL   # change remote URL
```

---

### `git push` — Sending to GitHub

**Basic Push:**

```bash
git push origin main
```

This pushes your main branch to origin (GitHub).

**First Push — Setting Upstream:**

The first time you push a branch, you must tell Git where to push it:

```bash
git push -u origin main
# -u sets upstream — remembers where to push/pull
# After this you can just type: git push
```

`-u` or `--set-upstream` links your local branch to the remote branch. After running this once — just type `git push` and it knows where to go.

**Pushing Different Branches:**

```bash
git push origin main              # push main branch
git push origin feature-login     # push feature branch
git push -u origin feature-login  # push and set upstream
git push                          # push current branch (after -u set)
git push origin --all             # push ALL branches at once
```

**Pushing Tags:**

```bash
git push origin v1.0      # push specific tag
git push origin --tags    # push all tags
```

> ⚠️ **Force Push — Use With Extreme Caution.** Force push rewrites the remote branch history. If someone else has pulled that branch, their history now conflicts.
>
> ```bash
> git push --force origin main        # overwrites remote history
> git push --force-with-lease         # safer force push — checks for others' changes
> ```
>
> Never force push to main/master in a team environment. Only acceptable on your own feature branches when rebasing.

---

### `git clone` — Downloading a Repository

**What is Cloning?**

Cloning creates a complete local copy of a remote repository — all files, all history, all branches.

```bash
git clone git@github.com:varun/devops-scripts.git
# Creates a folder called devops-scripts with everything inside

git clone git@github.com:varun/devops-scripts.git my-folder
# Clone into a specific folder name

git clone --depth 1 git@github.com:varun/devops-scripts.git
# Shallow clone — only latest commit, no history — faster
```

**Clone vs Init + Remote Add:**

```bash
# Use clone when:
# → You want to work on an existing project
git clone git@github.com:company/project.git

# Use init + remote add when:
# → You have existing local code you want to push to a new GitHub repo
git init
git remote add origin git@github.com:varun/project.git
```

**What Clone Creates:**

```bash
git clone git@github.com:varun/devops-scripts.git
cd devops-scripts

git remote -v          # origin already set automatically
git branch -a          # shows all remote branches too
git log --oneline      # full history is there
```

> Cloning automatically sets `origin` to point back to GitHub. No manual `remote add` needed.

---

### `git fetch` vs `git pull`

**The Difference — Very Important for Interviews**

Think of it like checking your mailbox:
- **`git fetch`** = go to the mailbox, see what arrived, bring it inside but don't open yet
- **`git pull`** = go to the mailbox, bring it inside AND read and act on everything

**`git fetch` — Safe Way to Check:**

```bash
git fetch origin    # downloads changes but does NOT merge

git log --oneline HEAD..origin/main   # see what changed on remote
git diff HEAD origin/main             # see exact differences
git merge origin/main                 # NOW merge if you are happy
```

> `git fetch` is safer because it lets you inspect what changed before integrating. In production environments many engineers prefer fetch + review + merge over a direct pull.

**`git pull` — Fetch + Merge in One:**

```bash
git pull origin main    # downloads AND merges into current branch
git pull                 # pull current branch (after upstream set)
git pull --rebase        # fetch and rebase instead of merge
```

> `git pull --rebase` keeps history cleaner — instead of creating a merge commit, it replays your commits on top of the fetched commits. Many teams require this.

**When to Use Which:**

| Situation | Use |
|-----------|-----|
| Want to see what changed before merging | `git fetch` then review |
| Quick update on your own branch | `git pull` |
| Team requires clean history | `git pull --rebase` |
| Before starting new work | `git fetch origin` |

---

### Tracking Remote Branches

**Seeing Remote Branches:**

```bash
git branch -a    # all branches including remote
git branch -r    # remote branches only
```

Output:
```
* main
  feature-login
  remotes/origin/main
  remotes/origin/feature-monitoring
```

> `remotes/origin/main` = the state of `main` on GitHub as of the last fetch.

**Working with Remote Branches:**

```bash
# Get a remote branch that does not exist locally:
git switch -c feature-login origin/feature-login
# or shorter:
git switch feature-login   # Git automatically tracks remote branch

# See tracking relationships:
git branch -vv
# Output shows which local branch tracks which remote branch
```

---

### Pull Requests — The Heart of Collaboration

**What is a Pull Request?**

A Pull Request (PR) is a request to merge your branch into another branch — with a code review step in between.

```
Your laptop                    GitHub Team
feature branch  →  Open PR  →  Review code
       ↓                            ↓
   CI runs tests          Approve / Request changes
       ↓                            ↓
                Merge to main → Code deployed
```

PRs are how professional teams:
- Review code before it goes to production
- Discuss changes and suggest improvements
- Trigger automated testing (CI/CD)
- Maintain code quality

### Creating a Pull Request — Step by Step

**Step 1 — Create and push feature branch:**

```bash
git switch -c feature/add-backup-script
echo "#!/bin/bash" > backup.sh
echo "tar -czf /backup/home_$(date +%Y%m%d).tar.gz /home/varun/" >> backup.sh
git add backup.sh
git commit -m "Add automated home directory backup script"
git push -u origin feature/add-backup-script
```

**Step 2 — Open PR on GitHub:**

- Go to your repository on GitHub
- GitHub shows a yellow banner: "feature/add-backup-script had recent pushes"
- Click "Compare & pull request"
- Add title: "Add automated home directory backup script"
- Add description: explain what it does and why
- Click "Create pull request"

**Step 3 — Review and Merge:**

- Team members review the code
- They can add comments on specific lines
- They approve or request changes
- Once approved — click "Merge pull request"
- Click "Delete branch" — clean up

---

### Writing a Good PR Description

Bad description:
```
Fixed the thing
```

Good description:
```
## What this does
Adds an automated backup script that creates a compressed tar archive
of the home directory daily.

## Why
Currently we have no automated backups. This prevents data loss.

## How to test
Run ./backup.sh manually — verify a .tar.gz file appears in /backup/

## Related
Closes #45 — feature request for automated backups
```

> Good PR descriptions get approved faster and show professional maturity.

---

### PR Review Best Practices

**As the reviewer:**
- Check the code does what the description says
- Look for security issues (hardcoded passwords, wrong permissions)
- Check error handling — what happens when it fails?
- Verify naming is clear and consistent
- Test it yourself if possible

**As the author:**
- Keep PRs small — one feature or fix per PR
- Respond to all comments
- Never merge your own PR if team policy requires review
- Update the PR if changes are requested

---

### Forking — Contributing to Other Projects

**What is Forking?**

Forking creates **your own copy of someone else's repository** on GitHub. Used when you don't have write access to the original repo.

```
Original repo:    github.com/company/project
Your fork:        github.com/varun/project (your copy)
Your local clone: ~/project (on your machine)
```

```bash
# 1. Fork on GitHub — click Fork button on the repo page

# 2. Clone YOUR fork:
git clone git@github.com:varun/project.git

# 3. Add original repo as upstream:
git remote add upstream git@github.com:company/project.git

# 4. Verify remotes:
git remote -v
# origin    git@github.com:varun/project.git   (your fork)
# upstream  git@github.com:company/project.git (original)

# 5. Create feature branch and make changes:
git switch -c fix-typo-in-docs

# 6. Push to YOUR fork:
git push origin fix-typo-in-docs

# 7. Open PR from your fork to original repo on GitHub
```

**Keeping Your Fork Updated:**

```bash
git fetch upstream         # get changes from original
git switch main
git merge upstream/main    # update your main
git push origin main       # update your fork on GitHub
```

---

### Real DevOps GitHub Workflows

**Workflow 1 — Solo DevOps Engineer**

```bash
# Daily workflow:
git pull                                  # start with latest code
git switch -c feature/add-monitoring      # new branch for each task
# ... make changes ...
git add .
git commit -m "Add disk monitoring cron job"
git push -u origin feature/add-monitoring
# Open PR on GitHub → merge → delete branch
git switch main
git pull                                  # get the merged changes
git branch -d feature/add-monitoring      # clean up local
```

**Workflow 2 — Team Environment**

```bash
# Before starting work:
git fetch origin
git log --oneline HEAD..origin/main   # see what teammates pushed
git pull --rebase                     # update with clean history

# Do your work:
git switch -c bugfix/fix-deploy-script
# ... make fixes ...
git add .
git commit -m "Fix deploy script — handle missing env variable"
git push -u origin bugfix/fix-deploy-script

# Open PR → get review → address feedback → merge
```

**Workflow 3 — Hotfix on Production**

```bash
# Critical bug in production — urgent fix needed
git switch main
git pull                          # get latest production code
git switch -c hotfix/fix-auth-crash

# Fix the bug
vim auth.sh
git add auth.sh
git commit -m "Fix auth crash when DB_PASSWORD not set"

git push -u origin hotfix/fix-auth-crash
# Open PR — get emergency review
# Merge immediately
# Deploy to production

git switch main
git pull
git branch -d hotfix/fix-auth-crash
```

**Workflow 4 — CI/CD Trigger on Push**

This is how GitHub Actions CI/CD works:

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]   # triggers when code pushed to main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ./deploy.sh
```

> Every time you merge a PR to main — GitHub automatically runs your deployment script. This is the core of DevOps automation. You will set this up fully when you reach CI/CD in your roadmap.

---

### Common GitHub Situations and Solutions

**Situation 1 — Remote Has Changes You Don't Have**

```bash
git push origin main
# Error: rejected — remote contains work not in local

# Fix:
git pull --rebase origin main   # bring in remote changes first
git push origin main            # now push works
```

**Situation 2 — Wrong Commit on Wrong Branch**

```bash
# You committed to main instead of feature branch
git log --oneline    # find the commit SHA — e.g. a1b2c3d

# Create feature branch with that commit
git switch -c feature/my-change

# Remove commit from main (move pointer back)
git switch main
git reset HEAD~1    # undo last commit — keeps changes in working dir
```

**Situation 3 — Need to Update PR with New Changes**

```bash
# Reviewer requested changes on your PR
git switch feature/add-monitoring   # go back to feature branch
# ... make the requested changes ...
git add .
git commit -m "Address review feedback — add error handling"
git push origin feature/add-monitoring   # PR updates automatically
```

**Situation 4 — Sync Feature Branch with Latest Main**

```bash
# Main has moved while you were working on feature branch
git switch main
git pull                    # update main
git switch feature/my-feature
git merge main               # bring main changes into feature
# or
git rebase main               # cleaner — replays your commits on top of main
```

---

### Summary — Git Day 3

| Command | What it does |
|---------|---------------|
| `git remote add origin URL` | Connect local repo to GitHub |
| `git remote -v` | Show all remotes |
| `git push -u origin main` | Push and set upstream tracking |
| `git push` | Push current branch (after upstream set) |
| `git push origin --all` | Push all branches |
| `git clone URL` | Download entire repository |
| `git clone --depth 1 URL` | Shallow clone — latest only |
| `git fetch origin` | Download changes without merging |
| `git pull` | Fetch and merge |
| `git pull --rebase` | Fetch and rebase — cleaner history |
| `git branch -a` | Show all branches including remote |
| `git branch -vv` | Show tracking relationships |
| `git remote add upstream URL` | Add original repo (for forks) |

---

### Interview Questions — Git Day 3

**Q1. What is the difference between `git fetch` and `git pull`?**
`git fetch` downloads changes from remote but does not merge them — you can review what changed before integrating. `git pull` does both fetch and merge in one step. `git fetch` is safer for production workflows — inspect first, merge when confident.

**Q2. What is a Pull Request?**
A Pull Request is a request to merge one branch into another with a code review step. Team members review the code, leave comments, and approve before it merges. PRs ensure code quality, catch bugs, and provide a discussion forum for changes. Core to all professional team workflows.

**Q3. What is the difference between `origin` and `upstream`?**
`origin` is your own remote repository — where you push your changes. `upstream` is the original repository you forked from — where you pull updates from. In a fork workflow: you pull from upstream to stay updated, push to origin for your changes, and open a PR from origin to upstream.

**Q4. What does `git clone --depth 1` do?**
Creates a shallow clone — downloads only the latest commit without the full history. Much faster and uses less disk space. Useful in CI/CD pipelines where you only need current code, not the entire git history.

**Q5. What is `git push --force-with-lease` and when would you use it?**
A safer version of force push — it checks that nobody else has pushed to the remote branch since your last fetch before overwriting. Used after rebasing a feature branch to update the remote. Never use on shared branches like main.

**Q6. Your push is rejected because remote has changes you don't have. What do you do?**
Run `git pull --rebase origin main` to bring in remote changes and replay your commits on top, then `git push`. The `--rebase` flag keeps history clean by avoiding an unnecessary merge commit.

**Q7. What is forking and when do you use it?**
Forking creates your own copy of someone else's repository on GitHub. Used when contributing to projects you don't have write access to — open source projects, company repos as a contractor. You push to your fork and open a PR to the original repository.

**Q8. How does GitHub integrate with CI/CD?**
GitHub Actions (or Jenkins, CircleCI, etc.) monitors repository events. When you push to main or merge a PR, it triggers automated workflows — running tests, building Docker images, deploying to servers. This is the core automation in DevOps — a code change automatically flows to production through the pipeline.

---

### Homework — Before Git Day 4

1. Create a GitHub account if you don't have one
2. Set up SSH key authentication with GitHub
3. Create a new repository on GitHub called `devops-scripts`
4. Push your local `git_practice` repo to GitHub
5. Create a feature branch, push it to GitHub
6. Open a Pull Request on GitHub — even if it is just for practice
7. Clone a public repository — try: `git clone git@github.com:torvalds/linux.git --depth 1`
8. Practice `git fetch` — make a change on GitHub directly (edit a file on the GitHub website), then fetch and see it locally

> Git Day 4 covers Advanced Git — rebase, stash, cherry-pick, reset, tags, and the commands that senior engineers use daily. This is what separates junior from senior Git knowledge in interviews.
