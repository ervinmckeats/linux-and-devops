# Lesson 31. Git for operations

## Goal
Keep scripts, configs and infrastructure code in Git, work through branches and pull requests, and recover from mistakes.

## Why ops people live in Git

- Every change has an author, a time and a reason. "Who changed the nginx config?" is one command.
- Rollback is `git revert`, not "I think it was like this before".
- CI/CD, GitOps and IaC (Module 8–9) all start from a Git repository.

## Theory

- A **commit** is a snapshot of all tracked files + author + message + parent commit(s). Its ID is a hash of all that.
- A **branch** is a movable pointer to a commit. `HEAD` points to the branch you are on.
- Three places: **working tree** (your files) → **index/staging** (`git add`) → **repository** (`git commit`).
- A **remote** (`origin`) is another copy of the repo, usually on GitHub/GitLab.

## Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
```

## Daily commands

```bash
git clone git@github.com:org/infra.git
git status

# new branch
git switch -c fix/nginx-timeout

git add nginx/app.conf

# review what you commit
git diff --staged

git commit -m "fix: raise proxy timeout to 60s"
git push -u origin fix/nginx-timeout

# update
git switch main && git pull

git log --oneline --graph -20

# history of one file with diffs
git log -p -- nginx/app.conf

# who changed each line
git blame nginx/app.conf

git show a1b2c3d
```

## Branches and pull requests

Typical flow (trunk-based, small changes):

1. Branch from `main`.
2. Small, focused commits.
3. Push, open a pull request (PR / merge request).
4. CI runs checks (lint, tests, `terraform plan`).
5. Review, merge (often squash), delete the branch.
6. Deploy from `main`.

Protect `main`: no direct pushes, required reviews and checks.

### Merge vs rebase

```bash
# adds a merge commit, keeps history as it happened
git merge main

# replays your commits on top of main: linear history
git rebase main
```

Rule: rebase your **own, unpushed or private** branch; never rewrite history that others already use.

### Conflicts

```bash
# CONFLICT in app.conf
git pull

# edit the file, remove <<<<<<< ======= >>>>>>> markers
git add app.conf

# or git commit (for a merge)
git rebase --continue

# give up, back to before
git rebase --abort
```

## Undo

| Situation | Command |
|---|---|
| Discard changes in a file | `git restore file` |
| Unstage a file | `git restore --staged file` |
| Fix the last commit (not pushed) | `git commit --amend` |
| Undo a pushed commit safely | `git revert <sha>` (new commit that reverses it) |
| Move branch back, keep changes | `git reset --soft HEAD~1` |
| Lost a commit after reset | `git reflog` then `git switch -c rescue <sha>` |
| Park work for later | `git stash`, `git stash pop` |

`git reset --hard` and `git push --force` destroy work. Prefer `revert` on shared branches; if you must force-push your own branch, use `--force-with-lease`.

## .gitignore and secrets

```
# .gitignore
*.log
.env
*.tfstate
*.tfstate.backup
.terraform/
__pycache__/
```

**Never commit secrets.** If you did: rotate the secret *first* (it is compromised), then clean history. Use scanners (`gitleaks`, `trufflehog`) and pre-commit hooks.

## Commit messages

Short imperative subject, what and why:

```
fix: raise nginx proxy timeout to 60s

Report generation takes up to 45s and was cut at 30s (504).
```

Many teams use prefixes (`feat:`, `fix:`, `docs:`) so changelogs can be generated.

## Tags and releases

```bash
git tag -a v1.4.0 -m "release 1.4.0"
git push origin v1.4.0
git describe --tags
```

Use semantic versioning: MAJOR (breaking) . MINOR (feature) . PATCH (fix).

## Useful extras

```bash
# binary search for the commit that broke it
git bisect start; git bisect bad; git bisect good v1.3.0

# second working copy of the same repo
git worktree add ../hotfix main

# search tracked files
git grep "proxy_read_timeout"

# commits that added/removed a string
git log -S "max_connections"
```

## Practice

1. Put your scripts from lessons 29–30 into a Git repo and push it to GitHub or GitLab.
2. Create a branch, change a script, open a PR, merge it with squash.
3. Create a conflict on purpose (two branches edit the same line) and resolve it.
4. Commit a fake `.env`, then remove it from history with `git filter-repo` (or start over) and add `.gitignore`.
5. Delete a branch with commits by mistake and get it back with `reflog`.
6. Use `git bisect` to find which of 10 commits broke a test script.

## Check yourself

- What is the difference between `git revert` and `git reset`?
- When is rebase dangerous?
- You pushed a password. What are the steps, in order?

## Next
[Lesson 32. Python for operations](32-python.md)
