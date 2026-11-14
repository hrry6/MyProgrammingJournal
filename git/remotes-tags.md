# Remotes, Tags & Submodules

[🏠 Home](../README.md) > [🔀 Git](./README.md)

## Remote Management & Fork Syncing

```bash
# List remote repositories and URLs
git remote -v

# Change remote URL (e.g., switching from HTTPS to SSH)
git remote set-url origin git@github.com:username/repo.git

# Remove remote URL
git remote remove origin

# Add upstream remote (for forked repositories)
git remote add upstream [https://github.com/original-owner/repo.git](https://github.com/original-owner/repo.git)

# Sync your fork's main branch with upstream
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

## Safe Force Pushing

After an interactive rebase or `--amend` on your personal feature branch:

```bash
# Safer than --force: aborts if someone else pushed new commits to the remote branch
git push --force-with-lease
```

## Tags & Releases

```bash
# Create an annotated tag (recommended for releases)
git tag -a v1.0.0 -m "Release v1.0.0: initial production build"

# Tag a specific past commit
git tag -a v0.9.0 <commit-hash> -m "Beta release"

# List all tags
git tag -l

# Push a single tag or all tags to remote (git push alone does NOT push tags)
git push origin v1.0.0
git push origin --tags

# Delete a tag locally and remotely
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## Submodules

```bash
# Clone a repository along with all its submodules
git clone --recurse-submodules <repo-url>

# Add a submodule to an existing repo
git submodule add <submodule-repo-url> libs/shared-utils

# Initialize and pull submodules after a regular git clone
git submodule update --init --recursive

# Pull latest changes for all submodules
git submodule update --remote --merge
```
