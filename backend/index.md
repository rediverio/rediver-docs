---
layout: default
title: Backend Services
has_children: true
nav_order: 3
---

# Backend API Service

[![Go Version](https://img.shields.io/badge/Go-1.25+-00ADD8?logo=go)](https://go.dev)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Hub-2496ED?logo=docker)](https://hub.docker.com/r/rediverio/api)

Unified Exposure Management platform built with Clean Architecture in Go.

## 📚 Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](https://docs.rediver.io/docs/getting-started) | Quick start guide |
| [API Reference](https://docs.rediver.io/docs/api/reference) | Complete API endpoints |
| [Authentication](https://docs.rediver.io/docs/guides/authentication) | JWT & OIDC auth flow |
| [Permissions](https://docs.rediver.io/docs/guides/permissions) | Role-based access control |
| [Architecture](https://docs.rediver.io/docs/architecture/overview) | System design |
| [Configuration](https://docs.rediver.io/docs/operations/configuration) | Environment variables |

## Features

- **Asset Management** - Track and categorize assets with criticality levels
- **Exposure Detection** - Identify vulnerabilities and risks
- **Attack Path Analysis** - Visualize attack vectors
- **Risk Scoring** - Calculate and prioritize risks
- **Multi-source Integration** - Wiz, Tenable, Snyk, CrowdStrike

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.25+ |
| HTTP | Standard `net/http` |
| Authentication | Local JWT / Keycloak OIDC |
| Database | PostgreSQL 17 |
| Cache | Redis 7 |
| Logging | Structured logging (slog) |

## Project Structure

```
rediver/
├── cmd/server/              # Application entry point
├── internal/
│   ├── domain/              # Core business logic (entities, value objects)
│   │   ├── asset/           # Asset domain
│   │   └── shared/          # Shared domain types (ID, errors)
│   ├── app/                 # Application services (use cases)
│   └── infra/               # Infrastructure adapters
│       ├── http/            # HTTP server, router
│       │   ├── routes/      # Route registration (by domain)
│       │   │   ├── routes.go      # Main entry point
│       │   │   ├── admin.go       # Platform admin routes
│       │   │   ├── auth.go        # Authentication
│       │   │   ├── tenant.go      # Tenant management
│       │   │   ├── assets.go      # Assets, components
│       │   │   ├── scanning.go    # Scans, agents, tools
│       │   │   ├── exposure.go    # Findings, credentials
│       │   │   ├── access_control.go  # Roles, permissions
│       │   │   ├── platform.go    # Platform agents/jobs
│       │   │   └── misc.go        # Health, docs, etc.
│       │   ├── handler/     # Request handlers (40+)
│       │   └── middleware/  # HTTP middleware
│       └── postgres/        # PostgreSQL repository
├── pkg/                     # Public utilities
│   ├── logger/              # Structured logging
│   ├── pagination/          # Pagination helpers
│   └── apierror/            # API error types
├── migrations/              # Database migrations
├── api/openapi/             # OpenAPI specification
├── tests/integration/       # Integration tests
└── docs/                    # Documentation
```

## Quick Start

### Prerequisites

- Go 1.25+
- Docker & Docker Compose
- Make (optional)

### Development

```bash
# Clone
git clone https://github.com/rediverio/api.git
cd api

# Setup environment
cp .env.example .env

# Start with hot reload
make docker-dev

# Or run locally
make dev
```

### Production

```bash
# Set required environment variables
export DB_PASSWORD=your_secure_password
export REDIS_PASSWORD=your_secure_password
export AUTH_JWT_SECRET=your_64_char_secret
export CORS_ALLOWED_ORIGINS=https://your-domain.com

# Start production environment
make docker-prod
```

### Verify

```bash
curl http://localhost:8080/health
# {"status":"healthy","timestamp":"2025-01-01T00:00:00Z"}
```

## Docker

### Docker Compose Files

| File | Purpose | Usage |
|------|---------|-------|
| `docker-compose.yml` | Base configuration | Shared services (postgres, redis) |
| `docker-compose.dev.yml` | Development | Hot reload, debug ports |
| `docker-compose.prod.yml` | Production | Security hardening, no exposed DB |

### Development

```bash
# Start with hot reload
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# With build
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

Features:
- Hot reload with Air
- Delve debugger on port 2345
- DB/Redis exposed for local tools
- Debug logging enabled

### Production

```bash
# Start production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Security features:
- DB/Redis NOT exposed externally
- `no-new-privileges` on all containers
- `read_only` filesystem for API
- Resource limits enforced
- JSON logging with rotation

### Environment Variables

**Required for Production:**

| Variable | Description |
|----------|-------------|
| `DB_PASSWORD` | Database password |
| `REDIS_PASSWORD` | Redis password |
| `AUTH_JWT_SECRET` | JWT signing secret (min 64 chars) |
| `CORS_ALLOWED_ORIGINS` | Allowed CORS origins |

## Make Commands

```bash
make help           # Show all commands

# Development
make dev            # Run with hot reload (Air)
make run            # Run without hot reload
make test           # Run tests
make lint           # Run linter
make build          # Build binary

# Docker
make docker-dev     # Start dev environment (hot reload)
make docker-prod    # Start production environment
make docker-down    # Stop all containers
make docker-logs    # View logs

# Database
make migrate-up     # Run migrations
make migrate-down   # Rollback migration

# Security & Pre-commit
make pre-commit-install  # Install pre-commit hooks
make pre-commit-run      # Run all security checks
make security-scan       # Full security scan (gitleaks + gosec + trivy)
make gitleaks            # Run secret detection only
```

## API Documentation

| Endpoint | Description |
|----------|-------------|
| `/docs` | Scalar API documentation UI |
| `/openapi.yaml` | OpenAPI 3.0 specification |

Access documentation at: `http://localhost:8080/docs`

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| GET | `/ready` | Readiness check |
| GET | `/docs` | API documentation |
| GET | `/openapi.yaml` | OpenAPI spec |
| GET | `/api/v1/assets` | List assets |
| POST | `/api/v1/assets` | Create asset |
| GET | `/api/v1/assets/{id}` | Get asset |
| PUT | `/api/v1/assets/{id}` | Update asset |
| DELETE | `/api/v1/assets/{id}` | Delete asset |

## Documentation

- [Getting Started](docs/getting-started.md)
- [Architecture](docs/architecture/overview.md)
- [API Reference](docs/api/)
- [Development](docs/development/)
- [Deployment](docs/deployment/)

## License

MIT
