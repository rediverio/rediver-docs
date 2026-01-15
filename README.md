# ReDiver - CTEM Platform

<p align="center">
  <strong>Continuous Threat Exposure Management Platform</strong>
</p>

<p align="center">
  <a href="https://rediver.io">Website</a> •
  <a href="https://app.rediver.io">Platform</a> •
  <a href="https://api.rediver.io/docs">API Docs</a> •
  <a href="docs/getting-started.md">Getting Started</a>
</p>

---

ReDiver helps organizations manage security risks through the **CTEM 5-stage process**:

**Scoping** → **Discovery** → **Prioritization** → **Validation** → **Mobilization**

---

## 📚 Documentation

### Getting Started
- [Quick Start](docs/getting-started.md) - Get up and running
- [Development Setup](docs/development-setup.md) - IDE, debugging, testing
- [Configuration](docs/operations/configuration.md) - Environment variables

### Guides
- [Authentication](docs/guides/authentication.md) - Login flow, JWT, sessions
- [Multi-tenancy](docs/guides/multi-tenancy.md) - Teams, tenant switching
- [Permissions](docs/guides/permissions.md) - Role-based access control

### Reference
- [API Reference](docs/api/reference.md) - Complete API endpoints
- [Architecture](docs/architecture/overview.md) - System design

### Operations
- [Troubleshooting](docs/operations/troubleshooting.md) - Common issues

---

## 🚀 Quick Start

```bash
# Clone repository
git clone https://github.com/rediverio/rediver.git
cd rediver

# Configure
cd rediver-api && cp .env.example .env && cd ..
cd rediver-ui && cp .env.example .env.local && cd ..

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
| **Backend** | Go 1.25, Chi, PostgreSQL 17, Redis 7 |
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind 4 |
| **Auth** | JWT (local) / Keycloak (OIDC) |

---

## 🤝 Contributing

We welcome contributions! Please see:

- [Contributing Guide](CONTRIBUTING.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Security Policy](SECURITY.md)

---

## 💖 Support

If you find ReDiver useful, consider supporting the project:

**BSC Network (BEP-20):**
```
0x97f0891b4a682904a78e6Bc854a58819Ea972454
```

---

## 📦 Repositories

| Repository | Description |
|------------|-------------|
| [rediver-api](https://github.com/rediverio/rediver-api) | Backend REST API (Go) |
| [rediver-ui](https://github.com/rediverio/rediver-ui) | Frontend Application (Next.js) |
| [rediver-keycloak](https://github.com/rediverio/rediver-keycloak) | Keycloak Configuration |
| [rediver-schemas](https://github.com/rediverio/rediver-schemas) | Database Schemas |
| [rediver-docs](https://github.com/rediverio/rediver-docs) | Documentation |

---

## 📧 Contact

- **Website:** https://rediver.io
- **Email:** rediverio@gmail.com
- **GitHub:** https://github.com/rediverio

---

## 📄 License

MIT License - see [LICENSE](https://github.com/rediverio/rediver-api/blob/main/LICENSE)
