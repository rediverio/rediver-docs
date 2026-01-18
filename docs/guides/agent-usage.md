---
layout: default
title: Agent Usage Guide
parent: Guides
nav_order: 12
---

# Rediver Agent Usage Guide

The Rediver Agent is a command-line security scanning tool that integrates with the Rediver platform.

---

## Installation

### Option 1: Go Install

```bash
go install github.com/rediverio/agent@latest
```

### Option 2: Build from Source

```bash
git clone https://github.com/rediverio/agent.git
cd agent
go build -o agent .
```

### Option 3: Docker

```bash
# Pull from Docker Hub
docker pull rediverio/agent:latest

# Available variants
docker pull rediverio/agent:full    # All tools included (~1GB)
docker pull rediverio/agent:slim    # Agent only (~20MB)
docker pull rediverio/agent:ci      # CI/CD optimized (~1.2GB)
```

---

## Quick Start

### 1. Create a Worker in Rediver UI

1. Navigate to **Scoping > Workers**
2. Click **+ Add Worker**
3. Fill in details (name, type, capabilities)
4. **Copy the API key** (shown only once!)

### 2. Run Your First Scan

```bash
# Set credentials
export API_URL=https://api.rediver.io
export API_KEY=rdw_your_api_key_here

# Run a scan and push results
./agent -tool semgrep -target /path/to/project -push -verbose
```

---

## Command Line Options

### Basic Options

| Flag | Description | Example |
|------|-------------|---------|
| `-tool` | Single tool to run | `-tool semgrep` |
| `-tools` | Multiple tools (comma-separated) | `-tools semgrep,gitleaks,trivy` |
| `-target` | Path or URL to scan | `-target ./src` |
| `-push` | Push results to Rediver | `-push` |
| `-verbose` | Enable verbose logging | `-verbose` |
| `-config` | Path to config file | `-config agent.yaml` |

### Execution Modes

| Flag | Description |
|------|-------------|
| `-daemon` | Run as continuous daemon |
| `-list-tools` | List available tools |
| `-check-tools` | Check tool installation status |

### CI/CD Options

| Flag | Description |
|------|-------------|
| `-auto-ci` | Auto-detect CI environment |
| `-comments` | Post findings as PR/MR comments |
| `-sarif` | Generate SARIF output |
| `-sarif-output` | SARIF output file path |

---

## Available Scanners

| Tool | Type | Description |
|------|------|-------------|
| `semgrep` | SAST | Code analysis with dataflow/taint tracking |
| `gitleaks` | Secret | Secret and credential detection |
| `trivy` | SCA | Vulnerability scanning (filesystem) |
| `trivy-config` | IaC | Infrastructure misconfiguration |
| `trivy-image` | Container | Container image scanning |
| `trivy-full` | All | vuln + misconfig + secret |

### Check Tool Status

```bash
# List all available tools
./agent -list-tools

# Check which tools are installed
./agent -check-tools
```

---

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `API_URL` | Yes* | Platform API URL |
| `API_KEY` | Yes* | Worker API key |
| `WORKER_ID` | No | Worker UUID for tracking |
| `GITHUB_TOKEN` | Auto | GitHub token (for PR comments) |
| `GITLAB_TOKEN` | Auto | GitLab token (for MR comments) |

*Required when using `-push` flag

### Configuration File

Create `agent.yaml`:

```yaml
# Agent settings
agent:
  name: "prod-scanner-01"
  enable_commands: true
  command_poll_interval: 30s
  heartbeat_interval: 1m
  log_level: info

# Platform connection
server:
  base_url: "https://api.rediver.io"
  api_key: "rdw_your_api_key"
  worker_id: ""  # Auto-generated if empty
  timeout: 30s
  max_retries: 3
  retry_delay: 2s

# Scanners
scanners:
  - name: semgrep
    enabled: true

  - name: gitleaks
    enabled: true

  - name: trivy
    enabled: true

# Targets for daemon mode
targets:
  - /opt/code/project1

# Output settings
output:
  format: json
  sarif: false

# CI/CD settings
ci:
  comments: true
  fail_on: critical
  push: true

# Advanced
advanced:
  max_concurrent: 2
  scan_timeout: 30m
  cache_dir: /tmp/app-cache
  exclude:
    - "**/node_modules/**"
    - "**/vendor/**"
    - "**/.git/**"
```

---

## Execution Modes

### One-Shot Mode (Single Scan)

Run a scan and exit:

```bash
# Single tool
./agent -tool semgrep -target ./src -push

# Multiple tools
./agent -tools semgrep,gitleaks,trivy -target . -push

# With config file
./agent -config agent.yaml -push

# Dry run (no push)
./agent -tool semgrep -target . -output ./results.json
```

> **Important**: In one-shot mode, use `-push` to send results to Rediver.

### Daemon Mode (Continuous)

Run as a long-running service:

```bash
# Start daemon
./agent -daemon -config agent.yaml

# With verbose logging
./agent -daemon -config agent.yaml -verbose

# Standalone (no server commands)
./agent -daemon -config agent.yaml -standalone
```

> **Note**: In daemon mode, findings are automatically pushed after each scan.

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Security Scan
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Security Scan
        uses: docker://rediverio/agent:ci
        with:
          args: >-
            -tools semgrep,gitleaks,trivy
            -target .
            -auto-ci
            -comments
            -push
            -verbose
            -sarif
            -sarif-output results.sarif
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          API_URL: ${{ secrets.API_URL }}
          API_KEY: ${{ secrets.API_KEY }}

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

### GitLab CI

```yaml
stages:
  - security

security-scan:
  stage: security
  image: rediverio/agent:ci
  variables:
    GITLAB_TOKEN: $CI_JOB_TOKEN
    API_URL: $API_URL
    API_KEY: $API_KEY
  script:
    - |
      agent \
        -tools semgrep,gitleaks,trivy \
        -target . \
        -auto-ci \
        -comments \
        -push \
        -verbose \
        -sarif \
        -sarif-output gl-sast-report.json
  artifacts:
    reports:
      sast: gl-sast-report.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### CI Features

| Feature | Flag | Description |
|---------|------|-------------|
| Auto CI detection | `-auto-ci` | Detects GitHub/GitLab automatically |
| Inline comments | `-comments` | Posts findings as PR/MR comments |
| Push to platform | `-push` | Sends results to Rediver |
| SARIF output | `-sarif` | Generates SARIF for security dashboards |
| Diff-based scan | Automatic | Only scans changed files in PR/MR |

---

## Docker Usage

### Basic Scan

```bash
docker run --rm -v $(pwd):/scan rediverio/agent:latest \
    -tools semgrep,gitleaks,trivy \
    -target /scan \
    -verbose
```

### With API Push

```bash
docker run --rm \
    -v $(pwd):/scan \
    -e API_URL=https://api.rediver.io \
    -e API_KEY=rdw_your_api_key \
    rediverio/agent:latest \
    -tools semgrep,gitleaks,trivy \
    -target /scan \
    -push
```

### Docker Compose

```yaml
version: '3.8'
services:
  agent:
    image: rediverio/agent:latest
    volumes:
      - ./:/scan:ro
      - ./agent.yaml:/app/agent.yaml
    environment:
      - API_KEY=${API_KEY}
    command: -daemon -config /app/agent.yaml
```

---

## Running as a Service

### Systemd

Create `/etc/systemd/system/rediver-agent.service`:

```ini
[Unit]
Description=Rediver Security Scanner Agent
After=network.target

[Service]
Type=simple
User=rediver
Group=rediver
WorkingDirectory=/opt/rediver
ExecStart=/opt/rediver/agent -daemon -config /opt/rediver/agent.yaml
Restart=always
RestartSec=10
Environment=API_KEY=rdw_your_api_key

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rediver-agent
sudo systemctl start rediver-agent
```

### Check Service Status

```bash
sudo systemctl status rediver-agent
journalctl -u rediver-agent -f
```

---

## Troubleshooting

### Worker Shows "Inactive"

Workers become inactive if no heartbeat is received for 5 minutes.

**Causes:**
- Agent not running
- API key invalid
- Network issues

**Solutions:**
1. Check agent is running: `ps aux | grep agent`
2. Check logs for errors
3. Verify API key is correct
4. Restart the agent

### 401 Unauthorized

- API key is invalid or revoked
- Regenerate key in UI: Workers > ... > Regenerate API Key

### Findings Not Appearing

**Most common cause**: Missing `-push` flag!

```bash
# WRONG
./agent -config agent.yaml

# CORRECT
./agent -config agent.yaml -push
```

Other causes:
- Check for "Pushed: X created" in output
- Verify API connection with `-verbose`
- Ensure asset exists in Rediver

### Tool Not Found

```bash
# Check tool installation
./agent -check-tools

# Install missing tools
./agent -install-tools
```

---

## Next Steps

- **[Running Workers](running-workers.md)** - Create workers in Rediver UI
- **[SDK Quick Start](sdk-quick-start.md)** - Use SDK directly
- **[Custom Tools Development](custom-tools-development.md)** - Build your own tools
