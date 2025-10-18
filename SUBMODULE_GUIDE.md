# Git Submodule Guide

This guide explains how to work with the "Accounts and Delivery Department Network Design" project and how to pin it to a specific version.

## Current Setup

Currently, the "Accounts and Delivery Department Network Design" folder is part of the main repository. This document explains how to reference specific versions and optionally convert it to a Git submodule.

## Pinning to a Specific Version

### Option 1: Using Git Tags (Current Approach)

The "Accounts and Delivery Department Network Design" project has been tagged to allow pinning to specific versions:

- **Tag**: `accounts-delivery-v1.0` 
- **Commit**: Points to the completed state of the project
- **Date**: October 18, 2025

To reference this specific version:

```bash
# Clone the repository
git clone https://github.com/jenniferwang2/cisco-packet-tracer-designs.git

# Checkout the specific tag
git checkout accounts-delivery-v1.0

# Or view the specific folder at that tag
git show accounts-delivery-v1.0:"Accounts and Delivery Department Network Design/README.md"
```

**Note for maintainers**: After merging the PR that creates this tag, push it to the remote repository:
```bash
git push origin accounts-delivery-v1.0
```

### Option 2: Converting to a Git Submodule (Future Enhancement)

If you want to manage the "Accounts and Delivery Department Network Design" as a completely separate repository, follow these steps:

#### Step 1: Extract the Folder to a New Repository

```bash
# Create a new repository for the Accounts project
# (This would be done on GitHub or your Git hosting service)

# Use git filter-branch or git subtree to extract the folder
git subtree split --prefix="Accounts and Delivery Department Network Design" -b accounts-delivery-branch

# Push this branch to the new repository
git remote add accounts-repo <NEW_REPO_URL>
git push accounts-repo accounts-delivery-branch:main
```

#### Step 2: Remove the Folder and Add as Submodule

```bash
# Remove the folder from the main repository
git rm -r "Accounts and Delivery Department Network Design"
git commit -m "Remove Accounts folder before converting to submodule"

# Add it back as a submodule
git submodule add <NEW_REPO_URL> "Accounts and Delivery Department Network Design"
git commit -m "Add Accounts project as submodule"
```

#### Step 3: Pin the Submodule to a Specific Commit

```bash
# Navigate to the submodule directory
cd "Accounts and Delivery Department Network Design"

# Checkout the specific commit you want to pin
git checkout <COMMIT_SHA>

# Go back to main repository
cd ..

# Stage the submodule change (this pins it)
git add "Accounts and Delivery Department Network Design"
git commit -m "Pin Accounts submodule to version 1.0"
```

#### Step 4: Working with the Pinned Submodule

```bash
# Clone the main repository with submodules
git clone --recurse-submodules https://github.com/jenniferwang2/cisco-packet-tracer-designs.git

# Or if already cloned, initialize submodules
git submodule update --init --recursive

# To update a submodule to the latest commit
cd "Accounts and Delivery Department Network Design"
git checkout main
git pull
cd ..
git add "Accounts and Delivery Department Network Design"
git commit -m "Update Accounts submodule to latest"
```

## Viewing Pinned Version Information

To see what version of the "Accounts and Delivery Department Network Design" is currently pinned:

```bash
# Using tags (current approach)
git describe --tags --match="accounts-delivery-*"

# If converted to submodule
git submodule status
# or
cd "Accounts and Delivery Department Network Design"
git log -1 --oneline
```

## Benefits of Each Approach

### Git Tags (Current)
- ✅ Simpler to maintain
- ✅ All content in one repository
- ✅ Easier for contributors
- ❌ Cannot independently version the project
- ❌ Shared commit history

### Git Submodules
- ✅ Independent versioning
- ✅ Can have separate contributors
- ✅ Precise pinning to specific commits
- ✅ Can reference same submodule from multiple projects
- ❌ More complex workflow
- ❌ Requires `--recurse-submodules` flag when cloning
- ❌ Submodule updates need explicit commits

## Recommended Workflow

For this repository, **using Git tags** is recommended for the following reasons:
1. The projects are educational labs, not production code
2. Simpler workflow for learners and reviewers
3. Maintains clear project history
4. Easy to reference specific states

If individual projects need independent development or reuse across multiple repositories, converting to submodules would be beneficial.

## References

- [Git Submodules Documentation](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Git Tags Documentation](https://git-scm.com/book/en/v2/Git-Basics-Tagging)
