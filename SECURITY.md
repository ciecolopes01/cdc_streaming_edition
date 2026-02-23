# Security Policy

## Supported Versions

This repository contains documentation and configuration examples, not executable software. However, we take security seriously in the configurations we recommend.

## Reporting a Vulnerability

If you find a security issue in any of our recommended configurations (e.g., a connector example that exposes credentials, a permission grant that is overly broad, or a configuration that creates a security risk), please:

1. **Do not** open a public GitHub issue
2. Email the maintainers at: security@your-org.com
3. Include:
   - Description of the issue
   - Which file and configuration it affects
   - Potential impact
   - Suggested fix (if any)

We will respond within 72 hours and publish a fix with credit to the reporter (unless anonymity is requested).

## Security Best Practices in This Project

All connector examples in this repository follow these principles:

- **No hardcoded credentials**: all passwords use `${file:...}` externalizer pattern
- **Least privilege**: database users are granted only the minimum permissions required
- **Encryption in transit**: TLS is assumed for all DB → Debezium and Debezium → Kafka connections
- **Encryption at rest**: storage-level encryption (SSE-KMS / CMK) is documented and recommended
- **Secret management**: we recommend Vault, AWS Secrets Manager, or Kubernetes Secrets for production credentials

## Known Considerations

- The `SELECT ANY TABLE` grant required for Oracle is broad. In environments with strict data isolation, consider limiting to specific schemas using fine-grained Oracle VPD policies.
- `FLASHBACK ANY TABLE` on Oracle should be reviewed against your security policy — it grants read access to historical row versions.
- `errors.log.include.messages=true` in connector configs will log message content to Kafka Connect logs — disable if messages may contain PII.
