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
    steps:
      - uses: actions/checkout@v4
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
          aws-region: us-east-1
      
      - name: Run Q CLI
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          q chat --no-interactive "Review this code for security issues"
```

## Features

- **Automatic caching** - Caches Q CLI binaries for faster subsequent runs
- **SIGV4 authentication** - Optional IAM-based authentication (undocumented feature)
- **Linux support** - Supports x64 and arm64 architectures
- **Lightweight** - Composite action with no external dependencies
- **Multiple binaries** - Installs `q`, `qchat`, and `qterm`

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

| OS | Architecture | Status |
|----|--------------|--------|
| Ubuntu | x64 | ✅ Supported |
| Ubuntu | arm64 | ✅ Supported |
| macOS | - | ❌ Not supported |
| Windows | - | ❌ Not supported |

**Note:** macOS binaries are not available via AWS CDN (403 Forbidden). macOS users should install Q CLI using official AWS methods.

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
    q chat --no-interactive "What is 2+2?"
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
  run: q chat --no-interactive "Review this code"
```

## Examples

### Example 1: Basic Code Review

```yaml
name: Q CLI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
      
      - name: Generate and review diff
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > changes.diff
          q chat --no-interactive "Review this diff for bugs: $(cat changes.diff)"
```

### Example 2: Terraform Security Scan

```yaml
name: Terraform Security Review
on: [push]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: clouatre-labs/setup-q-cli-action@v1
        with:
          enable-sigv4: true
          aws-region: ca-central-1
      
      - name: Scan Terraform files
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          for file in $(find . -name "*.tf"); do
            echo "Scanning $file..."
            q chat --no-interactive "Review this Terraform for security issues: $(cat $file)"
          done
```

### Example 3: Using Specific Version

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    version: '1.19.6'  # Pin to specific version (no 'v' prefix needed)
```

**Note:** The action accepts versions with or without the 'v' prefix (e.g., both `1.19.6` and `v1.19.6` work). The AWS CDN uses versions without the prefix.

### Example 4: With SHA256 Verification

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
  with:
    version: '1.19.6'
    verify-checksum: true  # Adds ~2s on first install, 0s on cache hits
```

**When to use:** Enable for security-sensitive environments or compliance requirements. Verification is skipped when using cached binaries.

## How It Works

1. Checks if running on Linux (macOS not supported)
2. Checks cache for Q CLI binaries matching version and platform
3. If cache miss, downloads from AWS CDN: `https://desktop-release.q.us-east-1.amazonaws.com/{version}/q-{arch}-linux.zip`
4. Optionally verifies SHA256 checksum (if `verify-checksum: true`)
5. Extracts and installs binaries (`q`, `qchat`, `qterm`) to `~/.local/bin/`
6. Adds binary location to `$GITHUB_PATH`
7. Optionally configures SIGV4 authentication
8. Verifies installation with `q --version`

## Cache Key Format

```
q-{version}-{os}-{arch}
```

Example: `q-latest-Linux-X64`

## Troubleshooting

### Binary not found after installation

Ensure you're using the action before attempting to run `q`:

```yaml
- uses: clouatre-labs/setup-q-cli-action@v1
- run: q --version  # This will work
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
