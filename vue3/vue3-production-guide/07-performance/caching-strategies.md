# Caching Strategies

> **File Purpose**: Implement KeepAlive, v-memo, HTTP caching
> **Agent Use Case**: Reference when optimizing performance

## KeepAlive

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

## v-memo

```vue
<template>
  <div v-for="item in items" :key="item.id" v-memo="[item.id, item.selected]">
    {{ item.name }}
  </div>
</template>
```

## HTTP Caching (nginx.conf)

```nginx
location /assets/ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

---

## Navigation
- **Up**: `00-overview.md`
