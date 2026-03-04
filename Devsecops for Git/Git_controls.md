
# 🔐 DevSecOps for Git – Detailed Notes
## 📌 Introduction

DevSecOps is not limited to CI/CD pipelines. It applies to every activity in DevOps, including source code management.

Git is one of the most common tools used in DevOps, and securing Git repositories is a critical responsibility of DevSecOps engineers.

This document covers 10 common security practices used by enterprises to secure Git repositories.

### 1️⃣ Git Ignore (.gitignore)
🔎 Problem

A developer accidentally commits sensitive files like:
- .env file with DB password
- terraform.tfstate
- id_rsa private key

Even if reverted, Git never forgets history. Once pushed:
- Others may have cloned the repo
- Repo may be forked
- Secret is exposed permanently

Only solution then: Rotate credentials

### ✅ Solution: .gitignore

.gitignore tells Git:

Do not track these files.

Example entries:
```bash
.env
*.tfstate
id_rsa
node_modules/
__pycache__/
```


How to implement:
- 1. Create .gitignore
- 2. Add entries
- 3. Commit and push to repository

```bash
git add .gitignore
git commit -m "Add gitignore"
git push
```

Now all developers cloning the repo will inherit it.

### ⚠️ Limitation

.gitignore works for known files, not patterns inside valid files.

If a developer hardcodes:
- AWS Secret inside .py file
- DB password inside Terraform file

.gitignore cannot prevent that.

### 2️⃣ Pre-Commit Hooks (Custom Script)
### 🔎 Problem

Sensitive values may exist inside tracked files.

Example:
AWS_SECRET_ACCESS_KEY = "XYZ"

### ✅ Solution: Pre-Commit Hook

A script that runs before every commit and blocks commits if sensitive patterns are detected.

### 📂 Location

Hooks are stored inside:
.git/hooks/

Create:
pre-commit

### 📝 Example Script
```bash
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash

echo "🔍 Running native pre-commit hook..."

if git diff --cached | grep -i "secret"; then
  echo "❌ Secret detected. Commit blocked."
  exit 1
fi

echo "✅ Commit passed security checks."
exit 0
EOF
```
Make it executable:

chmod +x .git/hooks/pre-commit

Now if a developer commits a file containing secret, commit is blocked.

### ⚠️ Limitation
- Must manually write pattern logic
- Hard to handle many secret patterns
- Not scalable across multiple repositories

## 3️⃣ Pre-Commit Framework (Using GitLeaks)

Instead of writing custom scripts, use:

👉 Pre-commit framework
👉 GitLeaks

🔧 Install Pre-Commit

Linux:
pip install pre-commit

### 📝 Create .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2
    hooks:
      - id: gitleaks

Install hook:
pre-commit install

Now:
- GitLeaks scans for secrets
- Detects AWS keys, tokens, passwords
- Blocks commit automatically

### ✅ Benefits Over Custom Script
- Detects multiple secret patterns
- Maintained by community
- Works across repositories
- More reliable detection

## 4️⃣ Repository Scanning (Historical Scan)
### 🔎 Problem

A secret might exist in:
- Old commits
- 6 months ago
- Reverted but still in Git history

Remember: Git never forgets

### ✅ Solution: Scan Entire Repository
gitleaks detect

Gitleaks — Repository & History Scanning
1. Create a custom rules file - custom-rules.toml
[[rules]]
id = "generic-password"
description = "Detect any PASSWORD assignment"
regex = '''(?i)password\s*=\s*["'][^"']+["']'''
tags = ["password", "custom"]

2. Run the gitleaks command
gitleaks detect --config custom-rules.toml


This scans:
- All commits
- Entire repository history

Recommended practice:
- Run once every 2–3 months
- Or schedule via cron job
- Or automate in CI

### 5️⃣ GitLeaks in CI/CD (GitHub Actions)
🔎 Problem

Developers may:
- Ignore pre-commit
- Not install framework
- Intentionally bypass controls

### ✅ Solution: Enforce via CI/CD

Create:
name: gitleaks
on: [pull_request, push, workflow_dispatch]
jobs:
  scan:
    name: gitleaks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE}} # Only required for Organizations, not personal accounts.

Now:
- Every PR is scanned
- Every push is scanned
- PR fails if secret detected

### 6️⃣ Branch Protection Rules
#### 🔎 Problem

Developers pushing directly to main branch.

Bad practice:
git push origin main

### ✅ Solution

Enable Branch Protection:
- 1. Go to Settings → Branches
- 2. Add rule for main
- 3. Enable:
    - Require pull request before merging
    - Require status checks
    - Require reviews

Now:
- No direct push allowed
- PR required
- CI must pass

### 7️⃣ RBAC in Git (Repository Access Control)

Control user permissions:
| Role  | Permission   |
| ----- | ------------ |
| Read  | View only    |
| Write | Push code    |
| Admin | Full control |

Manage under:
Settings → Collaborators

Enterprise level: manage via organization settings.

### 8️⃣ Mandatory Reviews

Require minimum reviewers before merge.

In branch protection:
- Require pull request
- Set number of approvals (e.g., 2)

Now PR merges only after required approvals.

### 9️⃣ CODEOWNERS File
#### 🔎 Problem

Only specific senior engineers should approve changes.
#### ✅ Solution: CODEOWNERS

Create:
CODEOWNERS

Example:
* @senior-dev1 @senior-dev2

Enable:
- Require review from Code Owners

Now GitHub automatically requests those reviewers.

🔟 Dependabot
🔎 Problem

Applications use:
- go.mod
- package.json
- pom.xml
- Docker images

These packages may contain vulnerabilities.

✅ Solution: Dependabot

Dependabot:
- Monitors dependency versions
- Checks vulnerability database
- Automatically creates PR with fixed version

Example config:
.github/dependabot.yml

version: 2
updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "daily"

Now:
- Vulnerability detected
- PR automatically created
- Can auto-merge

📊 Summary – 10 DevSecOps Git Controls
- .gitignore
- Pre-commit hook (custom)
- Pre-commit framework (GitLeaks)
- Repository scanning
- GitLeaks in CI/CD
- Branch protection rules
- RBAC access control
- Mandatory reviews
- CODEOWNERS
- Dependabot

🎯 DevSecOps Strategy for Git

Layered Security:
| Layer               | Protection              |
| ------------------- | ----------------------- |
| Developer Machine   | Git Ignore + Pre-commit |
| Repository History  | GitLeaks Detect         |
| CI/CD               | GitHub Actions Scan     |
| Governance          | Branch Protection       |
| Access              | RBAC                    |
| Code Quality        | Mandatory Reviews       |
| Dependency Security | Dependabot              |
