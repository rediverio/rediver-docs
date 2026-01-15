# Development Setup

Complete guide for setting up your development environment.

## IDE Setup

### VS Code (Recommended)

#### Required Extensions

**Backend (Go):**
```json
{
  "recommendations": [
    "golang.go",
    "ms-vscode.makefile-tools"
  ]
}
```

**Frontend (TypeScript/React):**
```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "formulahendry.auto-rename-tag"
  ]
}
```

#### Workspace Settings

Create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  },
  "[go]": {
    "editor.defaultFormatter": "golang.go",
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "go.lintTool": "golangci-lint",
  "go.lintFlags": ["--fast"],
  "typescript.preferences.importModuleSpecifier": "non-relative",
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"],
    ["cn\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"]
  ]
}
```

### JetBrains (GoLand/WebStorm)

1. **Go Settings:**
   - Enable `goimports` on save
   - Set `golangci-lint` as external linter

2. **TypeScript Settings:**
   - Enable ESLint integration
   - Enable Prettier on save

## Backend Development

### Install Go Tools

```bash
cd rediver-api

# Install all development tools
make install-tools

# This installs:
# - golangci-lint (linter)
# - air (hot reload)
# - migrate (database migrations)
# - mockgen (mock generation)
```

### Running the Backend

```bash
# With hot reload (recommended)
make dev

# Without hot reload
make run

# With Docker
make docker-dev
```

### Database Migrations

```bash
# Create new migration
make migrate-create name=add_users_table

# Run migrations
make migrate-up

# Rollback last migration
make migrate-down

# Check migration status
migrate -path migrations -database "$DATABASE_URL" version
```

### Testing

```bash
# Run all tests
make test

# Run with coverage
make test-coverage

# Run specific test
go test -v ./internal/domain/asset/...

# Run integration tests
go test -v ./tests/integration/...
```

### Debugging

#### VS Code Launch Configuration

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Backend",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/rediver-api/cmd/server",
      "env": {
        "APP_ENV": "development"
      },
      "args": []
    }
  ]
}
```

#### Using Delve

```bash
# Install delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug
dlv debug ./cmd/server/main.go

# Attach to running process
dlv attach <pid>
```

## Frontend Development

### Install Dependencies

```bash
cd rediver-ui

# Install packages
npm install

# Update packages
npm update
```

### Running the Frontend

```bash
# Development (with Turbopack)
npm run dev

# Production build
npm run build

# Start production server
npm run start
```

### Testing

```bash
# Run tests
npm run test

# Watch mode
npm run test:watch

# With UI
npm run test:ui

# Coverage report
npm run test:coverage
```

### Linting

```bash
# Run ESLint
npm run lint

# Fix auto-fixable issues
npm run lint -- --fix

# Type check
npx tsc --noEmit
```

### Debugging

#### VS Code Launch Configuration

Add to `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Frontend",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000",
      "webRoot": "${workspaceFolder}/rediver-ui"
    },
    {
      "name": "Debug Server Components",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,
      "localRoot": "${workspaceFolder}/rediver-ui"
    }
  ]
}
```

#### Debug Server Components

```bash
# Start with Node.js inspector
NODE_OPTIONS='--inspect' npm run dev
```

## Code Quality

### Pre-commit Hooks

Install Husky for Git hooks:

```bash
# Frontend
cd rediver-ui
npm install -D husky lint-staged
npx husky init

# Add pre-commit hook
echo "npx lint-staged" > .husky/pre-commit
```

Create `.lintstagedrc.json`:

```json
{
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.{json,md}": ["prettier --write"]
}
```

### Backend Linting

```bash
cd rediver-api

# Run linter
make lint

# Auto-fix formatting
make fmt
```

## Working with Docker

### Development Containers

```bash
# Start dev environment
make docker-dev

# View logs
make docker-logs

# Access container shell
docker compose exec app sh

# Access database
docker compose exec postgres psql -U rediver -d rediver
```

### Rebuild After Changes

```bash
# Rebuild specific service
docker compose up -d --build app

# Rebuild all
docker compose up -d --build
```

## Environment Variables

### Generate Secrets

```bash
# Generate JWT secret (backend)
openssl rand -base64 48

# Generate CSRF secret (frontend)
openssl rand -base64 32

# Or use npm script
cd rediver-ui
npm run generate-secret
```

### Validate Environment

```bash
# Backend - check required vars
cd rediver-api
go run cmd/server/main.go --validate-env

# Frontend - Next.js will error on missing NEXT_PUBLIC_* vars
npm run build
```

## API Development

### OpenAPI Spec

Location: `rediver-api/api/openapi/`

```bash
# Generate types from OpenAPI
cd rediver-api
make generate
```

### Testing APIs

Using curl:
```bash
# Health check
curl http://localhost:8080/health

# Create asset (with auth)
curl -X POST http://localhost:8080/api/v1/assets \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"name":"Test Asset","type":"server"}'
```

Using HTTPie:
```bash
http GET localhost:8080/health
http POST localhost:8080/api/v1/assets \
  Authorization:"Bearer <token>" \
  name="Test Asset" type="server"
```

## Performance Profiling

### Backend (Go)

```bash
# CPU profiling
go test -cpuprofile=cpu.prof -bench=.

# Memory profiling
go test -memprofile=mem.prof -bench=.

# View profile
go tool pprof cpu.prof
```

### Frontend (Next.js)

```bash
# Build with bundle analyzer
npm run analyze

# Web Vitals (built-in)
# Check console in development
```

## Troubleshooting Development Issues

### Go Module Issues

```bash
# Clear module cache
go clean -modcache

# Re-download dependencies
go mod download

# Tidy modules
go mod tidy
```

### Node Module Issues

```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules
rm -rf node_modules package-lock.json

# Fresh install
npm install
```

### Docker Issues

```bash
# Remove all containers and volumes
docker compose down -v

# Prune unused resources
docker system prune -a

# Rebuild from scratch
docker compose build --no-cache
```

## Next Steps

- [Environment Configuration](./operations/configuration.md) - All variables explained
- [Architecture](./architecture/overview.md) - Understand the system design
- [API Documentation](./api/reference.md) - API reference
