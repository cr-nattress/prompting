# Supply Chain Security

> **File Purpose**: Secure dependencies with locking and vulnerability scanning
> **Agent Use Case**: Reference when managing dependencies

## Best Practices

1. **Lock dependencies**: Use `package-lock.json` and `npm ci` in CI
2. **Audit regularly**: Run `npm audit` to check for vulnerabilities
3. **Automated scanning**: Enable Dependabot or Snyk

```bash
# In CI/CD
npm ci  # Not npm install

# Check for vulnerabilities
npm audit
npm audit fix
```

## GitHub Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Navigation
- **Up**: `00-overview.md`
