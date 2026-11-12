# ⌨️ Git CLI Cheatsheet

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## ⚡ Staging & Committing

```bash
# Check working tree status (short format)
git status -s

# Commit with inline message
git commit -m "feat: add user authentication endpoint"

# Modify the last commit message or add forgotten staged files (DO NOT use if already pushed)
git commit --amend
```

## 🔍 Logs & Diffs

```bash
# Compact visual commit graph
git log --graph --decorate --all -n 15

# Show changes in working directory (unstaged)
git diff

# Show changes currently staged for the next commit
git diff --staged

# Show what changed in a specific commit
git show <commit-hash>
```

## 📦 Stashing Work-in-Progress

```bash
# Save uncommitted changes with a descriptive label (including untracked files with -u)
git stash push -u -m "wip: refactor payment service"

# List all saved stashes
git stash list

# Apply the latest stash and remove it from the stash list
git stash pop

# Apply a specific stash without deleting it
git stash apply stash@{1}
```
