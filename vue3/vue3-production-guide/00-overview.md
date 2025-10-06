# Production-Grade Vue 3: AI Agent Reference Guide

> **Last Updated**: 2024-10-27
> **Original Source**: Vue 3 Production Architecture Guide for Senior Engineers
> **Optimization Version**: 1.0

## What This Is

A comprehensive, opinionated methodology for building production-grade applications using Vue 3 and TypeScript. This guide is designed for senior engineers and technical leads responsible for establishing architectural standards and leading development teams. It covers everything from initial project setup through deployment, with runnable code examples and deep analysis of architectural trade-offs.

Key focus areas: modular architecture, type safety, security hardening, testing strategies, and automated deployment pipelines.

## File Structure Guide

This guide has been organized into 11 directories that mirror the typical development workflow:

### 01-quick-start/
**Purpose**: Get your project scaffolded and configured correctly from day one
- `setup.md` - Project scaffolding with create-vue, dependencies, tsconfig
- `folder-structure.md` - Production-grade directory organization
- `tooling-config.md` - ESLint, Prettier, and Vite configuration

### 02-core-concepts/
**Purpose**: Master Vue 3 fundamentals and TypeScript best practices
- `composition-api.md` - `<script setup>` syntax, component organization patterns
- `typescript-guide.md` - Strict TypeScript configuration, advanced types
- `reactivity-deep-dive.md` - ref vs reactive, shallow reactivity, watchers

### 03-design-patterns/
**Purpose**: Apply proven architectural patterns for scalable applications
- `composables.md` - The "Hook" pattern for reusable stateful logic
- `container-presentational.md` - Separation of concerns in components
- `repository-adapter.md` - Abstracting data sources
- `state-machine.md` - Managing complex UI states with XState
- `other-patterns.md` - Strategy, Facade, and Observer patterns

### 04-state-data/
**Purpose**: Implement effective state management strategies
- `state-strategy.md` - Decision framework: local vs global state
- `pinia-stores.md` - Fully-typed global state with Pinia
- `tanstack-query.md` - Server state management with Vue Query

### 05-api-layer/
**Purpose**: Build a type-safe, resilient API communication layer
- `axios-setup.md` - Centralized Axios instance with interceptors
- `zod-validation.md` - Runtime validation with Zod schemas

### 06-ui-ux/
**Purpose**: Create accessible, internationalized user experiences
- `routing.md` - Vue Router 4, lazy loading, authentication guards
- `accessibility.md` - ARIA, semantic HTML, focus management
- `internationalization.md` - vue-i18n with typed messages
- `forms-validation.md` - Vee-Validate + Zod integration

### 07-performance/
**Purpose**: Optimize for speed and efficiency
- `code-splitting.md` - Route and component-level lazy loading
- `caching-strategies.md` - KeepAlive, v-memo, HTTP caching
- `reactivity-optimization.md` - Avoiding deep reactivity pitfalls

### 08-testing/
**Purpose**: Implement comprehensive quality assurance
- `testing-strategy.md` - Multi-layered testing approach overview
- `unit-testing.md` - Vitest for stores and composables
- `component-testing.md` - Vue Test Utils best practices
- `e2e-testing.md` - Playwright for critical user flows
- `quality-gates.md` - CI integration and coverage enforcement

### 09-security/
**Purpose**: Harden your application against common vulnerabilities
- `supply-chain.md` - Dependency management and vulnerability scanning
- `xss-prevention.md` - Content Security Policy, DOMPurify
- `authentication.md` - Hybrid token strategy (HttpOnly + in-memory)

### 10-deployment/
**Purpose**: Automate builds and deployments
- `vite-build.md` - Production build optimization
- `docker.md` - Multi-stage Dockerfile with Nginx
- `ci-cd.md` - GitHub Actions workflow

### 11-reference/
**Purpose**: Quick lookup and visual aids
- `architecture-diagram.md` - System architecture visualization
- `full-file-tree.md` - Complete project directory structure
- `resources.md` - Links to documentation and articles

## Agent Workflow Paths

### For Initial Project Setup (30-60 min)
1. Read `01-quick-start/setup.md`
2. Apply `01-quick-start/folder-structure.md`
3. Configure with `01-quick-start/tooling-config.md`
4. Reference `02-core-concepts/composition-api.md` to start coding

### For Feature Development
1. Review `02-core-concepts/` for fundamentals
2. Choose patterns from `03-design-patterns/`
3. Implement state with `04-state-data/`
4. Integrate API calls using `05-api-layer/`
5. Add UI with `06-ui-ux/routing.md` and `06-ui-ux/forms-validation.md`

### For Testing Implementation
1. Start with `08-testing/testing-strategy.md`
2. Implement unit tests per `08-testing/unit-testing.md`
3. Add component tests per `08-testing/component-testing.md`
4. Create E2E tests per `08-testing/e2e-testing.md`
5. Configure CI with `08-testing/quality-gates.md`

### For Security Hardening
1. Review all of `09-security/`
2. Implement CSP from `09-security/xss-prevention.md`
3. Configure auth flow from `09-security/authentication.md`
4. Set up scanning per `09-security/supply-chain.md`

### For Production Deployment
1. Optimize build with `10-deployment/vite-build.md`
2. Containerize using `10-deployment/docker.md`
3. Set up CI/CD pipeline with `10-deployment/ci-cd.md`

## Quick Lookup Index

**I need to...**
- Scaffold a new project → `01-quick-start/setup.md`
- Understand `<script setup>` → `02-core-concepts/composition-api.md`
- Use TypeScript correctly → `02-core-concepts/typescript-guide.md`
- Understand ref vs reactive → `02-core-concepts/reactivity-deep-dive.md`
- Create reusable logic → `03-design-patterns/composables.md`
- Manage global state → `04-state-data/pinia-stores.md`
- Fetch data from APIs → `04-state-data/tanstack-query.md` or `05-api-layer/axios-setup.md`
- Validate API responses → `05-api-layer/zod-validation.md`
- Set up routing → `06-ui-ux/routing.md`
- Make my app accessible → `06-ui-ux/accessibility.md`
- Add internationalization → `06-ui-ux/internationalization.md`
- Build forms → `06-ui-ux/forms-validation.md`
- Improve performance → `07-performance/code-splitting.md`, `07-performance/caching-strategies.md`
- Write tests → `08-testing/` (all files)
- Secure my app → `09-security/` (all files)
- Deploy to production → `10-deployment/` (all files)
- See the big picture → `11-reference/architecture-diagram.md`

## Critical Reading

**Must-read files** (minimum for production-ready implementation):
- [ ] `01-quick-start/setup.md`
- [ ] `01-quick-start/tooling-config.md`
- [ ] `02-core-concepts/composition-api.md`
- [ ] `02-core-concepts/typescript-guide.md`
- [ ] `04-state-data/state-strategy.md`
- [ ] `05-api-layer/zod-validation.md`
- [ ] `08-testing/testing-strategy.md`
- [ ] `09-security/authentication.md`
- [ ] `09-security/xss-prevention.md`

**Important but context-dependent**:
- `03-design-patterns/` (choose patterns as needed)
- `06-ui-ux/accessibility.md` (required for public-facing apps)
- `07-performance/` (when performance issues arise)
- `10-deployment/docker.md` (if containerizing)

## Complexity Overview

| Component | Complexity | Time Est. | Priority |
|-----------|------------|-----------|----------|
| Initial setup | Low | 30 min | Critical |
| TypeScript configuration | Medium | 1 hour | Critical |
| Core concepts mastery | Medium | 4 hours | Critical |
| State management setup | Medium | 2 hours | Critical |
| API layer + validation | Medium | 2 hours | Critical |
| Testing infrastructure | Medium | 3 hours | Critical |
| Security hardening | High | 4 hours | Critical |
| Performance optimization | Medium | 2 hours | Important |
| UI/UX features (a11y, i18n) | Medium | 3 hours | Context-dependent |
| Deployment pipeline | High | 4 hours | Critical |

## Key Architectural Decisions

This guide promotes these opinionated defaults:

1. **Architecture**: Modular, feature-centric with Composition API
2. **State**: Composables for local, Pinia for global
3. **API**: Centralized Axios + Zod runtime validation
4. **Testing**: Vitest (unit/component) + Playwright (E2E)
5. **Security**: Hybrid token flow + strict CSP
6. **Build**: Vite + multi-stage Docker + GitHub Actions

---

## Navigation

- **Start Here**: `01-quick-start/setup.md`
- **Reference**: `11-reference/` for quick lookups
