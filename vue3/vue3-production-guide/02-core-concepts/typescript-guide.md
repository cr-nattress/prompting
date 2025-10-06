# TypeScript Guide for Vue 3

> **File Purpose**: Learn advanced TypeScript patterns and strict configuration for Vue 3
> **Prerequisites**: `01-quick-start/tooling-config.md`
> **Agent Use Case**: Reference when implementing type-safe Vue components and state
> **Related Files**: `composition-api.md`, `reactivity-deep-dive.md`

## In One Sentence

Leverage TypeScript's strictest settings and advanced types (`as const`, `satisfies`, discriminated unions) to catch errors at compile time and improve code maintainability.

## Advanced Type Patterns

### as const for Readonly Types

Creates deeply immutable literal types:

```typescript
// Without as const
const routes = ['home', 'about', 'contact']
// Type: string[]

// With as const
const routes = ['home', 'about', 'contact'] as const
// Type: readonly ["home", "about", "contact"]

// Usage: Type-safe route checking
type Route = typeof routes[number] // "home" | "about" | "contact"

function navigateTo(route: Route) {
  // route is guaranteed to be one of the three values
}
```

### satisfies Operator

Validates type without widening:

```typescript
type Config = {
  apiUrl: string
  timeout: number
  retries?: number
}

// Problem: Type is widened
const config: Config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000
}
// config.apiUrl is type 'string', loses literal

// Solution: satisfies preserves literal types
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3
} satisfies Config
// config.apiUrl is type 'https://api.example.com'
// Still validates against Config interface
```

### Discriminated Unions for State Machines

Best practice for modeling mutually exclusive states:

```typescript
// Bad: Multiple booleans (can have impossible states)
interface BadState {
  isLoading: boolean
  isError: boolean
  isSuccess: boolean
  data?: User
  error?: Error
}

// Good: Discriminated union (impossible states are impossible)
type FetchState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: Error }

// Usage with type narrowing
function handleState(state: FetchState) {
  switch (state.status) {
    case 'idle':
      return 'Not started'
    case 'loading':
      return 'Loading...'
    case 'success':
      return state.data.name // TypeScript knows data exists
    case 'error':
      return state.error.message // TypeScript knows error exists
  }
}
```

### Generic Components

```typescript
// Generic list component
interface Props<T> {
  items: T[]
  renderItem: (item: T) => string
}

// Usage
defineProps<Props<User>>()
```

### Utility Types

```typescript
import type { User } from '@/types/models'

// Pick: Select subset of properties
type UserPreview = Pick<User, 'id' | 'name' | 'email'>

// Omit: Exclude properties
type UserWithoutPassword = Omit<User, 'password'>

// Partial: Make all properties optional
type PartialUser = Partial<User>

// Required: Make all properties required
type RequiredUser = Required<Partial<User>>

// Record: Create object type with specific keys
type UserMap = Record<string, User>

// ReturnType: Extract function return type
function getUser() {
  return { id: 1, name: 'John' }
}
type UserReturn = ReturnType<typeof getUser> // { id: number; name: string }
```

## Type-Safe Pinia Stores

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types/models'

export const useUserStore = defineStore('user', () => {
  // State
  const currentUser = ref<User | null>(null)
  const isAuthenticated = computed(() => currentUser.value !== null)

  // Actions
  function setUser(user: User) {
    currentUser.value = user
  }

  function logout() {
    currentUser.value = null
  }

  return {
    currentUser,
    isAuthenticated,
    setUser,
    logout
  }
})

// Type-safe usage
const userStore = useUserStore()
userStore.currentUser // Type: Ref<User | null>
userStore.setUser({ id: 1, name: 'John' }) // Type-checked
```

## Type-Safe Composables

```typescript
import { ref, type Ref } from 'vue'

interface UseFetchOptions {
  immediate?: boolean
}

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useFetch<T>(
  url: string,
  options: UseFetchOptions = {}
): UseFetchReturn<T> {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(url)
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  if (options.immediate) {
    execute()
  }

  return { data, error, loading, execute }
}

// Usage with type inference
const { data, loading, error } = useFetch<User>('/api/user')
// data is Ref<User | null>
```

## Strict TypeScript Options Explained

From `tsconfig.app.json`:

```json
{
  "compilerOptions": {
    // Core strict checks
    "strict": true,                          // Enable all strict checks
    "noUncheckedIndexedAccess": true,       // array[i] returns T | undefined
    "exactOptionalPropertyTypes": true,      // Distinguish missing vs undefined

    // Code quality
    "noUnusedLocals": true,                 // Error on unused variables
    "noUnusedParameters": true,             // Error on unused function params
    "noFallthroughCasesInSwitch": true,     // Prevent switch fallthrough bugs
    "noImplicitOverride": true,             // Require 'override' keyword

    // Module handling
    "verbatimModuleSyntax": true,           // Required for Vite
    "moduleResolution": "bundler",          // Use bundler resolution
    "resolveJsonModule": true               // Allow importing JSON files
  }
}
```

## Common Type Issues & Solutions

### Issue: Ref unwrapping in templates

```typescript
// In script
const count = ref(0)
count.value // Must use .value

// In template
<template>
  {{ count }} <!-- Auto-unwrapped, don't use .value -->
</template>
```

### Issue: Typing event handlers

```typescript
function handleClick(event: MouseEvent) {
  const target = event.target as HTMLButtonElement
  console.log(target.textContent)
}

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  console.log(target.value)
}
```

### Issue: Typing async data

```typescript
// Bad: data could be undefined during async load
const data = ref<User>()

// Good: Explicitly model loading state
const data = ref<User | null>(null)

// Better: Use discriminated union
const state = ref<
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: Error }
>({ status: 'loading' })
```

### Issue: Component instance types

```typescript
import MyComponent from './MyComponent.vue'

const componentRef = ref<InstanceType<typeof MyComponent> | null>(null)
```

---

## Navigation

- **Previous**: `02-core-concepts/composition-api.md`
- **Next**: `02-core-concepts/reactivity-deep-dive.md`
- **Up**: `00-overview.md`
- **See Also**: `04-state-data/pinia-stores.md` for store type patterns
