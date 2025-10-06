# Vue Router 4 Setup

> **File Purpose**: Configure routing with lazy loading and auth guards
> **Agent Use Case**: Reference when setting up navigation

## Lazy-Loaded Routes

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/views/HomeView.vue')
    },
    {
      path: '/dashboard',
      component: () => import('@/views/DashboardView.vue'),
      meta: { requiresAuth: true }
    }
  ]
})

// Auth guard
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isAuthenticated()) {
    return { name: 'login' }
  }
})

export default router
```

---

## Navigation
- **Up**: `00-overview.md`
