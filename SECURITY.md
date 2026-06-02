# Security Policy

## Reporting a Vulnerability

**Do NOT open a public issue.** Contact the maintainer directly:

- Email: vladarchitect@github
- Subject: "Security Vulnerability: sandbox-migration-shield"

### Response Timeline
- Acknowledgment: 48 hours
- Assessment: 5 business days
- Critical fix: 24-48 hours
- High fix: 1 week

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest (main) | ✅ Yes |
| Previous releases | ❌ No |

## Security Design

- Snapshots encrypted at rest (GlideEncrypter)
- Credential auto-redaction in snapshots
- External backup over HTTPS only
- Cross-instance restore blocked
- Read-only on production
- Audit logging for all operations
