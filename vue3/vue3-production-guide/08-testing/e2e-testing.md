# E2E Testing with Playwright

> **File Purpose**: Test critical user flows end-to-end
> **Agent Use Case**: Reference when writing E2E tests

## Example

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test('should allow user to login', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name="email"]', 'user@example.com')
  await page.fill('[name="password"]', 'password123')
  await page.click('button[type="submit"]')

  await expect(page).toHaveURL('/dashboard')
  await expect(page.locator('h1')).toContainText('Dashboard')
})
```

---

## Navigation
- **Up**: `00-overview.md`
