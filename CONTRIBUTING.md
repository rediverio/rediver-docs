# Contributing to ReDiver

Thank you for your interest in contributing to ReDiver! This document provides guidelines for contributing to the project.

## Repositories

| Repository | Description |
|------------|-------------|
| [rediver-api](https://github.com/rediverio/rediver-api) | Backend (Go) |
| [rediver-ui](https://github.com/rediverio/rediver-ui) | Frontend (Next.js) |
| [rediver-keycloak](https://github.com/rediverio/rediver-keycloak) | Keycloak Config |
| [rediver-schemas](https://github.com/rediverio/rediver-schemas) | Database Schemas |
| [rediver-docs](https://github.com/rediverio/rediver-docs) | Documentation |

## Development Workflow

1. **Fork** the repository
2. **Clone** your fork locally
3. **Create** a feature branch: `git checkout -b feature/your-feature`
4. **Make** changes and test
5. **Commit** using conventional commits
6. **Push** to your fork
7. **Open** a Pull Request

## Commit Convention

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new feature
fix: fix a bug
docs: update documentation
style: code formatting
refactor: code refactoring
perf: performance improvement
test: add tests
chore: maintenance
```

## Code Style

### Backend (Go)
- Run `make lint` before committing
- Follow [Effective Go](https://golang.org/doc/effective_go)
- Use `gofmt` for formatting

### Frontend (TypeScript)
- Run `npm run lint` before committing
- Follow ESLint configuration
- Use Prettier for formatting

## Pull Request Process

1. Update documentation if needed
2. Add tests for new features
3. Ensure CI passes
4. Request review from maintainers
5. Squash commits before merge

## Reporting Issues

- Check existing issues first
- Use issue templates
- Provide reproduction steps
- Include environment details

## Getting Help

- Check [Documentation](docs/getting-started.md)
- Search [Issues](https://github.com/rediverio/rediver/issues)
- Ask in [Discussions](https://github.com/rediverio/rediver/discussions)
- Email: rediverio@gmail.com
