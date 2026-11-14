# Undo & Disaster Recovery

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## Choosing the Right Undo Tool

| Scope | Command | Safe on Shared Branches? | Effect |
| :--- | :--- | :--- | :--- |
| **Uncommitted file** | `git restore` | Yes (local only) | Discards working changes or unstages files |
| **Local unpushed commits** | `git reset` | **No** (rewrites history) | Moves branch pointer backward |
| **Pushed shared commits** | `git revert` | **Yes** | Creates a new inverse commit to undo changes |

## Discarding Local Changes (`git restore`)

```bash
# Discard unstaged changes in a specific file (irreversible)
git restore src/App.tsx

# Unstage a file while keeping your edits in the working directory
git restore --staged src/App.tsx

# Discard ALL unstaged changes across the entire repo
git restore .
```

## Moving Branch History Backward (`git reset`)

```bash
# --soft: Undo last commit, keep all changes STAGED (great for re-committing)
git reset --soft HEAD~1

# --mixed (default): Undo last commit, keep changes UNSTAGED in working dir
git reset HEAD~1

# --hard: Undo last commit AND wipe all changes permanently
git reset --hard HEAD~1
```

## Safely Undoing Pushed Commits (`git revert`)

```bash
# Create a new commit that reverses the changes of a specific commit
git revert <commit-hash>

# Revert a merge commit (-m 1 keeps the main/parent branch side)
git revert -m 1 <merge-commit-hash>
```

## Time Machine Recovery (`git reflog`)

`reflog` records every movement of `HEAD` locally for ~90 days—even after `reset --hard` or deleted branches.

```bash
# 1. View chronological history of HEAD movements
git reflog

# Example output:
# a1b2c3d HEAD@{0}: reset: moving to HEAD~2
# f9e8d7c HEAD@{1}: commit: feat: complete payment webhook  <-- Lost commit!

# 2. Restore your branch back to before the accident
git reset --hard f9e8d7c

# Or recover a deleted branch by creating a new branch at its last commit hash
git switch -c recovered-branch f9e8d7c
```
