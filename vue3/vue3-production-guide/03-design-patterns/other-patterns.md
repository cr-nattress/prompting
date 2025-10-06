# Additional Design Patterns

> **File Purpose**: Strategy, Facade, and Observer patterns for Vue 3
> **Prerequisites**: `composables.md`
> **Agent Use Case**: Reference when implementing specific design patterns

## Strategy Pattern

Select algorithm at runtime:

```typescript
// Rendering strategies
type RenderStrategy = 'table' | 'grid' | 'list'

interface DataRenderer<T> {
  render(data: T[]): VNode
}

const strategies: Record<RenderStrategy, DataRenderer<any>> = {
  table: new TableRenderer(),
  grid: new GridRenderer(),
  list: new ListRenderer()
}

// Usage in component
const currentStrategy = ref<RenderStrategy>('table')
const renderer = computed(() => strategies[currentStrategy.value])
```

## Facade Pattern

Simplified interface to complex subsystem:

```typescript
// composables/useNotifications.ts
export function useNotifications() {
  function showSuccess(message: string) {
    // Internally uses toast, console, analytics
    toast.success(message)
    console.log(message)
    analytics.track('notification_shown', { type: 'success' })
  }

  function showError(error: Error) {
    toast.error(error.message)
    console.error(error)
    analytics.track('error_shown')
  }

  return { showSuccess, showError }
}
```

## Observer / Event Bus Pattern

Decoupled event communication (use sparingly):

```typescript
// composables/useEventBus.ts
import { useEventBus as vueUseEventBus } from '@vueuse/core'

type Events = {
  'user:updated': User
  'cart:item-added': Product
}

export function useEventBus<T extends keyof Events>(event: T) {
  return vueUseEventBus<Events[T]>(event)
}

// Usage
const bus = useEventBus('user:updated')
bus.on(user => console.log('User updated:', user))
bus.emit(updatedUser)
```

**Note**: Prefer Pinia for most cross-component communication.

---

## Navigation

- **Up**: `00-overview.md`
