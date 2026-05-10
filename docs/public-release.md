# Public release strategy

The `master` branch in this repo contains full git history from earlier iterations that used a different project name (since replaced by OMH) and an old local absolute path.

Publishing `master` with full history would expose that history. This document describes safe, non-destructive ways to publish a clean tree.

## Option A: orphan-commit branch (recommended)

Create a new branch from the current tree with no parent history. This keeps `master` intact and gives GitHub a clean root commit to display.

```bash
# 1. Save the current tree
cd /path/to/oh-my-hermes-agent
CURRENT_BRANCH=$(git branch --show-current)

# 2. Create and switch to a new orphan branch
git checkout --orphan public-release

# 3. Stage the current tracked tree
git add -A

# 4. Commit as a clean root commit
git commit -m "docs: initial public release"

# 5. Push the orphan branch to GitHub (do NOT force-push to master)
git push -u origin public-release

# 6. In GitHub repository settings, set public-release as the default branch
#    before making the repo public. Keep master private or delete it later
#    only after confirming everything is correct.
```

This is fully non-destructive: `master` remains in the repository unchanged.

## Option B: squash all history into a single commit on a new branch

If you prefer one commit but want to keep `master` intact:

```bash
cd /path/to/oh-my-hermes-agent

# 1. Create a new branch from current HEAD
git checkout -b public-release-squash

# 2. Soft-reset to the very first commit, keeping all changes staged
FIRST_COMMIT=$(git rev-list --max-parents=0 HEAD)
git reset --soft "$FIRST_COMMIT"

# 3. Re-commit everything as one clean commit
git commit -m "docs: initial public release (squashed)"

# 4. Push as a new branch
git push -u origin public-release-squash
```

Caution: This changes commit SHAs on the new branch, but `master` is untouched.

## Option C: reinitialize as a fresh repo (destructive to local history only)

If you are certain you no longer need the local history:

```bash
cd /path/to/oh-my-hermes-agent

# Back up the old .git just in case
cp -a .git .git.backup

# Reinitialize
git init
git add -A
git commit -m "docs: initial public release"

# Add GitHub remote and push as a new branch
git remote add origin https://github.com/YOURUSER/oh-my-hermes-agent.git
git push -u origin master
```

**Warning:** This destroys all local commit history. Only use if you have no need for the old history and have backed up `.git`.

## Before making any branch public

Run the validation checklist from [`README.md`](../README.md#validation-checklist) and confirm:

- No stale terminology hits outside the checklist files.
- No real credentials in the tracked tree.
- `configs/example.yaml` parses as valid YAML.

All of the above must return zero unexpected hits before publishing.

## Recommended default branch on GitHub

After pushing a clean branch, set it as the default in GitHub settings and keep `master` unlisted or archive it. This ensures visitors see only the clean history.
