---
layout: default
title: Features
nav_order: 5
has_children: true
permalink: /features/
---

# Features

Documentation for major features in the Rediver CTEM Platform.

---

## Core Features

| Feature | Description | Status |
|---------|-------------|--------|
| [Platform Agents](platform-agents.md) | Shared scanning infrastructure managed by Rediver | ✅ Implemented |
| [Scan Profiles](scan-profiles.md) | Reusable scan configs with Quality Gates | ✅ Implemented |
| [Scanner Templates](scanner-templates.md) | Custom detection rules for Nuclei, Semgrep, Gitleaks | ✅ Implemented |
| [Quality Gates](quality-gates.md) | CI/CD pass/fail decisions based on finding thresholds | ✅ Implemented |
| [Workflow Automation](workflows.md) | Event-driven automation for notifications, tickets, assignments | ✅ Implemented |
| [CTEM Finding Fields](ctem-fields.md) | Exposure, remediation, and business impact fields | ✅ Implemented |
| [Capabilities Registry](capabilities-registry.md) | Normalized tool capability management | ✅ Implemented |
| [Asset Sub-Modules](asset-sub-modules.md) | Hierarchical module system for asset types | ✅ Implemented |

---

## Feature Categories

### Infrastructure

| Feature | Description |
|---------|-------------|
| **Platform Agents** | Shared, multi-tenant scanning agents |
| **Storage Service** | Multi-tenant storage with BYOB support |

### Scanning

| Feature | Description |
|---------|-------------|
| **Scan Profiles** | Reusable configurations with Quality Gates and Template Modes |
| **Scan Pipelines** | Multi-step automated scanning workflows |
| **Scanner Templates** | Custom templates for Nuclei, Semgrep, Gitleaks |
| **Capabilities Registry** | Normalized tool capability management |
| **Quality Gates** | CI/CD pass/fail decisions based on finding thresholds |

### CTEM Framework

| Feature | Description |
|---------|-------------|
| **CTEM Finding Fields** | Exposure vector, remediation context, business impact fields |
| **Risk Calculation** | CTEM-aware risk scoring with multipliers |
| **Prioritization** | High-priority detection based on exposure and compliance |

### Automation

| Feature | Description |
|---------|-------------|
| **Workflow Automation** | Event-driven automation with triggers, conditions, actions |
| **Notifications** | Multi-channel alerts (Slack, Email, Teams, PagerDuty) |
| **Integrations** | Ticket creation (Jira), HTTP webhooks, custom scripts |

### Asset Management

| Feature | Description |
|---------|-------------|
| **Asset Sub-Modules** | Dynamic control over asset type visibility |
| **Asset Groups** | Organize assets into logical groups |

### Access Control

| Feature | Description |
|---------|-------------|
| **Module Access Control** | Subscription-based feature gating |
| **Group-Based Access** | Fine-grained resource permissions |

---

## Reference

| Document | Description |
|----------|-------------|
| [Component Interactions](component-interactions.md) | Complete overview of how components interact |

---

## Related Documentation

- [Architecture Overview](../architecture/overview.md)
- [Scan Pipeline Design](../architecture/scan-pipeline-design.md)
- [Platform Agents Implementation](../architecture/platform-agents-implementation-plan.md)
