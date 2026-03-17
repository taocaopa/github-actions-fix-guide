# GitHub Actions Workflow Failure: A Complete Post-Mortem and Fix

> **TL;DR**: Node.js 20 deprecation warning + missing write permissions + improper git authentication = failed workflow. Here's how to fix it properly.

---

## 🚨 The Issue

### Error Message
```
Process completed with exit code 1.
Node.js 20 actions are deprecated. The following actions are running on Node.js 20 
and may not work as expected: actions/checkout@v3.
```

### Failed Workflow
The workflow was designed to automatically update a profile README daily, but it failed with exit code 1. Two critical issues were identified:

1. **Deprecation Warning**: `actions/checkout@v3` uses deprecated Node.js 20
2. **Permission Error**: The workflow couldn't push changes back to the repository

---

## 🔍 Root Cause Analysis

### Issue 1: Node.js 20 Deprecation

**What happened?**
GitHub Actions announced the deprecation of Node.js 20. Starting June 2nd, 2026, all actions will be forced to run on Node.js 24. The warning appears because `actions/checkout@v3` runs on Node.js 20.

**Why it matters?**
- Deprecated actions may stop working unexpectedly
- Security updates won't be backported to old versions
- Future GitHub Actions runners may not support Node.js 20

**The fix**: Upgrade to `actions/checkout@v4`

```yaml
# ❌ Before (Deprecated)
- uses: actions/checkout@v3

# ✅ After (Current)
- uses: actions/checkout@v4
```

---

### Issue 2: Missing Write Permissions

**What happened?**
By default, GitHub Actions workflows run with read-only permissions on repository contents. When the workflow tried to push a commit, it was rejected due to insufficient permissions.

**The error (implied)**:
```
fatal: unable to access 'https://github.com/...': The requested URL returned error: 403
```

**Why it matters?**
GitHub enhanced security by making workflows read-only by default. This prevents malicious workflows from modifying your code without explicit permission.

**The fix**: Explicitly grant write permissions

```yaml
# Add this at the top level of your workflow
permissions:
  contents: write
```

---

### Issue 3: Improper Git Configuration

**What happened?**
The original workflow attempted to commit and push changes without:
1. Checking if there were actual changes to commit
2. Properly handling the case when no changes exist
3. Using the correct git push syntax

**The original problematic code**:
```yaml
- name: Commit changes
  run: |
    git config --local user.email "action@github.com"
    git config --local user.name "GitHub Action"
    git add update.log
    git commit -m "chore: update profile stats" || exit 0  # Silent failure
    git push  # May fail without proper auth
```

**Problems with this approach**:
1. `|| exit 0` hides errors silently
2. No verification that changes actually exist
3. `git push` without specifying branch or authentication

---

## ✅ The Complete Fix

### Fixed Workflow File

```yaml
name: Profile Update

on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight UTC
  workflow_dispatch:      # Allow manual triggers

# FIX 1: Grant write permissions
permissions:
  contents: write

jobs:
  update-readme:
    runs-on: ubuntu-latest

    steps:
    # FIX 2: Use actions/checkout@v4 (not v3)
    - uses: actions/checkout@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        fetch-depth: 0

    - name: Update README with latest stats
      run: |
        echo "Last Updated: $(date)" >> update.log

    # FIX 3: Separate git configuration
    - name: Configure Git
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"

    # FIX 4: Check if there are changes
    - name: Check for changes
      id: verify-changed-files
      run: |
        if [ -n "$(git status --porcelain)" ]; then
          echo "changed=true" >> $GITHUB_OUTPUT
        else
          echo "changed=false" >> $GITHUB_OUTPUT
        fi

    # FIX 5: Only commit if there are changes
    - name: Commit changes
      if: steps.verify-changed-files.outputs.changed == 'true'
      run: |
        git add update.log
        git commit -m "chore: update profile stats"
        
    - name: Push changes
      if: steps.verify-changed-files.outputs.changed == 'true'
      run: |
        git push origin ${{ github.ref_name }}
```

---

## 📝 Detailed Explanation of Each Fix

### Fix 1: Write Permissions

```yaml
permissions:
  contents: write
```

**Why this is needed:**
- GitHub Actions uses a token (`GITHUB_TOKEN`) to authenticate
- Default permission is read-only for security
- `contents: write` allows the workflow to create commits and push them

**Security best practice:**
Grant only the minimum required permissions. If your workflow only needs to read code, don't grant write access.

---

### Fix 2: Upgrade to actions/checkout@v4

```yaml
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    fetch-depth: 0
```

**Changes from v3 to v4:**
- Node.js runtime upgraded from 20 to 24
- Improved performance
- Better error messages
- Continued security updates

**The `fetch-depth: 0` parameter:**
- Fetches complete git history (all branches and tags)
- Required if you need to push commits back to the repo
- Default is `1` (shallow clone), which doesn't include full history

---

### Fix 3: Separate Concerns

Instead of one giant step that does everything, we separate:
1. **Configuration** - Set up git user
2. **Detection** - Check if changes exist
3. **Action** - Commit and push (only if needed)

This makes debugging easier and prevents unnecessary commits.

---

### Fix 4: Proper Change Detection

```yaml
- name: Check for changes
  id: verify-changed-files
  run: |
    if [ -n "$(git status --porcelain)" ]; then
      echo "changed=true" >> $GITHUB_OUTPUT
    else
      echo "changed=false" >> $GITHUB_OUTPUT
    fi
```

**How it works:**
- `git status --porcelain` returns empty string if no changes
- `-n` tests if string is non-empty
- Output is saved to `$GITHUB_OUTPUT` for use in later steps
- Subsequent steps use `if:` condition to skip when no changes

---

## 🎓 Lessons for MLOps Engineers

### 1. Always Specify Permissions

When deploying ML pipelines with GitHub Actions:

```yaml
permissions:
  contents: read        # For checking out code
  packages: write       # For pushing Docker images
  id-token: write       # For AWS/GCP authentication
```

### 2. Keep Actions Updated

Create a workflow to check for outdated actions:

```yaml
name: Check for Action Updates
on:
  schedule:
    - cron: '0 0 * * 1'  # Weekly
jobs:
  check-updates:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: reviewdog/action-depup@v1
```

### 3. Test Before Scheduling

Always use `workflow_dispatch` for manual testing:

```yaml
on:
  workflow_dispatch:  # Manual trigger for testing
  schedule:
    - cron: '0 0 * * *'  # Only enable after testing
```

### 4. Handle Failures Gracefully

```yaml
- name: Run ML Pipeline
  continue-on-error: true  # Don't fail entire workflow
  id: pipeline
  run: |
    python train_model.py || echo "status=failed" >> $GITHUB_OUTPUT

- name: Notify on Failure
  if: steps.pipeline.outputs.status == 'failed'
  uses: slack/notify-action@v1
```

---

## 🔧 Debugging Tips

### Enable Debug Logging

Add this to your workflow to see detailed logs:

```yaml
env:
  ACTIONS_STEP_DEBUG: true
  ACTIONS_RUNNER_DEBUG: true
```

### Check Token Permissions

Add this step to see what permissions your token has:

```yaml
- name: Check Token Permissions
  run: |
    curl -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
         https://api.github.com/repos/${{ github.repository }} \
         -o repo-info.json
    cat repo-info.json | jq '.permissions'
```

### Test Locally with act

Use [nektos/act](https://github.com/nektos/act) to test workflows locally:

```bash
# Install act
brew install act

# Run workflow locally
act -j update-readme
```

---

## 📚 Related Resources

### Official Documentation
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Token Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)

### Deprecation Notices
- [GitHub Actions: Deprecation of Node.js 20](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/)
- [actions/checkout releases](https://github.com/actions/checkout/releases)

### MLOps Best Practices
- [GitHub Actions for ML](https://github.com/marketplace?type=actions&query=machine+learning)
- [MLOps Community](https://mlops.community/)

---

## ✅ Checklist for MLOps Workflows

When building ML pipelines with GitHub Actions:

- [ ] Use `actions/checkout@v4` (not v3)
- [ ] Specify explicit permissions
- [ ] Use `workflow_dispatch` for manual testing
- [ ] Separate steps for clarity
- [ ] Check for changes before committing
- [ ] Handle errors gracefully
- [ ] Add debug logging for troubleshooting
- [ ] Document required secrets/tokens
- [ ] Test with `act` locally before pushing
- [ ] Monitor for deprecation warnings

---

## 🔮 Future-Proofing Your Workflows

### Pin to Major Versions

```yaml
# Good: Gets latest v4.x.x
- uses: actions/checkout@v4

# Better: Pin to specific version for reproducibility
- uses: actions/checkout@v4.1.1
```

### Use Dependabot for Updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Summary

| Issue | Root Cause | Solution |
|-------|------------|----------|
| Deprecation warning | Using actions/checkout@v3 | Upgrade to v4 |
| Permission denied | Missing write permissions | Add `permissions: contents: write` |
| Silent failures | Using `|| exit 0` | Proper error handling with conditional steps |
| Push failures | Improper git configuration | Separate config, detect changes, push with branch name |

---

**Author**: MLOps Engineering Team  
**Date**: March 17, 2026  
**Repository**: https://github.com/taocaopa/github-actions-fix-guide

*Learn from our mistakes so you don't have to make them!*
