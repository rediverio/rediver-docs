# Multi-Tenancy Guide

Hướng dẫn về kiến trúc multi-tenant trong Rediver CTEM Platform.

---

## Overview

Rediver sử dụng **multi-tenant architecture** với:
- **Tenant** (API) = **Team** (UI)
- Mỗi user có thể thuộc nhiều tenants
- Mỗi tenant có data isolation riêng
- Access token được scoped theo tenant

---

## Tenant Model

```
┌───────────────────────────────────────────────────────────┐
│                        User                                │
│  - id, email, name                                        │
│  - Can belong to multiple tenants                         │
└───────────────────────────┬───────────────────────────────┘
                            │
                            │ TenantMember (junction table)
                            │ - user_id, tenant_id, role
                            │
┌───────────────────────────┴───────────────────────────────┐
│                       Tenant                               │
│  - id, name, slug                                         │
│  - All data (assets, findings, etc) belongs to tenant     │
└───────────────────────────────────────────────────────────┘
```

---

## Tenant Roles

| Role | Priority | Capabilities |
|------|:--------:|--------------|
| **Owner** | 4 | Full control, billing, delete tenant |
| **Admin** | 3 | Manage members, all resources |
| **Member** | 2 | Create/edit resources |
| **Viewer** | 1 | Read-only access |

Chi tiết: [Permissions Matrix](./permissions-matrix.md)

---

## Tenant Selection Flow

### Case 1: No Tenants (New User)

```
Login → No tenants → Redirect to Create Team → Team created → Dashboard
```

### Case 2: Single Tenant

```
Login → 1 tenant → Auto-select → Exchange token → Dashboard
```

### Case 3: Multiple Tenants

```
Login → Multiple tenants → Show selector → User picks → Exchange token → Dashboard
```

---

## Create First Team

User đăng ký mới không có team. Flow:

```
POST /api/v1/auth/create-first-team
Cookie: refresh_token=<token>
{
  "team_name": "My Company",
  "team_slug": "my-company"
}
```

Response:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "tenant_id": "uuid",
  "tenant_slug": "my-company",
  "tenant_name": "My Company",
  "role": "owner"
}
```

---

## Tenant Switching

Để switch sang tenant khác:

1. Gọi `/api/v1/auth/token` với tenant_id mới
2. Cập nhật cookies
3. Reload hoặc navigate

```typescript
// Frontend
async function switchTenant(tenantId: string) {
  const result = await selectTenantAction(tenantId)
  if (result.success) {
    router.refresh() // Reload với tenant mới
  }
}
```

---

## Token-Based Tenant Scoping

Access token chứa `tenant_id`:

```json
{
  "sub": "user-id",
  "tid": "tenant-id",      // <- Tenant ID in token
  "tslug": "team-slug",
  "trole": "owner",
  ...
}
```

**Security benefits:**
- Không thể IDOR (access other tenant's data)
- Backend extract tenant từ token, không từ URL
- Token exchange requires valid membership

---

## Data Isolation

Mọi tenant-scoped resource đều có `tenant_id`:

```sql
-- Assets
SELECT * FROM assets WHERE tenant_id = $1;

-- Findings
SELECT * FROM findings WHERE tenant_id = $1;

-- Audit logs
SELECT * FROM audit_logs WHERE tenant_id = $1;
```

Middleware `RequireTenant()` validates token có `tenant_id`.

---

## Invitation System

### Invite Flow

1. Admin/Owner creates invitation
2. Email sent with invitation link
3. Invitee clicks link
4. Preview shown (public endpoint)
5. Login/Register required
6. Accept invitation
7. User added to tenant

### Invitation Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| POST | `/tenants/{t}/invitations` | Admin+ | Create invitation |
| GET | `/invitations/{token}/preview` | ❌ | Preview (public) |
| POST | `/invitations/{token}/decline` | ❌ | Decline (public) |
| GET | `/invitations/{token}` | ✅ | Get details |
| POST | `/invitations/{token}/accept` | ✅ | Accept |

---

## Tenant Settings

### General Settings
- Name, slug, description
- Admin+ can update

### Branding Settings
- Logo, colors
- Admin+ can update

### Security Settings
- MFA, IP whitelist
- **Owner only**

### API Settings
- API keys, webhooks
- **Owner only**

---

## Best Practices

### 1. Token Exchange Pattern
Always exchange tokens when switching context:
```
refresh_token + tenant_id → access_token
```

### 2. URL Structure
Use tenant slug in URLs for SEO and UX:
```
https://app.rediver.io/{tenant-slug}/dashboard
https://app.rediver.io/{tenant-slug}/assets
```

### 3. Default Tenant
Store last selected tenant for faster access.

### 4. Membership Check
Always validate membership before operations:
```go
middleware.RequireMembership(tenantRepo)
```

---

## Related Documentation

- [Authentication Guide](./authentication.md)
- [Permissions Matrix](./permissions.md)
- [API Reference](../api/reference.md)
