Prompt: Deep-Research + Build — Vue 3 + TypeScript Best Practices

Role: You are a senior front-end architect and technical writer.
Mission: Perform deep research and produce a single, publication-ready Markdown document that teaches how to build a production-grade Vue 3 (Composition API) + TypeScript app, covering best practices, design patterns, and runnable code examples.

Output Contract

Deliverable: One self-contained Markdown document.

Tone: Clear, pragmatic, slightly opinionated; justify trade-offs.

Audience: Senior engineers and tech leads.

Structure (exact order):

Title, Author, Date (ISO), Abstract

Table of Contents (with anchors)

Executive Summary (key recommendations)

Architecture Overview (Mermaid diagram)

Project Setup (CLI, folder structure)

Core Best Practices (Vue 3 + TS)

Design Patterns for Front-End (with when/why)

State Management (Pinia) & Data Fetching

Type-Safe API Layer (OpenAPI/typed clients)

UI/UX Foundations (accessibility, i18n, routing)

Performance (code-splitting, caching, reactivity pitfalls)

Testing (unit, component, e2e)

Tooling & DX (linting, formatting, commit hooks)

Security (supply chain, DOMXSS, auth flows)

Build & Deploy (Vite config, Docker, CI)

Appendix A: Full File Tree

Appendix B: References & Further Reading (with links)

Citations: Prefer primary sources (Vue docs, Vite docs, Pinia docs, MDN, W3C/WAI, OWASP). Use inline footnotes like [1] and list full references at the end.

Code: All code must run on Node 20+, Vite 5+, Vue 3.4+, TypeScript 5+. Label blocks (ts, vue, bash, json, yaml).

Deep Research Protocol

Compare at least 3 sources for each critical topic (state mgmt, routing, performance, testing, security).

Record URLs + publish dates; avoid stale Vue 2 content unless noting legacy and modern alternative.

When sources disagree, state the options and justify the chosen default.

Capture config defaults (Vite, TS tsconfig.json, ESLint) from official docs.

Opinionated Defaults (justify if you differ)

Scaffold: npm create vue@latest → Vite + TS + Vue Router + Pinia + Vitest.

Language & Style: Composition API with <script setup>, SFCs, Volar types, ESLint + Prettier.

State: Pinia for app-level state; composables for view-local state.

Data: fetch/axios via a typed API client; optional TanStack Query (Vue Query) for caching & retries.

Routing: Vue Router 4 with typed routes, lazy-loaded chunks, route guards, and auth flows.

Forms & Validation: Vee-Validate + Zod/Yup for schema validation (typed).

Styling: Tailwind CSS with design tokens and dark mode; prefer utility + component primitives.

i18n: vue-i18n with message types.

Security: Content Security Policy (CSP), trusted types where feasible, sanitize untrusted HTML, strict same-site cookies.

Testing: Vitest + Vue Test Utils; Playwright for e2e.

CI: GitHub Actions (install→lint→typecheck→test→build); preview deploys.

Perf: Route-level code splitting, dynamic imports, suspense, memoization, avoid over-reactivity, devtools performance profiling.

Must-Include Artifacts (copy-paste runnable)

Scaffold & Dependencies

npm create vue@latest my-app -- --ts
cd my-app
npm i @pinia/nuxt pinia # if not Nuxt, just 'pinia'
npm i -D eslint prettier @vue/eslint-config-typescript @vue/eslint-config-prettier
npm i axios zod @tanstack/vue-query
npm i -D vitest @vue/test-utils jsdom playwright @playwright/test
npm i -D eslint-plugin-security


Recommended file tree with src/ subfolders: components/, pages/, routes/, stores/, composables/, services/, assets/, styles/, types/, test/.

tsconfig.json with strict options (strict, noUncheckedIndexedAccess, exactOptionalPropertyTypes).

ESLint + Prettier configs (rules for Vue/TS, import order, security plugin).

Vite config: path aliases, env handling, chunking hints.

Router setup with lazy routes, typed params, auth guard example.

Pinia store with full typing, persist example, and a unit test.

Composable pattern (useUser, useFetchTyped) demonstrating ref, computed, watchEffect, cancellation, and error boundaries.

API client: Axios instance with interceptors, Zod schemas, narrow types from schemas, error normalization.

Vue Query example: caching, invalidation, optimistic updates.

Form validation: Vee-Validate + Zod schema, accessible form example.

Error handling: global error boundary component + error/reporting service (Sentry snippet optional).

Accessibility: ARIA examples, focus management, keyboard traps prevention.

i18n: locale switcher, message typing.

Testing:

Vitest unit test for store and composable.

Component test with Vue Test Utils (props, emits, v-model).

Playwright e2e smoke (route loads, auth redirect).

Security:

CSP example (meta header in dev, helmet proxy note for prod).

DOM sanitization snippet for untrusted HTML.

Build & Deploy:

Dockerfile (dist build + nginx serve), .dockerignore.

GitHub Actions CI example.

Static hosting notes (Netlify/Vercel) and container notes (Cloud Run/Fly.io).

Core Best Practices (with examples)

TypeScript Strictness: enable strict; prefer as const, satisfies, discriminated unions; avoid any.

Reactivity: Prefer computed for derivations; shallow vs. deep refs; avoid unnecessary watch; explain reactive pitfalls with objects/arrays.

SFC Structure: <script setup lang="ts">, defineProps/Emits, top-level await as needed, co-locate small components.

State Strategy: “local via composables, shared via Pinia.” Keep stores small; model cache/state separately.

API Layer: Centralize endpoints; pure functions returning typed results; map server models → view models.

Error UX: use Suspense + error boundaries; retry affordances and toasts; log with context IDs.

Routing: auth guards, optimistic navigation, scroll restore, 404/500 routes.

Styling: design tokens, semantic utilities, component variants; avoid global leakage.

Performance: lazy components, defineAsyncComponent, route prefetch, v-memo, keep-alive judiciously, web vitals budgets.

Accessibility: keyboard-first paths, focus outlines, color contrast, announce async states with ARIA live regions.

Internationalization: lazy-load locales, message keys as TS types.

Observability: web-vitals logging, error tracking (Sentry SDK) with source maps.

Design Patterns (teach when/why)

Composables (Hooks): share logic via useX(); accept options, return typed state/actions.

Container/Presentational Components: isolate fetch/side-effects from UI.

State Machines (XState optional) for complex flows (auth, checkout).

Repository/Adapter: API client behind interface; swap mocks in tests.

Strategy: pluggable algorithms (formatting, pricing).

Observer/Event Bus (lightweight) for cross-feature communication.

Facade: expose unified feature API to pages.

Example Snippets (required)

Typed Router with Guard

// src/routes/index.ts
import { createRouter, createWebHistory } from 'vue-router';
const routes = [
  { path: '/', component: () => import('../pages/Home.vue') },
  { path: '/users/:id', name: 'user', component: () => import('../pages/User.vue'), props: true },
  { path: '/login', component: () => import('../pages/Login.vue') },
] as const;

const router = createRouter({ history: createWebHistory(), routes: routes as any });

router.beforeEach((to, _from) => {
  const isAuthed = !!localStorage.getItem('token');
  if (to.path !== '/login' && !isAuthed) return '/login';
});

export default router;


Pinia Store (typed)

// src/stores/user.ts
import { defineStore } from 'pinia';

export interface User { id: string; email: string }
export const useUserStore = defineStore('user', {
  state: () => ({ current: null as User | null }),
  actions: {
    set(u: User) { this.current = u; },
    clear() { this.current = null; }
  }
});


Composable with Vue Query

// src/composables/useUser.ts
import { useQuery } from '@tanstack/vue-query';
import { ref, computed } from 'vue';
import { z } from 'zod';
const User = z.object({ id: z.string(), email: z.string().email() });
type User = z.infer<typeof User>;

export function useUser(idRef: { value: string }) {
  const { data, error, isLoading, refetch } = useQuery({
    queryKey: ['user', idRef.value],
    queryFn: async (): Promise<User> => User.parse(await (await fetch(`/api/users/${idRef.value}`)).json())
  });
  return { user: data, error, isLoading, refetch, email: computed(() => data.value?.email ?? '') };
}


Form Validation (Vee-Validate + Zod)

// schema.ts
import { z } from 'zod';
export const LoginSchema = z.object({ email: z.string().email(), password: z.string().min(8) });
export type Login = z.infer<typeof LoginSchema>;

<!-- LoginForm.vue -->
<template>
  <form @submit.prevent="onSubmit">
    <input v-model="values.email" type="email" aria-label="Email" />
    <input v-model="values.password" type="password" aria-label="Password" />
    <button :disabled="!meta.valid || isSubmitting">Sign in</button>
    <p v-if="errors.email">{{ errors.email }}</p>
  </form>
</template>

<script setup lang="ts">
import { useForm } from 'vee-validate';
import { toFormikValidationSchema } from 'vee-validate/zod';
import { LoginSchema } from '../types/schema';
const { handleSubmit, errors, values, meta, isSubmitting } = useForm({
  validationSchema: toFormikValidationSchema(LoginSchema)
});
const onSubmit = handleSubmit(async (v) => { /* call API */ });
</script>


Global Error Boundary

<!-- AppErrorBoundary.vue -->
<template>
  <ErrorBoundary>
    <slot />
    <template #error="{ error, reset }">
      <div role="alert">Something went wrong: {{ String(error) }}</div>
      <button @click="reset()">Retry</button>
    </template>
  </ErrorBoundary>
</template>
<script setup lang="ts">
import { ErrorBoundary } from 'vue';
</script>


Vite Config Alias

// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import path from 'node:path';

export default defineConfig({
  plugins: [vue()],
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } }
});

Testing & Quality Bars

Unit/Component: Vitest + Vue Test Utils (mount, props, emits, slots).

e2e: Playwright basic smoke and auth flow.

Static Analysis: ESLint (Vue/TS), security plugin, typecheck gate.

Coverage: Provide a minimal threshold strategy and commands to run all tests.

Example Commands

npm run typecheck
npm run lint
npm run test
npx playwright install && npm run test:e2e
npm run build && npm run preview

Security & Privacy

Lock dependencies; enable dependabot.

Escape/sanitize any dynamic HTML (DOMPurify if needed).

Avoid storing tokens in localStorage when possible; prefer httpOnly cookies; if SPA-only, document trade-offs.

Enforce CSP; avoid v-html with untrusted content.

Handle PII with care; log redaction client-side before shipping errors.

Build, Deploy, and CI

Dockerfile: Multi-stage (build with Node, serve with nginx).

Source Maps: Upload to error tracker; restrict public access.

CI: GH Actions workflow with install → lint → typecheck → unit tests → e2e (optional on PR labels) → build → preview deploy.

Hosting: Netlify/Vercel static; Cloud Run/Fly.io container. Include caching headers and HTML5 history fallback.

References Section

List each citation with: Title — Publisher — Date — URL.

Prioritize: Vue 3 docs, Vite docs, Pinia, Vue Router, TanStack Query, Vee-Validate, Zod, Playwright, MDN, WAI-ARIA, OWASP.

Final Checks

All examples compile on Node 20+ with Vue 3 + TS 5.

No Vue 2 or Options API unless explicitly marked as legacy.

Every nontrivial recommendation is cited.

Reminder: Produce the full document now, not an outline. Where multiple viable approaches exist, make the trade-offs explicit, choose a default, and cite sources.