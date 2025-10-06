# Runtime Validation with Zod

> **File Purpose**: Implement type-safe runtime validation for API boundaries
> **Prerequisites**: `axios-setup.md` (will be created)
> **Agent Use Case**: Reference when validating API responses or user input
> **Related Files**: `axios-setup.md`, `02-core-concepts/typescript-guide.md`

## In One Sentence

Zod provides runtime type validation that creates a robust boundary between your application and external data sources, preventing data corruption and runtime errors.

## Why This Matters

TypeScript only validates types at compile time. If an API returns unexpected data at runtime, TypeScript cannot protect you. Zod bridges this gap by:
- **Validating actual data** at runtime
- **Inferring TypeScript types** from validation schemas (single source of truth)
- **Providing detailed error messages** when validation fails

## Basic Schema Definition

```typescript
// services/validators/userSchema.ts
import { z } from 'zod'

export const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().min(0).max(150).optional(),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.string().datetime()
})

// Infer TypeScript type from schema
export type User = z.infer<typeof UserSchema>
// Equivalent to:
// type User = {
//   id: number
//   name: string
//   email: string
//   age?: number
//   role: 'admin' | 'user' | 'guest'
//   createdAt: string
// }
```

## Validating API Responses

```typescript
// services/api/users.ts
import { apiClient } from './client'
import { UserSchema } from '../validators/userSchema'

export async function fetchUser(id: number) {
  const response = await apiClient.get(`/users/${id}`)

  // Parse and validate response data
  const result = UserSchema.safeParse(response.data)

  if (!result.success) {
    console.error('API validation error:', result.error.flatten())
    throw new Error('Invalid user data received from API')
  }

  return result.data // Type: User
}

// Alternative: Throw on error
export async function fetchUserStrict(id: number) {
  const response = await apiClient.get(`/users/${id}`)
  return UserSchema.parse(response.data) // Throws ZodError if invalid
}
```

## Advanced Schemas

### Nested Objects

```typescript
const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string().regex(/^\d{5}$/)
})

const UserWithAddressSchema = z.object({
  id: z.number(),
  name: z.string(),
  address: AddressSchema
})

type UserWithAddress = z.infer<typeof UserWithAddressSchema>
```

### Arrays

```typescript
const UsersArraySchema = z.array(UserSchema)

// Validate API response
const users = UsersArraySchema.parse(response.data)
// Type: User[]
```

### Unions and Discriminated Unions

```typescript
// Simple union
const StringOrNumberSchema = z.union([z.string(), z.number()])

// Discriminated union (for state machines, API responses)
const ApiResponseSchema = z.discriminatedUnion('status', [
  z.object({ status: z.literal('success'), data: UserSchema }),
  z.object({ status: z.literal('error'), error: z.string() })
])

type ApiResponse = z.infer<typeof ApiResponseSchema>
// { status: 'success'; data: User } | { status: 'error'; error: string }
```

### Transformations

```typescript
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  // Transform string to Date object
  createdAt: z.string().transform(str => new Date(str))
})

const user = UserSchema.parse({
  id: 1,
  name: 'John',
  createdAt: '2024-01-01T00:00:00Z'
})
// user.createdAt is a Date object
```

### Refinements (Custom Validation)

```typescript
const PasswordSchema = z
  .string()
  .min(8)
  .refine(
    (val) => /[A-Z]/.test(val),
    { message: 'Password must contain at least one uppercase letter' }
  )
  .refine(
    (val) => /[0-9]/.test(val),
    { message: 'Password must contain at least one number' }
  )
```

## Form Validation Example

```typescript
// types/forms.ts
import { z } from 'zod'

export const LoginFormSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters')
})

export type LoginForm = z.infer<typeof LoginFormSchema>
```

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { LoginFormSchema } from '@/types/forms'

const formData = ref({ email: '', password: '' })
const errors = ref<Record<string, string>>({})

function handleSubmit() {
  const result = LoginFormSchema.safeParse(formData.value)

  if (!result.success) {
    // Extract field-specific errors
    const fieldErrors = result.error.flatten().fieldErrors
    errors.value = {
      email: fieldErrors.email?.[0] || '',
      password: fieldErrors.password?.[0] || ''
    }
    return
  }

  // Validation passed
  errors.value = {}
  // result.data is typed as LoginForm
  login(result.data)
}
</script>
```

## Environment Variable Validation

```typescript
// config/env.ts
import { z } from 'zod'

const EnvSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
  VITE_APP_TITLE: z.string().min(1),
  VITE_ENABLE_ANALYTICS: z.enum(['true', 'false']).transform(v => v === 'true')
})

// Validate at startup
export const env = EnvSchema.parse(import.meta.env)

// Now env is type-safe and validated
console.log(env.VITE_API_BASE_URL) // Type: string (and guaranteed to be a valid URL)
```

## Error Handling

```typescript
import { z } from 'zod'

const result = UserSchema.safeParse(data)

if (!result.success) {
  // Detailed errors
  console.log(result.error.issues)
  // [{ path: ['email'], message: 'Invalid email', ... }]

  // Flattened field errors
  console.log(result.error.flatten().fieldErrors)
  // { email: ['Invalid email'], age: ['Number must be less than 150'] }

  // Format for UI
  const errorMessage = result.error.issues
    .map(issue => `${issue.path.join('.')}: ${issue.message}`)
    .join(', ')
}
```

## Partial Schemas

```typescript
// For update operations where all fields are optional
const UpdateUserSchema = UserSchema.partial()

// Selective partial
const OptionalEmailSchema = UserSchema.partial({ email: true })
```

## Best Practices

1. **Single source of truth**: Define schemas, infer types
2. **Validate at boundaries**: API responses, form submissions, env variables
3. **Use safeParse for expected failures**: Forms, user input
4. **Use parse for unexpected failures**: API responses (throw error to be caught by error boundary)
5. **Create reusable schema components**: `AddressSchema`, `DateSchema`, etc.

---

## Navigation

- **See Also**: `05-api-layer/axios-setup.md` for integration with API client
- **Up**: `00-overview.md`
