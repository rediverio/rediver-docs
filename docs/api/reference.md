---
layout: default
---
# API Reference

Complete API reference for Rediver CTEM Platform.

**Base URL:** `http://localhost:8080` (development) | `https://api.rediver.io` (production)

---

## Authentication

All protected endpoints require JWT Bearer token:

```
Authorization: Bearer <access_token>
```

### Token Flow

1. **Login** → Returns `refresh_token` + tenant list
2. **Token Exchange** → Exchange `refresh_token` + `tenant_id` → `access_token`
3. **Use Access Token** → Tenant-scoped API calls

---

## Public Endpoints

### Health

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| GET | `/ready` | Readiness check |
| GET | `/metrics` | Prometheus metrics |

### Documentation

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/openapi.yaml` | OpenAPI specification |
| GET | `/docs` | Scalar API documentation UI |

---

## Auth Endpoints

**Prefix:** `/api/v1/auth`

### Local Authentication

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| GET | `/info` | ❌ | Auth provider info |
| POST | `/register` | ❌ | Register new user |
| POST | `/login` | ❌ | Login → refresh_token + tenants |
| POST | `/token` | ❌ | Exchange refresh_token → access_token |
| POST | `/refresh` | ❌ | Refresh tokens with rotation |
| POST | `/logout` | ✅ | Logout (revoke session) |
| POST | `/verify-email` | ❌ | Verify email with token |
| POST | `/forgot-password` | ❌ | Request password reset |
| POST | `/reset-password` | ❌ | Reset password with token |
| POST | `/create-first-team` | ❌* | Create first team for new user |

*Uses refresh_token from cookie

### OAuth Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| GET | `/oauth/providers` | ❌ | List available OAuth providers |
| GET | `/oauth/{provider}/authorize` | ❌ | Get OAuth authorization URL |
| POST | `/oauth/{provider}/callback` | ❌ | Handle OAuth callback |

---

## User Endpoints

**Prefix:** `/api/v1/users`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/me` | Authenticated | Get current user profile |
| PUT | `/me` | Authenticated | Update profile |
| PUT | `/me/preferences` | Authenticated | Update preferences |
| GET | `/me/tenants` | Authenticated | List user's teams |
| POST | `/me/change-password` | Authenticated | Change password (local auth) |
| GET | `/me/sessions` | Authenticated | List active sessions |
| DELETE | `/me/sessions` | Authenticated | Revoke all sessions |
| DELETE | `/me/sessions/{sessionId}` | Authenticated | Revoke specific session |

---

## Tenant (Team) Endpoints

**Prefix:** `/api/v1/tenants`

### User-scoped Tenant Operations

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | Authenticated | List user's tenants |
| POST | `/` | Authenticated | Create new tenant |
| GET | `/{tenant}` | Authenticated | Get tenant by ID or slug |

### Tenant-scoped Operations

**Prefix:** `/api/v1/tenants/{tenant}`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/members` | Viewer+ | List members |
| GET | `/members/stats` | Viewer+ | Member statistics |
| GET | `/invitations` | Viewer+ | List pending invitations |
| GET | `/settings` | Viewer+ | Get tenant settings |
| PATCH | `/` | Admin+ | Update tenant |
| POST | `/members` | Admin+ | Add member directly |
| PATCH | `/members/{memberId}` | Admin+ | Update member role |
| DELETE | `/members/{memberId}` | Admin+ | Remove member |
| POST | `/invitations` | Admin+ | Create invitation |
| DELETE | `/invitations/{invitationId}` | Admin+ | Delete invitation |
| PATCH | `/settings/general` | Admin+ | Update general settings |
| PATCH | `/settings/branding` | Admin+ | Update branding |
| PATCH | `/settings/security` | Owner | Update security settings |
| PATCH | `/settings/api` | Owner | Update API settings |
| DELETE | `/` | Owner | Delete tenant |

### Invitation Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| GET | `/api/v1/invitations/{token}/preview` | ❌ | Preview invitation |
| POST | `/api/v1/invitations/{token}/decline` | ❌ | Decline invitation |
| GET | `/api/v1/invitations/{token}` | ✅ | Get invitation details |
| POST | `/api/v1/invitations/{token}/accept` | ✅ | Accept invitation |

---

## Asset Endpoints (CTEM Discovery)

**Prefix:** `/api/v1/assets`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `assets:read` | List assets |
| GET | `/stats` | `assets:read` | Asset statistics |
| GET | `/{id}` | `assets:read` | Get asset |
| GET | `/{id}/full` | `assets:read` | Get asset with repository |
| GET | `/{id}/repository` | `assets:read` | Get repository details |
| POST | `/` | `assets:write` | Create asset |
| POST | `/repository` | `assets:write` | Create repository asset |
| PUT | `/{id}` | `assets:write` | Update asset |
| PUT | `/{id}/repository` | `assets:write` | Update repository |
| POST | `/{id}/activate` | `assets:write` | Activate asset |
| POST | `/{id}/deactivate` | `assets:write` | Deactivate asset |
| POST | `/{id}/archive` | `assets:write` | Archive asset |
| POST | `/{id}/sync` | `assets:write` | Sync repository from SCM |
| POST | `/{id}/scan` | `assets:write` | Trigger security scan |
| DELETE | `/{id}` | `assets:delete` | Delete asset |

### Branch Endpoints

**Prefix:** `/api/v1/assets/{assetId}/branches`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `assets:read` | List branches |
| GET | `/default` | `assets:read` | Get default branch |
| GET | `/{branchId}` | `assets:read` | Get branch |
| POST | `/` | `assets:write` | Create branch |
| PUT | `/{branchId}` | `assets:write` | Update branch |
| PUT | `/{branchId}/default` | `assets:write` | Set as default |
| DELETE | `/{branchId}` | `assets:delete` | Delete branch |

---

## Component Endpoints

**Prefix:** `/api/v1/components`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `components:read` | List components |
| GET | `/{id}` | `components:read` | Get component |
| POST | `/` | `components:write` | Create component |
| PUT | `/{id}` | `components:write` | Update component |
| DELETE | `/{id}` | `components:delete` | Delete component |

Asset-scoped:
| GET | `/api/v1/assets/{id}/components` | `components:read` | List asset components |

---

## Vulnerability Endpoints (Global CVE)

**Prefix:** `/api/v1/vulnerabilities`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `vulnerabilities:read` | List vulnerabilities |
| GET | `/{id}` | `vulnerabilities:read` | Get vulnerability |
| GET | `/cve/{cve_id}` | `vulnerabilities:read` | Get by CVE ID |
| POST | `/` | `vulnerabilities:write` | Create vulnerability |
| PUT | `/{id}` | `vulnerabilities:write` | Update vulnerability |
| DELETE | `/{id}` | `vulnerabilities:delete` | Delete vulnerability |

---

## Finding Endpoints (CTEM Prioritization)

**Prefix:** `/api/v1/findings`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `findings:read` | List findings |
| GET | `/{id}` | `findings:read` | Get finding |
| POST | `/` | `findings:write` | Create finding |
| PATCH | `/{id}/status` | `findings:write` | Update status |
| DELETE | `/{id}` | `findings:delete` | Delete finding |

### Finding Comments

**Prefix:** `/api/v1/findings/{id}/comments`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `findings:read` | List comments |
| POST | `/` | `findings:write` | Add comment |
| PUT | `/{comment_id}` | `findings:write` | Update comment |
| DELETE | `/{comment_id}` | `findings:write` | Delete comment |

Asset-scoped:
| GET | `/api/v1/assets/{id}/findings` | `findings:read` | List asset findings |

---

## Dashboard Endpoints

**Prefix:** `/api/v1/dashboard`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/stats/global` | `dashboard:read` | Global statistics |
| GET | `/stats` | `dashboard:read` | Tenant statistics |

---

## Audit Log Endpoints

**Prefix:** `/api/v1/audit-logs`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `audit:read` | List audit logs |
| GET | `/stats` | `audit:read` | Audit statistics |
| GET | `/{id}` | `audit:read` | Get audit log |
| GET | `/resource/{type}/{id}` | `audit:read` | Resource history |
| GET | `/user/{id}` | `audit:read` | User activity |

---

## SLA Policy Endpoints

**Prefix:** `/api/v1/sla-policies`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `sla:read` | List SLA policies |
| GET | `/default` | `sla:read` | Get default policy |
| GET | `/{id}` | `sla:read` | Get SLA policy |
| POST | `/` | `sla:write` | Create policy |
| PUT | `/{id}` | `sla:write` | Update policy |
| DELETE | `/{id}` | `sla:delete` | Delete policy |

Asset-scoped:
| GET | `/api/v1/assets/{assetId}/sla-policy` | `assets:read` | Get asset SLA |

---

## SCM Connection Endpoints

**Prefix:** `/api/v1/scm-connections`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `scm-connections:read` | List connections |
| POST | `/` | `scm-connections:write` | Create connection |
| POST | `/test-credentials` | `scm-connections:write` | Test credentials (before create) |
| GET | `/{id}` | `scm-connections:read` | Get connection |
| PUT | `/{id}` | `scm-connections:write` | Update connection |
| DELETE | `/{id}` | `scm-connections:delete` | Delete connection |
| POST | `/{id}/test` | `scm-connections:write` | Test existing connection |
| GET | `/{id}/repositories` | `scm-connections:read` | List repositories from SCM |

---

## Scan Endpoints (CTEM Scanning)

**Prefix:** `/api/v1/scans`

Scan configurations bind asset groups with scanners/workflows and schedules.

### Scan CRUD

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `scans:read` | List scans |
| GET | `/stats` | `scans:read` | Scan statistics |
| GET | `/{id}` | `scans:read` | Get scan |
| POST | `/` | `scans:write` | Create scan |
| PUT | `/{id}` | `scans:write` | Update scan |
| DELETE | `/{id}` | `scans:delete` | Delete scan |

### Scan Status Operations

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| POST | `/{id}/activate` | `scans:write` | Activate scan (enable scheduling) |
| POST | `/{id}/pause` | `scans:write` | Pause scan (suspend scheduling) |
| POST | `/{id}/disable` | `scans:write` | Disable scan |

### Scan Execution

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| POST | `/{id}/trigger` | `scans:write` | Trigger scan execution manually |
| POST | `/{id}/clone` | `scans:write` | Clone scan configuration |

### Scan Runs (Sub-resource)

**Prefix:** `/api/v1/scans/{scanId}/runs`

| Method | Endpoint | Permission | Description |
|--------|----------|------------|-------------|
| GET | `/` | `scans:read` | List runs for scan |
| GET | `/latest` | `scans:read` | Get latest run |
| GET | `/{runId}` | `scans:read` | Get specific run |

---

## Error Responses

```json
{
  "code": "UNAUTHORIZED",
  "message": "Invalid or expired token",
  "status": 401
}
```

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid request body |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `CONFLICT` | Resource already exists |
| 422 | `VALIDATION_FAILED` | Validation error |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |
