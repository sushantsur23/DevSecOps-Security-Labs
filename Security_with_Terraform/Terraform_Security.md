# 🔐 DevSecOps for Terraform – Detailed Notes
## 📌 Introduction

This section focuses on Infrastructure as Code (IaC) security, specifically Terraform security in a DevSecOps workflow.

The content is divided into three primary sections:
- 1. Common Terraform Security Best Practices
- 2. Terraform Misconfiguration Scanning using Checkov
- 3. HashiCorp Vault for Secure Secret Management in CI/CD

### 1️⃣ Common Terraform Security Best Practices

When working with Terraform, especially in cloud environments like AWS, security must be embedded into the workflow.


### 🚫 Do Not Hardcode Credentials

Never hardcode:
- AWS Access Key
- AWS Secret Key
- Tokens
- Passwords

Instead:
- Use environment variables
- Use secrets management solutions
- Never commit credentials to Git


### 📄 Use .gitignore

Create a .gitignore file in every Terraform repository.

Must include:
```bash
.env
*.tfstate
*.tfstate.backup
id_rsa
*.pem
```

Why?
- Prevent accidental commit of credentials
- Prevent Terraform state exposure
- Prevent private key exposure

Commit .gitignore to the repository so all team members inherit it.

## 🛡 Use Pre-Commit Hooks

Even with .gitignore, developers may:
- Forget entries
- Hardcode credentials inside .tf files

Solution: Pre-Commit + GitLeaks

Install:
pip install pre-commit

Create .pre-commit-config.yaml:
```bash
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

Install hook:
pre-commit install

Now:
- Git commit is blocked if secrets are detected
- AWS keys, tokens, passwords are automatically scanned

🔁 Enforce via CI/CD

Developers may skip pre-commit locally.

Therefore:
- Integrate GitLeaks in CI/CD
- Run scan on every commit
- Run scan on every pull request

This ensures 100% enforcement at repository level.

### 2️⃣ Terraform Misconfiguration Scanning – Checkov
📌 Problem: Insecure Terraform Configuration

Example:

A junior engineer creates an S3 bucket using Terraform.

Bucket configuration:
block_public_acls       = false
block_public_policy     = false
restrict_public_buckets = false

Terraform syntax is valid.
Pre-commit does not detect anything.
CI pipeline passes.

But the bucket is public → Major security risk.

This is called:
Terraform Misconfiguration

🔎 Solution: Checkov

Checkov scans Terraform configuration for:

Security misconfigurations

Cloud security best practice violations

#### ▶️ Run Checkov

```bash
checkov -d .
```

Output shows:
- Total checks executed
- Passed checks
- Failed checks
- Security violations

#### ❌ Example Failures
- Ensure S3 bucket blocks public ACL
- Ensure public access block is enabled
- Ensure no public policies

#### ✅ Fix Configuration

Update:
block_public_acls       = true
block_public_policy     = true
ignore_public_acls      = true
restrict_public_buckets = true

Re-run:
```bash
checkov -d .
```
All checks pass.

How do you test Terraform security?

Answer: We integrate Checkov locally and in CI/CD to validate Terraform misconfigurations before infrastructure provisioning.

## 3️⃣ HashiCorp Vault – Secure Secret Management
### 📌 Problem: Long-Lived Credentials

Common insecure approach:
- 1. Create AWS IAM user
- 2. Generate access keys
- 3. Store keys in GitHub repository secrets
- 4. CI uses those keys permanently

Issues:
- Long-lived credentials
- Many people can access repository
- No accountability
- Risk of internal leakage
- Hard to rotate frequently

### 🔐 What Vault Solves

Vault:
- Eliminates long-lived credentials
- Generates short-lived credentials
- Credentials expire automatically (e.g., 10–15 minutes)

### 🏗 Production Workflow
Step 1 – Store Terraform in Git

Terraform code is stored in Git repository.

Step 2 – Developer Workflow
- 1. Clone repository
- 2. Update Terraform files
- 3. Create pull request
- 4. Code review
- 5. Merge

Step 3 – CI/CD Execution

CI (GitHub Actions):
- Triggers on merge
- Requests Vault for temporary AWS credentials
- Executes Terraform
- Credentials expire after execution

#### 🔁 Vault Workflow
- 1. GitHub Actions uses OIDC (OpenID Connect)
- 2. Authenticates with Vault
- 3. Vault generates temporary IAM credentials
- 4. CI uses credentials
- 5. Credentials expire automatically    

### 🔐 Vault Setup (High-Level)
- Install Vault on EC2
- Start Vault server
- Enable AWS secret engine
- Provide AWS IAM credentials to Vault
- Enable JWT authentication
- Configure GitHub OIDC
- Create policy
- Bind policy to GitHub repository

### 🔄 GitHub Actions + Vault + Terraform Flow
- 1. Developer pushes Terraform code
- 2. GitHub Actions runs
- 3. GitHub requests temporary credentials from Vault
- 4. Vault generates short-lived IAM user
- 5. Terraform runs using temporary credentials
- 6. IAM user expires automatically

### 📊 Advantages of CI-Based Terraform Execution
| Feature               | Benefit                      |
| --------------------- | ---------------------------- |
| Centralized execution | No personal credentials used |
| Code review           | Prevents junior mistakes     |
| Version control       | Easy rollback                |
| Auditing              | Track who changed what       |
| Security              | Temporary credentials        |


📌 Why Not Store Credentials in GitHub Secrets?

Even if stored in GitHub:
- Many users may access repository
- Credentials live for months
- Internal threats possible
- No strong accountability

Vault eliminates this risk by issuing short-lived tokens.

📋 Summary
Terraform Security Layers
- 1. .gitignore
- 2. Pre-commit + GitLeaks
- 3. CI/CD secret scanning
- 4. Checkov for misconfiguration
- 5. Store Terraform in Git
- 6. Execute via CI only
- 7. Use Vault for short-lived credentials


If in yout irganization people are not following the best practices then we can always create github workflow using the files in the current directory.
gitleaks-workflow.yml
