# 🌐 Remotes & Collaboration

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 🔄 Syncing with Remote

```bash
# Download remote changes without modifying local files
git fetch origin

# Pull latest changes using rebase instead of creating messy merge commits
git pull --rebase origin main

# Push a new local branch and set upstream tracking (-u)
git push -u origin feat/oauth-login
```

## 🛡️ Safe Force Pushing

After amending or rebasing a local feature branch, use `--force-with-lease` instead of `--force` so you don't accidentally overwrite someone else's commits on that branch.

```bash
git push --force-with-lease origin feat/oauth-login
```

## 🧹 Remote Maintenance

```bash
# View configured remote URLs
git remote -v

# Remove stale local references to remote branches that were already deleted
git fetch --prune

# Delete a branch directly on the remote repository
git push origin --delete feat/old-feature
```
