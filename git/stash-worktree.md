# Stashing & Worktrees

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## Stashing Unfinished Work (`git stash`)

Temporarily shelves dirty working directory changes so you can switch branches cleanly.

```bash
# Stash tracked modified files with a descriptive label
git stash push -m "wip: user profile validation"

# Include untracked new files (-u) — crucial when you created new files
git stash push -u -m "wip: new service class"

# Stash only a specific file
git stash push -m "wip: navbar css" src/components/Navbar.css

# List all saved stashes
git stash list

# Inspect diff inside a stash without applying it
git stash show -p stash@{0}

# Restore latest stash AND remove it from stash list
git stash pop

# Restore a specific stash AND keep it in the stash list
git stash apply stash@{1}

# Delete a specific stash or wipe all stashes
git stash drop stash@{0}
git stash clear
```

## Parallel Checkouts (`git worktree`)

Allows checking out multiple branches into separate directories simultaneously—no stashing or re-cloning needed when an urgent hotfix interrupts feature work.

```bash
# Create a new folder and checkout an existing branch into it
git worktree add ../project-hotfix main

# Create a new folder AND a new branch simultaneously
git worktree add -b hotfix/auth-bug ../project-hotfix main

# List all active worktrees
git worktree list

# Remove a worktree after finishing work
git worktree remove ../project-hotfix

# Clean up stale worktree metadata if you deleted the folder manually
git worktree prune
```
