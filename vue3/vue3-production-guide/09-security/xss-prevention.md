# XSS Prevention & Content Security Policy

> **File Purpose**: Prevent Cross-Site Scripting attacks with CSP and DOMPurify
> **Agent Use Case**: Reference when rendering user-generated content or configuring CSP
> **Related Files**: `authentication.md`

## In One Sentence

Never render untrusted HTML without sanitization (use DOMPurify); implement a strict Content Security Policy to prevent script injection.

## Sanitizing User HTML with DOMPurify

```typescript
// utils/sanitize.ts
import DOMPurify from 'dompurify'

export function sanitizeHtml(dirty: string): string {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
    ALLOWED_ATTR: ['href']
  })
}
```

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { sanitizeHtml } from '@/utils/sanitize'

const props = defineProps<{ userContent: string }>()

const safeHtml = computed(() => sanitizeHtml(props.userContent))
</script>

<template>
  <!-- Safe: Sanitized HTML -->
  <div v-html="safeHtml" />
</template>
```

## Content Security Policy

### Development CSP (index.html)

```html
<meta
  http-equiv="Content-Security-Policy"
  content="
    default-src 'self';
    script-src 'self' 'unsafe-eval';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self' data:;
    connect-src 'self' http://localhost:*;
  "
/>
```

### Production CSP (via HTTP header)

Configure your server (Nginx, Cloudflare, etc.) to send:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' https:; font-src 'self'; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
```

### CSP Directives Explained

- `default-src 'self'`: Only load resources from same origin by default
- `script-src 'self'`: Only execute scripts from same origin (no inline `<script>`)
- `style-src 'self' 'unsafe-inline'`: Styles from same origin + inline (required for Vue)
- `connect-src 'self' https://api.example.com`: Only fetch from listed origins
- `frame-ancestors 'none'`: Prevent clickjacking

## Common XSS Vulnerabilities

### Vulnerability 1: v-html with user input

```vue
<!-- ❌ NEVER DO THIS -->
<div v-html="userComment" />

<!-- ✅ DO THIS -->
<div v-html="sanitizeHtml(userComment)" />

<!-- ✅ OR AVOID v-html ENTIRELY -->
<div>{{ userComment }}</div>  <!-- Auto-escaped -->
```

### Vulnerability 2: Dynamic attributes

```vue
<!-- ❌ Dangerous -->
<a :href="userProvidedUrl">Link</a>

<!-- ✅ Validate URL protocol -->
<a :href="isSafeUrl(userProvidedUrl) ? userProvidedUrl : '#'">Link</a>
```

```typescript
function isSafeUrl(url: string): boolean {
  try {
    const parsed = new URL(url)
    return ['http:', 'https:', 'mailto:'].includes(parsed.protocol)
  } catch {
    return false
  }
}
```

### Vulnerability 3: innerHTML in script

```typescript
// ❌ NEVER
element.innerHTML = userInput

// ✅ Use textContent or sanitize
element.textContent = userInput
// OR
element.innerHTML = DOMPurify.sanitize(userInput)
```

---

## Navigation

- **See Also**: `09-security/authentication.md`, `09-security/supply-chain.md`
- **Up**: `00-overview.md`
