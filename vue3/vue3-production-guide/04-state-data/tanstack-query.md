# TanStack Query (Vue Query)

> **File Purpose**: Manage server state with caching and automatic refetching
> **Agent Use Case**: Reference when fetching data from APIs

## Setup

```typescript
// main.ts
import { VueQueryPlugin } from '@tanstack/vue-query'

app.use(VueQueryPlugin)
```

## Usage

```typescript
// composables/useUser.ts
import { useQuery } from '@tanstack/vue-query'
import { fetchUser } from '@/services/api/users'

export function useUser(userId: Ref<string>) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId.value),
    staleTime: 5 * 60 * 1000 // 5 minutes
  })
}
```

```vue
<script setup lang="ts">
const userId = ref('123')
const { data, isLoading, isError, error } = useUser(userId)
</script>

<template>
  <div v-if="isLoading">Loading...</div>
  <div v-else-if="isError">Error: {{ error.message }}</div>
  <div v-else>{{ data.name }}</div>
</template>
```

**Benefits**: Automatic caching, background refetching, optimistic updates.

---

## Navigation
- **Up**: `00-overview.md`
