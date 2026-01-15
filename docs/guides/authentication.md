# Authentication Guide

Hướng dẫn chi tiết về authentication trong Rediver CTEM Platform.

---

## Authentication Modes

| Mode | Description | Sử dụng |
|------|-------------|---------|
| `local` | Email/Password với JWT | Development, simple deployments |
| `oidc` | Keycloak/OIDC only | Enterprise SSO |
| `hybrid` | Cả local và OIDC | Flexible authentication |

Cấu hình qua `AUTH_PROVIDER` environment variable.

---

## Token Types

### Refresh Token (Global)
- **Thời hạn:** 7 ngày
- **Scope:** User-level (không có tenant)
- **Storage:** httpOnly cookie
- **Dùng để:** Exchange cho access_token

### Access Token (Tenant-scoped)
- **Thời hạn:** 15 phút
- **Scope:** Tenant-specific
- **Storage:** httpOnly cookie
- **Dùng để:** API requests

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

Sau khi login, exchange refresh_token để lấy tenant-scoped access_token:

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

Access token hết hạn → refresh với rotation:

```
POST /api/v1/auth/refresh
{
  "refresh_token": "eyJ...",
  "tenant_id": "tenant-uuid"
}
```

Response bao gồm cả access_token mới và refresh_token mới (rotation).

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
| `rediver_tenant` | ❌ | Production | Lax | 7 days |
| `csrf_token` | ❌ | Production | Lax | Session |

### CSRF Protection

Double-submit cookie pattern:
1. Server generates `csrf_token` và set trong cookie
2. Client đọc cookie và gửi trong header `X-CSRF-Token`
3. Server verify header matches cookie

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
  "iss": "rediver-api"
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
