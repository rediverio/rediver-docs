---
layout: default
title: Permissions Matrix
parent: Guides
nav_order: 3
---
# Permission Matrix

Exact permission matrix for the Rediver CTEM Platform.

> Mapping from `internal/domain/permission/role_mapping.go`

---

## Role Hierarchy

| Role | Priority | Description |
|------|:--------:|-------------|
| **Owner** | 4 | Full access + billing + team deletion |
| **Admin** | 3 | Full resource access + member management |
| **Member** | 2 | Read + Write (no delete, no member management) |
| **Viewer** | 1 | Read-only access |

---

## Complete Role → Permissions Mapping

### 📋 Owner Permissions (41 permissions)

```
# Resource Access
assets:read, assets:write, assets:delete
repositories:read, repositories:write, repositories:delete
components:read, components:write, components:delete
findings:read, findings:write, findings:delete
vulnerabilities:read

# Dashboard & Reports
dashboard:read
audit:read
reports:read, reports:write

# Security Operations
scans:read, scans:write, scans:delete
credentials:read, credentials:write
pentest:read, pentest:write
remediation:read, remediation:write
workflows:read, workflows:write

# Team Management (FULL)
members:read, members:invite, members:manage
team:read, team:update, team:delete

# Billing (EXCLUSIVE)
billing:read, billing:manage

# Integrations
integrations:read, integrations:manage
```

---

### 🛡️ Admin Permissions (38 permissions)

```
# Resource Access (SAME AS OWNER)
assets:read, assets:write, assets:delete
repositories:read, repositories:write, repositories:delete
components:read, components:write, components:delete
findings:read, findings:write, findings:delete
vulnerabilities:read

# Dashboard & Reports (SAME AS OWNER)
dashboard:read
audit:read
reports:read, reports:write

# Security Operations (SAME AS OWNER)
scans:read, scans:write, scans:delete
credentials:read, credentials:write
pentest:read, pentest:write
remediation:read, remediation:write
workflows:read, workflows:write

# Team Management (PARTIAL - NO team:delete)
members:read, members:invite, members:manage
team:read, team:update
⛔ team:delete

# Billing (READ ONLY)
billing:read
⛔ billing:manage

# Integrations (SAME AS OWNER)
integrations:read, integrations:manage
```

---

### 👤 Member Permissions (24 permissions)

```
# Resource Access (READ + WRITE, NO DELETE)
assets:read, assets:write
⛔ assets:delete
repositories:read, repositories:write
⛔ repositories:delete
components:read, components:write
⛔ components:delete
findings:read, findings:write
⛔ findings:delete
vulnerabilities:read

# Dashboard & Reports
dashboard:read
reports:read, reports:write

# Security Operations
scans:read, scans:write
⛔ scans:delete
credentials:read
⛔ credentials:write
pentest:read, pentest:write
remediation:read, remediation:write
workflows:read
⛔ workflows:write

# Team (READ ONLY)
members:read
⛔ members:invite, members:manage
team:read
⛔ team:update, team:delete

# Billing
⛔ billing:read, billing:manage

# Integrations (READ ONLY)
integrations:read
⛔ integrations:manage
```

---

### 👁️ Viewer Permissions (16 permissions)

```
# Resource Access (READ ONLY)
assets:read
repositories:read
components:read
findings:read
vulnerabilities:read

# Dashboard & Reports (READ ONLY)
dashboard:read
reports:read
scans:read
credentials:read
pentest:read
remediation:read
workflows:read

# Team (READ ONLY)
members:read
team:read

# Integrations (READ ONLY)
integrations:read
```

---

## Permission Matrix Table

| Permission | Owner | Admin | Member | Viewer |
|-----------|:-----:|:-----:|:------:|:------:|
| **Assets** ||||
| `assets:read` | ✅ | ✅ | ✅ | ✅ |
| `assets:write` | ✅ | ✅ | ✅ | ❌ |
| `assets:delete` | ✅ | ✅ | ❌ | ❌ |
| **Repositories** ||||
| `repositories:read` | ✅ | ✅ | ✅ | ✅ |
| `repositories:write` | ✅ | ✅ | ✅ | ❌ |
| `repositories:delete` | ✅ | ✅ | ❌ | ❌ |
| **Components** ||||
| `components:read` | ✅ | ✅ | ✅ | ✅ |
| `components:write` | ✅ | ✅ | ✅ | ❌ |
| `components:delete` | ✅ | ✅ | ❌ | ❌ |
| **Findings** ||||
| `findings:read` | ✅ | ✅ | ✅ | ✅ |
| `findings:write` | ✅ | ✅ | ✅ | ❌ |
| `findings:delete` | ✅ | ✅ | ❌ | ❌ |
| **Vulnerabilities** ||||
| `vulnerabilities:read` | ✅ | ✅ | ✅ | ✅ |
| `vulnerabilities:write` | ⚠️ | ⚠️ | ❌ | ❌ |
| `vulnerabilities:delete` | ⚠️ | ⚠️ | ❌ | ❌ |
| **Dashboard & Audit** ||||
| `dashboard:read` | ✅ | ✅ | ✅ | ✅ |
| `audit:read` | ✅ | ✅ | ❌ | ❌ |
| **Scans** ||||
| `scans:read` | ✅ | ✅ | ✅ | ✅ |
| `scans:write` | ✅ | ✅ | ✅ | ❌ |
| `scans:delete` | ✅ | ✅ | ❌ | ❌ |
| **Credentials** ||||
| `credentials:read` | ✅ | ✅ | ✅ | ✅ |
| `credentials:write` | ✅ | ✅ | ❌ | ❌ |
| **Reports** ||||
| `reports:read` | ✅ | ✅ | ✅ | ✅ |
| `reports:write` | ✅ | ✅ | ✅ | ❌ |
| **Pentest** ||||
| `pentest:read` | ✅ | ✅ | ✅ | ✅ |
| `pentest:write` | ✅ | ✅ | ✅ | ❌ |
| **Remediation** ||||
| `remediation:read` | ✅ | ✅ | ✅ | ✅ |
| `remediation:write` | ✅ | ✅ | ✅ | ❌ |
| **Workflows** ||||
| `workflows:read` | ✅ | ✅ | ✅ | ✅ |
| `workflows:write` | ✅ | ✅ | ❌ | ❌ |
| **Members** ||||
| `members:read` | ✅ | ✅ | ✅ | ✅ |
| `members:invite` | ✅ | ✅ | ❌ | ❌ |
| `members:manage` | ✅ | ✅ | ❌ | ❌ |
| **Team** ||||
| `team:read` | ✅ | ✅ | ✅ | ✅ |
| `team:update` | ✅ | ✅ | ❌ | ❌ |
| `team:delete` | ✅ | ❌ | ❌ | ❌ |
| **Billing** ||||
| `billing:read` | ✅ | ✅ | ❌ | ❌ |
| `billing:manage` | ✅ | ❌ | ❌ | ❌ |
| **Integrations** ||||
| `integrations:read` | ✅ | ✅ | ✅ | ✅ |
| `integrations:manage` | ✅ | ✅ | ❌ | ❌ |

> ⚠️ = Admin write/delete for vulnerabilities managed via routes

---

## How Permissions Work

1. **Token Generation**: When a user exchanges a token for a tenant, the role is mapped to permissions:
   ```go
   // jwt.go
   permissions := roleToPermissions(tenant.Role)
   
   // role_mapping.go  
   func roleToPermissions(role string) []string {
       r := tenant.Role(role)
       return permission.GetPermissionStringsForRole(r)
   }
   ```

2. **Permission Check**: API routes check permissions from the token:
   ```go
   // routes.go
   r.GET("/", h.List, middleware.Require(permission.AssetsRead))
   r.POST("/", h.Create, middleware.Require(permission.AssetsWrite))
   ```

3. **Middleware Flow**: 
   ```
   Request → Auth Middleware → Extract permissions from JWT → 
   Require(permission) Middleware → Check HasPermission → Allow/Deny
   ```

---

## Related Documentation

- [Authentication Guide](./authentication.md) - Token generation
- [API Reference](../api/reference.md) - Endpoint permissions
- [Multi-tenancy Guide](./multi-tenancy.md) - Role hierarchy
