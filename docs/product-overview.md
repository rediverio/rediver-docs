---
layout: default
title: Product Overview
nav_order: 1
---

# Rediver CTEM Platform - Product Overview

## What is Rediver?

Rediver is an enterprise-grade **Continuous Threat Exposure Management (CTEM)** platform designed to help security teams:

- **Discover** assets across your attack surface
- **Identify** vulnerabilities and security exposures
- **Prioritize** risks based on business impact
- **Validate** with automated scanning
- **Remediate** with tracked workflows

---

## The CTEM Framework

Gartner's CTEM framework provides a systematic approach to managing exposure. Rediver implements all 5 stages:

### Stage 1: Scoping

Define your attack surface boundaries and business context.

**Rediver Features:**
- Asset Groups with business unit assignment
- Crown Jewels identification
- Scope configuration (targets, exclusions)
- Risk thresholds and SLA policies

### Stage 2: Discovery

Automated and continuous discovery of assets and exposures.

**Rediver Features:**
- 6 Asset Types: Domains, Websites, Services, Repositories, Cloud, Credentials
- Workers for distributed scanning
- SCM Connections (GitHub, GitLab, Bitbucket)
- Agent-based continuous discovery

### Stage 3: Prioritization

Rank exposures by risk and business impact.

**Rediver Features:**
- Risk scoring (0-100 scale)
- CVSS-based severity (Critical/High/Medium/Low/Info)
- Asset criticality weighting
- Business context enrichment
- Findings deduplication with fingerprinting

### Stage 4: Validation

Verify vulnerabilities and test security controls.

**Rediver Features:**
- Multi-tool scanning (SAST, SCA, Secrets, IaC, Container)
- Scan Profiles for consistent testing
- Pipeline workflows for multi-step validation
- Scan Sessions for execution tracking

### Stage 5: Mobilization

Remediate and track progress.

**Rediver Features:**
- Remediation task management
- Assignee tracking
- SLA monitoring
- Audit logging
- Status progression (Open → In Progress → Resolved)

---

## Core Modules

### Asset Management

| Asset Type | Description | Examples |
|------------|-------------|----------|
| **Domain** | DNS domains | example.com |
| **Website** | Web applications | https://app.example.com |
| **Service** | Network services | SSH on port 22 |
| **Repository** | Code repositories | github.com/org/repo |
| **Cloud** | Cloud resources | AWS EC2, S3 buckets |
| **Credential** | Leaked credentials | Email/password pairs |

**Features:**
- Asset groups for organization
- Criticality levels (Critical/High/Medium/Low)
- Owner and business unit assignment
- Tag-based filtering
- Risk score visualization

### Vulnerability & Findings Management

**Finding Types:**
- Vulnerability (CVE-based)
- Secret (hardcoded credentials)
- Misconfiguration (IaC issues)
- Compliance (policy violations)
- Web3 (smart contract vulnerabilities)

**Features:**
- CVSS scoring
- CWE classification
- Status tracking (Open/In Progress/Resolved/Suppressed)
- Comments and collaboration
- SLA breach alerts

### Scan Management

**Components:**
- **Tools**: Security scanners (Semgrep, Trivy, Gitleaks, etc.)
- **Tool Categories**: SAST, SCA, DAST, Secrets, IaC, Container, Recon, OSINT
- **Scan Profiles**: Reusable scan configurations
- **Scans**: Scheduled or on-demand scan jobs
- **Pipelines**: Multi-step workflow automation
- **Scan Sessions**: Agent execution tracking

### Workers & Integration

**Worker Types:**
- **Scanner**: Runs security tools
- **Agent**: Full daemon with scanning + commands
- **Collector**: Pulls data from external sources

**Integration Options:**
- REST API with JWT authentication
- Go SDK for custom tools
- Agent for CI/CD pipelines
- Webhook support

### Multi-Tenancy

**Team/Tenant Features:**
- Isolated data per tenant
- Role-based access control
- Tenant switching
- Invitation system

**Roles:**
| Role | Capabilities |
|------|--------------|
| Owner | Full access + billing + delete team |
| Admin | Full resource access + member management |
| Member | Read + Write (no delete) |
| Viewer | Read-only access |

---

## Technology Stack

| Layer | Technologies |
|-------|--------------|
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind CSS 4, shadcn/ui |
| **Backend** | Go 1.25, Chi Router, Clean Architecture |
| **Database** | PostgreSQL 17 with migrations |
| **Cache** | Redis 7 |
| **Auth** | JWT (local) or Keycloak (OIDC) |
| **SDK** | Go SDK with Scanner/Parser/Collector interfaces |
| **Agent** | Go-based security scanning agent |
| **Deployment** | Docker, Docker Compose, Kubernetes-ready |

---

## Deployment Options

### Option 1: Docker Compose (Recommended)

```bash
cd setup
make staging-up
```

### Option 2: Kubernetes

Helm charts available for production deployments.

### Option 3: Manual

Run API and UI separately with external database.

---

## Use Cases

### Enterprise Security Teams

- Continuous attack surface monitoring
- Vulnerability prioritization at scale
- Compliance tracking and reporting
- Multi-team collaboration

### DevSecOps

- CI/CD security scanning
- Shift-left vulnerability detection
- Developer feedback via PR comments
- SARIF integration with GitHub/GitLab

### Managed Security Service Providers (MSSPs)

- Multi-tenant client management
- White-label capabilities
- Centralized dashboard
- Custom tool integration

### Security Researchers

- Custom scanner development with SDK
- RIS format for data normalization
- Web3/smart contract analysis
- Credential leak monitoring

---

## Getting Started

1. **[Quick Start Guide](getting-started.md)** - Get running in 10 minutes
2. **[Development Setup](development-setup.md)** - Setup local development
3. **[Running Workers](guides/running-workers.md)** - Deploy scanning agents
4. **[SDK Development](guides/sdk-development.md)** - Build custom tools

---

## Related Documentation

- [Architecture Overview](architecture/overview.md)
- [API Reference](api/reference.md)
- [Authentication Guide](guides/authentication.md)
- [Permissions Matrix](guides/permissions.md)
