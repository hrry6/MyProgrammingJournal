# ⏪ Undoing Mistakes & Recovery

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 🎯 Quick Decision Guide

| Situation | Command | Safe for Pushed History? |
| :--- | :--- | :--- |
| Discard uncommitted edits in a file | `git restore <file>` | Yes (local only) |
| Unstage a file (keep edits) | `git restore --staged <file>` | Yes (local only) |
| Undo local commit, keep changes staged | `git reset --soft HEAD~1` | **No** (rewrites history) |
| Undo local commit, keep changes unstaged | `git reset HEAD~1` | **No** (rewrites history) |
| Undo local commit & wipe all changes | `git reset --hard HEAD~1` | **No** (destructive) |
| Undo a commit already pushed to remote | `git revert <commit-hash>` | **Yes** (creates a new inverse commit) |

## 🚑 Emergency Recovery (`git reflog`)

Git tracks every movement of `HEAD` locally for ~30 days—even after a hard reset or deleted branch.

```bash
# 1. View the history of HEAD movements
git reflog

# 2. Find the hash right before the mistake (e.g., HEAD@{2}) and restore it
git reset --hard HEAD@{2}
```
