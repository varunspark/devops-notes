## Git Day 4 — Advanced Git

---

### Why This Matters for 10-12 LPA Roles

Days 1-3 covered what every developer knows. Today covers what separates junior from senior Git knowledge.

At a 10-12 LPA level, interviewers expect you to know:

- Why rebase is preferred over merge in many teams
- How to recover from mistakes without panicking
- How to manage releases with tags
- How to move specific commits between branches

> These commands also come up in real daily work constantly.

---

### What You Will Learn Today

- `git stash` — save work without committing
- `git rebase` — rewrite history cleanly
- `git cherry-pick` — take specific commits
- `git reset` — undo commits
- `git revert` — safe undo for shared branches
- `git reflog` — recover anything
- `git tag` — mark releases
- `git bisect` — find which commit broke something
- `.git` internals — what is actually happening
- Git hooks — automation at every step

---

### `git stash` — Save Work Without Committing

**The Problem**

You are halfway through fixing a bug. Your manager calls — a critical issue on production needs immediate attention. You cannot commit half-done work. You cannot lose your changes.

> `git stash` saves your work temporarily so you can switch context.

**Basic Stash Commands:**

```bash
git stash                          # save all uncommitted changes
git stash push -m "half-done login fix"   # save with a description
git stash list                     # see all stashes
git stash pop                      # restore latest stash AND remove from list
git stash apply                    # restore latest stash BUT keep in list
git stash apply stash@{2}          # restore specific stash by index
git stash drop stash@{0}           # delete specific stash
git stash clear                    # delete ALL stashes
git stash show -p                  # show what is in the stash
```

**`stash pop` vs `stash apply`:**

```bash
git stash pop      # restore AND remove from stash list — use when done
git stash apply    # restore BUT keep in stash list — use if you want to
                    # apply same changes to multiple branches
```

**Real Scenario — Interrupted Work:**

```bash
# You are working on feature
vim deploy.sh    # half done changes
# Manager calls — production is down!

git stash push -m "deploy script — half done"
git switch main
git switch -c hotfix/production-crash

# Fix the production issue
vim auth.sh
git add auth.sh
git commit -m "Fix auth crash on missing token"
git push origin hotfix/production-crash
# PR merged — production fixed

# Back to your original work
git switch feature/deploy-improvements
git stash pop    # your half-done work is back
# Continue where you left off
```

**Stashing Untracked Files:**

```bash
git stash      # only stashes tracked files
git stash -u   # also stashes untracked new files
git stash -a   # stashes everything including .gitignored files
```

> ⚠️ **Stash is local only.** Stashes are stored in `.git/refs/stash` — they are not pushed to GitHub. If you delete the repo, stashes are lost too.

**Interview Question**

**Q. What is `git stash` and when would you use it?**
`git stash` temporarily saves uncommitted changes so you can switch to another branch (e.g. for an urgent hotfix) without committing incomplete work. `git stash pop` restores the work when you return.

---

### `git rebase` — Rewrite History Cleanly

**The Problem With Merge**

When you merge main into your feature branch repeatedly, you get a messy history full of merge commits.

```
feature: A---B---C---M---D---E---M---F
              |               |
main:         A---B---C---G---H---I
```

`M` commits are merge commits — they clutter the history and make it hard to understand what actually happened.

**What Rebase Does**

Rebase **replays your commits on top of another branch** — as if you started your feature branch from the latest main.

```
Before rebase:
main:    A---B---C---D---E
                          \
feature:                   F---G

After: git rebase main (from feature branch)
main:    A---B---C---D---E
                          \
feature:                   F'---G'
```

`F` and `G` are replayed as `F'` and `G'` on top of `E`. Clean linear history. No merge commits.

**Basic Rebase:**

```bash
# Update your feature branch with latest main — cleanly:
git switch feature/my-feature
git rebase main

# If conflicts during rebase:
# Fix the conflict in the file
git add conflicting-file.txt
git rebase --continue    # continue to next commit

# If you want to give up:
git rebase --abort       # goes back to state before rebase started
```

**Merge vs Rebase — Side by Side:**

```bash
# Merge approach — preserves full history:
git switch feature
git merge main
# Creates merge commit — non-destructive, safer

# Rebase approach — clean linear history:
git switch feature
git rebase main
# Rewrites commits — cleaner but changes commit SHAs
```

| Merge | Rebase |
|-------|--------|
| Preserves complete history | Creates clean linear history |
| Safe on shared branches | Never rebase shared branches |
| Creates merge commits | No merge commits |

### The Golden Rule of Rebase

> **Never rebase commits that exist on a public/shared branch.**

When you rebase, commits get new SHAs. If someone else pulled the old SHAs, their history now conflicts badly.

```bash
# SAFE — rebasing your local feature branch:
git switch feature/my-feature
git rebase main    # OK — nobody else has this branch

# DANGEROUS — rebasing main:
git switch main
git rebase feature  # NEVER DO THIS — main is shared
```

---

### Interactive Rebase — Powerful Cleanup

```bash
git rebase -i HEAD~3    # interactively edit last 3 commits
git rebase -i HEAD~5    # interactively edit last 5 commits
```

Opens an editor showing your recent commits:

```
pick a1b2c3d Add monitoring script
pick d4e5f6g Fix typo in monitoring script
pick h7i8j9k Add another typo fix
```

Change the word `pick` to:

| Keyword | Effect |
|---------|--------|
| `pick` | keep commit as-is |
| `reword` | keep commit but edit the message |
| `edit` | pause and amend this commit |
| `squash` | combine with previous commit |
| `fixup` | combine with previous — discard this commit message |
| `drop` | delete this commit entirely |

### Squashing Commits — Very Common in Teams

Many teams require you to squash all your feature commits into one clean commit before merging.

```
Before squash:
d4e5f6g  Fix typo
a1b2c3d  Fix another typo
z9y8x7w  Actually fix the bug
h7i8j9k  Add monitoring script

After squash into one:
a1b2c3d  Add nginx monitoring script with error handling
```

```bash
git rebase -i HEAD~4    # edit last 4 commits

# In the editor:
pick   h7i8j9k Add monitoring script
squash z9y8x7w Actually fix the bug
squash a1b2c3d Fix another typo
squash d4e5f6g Fix typo

# Save — Git combines all into one commit
# Edit the final commit message to be clean and clear
```

**Interview Question**

**Q. What is the difference between `git merge` and `git rebase`?**
Both integrate changes from one branch into another. Merge preserves complete history by creating a merge commit — safe and non-destructive. Rebase replays commits on top of the target branch, creating a clean linear history but rewriting SHAs. Never rebase commits on shared branches that others have pulled.

---

### `git cherry-pick` — Take Specific Commits

**What is Cherry-Pick?**

Cherry-pick applies a **specific commit from any branch** onto your current branch.

```
main:    A---B---C---D---E
                          \
feature:                   F---G---H

# You want only commit G on main — not F or H
git switch main
git cherry-pick G_SHA

main: A---B---C---D---E---G'
```

```bash
# Find the SHA of the commit you want:
git log --oneline feature-branch

# Apply it to current branch:
git cherry-pick a1b2c3d              # apply one commit
git cherry-pick a1b2c3d d4e5f6g      # apply multiple commits
git cherry-pick a1b2c3d..h7i8j9k     # apply a range of commits
git cherry-pick -n a1b2c3d           # apply changes but don't commit yet
```

**Real DevOps Scenario — Backport a Fix**

You have two branches: `main` (latest) and `release-v1` (older version still in production). You fixed a critical security bug on `main`. You need that same fix on `release-v1` without merging all other changes.

```bash
# On main — find the fix commit SHA
git log --oneline main
# a1b2c3d  Fix SQL injection in user auth

# Switch to release branch
git switch release-v1

# Cherry-pick just that fix
git cherry-pick a1b2c3d

# Push to deploy the fix
git push origin release-v1
```

> This is called **backporting** — applying a fix to older releases. Very common in real DevOps work.

**Cherry-Pick Conflicts:**

```bash
git cherry-pick a1b2c3d
# CONFLICT — same as merge conflicts

# Fix the conflict in the file
git add conflicting-file.txt
git cherry-pick --continue    # apply the cherry-pick with fix
# or
git cherry-pick --abort       # cancel and go back
```

**Interview Question**

**Q. What is `git cherry-pick` and when would you use it?**
Cherry-pick applies a specific commit from one branch onto another without merging the entire branch. Used for backporting — applying a critical fix to an older release branch — or when you accidentally committed to the wrong branch.

---

### `git reset` — Undo Commits

**What is Reset?**

Reset moves the branch pointer backwards — effectively undoing commits.

```
Before reset:
A---B---C---D  (HEAD, main)

After git reset HEAD~2:
A---B  (HEAD, main)
C and D are gone from branch history
```

### Three Types of Reset

| Flag | Commit | Staging Area | Working Dir |
|------|--------|---------------|--------------|
| `--soft` | Undone | Changes kept staged | Unchanged |
| `--mixed` (default) | Undone | Cleared | Changes kept |
| `--hard` | Undone | Cleared | Changes DELETED |

```bash
git reset --soft HEAD~1    # undo commit — keep changes STAGED
git reset --mixed HEAD~1   # undo commit — keep changes in WORKING DIR (default)
git reset --hard HEAD~1    # undo commit — DELETE changes completely
```

**When to Use Each:**

```bash
# --soft: undo commit but keep changes staged — want to recommit differently
git reset --soft HEAD~1
# Now files are staged — just commit again with a better message

# --mixed (default): undo commit, unstage changes — most common
git reset HEAD~1
# Changes are in working dir — review, re-stage what you want

# --hard: completely undo — discard all changes
git reset --hard HEAD~1
# Changes are GONE — use with extreme caution
```

**Reset with Specific SHA:**

```bash
git log --oneline
# a1b2c3d  Latest feature
# d4e5f6g  Good stable state
# h7i8j9k  Initial commit

# Reset to specific point:
git reset --hard d4e5f6g
# Now at d4e5f6g — everything after it is gone from history
```

### Reset vs Revert

```bash
# reset — rewrites history — NEVER use on shared branches
git reset HEAD~1

# revert — creates new commit that undoes changes — SAFE for shared branches
git revert HEAD
```

---

### `git revert` — Safe Undo for Shared Branches

**What is Revert?**

Revert creates a **new commit that undoes a previous commit.** History is preserved.

```
Before revert:
A---B---C---D  (bad commit D)

After git revert D:
A---B---C---D---D'
D' undoes what D did — D still exists in history
```

```bash
git revert HEAD              # revert last commit
git revert a1b2c3d           # revert specific commit
git revert HEAD~3..HEAD      # revert last 3 commits
git revert --no-commit HEAD  # stage the revert without committing
```

> **Rule: Reset for local, Revert for shared/remote.**

**Real Scenario — Bad Deployment:**

```bash
# Bad code was merged and pushed to main
git log --oneline
# a1b2c3d  Deploy broken feature  ← this broke production
# d4e5f6g  Previous stable release

# Cannot reset — other people have pulled main
# Use revert:
git revert a1b2c3d
# Creates new commit that undoes the broken changes
git push origin main   # production is fixed, history preserved
```

```bash
# Use reset when:
# → You have NOT pushed yet — local only
# → You want to completely undo
git reset --hard HEAD~1

# Use revert when:
# → You HAVE pushed to a shared branch
# → You need to undo while preserving history
git revert a1b2c3d
git push origin main   # safe — history preserved
```

**Interview Question**

**Q. What is the difference between `git reset` and `git revert`?**
`git reset` moves the branch pointer backwards, rewriting history — only safe for local commits not yet pushed. `git revert` creates a new commit that undoes the changes of a previous commit while preserving history — safe for shared branches that others have pulled. Rule: reset locally, revert remotely.

---

### `git reflog` — Recover Anything

**What is Reflog?**

Reflog records every movement of HEAD — every commit, checkout, reset, merge, rebase. It is Git's safety net. Even after `git reset --hard`, your work is recoverable with reflog.

**Recovering Lost Commits:**

```bash
git reflog              # show all HEAD movements
git reflog show main    # show movements of main branch
```

Output:
```
a1b2c3d  HEAD@{0}: commit: Add monitoring script
d4e5f6g  HEAD@{1}: checkout: moving from main to feature
h7i8j9k  HEAD@{2}: reset: moving to HEAD~1
z9y8x7w  HEAD@{3}: commit: Previous good commit
```

```bash
# You ran git reset --hard and lost your work
git reset --hard HEAD~3   # oops — lost 3 commits

# Find the lost commits in reflog:
git reflog
# a1b2c3d  HEAD@{0}: reset: moving to HEAD~3
# d4e5f6g  HEAD@{1}: commit: Lost commit 3  ← this is what you want

# Recover by creating branch at that point:
git switch -c recovery-branch d4e5f6g

# Or reset back to that point:
git reset --hard d4e5f6g   # back to before the accidental reset
```

> **Reflog Expiry:** Reflog entries expire after 90 days by default. So you have 90 days to recover anything.

**Interview Question**

**Q. How do you recover commits after `git reset --hard`?**
Using `git reflog` — it records every HEAD movement including resets. Find the SHA of your lost commits, then `git reset --hard that_SHA` to restore, or `git switch -c recovery that_SHA` to recover on a new branch. Works within 90 days before reflog entries expire.

---

### `git tag` — Marking Releases

**What are Tags?**

Tags mark specific commits as important — usually software releases. Unlike branches, tags don't move.

```
main:  A---B---C---D---E---F
              |           |
            v1.0.0      v1.1.0
```

### Two Types of Tags

```bash
# Lightweight tag — just a pointer to commit:
git tag v1.0.0

# Annotated tag — has metadata — PREFERRED for releases:
git tag -a v1.0.0 -m "Release version 1.0.0 — initial production release"
```

> Always use annotated tags for releases — they contain tagger name, date, and message.

**Tag Commands:**

```bash
git tag                              # list all tags
git tag -l "v1.*"                    # list tags matching pattern
git tag -a v1.0.0 -m "Release v1.0"  # create annotated tag
git tag -a v1.0.0 a1b2c3d -m "msg"   # tag specific past commit
git show v1.0.0                      # show tag details
git push origin v1.0.0               # push specific tag to GitHub
git push origin --tags               # push all tags to GitHub
git tag -d v1.0.0                    # delete tag locally
git push origin --delete v1.0.0      # delete tag on remote
```

### Semantic Versioning — What the Numbers Mean

```
v1.2.3
 | | |
 | | └──── Patch — bug fixes only (backwards compatible)
 | └────── Minor — new features (backwards compatible)
 └──────── Major — breaking changes (not backwards compatible)
```

```bash
git tag -a v1.0.0 -m "First stable release"
git tag -a v1.0.1 -m "Fix login crash bug"        # patch
git tag -a v1.1.0 -m "Add monitoring dashboard"   # minor feature
git tag -a v2.0.0 -m "Rewrite authentication"     # major — breaking
```

### Checking Out a Tag

```bash
git checkout v1.0.0    # go to exact state of v1.0.0
# You are in detached HEAD state — not on any branch
# To make changes:
git switch -c fix-v1.0.0   # create branch from this tag
```

### Real DevOps Use

```bash
# Release workflow:
git switch main
git pull

# Run tests — all passing
# Create release tag
git tag -a v2.1.0 -m "Release v2.1.0 — add backup automation"
git push origin v2.1.0

# CI/CD pipeline detects new tag → automatically builds Docker image
# → deploys to production
```

> Tags trigger CI/CD pipelines in most setups. When you push a `v*` tag, the pipeline automatically builds and deploys. You will implement this when you reach CI/CD.

---

### `git bisect` — Find the Breaking Commit

**What is Bisect?**

Bisect uses binary search to find which commit introduced a bug. Instead of checking 100 commits one by one, bisect narrows it down in about 7 steps.

**How Bisect Works:**

```bash
git bisect start         # start bisect session
git bisect bad           # current commit is broken
git bisect good v1.0.0   # v1.0.0 was working

# Git checks out a commit halfway between bad and good
# You test — is this commit good or bad?
git bisect good   # this commit works
# or
git bisect bad    # this commit is broken

# Git narrows down — repeat until found:
# Output: a1b2c3d is the first bad commit

git bisect reset   # exit bisect — back to original branch
```

**Automated Bisect:**

```bash
# If you have a test script:
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh   # runs test automatically on each commit
# Git finds the bad commit automatically
git bisect reset
```

**Real Scenario:**

Production broke sometime in the last week. You have 50 commits. Bisect finds the culprit in about 6 steps instead of 50.

---

### `.git` Internals — What is Actually Happening

**Understanding What Git Stores:**

```bash
ls .git/
```

```
HEAD            # pointer to current branch
config          # repo config
objects/        # ALL your file content stored here
refs/           # branches and tags
hooks/          # scripts triggered by git events
COMMIT_EDITMSG  # last commit message
index           # staging area data
```

**Git Object Types:**

```bash
# 4 types of objects:
blob    # file content
tree    # directory structure
commit  # commit metadata + pointer to tree
tag     # annotated tag object

# See any object:
git cat-file -t a1b2c3d   # type of object
git cat-file -p a1b2c3d   # contents of object
```

---

### Git Hooks — Automation at Every Step

Hooks are scripts in `.git/hooks/` that run automatically on git events:

```bash
ls .git/hooks/
# pre-commit    — runs before every commit
# commit-msg    — validates commit message format
# pre-push      — runs before push
# post-merge    — runs after merge
# post-checkout — runs after checkout
```

**Example — Enforce Commit Message Format:**

```bash
vim .git/hooks/commit-msg
```

```bash
#!/bin/bash
# Enforce conventional commits format
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore): .+"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "ERROR: Commit message must follow format:"
  echo "  feat: add new feature"
  echo "  fix: fix a bug"
  echo "  docs: update documentation"
  exit 1
fi
```

```bash
chmod +x .git/hooks/commit-msg

# Now bad commit messages are rejected:
git commit -m "stuff"
# ERROR: Commit message must follow format
```

**Example — Run Tests Before Push:**

```bash
vim .git/hooks/pre-push
```

```bash
#!/bin/bash
echo "Running tests before push..."
./run_tests.sh
if [ $? -ne 0 ]; then
  echo "Tests failed — push blocked"
  exit 1
fi
echo "Tests passed — pushing"
```

```bash
chmod +x .git/hooks/pre-push
```

> Now broken code can never be pushed — tests must pass first. This is basic CI enforcement at the local level.

---

### Complete Advanced Git Cheat Sheet

```bash
# Stash
git stash
git stash push -m "description"
git stash list
git stash pop
git stash apply stash@{1}
git stash clear

# Rebase
git rebase main
git rebase -i HEAD~3
git rebase --continue
git rebase --abort

# Cherry-Pick
git cherry-pick SHA
git cherry-pick SHA1 SHA2
git cherry-pick SHA1..SHA2
git cherry-pick --abort

# Reset and Revert
git reset --soft HEAD~1
git reset HEAD~1
git reset --hard HEAD~1
git revert HEAD
git revert SHA

# Reflog and Recovery
git reflog
git reset --hard SHA
git switch -c recovery SHA

# Tags
git tag -a v1.0.0 -m "msg"
git push origin --tags
git tag -d v1.0.0
git push origin --delete v1.0.0
```

---

### Full Interview Questions — Git Day 4

**Q1. What is `git stash` and when would you use it?**
`git stash` temporarily saves uncommitted changes so you can switch branches without committing incomplete work. Used when you need to urgently switch context — like a production hotfix interrupting feature work. `git stash pop` restores the work when you return.

**Q2. What is the difference between `git merge` and `git rebase`?**
Merge integrates branches by creating a merge commit, preserving full history — safe and non-destructive. Rebase replays commits on top of the target branch, creating a clean linear history but rewriting SHAs. Never rebase commits on shared branches that others have pulled.

**Q3. What is interactive rebase used for?**
`git rebase -i HEAD~N` lets you edit, reorder, squash, or drop recent commits before pushing. Used to clean up messy commit history — squash multiple small commits into one clean commit, fix commit messages, remove debug commits before opening a PR.

**Q4. What is `git cherry-pick`?**
Applies a specific commit from one branch onto another without merging the entire branch. Used for backporting — applying a critical fix to an older release branch. Also used when you accidentally committed to the wrong branch.

**Q5. What is the difference between `git reset` and `git revert`?**
`git reset` moves the branch pointer backwards, rewriting history — only safe for local commits not pushed. `git revert` creates a new commit undoing changes while preserving history — safe for shared branches. Rule: reset locally, revert remotely.

**Q6. How do you recover commits after `git reset --hard`?**
Using `git reflog` — it records every HEAD movement. Find the SHA of your lost commits, then `git reset --hard that_SHA` to restore, or `git switch -c recovery that_SHA` to recover on a new branch. Works within 90 days before reflog entries expire.

**Q7. What are Git tags, and what is the difference between lightweight and annotated?**
Tags mark specific commits permanently — usually for releases. Lightweight tags are just pointers to commits. Annotated tags contain metadata — tagger name, date, message — and are preferred for official releases. Use semantic versioning: `v1.2.3` — major.minor.patch.

**Q8. What are Git hooks?**
Scripts in `.git/hooks/` that run automatically on git events. `pre-commit` runs before every commit, `commit-msg` validates commit message format, `pre-push` runs before pushing. Used to enforce code standards, run tests, and prevent bad commits locally before they reach the remote.

**Q9. What is `git bisect`?**
Uses binary search to find which commit introduced a bug. You tell it a good commit and a bad commit — Git checks out the midpoint, you test and mark good or bad, repeating until the exact breaking commit is found. Reduces a 100-commit investigation to about 7 steps.

**Q10. What does `git reflog` show?**
Every movement of HEAD — every commit, checkout, reset, merge, rebase. It is Git's safety net. Even commits deleted with `git reset --hard` are recoverable through reflog within 90 days.

---

### Homework — Consolidate Everything

```bash
cd ~/linux_practice/git_practice
```

1. Make 3 messy commits with bad messages — then use `git rebase -i HEAD~3` to squash them
2. Create a stash, switch branches, come back and pop the stash
3. Find a commit SHA from `git log --oneline` and cherry-pick it to a new branch
4. Make a commit then `git reset --soft HEAD~1` — see that changes are still staged
5. Make a commit then `git revert HEAD` — see that it creates a new commit
6. Run `git reflog` — understand every line in the output
7. Create an annotated tag `v1.0.0` and push it to GitHub
8. Create a `pre-commit` hook that prevents commits with the message "WIP"

---

## 🎉 Git Training Complete!

```
Git Day 1 ✅ init, add, commit, log, .gitignore
Git Day 2 ✅ branches, merging, conflicts
Git Day 3 ✅ GitHub, push, pull, clone, PRs
Git Day 4 ✅ stash, rebase, cherry-pick, reset, revert, reflog, tags
```
