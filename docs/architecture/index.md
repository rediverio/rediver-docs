---
layout: default
title: Architecture
nav_order: 4
has_children: true
---

# Architecture

Technical architecture and design documentation for the Rediver CTEM Platform.

---

## System Design

| Document | Description |
|----------|-------------|
| [Overview](overview.md) | High-level system architecture |
| [Deployment Modes](deployment-modes.md) | Standalone vs distributed deployment |
| [Server-Agent Communication](server-agent-command.md) | Command & control protocol |
| [Scan Pipeline Design](scan-pipeline-design.md) | Workflow execution engine |

---

## Integration

| Document | Description |
|----------|-------------|
| [SDK-API Integration](sdk-api-integration.md) | SDK and API design |
| [Agent Key Management](agent-key-management.md) | Authentication and key handling |
| [gRPC Design](grpc-design.md) | gRPC protocol design (future) |

---

## Key Concepts

### Clean Architecture

The backend follows Clean Architecture with three layers:

```
┌─────────────────────────────────────────┐
│              Infrastructure              │
│  (HTTP handlers, DB, external services) │
├─────────────────────────────────────────┤
│              Application                 │
│         (Use cases, services)           │
├─────────────────────────────────────────┤
│                Domain                    │
│      (Entities, business rules)         │
└─────────────────────────────────────────┘
```

### Multi-Tenancy

All data is tenant-scoped with complete isolation:

- Tenant ID embedded in JWT tokens
- Row-level security via `tenant_id` column
- Middleware validates tenant membership

### SDK Integration

Agents and scanners integrate via:

1. **REST API** - Push findings and assets
2. **Go SDK** - Build custom tools
3. **RIS Schema** - Standardized data format
