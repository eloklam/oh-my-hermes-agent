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
#    before making the repo public. After confirming everything is correct,
#    delete the `master` branch from the remote; GitHub visibility is
#    per-repository, not per-branch, so old history remains reachable until
#    the branch is removed.
```

This is fully non-destructive: the local `master` branch remains in your repository unchanged.

## Option B: squash all history into a single commit on a new branch

If you prefer one commit but want to keep the local `master` intact:

```bash
cd /path/to/oh-my-hermes-agent

# 1. Create a new branch from current HEAD
git checkout -b public-release-squash

# 2. Soft-reset to the root commit, keeping all changes staged.
#    If your history has multiple root commits, this selects the first one.
ROOT_COMMIT=$(git rev-list --max-parents=0 HEAD | head -n 1)
git reset --soft "$ROOT_COMMIT"

# 3. Amend the root commit so it absorbs all changes into a single commit
#    with no parent history beyond the (now amended) root.
#    Use --reset-author so the squashed commit does not inherit legacy identity metadata.
git commit --amend --reset-author -m "docs: initial public release (squashed)"

# 4. Push as a new branch
git push -u origin public-release-squash
```

Caution: This rewrites the root commit SHA on the new branch, but the local `master` is untouched.

## Option C: reinitialize as a fresh repo (destructive to local history only)

If you are certain you no longer need the local history:

```bash
cd /path/to/oh-my-hermes-agent

# Back up the old .git just in case
cp -a .git .git.backup

# Remove existing Git metadata and reinitialize
rm -rf .git
git init
git add -A
git commit -m "docs: initial public release"

# Add (or replace) GitHub remote and push the current branch
git remote remove origin 2>/dev/null || true
git remote add origin https://github.com/YOURUSER/oh-my-hermes-agent.git
git push -u origin HEAD
```

**Warning:** This destroys all local commit history. Only use if you have no need for the old history and have backed up `.git`.

## Before making any branch public

Run the validation checklist from [`README.md`](../README.md#validation-checklist) and confirm:

- No stale terminology hits outside the checklist files.
- No real credentials in the tracked tree.
- `configs/example.yaml` parses as valid YAML.

All of the above must return zero unexpected hits before publishing.

## Recommended default branch on GitHub

After pushing a clean branch, set it as the default in GitHub settings and delete the old `master` branch from the remote. This limits what visitors see by default, but it does not guarantee old commits are completely unreachable: cached SHAs, forks, pull-request references, and tags can still expose prior history. If you need stronger privacy guarantees, review and remove any tags pointing to old commits and consider contacting GitHub Support for cache clearing.
