# Bilateral Sync Setup Guide

This document explains how to set up the bilateral sync between the private and public repositories.

## Overview

The sync system consists of three workflows:

1. **Daily Sync (Public → Private)**: `.github/workflows/sync-from-public.yml`
   - Runs daily at 8 AM UTC
   - Pulls changes from public repo to private repo
   - Acts as a safety net for missed syncs

2. **PR Merge Sync (Private → Public)**: `.github/workflows/sync-to-public.yml`
   - Runs when a PR is merged to `main` in the private repo
   - Pushes changes from private to public

3. **PR Merge Sync (Public → Private)**: `.github/workflows/sync-to-private.yml`
   - Runs when a PR is merged to `main` in the public repo
   - Pushes changes from public to private

## Required Secrets

### For Private Repository

**Secret Name:** `SYNC_TOKEN`
**Required Permissions:** Write access to the public repository

#### How to Create:

1. Go to GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Click "Generate new token"
3. Configure:
   - **Token name:** `private-to-public-sync`
   - **Expiration:** Choose appropriate duration (90 days, 1 year, or no expiration)
   - **Repository access:** Select "Only select repositories" → Choose `anthropics/claude-cookbooks`
   - **Permissions:**
     - Repository permissions → Contents: Read and write
     - Repository permissions → Pull requests: Read (optional, for metadata)
4. Generate token and copy it
5. Go to the private repo → Settings → Secrets and variables → Actions
6. Click "New repository secret"
7. Name: `SYNC_TOKEN`
8. Value: Paste the token
9. Click "Add secret"

### For Public Repository

**Secret Name:** `PRIVATE_SYNC_TOKEN`
**Required Permissions:** Write access to the private repository

#### How to Create:

1. Follow the same steps as above, but:
   - **Token name:** `public-to-private-sync`
   - **Repository access:** Choose `anthropics/claude-cookbooks-private` (the private repo)
   - **Permissions:**
     - Repository permissions → Contents: Read and write
2. Add this secret to the **public** repository:
   - Go to public repo → Settings → Secrets and variables → Actions
   - Name: `PRIVATE_SYNC_TOKEN`

## Deployment Steps

### Step 1: Add Workflows to Private Repository

```bash
# These files should already exist in the branch:
# - .github/workflows/sync-from-public.yml
# - .github/workflows/sync-to-public.yml
# - .github/workflows/sync-to-private.yml

# Merge the PR to add these workflows to main
```

### Step 2: Configure Secrets

1. Create `SYNC_TOKEN` in private repository (see above)
2. Create `PRIVATE_SYNC_TOKEN` in public repository (see above)

### Step 3: Add sync-to-private.yml to Public Repository

```bash
# Copy .github/workflows/sync-to-private.yml to the public repo
git checkout main
git pull public main

# Copy the file
cp .github/workflows/sync-to-private.yml /path/to/public/repo/.github/workflows/

# In the public repo:
cd /path/to/public/repo
git add .github/workflows/sync-to-private.yml
git commit -m "Add workflow to sync changes to private mirror"
git push origin main
```

### Step 4: Test the Setup

1. **Test daily sync:**
   ```bash
   # In private repo, manually trigger the workflow
   gh workflow run sync-from-public.yml

   # Check the run
   gh run list --workflow=sync-from-public.yml
   ```

2. **Test private → public sync:**
   - Create a test PR in private repo
   - Merge it
   - Check that the sync workflow runs
   - Verify changes appear in public repo

3. **Test public → private sync:**
   - Create a test PR in public repo (or have someone else do it)
   - Merge it
   - Check that the sync workflow runs
   - Verify changes appear in private repo

## How It Works

### Normal Flow (No Divergence)

```
Private PR merges → private/main updated → workflow pushes to public/main → ✅
Public PR merges → public/main updated → workflow pushes to private/main → ✅
```

### Divergence Handling

If both repos have unique commits (rare but possible):

1. Sync workflow fails (can't fast-forward)
2. GitHub issue is automatically created with details
3. Manual resolution required:

```bash
# Clone both remotes
git remote add public git@github.com:anthropics/claude-cookbooks.git
git remote add private git@github.com:anthropics/claude-cookbooks-private.git

# Fetch all
git fetch public
git fetch private

# Check divergence
git log public/main..private/main  # commits only in private
git log private/main..public/main  # commits only in public

# Decide which to keep or merge both
git checkout -b resolve-divergence private/main
git merge public/main
# Resolve conflicts if any

# Push to both
git push private HEAD:main
git push public HEAD:main
```

## Monitoring

### Check Workflow Status

```bash
# In private repo
gh run list --workflow=sync-from-public.yml
gh run list --workflow=sync-to-public.yml

# In public repo
gh run list --workflow=sync-to-private.yml
```

### Watch for Issues

Both workflows create GitHub issues automatically when sync fails. Monitor for:
- Issues labeled `sync-failure`
- Issues labeled `urgent`

## Troubleshooting

### Workflow doesn't trigger on PR merge

- Check that the PR was merged to `main` branch (not closed without merging)
- Check workflow permissions: Repo → Settings → Actions → General → Workflow permissions
- Ensure "Read and write permissions" is enabled

### Authentication errors

- Verify tokens haven't expired
- Verify token has correct permissions (Contents: Read and write)
- Verify token is added to correct repository's secrets

### Daily sync keeps resetting private main

- This means commits are being made directly to private `main` (bypassing PRs)
- All changes should go through PRs
- Consider enabling branch protection on `main`

## Security Notes

- Tokens should be rotated periodically (every 90 days recommended)
- Use fine-grained tokens (not classic tokens) for better security
- Tokens are stored as encrypted secrets in GitHub
- Never commit tokens to the repository

## Support

If you encounter issues with the sync system:
1. Check the workflow run logs in Actions tab
2. Look for automatically created issues with `sync-failure` label
3. Contact the team via [your communication channel]
