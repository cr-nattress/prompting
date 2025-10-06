# Secure Authentication Flow

> **File Purpose**: Implement hybrid token authentication (HttpOnly cookies + in-memory access tokens)
> **Prerequisites**: `05-api-layer/axios-setup.md`
> **Agent Use Case**: Reference when implementing authentication
> **Related Files**: `xss-prevention.md`, `supply-chain.md`

## In One Sentence

Use a hybrid approach: store refresh tokens in HttpOnly, SameSite=Strict cookies (protected from XSS) and access tokens in memory (protected from CSRF).

## Why Hybrid Token Flow?

| Storage Method | XSS Protection | CSRF Protection | Best For |
|----------------|----------------|-----------------|----------|
| localStorage | ❌ Vulnerable | ✅ Protected | ❌ Not recommended |
| HttpOnly Cookie | ✅ Protected | ❌ Vulnerable | Refresh tokens |
| In-Memory (Ref) | ✅ Protected | ✅ Protected | Access tokens |

**Hybrid approach**: Combines strengths of both methods.

## Architecture

```
┌─────────────────────────────────────────────┐
│ Client (Vue App)                            │
│                                             │
│  ┌──────────────────┐  ┌─────────────────┐ │
│  │ Access Token     │  │ Refresh Token   │ │
│  │ (In-Memory Ref)  │  │ (HttpOnly Cookie│ │
│  │ Short-lived      │  │ Long-lived)     │ │
│  │ 15 min           │  │ 7 days          │ │
│  └──────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────┘
         │                        │
         │ Authorization:         │ Cookie auto-sent
         │ Bearer <token>         │ on /refresh
         ▼                        ▼
┌─────────────────────────────────────────────┐
│ Backend API                                 │
│  /api/...  (requires access token)          │
│  /auth/refresh (reads refresh cookie)       │
└─────────────────────────────────────────────┘
```

## Implementation

### 1. Auth Store (Pinia)

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types/models'
import { apiClient } from '@/services/api/client'

export const useAuthStore = defineStore('auth', () => {
  // State: Access token in memory (NOT persisted)
  const accessToken = ref<string | null>(null)
  const user = ref<User | null>(null)

  const isAuthenticated = computed(() => !!user.value && !!accessToken.value)

  // Login: Receives access token in response, refresh token set as cookie by server
  async function login(email: string, password: string) {
    const response = await apiClient.post<{ accessToken: string; user: User }>(
      '/auth/login',
      { email, password }
    )

    accessToken.value = response.data.accessToken
    user.value = response.data.user

    // Server should have set HttpOnly refresh token cookie
  }

  // Refresh: Server reads refresh token from cookie, returns new access token
  async function refreshAccessToken() {
    try {
      const response = await apiClient.post<{ accessToken: string }>(
        '/auth/refresh',
        {},
        { withCredentials: true } // Send cookies
      )

      accessToken.value = response.data.accessToken
      return response.data.accessToken
    } catch (error) {
      // Refresh failed, user must re-login
      logout()
      throw error
    }
  }

  // Logout: Clear local state and tell server to invalidate refresh token
  async function logout() {
    await apiClient.post('/auth/logout', {}, { withCredentials: true })
    accessToken.value = null
    user.value = null
  }

  return {
    accessToken,
    user,
    isAuthenticated,
    login,
    refreshAccessToken,
    logout
  }
})
```

### 2. Axios Interceptor for Access Token

```typescript
// services/api/client.ts
import axios from 'axios'
import { useAuthStore } from '@/stores/auth'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true // Always send cookies
})

// Request interceptor: Attach access token
apiClient.interceptors.request.use((config) => {
  const authStore = useAuthStore()

  if (authStore.accessToken) {
    config.headers.Authorization = `Bearer ${authStore.accessToken}`
  }

  return config
})

// Response interceptor: Refresh token on 401
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config

    // If 401 and haven't retried yet, try to refresh
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true

      try {
        const authStore = useAuthStore()
        await authStore.refreshAccessToken()

        // Retry original request with new token
        return apiClient(originalRequest)
      } catch (refreshError) {
        // Refresh failed, redirect to login
        window.location.href = '/login'
        return Promise.reject(refreshError)
      }
    }

    return Promise.reject(error)
  }
)
```

### 3. Backend Requirements

Your backend must:

**Login endpoint (`POST /auth/login`)**:
```javascript
// Response
res
  .cookie('refreshToken', refreshToken, {
    httpOnly: true,      // Not accessible via JavaScript
    secure: true,        // HTTPS only
    sameSite: 'strict',  // CSRF protection
    maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
  })
  .json({
    accessToken: accessToken,  // Send in body
    user: user
  })
```

**Refresh endpoint (`POST /auth/refresh`)**:
```javascript
// Read refresh token from cookie
const refreshToken = req.cookies.refreshToken

// Validate and generate new access token
const newAccessToken = validateAndGenerateAccessToken(refreshToken)

res.json({ accessToken: newAccessToken })
```

**Logout endpoint (`POST /auth/logout`)**:
```javascript
// Clear the cookie
res.clearCookie('refreshToken').json({ message: 'Logged out' })
```

### 4. Router Guard

```typescript
// router/guards.ts
import { useAuthStore } from '@/stores/auth'

export function setupAuthGuard(router: Router) {
  router.beforeEach(async (to) => {
    const authStore = useAuthStore()

    // Check if route requires auth
    if (to.meta.requiresAuth && !authStore.isAuthenticated) {
      // Try to refresh token first
      try {
        await authStore.refreshAccessToken()
        // If successful, user is authenticated
        return true
      } catch {
        // Refresh failed, redirect to login
        return { name: 'login', query: { redirect: to.fullPath } }
      }
    }

    return true
  })
}
```

```typescript
// router/index.ts
import { createRouter } from 'vue-router'
import { setupAuthGuard } from './guards'

const router = createRouter({
  routes: [
    {
      path: '/dashboard',
      component: DashboardView,
      meta: { requiresAuth: true }  // Protected route
    },
    {
      path: '/login',
      component: LoginView
    }
  ]
})

setupAuthGuard(router)

export default router
```

### 5. App Initialization (Token Refresh on Load)

```typescript
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import { useAuthStore } from './stores/auth'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.use(router)

// Try to restore session on app load
const authStore = useAuthStore()
authStore.refreshAccessToken().catch(() => {
  // No valid refresh token, user is not logged in
  console.log('No active session')
})

app.mount('#app')
```

## Security Checklist

- [ ] Access tokens are short-lived (15-30 minutes)
- [ ] Refresh tokens are long-lived (7-30 days)
- [ ] Refresh tokens stored in HttpOnly cookies
- [ ] Cookies have `SameSite=Strict`
- [ ] Cookies have `Secure=true` in production (HTTPS only)
- [ ] Access tokens never stored in localStorage
- [ ] Token refresh implemented in axios interceptor
- [ ] Router guard checks authentication before protected routes
- [ ] Logout clears both access token and refresh cookie

## Alternative: Single HttpOnly Cookie (Simpler, Less Secure)

If CSRF protection is handled separately (CSRF tokens), you can use a single HttpOnly cookie:

```typescript
// All requests automatically include auth cookie
// Backend reads cookie to authenticate
// Simpler but requires CSRF token implementation
```

---

## Navigation

- **See Also**: `09-security/xss-prevention.md`, `09-security/supply-chain.md`
- **Up**: `00-overview.md`
