# Repository/Adapter Pattern

> **File Purpose**: Abstract data sources for testability and flexibility
> **Agent Use Case**: Reference when building API abstraction layers

## Pattern

```typescript
// Repository interface
interface UserRepository {
  getUsers(): Promise<User[]>
  getUser(id: number): Promise<User>
  createUser(user: CreateUserDto): Promise<User>
}

// API implementation
class ApiUserRepository implements UserRepository {
  async getUsers() {
    const response = await apiClient.get<User[]>('/users')
    return UserSchema.array().parse(response.data)
  }
  // ...
}

// Mock implementation for testing
class MockUserRepository implements UserRepository {
  async getUsers() {
    return [{ id: 1, name: 'Test User' }]
  }
  // ...
}

// Usage
const userRepo: UserRepository = import.meta.env.PROD
  ? new ApiUserRepository()
  : new MockUserRepository()
```

**Benefits**: Easy to test, swap implementations, support multiple backends.

---

## Navigation
- **Up**: `00-overview.md`
