# Internationalization (i18n)

> **File Purpose**: Setup vue-i18n for multi-language support
> **Agent Use Case**: Reference when adding i18n

## Setup

```typescript
// i18n/index.ts
import { createI18n } from 'vue-i18n'
import en from './locales/en.json'
import fr from './locales/fr.json'

export const i18n = createI18n({
  locale: 'en',
  fallbackLocale: 'en',
  messages: { en, fr }
})
```

```vue
<template>
  <p>{{ $t('hello') }}</p>
  <p>{{ $t('welcome', { name: 'John' }) }}</p>
</template>
```

---

## Navigation
- **Up**: `00-overview.md`
