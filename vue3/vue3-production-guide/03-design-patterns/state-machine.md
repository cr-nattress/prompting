# State Machine Pattern

> **File Purpose**: Manage complex UI states with XState
> **Agent Use Case**: Reference for multi-step forms, wizards, complex workflows

## XState Integration

```typescript
import { createMachine, interpret } from 'xstate'

const formMachine = createMachine({
  id: 'form',
  initial: 'editing',
  states: {
    editing: { on: { SUBMIT: 'validating' } },
    validating: {
      on: {
        VALID: 'submitting',
        INVALID: 'editing'
      }
    },
    submitting: {
      on: {
        SUCCESS: 'success',
        ERROR: 'error'
      }
    },
    success: {},
    error: { on: { RETRY: 'submitting' } }
  }
})

// Usage in composable
export function useFormStateMachine() {
  const service = interpret(formMachine)
  service.start()
  return service
}
```

See: https://stately.ai/docs/xstate-vue

---

## Navigation
- **Up**: `00-overview.md`
