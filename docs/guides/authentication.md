---
layout: default
title: Authentication
parent: Guides
nav_order: 1
---
# Authentication Guide

Detailed guide for authentication in the Rediver CTEM Platform.

---

## Authentication Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `local` | Email/Password with JWT | Development, simple deployments |
| `oidc` | Keycloak/OIDC only | Enterprise SSO |
| `hybrid` | Both local and OIDC | Flexible authentication |

Configure via the `AUTH_PROVIDER` environment variable.

---

## Token Types

### Refresh Token (Global)
- **Lifetime:** 7 days
- **Scope:** User-level (no tenant)
- **Storage:** httpOnly cookie
- **Purpose:** Exchange for access_token

### Access Token (Tenant-scoped)
- **Lifetime:** 15 minutes
- **Scope:** Tenant-specific
- **Storage:** httpOnly cookie
- **Purpose:** API requests

---

## Local Authentication Flow

### 1. Registration

```
POST /api/v1/auth/register
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "name": "Full Name"
}
```

Response:
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "Full Name",
  "requires_verification": true,
  "message": "Please check your email to verify your account."
}
```

### 2. Login

```
POST /api/v1/auth/login
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

Response:
```json
{
  "refresh_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "Full Name"
  },
  "tenants": [
    {
      "id": "tenant-uuid",
      "slug": "my-team",
      "name": "My Team",
      "role": "owner"
    }
  ]
}
```

### 3. Token Exchange

After login, exchange the refresh_token to get a tenant-scoped access_token:

```
POST /api/v1/auth/token
{
  "refresh_token": "eyJ...",
  "tenant_id": "tenant-uuid"
}
```

Response:
```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 900,
  "tenant_id": "tenant-uuid",
  "tenant_slug": "my-team",
  "role": "owner"
}
```

### 4. Token Refresh

When access token expires, refresh with rotation:

```
POST /api/v1/auth/refresh
{
  "refresh_token": "eyJ...",
  "tenant_id": "tenant-uuid"
}
```

Response includes both a new access_token and a new refresh_token (rotation).

### 5. Logout

```
POST /api/v1/auth/logout
Authorization: Bearer <access_token>
```

---

## Multi-Tenant Login Flow Diagram

```
┌─────────┐      ┌─────────────┐      ┌──────────────┐
│ Browser │      │   Next.js   │      │   Go API     │
└────┬────┘      └──────┬──────┘      └──────┬───────┘
     │                  │                    │
     │  1. Login Form   │                    │
     │─────────────────>│                    │
     │                  │                    │
     │                  │  2. POST /login    │
     │                  │───────────────────>│
     │                  │                    │
     │                  │  3. refresh_token  │
     │                  │     + tenants[]    │
     │                  │<───────────────────│
     │                  │                    │
     │                  │ 4. Store refresh   │
     │                  │    in httpOnly     │
     │                  │    cookie          │
     │                  │                    │
     │  ┌───────────────┴───────────────┐    │
     │  │ tenants.length == 0           │    │
     │  │ → Redirect to Create Team     │    │
     │  ├───────────────────────────────┤    │
     │  │ tenants.length == 1           │    │
     │  │ → Auto-exchange token         │────>│ 5. POST /token
     │  │                               │    │    {tenant_id}
     │  │                               │<───│
     │  │                               │    │ 6. access_token
     │  ├───────────────────────────────┤    │
     │  │ tenants.length > 1            │    │
     │  │ → Show tenant selection       │    │
     │  └───────────────────────────────┘    │
     │                  │                    │
     │  7. Set cookies  │                    │
     │<─────────────────│                    │
     │                  │                    │
     │  8. Redirect to  │                    │
     │     /dashboard   │                    │
```

---

## Cookie Security

| Cookie | HttpOnly | Secure | SameSite | Max Age |
|--------|:--------:|:------:|:--------:|---------|
| `refresh_token` | ✅ | Production | Lax | 7 days |
| `access_token` | ✅ | Production | Lax | 15 min |
| `app_tenant` | ❌ | Production | Lax | 7 days |
| `csrf_token` | ❌ | Production | Lax | Session |

### CSRF Protection

Double-submit cookie pattern:
1. Server generates `csrf_token` and sets it in a cookie
2. Client reads the cookie and sends it in the `X-CSRF-Token` header
3. Server verifies the header matches the cookie

---

## JWT Claims

### Access Token Claims

```json
{
  "sub": "user-id",
  "sid": "session-id",
  "tid": "tenant-id",
  "tslug": "tenant-slug",
  "trole": "owner",
  "troles": ["owner"],
  "email": "user@example.com",
  "name": "Full Name",
  "permissions": ["assets:read", "assets:write"],
  "exp": 1234567890,
  "iat": 1234567890,
  "iss": "api"
}
```

---

## Session Management

### List Active Sessions

```
GET /api/v1/users/me/sessions
Authorization: Bearer <access_token>
```

### Revoke Specific Session

```
DELETE /api/v1/users/me/sessions/{sessionId}
Authorization: Bearer <access_token>
```

### Revoke All Sessions

```
DELETE /api/v1/users/me/sessions
Authorization: Bearer <access_token>
```

---

## Password Management

### Forgot Password

```
POST /api/v1/auth/forgot-password
{
  "email": "user@example.com"
}
```

### Reset Password

```
POST /api/v1/auth/reset-password
{
  "token": "reset-token-from-email",
  "new_password": "NewSecurePass123!"
}
```

### Change Password (Authenticated)

```
POST /api/v1/users/me/change-password
Authorization: Bearer <access_token>
{
  "current_password": "OldPass123!",
  "new_password": "NewSecurePass123!"
}
```

---

## Security Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_JWT_SECRET` | - | JWT signing secret (min 64 chars) |
| `AUTH_ACCESS_TOKEN_DURATION` | 15m | Access token lifetime |
| `AUTH_REFRESH_TOKEN_DURATION` | 168h | Refresh token lifetime (7 days) |
| `AUTH_MAX_LOGIN_ATTEMPTS` | 5 | Max failed attempts before lockout |
| `AUTH_LOCKOUT_DURATION` | 15m | Lockout duration |
| `AUTH_MAX_ACTIVE_SESSIONS` | 10 | Max concurrent sessions per user |

---

## Related Documentation

- [Multi-tenancy Guide](./multi-tenancy.md)
- [Permissions Matrix](./permissions.md)
- [API Reference](../api/reference.md)
