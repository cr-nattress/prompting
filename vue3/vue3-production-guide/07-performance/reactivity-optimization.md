# Reactivity Optimization

> **File Purpose**: Avoid deep reactivity pitfalls
> **Agent Use Case**: Reference when optimizing reactivity performance

## Use Shallow Reactivity for Large Objects

```typescript
import { shallowRef } from 'vue'

// Large API response
const data = shallowRef(largeObject)

// Only triggers update on .value reassignment
data.value = newLargeObject
```

## Avoid Unnecessary Watchers

```typescript
// ❌ Bad: Manual watcher
const doubled = ref(0)
watch(count, (val) => doubled.value = val * 2)

// ✅ Good: Computed
const doubled = computed(() => count.value * 2)
```

---

## Navigation
- **Up**: `00-overview.md`
