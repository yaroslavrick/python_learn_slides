---
title: Git basics
---

![Commit graph with main and feature branch](/assets/images/topics/git-basics.svg)
<!-- .element: class="title-illustration" -->

# Git basics

Distributed version control — the daily commands.

---

## Why Git?

- **Track every change** — every save is recoverable
- **Branch freely** — try ideas without breaking `main`
- **Collaborate** — many people, one history
- **Distributed** — every clone is a full repository
- **Industry standard** — GitHub, GitLab, Bitbucket all speak Git

If you write code, you use Git.

---

## Three places code lives

```
working tree   →   index (staging)   →   repository (commits)
   files            git add               git commit
   you edit         changes you'll        the recorded
                    include next          history
```

- **Working tree** — the files in your folder right now
- **Index / staging** — what `git commit` will record next
- **Repository** — the committed history (`.git/` directory)

`git status` shows the difference between these three.

---

## First-time setup

```bash
git config --global user.name  "Alice"
git config --global user.email "alice@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

Configure once per machine. Stored in `~/.gitconfig`.

---

## Starting a repository

```bash
# New repo from scratch
git init my-project
cd my-project

# OR clone an existing repo
git clone https://github.com/owner/repo.git
git clone git@github.com:owner/repo.git    # SSH
```

`git init` creates `.git/` in the current directory. `git clone` does the same plus downloads history.

---

## The everyday cycle

```bash
git status                  # see what changed
git add file.py             # stage one file
git add .                   # stage everything in cwd
git commit -m "Add foo"     # record staged changes
git push                    # send to the remote
```

```bash
git pull                    # fetch + merge from remote
```

These five commands cover ~80% of daily work.

---

## Inspecting changes

```bash
git status                  # what's modified, staged, untracked
git diff                    # unstaged changes
git diff --staged           # staged changes (ready to commit)
git diff main..my-branch    # between branches
git log --oneline -10       # last 10 commits, one line each
git log --graph --oneline   # ASCII branch diagram
git show HEAD               # the most recent commit's diff
git blame file.py           # who last edited each line
```

`git log -p path/to/file` walks the history of one file with diffs.

---

## Branches

```bash
git branch                  # list local branches (* = current)
git branch feature-x        # create
git switch feature-x        # switch to it
git switch -c feature-y     # create + switch in one step

git switch main             # back to main
git merge feature-x         # merge feature-x into current branch

git branch -d feature-x     # delete (safe — refuses if unmerged)
git branch -D feature-x     # force delete
```

`git checkout` does the same job but is overloaded — prefer `git switch` and `git restore` (newer, clearer).

---

## Remotes

```bash
git remote -v                                       # list remotes
git remote add origin git@github.com:me/repo.git    # add one
git fetch origin                                    # download (no merge)
git push -u origin main                             # push + set upstream
git push                                            # subsequent pushes
git pull                                            # fetch + merge upstream
```

`origin` is the conventional default name for the primary remote. You can have several (`upstream`, `staging`, ...).

---

## Undoing things

```bash
# Unstage a file (keep the changes in working tree)
git restore --staged file.py

# Discard unstaged changes to a file
git restore file.py
git restore .                          # all files (destructive!)

# Amend the most recent commit (message or content)
git commit --amend

# Revert a published commit by making a new "undo" commit
git revert <sha>

# Reset HEAD back N commits, keeping changes uncommitted
git reset HEAD~3
```

**`git reset --hard`** discards work — be sure before using it.

---

## Tags

```bash
git tag v1.0.0                          # lightweight tag
git tag -a v1.0.0 -m "First release"    # annotated tag (preferred for releases)
git push origin v1.0.0                  # tags don't auto-push
git push --tags                         # push all tags

git tag                                 # list tags
git tag -d v0.9.0                       # delete locally
```

Tags pin a commit to a name — typically used for releases. CI often triggers on tag pushes.

---

## .gitignore

A list of files Git should pretend don't exist:

```gitignore
# Python
__pycache__/
*.pyc
.venv/
.env
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Editor
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Build
dist/
build/
*.egg-info/
```

Commit this **before** adding files. To ignore an already-tracked file: remove from the index with `git rm --cached <file>`.

---

## Useful aliases

```bash
git config --global alias.s   "status -s"
git config --global alias.co  "switch"
git config --global alias.cob "switch -c"
git config --global alias.lg  "log --graph --oneline --decorate"
git config --global alias.last "log -1 HEAD"
```

```bash
git s                       # short status
git lg                      # pretty graph
git last                    # most recent commit
```

Tune to taste — your future self will thank you.

---

## Stashing

When you need to switch branches but aren't ready to commit:

```bash
git stash                   # save changes, clear working tree
git stash -u                # include untracked files
git stash list              # see all stashes
git stash pop               # apply most recent and remove from list
git stash apply stash@{1}   # apply a specific one without removing
git stash drop stash@{0}    # delete a stash
```

Don't use stash as long-term storage — it's a working-day buffer.

---

## Resolving merge conflicts

```bash
git merge feature-x
# Auto-merging file.py
# CONFLICT (content): Merge conflict in file.py
```

Open the file — you'll see:

```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> feature-x
```

Edit to the desired result, remove the markers, then:

```bash
git add file.py
git commit                  # or `git merge --continue`
```

To bail: `git merge --abort`.

---

## A safe daily workflow

1. `git status` — know where you are
2. Pull on the right branch: `git switch main && git pull`
3. Branch off: `git switch -c feature-x`
4. Commit often, in logical chunks
5. `git push -u origin feature-x` once
6. Open a PR, get review, merge
7. Switch back to `main`, pull, delete the local branch

The fewer commands you type from memory, the better — keep aliases, IDE integration, or a TUI like `lazygit` close.

---

## What's next

- **GitFlow** — when (and when not) to use branch models
- **CI** — automate builds on every push
