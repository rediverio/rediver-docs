# Security Policy

## Reporting a Vulnerability

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, please report them via email to: **rediverio@gmail.com**

Please include:

- Type of issue (e.g., buffer overflow, SQL injection, cross-site scripting)
- Full paths of source file(s) related to the issue
- Location of the affected source code (tag/branch/commit or direct URL)
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact of the issue, including how an attacker might exploit it

## Response Timeline

- **Initial Response**: Within 48 hours
- **Status Update**: Within 7 days
- **Resolution**: Depends on severity and complexity

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 1.x.x   | :white_check_mark: |
| < 1.0   | :x:                |

## Security Features

ReDiver implements several security measures:

- **Authentication**: JWT with token rotation
- **Authorization**: Role-based access control (RBAC)
- **Data Protection**: Encryption at rest and in transit
- **Session Management**: Configurable session limits
- **CSRF Protection**: Double-submit cookie pattern
- **Rate Limiting**: Configurable request limits

## Security Best Practices

1. Use strong, unique JWT secrets (64+ characters)
2. Enable HTTPS in production
3. Configure proper CORS settings
4. Enable rate limiting
5. Use secure cookie settings
6. Regularly rotate secrets

See [Security Configuration](docs/operations/configuration.md) for details.
