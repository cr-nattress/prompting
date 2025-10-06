# Production-Grade Folder Structure

> **File Purpose**: Define a scalable, feature-centric directory organization
> **Prerequisites**: `setup.md` - Project scaffolded
> **Agent Use Case**: Reference when organizing files in a Vue 3 project
> **Related Files**: `setup.md`, `02-core-concepts/composition-api.md`

## In One Sentence

Organize your project by feature and concern (components, stores, services) to scale from small to large applications while maintaining clarity.

## Why This Matters

A well-organized folder structure is essential for:
- **Discoverability**: New team members can find code quickly
- **Scalability**: Structure doesn't collapse as the project grows
- **Separation of concerns**: Business logic, UI, and data are clearly separated

## Recommended Structure

```
src/
├── assets/                    # Static assets processed by Vite
│   ├── images/
│   ├── fonts/
│   └── styles/
│       ├── main.css
│       └── variables.css
├── components/                # Reusable UI components
│   ├── common/               # Generic components (Button, Modal, etc.)
│   │   ├── BaseButton.vue
│   │   ├── BaseModal.vue
│   │   └── BaseInput.vue
│   ├── layout/               # Layout components (Header, Footer, Sidebar)
│   │   ├── AppHeader.vue
│   │   ├── AppFooter.vue
│   │   └── AppSidebar.vue
│   └── features/             # Feature-specific components
│       ├── user/
│       │   ├── UserProfile.vue
│       │   └── UserAvatar.vue
│       └── product/
│           ├── ProductCard.vue
│           └── ProductList.vue
├── composables/              # Reusable composition functions
│   ├── useAuth.ts
│   ├── useLocalStorage.ts
│   └── useFetch.ts
├── stores/                   # Pinia global state stores
│   ├── auth.ts
│   ├── cart.ts
│   └── settings.ts
├── services/                 # Business logic and API communication
│   ├── api/
│   │   ├── client.ts        # Axios instance
│   │   ├── users.ts         # User-related API calls
│   │   └── products.ts      # Product-related API calls
│   └── validators/
│       ├── userSchema.ts    # Zod schemas for users
│       └── productSchema.ts # Zod schemas for products
├── router/                   # Vue Router configuration
│   ├── index.ts             # Main router setup
│   └── guards.ts            # Navigation guards
├── views/                    # Route-level components (pages)
│   ├── HomeView.vue
│   ├── LoginView.vue
│   ├── DashboardView.vue
│   └── NotFoundView.vue
├── types/                    # TypeScript type definitions
│   ├── models.ts            # Data models (User, Product, etc.)
│   ├── api.ts               # API request/response types
│   └── env.d.ts             # Environment variable types
├── utils/                    # Pure utility functions
│   ├── formatters.ts
│   ├── validators.ts
│   └── constants.ts
├── App.vue                   # Root component
└── main.ts                   # Application entry point
```

## Directory Explanations

### `/assets`
**Purpose**: Static files that Vite will process (optimize, bundle)
**Contents**: Images, fonts, global CSS
**Best Practice**: Use for files that need optimization. Put truly static files (e.g., `robots.txt`) in `/public` instead.

### `/components`
**Purpose**: Reusable UI components
**Organization**:
- `common/`: Generic, project-agnostic components (prefix with `Base`)
- `layout/`: Application shell components (prefix with `App`)
- `features/`: Domain-specific components grouped by feature

**Naming Convention**: PascalCase, descriptive names (e.g., `UserProfileCard.vue`, not `Card.vue`)

### `/composables`
**Purpose**: Encapsulated, reusable stateful logic
**Naming Convention**: `use` prefix (e.g., `useAuth.ts`, `useMouse.ts`)
**Example**: Form state, API fetching logic, browser API wrappers

### `/stores`
**Purpose**: Global, shared application state
**Organization**: One file per logical domain (e.g., `auth.ts`, `cart.ts`)
**Best Practice**: Keep stores focused. If a store exceeds 200 lines, consider splitting it.

### `/services`
**Purpose**: Business logic and external communication
**Organization**:
- `/api`: API clients and request functions
- `/validators`: Zod schemas for runtime validation

**Best Practice**: Services should be pure functions or classes. No Vue-specific code.

### `/router`
**Purpose**: Application routing configuration
**Organization**:
- `index.ts`: Route definitions
- `guards.ts`: Authentication and authorization logic

### `/views`
**Purpose**: Top-level components mapped to routes
**Naming Convention**: Suffix with `View` (e.g., `DashboardView.vue`)
**Best Practice**: Views should orchestrate components and composables. Keep business logic in services/composables.

### `/types`
**Purpose**: Shared TypeScript type definitions
**Organization**:
- `models.ts`: Business domain types (User, Product, Order)
- `api.ts`: API-specific types (request payloads, response shapes)
- `env.d.ts`: Environment variable types

### `/utils`
**Purpose**: Pure utility functions (no side effects)
**Example**: Date formatters, string manipulators, constants

## Alternative: Feature-Based Organization

For very large applications, consider grouping by feature instead:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── composables/
│   │   ├── stores/
│   │   ├── services/
│   │   ├── views/
│   │   └── types/
│   ├── products/
│   │   ├── components/
│   │   ├── composables/
│   │   └── ...
│   └── orders/
│       └── ...
└── shared/                   # Shared across features
    ├── components/
    ├── composables/
    └── utils/
```

**When to use**: Projects with 10+ distinct features and multiple teams

## File Naming Conventions

- **Components**: PascalCase (e.g., `UserProfile.vue`)
- **Composables**: camelCase with `use` prefix (e.g., `useAuth.ts`)
- **Stores**: camelCase (e.g., `userStore.ts` or just `user.ts`)
- **Utils/Services**: camelCase (e.g., `apiClient.ts`)
- **Types**: camelCase (e.g., `models.ts`)

## Import Aliases

Configure path aliases in `vite.config.ts` for cleaner imports:

```typescript
// Instead of: import Button from '../../../components/common/Button.vue'
// Use: import Button from '@/components/common/Button.vue'
```

See `tooling-config.md` for Vite configuration details.

---

## Navigation

- **Previous**: `01-quick-start/setup.md`
- **Next**: `01-quick-start/tooling-config.md`
- **Up**: `00-overview.md`
