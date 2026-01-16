---
layout: default
---
# Rediver SDK Development Guide

This guide shows how to use the **rediver-sdk** to build custom scanners, collectors, and agents.

---

## Overview

The `rediver-sdk` provides a complete toolkit for building security scanning and data collection tools that integrate with Rediver. Key features:

- **Base implementations** - Extend `BaseScanner`, `BaseCollector`, `BaseAgent` for rapid development
- **Preset scanners** - Ready-to-use configurations for popular tools
- **Parser registry** - Built-in SARIF and JSON parsers with plugin support
- **RIS types** - Complete type definitions for Rediver Ingest Schema
- **Push/Pull modes** - Support for both agent-based and collector-based architectures

---

## Installation

```bash
go get github.com/rediverio/rediver-sdk
```

---

## Quick Start

### Run a Preset Scanner

```go
package main

import (
    "context"
    "fmt"

    "github.com/rediverio/rediver-sdk/sdk/core"
)

func main() {
    ctx := context.Background()

    // Create a preset scanner (semgrep, trivy-fs, gitleaks, slither, etc.)
    scanner, _ := core.NewPresetScanner("semgrep")

    // Check if installed
    installed, version, _ := scanner.IsInstalled(ctx)
    if !installed {
        fmt.Println("Semgrep not installed")
        return
    }
    fmt.Printf("Using %s version %s\n", scanner.Name(), version)

    // Run scan
    result, _ := scanner.Scan(ctx, "/path/to/project", &core.ScanOptions{
        Verbose: true,
    })

    // Parse results
    parsers := core.NewParserRegistry()
    parser := parsers.FindParser(result.RawOutput)
    report, _ := parser.Parse(ctx, result.RawOutput, nil)

    fmt.Printf("Found %d findings\n", len(report.Findings))
}
```

### List Available Preset Scanners

```go
for _, name := range core.ListPresetScanners() {
    cfg := core.PresetScanners[name]
    fmt.Printf("%-15s - %s (%s)\n", name, cfg.Name, strings.Join(cfg.Capabilities, ", "))
}
```

**Available presets:**
| Name | Tool | Capabilities |
|------|------|--------------|
| `semgrep` | Semgrep | sast, secret |
| `trivy-fs` | Trivy (filesystem) | vulnerability, secret |
| `trivy-config` | Trivy (IaC) | misconfiguration |
| `gitleaks` | Gitleaks | secret |
| `slither` | Slither | web3 |
| `checkov` | Checkov | misconfiguration |
| `bandit` | Bandit | sast |
| `gosec` | Gosec | sast |
| `snyk` | Snyk Code | sast |
| `codeql` | CodeQL | sast |

---

## Building Custom Scanners

### Method 1: Use BaseScanner Directly

```go
package main

import (
    "context"
    "time"

    "github.com/rediverio/rediver-sdk/sdk/core"
)

func main() {
    // Create a custom scanner configuration
    scanner := core.NewBaseScanner(&core.BaseScannerConfig{
        Name:    "my-scanner",
        Version: "1.0.0",
        Binary:  "my-tool",

        // Arguments with {target} placeholder
        DefaultArgs: []string{
            "scan",
            "--format", "sarif",
            "--output", "-",
            "{target}",
        },

        Timeout:      30 * time.Minute,
        OKExitCodes:  []int{0, 1}, // Exit codes that indicate success
        Capabilities: []string{"vulnerability", "secret"},
        Verbose:      true,
    })

    // Run scan
    result, err := scanner.Scan(context.Background(), "/path/to/scan", nil)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Output: %s\n", result.RawOutput)
}
```

### Method 2: Embed BaseScanner (Recommended)

For more control, embed `BaseScanner` in your own struct:

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/rediverio/rediver-sdk/sdk/core"
)

// MyScanner extends BaseScanner with custom logic
type MyScanner struct {
    *core.BaseScanner

    // Custom fields
    ruleset string
    configPath string
}

// NewMyScanner creates a new custom scanner
func NewMyScanner(ruleset string, verbose bool) *MyScanner {
    base := core.NewBaseScanner(&core.BaseScannerConfig{
        Name:    "my-scanner",
        Version: "1.0.0",
        Binary:  "semgrep",
        DefaultArgs: []string{
            "scan",
            "--sarif",
            "--config", ruleset,
            "{target}",
        },
        Timeout:      30 * time.Minute,
        OKExitCodes:  []int{0, 1},
        Capabilities: []string{"sast", "secret"},
        Verbose:      verbose,
    })

    return &MyScanner{
        BaseScanner: base,
        ruleset:     ruleset,
    }
}

// BuildArgs overrides argument building to add custom logic
func (s *MyScanner) BuildArgs(target string, opts *core.ScanOptions) []string {
    // Get base args
    args := s.BaseScanner.BuildArgs(target, opts)

    // Add custom config if set
    if s.configPath != "" {
        args = append(args, "--config", s.configPath)
    }

    // Add exclude patterns
    if opts != nil {
        for _, pattern := range opts.Exclude {
            args = append(args, "--exclude", pattern)
        }
    }

    return args
}

// Scan overrides to add pre/post processing
func (s *MyScanner) Scan(ctx context.Context, target string, opts *core.ScanOptions) (*core.ScanResult, error) {
    // Pre-scan hook
    fmt.Printf("[%s] Starting scan with ruleset: %s\n", s.Name(), s.ruleset)

    // Call base scan
    result, err := s.BaseScanner.Scan(ctx, target, opts)

    // Post-scan hook
    if err == nil {
        fmt.Printf("[%s] Scan completed, output size: %d bytes\n", s.Name(), len(result.RawOutput))
    }

    return result, err
}

// SetConfigPath allows runtime configuration
func (s *MyScanner) SetConfigPath(path string) {
    s.configPath = path
}
```

---

## Working with Parsers

### Using Built-in Parsers

```go
// Create parser registry (includes SARIF and JSON parsers)
parsers := core.NewParserRegistry()

// Auto-detect parser based on content
parser := parsers.FindParser(scanResult.RawOutput)
if parser == nil {
    // Default to SARIF
    parser = parsers.Get("sarif")
}

// Parse with options
report, err := parser.Parse(ctx, scanResult.RawOutput, &core.ParseOptions{
    ToolName:  "semgrep",
    ToolType:  "sast",
    AssetType: ris.AssetTypeRepository,
    AssetValue: "github.com/org/repo",
    Branch:    "main",
    CommitSHA: "abc123",
})
```

### Creating Custom Parsers

```go
package main

import (
    "context"
    "encoding/json"

    "github.com/rediverio/rediver-sdk/sdk/core"
    "github.com/rediverio/rediver-sdk/sdk/ris"
)

// MyToolParser parses output from my-tool
type MyToolParser struct {
    *core.BaseParser
}

func NewMyToolParser() *MyToolParser {
    return &MyToolParser{
        BaseParser: core.NewBaseParser("my-tool", []string{"json", "mytool"}),
    }
}

// CanParse checks if this parser can handle the data
func (p *MyToolParser) CanParse(data []byte) bool {
    // Check for specific markers
    var raw map[string]any
    if err := json.Unmarshal(data, &raw); err != nil {
        return false
    }
    _, hasMyToolMarker := raw["my_tool_version"]
    return hasMyToolMarker
}

// Parse converts to RIS format
func (p *MyToolParser) Parse(ctx context.Context, data []byte, opts *core.ParseOptions) (*ris.Report, error) {
    // Parse your tool's output format
    var toolOutput MyToolOutput
    if err := json.Unmarshal(data, &toolOutput); err != nil {
        return nil, err
    }

    // Create report using helper
    report := p.CreateReport("my-tool", toolOutput.Version)

    // Convert findings
    for _, issue := range toolOutput.Issues {
        finding := p.CreateFinding(
            issue.ID,
            issue.Title,
            mapSeverity(issue.Severity),
        )
        finding.Description = issue.Description
        finding.Location = &ris.FindingLocation{
            Path:      issue.File,
            StartLine: issue.Line,
        }
        report.Findings = append(report.Findings, finding)
    }

    return report, nil
}

// Register the parser
func init() {
    parsers := core.NewParserRegistry()
    parsers.Register(NewMyToolParser())
}
```

---

## Building Collectors

Collectors pull data from external sources (GitHub, GitLab, APIs, etc.).

### Using GitHub Collector

```go
collector := core.NewGitHubCollector(&core.GitHubCollectorConfig{
    Token:   os.Getenv("GITHUB_TOKEN"),
    Owner:   "myorg",
    Repo:    "myrepo",
    Verbose: true,
})

// Test connection
if err := collector.TestConnection(ctx); err != nil {
    log.Fatal(err)
}

// Collect security alerts
result, err := collector.Collect(ctx, nil)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Collected %d items\n", result.TotalItems)
for _, report := range result.Reports {
    fmt.Printf("  - %d findings\n", len(report.Findings))
}
```

### Using Webhook Collector

```go
collector := core.NewWebhookCollector(&core.WebhookCollectorConfig{
    ListenAddr: ":8080",
    Verbose:    true,
})

// Start webhook server
collector.Start(ctx)
defer collector.Stop(ctx)

// Wait for incoming webhooks
for {
    result, err := collector.Collect(ctx, nil)
    if err != nil {
        continue
    }

    // Process received data
    for _, report := range result.Reports {
        processReport(report)
    }
}
```

### Creating Custom Collectors

```go
type MyAPICollector struct {
    *core.BaseCollector

    projectID string
}

func NewMyAPICollector(apiKey, projectID string) *MyAPICollector {
    return &MyAPICollector{
        BaseCollector: core.NewBaseCollector(&core.BaseCollectorConfig{
            Name:       "my-api",
            SourceType: "api",
            BaseURL:    "https://api.myservice.com",
            APIKey:     apiKey,
            Timeout:    30 * time.Second,
        }),
        projectID: projectID,
    }
}

func (c *MyAPICollector) Collect(ctx context.Context, opts *core.CollectOptions) (*core.CollectResult, error) {
    // Fetch data using built-in HTTP helpers
    var data MyAPIResponse
    err := c.FetchJSON(ctx, "/v1/vulnerabilities", map[string]string{
        "project_id": c.projectID,
    }, &data)
    if err != nil {
        return nil, err
    }

    // Convert to RIS
    report := ris.NewReport()
    for _, vuln := range data.Vulnerabilities {
        report.Findings = append(report.Findings, convertVuln(vuln))
    }

    return &core.CollectResult{
        SourceName: c.Name(),
        SourceType: c.Type(),
        Reports:    []*ris.Report{report},
        TotalItems: len(data.Vulnerabilities),
    }, nil
}
```

---

## Building Agents

Agents run as daemons, continuously scanning and pushing results.

### Using BaseAgent

```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/rediverio/rediver-sdk/sdk/client"
    "github.com/rediverio/rediver-sdk/sdk/core"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle signals
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigCh
        cancel()
    }()

    // Create API client for pushing results
    pusher := client.New(&client.Config{
        BaseURL: "https://api.rediver.io",
        APIKey:  os.Getenv("REDIVER_API_KEY"),
    })

    // Create agent
    agent := core.NewBaseAgent(&core.BaseAgentConfig{
        Name:              "my-scanner-agent",
        Version:           "1.0.0",
        ScanInterval:      1 * time.Hour,
        HeartbeatInterval: 1 * time.Minute,
        Targets:           []string{"/path/to/scan"},
        Verbose:           true,
    }, pusher)

    // Add scanners
    semgrep, _ := core.NewPresetScanner("semgrep")
    agent.AddScanner(semgrep)

    gitleaks, _ := core.NewPresetScanner("gitleaks")
    agent.AddScanner(gitleaks)

    // Add collectors
    ghCollector := core.NewGitHubCollector(&core.GitHubCollectorConfig{
        Token: os.Getenv("GITHUB_TOKEN"),
        Owner: "myorg",
        Repo:  "myrepo",
    })
    agent.AddCollector(ghCollector)

    // Start agent
    if err := agent.Start(ctx); err != nil {
        panic(err)
    }

    fmt.Println("Agent started. Press Ctrl+C to stop.")

    // Wait for shutdown
    <-ctx.Done()

    // Graceful shutdown
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    agent.Stop(shutdownCtx)
}
```

### Using CLI Tool

The SDK includes a ready-to-use CLI:

```bash
# Build
go build -o rediver-agent ./cmd/rediver-agent

# Run single scan
./rediver-agent -tool semgrep -target /path/to/project -api-url https://api.rediver.io -api-key $API_KEY

# Run as daemon with config file
./rediver-agent -daemon -config config.yaml

# List available tools
./rediver-agent -list-tools
```

**Example config.yaml:**

```yaml
agent:
  name: my-scanner-agent
  scan_interval: 1h
  heartbeat_interval: 1m
  verbose: true

rediver:
  base_url: https://api.rediver.io
  api_key: ${REDIVER_API_KEY}
  timeout: 30s

scanners:
  - name: semgrep
    enabled: true
  - name: gitleaks
    enabled: true
  - name: trivy-fs
    enabled: true

collectors:
  - name: github
    token: ${GITHUB_TOKEN}
    owner: myorg
    repo: myrepo
    enabled: true

targets:
  - /path/to/project1
  - /path/to/project2
```

---

## RIS (Rediver Ingest Schema)

### Creating Reports Programmatically

```go
import "github.com/rediverio/rediver-sdk/sdk/ris"

// Create new report
report := ris.NewReport()

// Set tool info
report.Tool = &ris.Tool{
    Name:         "my-scanner",
    Version:      "1.0.0",
    Capabilities: []string{"vulnerability", "secret"},
}

// Add assets
report.Assets = append(report.Assets, ris.Asset{
    ID:          "asset-1",
    Type:        ris.AssetTypeRepository,
    Value:       "github.com/org/repo",
    Name:        "My Repository",
    Criticality: ris.CriticalityHigh,
})

// Add findings
report.Findings = append(report.Findings, ris.Finding{
    ID:          "finding-1",
    Type:        ris.FindingTypeVulnerability,
    Title:       "SQL Injection",
    Severity:    ris.SeverityCritical,
    Confidence:  95,
    RuleID:      "CWE-89",
    AssetRef:    "asset-1",
    Location: &ris.FindingLocation{
        Path:      "src/db/query.go",
        StartLine: 45,
        EndLine:   47,
        Snippet:   "query := \"SELECT * FROM users WHERE id = \" + userID",
    },
    Vulnerability: &ris.VulnerabilityDetails{
        CWEID:      "CWE-89",
        CVSSScore:  9.8,
        CVSSVector: "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
    },
    Remediation: &ris.Remediation{
        Recommendation: "Use parameterized queries",
        Steps: []string{
            "Replace string concatenation with prepared statements",
            "Use database/sql placeholder syntax",
        },
    },
})
```

### Converting SARIF to RIS

```go
import "github.com/rediverio/rediver-sdk/sdk/ris"

sarifData := []byte(`{"version": "2.1.0", ...}`)

report, err := ris.FromSARIF(sarifData, &ris.ConvertOptions{
    AssetType:  ris.AssetTypeRepository,
    AssetValue: "github.com/org/repo",
    Branch:     "main",
    CommitSHA:  "abc123",
    ToolType:   "sast", // sast, sca, secret, iac, web3
})
```

### Type Constants

```go
// Asset Types
ris.AssetTypeDomain         // "domain"
ris.AssetTypeIPAddress      // "ip_address"
ris.AssetTypeRepository     // "repository"
ris.AssetTypeSmartContract  // "smart_contract"
// ... see types.go for full list

// Finding Types
ris.FindingTypeVulnerability    // "vulnerability"
ris.FindingTypeSecret           // "secret"
ris.FindingTypeMisconfiguration // "misconfiguration"
ris.FindingTypeCompliance       // "compliance"
ris.FindingTypeWeb3             // "web3"

// Severity Levels
ris.SeverityCritical // "critical"
ris.SeverityHigh     // "high"
ris.SeverityMedium   // "medium"
ris.SeverityLow      // "low"
ris.SeverityInfo     // "info"

// Criticality Levels
ris.CriticalityCritical // "critical"
ris.CriticalityHigh     // "high"
ris.CriticalityMedium   // "medium"
ris.CriticalityLow      // "low"
ris.CriticalityInfo     // "info"
```

---

## Web3 Scanner Example

For smart contract security scanning:

```go
package main

import (
    "context"
    "fmt"

    "github.com/rediverio/rediver-sdk/sdk/core"
    "github.com/rediverio/rediver-sdk/sdk/ris"
)

func main() {
    ctx := context.Background()

    // Use Slither preset
    scanner, _ := core.NewPresetScanner("slither")

    // Scan Solidity contracts
    result, _ := scanner.Scan(ctx, "/path/to/contracts", &core.ScanOptions{
        Verbose: true,
    })

    // Parse results
    parsers := core.NewParserRegistry()
    report, _ := parsers.Get("sarif").Parse(ctx, result.RawOutput, &core.ParseOptions{
        ToolType: "web3",
        AssetType: ris.AssetTypeSmartContract,
        AssetValue: "0x1234567890abcdef1234567890abcdef12345678",
    })

    // Print findings
    for _, f := range report.Findings {
        fmt.Printf("[%s] %s: %s\n", f.Severity, f.RuleID, f.Title)
        if f.Web3 != nil {
            fmt.Printf("  SWC: %s, Chain: %s\n", f.Web3.SWCID, f.Web3.Chain)
        }
    }
}
```

---

## Pushing Results to Rediver

```go
import "github.com/rediverio/rediver-sdk/sdk/client"

// Create API client
c := client.New(&client.Config{
    BaseURL: "https://api.rediver.io",
    APIKey:  os.Getenv("REDIVER_API_KEY"),
    Timeout: 30 * time.Second,
    Verbose: true,
})

// Test connection
if err := c.TestConnection(ctx); err != nil {
    log.Fatal(err)
}

// Push findings
result, err := c.PushFindings(ctx, report)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Created: %d, Updated: %d\n", result.FindingsCreated, result.FindingsUpdated)

// Push assets
assetResult, err := c.PushAssets(ctx, report)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Assets Created: %d, Updated: %d\n", assetResult.AssetsCreated, assetResult.AssetsUpdated)
```

---

## Best Practices

### 1. Error Handling

```go
result, err := scanner.Scan(ctx, target, opts)
if err != nil {
    // Check if it's a timeout
    if errors.Is(err, context.DeadlineExceeded) {
        log.Printf("Scan timed out for %s", target)
        return
    }
    // Handle other errors
    log.Printf("Scan failed: %v", err)
    return
}

// Check exit code for tool-specific errors
if result.ExitCode != 0 && result.ExitCode != 1 {
    log.Printf("Tool returned unexpected exit code: %d", result.ExitCode)
    log.Printf("Stderr: %s", result.Stderr)
}
```

### 2. Context Cancellation

```go
// Use context for timeout and cancellation
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Minute)
defer cancel()

// Pass to all operations
result, err := scanner.Scan(ctx, target, opts)
```

### 3. Graceful Shutdown

```go
// Handle signals
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

go func() {
    <-sigCh
    cancel() // Cancel context
}()

// Wait for agent to stop
<-ctx.Done()
agent.Stop(context.WithTimeout(context.Background(), 30*time.Second))
```

### 4. Logging

```go
scanner.SetVerbose(true) // Enable verbose logging

// Or use structured logging
log.Printf("[%s] Scanning %s", scanner.Name(), target)
```

---

## Server-Controlled Agents

Agents can be controlled remotely from the Rediver platform using the CommandPoller.

### Command Poller Setup

```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/rediverio/rediver-sdk/sdk/client"
    "github.com/rediverio/rediver-sdk/sdk/core"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle signals
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigCh
        cancel()
    }()

    // Create API client (implements core.CommandClient)
    apiClient := client.New(&client.Config{
        BaseURL:  "https://api.rediver.io",
        APIKey:   os.Getenv("REDIVER_API_KEY"),
        SourceID: os.Getenv("REDIVER_SOURCE_ID"), // For tenant tracking
        Verbose:  true,
    })

    // Create command executor
    executor := core.NewDefaultCommandExecutor(apiClient)

    // Add scanners to executor
    semgrep, _ := core.NewPresetScanner("semgrep")
    executor.AddScanner(semgrep)

    gitleaks, _ := core.NewPresetScanner("gitleaks")
    executor.AddScanner(gitleaks)

    // Create command poller
    poller := core.NewCommandPoller(apiClient, executor, &core.CommandPollerConfig{
        PollInterval:  30 * time.Second,
        MaxConcurrent: 5,
        AllowedTypes:  []string{"scan", "collect", "health_check"},
        Verbose:       true,
    })

    // Start polling for commands
    if err := poller.Start(ctx); err != nil {
        if err != context.Canceled {
            panic(err)
        }
    }
}
```

### Operating Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Standalone** | Agent runs scans based on local config | Edge deployments, offline |
| **Server-Controlled** | Agent waits for server commands | Centralized management |
| **Hybrid** | Both local scans and server commands | Flexible deployments |

### Hybrid Mode Configuration

```yaml
agent:
  name: hybrid-scanner
  scan_interval: 24h  # Daily local scans
  enable_command_polling: true
  command_poll_interval: 30s

targets:
  - /opt/code/project1

# Server can also dispatch additional scans on-demand
```

---

## Using the Processor

The Processor provides a complete scan-parse-push workflow.

```go
import (
    "github.com/rediverio/rediver-sdk/sdk/client"
    "github.com/rediverio/rediver-sdk/sdk/core"
)

// Create API client
pusher := client.New(&client.Config{
    BaseURL:  "https://api.rediver.io",
    APIKey:   os.Getenv("REDIVER_API_KEY"),
    SourceID: "src_abc123",
})

// Create processor
processor := core.NewBaseProcessor(pusher)
processor.SetVerbose(true)

// Process a single scanner
scanner, _ := core.NewPresetScanner("semgrep")
result, err := processor.Process(ctx, scanner, &core.ProcessOptions{
    ScanOptions: &core.ScanOptions{
        TargetDir: "/path/to/project",
    },
    ParseOptions: &core.ParseOptions{
        ToolName: "semgrep",
        ToolType: "sast",
    },
    Push:      true,  // Push to Rediver
    SaveLocal: true,  // Also save locally
    OutputDir: "./reports",
})

// Process multiple scanners in parallel
scanners := []core.Scanner{
    semgrep,
    gitleaks,
    trivy,
}
results, err := processor.ProcessBatch(ctx, scanners, opts)
```

---

## Custom Logging

Replace the default `fmt.Printf` logging with your own logger.

### Logger Interface

```go
// Logger is the interface for logging in the SDK
type Logger interface {
    Debug(format string, args ...interface{})
    Info(format string, args ...interface{})
    Warn(format string, args ...interface{})
    Error(format string, args ...interface{})
}
```

### Using a Custom Logger

```go
import (
    "github.com/rediverio/rediver-sdk/sdk/core"
    "github.com/sirupsen/logrus"
)

// LogrusAdapter adapts logrus to the SDK Logger interface
type LogrusAdapter struct {
    logger *logrus.Logger
}

func (l *LogrusAdapter) Debug(format string, args ...interface{}) {
    l.logger.Debugf(format, args...)
}

func (l *LogrusAdapter) Info(format string, args ...interface{}) {
    l.logger.Infof(format, args...)
}

func (l *LogrusAdapter) Warn(format string, args ...interface{}) {
    l.logger.Warnf(format, args...)
}

func (l *LogrusAdapter) Error(format string, args ...interface{}) {
    l.logger.Errorf(format, args...)
}

// Set as default logger
func main() {
    logger := logrus.New()
    logger.SetLevel(logrus.DebugLevel)

    core.SetDefaultLogger(&LogrusAdapter{logger: logger})
}
```

### Built-in Loggers

```go
// Default logger - writes to stderr with timestamps
logger := core.NewDefaultLogger("my-scanner", core.LogLevelInfo)

// Printf logger - simple fmt.Printf output (like verbose mode)
logger := core.NewPrintfLogger("my-scanner")

// Nop logger - discards all output
logger := &core.NopLogger{}

// Create logger based on verbose flag
logger := core.LoggerFromVerbose("my-scanner", verbose)
```

---

## Configuration Validation

Validate configurations before use.

### Using the Validator

```go
import "github.com/rediverio/rediver-sdk/sdk/core"

// Validate scanner config
err := core.ValidateBaseScannerConfig(&core.BaseScannerConfig{
    Name:    "my-scanner",
    Binary:  "semgrep",
    Timeout: 30 * time.Minute,
})
if err != nil {
    log.Fatalf("Invalid config: %v", err)
}

// Validate agent config
err = core.ValidateBaseAgentConfig(&core.BaseAgentConfig{
    Name:              "my-agent",
    ScanInterval:      1 * time.Hour,
    HeartbeatInterval: 1 * time.Minute,
})

// Validate command poller config
err = core.ValidateCommandPollerConfig(&core.CommandPollerConfig{
    PollInterval:  30 * time.Second,
    MaxConcurrent: 5,
})
```

### Custom Validation

```go
v := core.NewValidator()

v.Required("name", cfg.Name).
  URL("base_url", cfg.BaseURL).
  APIKey("api_key", cfg.APIKey).
  SourceID("source_id", cfg.SourceID).
  MinDuration("timeout", cfg.Timeout, 5*time.Second).
  MaxDuration("timeout", cfg.Timeout, 1*time.Hour).
  DirectoryExists("target_dir", cfg.TargetDir)

if err := v.Validate(); err != nil {
    // err contains all validation errors
    log.Fatal(err)
}
```

---

## Tenant Identification

When pushing data to Rediver, tenant identification is done via API Key + Source ID.

### How It Works

1. **API Key** identifies the tenant: `rs_src_xxxxxxxx` (Source Key)
2. **Source ID** provides audit trail: `src_abc123def456`

```go
// Create client with Source ID
client := client.New(&client.Config{
    BaseURL:  "https://api.rediver.io",
    APIKey:   "rs_src_xxxxxxxxxxxxxxxxxxxxxxxx", // Identifies tenant
    SourceID: "src_abc123def456",                // Identifies this scanner/agent
    Timeout:  30 * time.Second,
})
```

### Headers Sent

```http
POST /api/v1/ingest/findings HTTP/1.1
Host: api.rediver.io
Content-Type: application/json
Authorization: Bearer rs_src_xxxxxxxxxxxxxxxxxxxxxxxx
X-Rediver-Source-ID: src_abc123def456
User-Agent: rediver-sdk/1.0
```

### Registering a Source

Sources should be registered via the Rediver API or UI before use:

```bash
curl -X POST https://api.rediver.io/api/v1/sources \
  -H "Authorization: Bearer $USER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Production Semgrep Scanner",
    "type": "scanner",
    "capabilities": ["sast", "secret"]
  }'
```

Response:
```json
{
  "id": "src_abc123def456",
  "api_key": "rs_src_xxxxxxxxxxxxxxxxxxxx"
}
```

---

## Related Documentation

- [SDK & API Integration Architecture](../architecture/sdk-api-integration.md) - How SDK integrates with API
- [Server-Agent Command Architecture](../architecture/server-agent-command.md) - Remote agent control
- [Building Ingestion Tools](./building-ingestion-tools.md) - RIS schema reference
- [API Reference](../api/reference.md) - Full API documentation
- [Authentication Guide](./authentication.md) - API key management

---

## Examples

Full working examples are available in the SDK repository:

- [`examples/custom-scanner`](https://github.com/rediverio/rediver-sdk/tree/main/examples/custom-scanner) - Custom scanner implementation
- [`cmd/rediver-agent`](https://github.com/rediverio/rediver-sdk/tree/main/cmd/rediver-agent) - CLI agent
