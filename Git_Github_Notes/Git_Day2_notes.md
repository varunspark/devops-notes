## Git Day 2 — Branches, Merging & Conflict Resolution

---

### Why This is Critical for DevOps

Imagine your team is working on a live production website. One developer is fixing a critical bug. Another is building a new feature. A third is updating the database schema.

If everyone works on the same code at the same time — they overwrite each other's changes. Everything breaks.

**Branches solve this.** Every developer works in their own isolated copy. When ready — they merge back. This is how every real company works.

> Branching and merging is also the most asked Git topic in DevOps interviews.

---

### What You Will Learn Today

- What a branch is
- Creating and switching branches
- The main/master branch
- Merging branches
- Fast-forward vs three-way merge
- Merge conflicts — what they are and how to fix them
- Deleting branches
- Real DevOps branching strategies
- Branch naming conventions

---

### What is a Branch?

A branch is an independent line of development. It is a pointer to a specific commit. When you create a branch, you get your own copy of the codebase to work in — without affecting anyone else.

```
main branch:    A --- B --- C
                       \
feature branch:         D --- E --- F
```

The main branch continues normally. The feature branch has its own commits. When the feature is done, merge F back into main.

### What is HEAD?

HEAD is a pointer that shows where you currently are in the repository — which branch and which commit you are on.

When you switch branches, HEAD moves. When you make a commit, HEAD moves forward.

```bash
cat .git/HEAD
# ref: refs/heads/main
# This means you are on the main branch
```

---

### Creating and Switching Branches

```bash
git branch          # list all local branches
git branch -a       # list all branches including remote
git branch -v       # list branches with last commit message

git branch feature-login   # create new branch
git switch feature-login    # switch to that branch
git switch main             # switch back to main

# Create AND switch in one command — most common:
git switch -c feature-login   # modern way — recommended
git checkout -b feature-login # older way — still works, same result
```

### `git switch` vs `git checkout`

`git switch` was introduced in Git 2.23 (2019) specifically for switching branches. `git checkout` is older and does multiple things — switching branches, restoring files, checking out commits. Both work, but `git switch` is clearer and recommended for switching branches.

```bash
git switch branch-name      # modern — only for switching branches
git checkout branch-name    # older — works but does more things
```

### Seeing Which Branch You Are On

```bash
git branch    # current branch has * in front
git status    # first line always shows current branch
```

Output of `git branch`:
```
* feature-login
  main
  bugfix-navbar
```

> The `*` shows you are on `feature-login`.

### Practice — Create Your First Branch

```bash
cd ~/linux_practice/git_practice

# Make sure you have some commits first
git log --oneline

# Create and switch to feature branch
git switch -c feature-monitoring

# Verify you are on the new branch
git branch
git status

# Make a change on this branch
echo "#!/bin/bash" > monitor.sh
echo "systemctl is-active nginx || sudo systemctl start nginx" >> monitor.sh

git add monitor.sh
git commit -m "Add nginx monitoring script"

# Your main branch does NOT have this file yet
git switch main
ls   # monitor.sh is NOT here — it only exists on feature-monitoring

git switch feature-monitoring
ls   # monitor.sh IS here — back on feature branch
```

> This demonstrates that branches are truly isolated. Your changes on one branch do not exist on others until you merge.

---

### Merging Branches

**What is Merging?**

Merging takes the changes from one branch and combines them into another branch.

**Standard workflow:**
1. Finish work on feature branch
2. Switch to main branch
3. Merge feature branch into main
4. Delete feature branch

```bash
git switch main
git merge feature-monitoring
```

---

### Two Types of Merges

#### Type 1 — Fast-Forward Merge

Happens when main has not moved since you created your branch. Git simply moves the pointer forward. No merge commit created — clean, linear history.

```
Before merge:
main:     A --- B
                 \
feature:          C --- D

After fast-forward merge:
main: A --- B --- C --- D
```

```bash
git switch main
git merge feature-monitoring
# Output: Fast-forward
```

#### Type 2 — Three-Way Merge

Happens when both branches have new commits since the branch was created. Git creates a new merge commit combining both histories.

```
Before merge:
main:    A --- B --- C
                       \
feature:                D --- E

After three-way merge:
main:    A --- B --- C ------ M
                       \      /
feature:                D --- E
```

`M` is the merge commit — it has two parents.

```bash
git switch main
git merge feature-login
# Output: Merge made by the 'recursive' strategy.
# A new merge commit is created automatically
```

> When a three-way merge happens, Git opens your editor asking for a merge commit message. The default message is fine — just save and quit:
> - In vim — type `:wq` to accept default message
> - In nano — `Ctrl+X` then `Y` then `Enter`

---

### Merge Conflicts — The Most Important Part

**What is a Merge Conflict?**

A conflict happens when **both branches changed the same part of the same file.** Git cannot decide which version to keep.

```
main branch:    line 5 = "port 8080"
feature branch: line 5 = "port 9090"
```

When you merge — Git says: CONFLICT — which one do you want?

```bash
git switch main
git merge feature-branch
# Output:
# Auto-merging config.yml
# CONFLICT (content): Merge conflict in config.yml
# Automatic merge failed; fix conflicts and then commit the result.
```

**What a Conflict Looks Like Inside the File:**

```bash
cat config.yml
```
```
server_name = "myapp"
<<<<<<< HEAD
port = 8080
=======
port = 9090
>>>>>>> feature-branch
database_host = "localhost"
```

| Part | Meaning |
|------|---------|
| `<<<<<<< HEAD` | Start of YOUR version (current branch) |
| `=======` | Divider between the two versions |
| `>>>>>>> feature-branch` | End of THEIR version (branch being merged) |

### Resolving a Conflict — Step by Step

**Step 1 — Open the conflicting file**

```bash
vim config.yml
```

**Step 2 — Decide which version to keep.** You have three choices:
- Keep your version (HEAD)
- Keep their version (feature-branch)
- Keep both / write something new

Edit the file to remove the conflict markers and leave only the final version:

```
# Remove all conflict markers and leave what you want:
server_name = "myapp"
port = 9090
database_host = "localhost"
```

**Step 3 — Stage the resolved file**

```bash
git add config.yml
```

**Step 4 — Complete the merge**

```bash
git commit -m "Merge feature-branch — use port 9090"
```

### Checking Conflict Status

```bash
git status    # shows files with conflicts under "both modified"
git diff      # shows conflict markers in files
```

### Aborting a Merge

If conflicts are too complex and you want to start over:

```bash
git merge --abort    # cancels merge and restores previous state
```

### Practice — Create and Resolve a Conflict

```bash
cd ~/linux_practice/git_practice

# Setup — create a file on main
git switch main
echo "port=8080" > app.conf
git add app.conf
git commit -m "Add app config with port 8080"

# Create branch and change same line
git switch -c fix-port
sed -i 's/port=8080/port=9090/' app.conf
git add app.conf
git commit -m "Change port to 9090"

# Switch to main and also change same line differently
git switch main
sed -i 's/port=8080/port=7777/' app.conf
git add app.conf
git commit -m "Change port to 7777"

# Now try to merge — CONFLICT will happen
git merge fix-port

# See the conflict
cat app.conf

# Resolve it — edit file, keep 9090
echo "port=9090" > app.conf
git add app.conf
git commit -m "Resolve port conflict — use 9090"

git log --oneline --graph
```

---

### Deleting Branches

**Cleaning Up After Merge:**

```bash
git branch -d feature-monitoring   # delete merged branch — safe
git branch -D feature-monitoring   # force delete — even if not merged
```

- `-d` (lowercase) only deletes if the branch has been fully merged. It protects you from accidentally deleting unmerged work.
- `-D` (uppercase) force deletes regardless. Use only when you are sure you do not need the branch.

**Best Practice — Always Delete After Merging:**

```bash
# Full workflow:
git switch main
git merge feature-login
git branch -d feature-login        # clean up immediately
git log --oneline --graph          # verify clean history
```

> Undeleted branches pile up and make `git branch` output messy. Delete them as soon as they're merged.

---

### Real DevOps Branching Strategies

#### Strategy 1 — GitHub Flow (Most Common for DevOps)

Simple and practical. Used by most modern teams:

```
main branch     = always deployable — production code
feature branches = all new work happens here
```

```bash
# Workflow:
git switch -c feature/add-monitoring   # create feature branch
# ... do work, make commits ...
git switch main
git merge feature/add-monitoring        # merge when done
git branch -d feature/add-monitoring    # clean up
git push origin main                    # deploy to production
```

#### Strategy 2 — GitFlow (Larger Teams)

More structured — used in companies with scheduled releases:

```
main      = production code — only release commits here
develop   = integration branch — all features merge here first
feature/* = individual features
release/* = release preparation
hotfix/*  = emergency production fixes
```

```bash
# Feature work:
git switch -c feature/new-dashboard develop
# ... work ...
git switch develop
git merge feature/new-dashboard

# Release:
git switch -c release/v1.2 develop
# ... testing, fixes ...
git switch main
git merge release/v1.2
git tag v1.2
```

> For a DevOps engineer starting out — **GitHub Flow** is what you will use and be asked about most.

#### Strategy 3 — Trunk Based Development

Used by large companies like Google and Facebook. Requires a very good CI/CD pipeline and testing.

```
main = everyone pushes here daily
# Features hidden with feature flags until ready
```

Everyone commits directly to main (trunk) frequently. Feature flags control what users see.

### Branch Naming Conventions

| Type | Format | Example |
|------|--------|---------|
| Feature | `feature/description` | `feature/add-monitoring` |
| Bug fix | `bugfix/description` | `bugfix/fix-login-error` |
| Hotfix | `hotfix/description` | `hotfix/critical-security-patch` |
| Release | `release/version` | `release/v1.2.0` |
| Experiment | `experiment/description` | `experiment/new-auth-system` |

> Always use lowercase and hyphens. No spaces. No special characters.

---

### Useful Branch Commands

**Seeing Branch History:**

```bash
git log --oneline --graph --all     # see ALL branches and their commits
git log --oneline main..feature     # commits in feature not in main
git log --oneline feature..main     # commits in main not in feature
```

**Comparing Branches:**

```bash
git diff main feature-branch              # all differences between branches
git diff main..feature-branch             # same with .. notation
git diff main feature-branch -- file.txt  # differences in specific file
```

**Checking if Branch is Merged:**

```bash
git branch --merged       # branches already merged into current branch
git branch --no-merged    # branches NOT yet merged — safe to check before delete
```

**Renaming a Branch:**

```bash
git branch -m old-name new-name   # rename branch
git branch -m new-name            # rename CURRENT branch
```

---

### Summary — Git Day 2

| Command | What it does |
|---------|---------------|
| `git branch` | List all branches |
| `git branch name` | Create new branch |
| `git switch name` | Switch to branch |
| `git switch -c name` | Create AND switch in one step |
| `git merge branch` | Merge branch into current |
| `git merge --abort` | Cancel merge in progress |
| `git branch -d name` | Delete merged branch safely |
| `git branch -D name` | Force delete branch |
| `git log --graph --all` | See all branches visually |
| `git diff main..feature` | Compare two branches |
| `git branch --merged` | Show merged branches |
| `git branch --no-merged` | Show unmerged branches |

---

### Interview Questions — Git Day 2

**Q1. What is a Git branch and why is it used?**
A branch is an independent line of development — a pointer to a specific commit. Used so multiple developers can work on different features simultaneously without affecting each other's code. Each developer works in isolation and merges when ready.

**Q2. What is the difference between fast-forward and three-way merge?**
Fast-forward happens when the target branch has not moved since the feature branch was created — Git simply moves the pointer forward, no merge commit created, clean linear history. Three-way merge happens when both branches have new commits — Git creates a new merge commit combining both histories.

**Q3. What is a merge conflict and how do you resolve it?**
A conflict occurs when both branches changed the same part of the same file. Git marks the conflicting section with conflict markers. To resolve: open the file, remove the markers and edit to keep the correct version, then `git add` the file and `git commit` to complete the merge.

**Q4. What is HEAD in Git?**
HEAD is a pointer showing where you currently are in the repository — which branch and commit you are on. When you switch branches, HEAD moves. When you commit, HEAD moves forward. `cat .git/HEAD` shows where HEAD points.

**Q5. What is the difference between `git branch -d` and `git branch -D`?**
`-d` (lowercase) safely deletes a branch only if it has been fully merged — protects against losing work. `-D` (uppercase) force deletes regardless of merge status. Always use `-d` first and only use `-D` when certain the work is not needed.

**Q6. Explain GitHub Flow branching strategy.**
Main branch always contains deployable production code. All new work happens on feature branches. When a feature is complete, it is merged into main and the feature branch is deleted. Simple, practical, and used by most modern DevOps teams.

**Q7. What happens to your changes when you switch branches?**
If changes are committed, they stay on their branch and disappear when you switch. If changes are not committed, Git either carries them over or refuses to switch if there is a conflict — preventing data loss.

**Q8. How do you see the commit history across all branches?**
`git log --oneline --graph --all` — shows all branches and their commits in a visual graph format.

---

### Homework — Before Git Day 3

```bash
cd ~/linux_practice/git_practice
```

1. Create a branch called `feature/disk-monitor`
2. Add a script `disk_monitor.sh` with content from Day 9
3. Commit it with a good message
4. Switch back to main and create another branch `feature/log-cleaner`
5. Add a script `log_cleaner.sh` that deletes logs older than 30 days
6. Commit it
7. Merge `feature/disk-monitor` into main
8. Merge `feature/log-cleaner` into main
9. Delete both feature branches
10. Run `git log --oneline --graph` — see your merge history

> Git Day 3 covers GitHub, Remote Repositories, Push, Pull, and Pull Requests — this is where Git goes from local to team collaboration. Every DevOps workflow depends on this.
