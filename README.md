# Setup Q CLI Action

[![Test Action](https://github.com/clouatre-labs/setup-q-cli-action/actions/workflows/test.yml/badge.svg)](https://github.com/clouatre-labs/setup-q-cli-action/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Latest Release](https://img.shields.io/github/v/release/clouatre-labs/setup-q-cli-action)](https://github.com/clouatre-labs/setup-q-cli-action/releases/latest)

GitHub Action to install and cache [Amazon Q Developer CLI](https://github.com/aws/amazon-q-developer-cli) for use in workflows.

**Unofficial community action.** Not affiliated with or endorsed by Amazon Web Services (AWS). "Amazon Q" and "Amazon Web Services" are trademarks of AWS.

## Quick Start

```yaml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Generate code review
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > changes.diff
          qchat chat --no-interactive "Review this diff for bugs: $(cat changes.diff)" > review.md
      
      - name: Post review as PR comment
        uses: actions/github-script@v7
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

### Method 1: SIGV4 Authentication (IAM)

SIGV4 authentication uses AWS IAM credentials for headless operation in CI/CD.

**Discovered feature:** `AMAZON_Q_SIGV4` environment variable (added in commit [42b16763](https://github.com/aws/amazon-q-developer-cli/commit/42b16763), June 2025).

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    enable-sigv4: true
    aws-region: ca-central-1

- name: Use Q CLI with IAM credentials
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: |
    qchat chat --no-interactive "What is 2+2?"
```

**Required IAM Permissions:**
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

**Reference:** [AWS Q Developer IAM Permissions](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/security_iam_permissions.html)

### Method 2: Standard AWS Authentication

Q CLI respects the standard AWS credential chain:
- Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- AWS profiles (`AWS_PROFILE`)
- IAM roles (when running on EC2, ECS, Lambda)

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1

- name: Use Q CLI
  run: qchat chat --no-interactive "Review this code"
```

## Examples

### Example 1: Post Code Review as PR Comment

```yaml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Generate code review
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > changes.diff
          qchat chat --no-interactive "Review this diff for bugs, security issues, and best practices: $(cat changes.diff)" > review.md
      
      - name: Post review as PR comment
        uses: actions/github-script@v7
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

### Example 2: Save Security Scan as Artifact

```yaml
name: Security Scan
on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Scan for security issues
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          mkdir -p reports
          find . -name "*.py" -o -name "*.js" -o -name "*.tf" | while read file; do
            echo "Scanning $file..." >&2
            qchat chat --no-interactive "Analyze this file for security vulnerabilities: $(cat $file)" >> reports/security-scan.txt
            echo -e "\n---\n" >> reports/security-scan.txt
          done
      
      - name: Upload scan results
        uses: actions/upload-artifact@v4
        with:
          name: security-scan-results
          path: reports/
          retention-days: 30
```

### Example 3: Pin to Specific Version

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    version: '1.19.6'  # Accepts with or without 'v' prefix
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
