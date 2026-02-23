# Contributing to CDC Enterprise Extreme

Thank you for your interest in contributing! This project is built from real production experience and peer review. We welcome contributions that meet the same standard.

---

## What We're Looking For

- **Corrections**: technical inaccuracies, outdated information, wrong configurations
- **New connectors**: Cassandra, CockroachDB, Vitess, Db2, SAP HANA
- **Post-mortems**: real incidents from the community (anonymized is fine)
- **Runbooks**: additional incident response procedures
- **Deployment examples**: Kubernetes/Strimzi, Confluent Operator, Terraform
- **Translations**: the guide in other languages

---

## Contribution Process

### 1. Open an issue first

For non-trivial changes, open an issue before writing code or documentation. Describe:
- What you want to add or fix
- Why it matters in production
- Any sources or references

### 2. Fork and branch

```bash
git clone https://github.com/your-org/cdc-enterprise-extreme.git
cd cdc-enterprise-extreme
git checkout -b feat/your-feature-name
```

### 3. Make your changes

Follow the style of existing documentation:
- Be specific and technical — avoid vague statements
- Include code examples for every configuration recommendation
- Cite production impact: what breaks if this is wrong?
- Use Mermaid diagrams for flows and architectures
- Add warnings (⚠️) for non-obvious pitfalls

### 4. Submit a pull request

- Fill out the PR template completely
- Link to the related issue
- Wait for at least one review from a maintainer

---

## Documentation Standards

### Every connector section must include

- How the change log mechanism works (diagram preferred)
- Prerequisites and setup SQL/config
- Full connector JSON example
- Offset key explanation and code
- At least two production pitfalls with remediation
- Monitoring queries

### Post-mortems must include

- What happened (timeline)
- Root cause (specific, not vague)
- Resolution (with commands)
- Prevention (concrete config or process change)

---

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md). We follow the Contributor Covenant.

---

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
