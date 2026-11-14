# 🔀 Branching & Merging

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 🌿 Branch Management

```bash
# List all local and remote branches
git branch -a

# Create and switch to a new branch (modern syntax)
git switch -c feat/payment-gateway

# Switch back to the previous branch
git switch -

# Rename current branch
git branch -m new-branch-name

# Delete local branch (-d = safe/merged only, -D = force delete)
git branch -d feat/payment-gateway

# Delete a remote branch
git push origin --delete feat/payment-gateway
```

## ⚖️ `merge` vs `rebase`

| Strategy | Command | History Shape | Best Used When |
| :--- | :--- | :--- | :--- |
| **Merge** | `git merge feat/login` | Preserves true chronology with a merge commit | Merging finished PRs into `main` or shared branches |
| **Rebase** | `git rebase main` | Linear, clean commit history (replays commits on top) | Updating your local feature branch with latest `main` |

> **Golden Rule of Rebase:** Never rebase commits that have already been pushed to a shared/public branch (`main`, `develop`). Rebase rewrites commit hashes and will break history for teammates.

```bash
# Force a merge commit even if fast-forward is possible
git merge --no-ff feat/login

# Only merge if fast-forward is possible (no extra merge commit)
git merge --ff-only feat/login

# Update feature branch with latest changes from main
git switch feat/login
git rebase main
```

## ⚔️ Resolving Merge Conflicts

When Git pauses with `CONFLICT (content)`:

```bash
# 1. Check which files have conflicts
git status

# 2. Option A: Manually edit conflict markers (<<<<<<<, =======, >>>>>>>)
#    Option B: Accept one side completely for a specific file
git checkout --ours <file>    # Keep current branch's version
git checkout --theirs <file>  # Keep incoming branch's version

# 3. Mark conflict as resolved and complete the operation
git add <file>
git commit                    # If resolving a merge
git rebase --continue         # If resolving a rebase

# Abort and return to pre-conflict state at any time
git merge --abort
git rebase --abort
```
