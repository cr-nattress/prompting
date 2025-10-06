# Reactivity Deep Dive

> **File Purpose**: Master Vue 3's reactivity system: ref vs reactive, shallow reactivity, watchers
> **Prerequisites**: `composition-api.md`
> **Agent Use Case**: Reference when choosing reactivity primitives or debugging reactivity issues
> **Related Files**: `typescript-guide.md`, `07-performance/reactivity-optimization.md`

## In One Sentence

Prefer `ref` for all reactive state to avoid reactivity loss pitfalls; use `shallowRef`/`shallowReactive` for large objects; choose `watch` for specific dependencies, `watchEffect` for automatic tracking.

## ref vs reactive: The Definitive Guide

### Opinionated Best Practice: Prefer ref

**Use `ref` for everything**, including objects:

```typescript
import { ref } from 'vue'

// Primitives (only option)
const count = ref(0)
const message = ref('hello')

// Objects (recommended)
const user = ref({ name: 'John', age: 30 })

// Access
count.value++
user.value.name = 'Jane'
user.value = { name: 'Bob', age: 25 } // Reassignment works ✓
```

**Avoid `reactive` unless you have a specific reason:**

```typescript
import { reactive } from 'vue'

const user = reactive({ name: 'John', age: 30 })

// Access (no .value)
user.name = 'Jane' // Looks cleaner

// BUT: Pitfalls
user = { name: 'Bob', age: 25 } // ❌ Breaks reactivity!

// Destructuring breaks reactivity
const { name } = user
// 'name' is now a plain string, not reactive ❌
```

### Why ref is Safer

| Scenario | ref | reactive |
|----------|-----|----------|
| Reassignment | ✅ Works | ❌ Breaks reactivity |
| Destructuring | ✅ Stays reactive (with `toRefs`) | ❌ Loses reactivity |
| All types | ✅ Primitives & objects | ❌ Objects only |
| Template unwrapping | ✅ Automatic | ✅ Automatic |
| Consistency | ✅ Always use `.value` | ⚠️ Sometimes needed |

### When to Use reactive

Only use `reactive` when:
1. You have a large, stable object that will never be reassigned
2. You never destructure it
3. You want to avoid `.value` for ergonomics (rare)

```typescript
// Acceptable use case: app config
const config = reactive({
  theme: 'dark',
  language: 'en',
  apiUrl: 'https://api.example.com'
})
// Never reassigned, never destructured
```

## Shallow Reactivity for Performance

### shallowRef

Only `.value` assignment is tracked; nested properties are not reactive:

```typescript
import { shallowRef, triggerRef } from 'vue'

const state = shallowRef({
  count: 0,
  nested: { value: 1 }
})

// This does NOT trigger updates ❌
state.value.count++
state.value.nested.value++

// This DOES trigger update ✓
state.value = { count: 1, nested: { value: 2 } }

// Or manually trigger
state.value.count++
triggerRef(state) // Force update
```

**Use cases**:
- Large, immutable data structures (e.g., API responses)
- Integration with external libraries (e.g., D3.js, Chart.js)
- Performance optimization for deeply nested objects

### shallowReactive

Only root-level properties are reactive:

```typescript
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  nested: { value: 1 }
})

state.count++ // ✓ Reactive
state.nested.value++ // ❌ Not reactive
state.nested = { value: 2 } // ✓ Reactive (root-level change)
```

## Watchers: watch vs watchEffect

### watch

Explicit dependencies, lazy by default:

```typescript
import { ref, watch } from 'vue'

const count = ref(0)
const message = ref('hello')

// Watch single source
watch(count, (newVal, oldVal) => {
  console.log(`Count changed from ${oldVal} to ${newVal}`)
})

// Watch multiple sources
watch([count, message], ([newCount, newMessage], [oldCount, oldMessage]) => {
  console.log('Either changed')
})

// Watch object property
const user = ref({ name: 'John' })
watch(
  () => user.value.name,
  (newName) => {
    console.log(`Name changed to ${newName}`)
  }
)

// Immediate execution
watch(count, (val) => {
  console.log(val)
}, { immediate: true })

// Deep watching
const state = ref({ nested: { count: 0 } })
watch(state, () => {
  console.log('Deep change detected')
}, { deep: true })
```

### watchEffect

Automatic dependency tracking, runs immediately:

```typescript
import { ref, watchEffect } from 'vue'

const count = ref(0)
const doubled = ref(0)

// Runs immediately and whenever dependencies change
watchEffect(() => {
  doubled.value = count.value * 2
  console.log(`Count: ${count.value}, Doubled: ${doubled.value}`)
})
// Logs immediately, then on every count change
```

### Choosing Between watch and watchEffect

**Use `watch` when you need:**
- Previous value access
- Lazy execution (don't run on mount)
- Explicit control over dependencies
- Different logic for different sources

**Use `watchEffect` when you need:**
- Automatic dependency tracking
- Immediate execution
- Side effects based on multiple reactive sources
- Simpler syntax for complex dependencies

```typescript
// watch: Explicit, previous value
watch(userId, async (newId, oldId) => {
  console.log(`User changed from ${oldId} to ${newId}`)
  await fetchUser(newId)
})

// watchEffect: Automatic, immediate
watchEffect(async () => {
  // Automatically tracks userId, userType, etc.
  if (userId.value && userType.value) {
    await fetchUserData(userId.value, userType.value)
  }
})
```

## computed for Derived State

Always prefer `computed` over manually updating reactive values:

```typescript
import { ref, computed } from 'vue'

const count = ref(0)

// Good: Computed (cached, automatic)
const doubled = computed(() => count.value * 2)

// Bad: Manual watch (unnecessary, error-prone)
const doubled = ref(0)
watch(count, (val) => {
  doubled.value = val * 2
})
```

### Writable computed

```typescript
const firstName = ref('John')
const lastName = ref('Doe')

const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(value) {
    const parts = value.split(' ')
    firstName.value = parts[0]
    lastName.value = parts[1]
  }
})

fullName.value = 'Jane Smith' // Updates firstName and lastName
```

## Common Reactivity Pitfalls

### 1. Destructuring reactive objects

```typescript
// ❌ Loses reactivity
const state = reactive({ count: 0 })
const { count } = state // count is now a plain number

// ✅ Use toRefs
import { toRefs } from 'vue'
const { count } = toRefs(state) // count is now Ref<number>
```

### 2. Reassigning reactive

```typescript
// ❌ Breaks reactivity
let state = reactive({ count: 0 })
state = reactive({ count: 1 }) // Doesn't update watchers

// ✅ Use ref
const state = ref({ count: 0 })
state.value = { count: 1 } // Works correctly
```

### 3. Array mutation reactivity

```typescript
const items = ref([1, 2, 3])

// ✅ All of these are reactive
items.value.push(4)
items.value.pop()
items.value[0] = 10
items.value = items.value.filter(i => i > 1)
```

---

## Navigation

- **Previous**: `02-core-concepts/typescript-guide.md`
- **Next**: `03-design-patterns/composables.md`
- **Up**: `00-overview.md`
- **See Also**: `07-performance/reactivity-optimization.md` for performance considerations
