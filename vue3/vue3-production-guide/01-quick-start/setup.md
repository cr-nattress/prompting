# Project Setup & Scaffolding

> **File Purpose**: Step-by-step guide to scaffold a production-ready Vue 3 + TypeScript project
> **Prerequisites**: Node.js 18+, npm or pnpm
> **Agent Use Case**: Reference when initializing a new Vue 3 project
> **Related Files**: `folder-structure.md`, `tooling-config.md`

## In One Sentence

Use `create-vue` to scaffold a Vue 3 + TypeScript project with all production essentials (router, state, testing, linting) pre-configured.

## Why This Matters

Starting with the right foundation prevents technical debt and ensures consistency. The official `create-vue` tool provides battle-tested defaults that align with Vue core team recommendations.

## Scaffolding the Project

### Interactive Method

Run the following command and select the options shown:

```bash
npm create vue@latest
```

**Recommended selections for production projects:**
- **Project name**: `your-project-name`
- **TypeScript**: `Yes` ✓
- **JSX Support**: `No` (unless specifically required)
- **Vue Router**: `Yes` ✓
- **Pinia**: `Yes` ✓
- **Vitest**: `Yes` ✓
- **Playwright**: `Yes` ✓
- **ESLint**: `Yes` ✓
- **Prettier**: `Yes` ✓

### Non-Interactive Method

For CI/CD or scripted setup:

```bash
npm create vue@latest my-vue-app -- --typescript --router --pinia --vitest --playwright --eslint --prettier
cd my-vue-app
npm install
```

## Additional Production Dependencies

After scaffolding, install these essential libraries:

```bash
# API client and validation
npm install axios zod

# Server state management
npm install @tanstack/vue-query

# Security
npm install dompurify
npm install -D @types/dompurify

# Security linting
npm install -D eslint-plugin-security

# Advanced form handling (optional)
npm install vee-validate
```

### Dependency Justifications

- **axios**: Industry-standard HTTP client with interceptor support
- **zod**: Runtime type validation for API boundaries
- **@tanstack/vue-query**: Purpose-built for server state (caching, refetching, optimistic updates)
- **dompurify**: Sanitizes HTML to prevent XSS attacks
- **eslint-plugin-security**: Catches security vulnerabilities during development
- **vee-validate**: Comprehensive form state management

## Initial Project Structure

After running `create-vue`, your project will have this structure:

```
my-vue-app/
├── .vscode/              # VSCode editor settings
├── public/               # Static assets (not processed by Vite)
├── src/
│   ├── assets/          # Assets processed by Vite (images, styles)
│   ├── components/      # Reusable components
│   ├── router/          # Vue Router configuration
│   ├── stores/          # Pinia stores
│   ├── views/           # Route-level components
│   ├── App.vue          # Root component
│   └── main.ts          # Application entry point
├── .gitignore
├── env.d.ts             # TypeScript declarations for Vite
├── index.html           # HTML entry point
├── package.json
├── tsconfig.json        # TypeScript base config
├── tsconfig.app.json    # App-specific TypeScript config
├── tsconfig.node.json   # Node-specific TypeScript config
└── vite.config.ts       # Vite build configuration
```

## Verify Installation

Run these commands to ensure everything is working:

```bash
# Start dev server
npm run dev

# Run unit tests
npm run test:unit

# Run E2E tests (installs browsers on first run)
npm run test:e2e

# Lint code
npm run lint

# Format code
npm run format
```

## Next Steps

1. **Configure TypeScript strictness**: See `01-quick-start/tooling-config.md` for `tsconfig.json` settings
2. **Organize folder structure**: See `01-quick-start/folder-structure.md` for production-grade organization
3. **Set up tooling**: See `01-quick-start/tooling-config.md` for ESLint, Prettier, and Vite configuration

## Common Issues

### Issue: "Cannot find module '@vue/runtime-core'"

**Cause**: TypeScript types not properly installed
**Solution**:
```bash
npm install --save-dev @vue/tsconfig
```

### Issue: Playwright browser download fails

**Cause**: Network restrictions or missing dependencies
**Solution**:
```bash
npx playwright install --with-deps chromium
```

### Issue: ESLint errors on .vue files

**Cause**: Missing Vue plugin for ESLint
**Solution**: Already included by `create-vue`, but verify `eslint-plugin-vue` is in `package.json`

---

## Navigation

- **Next**: `01-quick-start/folder-structure.md` - Organize your project
- **Also See**: `02-core-concepts/typescript-guide.md` for TypeScript configuration details
- **Up**: `00-overview.md`
