# Setup & Daily Workflow

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## Essential Configuration

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Modern defaults
git config --global init.defaultBranch main
git config --global pull.rebase true          # Avoid unnecessary merge commits on pull
git config --global fetch.prune true          # Auto-clean deleted remote branches
git config --global core.editor "code --wait" # Or "vim" / "nano"

# Line endings (Linux/macOS: input | Windows: true)
git config --global core.autocrlf input

# Useful shorthand aliases
git config --global alias.st "status -s"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD --stat"
```

## Core Daily Commands

```bash
# Compact status (M = modified, A = added, ?? = untracked)
git status -s

# Interactive staging: review and stage specific hunks/lines instead of whole files
git add -p

# Commit with inline message
git commit -m "feat(auth): add JWT refresh token rotation"
```

### Conventional Commits Quick Reference

| Prefix | Purpose |
| :--- | :--- |
| `feat:` | New feature for the user |
| `fix:` | Bug fix |
| `refactor:` | Code restructuring without behavior change |
| `chore:` | Dependencies, build scripts, config updates |
| `docs:` | Documentation only (`README`, comments) |
| `test:` | Adding or updating tests |

## Inspecting Diffs & History

```bash
# Unstaged changes (working directory vs staging)
git diff

# Staged changes (staging vs last commit)
git diff --staged

# Visual branch & commit graph
git log --oneline --graph --decorate --all

# Inspect a specific commit's metadata and diff
git show <commit-hash>

# See who last modified each line of a file
git blame <file>
```

## `.gitignore` Management

```bash
# Stop tracking a file that was already committed (keeps it on local disk)
git rm --cached .env
git commit -m "chore: stop tracking .env"

# Configure a global .gitignore across all repositories (for OS/IDE files)
git config --global core.excludesfile ~/.gitignore_global
```
