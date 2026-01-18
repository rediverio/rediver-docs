---
layout: default
title: Agent Configuration
parent: Guides
nav_order: 14
---
# Agent Configuration Guide

Configure the Rediver Agent for your security scanning needs.

---

## Quick Start

1. Copy the template file:
```bash
cp agent.yaml.template agent.yaml
```

2. Update with your settings:
```yaml
rediver:
  base_url: "https://api.rediver.io"
  api_key: "your-api-key"
```

3. Run the agent:
```bash
agent -config agent.yaml -daemon
```

---

## Configuration File

### Minimal Configuration

```yaml
agent:
  name: "My Scanner"

server:
  base_url: "https://api.rediver.io"
  api_key: "your-api-key"

scanners:
  - name: semgrep
    enabled: true
```

### Full Configuration

```yaml
# Agent Settings
agent:
  name: "Security Scanner"
  enable_commands: true
  command_poll_interval: 30s
  heartbeat_interval: 1m
  log_level: info

# Platform Connection
server:
  base_url: "https://api.rediver.io"
  api_key: "your-api-key"
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

# Targets
targets:
  - /path/to/project

# Output
output:
  format: json
  sarif: true

# CI/CD
ci:
  comments: true
  push: true
  fail_on: critical

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

## Configuration Options

### Agent Section

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `name` | string | `""` | Display name in Rediver UI |
| `enable_commands` | bool | `true` | Accept remote commands |
| `command_poll_interval` | duration | `30s` | Poll interval for commands |
| `heartbeat_interval` | duration | `1m` | Heartbeat frequency |
| `log_level` | string | `info` | Log level: debug, info, warn, error |

### Server Section

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `base_url` | string | required | Rediver API URL |
| `api_key` | string | required | API authentication key |
| `worker_id` | string | auto | Unique worker identifier |
| `timeout` | duration | `30s` | Request timeout |
| `max_retries` | int | `3` | Max retry attempts |
| `retry_delay` | duration | `2s` | Delay between retries |

### Scanners Section

| Scanner | Type | Description |
|---------|------|-------------|
| `semgrep` | SAST | Static analysis with taint tracking |
| `gitleaks` | Secret | Secret and credential detection |
| `trivy` | SCA | Dependency vulnerability scanning |
| `trivy-config` | IaC | Infrastructure as Code scanning |
| `trivy-image` | Container | Container image scanning |

**Scanner Configuration:**

```yaml
scanners:
  - name: semgrep
    enabled: true
    binary: /custom/path/semgrep  # Optional
    config:
      rules: ["p/default", "p/security-audit"]
      exclude: ["*_test.go"]

  - name: gitleaks
    enabled: true
    config:
      config_path: ".gitleaks.toml"

  - name: trivy
    enabled: true
    config:
      scanners: ["vuln", "secret"]
      severity: ["HIGH", "CRITICAL"]
```

### Output Section

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `format` | string | `json` | Output format: json, sarif, text |
| `file` | string | stdout | Output file path |
| `sarif` | bool | `false` | Generate SARIF report |
| `sarif_file` | string | `""` | SARIF output file |

### CI Section

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `comments` | bool | `true` | Create inline PR/MR comments |
| `push` | bool | `true` | Push results to platform |
| `fail_on` | string | `critical` | Fail on severity level |

---

## Environment Variables

Configuration values can use environment variables with `${VAR_NAME}` syntax:

```yaml
server:
  base_url: ${API_URL}
  api_key: ${API_KEY}
  worker_id: ${WORKER_ID}
```

### Supported Environment Variables

| Variable | Description |
|----------|-------------|
| `API_URL` | Platform API URL |
| `API_KEY` | API key for authentication |
| `WORKER_ID` | Worker identifier |
| `GITHUB_TOKEN` | GitHub token for PR comments |
| `GITLAB_TOKEN` | GitLab token for MR comments |

---

## Usage Examples

### CLI Mode

```bash
# Single scan with config
agent -config agent.yaml -target ./src

# Override scanners
agent -config agent.yaml -tools semgrep,gitleaks -target .

# Push results to platform
agent -config agent.yaml -target . -push
```

### Daemon Mode

```bash
# Start daemon
agent -config agent.yaml -daemon

# Daemon with verbose logging
agent -config agent.yaml -daemon -verbose
```

### Docker

```bash
# With config file
docker run --rm \
  -v $(pwd):/scan \
  -v $(pwd)/agent.yaml:/config/agent.yaml \
  rediverio/agent:latest \
  -config /config/agent.yaml -target /scan

# With environment variables
docker run --rm \
  -v $(pwd):/scan \
  -e API_URL=https://api.rediver.io \
  -e API_KEY=your-key \
  rediverio/agent:latest \
  -tools semgrep,gitleaks,trivy -target /scan -push
```

---

## Configuration Precedence

Configuration is applied in this order (later overrides earlier):

1. Default values
2. Configuration file (`-config`)
3. Environment variables
4. Command-line flags

**Example:**
```bash
# api_key from config file is overridden by environment variable
API_KEY=new-key agent -config agent.yaml
```

---

## Validation

Validate configuration before running:

```bash
agent -config agent.yaml -validate
```

Check available scanners:

```bash
agent -list-tools
agent -check-tools
```

---

## Security Best Practices

1. **Never commit API keys** - Use environment variables
2. **Use least privilege** - Only enable required scanners
3. **Secure config files** - Set proper file permissions
4. **Rotate keys regularly** - Generate new API keys periodically

```bash
# Secure file permissions
chmod 600 agent.yaml
```

---

## Related Documentation

- [Docker Deployment Guide](./docker-deployment.md)
- [SDK Development Guide](./sdk-development.md)
- [CI/CD Examples](https://github.com/rediverio/sdk/tree/main/examples/ci-cd)
