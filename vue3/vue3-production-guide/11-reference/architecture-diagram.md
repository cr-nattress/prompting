# Architecture Diagram

> **File Purpose**: Visual overview of system architecture and data flow
> **Agent Use Case**: Reference when understanding or explaining system design

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER BROWSER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │    Route     │  Rendered based on Vue Router                │
│  │   (Page/View)│                                               │
│  └──────┬───────┘                                               │
│         │                                                       │
│         │ Uses                                                  │
│         ▼                                                       │
│  ┌──────────────────────┐                                      │
│  │   Components         │  Reusable UI elements                │
│  │   (Presentational)   │                                       │
│  └──────┬───────────────┘                                       │
│         │                                                       │
│         │ Consumes state/methods from                          │
│         ▼                                                       │
│  ┌──────────────────────┬──────────────────────┐               │
│  │   Composables        │   Pinia Stores       │               │
│  │   (Local State)      │   (Global State)     │               │
│  └──────┬───────────────┴──────────┬───────────┘               │
│         │                          │                           │
│         │ Delegates business logic │                           │
│         ▼                          ▼                           │
│  ┌─────────────────────────────────────────┐                   │
│  │         Services Layer                  │                   │
│  │  (Business logic, data transformation)  │                   │
│  └──────────────────┬──────────────────────┘                   │
│                     │                                          │
│                     │ Uses                                     │
│                     ▼                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │         API Client (Axios)              │                   │
│  │  - Request/Response Interceptors        │                   │
│  │  - Auth token injection                 │                   │
│  └──────────────────┬──────────────────────┘                   │
│                     │                                          │
│                     │ Validates with                           │
│                     ▼                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │         Zod Schemas                     │                   │
│  │  (Runtime validation of API responses)  │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                        │
                        │ HTTP Requests
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BACKEND API                                │
│                   (REST/GraphQL)                                │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow Example: User Login

```
1. User enters credentials in LoginView
   ↓
2. LoginView calls authStore.login(email, password)
   ↓
3. Auth Store delegates to API service
   ↓
4. API Service uses axios client to POST /auth/login
   ↓
5. Axios interceptor adds headers
   ↓
6. Backend validates and returns { accessToken, user }
   ↓
7. Zod schema validates response structure
   ↓
8. Auth Store updates state (token, user)
   ↓
9. Router guard detects authentication, redirects to /dashboard
   ↓
10. DashboardView renders with authenticated state
```

## State Management Flow

```
┌────────────────────────────────────────────┐
│        Component (e.g., UserProfile)       │
└────────┬───────────────────┬───────────────┘
         │                   │
    Local state          Global state
         │                   │
         ▼                   ▼
┌─────────────────┐   ┌──────────────┐
│  Composables    │   │ Pinia Stores │
│  (useFetch,     │   │ (useAuth,    │
│   useForm)      │   │  useCart)    │
└─────────────────┘   └──────────────┘
         │                   │
         └───────┬───────────┘
                 │
                 ▼
         ┌───────────────┐
         │   Services    │
         │   (API calls) │
         └───────────────┘
```

## Testing Layers

```
┌──────────────────────────────────────┐
│           E2E Tests                  │  Full user flows
│        (Playwright)                  │  (5-10 critical paths)
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│       Component Tests                │  UI contract testing
│    (Vitest + Test Utils)             │  (All public APIs)
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│          Unit Tests                  │  Business logic
│         (Vitest)                     │  (80%+ coverage)
└──────────────────────────────────────┘
```

---

## Navigation

- **See Also**: `11-reference/full-file-tree.md`, `11-reference/resources.md`
- **Up**: `00-overview.md`
