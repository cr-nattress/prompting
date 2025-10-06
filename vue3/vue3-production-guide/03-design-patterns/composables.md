# Composable Functions Pattern

> **File Purpose**: Master the composable pattern for reusable stateful logic
> **Prerequisites**: `02-core-concepts/composition-api.md`, `02-core-concepts/reactivity-deep-dive.md`
> **Agent Use Case**: Reference when extracting reusable logic from components
> **Related Files**: `04-state-data/state-strategy.md`

## In One Sentence

Composables are functions that use Composition API primitives to encapsulate and reuse stateful logic across components, replacing Vue 2's mixins.

## Why This Matters

Composables enable:
- **Logic reuse** without component inheritance
- **Better code organization** (separation of concerns)
- **Testability** (pure functions, no component context needed)
- **Type safety** with TypeScript

## Basic Pattern

```typescript
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event: MouseEvent) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}
```

Usage:

```vue
<script setup lang="ts">
import { useMouse } from '@/composables/useMouse'

const { x, y } = useMouse()
</script>

<template>
  <div>Mouse position: {{ x }}, {{ y }}</div>
</template>
```

## Advanced Composable Patterns

### Async Data Fetching

```typescript
// composables/useFetch.ts
import { ref, type Ref } from 'vue'

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  refetch: () => Promise<void>
}

export function useFetch<T>(url: string): UseFetchReturn<T> {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function fetchData() {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(url)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  fetchData() // Execute on creation

  return {
    data,
    error,
    loading,
    refetch: fetchData
  }
}
```

### Accepting Reactive Arguments

```typescript
// composables/useFilteredList.ts
import { computed, type Ref } from 'vue'
import { toValue, type MaybeRefOrGetter } from 'vue'

export function useFilteredList<T>(
  list: MaybeRefOrGetter<T[]>,
  filter: MaybeRefOrGetter<string>
) {
  return computed(() => {
    const listValue = toValue(list)
    const filterValue = toValue(filter)
    return listValue.filter(item =>
      String(item).toLowerCase().includes(filterValue.toLowerCase())
    )
  })
}

// Usage
const items = ref(['Apple', 'Banana', 'Cherry'])
const search = ref('a')
const filtered = useFilteredList(items, search) // reactive!
```

### Local Storage Composable

```typescript
// composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue'

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const storedValue = localStorage.getItem(key)
  const data = ref<T>(storedValue ? JSON.parse(storedValue) : defaultValue)

  watch(
    data,
    (newValue) => {
      localStorage.setItem(key, JSON.stringify(newValue))
    },
    { deep: true }
  )

  return data as Ref<T>
}

// Usage
const theme = useLocalStorage('theme', 'light')
theme.value = 'dark' // Automatically saved to localStorage
```

### State Machine Composable

```typescript
// composables/useFormState.ts
import { ref, computed } from 'vue'

type FormState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; message: string }
  | { status: 'error'; error: string }

export function useFormState() {
  const state = ref<FormState>({ status: 'idle' })

  const isSubmitting = computed(() => state.value.status === 'submitting')
  const isSuccess = computed(() => state.value.status === 'success')
  const isError = computed(() => state.value.status === 'error')

  function setSubmitting() {
    state.value = { status: 'submitting' }
  }

  function setSuccess(message: string) {
    state.value = { status: 'success', message }
  }

  function setError(error: string) {
    state.value = { status: 'error', error }
  }

  function reset() {
    state.value = { status: 'idle' }
  }

  return {
    state,
    isSubmitting,
    isSuccess,
    isError,
    setSubmitting,
    setSuccess,
    setError,
    reset
  }
}
```

## Best Practices

### Naming Convention

- **Prefix with `use`**: `useMouse`, `useFetch`, `useAuth`
- **Be descriptive**: `useUserAuthentication` > `useAuth` if ambiguous

### Return Values

```typescript
// Good: Object with named properties (destructurable)
export function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }
}

// Avoid: Array return (loses names)
export function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return [count, increment] // Less clear
}
```

### Side Effects Cleanup

Always clean up side effects in `onUnmounted`:

```typescript
export function useWebSocket(url: string) {
  const data = ref(null)
  const ws = new WebSocket(url)

  ws.onmessage = (event) => {
    data.value = event.data
  }

  onUnmounted(() => {
    ws.close() // Important: prevent memory leaks
  })

  return { data }
}
```

### Composing Composables

Composables can call other composables:

```typescript
export function useAuth() {
  const user = ref<User | null>(null)
  const token = useLocalStorage('auth_token', '')

  function login(credentials: Credentials) {
    // Use token from useLocalStorage
    // ...
  }

  return { user, login }
}
```

## Testing Composables

```typescript
// useMouse.test.ts
import { describe, it, expect } from 'vitest'
import { useMouse } from './useMouse'

describe('useMouse', () => {
  it('should track mouse position', () => {
    const { x, y } = useMouse()

    expect(x.value).toBe(0)
    expect(y.value).toBe(0)

    // Simulate mouse event
    const event = new MouseEvent('mousemove', { pageX: 100, pageY: 200 })
    window.dispatchEvent(event)

    expect(x.value).toBe(100)
    expect(y.value).toBe(200)
  })
})
```

---

## Navigation

- **Previous**: `02-core-concepts/reactivity-deep-dive.md`
- **Next**: `03-design-patterns/container-presentational.md`
- **Up**: `00-overview.md`
- **See Also**: `04-state-data/state-strategy.md` for when to use composables vs Pinia
