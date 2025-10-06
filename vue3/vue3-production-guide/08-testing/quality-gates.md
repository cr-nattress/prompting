# Quality Gates & CI Integration

> **File Purpose**: Enforce coverage thresholds in CI
> **Agent Use Case**: Reference when setting up CI quality gates

## Vitest Coverage

```json
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      statements: 80,
      branches: 80,
      functions: 80,
      lines: 80
    }
  }
})
```

## GitHub Actions

```yaml
- name: Run tests with coverage
  run: npm run test:unit -- --coverage

- name: Fail if coverage below threshold
  run: npm run test:unit -- --coverage --coverage.thresholds.lines=80
```

---

## Navigation
- **Up**: `00-overview.md`
