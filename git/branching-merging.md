# 🌿 Branching, Merge & Rebase

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 🔀 Branch Management

```bash
# List local and remote branches
git branch -a

# Create and switch to a new branch (modern syntax)
git switch -c feat/oauth-login

# Switch back to the previous branch
git switch -

# Rename current branch
git branch -m new-branch-name

# Delete a merged local branch (-D to force delete unmerged)
git branch -d feat/oauth-login
```

## 🔗 Merge vs. Rebase

| Strategy | Command | When to Use |
| :--- | :--- | :--- |
| **Merge** | `git merge feat/login` | Preserves exact branch history with a merge commit; safest for shared branches (`main`, `develop`). |
| **Rebase** | `git rebase main` | Replays your feature commits on top of `main` for a clean, linear history. **Never rebase public/shared branches.** |

### Interactive Rebase (Squashing Commits)

Clean up messy WIP commits before opening a Pull Request:

```bash
# Squash or edit the last 3 commits
git rebase -i HEAD~3
# In the editor, change 'pick' to 'squash' (or 's') to merge into the previous commit
```

## 🍒 Cherry-Pick & Conflicts

```bash
# Copy a specific commit from another branch onto your current branch
git cherry-pick <commit-hash>

# Abort an ongoing merge or rebase if conflicts get messy
git merge --abort
git rebase --abort
```
