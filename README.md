# Setup Q CLI Action

[![Test Action](https://github.com/clouatre-labs/setup-q-cli-action/actions/workflows/test.yml/badge.svg)](https://github.com/clouatre-labs/setup-q-cli-action/actions/workflows/test.yml)
[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Setup%20Q%20CLI-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/setup-q-cli)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Latest Release](https://img.shields.io/github/v/release/clouatre-labs/setup-q-cli-action)](https://github.com/clouatre-labs/setup-q-cli-action/releases/latest)

GitHub Action to install and cache [Amazon Q Developer CLI](https://github.com/aws/amazon-q-developer-cli) for use in workflows.

**Unofficial community action.** Not affiliated with or endorsed by Amazon Web Services (AWS). "Amazon Q" and "Amazon Web Services" are trademarks of AWS.

## Quick Start

```yaml
name: AI Code Review
on: [pull_request]

permissions:
  id-token: write      # Required for OIDC
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: us-east-1
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Generate and post code review
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > changes.diff
          qchat chat --no-interactive "Review this diff: $(cat changes.diff)" > review.md
      
      - uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.md', 'utf8');
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## AI Code Review\n\n${review}`
            });
```

## Features

- **Automatic caching** - Caches Q CLI binaries for faster subsequent runs
- **SIGV4 authentication** - IAM-based headless authentication for CI/CD
- **GitHub-hosted runners** - Supports x64 Ubuntu runners (simple, fast, manageable)
- **Lightweight** - Composite action with no external dependencies

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | Q CLI version to install | No | `latest` |
| `aws-region` | AWS region for Q CLI operations | No | `us-east-1` |
| `enable-sigv4` | Enable SIGV4 authentication mode | No | `false` |
| `verify-checksum` | Verify SHA256 checksum of downloaded binary | No | `false` |

## Outputs

| Output | Description |
|--------|-------------|
| `q-version` | Installed Q CLI version |
| `q-path` | Path to Q CLI binary directory |

## Supported Platforms

**GitHub-hosted runners only** - Designed for simple, fast, manageable CI/CD.

| OS | Architecture | Runner Label |
|----|--------------|--------------|
| Ubuntu | x64 | `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` |

**Not supported:** macOS, Windows (binaries not available via AWS CDN). Self-hosted ARM64 runners may work but are untested.

## Authentication Methods

### Method 1: OIDC (Recommended for GitHub Actions)

Uses GitHub's OIDC provider for secure, credential-free authentication.

**One-time AWS Setup:**

1. Create OIDC provider:
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

2. Create IAM role with trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:<ORG>/*:*"
      }
    }
  }]
}
```

3. Attach Q Developer policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "q:StartConversation",
      "q:SendMessage",
      "q:GetConversation"
    ],
    "Resource": "*"
  }]
}
```

**Workflow usage:**
```yaml
permissions:
  id-token: write  # Required for OIDC

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
    aws-region: us-east-1

- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    enable-sigv4: true  # Required with OIDC
```

**Benefits:**
- No long-lived credentials in GitHub Secrets
- Automatic token rotation (1-hour sessions)
- Scope to specific repos/branches
- AWS security best practice

### Method 2: IAM User Credentials (Local Development)

For local testing or non-GitHub CI/CD environments.

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  # Do NOT set enable-sigv4 with long-lived credentials

- name: Use Q CLI
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-1
  run: qchat chat --no-interactive "What is 2+2?"
```

**Important:** SIGV4 mode requires temporary credentials (session token). Do not use `enable-sigv4: true` with IAM user credentials (AKIA* keys).

## Examples

### Security Scan with Artifact Upload

```yaml
name: Security Scan
on: [push]

permissions:
  id-token: write
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: us-east-1
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Scan files
        run: |
          mkdir -p reports
          find . -name "*.py" -o -name "*.js" | head -5 | while read file; do
            qchat chat --no-interactive "Security review: $(cat $file)" >> reports/scan.txt
            echo -e "\n---\n" >> reports/scan.txt
          done
      
      - uses: actions/upload-artifact@v4
        with:
          name: security-scan
          path: reports/
          retention-days: 30
```

### Pin to Specific Version

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    version: '1.19.6'
    verify-checksum: true  # Recommended for production
```

## How It Works

1. Checks if running on Linux (macOS not supported)
2. Checks cache for Q CLI binary matching version and platform
3. If cache miss, downloads from AWS CDN: `https://desktop-release.q.us-east-1.amazonaws.com/{version}/q-{arch}-linux.zip`
4. Optionally verifies SHA256 checksum (if `verify-checksum: true`)
5. Extracts and installs `qchat` binary to `~/.local/bin/`
6. Adds binary location to `$GITHUB_PATH`
7. Optionally configures SIGV4 authentication
8. Verifies installation with `qchat --version`

**Note:** Only the `qchat` binary is installed (130MB). This is sufficient for CI/CD use cases. The `q` wrapper (99MB) and `qterm` (74MB) are not needed for automated workflows.

## Cache Key Format

```
q-{version}-{os}-{arch}
```

Example: `q-latest-Linux-X64`

## Troubleshooting

### Binary not found after installation

Ensure you're using the action before attempting to run `qchat`:

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
- run: qchat --version  # This will work
```

### SIGV4 authentication not working

Verify:
1. `enable-sigv4: true` is set in action inputs
2. AWS credentials are available as environment variables
3. IAM permissions include Amazon Q access
4. Correct AWS region is configured

### Unsupported platform error

Q CLI binaries are only available for Linux via AWS CDN. Use `ubuntu-latest`, `ubuntu-24.04`, or `ubuntu-22.04` runners:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest  # Recommended
```

### Cache not working

The cache key includes OS and architecture. If you change runners or platforms, a new cache entry will be created. This is expected behavior.

### SHA256 verification failed

If checksum verification fails:

1. **Retry the workflow** - May be a transient CDN issue
2. **Check AWS CDN status** - Verify https://status.aws.amazon.com/
3. **Disable verification temporarily:**
   ```yaml
   verify-checksum: false
   ```
4. **Report the issue** - If problem persists, open an issue with the version number

## Development

This is a composite action (YAML-based) with no compilation required.

### Testing Locally

```bash
# Clone the repository
git clone https://github.com/clouatre-labs/setup-q-cli-action
cd setup-q-cli-action

# Test in a workflow (see .github/workflows/test.yml)
```

## Contributing

Contributions are welcome! Please open an issue or PR.

## License

MIT - See [LICENSE](LICENSE)

## Related

- [Amazon Q Developer CLI](https://github.com/aws/amazon-q-developer-cli) - Official Q CLI repository (Apache 2.0)
- [Q CLI Documentation](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line.html) - Official AWS documentation
- [Setup Goose Action](https://github.com/clouatre-labs/setup-goose-action) - Similar action for Goose AI agent

## Acknowledgments

Built by [clouatre-labs](https://github.com/clouatre-labs) for the developer community.

**Trademark Notice:** "Amazon Q" and "Amazon Web Services" are trademarks of Amazon.com, Inc. or its affiliates. This project is not affiliated with, endorsed by, or sponsored by Amazon Web Services.

**SIGV4 Discovery:** The `AMAZON_Q_SIGV4` authentication mechanism was discovered through source code analysis of the [amazon-q-developer-cli](https://github.com/aws/amazon-q-developer-cli) repository. It is an undocumented feature that enables headless IAM authentication for CI/CD environments.
