# Container/Presentational Pattern

> **File Purpose**: Separate business logic (containers) from UI (presentational components)
> **Prerequisites**: `composables.md`
> **Agent Use Case**: Reference when designing component architecture

## Pattern Overview

**Presentational Components**: Receive data via props, emit events. No business logic.

**Container Components**: Manage state, fetch data, contain logic. Pass data to presentational components.

## Example

```vue
<!-- Presentational: UserCard.vue -->
<script setup lang="ts">
defineProps<{ user: User; onDelete: (id: number) => void }>()
</script>

<template>
  <div class="user-card">
    <h3>{{ user.name }}</h3>
    <button @click="onDelete(user.id)">Delete</button>
  </div>
</template>

<!-- Container: UserList.vue -->
<script setup lang="ts">
import { useUsers } from '@/composables/useUsers'
import UserCard from './UserCard.vue'

const { users, deleteUser } = useUsers()
</script>

<template>
  <UserCard
    v-for="user in users"
    :key="user.id"
    :user="user"
    :on-delete="deleteUser"
  />
</template>
```

**Benefits**: Presentational components are reusable, testable, and UI-focused.

---

## Navigation
- **Up**: `00-overview.md`
