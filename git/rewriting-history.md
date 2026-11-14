# ✍️ Rewriting History

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 🩹 Amending the Last Commit (`--amend`)

```bash
# Add forgotten files to the previous commit without changing the commit message
git add forgotten-file.ts
git commit --amend --no-edit

# Edit only the previous commit message
git commit --amend -m "fix(api): handle null response payload"
```

## 🧹 Interactive Rebase (`rebase -i`)

Clean up messy WIP commits into logical, atomic commits before opening a Pull Request.

```bash
# Interactively edit the last 4 commits
git rebase -i HEAD~4
```

### Interactive Rebase Actions

| Action | Short | Effect |
| :--- | :--- | :--- |
| `pick` | `p` | Keep the commit as-is |
| `reword` | `r` | Keep changes, edit the commit message |
| `edit` | `e` | Pause rebase to amend files or split the commit |
| `squash` | `s` | Meld commit into the previous commit and combine messages |
| `fixup` | `f` | Meld commit into the previous commit and discard its message |
| `drop` | `d` | Delete the commit completely |

## 🍒 Cherry-Picking (`git cherry-pick`)

Apply specific commits from another branch onto your current branch without merging the entire branch.

```bash
# Copy a single commit onto current branch
git cherry-pick <commit-hash>

# Copy a range of commits (exclusive of start hash)
git cherry-pick <start-hash>..<end-hash>

# Apply changes into working directory/staging WITHOUT auto-committing
git cherry-pick -n <commit-hash>

# Handle conflicts during cherry-pick
git cherry-pick --continue
git cherry-pick --abort
```
