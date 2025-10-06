# Code Splitting Strategies

> **File Purpose**: Implement lazy loading for routes and components
> **Agent Use Case**: Reference when optimizing bundle size

## Route-Level Splitting

```typescript
const routes = [
  {
    path: '/dashboard',
    component: () => import('@/views/DashboardView.vue') // Lazy-loaded
  }
]
```

## Component-Level Splitting

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent(
  () => import('@/components/HeavyChart.vue')
)
</script>

<template>
  <Suspense>
    <HeavyChart />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

**Result**: Smaller initial bundle, faster load times.

---

## Navigation
- **Up**: `00-overview.md`
