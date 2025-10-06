# State Management Strategy

> **File Purpose**: Decision framework for choosing between local (composables) and global (Pinia) state
> **Prerequisites**: `03-design-patterns/composables.md`
> **Agent Use Case**: Reference when deciding where state should live
> **Related Files**: `pinia-stores.md`, `tanstack-query.md`

## In One Sentence

Use composables for local/feature-specific state (each instance gets its own state); use Pinia only for truly global state shared across unrelated components.

## The Decision Matrix

| State Type | Example | Solution |
|------------|---------|----------|
| UI state (single component) | Modal open/closed, form input | Component `ref` |
| UI state (parent + children) | Accordion active panel | Props + emits |
| Feature state (isolated) | Product filter state | Composable |
| Shared state (2-3 related components) | Composable (singleton) or props |
| Global state (app-wide) | Current user, cart, theme | Pinia store |
| Server state | API data, caching | Vue Query |

## Pattern 1: Component-Local State

**Use when**: State is only used within one component

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isModalOpen = ref(false)
const formData = ref({ name: '', email: '' })
</script>
```

## Pattern 2: Props + Emits (Parent-Child)

**Use when**: State needs to be shared between parent and direct children

```vue
<!-- Parent -->
<script setup lang="ts">
import { ref } from 'vue'
const selectedId = ref(1)
</script>

<template>
  <ChildComponent
    :selected-id="selectedId"
    @update:selected-id="selectedId = $event"
  />
</template>

<!-- Child -->
<script setup lang="ts">
defineProps<{ selectedId: number }>()
defineEmits<{ (e: 'update:selectedId', id: number): void }>()
</script>
```

## Pattern 3: Composable (Instance-Scoped)

**Use when**: Logic needs to be reused, but each component gets its own state

```typescript
// composables/useCounter.ts
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const increment = () => count.value++
  return { count, increment }
}

// ComponentA.vue - gets its own count
const { count } = useCounter()

// ComponentB.vue - gets a separate count
const { count } = useCounter()
```

## Pattern 4: Composable (Singleton)

**Use when**: Multiple components need to share the same state instance

```typescript
// composables/useFilters.ts
import { ref } from 'vue'

// Defined OUTSIDE the function = shared
const searchQuery = ref('')
const selectedCategory = ref<string | null>(null)

export function useFilters() {
  function reset() {
    searchQuery.value = ''
    selectedCategory.value = null
  }

  return {
    searchQuery,
    selectedCategory,
    reset
  }
}

// ComponentA and ComponentB share the same searchQuery
```

## Pattern 5: Pinia (Global Store)

**Use when**: State is truly global and accessed by many unrelated components

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  const isAuthenticated = computed(() => !!user.value)

  function setUser(newUser: User, authToken: string) {
    user.value = newUser
    token.value = authToken
  }

  function logout() {
    user.value = null
    token.value = null
  }

  return { user, token, isAuthenticated, setUser, logout }
})
```

**Global state examples**:
- Current authenticated user
- Shopping cart contents
- Application settings (theme, language)
- Notifications queue

## Pattern 6: Vue Query (Server State)

**Use when**: Data comes from an API and needs caching/revalidation

```typescript
import { useQuery } from '@tanstack/vue-query'

export function useUser(userId: Ref<string>) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId.value)
  })
}
```

See `tanstack-query.md` for details.

## Decision Flowchart

```
Is it server data from an API?
├─ Yes → Use Vue Query (tanstack-query.md)
└─ No ↓

Is it used in only one component?
├─ Yes → Use component ref
└─ No ↓

Is it only shared between parent and children?
├─ Yes → Use props + emits
└─ No ↓

Is the logic reusable but each instance needs separate state?
├─ Yes → Use composable (instance-scoped)
└─ No ↓

Is it shared across 2-3 related components?
├─ Yes → Use composable (singleton)
└─ No ↓

Is it truly global (user auth, cart, app settings)?
└─ Yes → Use Pinia store
```

## Anti-Patterns to Avoid

### 1. Over-using Pinia

```typescript
// ❌ Don't create a store for UI state
export const useModalStore = defineStore('modal', () => {
  const isOpen = ref(false)
  return { isOpen }
})

// ✅ Just use a ref in the component
const isModalOpen = ref(false)
```

### 2. Prop Drilling

```typescript
// ❌ Passing props through 5 levels
<GrandParent :user="user">
  <Parent :user="user">
    <Child :user="user">
      <DeepChild :user="user" />

// ✅ Use Pinia for deeply nested access
const userStore = useAuthStore()
```

### 3. Mixing Concerns

```typescript
// ❌ Don't mix UI state with global state
export const useUserStore = defineStore('user', () => {
  const currentUser = ref<User | null>(null)
  const isModalOpen = ref(false) // UI state doesn't belong here!
  return { currentUser, isModalOpen }
})
```

---

## Navigation

- **Next**: `04-state-data/pinia-stores.md`
- **See Also**: `04-state-data/tanstack-query.md` for server state
- **Up**: `00-overview.md`
