---
layout: default
title: Getting Started
nav_order: 1
---
# Getting Started

This guide will help you get ReDiver up and running in under 10 minutes.

## Prerequisites

Before you begin, ensure you have the following installed:

| Tool | Version | Check Command | Install |
|------|---------|---------------|---------|
| Node.js | 20+ | `node -v` | [nodejs.org](https://nodejs.org) |
| Go | 1.25+ | `go version` | [go.dev](https://go.dev) |
| Docker | 24+ | `docker -v` | [docker.com](https://docker.com) |
| Docker Compose | 2.0+ | `docker compose version` | Included with Docker |

## Option 1: Docker (Recommended)

The fastest way to get started is using Docker Compose.

### Step 1: Clone the Repository

```bash
git clone https://github.com/rediverio/rediver.git
cd rediver
```

### Step 2: Configure Environment

```bash
# Backend
cd api
cp .env.example .env

# Frontend
cd ../ui
cp .env.example .env.local

# Return to root
cd ..
```

### Step 3: Start Services

```bash
# Start all services (PostgreSQL, Redis, Backend, Frontend)
docker compose up -d

# Watch logs
docker compose logs -f
```

### Step 4: Verify

```bash
# Check all services are running
docker compose ps

# Test backend health
curl http://localhost:8080/health

# Open frontend
open http://localhost:3000
```

Expected health response:
```json
{"status":"healthy","timestamp":"2025-01-12T00:00:00Z"}
```

### Step 5: Database Migration & Seed

```bash
cd api

# Run database migrations
make docker-migrate-up

# Seed required data (permissions, roles)
make docker-seed-required

# Seed test data (optional - for development)
make docker-seed-test

# Or seed all data at once
make docker-seed-all
```

**Seed options:**
| Command | Description |
|---------|-------------|
| `make docker-seed-required` | Required data only (permissions) |
| `make docker-seed-test` | Test users and teams |
| `make docker-seed-all` | All seed data including sample assets |

## Option 2: Manual Setup

For development with hot-reload capabilities.

### Step 1: Start Infrastructure

```bash
# Start only PostgreSQL and Redis
docker compose up -d postgres redis

# Verify
docker compose ps
```

### Step 2: Setup Backend

```bash
cd api

# Copy environment file
cp .env.example .env

# Install Go tools (first time only)
make install-tools

# Run database migrations
make migrate-up

# Seed required data
make seed-required

# Seed test data (optional)
make seed-test

# Start with hot reload
make dev
```

Backend should now be running at http://localhost:8080

### Step 3: Setup Frontend

Open a new terminal:

```bash
cd ui

# Copy environment file
cp .env.example .env.local

# Install dependencies
npm install

# Start development server
npm run dev
```

Frontend should now be running at http://localhost:3000

## First Login

### Local Authentication Mode (Default)

1. Open http://localhost:3000
2. Click **"Sign Up"** to create a new account
3. Enter email, password, and name
4. After login:
   - **No teams?** → Create your first team
   - **One team?** → Auto-redirected to dashboard
   - **Multiple teams?** → Select which team to access

### Default Test Account (Development)

If seeded data is available:
- **Email:** `admin@rediver.io`
- **Password:** `Admin@123`

### Creating First Team

New users without teams will be prompted to create one:
- Team Name: Display name for the team
- Team Slug: URL-friendly identifier (e.g., `my-company`)

## Verify Setup

Run these commands to verify everything is working:

```bash
# Backend health
curl http://localhost:8080/health
# Expected: {"status":"healthy",...}

# Backend readiness
curl http://localhost:8080/ready
# Expected: {"status":"ready",...}

# List assets (requires auth token)
curl http://localhost:8080/api/v1/assets \
  -H "Authorization: Bearer <your-token>"
```

## Project URLs

| Service | Development URL |
|---------|-----------------|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8080 |
| API Docs | http://localhost:8080/docs |
| Keycloak (if enabled) | http://localhost:8180 |

## Common Issues

### Port Already in Use

```bash
# Find process using port 8080
lsof -i :8080

# Kill process
kill -9 <PID>

# Or change port in .env
SERVER_PORT=8081
```

### Database Connection Failed

```bash
# Check PostgreSQL is running
docker compose ps postgres

# Check logs
docker compose logs postgres

# Restart PostgreSQL
docker compose restart postgres
```

### Frontend Can't Connect to Backend

1. Ensure backend is running: `curl http://localhost:8080/health`
2. Check CORS settings in backend `.env`:
   ```
   CORS_ALLOWED_ORIGINS=http://localhost:3000
   ```
3. Verify frontend `.env.local`:
   ```
   NEXT_PUBLIC_API_URL=http://localhost:8080
   ```

## Next Steps

- [Development Setup](./development-setup.md) - IDE setup, debugging, testing
- [Environment Configuration](./operations/configuration.md) - All environment variables
- [Architecture](./architecture/overview.md) - System design and patterns
- [API Documentation](./api/reference.md) - API reference

## Getting Help

- Check [Troubleshooting Guide](./operations/troubleshooting.md)
- Search [GitHub Issues](https://github.com/rediverio/api/issues)
- Create a new issue with:
  - OS and versions
  - Steps to reproduce
  - Expected vs actual behavior
  - Relevant logs
