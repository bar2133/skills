---
name: git-worktree
description: >
  Create and manage git worktrees for isolated feature/bugfix work.
  Handles single-repo and multi-repo (flat) workspaces with automatic detection.
  Use when the user wants to start work on a new feature or bugfix in an isolated branch,
  set up worktrees across multiple repos, or clean up finished worktrees.
---

# Git Worktree

Create and manage git worktrees for isolated feature and bugfix work. Works seamlessly
with single-repo workspaces and flat multi-repo workspaces (a parent folder containing
multiple independent git repos side by side).

**Announce at start:** "I'm using the git-worktree skill to set up an isolated workspace."

## Unified Flow

The same pipeline handles 1 repo or N repos:

1. Discover repos
2. Select repos (if multiple)
3. Ask purpose and suggest branch names
4. Ask worktree location
5. Create worktrees and copy .env files
6. Run project setup
7. Report summary

---

## 1. Discover Repos

Detect all git repos in the current workspace. Use two strategies and combine results:

```bash
# Strategy 1: Scan current directory and immediate children for .git directories
# This works whether you're in a repo, a parent of repos, or anywhere
find . -maxdepth 2 -name .git -type d 2>/dev/null | sed 's|/.git$||'

# Strategy 2: If currently inside a git repo, also include it
# (handles the case where cwd is deep inside a single repo)
git_root=$(git rev-parse --show-toplevel 2>/dev/null) && echo "$git_root"
```

Note: `git rev-parse --show-toplevel` will fail if the current directory is NOT inside
a git repo (e.g. a plain parent folder containing multiple repos). That is expected —
rely on `find` as the primary discovery method, and use `git rev-parse` only as a
fallback to catch the case where cwd is deep inside a single repo.

Deduplicate the combined results to build the final repo list. Present to the user:

- **If only one repo found:** use it directly, move to step 3.
- **If multiple repos found:** list them with their current branch and short status,
  then let the user **select** which ones to include:

```
Found 3 git repos in this workspace:

  1. service-a       (on main, clean)
  2. service-b       (on main, 2 uncommitted changes)
  3. lib-common      (on develop, clean)

Which repos should I create worktrees for? (e.g. "1,3" or "all")
```

---

## 2. Configure Worktree

### Ask the Purpose

Ask the user what they are working on:

```
What bug or feature is this worktree for?
(e.g. "fix the login timeout issue" or "add dark mode support")
```

### Suggest Branch Names

Based on their answer, **suggest 2-3 meaningful branch/worktree names**:

- User says "fix the login timeout issue" -> suggest:
  1. `bugfix/login-timeout`
- User says "add dark mode support" -> suggest:
  1. `feature/dark-mode`
  2. `feature/dark-mode-support`

Use the convention: `bugfix/` for bugs, `feature/` for features. Do not use shortcuts like `fix/` or `feat/`.
Let the user pick one or type their own.

### Ask Worktree Location

```
Where should I create the worktree(s)?

  1. .worktrees/<branch-name>/ inside each repo (project-local, hidden)
  2. Shared folder at the workspace level (e.g. ../.worktrees/<branch-name>/)
  3. ~/.worktrees/<project>/<branch-name>/ (global location)
  4. Custom path

Which do you prefer?
```

In all cases, the worktree directory is named after the branch/feature name. For example,
if the branch is `bugfix/login-timeout`, worktrees are placed under a `login-timeout/`
folder within the chosen location. This keeps worktrees organized by the feature or bug
they relate to.

For multi-repo workspaces, the worktree folder mirrors the original workspace layout so
that opening it as a new workspace auto-detects all the git repos inside it:

```
.worktrees/login-timeout/
├── service-a/        <-- git worktree (auto-detected as git repo)
├── service-b/        <-- git worktree (auto-detected as git repo)
└── lib-common/       <-- git worktree (auto-detected as git repo)
```

The user can open `.worktrees/login-timeout/` directly in their IDE and it will
automatically recognize each subdirectory as a git repo, just like the original workspace.

### Verify .gitignore (project-local only)

If the user chose a project-local location, verify it is git-ignored:

```bash
git check-ignore -q .worktrees 2>/dev/null
```

If NOT ignored, add the entry to `.gitignore` and commit:

```bash
echo ".worktrees/" >> .gitignore
git add .gitignore
git commit -m "Add .worktrees/ to .gitignore"
```

### Per-Repo Branch Overrides (multi-repo only)

If multiple repos are selected, the chosen branch name applies to all by default. Ask:

```
Use branch "feature/dark-mode" for all 3 repos, or override per repo?
```

If the user wants overrides, let them specify a branch name for each repo.

### Existing Worktree Detection

Before creating, check if a worktree already exists for the target branch:

```bash
git worktree list | grep <branch-name>
```

If found, offer to reuse or recreate it.

---

## 3. Create Worktrees + Copy .env Files

For **each** selected repo:

### Create the Worktree

```bash
git worktree add <worktree-path> -b <branch-name>
```

### Copy Untracked .env Files

Find all untracked `.env` files in the **original** repo checkout (the source, not the worktree):

```bash
# Find untracked/ignored .env files
find <original-repo> -maxdepth 3 -name '.env*' -not -path '*/.git/*' 2>/dev/null
```

Filter to only files that are NOT tracked by git:

```bash
git -C <original-repo> ls-files --error-unmatch <file> 2>/dev/null
# If this fails (exit code 1), the file is untracked -> copy it
```

Common patterns to look for: `.env`, `.env.local`, `.env.development`, `.env.test`,
`.env.production`, and any nested `.env` files in subdirectories.

Copy each found untracked `.env` file to the **same relative path** in the new worktree:

```bash
cp <original-repo>/.env <worktree-path>/.env
cp <original-repo>/config/.env.local <worktree-path>/config/.env.local
```

Report which files were copied. **Never** commit or stage these files.

---

## 4. Project Setup

Auto-detect the project type in each worktree and install dependencies.

**Do NOT run tests** unless the user explicitly asks.

### Multiple Python Repos: Shared Virtual Environment

When **2 or more** selected repos are Python projects (detected by `setup.py`, `setup.cfg`,
or `pyproject.toml` with a `[project]` or `[tool.setuptools]` section), create a **single
shared virtualenv** instead of per-repo venvs:

```bash
python -m venv <workspace-or-worktree-parent>/.venv
source <workspace-or-worktree-parent>/.venv/bin/activate
pip install -e <repo-a-worktree> -e <repo-b-worktree> -e <repo-c-worktree>
```

This ensures cross-repo imports resolve to the editable worktree code. Use a single
`pip install -e` call with all repos and let pip resolve the dependency order.

Place the venv at the workspace level (e.g. `workspace/.venv/`) or next to the chosen
worktree location parent.

Report the venv path and activation command in the summary.

### Single Python Repo

```bash
python -m venv <worktree-path>/.venv
source <worktree-path>/.venv/bin/activate
pip install -e <worktree-path>
```

Or if the repo uses `requirements.txt` without a `setup.py`/`pyproject.toml`:

```bash
pip install -r <worktree-path>/requirements.txt
```

### Other Project Types

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

For non-Python repos in a mixed workspace, fall back to these per-repo setups.

---

## 5. Report Summary

Print a summary table for all created worktrees:

```
| Repo       | Branch             | Worktree Path              | .env Copied        | Status |
|------------|--------------------|----------------------------|--------------------:|--------|
| service-a  | feature/dark-mode  | service-a/.worktrees/dm    | .env, .env.local   | Ready  |
| lib-common | feature/dark-mode  | lib-common/.worktrees/dm   | .env               | Ready  |
```

If a shared Python venv was created, also report:

```
Shared virtualenv: workspace/.venv/
Activate with:     source workspace/.venv/bin/activate
```

For a single repo, a simple report is fine:

```
Worktree ready at <full-path>
Branch: <branch-name>
Copied .env files: .env, .env.local
Ready to implement <feature-description>
```

---

## 6. Cleanup / Teardown

When the user asks to clean up worktrees:

### List Active Worktrees

Scan all repos in the workspace and list active worktrees:

```bash
# For each repo
git -C <repo> worktree list
```

Present a numbered list and let the user select which to remove.

### Safety Checks

Before removing, check for uncommitted changes:

```bash
git -C <worktree-path> status --porcelain
```

If there are uncommitted changes, warn the user and prompt for confirmation before
force-removing.

### Remove Worktrees

**Always ask the user for explicit permission before removing each worktree:**

```
Remove worktree at <worktree-path>? (y/n)
```

Only proceed after the user confirms:

```bash
git -C <repo> worktree remove <worktree-path>
```

If the worktree has uncommitted changes, require a second explicit confirmation:

```
WARNING: <worktree-path> has uncommitted changes. Force-remove anyway? (y/n)
```

Only if the user confirms again:

```bash
git -C <repo> worktree remove --force <worktree-path>
```

### Clean Up Branches

**Always ask the user for explicit permission before deleting each branch:**

```
Delete branch <branch-name>? (merged: yes/no) (y/n)
```

If merged and user confirms:

```bash
git -C <repo> branch -d <branch-name>
```

If not merged, warn clearly and only delete with `-D` if the user explicitly confirms:

```
WARNING: Branch <branch-name> is NOT merged. Delete anyway? This cannot be undone. (y/n)
```

### Clean Up Shared Venv

**Always ask the user for explicit permission before deleting the shared venv:**

```
All worktrees using workspace/.venv/ have been removed. Delete the shared venv? (y/n)
```

Only delete after the user confirms.

### General Rule

**Never perform any delete operation (worktree removal, branch deletion, venv deletion)
without explicit user permission. Always ask first, always wait for confirmation.**

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| Single repo detected | Use directly, skip selection |
| Multiple repos detected | List with status, let user select |
| User describes bug | Suggest `bugfix/` branch names |
| User describes feature | Suggest `feature/` branch names |
| Project-local worktree dir not ignored | Add to `.gitignore` and commit |
| Worktree already exists for branch | Offer to reuse or recreate |
| Untracked `.env` files found | Copy to worktree, never commit them |
| Multiple Python repos selected | Create shared venv with `pip install -e` |
| Single Python repo | Create per-repo venv |
| Non-Python repo | Auto-detect and run setup (npm, cargo, go) |
| Tests | Do NOT run unless user explicitly asks |
| Any delete operation | **Always** ask user for explicit permission first |
| Cleanup with uncommitted changes | Warn and require double confirmation |
| Branch not merged on cleanup | Warn clearly, only `-D` with explicit confirmation |
| All worktrees removed for shared venv | Ask before deleting the venv |

## Common Mistakes

### Skipping .gitignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktrees

### Forgetting to copy .env files

- **Problem:** Worktree fails to run because of missing environment config
- **Fix:** Always scan for untracked `.env*` files and copy them over

### Creating separate venvs for interdependent Python repos

- **Problem:** Cross-repo imports fail because each venv only has its own repo installed
- **Fix:** Detect multiple Python repos and create a shared venv with `pip install -e`

### Running tests without being asked

- **Problem:** Wastes time, may fail due to missing external services
- **Fix:** Only run tests when the user explicitly requests it

### Deleting anything without asking

- **Problem:** Loses uncommitted work, removes branches or venvs the user still needs
- **Fix:** Always ask for explicit user permission before every delete operation (worktree, branch, venv)
