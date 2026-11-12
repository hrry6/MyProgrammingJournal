# ⚙️ Conventions & Config

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## 📝 Conventional Commits

Format: `<type>(<optional-scope>): <description>`

| Type | Purpose |
| :--- | :--- |
| `feat` | A new feature for the user |
| `fix` | A bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `docs` | Documentation-only changes |
| `chore` | Build process, dependencies, or tooling updates |
| `test` | Adding or updating tests |
| `perf` | Performance improvements |

## 🔧 Recommended `.gitconfig` Setup

```bash
# Set default branch name to main
git config --global init.defaultBranch main

# Always rebase when running 'git pull'
git config --global pull.rebase true

# Automatically prune deleted remote branches on fetch/pull
git config --global fetch.prune true

# Useful global aliases
git config --global alias.st "status -s"
git config --global alias.lg "log --oneline --graph --decorate --all -n 15"
```

## 🙈 Fixing `.gitignore` Not Working

If a file was already tracked by Git before you added it to `.gitignore`, clear the index cache without deleting your local files:

```bash
git rm -r --cached .
git add .
git commit -m "chore: refresh gitignore tracked files"
```
