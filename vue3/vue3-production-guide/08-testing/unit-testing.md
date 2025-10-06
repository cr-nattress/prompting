# Unit Testing with Vitest

> **File Purpose**: Test composables, stores, and utility functions
> **Agent Use Case**: Reference when writing unit tests

## Testing a Composable

```typescript
// useFetch.spec.ts
import { describe, it, expect } from 'vitest'
import { useFetch } from './useFetch'

describe('useFetch', () => {
  it('should fetch data', async () => {
    const { data, execute } = useFetch('/api/users')
    await execute()
    expect(data.value).toBeDefined()
  })
})
```

## Testing a Pinia Store

```typescript
// auth.spec.ts
import { setActivePinia, createPinia } from 'pinia'
import { useAuthStore } from './auth'

beforeEach(() => {
  setActivePinia(createPinia())
})

it('should set user on login', () => {
  const store = useAuthStore()
  store.setUser({ id: 1, name: 'John' }, 'token')
  expect(store.user?.name).toBe('John')
})
```

---

## Navigation
- **Up**: `00-overview.md`
