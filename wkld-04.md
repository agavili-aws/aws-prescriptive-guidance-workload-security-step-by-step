# WKLD.04 - Prevent Application Secrets from Being Exposed

## 🎯 What This Control Does

This control implements tools and processes to prevent secrets from being accidentally committed to source code repositories. It establishes automated scanning and manual review processes to catch secrets before they're exposed.

## 🔍 Why This Matters

- **Prevents data breaches**: Exposed secrets in public repos are a common attack vector
- **Compliance**: Many regulations require protection of sensitive credentials
- **Reputation protection**: Avoids embarrassing public exposure of internal systems
- **Cost prevention**: Prevents unauthorized usage of paid services
- **Security hygiene**: Establishes good development practices

## ⏱️ Time Required
**15 minutes**

## 🛠️ Recommended Tools

### Git-secrets (AWS Labs)
- Prevents AWS credentials from being committed
- Integrates with Git hooks
- Scans existing repositories

### Gitleaks
- Detects secrets in Git repositories
- Supports many secret types
- Can run in CI/CD pipelines

### TruffleHog
- Searches for high-entropy strings
- Scans Git history
- Active development and updates

---

## 📋 Step-by-Step Instructions

### Step 1: Install and Configure Git-secrets

#### Install Git-secrets
```bash
# macOS with Homebrew
brew install git-secrets

# Linux/macOS manual install
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets
make install

# Windows (using Git Bash)
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets
./install.ps1
```

#### Configure for AWS Secrets
```bash
# Navigate to your repository
cd /path/to/your/repo

# Install git-secrets hooks
git secrets --install

# Add AWS patterns
git secrets --register-aws

# Add custom patterns
git secrets --add 'password\s*=\s*.+'
git secrets --add 'api[_-]?key\s*=\s*.+'
git secrets --add 'secret[_-]?key\s*=\s*.+'
```

### Step 2: Install and Configure Gitleaks

#### Install Gitleaks
```bash
# macOS with Homebrew
brew install gitleaks

# Linux
wget https://github.com/zricethezav/gitleaks/releases/download/v8.18.0/gitleaks_8.18.0_linux_x64.tar.gz
tar -xzf gitleaks_8.18.0_linux_x64.tar.gz
sudo mv gitleaks /usr/local/bin/

# Windows
# Download from GitHub releases and add to PATH
```

#### Create Gitleaks Configuration
```bash
# Create .gitleaks.toml in your repository root
cat > .gitleaks.toml << 'EOF'
title = "Gitleaks Configuration"

[extend]
useDefault = true

[[rules]]
description = "AWS Access Key ID"
id = "aws-access-key-id"
regex = '''AKIA[0-9A-Z]{16}'''

[[rules]]
description = "AWS Secret Access Key"
id = "aws-secret-access-key"
regex = '''[A-Za-z0-9/+=]{40}'''

[[rules]]
description = "Database Password"
id = "database-password"
regex = '''(?i)(password|pwd)\s*[:=]\s*['"][^'"]{8,}['"]'''

[[rules]]
description = "API Key"
id = "api-key"
regex = '''(?i)(api[_-]?key|apikey)\s*[:=]\s*['"][^'"]{16,}['"]'''

[allowlist]
description = "Allowlist for test files"
files = [
    '''.*test.*''',
    '''.*example.*''',
    '''.*\.md$'''
]
EOF
```

### Step 3: Set Up Pre-commit Hooks

#### Install Pre-commit Framework
```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/awslabs/git-secrets
    rev: master
    hooks:
      - id: git-secrets
        entry: git-secrets --scan
        types: [text]

  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: local
    hooks:
      - id: check-secrets
        name: Check for secrets
        entry: ./scripts/check-secrets.sh
        language: script
        pass_filenames: false
EOF

# Install the hooks
pre-commit install
```

#### Create Custom Secret Check Script
```bash
# Create scripts directory
mkdir -p scripts

# Create custom check script
cat > scripts/check-secrets.sh << 'EOF'
#!/bin/bash

# Custom secret detection script
echo "Running custom secret checks..."

# Check for common secret patterns
if grep -r -E "(password|pwd|secret|key)\s*[:=]\s*['\"][^'\"]{8,}['\"]" . --exclude-dir=.git --exclude-dir=node_modules; then
    echo "❌ Potential secrets found in code!"
    exit 1
fi

# Check for AWS credentials
if grep -r -E "AKIA[0-9A-Z]{16}" . --exclude-dir=.git; then
    echo "❌ AWS Access Key found!"
    exit 1
fi

echo "✅ No secrets detected"
exit 0
EOF

chmod +x scripts/check-secrets.sh
```

### Step 4: Integrate with CI/CD Pipeline

#### GitHub Actions Example
```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0

    - name: Run Gitleaks
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

    - name: Run TruffleHog
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD
```

#### GitLab CI Example
```yaml
# .gitlab-ci.yml
stages:
  - security

secret-detection:
  stage: security
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --verbose
  rules:
    - if: $CI_COMMIT_BRANCH
    - if: $CI_MERGE_REQUEST_ID
```

### Step 5: Scan Existing Repository

#### Scan with Gitleaks
```bash
# Scan current repository
gitleaks detect --source . --verbose

# Scan with custom config
gitleaks detect --config .gitleaks.toml --source . --verbose

# Generate report
gitleaks detect --source . --report-format json --report-path gitleaks-report.json
```

#### Scan with Git-secrets
```bash
# Scan all files
git secrets --scan

# Scan specific files
git secrets --scan path/to/file.js

# Scan entire history (can be slow)
git secrets --scan-history
```

#### Scan with TruffleHog
```bash
# Install TruffleHog
pip install trufflehog3

# Scan repository
trufflehog3 --format json --output trufflehog-report.json .
```

## ✅ Verification Steps

### Test Secret Detection
1. **Create test file with fake secret**:
   ```bash
   echo 'password = "supersecretpassword123"' > test-secret.txt
   git add test-secret.txt
   git commit -m "Test secret detection"
   ```

2. **Verify tools catch the secret**:
   - Pre-commit hooks should block the commit
   - Manual scans should detect the secret

3. **Clean up test**:
   ```bash
   git reset --hard HEAD~1
   rm test-secret.txt
   ```

### Test CI/CD Integration
1. **Create pull request** with intentional secret
2. **Verify CI pipeline fails** with secret detection
3. **Remove secret** and verify pipeline passes

## 💡 Best Practices

### Development Workflow
- **Never commit real secrets**: Use placeholder values in code
- **Use environment variables**: For local development configuration
- **Regular scanning**: Run secret scans weekly on all repositories
- **Team training**: Educate developers on secret management

### Tool Configuration
- **Custom patterns**: Add patterns specific to your organization
- **Allowlist carefully**: Only allowlist files that truly need exceptions
- **Multiple tools**: Use different tools for comprehensive coverage
- **Regular updates**: Keep secret detection tools updated

### Incident Response
- **Immediate rotation**: Rotate any exposed secrets immediately
- **Git history cleanup**: Use tools like BFG Repo-Cleaner for history
- **Notification process**: Alert security team of any exposures
- **Post-incident review**: Analyze how secrets were exposed

## 🚨 Common Mistakes to Avoid

- ❌ Only scanning new commits (not existing history)
- ❌ Too many false positives (poorly configured patterns)
- ❌ Not integrating with CI/CD pipelines
- ❌ Ignoring tool warnings without investigation
- ❌ Not training developers on proper secret handling
- ❌ Using the same secrets across environments

## 🔧 Advanced Configuration

### Custom Gitleaks Rules
```toml
[[rules]]
description = "Custom API Token"
id = "custom-api-token"
regex = '''(?i)mycompany[_-]?token\s*[:=]\s*['"][a-zA-Z0-9]{32}['"]'''
tags = ["api", "token"]

[[rules]]
description = "Database Connection String"
id = "db-connection"
regex = '''(?i)(mongodb|mysql|postgres)://[^\s'"]+'''
tags = ["database", "connection"]
```

### Git-secrets Custom Patterns
```bash
# Add custom patterns for your organization
git secrets --add 'mycompany[_-]?api[_-]?key\s*[:=]\s*.+'
git secrets --add 'internal[_-]?token\s*[:=]\s*.+'
git secrets --add --literal 'DO_NOT_COMMIT'
```

### Automated Remediation
```bash
#!/bin/bash
# auto-remediate.sh - Automatically fix common secret patterns

# Replace hardcoded passwords with environment variable references
sed -i 's/password = "[^"]*"/password = os.getenv("DB_PASSWORD")/g' *.py

# Replace API keys
sed -i 's/api_key = "[^"]*"/api_key = os.getenv("API_KEY")/g' *.py

echo "Automated remediation complete. Please review changes."
```

## 🔗 Official AWS Documentation

- [Git-secrets (AWS Labs)](https://github.com/awslabs/git-secrets)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [Secrets Manager Best Practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html)

### Third-Party Tools
- [Gitleaks](https://github.com/zricethezav/gitleaks)
- [TruffleHog](https://github.com/trufflesecurity/truffleHog)
- [Whispers](https://github.com/Skyscanner/whispers)
- [detect-secrets](https://github.com/Yelp/detect-secrets)

## ➡️ What's Next?

Great job setting up secret detection! Now let's add monitoring to detect if secrets have already been exposed:

**Next Control**: [WKLD.05 - Detect and Remediate Exposed Secrets](./wkld-05.md)

---

[← Previous: WKLD.03](./wkld-03.md) | [Back to Main Guide](./README.md) | [Next: WKLD.05 →](./wkld-05.md)
