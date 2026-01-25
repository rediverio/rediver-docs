# Implementation Plan: Platform Agents Architecture v3.2

## Overview

Implementation of the Platform Agents feature for Rediver SaaS platform. Platform agents are Rediver-managed, shared agents that can be used by any tenant (with access control), providing a shared scanning infrastructure for tenants who don't want to manage their own agents.

## Problem

Rediver needs to support:
1. **Platform Agents**: Shared agents managed by Rediver that can be used by multiple tenants
2. **Job Queue Management**: Fair queuing with weighted priorities to ensure tenants get fair access
3. **Bootstrap Tokens**: kubeadm-style tokens for agent self-registration
4. **Load Balancing**: Distribute jobs across agents based on load and capabilities

## Key Design Decisions

- **SystemTenantID**: `00000000-0000-0000-0000-000000000001` - Special tenant that "owns" all platform agents
- **FOR UPDATE SKIP LOCKED**: Used for atomic job claiming to prevent race conditions
- **Auth Tokens per Job**: Defense-in-depth authentication for job status updates
- **Redis for Ephemeral State**: Heartbeats, online tracking, queue stats in Redis
- **PostgreSQL for Durable State**: Job data, agent configuration, bootstrap tokens in Postgres

## Implementation Tasks

- [x] **Phase 1: Database Schema** - Completed 2026-01-24
  - Migration 000080: Added platform agent fields to agents table
  - Migration 000081: Created bootstrap_tokens and agent_registrations tables

- [x] **Phase 2: Domain Layer** - Completed 2026-01-24
  - `internal/domain/agent/entity.go` - PlatformAgentStats, selection types
  - `internal/domain/agent/bootstrap_token.go` - Bootstrap token entity
  - `internal/domain/agent/errors.go` - Platform agent errors
  - `internal/domain/agent/repository.go` - Platform agent repository interface
  - `internal/domain/command/entity.go` - Platform job fields

- [x] **Phase 3: Infrastructure Layer** - Completed 2026-01-24
  - `internal/infra/postgres/agent_repository.go` - Platform agent DB methods
  - `internal/infra/postgres/command_repository.go` - Queue operations
  - `internal/infra/postgres/bootstrap_token_repository.go` - Bootstrap token repo
  - `internal/infra/redis/agent_state.go` - Redis agent state store

- [x] **Phase 4: Application Services** - Completed 2026-01-24
  - `internal/app/platform_agent_service.go` - Platform agent management
  - `internal/app/platform_job_service.go` - Job queue management

- [x] **Phase 5: HTTP Handlers** - Completed 2026-01-24
  - `internal/infra/http/handler/platform_agent_handler.go` - Admin endpoints for platform agents
  - `internal/infra/http/handler/platform_job_handler.go` - Tenant job submission endpoints

- [x] **Phase 6: Background Workers** - Completed 2026-01-24
  - `internal/infra/jobs/platform_queue_worker.go` - Queue maintenance worker
  - `internal/infra/jobs/platform_agent_health_checker.go` - Platform agent health monitoring

- [x] **Phase 7: Route Registration** - Completed 2026-01-25
  - Refactored routes into `routes/` subfolder for better organization
  - Created `routes/admin.go` - Platform admin routes (isolated from tenant routes)
  - Created `routes/platform.go` - Tenant & agent-facing platform routes
  - Added authentication middleware (RequirePlatformAdmin, API key auth)
  - All routes registered in `routes/routes.go`

- [ ] **Phase 8: Integration Testing** - Pending
  - Unit tests for services
  - Integration tests for handlers
  - E2E tests for job flow

## Files Created/Modified

### Created
- `api/migrations/000080_add_platform_agents.up.sql`
- `api/migrations/000080_add_platform_agents.down.sql`
- `api/migrations/000081_add_bootstrap_tokens.up.sql`
- `api/migrations/000081_add_bootstrap_tokens.down.sql`
- `api/internal/domain/agent/bootstrap_token.go`
- `api/internal/infra/postgres/bootstrap_token_repository.go`
- `api/internal/infra/redis/agent_state.go`
- `api/internal/app/platform_agent_service.go`
- `api/internal/app/platform_job_service.go`
- `api/internal/infra/http/handler/platform_agent_handler.go`
- `api/internal/infra/http/handler/platform_job_handler.go`
- `api/internal/infra/jobs/platform_queue_worker.go`
- `api/internal/infra/jobs/platform_agent_health_checker.go`
- `api/internal/infra/http/routes/` - New routes subfolder with:
  - `routes.go` - Main entry point, Handlers struct, Register()
  - `admin.go` - Platform admin routes (isolated from tenant routes)
  - `auth.go` - Authentication routes
  - `tenant.go` - Tenant management routes
  - `assets.go` - Asset, Component, AssetGroup routes
  - `scanning.go` - Agent, Command, Scan, Pipeline, Tool routes
  - `exposure.go` - Exposure, ThreatIntel, Credential routes
  - `access_control.go` - Group, Role, Permission routes
  - `platform.go` - Platform agent/job routes (tenant & agent facing)
  - `misc.go` - Health, Docs, Dashboard, Audit, SLA routes

### Modified
- `api/internal/domain/agent/entity.go` - Added PlatformAgentStats, selection types
- `api/internal/domain/agent/errors.go` - Added platform agent errors
- `api/internal/domain/agent/repository.go` - Added platform agent interfaces
- `api/internal/domain/command/entity.go` - Added platform job fields
- `api/internal/domain/command/repository.go` - Added queue methods interface
- `api/internal/infra/postgres/agent_repository.go` - Added platform agent methods
- `api/internal/infra/postgres/command_repository.go` - Added queue operations

## API Endpoints

### Admin Endpoints (requires admin role)
- `GET    /api/v1/admin/platform-agents` - List platform agents
- `GET    /api/v1/admin/platform-agents/stats` - Get agent statistics
- `GET    /api/v1/admin/platform-agents/{id}` - Get platform agent
- `POST   /api/v1/admin/platform-agents` - Create platform agent
- `POST   /api/v1/admin/platform-agents/{id}/disable` - Disable agent
- `POST   /api/v1/admin/platform-agents/{id}/enable` - Enable agent
- `DELETE /api/v1/admin/platform-agents/{id}` - Delete agent
- `GET    /api/v1/admin/bootstrap-tokens` - List bootstrap tokens
- `POST   /api/v1/admin/bootstrap-tokens` - Create bootstrap token
- `POST   /api/v1/admin/bootstrap-tokens/{id}/revoke` - Revoke token
- `GET    /api/v1/admin/platform-jobs/stats` - Queue statistics

### Public Endpoints (no auth required)
- `POST   /api/v1/platform-agents/register` - Agent self-registration

### Tenant Endpoints (requires tenant auth)
- `POST   /api/v1/platform-jobs` - Submit job
- `GET    /api/v1/platform-jobs` - List jobs
- `GET    /api/v1/platform-jobs/{id}` - Get job status
- `POST   /api/v1/platform-jobs/{id}/cancel` - Cancel job

### Platform Agent Endpoints (requires agent API key + platform agent)
- `POST   /api/v1/platform-agent/heartbeat` - Record heartbeat
- `POST   /api/v1/platform-agent/jobs/claim` - Claim next job
- `POST   /api/v1/platform-agent/jobs/{id}/status` - Update job status

## Verification

- [x] All files compile without errors (`go build ./...`) - Completed 2026-01-25
- [ ] Unit tests pass for new services
- [ ] Integration tests pass for handlers
- [ ] Database migrations apply cleanly
- [ ] Platform agent registration works with bootstrap token
- [ ] Job submission and claiming flow works
- [ ] Queue priority aging works correctly
- [ ] Stuck job recovery works
- [ ] Agent health monitoring works

## Notes

- The implementation uses the existing agent infrastructure where possible
- Platform agents are distinguished by `is_platform_agent = true` flag
- The `SystemTenantID` is a well-known UUID that cannot be used by regular tenants
- Bootstrap tokens use kubeadm-style format: `xxxxxx.yyyyyyyyyyyyyyyy`
