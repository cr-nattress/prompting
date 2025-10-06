# Axios Setup & Interceptors

> **File Purpose**: Configure centralized API client with interceptors
> **Agent Use Case**: Reference when setting up API communication

## API Client

```typescript
// services/api/client.ts
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// Request interceptor: Add auth token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Response interceptor: Handle errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

## Usage

```typescript
// services/api/users.ts
import { apiClient } from './client'
import { UserSchema } from '../validators/userSchema'

export async function fetchUsers() {
  const response = await apiClient.get('/users')
  return UserSchema.array().parse(response.data)
}
```

---

## Navigation
- **Next**: `05-api-layer/zod-validation.md`
- **Up**: `00-overview.md`
