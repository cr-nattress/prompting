Technologies for Vue 3 + TypeScript Apps
Core & Language

Node.js 20+, npm/pnpm

TypeScript 5+

Vue 3.4+ (Composition API, <script setup>)

Build, Dev Server, Tooling

Vite 5+ (build/dev/optimize)

ESLint + @vue/eslint-config-typescript, Prettier

Volar (TS language features for Vue)

Husky + lint-staged (pre-commit hooks)

Commitlint + Conventional Commits (optional)

Changesets (versioning/releases, optional)

rollup-plugin-visualizer (bundle analysis)

vite-plugin-inspect (DX profiling)

Routing, State, Data

Vue Router 4 (typed routes, guards, lazy chunks)

Pinia (global state) + optional pinia-plugin-persistedstate

TanStack Query (Vue Query) (data fetching, caching, retries)

Axios (API client) or native fetch

Zod (runtime validation & type inference)

VueUse (batteries-included composables)

Forms, Validation, UX

Vee-Validate + Zod (schema-based forms)

Tailwind CSS + PostCSS + Autoprefixer

Headless UI for Vue (or Radix Vue / Ark UI)

vue-i18n (localization)

Security & Privacy

CSP (Content Security Policy) config

DOMPurify (sanitize untrusted HTML)

Token handling strategy (httpOnly cookies preferred; SPA trade-offs)

Dependency audit (e.g., npm audit, dependabot)

Observability

Sentry (error tracking) or equivalent

Web-Vitals logging

Source maps upload in CI

Testing

Vitest + Vue Test Utils (unit/component tests)

Playwright (e2e tests)

jsdom (DOM in unit tests)

CI/CD & Deploy

GitHub Actions (install → lint → typecheck → test → build → deploy)

Docker (multi-stage build) + nginx (static serve)

Hosting targets: Vercel / Netlify (static) or Cloud Run / Fly.io (container)

Caching headers, HTML5 history fallback

Prompt: Deep-Research Implementation Guidelines & Best Practices (Vue 3 + TS)

Role: You are a senior front-end architect and standards author.
Mission: Produce a single, publication-ready Markdown document that gives implementation guidelines and industry best practices for each technology in our Vue 3 stack, with runnable code, pitfalls, and checklists.

Scope (cover each item below)

Core: Node 20+, TypeScript 5+, Vue 3.4+ (<script setup>, Composition API)

Tooling: Vite 5+, ESLint + Prettier, Volar, Husky + lint-staged, Commitlint, Changesets (optional), rollup-plugin-visualizer, vite-plugin-inspect

Routing: Vue Router 4 (typed params, guards, lazy routes, error routes, scroll restore)

State: Pinia (+ persistedstate plugin), store structure, testing stores

Data: TanStack Query (caching, invalidation, retries), Axios/fetch API client, error normalization

Validation: Zod; Vee-Validate + Zod forms; accessibility considerations

Composables: useX() patterns, cancellation, error boundaries, SSR-safety notes (if relevant)

Styling/UI: Tailwind CSS conventions (tokens, dark mode), Headless component libs (Headless UI/Radix/Ark)

i18n: vue-i18n setup, lazy-loading locales, typed messages

Security: CSP, sanitize untrusted HTML (DOMPurify), auth token strategies (httpOnly vs SPA), dependency updates

Observability: Sentry integration, web-vitals logging, source maps

Testing: Vitest + Vue Test Utils (unit/component), Playwright (e2e), test data builders & fixtures

CI/CD: GitHub Actions workflow (install, lint, typecheck, tests, build, preview deploy), Dockerfile, nginx config, caching headers, history fallback

Performance: code-splitting, dynamic imports, Suspense, defineAsyncComponent, memoization, over-reactivity pitfalls, bundle analysis

Output Contract

Deliverable: One cohesive Markdown guide.

Audience: Senior engineers & tech leads.

Tone: Clear, pragmatic, opinionated where needed; justify trade-offs.

Citations: Prefer primary sources (Vue, Vite, Pinia, Router, TanStack Query, Vee-Validate, Zod, MDN/WAI, Playwright). Use footnotes like [1] and include a References section with Title — Publisher — Date — URL.

Code: All examples must work on Node 20+, Vite 5+, Vue 3.4+, TS 5+. Tag blocks with ts, vue, bash, json, yaml, nginx.

Required Structure (in this exact order)

Title, Author, Date (ISO), Abstract

Table of Contents (anchor links)

Principles & Non-Goals (security, accessibility, testability, performance budgets)

Project Bootstrap

Commands to scaffold a TS Vue app via npm create vue@latest

Install commands for all listed libraries/tools

Recommended file tree (src/components, src/pages, src/routes, src/stores, src/composables, src/services, src/styles, src/types, tests/, etc.)

TypeScript & Linting Setup

tsconfig.json (strict mode, noUncheckedIndexedAccess, exactOptionalPropertyTypes)

ESLint + Prettier configs (import order, Vue + TS rules)

Husky + lint-staged, Commitlint, Conventional Commits

Vite Configuration

Aliases (@), environment variables, chunking hints, dev server security notes

Bundle analysis (visualizer), DX profiling (inspect)

Vue Router 4

Typed routes/params, lazy routes & chunk names

Auth guards (sync/async), error boundaries, 404/500 routes, scroll behavior

Example: guarded route + redirect

Pinia

Store patterns, action vs. getter guidance, persisted state, SSR notes

Example: fully typed store with tests

Data Layer

Axios instance (interceptors, timeouts), or fetch wrapper

Zod schemas for IO validation; narrow types via z.infer

TanStack Query: query keys, caching, invalidation, background refetch, optimistic updates, error handling

Example: typed useUser() composable with cancellation and retries

Forms & Validation

Vee-Validate + Zod; accessible forms (labels, errors, ARIA live regions)

Example: login form with schema, disabled submit until valid

Composables Patterns

Options in, typed state/actions out; handling refs/reactivity; cleanup; testability

Styling & Components

Tailwind tokens/variants, dark mode strategy, component primitives

Headless UI/Radix/Ark patterns; focus management utilities

i18n

vue-i18n setup; lazy-loaded locales; typed messages; fallback strategy

Security

CSP examples; avoiding v-html with untrusted content; DOMPurify usage

Auth token storage strategies; cookie vs. localStorage trade-offs

Dependency auditing and update cadence

Observability

Sentry setup; source maps in CI; correlation IDs; web-vitals logging pattern

Testing Strategy

Unit/component patterns with Vitest + Vue Test Utils (props, emits, slots)

e2e with Playwright; auth flow example; fixtures & test data builders

Commands to run all tests; minimal coverage bars

Performance Playbook

Code-splitting, async components, Suspense; avoiding over-reactivity

Prefetch strategies, keep-alive cautions, memoization, image budgets

Bundle analysis workflow & remediation checklist

CI/CD & Deployment

GitHub Actions workflow (matrix Node, caching)

Dockerfile (multi-stage) + .dockerignore

nginx config for SPA (history fallback) and caching headers

Hosting notes (Vercel/Netlify vs. container) and environment strategy

Checklists

Security (CSP, sanitize, tokens)

Accessibility (keyboard paths, contrast, focus outlines, ARIA live)

Performance (budgets, lazy routes, analysis)

Quality (typecheck, lint, tests in CI)

References (primary sources with dates/URLs)

Example Snippets (must include)

vite.config.ts with aliases

router.ts with a typed param route + guard

Pinia store + unit test

useQuery example with Zod validation and error normalization

Vee-Validate + Zod form SFC

Tailwind config (tokens, dark mode)

CSP example + DOMPurify usage snippet

Sentry init + source maps upload step in CI

Playwright e2e smoke test

GitHub Actions workflow YAML

Dockerfile (multi-stage) + nginx config for SPA

Research & Citations

For each major section (Router, Pinia, Query, Forms, Security, Testing, Deploy), consult at least 3 primary sources, compare recommendations, call out disagreements, choose a default, and justify it.

Include publish/update dates; avoid Vue 2 content unless clearly marked “legacy”.

Final Checks

All code targets Node 20+, Vite 5+, Vue 3.4+, TS 5+.

No Options API unless marked legacy.

Every guideline has either a code sample, a checklist, or both.

The document is self-contained and immediately actionable.