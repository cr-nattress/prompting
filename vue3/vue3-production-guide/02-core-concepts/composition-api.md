# Composition API with `<script setup>`

> **File Purpose**: Master the `<script setup>` syntax and component organization patterns
> **Prerequisites**: `01-quick-start/setup.md`
> **Agent Use Case**: Reference when writing Vue 3 components
> **Related Files**: `typescript-guide.md`, `reactivity-deep-dive.md`

## In One Sentence

`<script setup>` is the recommended syntax for using the Composition API in Single File Components, offering conciseness, performance, and superior TypeScript integration.

## Why This Matters

The Composition API with `<script setup>` provides:
- **Less boilerplate**: No need for `setup()` function or return statements
- **Better performance**: Template compiled into a render function in the same scope
- **Type safety**: Superior type inference for props and emits

## Basic Syntax

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// Reactive state
const count = ref(0)

// Computed values
const doubled = computed(() => count.value * 2)

// Methods
function increment() {
  count.value++
}
</script>

<template>
  <button @click="increment">
    Count: {{ count }}
    Doubled: {{ doubled }}
  </button>
</template>
```

## Conventional Code Organization

Follow this order for readability and consistency:

```vue
<script setup lang="ts">
// 1. Imports
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/user'
import type { User } from '@/types/models'

// 2. Props
interface Props {
  userId: string
  initialValue?: number
}

const props = withDefaults(defineProps<Props>(), {
  initialValue: 0
})

// 3. Emits
interface Emits {
  (e: 'update', value: number): void
  (e: 'save', user: User): void
}

const emit = defineEmits<Emits>()

// 4. Composables
const router = useRouter()
const userStore = useUserStore()

// 5. Reactive state
const count = ref(props.initialValue)
const isLoading = ref(false)

// 6. Computed properties
const doubled = computed(() => count.value * 2)

// 7. Watchers
watch(count, (newVal) => {
  emit('update', newVal)
})

// 8. Lifecycle hooks
onMounted(() => {
  console.log('Component mounted')
})

// 9. Methods
function increment() {
  count.value++
}

function save() {
  // Business logic
  const user = userStore.currentUser
  if (user) {
    emit('save', user)
  }
}
</script>
```

## TypeScript Props & Emits

### Props with Type Safety

```vue
<script setup lang="ts">
// Simple props
interface Props {
  title: string
  count: number
  isActive?: boolean
}

const props = defineProps<Props>()

// Props with defaults
const propsWithDefaults = withDefaults(defineProps<Props>(), {
  isActive: false
})

// Alternative: runtime props (less type-safe)
const runtimeProps = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 }
})
</script>
```

### Emits with Type Safety

```vue
<script setup lang="ts">
// Typed emits
interface Emits {
  (e: 'update:modelValue', value: string): void
  (e: 'submit', payload: { name: string; email: string }): void
  (e: 'cancel'): void
}

const emit = defineEmits<Emits>()

// Usage
function handleSubmit() {
  emit('submit', { name: 'John', email: 'john@example.com' })
}

// Alternative: runtime emits
const runtimeEmit = defineEmits(['update:modelValue', 'submit', 'cancel'])
</script>
```

## defineExpose for Template Refs

By default, `<script setup>` components are closed. To expose methods/data to parent components via template refs:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

// Expose to parent
defineExpose({
  increment,
  count
})
</script>
```

Parent usage:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import ChildComponent from './ChildComponent.vue'

const childRef = ref<InstanceType<typeof ChildComponent> | null>(null)

onMounted(() => {
  childRef.value?.increment() // Call child method
  console.log(childRef.value?.count) // Access child state
})
</script>

<template>
  <ChildComponent ref="childRef" />
</template>
```

## Slots with Types

```vue
<script setup lang="ts">
interface Slots {
  default(props: { item: string }): any
  header(): any
  footer(props: { count: number }): any
}

defineSlots<Slots>()
</script>

<template>
  <div>
    <slot name="header" />
    <slot :item="someItem" />
    <slot name="footer" :count="10" />
  </div>
</template>
```

## Component Naming

```vue
<script setup lang="ts">
// Explicitly name component for DevTools and KeepAlive
defineOptions({
  name: 'UserProfile'
})
</script>
```

## Script Setup vs Standard setup()

**Prefer `<script setup>`** for:
- All new components
- Components with TypeScript
- Better performance needs

**Use standard `setup()`** only when:
- You need programmatic access to the component instance before template renders (rare)
- Migrating from Options API incrementally

## Common Patterns

### v-model with Custom Components

```vue
<script setup lang="ts">
interface Props {
  modelValue: string
}

const props = defineProps<Props>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void
}>()

function updateValue(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <input :value="modelValue" @input="updateValue" />
</template>
```

### Async Setup

Use top-level `await` (requires Suspense in parent):

```vue
<script setup lang="ts">
const data = await fetch('/api/data').then(r => r.json())
</script>

<template>
  <div>{{ data }}</div>
</template>
```

Parent with Suspense:

```vue
<template>
  <Suspense>
    <AsyncComponent />
    <template #fallback>
      <div>Loading...</div>
    </template>
  </Suspense>
</template>
```

---

## Navigation

- **Previous**: `01-quick-start/tooling-config.md`
- **Next**: `02-core-concepts/typescript-guide.md`
- **Up**: `00-overview.md`
- **See Also**: `03-design-patterns/composables.md` for reusable logic patterns
