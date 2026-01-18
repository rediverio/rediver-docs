---
layout: home
title: Rediver CTEM Platform
---

<p align="center">
  <img src="docs/images/logo.png" alt="Rediver Logo" width="200">
</p>

<h1 align="center">Rediver CTEM Platform</h1>

<p align="center">
  <strong>Continuous Threat Exposure Management Platform</strong><br>
  Unified Attack Surface Management & Vulnerability Management
</p>

<p align="center">
  <a href="https://github.com/rediverio/api"><img src="https://img.shields.io/badge/Go-1.25+-00ADD8?logo=go" alt="Go Version"></a>
  <a href="https://github.com/rediverio/ui"><img src="https://img.shields.io/badge/Next.js-16-000000?logo=next.js" alt="Next.js"></a>
  <a href="https://hub.docker.com/u/rediverio"><img src="https://img.shields.io/badge/Docker-Hub-2496ED?logo=docker" alt="Docker"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License"></a>
</p>

<p align="center">
  <a href="https://rediver.io">Website</a> •
  <a href="https://app.rediver.io">Platform</a> •
  <a href="https://api.rediver.io/docs">API Docs</a> •
  <a href="docs/getting-started.md">Getting Started</a>
</p>

---

## What is Rediver?

Rediver is an enterprise-grade **Continuous Threat Exposure Management (CTEM)** platform that helps security teams continuously monitor, assess, and remediate security risks across their digital infrastructure.

### The CTEM 5-Stage Process

```
┌─────────────┐    ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    ┌──────────────┐
│   SCOPING   │───▶│  DISCOVERY  │───▶│  PRIORITIZATION  │───▶│  VALIDATION │───▶│ MOBILIZATION │
│             │    │             │    │                  │    │             │    │              │
│ Define your │    │ Find assets │    │ Rank by risk &   │    │ Verify with │    │ Remediate &  │
│ attack      │    │ & exposures │    │ business impact  │    │ scanning    │    │ track tasks  │
│ surface     │    │             │    │                  │    │             │    │              │
└─────────────┘    └─────────────┘    └──────────────────┘    └─────────────┘    └──────────────┘
```

---

## Key Features

| Category | Features |
|----------|----------|
| **Asset Management** | 6 asset types (Domains, Websites, Services, Repositories, Cloud, Credentials) |
| **Vulnerability Management** | Findings, CVE tracking, CVSS scoring, SLA policies |
| **Scan Management** | Workers, Scan Profiles, Pipelines, Tool Categories |
| **Multi-tenancy** | Teams, Role-based access (Owner/Admin/Member/Viewer) |
| **Integrations** | SDK for custom tools, Agent for CI/CD, SCM connections |
| **Security** | JWT/OIDC auth, CSRF protection, audit logging |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Rediver Platform                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│   │   Web UI    │    │   REST API  │    │  Database   │    │    Cache    │ │
│   │  (Next.js)  │───▶│    (Go)     │───▶│ (PostgreSQL)│    │   (Redis)   │ │
│   │  Port 3000  │    │  Port 8080  │    │             │    │             │ │
│   └─────────────┘    └──────┬──────┘    └─────────────┘    └─────────────┘ │
│                             │                                               │
│                             ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        Agent / SDK Integration                       │  │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────────┐ │  │
│   │  │  Semgrep  │  │   Trivy   │  │ Gitleaks  │  │   Custom Tools    │ │  │
│   │  │   (SAST)  │  │   (SCA)   │  │ (Secrets) │  │   (SDK-built)     │ │  │
│   │  └───────────┘  └───────────┘  └───────────┘  └───────────────────┘ │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation

### Getting Started
| Guide | Description |
|-------|-------------|
| [Quick Start](docs/getting-started.md) | Get up and running in 10 minutes |
| [Development Setup](docs/development-setup.md) | IDE, debugging, testing |
| [Configuration](docs/operations/configuration.md) | Environment variables |

### Guides
| Guide | Description |
|-------|-------------|
| [Authentication](docs/guides/authentication.md) | Login flow, JWT, sessions |
| [Multi-tenancy](docs/guides/multi-tenancy.md) | Teams, tenant switching |
| [Permissions](docs/guides/permissions.md) | Role-based access control |
| [Running Workers](docs/guides/running-workers.md) | Setup and run scanning agents |
| [SDK Development](docs/guides/sdk-development.md) | Build custom scanners |
| [Building Ingestion Tools](docs/guides/building-ingestion-tools.md) | Custom data collectors |

### Architecture
| Document | Description |
|----------|-------------|
| [Overview](docs/architecture/overview.md) | System design |
| [Deployment Modes](docs/architecture/deployment-modes.md) | Standalone, distributed |
| [Server-Agent Communication](docs/architecture/server-agent-command.md) | Command & control |
| [Scan Pipeline Design](docs/architecture/scan-pipeline-design.md) | Workflow execution |

### Reference
| Document | Description |
|----------|-------------|
| [API Reference](docs/api/reference.md) | Complete API endpoints |
| [RIS Schema](https://github.com/rediverio/schemas) | Rediver Ingest Schema |

### Operations
| Document | Description |
|----------|-------------|
| [Troubleshooting](docs/operations/troubleshooting.md) | Common issues |
| [Docker Deployment](docs/guides/docker-deployment.md) | Container deployment |

---

## 🚀 Quick Start

```bash
# Clone repository
git clone https://github.com/rediverio/rediver.git
cd rediver

# Configure
cd api && cp .env.example .env && cd ..
cd ui && cp .env.example .env.local && cd ..

# Start with Docker
docker compose up -d
```

| Service | Local | Production |
|---------|-------|------------|
| Frontend | http://localhost:3000 | https://app.rediver.io |
| Backend API | http://localhost:8080 | https://api.rediver.io |
| API Docs | http://localhost:8080/docs | https://api.rediver.io/docs |

---

## 🛠 Tech Stack

| Component | Technologies |
|-----------|-------------|
| **Backend** | Go 1.25, Chi Router, PostgreSQL 17, Redis 7 |
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind 4 |
| **Auth** | JWT (local) / Keycloak (OIDC) |
| **SDK** | Go SDK with Scanner/Parser/Collector interfaces |

---

## 📦 Repositories

| Repository | Description |
|------------|-------------|
| [api](https://github.com/rediverio/api) | Backend REST API (Go) |
| [ui](https://github.com/rediverio/ui) | Frontend Application (Next.js) |
| [sdk](https://github.com/rediverio/sdk) | Go SDK for building tools |
| [agent](https://github.com/rediverio/agent) | Security scanning agent |
| [setup](https://github.com/rediverio/setup) | Deployment & Docker Compose |
| [schemas](https://github.com/rediverio/schemas) | RIS JSON Schemas |
| [keycloak](https://github.com/rediverio/keycloak) | Keycloak Configuration |
| [docs](https://github.com/rediverio/docs) | Documentation (this repo) |

---

## 🤝 Contributing

We welcome contributions! Please see:

- [Contributing Guide](CONTRIBUTING.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Security Policy](SECURITY.md)

---

## 💖 Support

If you find Rediver useful, consider supporting the project:

**BSC Network (BEP-20):**
```
0x97f0891b4a682904a78e6Bc854a58819Ea972454
```

---

## 📧 Contact

- **Website:** https://rediver.io
- **Email:** rediverio@gmail.com
- **GitHub:** https://github.com/rediverio

---

## 📄 License

MIT License - see [LICENSE](https://github.com/rediverio/api/blob/main/LICENSE)
