---
layout: default
title: Platform Agents
parent: Features
nav_order: 1
---

# Platform Agents

> **Status**: ✅ Implemented
> **Version**: v3.2
> **Released**: 2026-01-24

## Overview

Platform Agents are Rediver-managed, shared scanning agents that can be used by any tenant. This provides a scalable, shared scanning infrastructure for tenants who don't want to manage their own agents.

## Problem Statement

Previously, each tenant needed to deploy and manage their own agents. This created challenges:
1. **Operational overhead** for tenants to maintain agent infrastructure
2. **Underutilization** of scanning capacity across tenants
3. **No SaaS-native option** for customers who want a fully managed experience

## Solution: Hybrid Agent Model

```
┌─────────────────────────────────────────────────────────────────┐
│                     PLATFORM AGENTS                              │
│                 (Managed by Rediver)                            │
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│   │Agent-US1│  │Agent-US2│  │Agent-EU1│  │Agent-AP1│          │
│   │us-east-1│  │us-west-2│  │eu-west-1│  │ap-south │          │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘          │
│        └────────────┴────────────┴────────────┘                 │
│                          │                                       │
│              ┌───────────┴───────────┐                          │
│              │    JOB QUEUE          │                          │
│              │  (Fair Weighted)      │                          │
│              └───────────────────────┘                          │
│                          │                                       │
│     ┌────────────────────┼────────────────────┐                 │
│     ▼                    ▼                    ▼                 │
│ ┌────────┐          ┌────────┐          ┌────────┐             │
│ │Tenant A│          │Tenant B│          │Tenant C│             │
│ │ Jobs   │          │ Jobs   │          │ Jobs   │             │
│ └────────┘          └────────┘          └────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. System Tenant

All platform agents are owned by a special "System Tenant":
- **ID**: `00000000-0000-0000-0000-000000000001`
- Regular tenants cannot use this ID
- Platform agents have `is_platform_agent = true`

### 2. Job Queue with Fair Scheduling

Jobs from all tenants are queued and scheduled fairly:

```
Queue Priority = Base Priority + Age Bonus

Where:
- Base Priority: critical=1000, high=750, normal=500, low=250
- Age Bonus: minutes_in_queue × 10 (capped at 500)
```

This ensures:
- Older jobs get priority over newer ones
- Higher priority jobs still get processed first
- No tenant can starve others

### 3. Bootstrap Tokens

Platform agents self-register using kubeadm-style tokens:

```
Format: xxxxxx.yyyyyyyyyyyyyyyy
Example: abc123.0123456789abcdef

Features:
- Configurable expiration (1-168 hours)
- Usage limits (1-100 uses per token)
- Capability/tool/region constraints
- Audit trail of registrations
```

### 4. Per-Plan Limits

Each subscription plan has different limits:

| Plan | Max Concurrent Jobs | Max Queue Size |
|------|---------------------|----------------|
| Free | 2 | 10 |
| Starter | 5 | 50 |
| Pro | 10 | 100 |
| Enterprise | 50 | 500 |

## API Endpoints

### Admin Endpoints

```http
# Platform Agents
GET    /api/v1/admin/platform-agents           # List agents
GET    /api/v1/admin/platform-agents/stats     # Statistics
GET    /api/v1/admin/platform-agents/{id}      # Get agent
POST   /api/v1/admin/platform-agents           # Create agent
POST   /api/v1/admin/platform-agents/{id}/disable
POST   /api/v1/admin/platform-agents/{id}/enable
DELETE /api/v1/admin/platform-agents/{id}

# Bootstrap Tokens
GET    /api/v1/admin/bootstrap-tokens          # List tokens
POST   /api/v1/admin/bootstrap-tokens          # Create token
POST   /api/v1/admin/bootstrap-tokens/{id}/revoke

# Queue Stats
GET    /api/v1/admin/platform-jobs/stats
```

### Tenant Endpoints

```http
POST   /api/v1/platform-jobs                   # Submit job
GET    /api/v1/platform-jobs                   # List jobs
GET    /api/v1/platform-jobs/{id}              # Get status
POST   /api/v1/platform-jobs/{id}/cancel       # Cancel job
```

### Agent Endpoints (Public)

```http
POST   /api/v1/platform-agents/register        # Self-registration
```

### Platform Agent Endpoints (API Key Auth)

```http
POST   /api/v1/platform-agent/heartbeat        # Heartbeat
POST   /api/v1/platform-agent/jobs/claim       # Claim job
POST   /api/v1/platform-agent/jobs/{id}/status # Update status
```

## Usage Examples

### Creating a Bootstrap Token (Admin)

```bash
curl -X POST /api/v1/admin/bootstrap-tokens \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{
    "description": "Production agents - US region",
    "expires_in_hours": 24,
    "max_uses": 10,
    "required_region": "us-east-1",
    "required_capabilities": ["nuclei", "nmap"]
  }'

# Response:
{
  "token": {
    "id": "...",
    "token_prefix": "abc123",
    "status": "active",
    ...
  },
  "raw_token": "abc123.0123456789abcdef"  // Save this! Only shown once
}
```

### Registering an Agent

```bash
curl -X POST /api/v1/platform-agents/register \
  -d '{
    "bootstrap_token": "abc123.0123456789abcdef",
    "name": "scanner-us-east-1-01",
    "capabilities": ["nuclei", "nmap", "subfinder"],
    "tools": ["nuclei", "nmap", "subfinder"],
    "region": "us-east-1",
    "max_concurrent": 10,
    "version": "1.0.0"
  }'

# Response:
{
  "agent": { ... },
  "api_key": "rdv_agent_..."  // Save this! Only shown once
}
```

### Submitting a Job (Tenant)

```bash
curl -X POST /api/v1/platform-jobs \
  -H "Authorization: Bearer $TENANT_TOKEN" \
  -d '{
    "type": "scan",
    "priority": "normal",
    "payload": {
      "target": "example.com",
      "tool": "nuclei",
      "templates": ["cves", "exposures"]
    },
    "capabilities": ["nuclei"],
    "preferred_region": "us-east-1"
  }'

# Response:
{
  "job": { ... },
  "auth_token": "...",  // For agent to authenticate status updates
  "status": "queued",   // or "assigned" if agent available
  "queue": {
    "position": 5
  }
}
```

### Agent Claiming Jobs

```bash
# Agent polls for work
curl -X POST /api/v1/platform-agent/jobs/claim \
  -H "X-API-Key: rdv_agent_..." \
  -d '{
    "capabilities": ["nuclei", "nmap"],
    "tools": ["nuclei", "nmap"]
  }'

# Response:
{
  "job": {
    "id": "...",
    "type": "scan",
    "payload": { ... }
  },
  "no_job_available": false
}
```

> **Note**: Platform agents are authenticated via their API key (`X-API-Key` header), so no additional auth token is needed for status updates.

## Background Workers

Two workers maintain queue health:

### PlatformQueueWorker
- **Recovery** (1min): Recovers stuck jobs
- **Expiry** (5min): Expires old queued jobs
- **Priority Update** (30s): Recalculates queue priorities

### PlatformAgentHealthChecker
- **Health Check** (30s): Marks stale agents offline
- **Stats Logging** (5min): Logs aggregate statistics

## Database Schema

### Platform Agent Fields (agents table)

```sql
ALTER TABLE agents ADD COLUMN is_platform_agent BOOLEAN DEFAULT FALSE;
ALTER TABLE agents ADD COLUMN region VARCHAR(50);
```

### Bootstrap Tokens

```sql
CREATE TABLE bootstrap_tokens (
    id UUID PRIMARY KEY,
    token_hash VARCHAR(64) UNIQUE NOT NULL,
    token_prefix VARCHAR(10) NOT NULL,
    description TEXT,
    status VARCHAR(20) DEFAULT 'active',
    max_uses INT DEFAULT 1,
    current_uses INT DEFAULT 0,
    required_capabilities TEXT[],
    required_tools TEXT[],
    required_region VARCHAR(50),
    expires_at TIMESTAMPTZ NOT NULL,
    created_by UUID REFERENCES users(id),
    revoked_by UUID REFERENCES users(id),
    revoked_at TIMESTAMPTZ,
    revoke_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Platform Job Fields (commands table)

```sql
ALTER TABLE commands ADD COLUMN is_platform_job BOOLEAN DEFAULT FALSE;
ALTER TABLE commands ADD COLUMN platform_agent_id UUID REFERENCES agents(id);
ALTER TABLE commands ADD COLUMN queued_at TIMESTAMPTZ;
ALTER TABLE commands ADD COLUMN queue_priority INT DEFAULT 0;
ALTER TABLE commands ADD COLUMN auth_token_hash VARCHAR(64);
ALTER TABLE commands ADD COLUMN auth_token_expires_at TIMESTAMPTZ;
ALTER TABLE commands ADD COLUMN retry_count INT DEFAULT 0;
```

## Security Considerations

1. **Bootstrap Token Security**
   - Tokens are hashed before storage
   - Only shown once at creation
   - Can be revoked anytime
   - Have configurable constraints

2. **Job Authentication**
   - Each job has a unique auth token
   - Agents must provide token to update status
   - Prevents unauthorized status updates

3. **Tenant Isolation**
   - Jobs are always scoped to tenant
   - Agents cannot access other tenants' data
   - Queue limits per tenant

## Migration Notes

For existing deployments:
1. Run migrations 000080 and 000081
2. Create bootstrap tokens for agent registration
3. Deploy platform agents with bootstrap tokens
4. Configure tenant plan limits

## Related Documentation

- [Architecture: Platform Agents](../architecture/platform-agents-implementation-plan.md)
- [Guide: Agent Configuration](../guides/agent-configuration.md)
- [Guide: Running Agents](../guides/running-agents.md)
