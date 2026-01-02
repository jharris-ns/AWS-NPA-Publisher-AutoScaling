# Secret Scanning Guide

## Overview

This project uses automated secret scanning to prevent accidental commits of sensitive credentials, API keys, tokens, and other secrets to the repository.

## Tools Used

### 1. detect-secrets (Primary Tool)
- **Purpose:** Comprehensive secret detection using 27+ specialized plugins
- **Version:** 1.5.0
- **Plugins Include:**
  - AWS Access Keys
  - Private SSH/RSA Keys
  - GitHub/GitLab Tokens
  - Azure Storage Keys
  - JWT Tokens
  - Basic Auth Credentials
  - High Entropy Strings (Base64, Hex)
  - API Keys (Slack, Stripe, Twilio, OpenAI, etc.)

### 2. detect-private-key (Built-in)
- **Purpose:** Quick detection of private SSH keys
- **Part of:** pre-commit-hooks standard library

## How It Works

### Pre-Commit Hook
Secret scanning runs automatically before every commit:

1. You run `git commit`
2. Pre-commit hooks execute
3. **detect-secrets** scans all staged files
4. If secrets are found → commit is **BLOCKED**
5. If no secrets found → commit proceeds

### Baseline System
The project maintains a baseline file (`.secrets.baseline`) that contains:
- Known "secrets" that are actually safe (example values, test data, etc.)
- Prevents false positives on legitimate code
- Only NEW secrets trigger alerts

## Setup (Already Completed)

The following setup has already been done:

```bash
# Install tools
pip install pre-commit detect-secrets

# Install pre-commit hooks
pre-commit install

# Generate baseline
detect-secrets scan --all-files --force-use-all-plugins > .secrets.baseline
```

## Usage

### Daily Workflow
Normal git workflow - no changes needed:

```bash
git add file.py
git commit -m "Add new feature"
# Secret scanning runs automatically
# Commit succeeds if no new secrets detected
```

### If Secrets Are Detected

When a commit is blocked due to secrets:

```
Detect Secrets in Code...................................................Failed
- hook id: detect-secrets
- exit code: 1

Potential secrets about to be added to git repo!

Secret Type: AWS Access Key
Location:    scripts/deploy.py:42
```

**What to do:**

1. **Remove the secret from the file**
   ```bash
   # Replace hardcoded secret with environment variable
   api_key = os.environ.get('API_KEY')
   ```

2. **Use AWS Secrets Manager or environment variables**
   - Store secrets in AWS Secrets Manager
   - Reference them via ARN in CloudFormation
   - Use environment variables for local development

3. **Try committing again**
   ```bash
   git add scripts/deploy.py
   git commit -m "Use environment variable for API key"
   ```

### Manual Scanning

Run secret detection manually on all files:

```bash
# Scan all files (doesn't check baseline)
detect-secrets scan --all-files

# Scan specific file
detect-secrets scan path/to/file.py

# Run all pre-commit hooks
pre-commit run --all-files

# Run only secret detection
pre-commit run detect-secrets --all-files
```

### Updating the Baseline

If you have legitimate "secrets" that are actually safe (e.g., example values in documentation):

```bash
# Regenerate baseline with current repository state
detect-secrets scan --all-files --force-use-all-plugins > .secrets.baseline

# Audit baseline to verify no real secrets
detect-secrets audit .secrets.baseline
```

**Baseline Audit Process:**
1. Run `detect-secrets audit .secrets.baseline`
2. For each detected "secret", mark as:
   - `y` = Real secret (should not be committed!)
   - `n` = False positive (safe to commit)
   - `s` = Skip decision for now

## What Gets Detected

### ✅ Secrets That Will Be Detected

- **AWS Access Keys:** `AKIAIOSFODNN7EXAMPLE`
- **Private Keys:** `-----BEGIN RSA PRIVATE KEY-----`
- **GitHub Tokens:** `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- **JWT Tokens:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
- **API Keys:** High-entropy strings like `sk_live_xxxxxxxxxxxxxxxxxxxx`
- **Basic Auth:** `http://user:password@example.com`
- **Passwords:** Variables named `password`, `secret`, `token` with values

### ❌ False Positives (Use Baseline)

Some legitimate code may trigger false positives:
- Example values in documentation
- Test fixtures with fake credentials
- High-entropy hash values (checksums, UUIDs)
- Placeholder secrets in templates (marked with comments)

These should be added to the baseline.

## Best Practices

### 1. Never Commit Real Secrets
- Use environment variables
- Use AWS Secrets Manager
- Reference secrets by ARN in CloudFormation
- Use IAM roles instead of access keys

### 2. Keep Baseline Clean
- Periodically audit baseline: `detect-secrets audit .secrets.baseline`
- Remove entries when code is deleted
- Document why baseline entries exist

### 3. Handle False Positives
If you have a legitimate high-entropy string:

```python
# pragma: allowlist secret
CHECKSUM = "a1b2c3d4e5f6..." # SHA256 checksum, not a secret
```

Or add to `.secrets.baseline` via audit.

### 4. Review Pre-Commit Failures
Don't bypass secret detection:
```bash
# ❌ DON'T DO THIS
git commit --no-verify  # Bypasses ALL pre-commit hooks

# ✅ DO THIS INSTEAD
# Remove the secret, then commit properly
```

## Baseline File Location

- **File:** `.secrets.baseline`
- **Format:** JSON
- **Version:** 1.5.0
- **Current Status:** Contains baseline for existing repository files

### Baseline Structure
```json
{
  "version": "1.5.0",
  "plugins_used": [...],  // 27 detection plugins
  "filters_used": [...],  // Heuristic filters
  "results": {            // Known safe "secrets"
    "file1.py": [...],
    "file2.yml": [...]
  }
}
```

## CI/CD Integration

The secret scanning is configured for both local development and CI/CD:

### Local (Pre-Commit)
- Runs before every `git commit`
- Blocks commits with new secrets
- Fast feedback loop

### CI/CD (Future)
Add to your CI pipeline:

```yaml
# .github/workflows/security.yml
- name: Secret Scanning
  run: |
    pip install detect-secrets
    detect-secrets scan --all-files --baseline .secrets.baseline
```

## Troubleshooting

### "No plugins to scan with" Error
**Cause:** Baseline created with newer version than pre-commit uses

**Fix:**
```bash
pre-commit autoupdate
pre-commit run detect-secrets --all-files
```

### Secret Detected in Safe File
**Cause:** Legitimate high-entropy value flagged as secret

**Fix:** Add to baseline via audit
```bash
detect-secrets audit .secrets.baseline
# Mark the finding as 'n' (not a secret)
```

### Hook Takes Too Long
**Cause:** Large repository with many files

**Optimization:**
```yaml
# .pre-commit-config.yaml
- id: detect-secrets
  files: ^(scripts|templates)/  # Only scan specific directories
```

## Additional Security Hooks

This project also includes:

### CFN Security (cfn-nag)
- Scans CloudFormation templates for security issues
- Checks IAM policies, security groups, encryption settings
- See `.cfn-nag-deny-list.yml` for suppressed rules

### Python Security (bandit)
- Scans Python code for security vulnerabilities
- Detects SQL injection, shell injection, weak crypto
- Only scans `scripts/` directory

## Support

### Documentation
- detect-secrets: https://github.com/Yelp/detect-secrets
- pre-commit: https://pre-commit.com/

### Common Questions

**Q: Can I bypass secret detection temporarily?**
A: No. If you must commit something urgent, fix the secret issue first or use environment variables.

**Q: What if I already committed a secret?**
A:
1. Rotate the secret immediately (revoke and create new one)
2. Use `git filter-branch` or BFG Repo-Cleaner to remove from history
3. Force push to remote (if allowed)
4. Update all systems using the old secret

**Q: How do I know if baseline is up to date?**
A: Run `detect-secrets scan --all-files --baseline .secrets.baseline` - should show 0 new findings

## Version History

- **v1.5.0 (2026-01-02):** Initial setup with 27 detection plugins
  - Added comprehensive baseline
  - Updated pre-commit configuration
  - Integrated with existing security hooks