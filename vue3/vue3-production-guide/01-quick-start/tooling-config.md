# Tooling Configuration

> **File Purpose**: Configure TypeScript, ESLint, Prettier, and Vite for production-grade development
> **Prerequisites**: `setup.md` - Project scaffolded
> **Agent Use Case**: Reference when setting up or troubleshooting tooling configurations
> **Related Files**: `02-core-concepts/typescript-guide.md`

## In One Sentence

Strict TypeScript settings, automated linting/formatting with ESLint/Prettier, and Vite path aliases create a robust development environment that catches errors early.

## TypeScript Configuration

### Strict tsconfig.json

Create or update `tsconfig.json` in your project root:

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.node.json" },
    { "path": "./tsconfig.app.json" }
  ]
}
```

### Application TypeScript Config (tsconfig.app.json)

This is where strict settings are enforced for your application code:

```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
  "exclude": ["src/**/__tests__/*"],
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },

    // Strict type-checking options
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": false,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    // Module resolution
    "verbatimModuleSyntax": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,

    // Other options
    "skipLibCheck": true,
    "allowImportingTsExtensions": true
  }
}
```

### Key Configuration Justifications

- **`"strict": true`**: Enables all strict type-checking options. Single most important setting.
- **`"noUncheckedIndexedAccess": true`**: Adds `| undefined` to indexed access, forcing null checks.
- **`"exactOptionalPropertyTypes": true`**: Distinguishes between missing properties and `undefined` values.
- **`"verbatimModuleSyntax": true`**: Critical for Vite/esbuild compatibility.
- **`"paths": { "@/*": ["./src/*"] }`**: Enables `@/` import alias for clean imports.

## Vite Configuration

Update `vite.config.ts`:

```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
  server: {
    port: 3000
  },
  preview: {
    port: 8080
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['vue', 'vue-router', 'pinia'],
          'ui': ['@headlessui/vue'] // Example UI library
        }
      }
    }
  }
})
```

### Key Configuration Justifications

- **`alias: { '@': ... }`**: Mirrors TypeScript path mapping for consistency.
- **`server.port`**: Explicit port prevents conflicts.
- **`build.sourcemap: true`**: Essential for production debugging with error trackers (Sentry, etc.).
- **`manualChunks`**: Splits vendor code for better caching.

## ESLint Configuration

Update `.eslintrc.cjs` to include security plugin:

```javascript
/* eslint-env node */
require('@rushstack/eslint-patch/modern-module-resolution')

module.exports = {
  root: true,
  extends: [
    'plugin:vue/vue3-essential',
    'eslint:recommended',
    '@vue/eslint-config-typescript',
    '@vue/eslint-config-prettier/skip-formatting',
    'plugin:security/recommended'
  ],
  parserOptions: {
    ecmaVersion: 'latest'
  },
  rules: {
    'vue/multi-word-component-names': 'warn',
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off'
  }
}
```

### Install Security Plugin

```bash
npm install -D eslint-plugin-security
```

### Key Configuration Justifications

- **`plugin:security/recommended`**: Detects security vulnerabilities (e.g., `eval()`, insecure regex).
- **`@vue/eslint-config-prettier/skip-formatting`**: Modern integration preventing ESLint/Prettier conflicts.
- **`no-console`: 'warn' in production**: Reminds you to remove debug logs.

## Prettier Configuration

Create `.prettierrc.json` in your project root:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "none",
  "arrowParens": "avoid",
  "printWidth": 100
}
```

### Opinionated Settings

- **`semi: false`**: Vue community preference
- **`singleQuote: true`**: Consistency with Vue style guide
- **`printWidth: 100`**: Balance between readability and line count

Create `.prettierignore`:

```
dist
node_modules
.husky
```

## Package.json Scripts

Add these scripts to `package.json` for common tooling tasks:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "test:unit": "vitest",
    "test:e2e": "playwright test",
    "lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs,.ts,.tsx,.cts,.mts --fix --ignore-path .gitignore",
    "format": "prettier --write src/"
  }
}
```

## Environment Variables

### Type-safe Environment Variables

Create `src/types/env.d.ts`:

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
  // Add more env variables here
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

### Usage

Create `.env` files:

```bash
# .env (committed to git, defaults)
VITE_API_BASE_URL=http://localhost:3001
VITE_APP_TITLE=My App

# .env.local (not committed, local overrides)
VITE_API_BASE_URL=http://localhost:8080
```

Access in code:

```typescript
const apiUrl = import.meta.env.VITE_API_BASE_URL // Type-safe!
```

## VSCode Settings

Create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "[vue]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

Create `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "vue.volar",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode"
  ]
}
```

## Verification Checklist

After configuration, verify:

```bash
# TypeScript compilation
npm run build

# Linting
npm run lint

# Formatting
npm run format

# Tests
npm run test:unit
```

All commands should complete without errors.

---

## Navigation

- **Previous**: `01-quick-start/folder-structure.md`
- **Next**: `02-core-concepts/composition-api.md`
- **Up**: `00-overview.md`
- **See Also**: `02-core-concepts/typescript-guide.md` for advanced TypeScript patterns
