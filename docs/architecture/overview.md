# Architecture Overview

This document describes the architecture of the ReDiver platform.

## System Architecture

```
                                    ┌─────────────────────────────────────────────┐
                                    │              Load Balancer / CDN            │
                                    └─────────────────────────────────────────────┘
                                                         │
                          ┌──────────────────────────────┼──────────────────────────────┐
                          │                              │                              │
                          ▼                              ▼                              ▼
               ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
               │   rediver-ui        │      │   rediver-api       │      │   rediver-keycloak  │
               │   (Next.js 16)      │      │   (Go)              │      │   (Optional)        │
               │   Port: 3000        │      │   Port: 8080        │      │   Port: 8180        │
               └─────────────────────┘      └─────────────────────┘      └─────────────────────┘
                          │                              │                              │
                          │                              │                              │
                          │         ┌────────────────────┼────────────────────┐        │
                          │         │                    │                    │        │
                          │         ▼                    ▼                    ▼        │
                          │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
                          │  │ PostgreSQL  │    │   Redis     │    │ PostgreSQL  │    │
                          │  │ Port: 5432  │    │ Port: 6379  │    │ (Keycloak)  │    │
                          │  └─────────────┘    └─────────────┘    └─────────────┘    │
                          │                                                            │
                          └────────────────────────────────────────────────────────────┘
```

## Component Overview

### Frontend (rediver-ui)

**Technology Stack:**
- Next.js 16 (App Router, React Server Components)
- React 19
- TypeScript (strict mode)
- Tailwind CSS 4
- shadcn/ui components
- Zustand (global state)

**Architecture Pattern:** Feature-based architecture

```
src/
├── app/                    # Next.js App Router
│   ├── (auth)/            # Authentication routes
│   ├── (dashboard)/       # Protected dashboard routes
│   ├── api/               # API routes (BFF pattern)
│   └── layout.tsx         # Root layout
│
├── features/              # Feature modules
│   └── [feature]/
│       ├── components/    # Feature-specific components
│       ├── actions/       # Server Actions
│       ├── schemas/       # Zod validation schemas
│       ├── types/         # TypeScript types
│       └── hooks/         # Feature hooks
│
├── components/            # Shared components
│   ├── ui/               # shadcn/ui primitives
│   └── layout/           # Layout components
│
├── stores/               # Zustand stores
├── context/              # React Context providers
├── lib/                  # Utilities
└── hooks/                # Global hooks
```

### Backend (rediver-api)

**Technology Stack:**
- Go 1.25
- Chi router (standard net/http)
- PostgreSQL 17
- Redis 7
- Structured logging (slog)

**Architecture Pattern:** Clean Architecture / Hexagonal Architecture

```
cmd/
└── server/               # Application entry point

internal/
├── domain/               # Core Business Logic (innermost layer)
│   ├── asset/           # Asset domain
│   │   ├── entity.go    # Asset entity
│   │   ├── repository.go # Repository interface
│   │   └── service.go   # Domain service interface
│   └── shared/          # Shared domain types
│       ├── id.go        # ID value object
│       └── errors.go    # Domain errors
│
├── app/                  # Application Layer (use cases)
│   └── asset/
│       └── service.go   # Application service (orchestrates domain)
│
└── infra/               # Infrastructure Layer (outermost layer)
    ├── http/            # HTTP adapter
    │   ├── server.go    # HTTP server
    │   ├── router.go    # Route definitions
    │   ├── handler/     # HTTP handlers
    │   └── middleware/  # HTTP middleware
    ├── postgres/        # PostgreSQL adapter
    │   └── asset_repo.go
    └── redis/           # Redis adapter
        └── cache.go

pkg/                      # Public packages
├── logger/              # Structured logging
├── pagination/          # Pagination helpers
└── apierror/            # API error types
```

### Dependency Flow

```
HTTP Request
     │
     ▼
┌─────────────────┐
│   HTTP Handler  │  ◄── Infrastructure Layer
│   (infra/http)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   App Service   │  ◄── Application Layer
│   (app/)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Domain Service  │  ◄── Domain Layer
│ (domain/)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Repository     │  ◄── Infrastructure Layer
│  (infra/*)      │      (implements domain interface)
└─────────────────┘
```

**Key Principles:**
1. **Domain layer has no external dependencies**
2. **Dependencies point inward** (outer layers depend on inner)
3. **Interfaces defined in domain**, implemented in infra
4. **Use cases orchestrate** domain logic

## Authentication Flow

### Local Authentication (2-Step Token Flow)

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Client  │     │   Next.js    │     │   Go API     │
└────┬─────┘     └──────┬───────┘     └──────┬───────┘
     │                  │                    │
     │ 1. Login Form    │                    │
     │─────────────────>│                    │
     │                  │ 2. POST /auth/login│
     │                  │───────────────────>│
     │                  │                    │ Verify creds
     │                  │ 3. refresh_token   │ Create session
     │                  │    + tenants[]     │
     │                  │<───────────────────│
     │                  │                    │
     │                  │ 4. POST /auth/token│
     │                  │    {tenant_id}     │
     │                  │───────────────────>│
     │                  │                    │ Validate tenant
     │                  │ 5. access_token    │ membership
     │                  │    (tenant-scoped) │
     │                  │<───────────────────│
     │                  │                    │
     │ 6. Set Cookies   │                    │
     │    - refresh (httpOnly)               │
     │    - access (httpOnly)                │
     │<─────────────────│                    │
     │                  │                    │
     │ 7. API Request   │                    │
     │    Authorization: Bearer <access>     │
     │─────────────────>│───────────────────>│
     │                  │                    │ Verify JWT
     │                  │                    │ Extract tenant_id
     │ 8. Response      │                    │
     │<─────────────────│<───────────────────│
```

**Token Types:**
- `refresh_token`: Global, 7 days, httpOnly cookie
- `access_token`: Tenant-scoped, 15 min, httpOnly cookie

### OIDC Authentication (Keycloak)

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Client  │     │   Next.js    │     │   Keycloak   │     │   Go API     │
└────┬─────┘     └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
     │                  │                    │                    │
     │ 1. Login Click   │                    │                    │
     │─────────────────>│                    │                    │
     │                  │ 2. Redirect to KC  │                    │
     │<─────────────────│───────────────────>│                    │
     │ 3. KC Login Form │                    │                    │
     │─────────────────────────────────────>│                    │
     │ 4. Redirect + Code                    │                    │
     │<──────────────────────────────────────│                    │
     │─────────────────>│                    │                    │
     │                  │ 5. Exchange Code   │                    │
     │                  │───────────────────>│                    │
     │                  │ 6. ID + Access     │                    │
     │                  │<───────────────────│                    │
     │ 7. Set Cookies   │                    │                    │
     │<─────────────────│                    │                    │
     │                  │                    │                    │
     │ 8. API Request   │                    │                    │
     │─────────────────>│───────────────────────────────────────>│
     │                  │                    │ 9. Verify via JWKS │
     │<─────────────────│<───────────────────────────────────────│
```

**Details:** See [Authentication Guide](./authentication-guide.md)

## Data Flow

### Request Lifecycle (Backend)

```
1. HTTP Request
       │
       ▼
2. Middleware Chain
   ├── Request ID
   ├── Logger
   ├── Recovery (panic handling)
   ├── CORS
   ├── Rate Limiter
   └── Auth (JWT validation)
       │
       ▼
3. Router (chi)
       │
       ▼
4. Handler
   ├── Parse request
   ├── Validate input
   └── Call service
       │
       ▼
5. Application Service
   ├── Business logic
   └── Call repository
       │
       ▼
6. Repository
   ├── Database query
   └── Return data
       │
       ▼
7. Response
   └── JSON response
```

### State Management (Frontend)

```
┌─────────────────────────────────────────────────────────────┐
│                        Client State                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  Zustand Store  │    │  React Context  │                │
│  │  (Global State) │    │  (UI State)     │                │
│  │                 │    │                 │                │
│  │  - Auth state   │    │  - Theme        │                │
│  │  - User data    │    │  - Direction    │                │
│  │                 │    │  - Layout       │                │
│  └─────────────────┘    └─────────────────┘                │
│           │                      │                          │
│           └──────────┬───────────┘                          │
│                      ▼                                      │
│  ┌─────────────────────────────────────────┐               │
│  │           React Components               │               │
│  │  (Server Components + Client Components) │               │
│  └─────────────────────────────────────────┘               │
│                      │                                      │
└──────────────────────┼──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                       Server State                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  Server Actions │    │  SWR Cache      │                │
│  │  (Mutations)    │    │  (Data Fetching)│                │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

## Database Schema

### Core Tables

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'user',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Assets table
CREATE TABLE assets (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    criticality VARCHAR(20) DEFAULT 'medium',
    status VARCHAR(20) DEFAULT 'active',
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Exposures table
CREATE TABLE exposures (
    id UUID PRIMARY KEY,
    asset_id UUID REFERENCES assets(id),
    title VARCHAR(500) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'open',
    source VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## Caching Strategy

### Redis Usage

```
┌─────────────────────────────────────────────┐
│                Redis Cache                   │
├─────────────────────────────────────────────┤
│                                              │
│  Session Storage                             │
│  ├── session:{user_id} → session data       │
│  └── TTL: AUTH_SESSION_DURATION             │
│                                              │
│  Rate Limiting                               │
│  ├── ratelimit:{ip} → request count         │
│  └── TTL: 1 minute (sliding window)         │
│                                              │
│  API Cache                                   │
│  ├── cache:assets:list → cached response    │
│  └── TTL: 5 minutes                         │
│                                              │
│  JWKS Cache (Keycloak)                       │
│  ├── jwks:{realm} → public keys             │
│  └── TTL: KEYCLOAK_JWKS_REFRESH_INTERVAL    │
│                                              │
└─────────────────────────────────────────────┘
```

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Network                                             │
│ ├── TLS/HTTPS                                               │
│ ├── Firewall rules                                          │
│ └── Rate limiting at load balancer                          │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Application Gateway                                 │
│ ├── CORS policy                                             │
│ ├── Request validation                                      │
│ └── Rate limiting                                           │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Authentication                                      │
│ ├── JWT validation                                          │
│ ├── Session management                                      │
│ └── CSRF protection                                         │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: Authorization                                       │
│ ├── Role-based access control (RBAC)                        │
│ ├── Resource-level permissions                              │
│ └── Audit logging                                           │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: Data                                               │
│ ├── Input validation (Zod schemas)                          │
│ ├── SQL injection prevention (parameterized queries)        │
│ ├── Encryption at rest                                      │
│ └── Encryption in transit                                   │
└─────────────────────────────────────────────────────────────┘
```

## Deployment Architecture

### Production Setup

```
                    ┌─────────────────────────────────────┐
                    │           CDN (CloudFlare)           │
                    │         Static Assets Cache          │
                    └─────────────────┬───────────────────┘
                                      │
                    ┌─────────────────┴───────────────────┐
                    │        Load Balancer (nginx)         │
                    │         SSL Termination              │
                    └─────────────────┬───────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Frontend Pod   │      │  Backend Pod    │      │  Keycloak Pod   │
│  (Next.js)      │      │  (Go)           │      │                 │
│  Replicas: 2-4  │      │  Replicas: 2-4  │      │  Replicas: 2    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   PostgreSQL    │      │     Redis       │      │   PostgreSQL    │
│   Primary +     │      │   Cluster       │      │   (Keycloak)    │
│   Replicas      │      │   Sentinel      │      │                 │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

## Monitoring & Observability

```
┌─────────────────────────────────────────────────────────────┐
│                    Observability Stack                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Metrics (Prometheus)                                        │
│  ├── HTTP request duration                                  │
│  ├── Request count by status                                │
│  ├── Database connection pool                               │
│  └── Custom business metrics                                │
│                                                              │
│  Logging (Structured JSON)                                   │
│  ├── Request/Response logs                                  │
│  ├── Error logs with stack traces                           │
│  ├── Audit logs                                             │
│  └── Aggregated in ELK/Loki                                 │
│                                                              │
│  Tracing (OpenTelemetry)                                    │
│  ├── Distributed request tracing                            │
│  ├── Database query tracing                                 │
│  └── External API call tracing                              │
│                                                              │
│  Error Tracking (Sentry)                                    │
│  ├── Frontend errors                                        │
│  ├── Backend errors                                         │
│  └── Source map support                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Related Documentation

- [Backend Architecture](../rediver-api/docs/architecture/)
- [Frontend Architecture](../rediver-ui/.claude/architecture.md)
- [API Documentation](../rediver-api/docs/api/)
