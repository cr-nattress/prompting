# Vite Production Build

> **File Purpose**: Optimize Vite build for production
> **Agent Use Case**: Reference when configuring production builds

## Configuration

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    sourcemap: true,  // For error tracking
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          utils: ['axios', 'zod']
        }
      }
    },
    chunkSizeWarningLimit: 1000
  }
})
```

## Build Command

```bash
npm run build

# Output in dist/ directory
# Upload sourcemaps to error tracker (Sentry, etc.)
```

---

## Navigation
- **Up**: `00-overview.md`
