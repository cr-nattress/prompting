# Pinia Stores Implementation

> **File Purpose**: Implement fully-typed global state with Pinia
> **Prerequisites**: `state-strategy.md`
> **Agent Use Case**: Reference when creating global state stores
> **Related Files**: `02-core-concepts/typescript-guide.md`

## Setup Store Syntax (Recommended)

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types/models'

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  // Getters
  const isAuthenticated = computed(() => !!user.value)
  const userName = computed(() => user.value?.name ?? 'Guest')

  // Actions
  function setUser(newUser: User, authToken: string) {
    user.value = newUser
    token.value = authToken
  }

  function logout() {
    user.value = null
    token.value = null
  }

  return {
    // State
    user,
    token,
    // Getters
    isAuthenticated,
    userName,
    // Actions
    setUser,
    logout
  }
})
```

## Usage in Components

```vue
<script setup lang="ts">
import { useAuthStore } from '@/stores/auth'

const authStore = useAuthStore()

// Access state
console.log(authStore.user)

// Access getters
console.log(authStore.isAuthenticated)

// Call actions
authStore.setUser({ id: 1, name: 'John' }, 'token123')
</script>

<template>
  <div v-if="authStore.isAuthenticated">
    Welcome, {{ authStore.userName }}
  </div>
</template>
```

## Async Actions

```typescript
export const useUserStore = defineStore('user', () => {
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function fetchUsers() {
    loading.value = true
    error.value = null
    try {
      const response = await apiClient.get<User[]>('/users')
      users.value = response.data
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  return { users, loading, error, fetchUsers }
})
```

## Store Composition

```typescript
// stores/cart.ts
import { useAuthStore } from './auth'

export const useCartStore = defineStore('cart', () => {
  const authStore = useAuthStore()
  const items = ref<CartItem[]>([])

  function addItem(item: Product) {
    if (!authStore.isAuthenticated) {
      throw new Error('Must be logged in to add to cart')
    }
    items.value.push({ ...item, quantity: 1 })
  }

  return { items, addItem }
})
```

---

## Navigation

- **Previous**: `04-state-data/state-strategy.md`
- **Next**: `04-state-data/tanstack-query.md`
- **Up**: `00-overview.md`
