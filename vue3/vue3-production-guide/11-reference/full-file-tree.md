# Complete Project File Tree

> **File Purpose**: Reference for complete production Vue 3 project structure
> **Agent Use Case**: Copy/paste when scaffolding new projects

## Production-Ready File Structure

```
my-vue-app/
├── .github/
│   └── workflows/
│       └── ci.yml                    # GitHub Actions CI/CD
├── .husky/
│   └── pre-commit                    # Git hooks
├── .vscode/
│   ├── extensions.json               # Recommended extensions
│   └── settings.json                 # Editor settings
├── public/
│   ├── favicon.ico
│   └── robots.txt
├── src/
│   ├── assets/
│   │   ├── images/
│   │   ├── fonts/
│   │   └── styles/
│   │       ├── main.css
│   │       └── variables.css
│   ├── components/
│   │   ├── common/
│   │   │   ├── BaseButton.vue
│   │   │   ├── BaseModal.vue
│   │   │   └── BaseInput.vue
│   │   ├── layout/
│   │   │   ├── AppHeader.vue
│   │   │   ├── AppFooter.vue
│   │   │   └── AppSidebar.vue
│   │   └── features/
│   │       ├── user/
│   │       │   ├── UserProfile.vue
│   │       │   └── UserAvatar.vue
│   │       └── product/
│   │           ├── ProductCard.vue
│   │           └── ProductList.vue
│   ├── composables/
│   │   ├── useAuth.ts
│   │   ├── useFetch.ts
│   │   ├── useLocalStorage.ts
│   │   └── __tests__/
│   │       └── useAuth.spec.ts
│   ├── router/
│   │   ├── index.ts
│   │   └── guards.ts
│   ├── services/
│   │   ├── api/
│   │   │   ├── client.ts
│   │   │   ├── users.ts
│   │   │   └── products.ts
│   │   └── validators/
│   │       ├── userSchema.ts
│   │       └── productSchema.ts
│   ├── stores/
│   │   ├── auth.ts
│   │   ├── cart.ts
│   │   └── __tests__/
│   │       └── auth.spec.ts
│   ├── types/
│   │   ├── models.ts
│   │   ├── api.ts
│   │   └── env.d.ts
│   ├── utils/
│   │   ├── formatters.ts
│   │   ├── sanitize.ts
│   │   └── constants.ts
│   ├── views/
│   │   ├── HomeView.vue
│   │   ├── LoginView.vue
│   │   ├── DashboardView.vue
│   │   └── NotFoundView.vue
│   ├── App.vue
│   └── main.ts
├── tests/
│   └── e2e/
│       ├── auth.spec.ts
│       └── fixtures/
│           └── users.json
├── .dockerignore
├── .env
├── .env.local
├── .eslintrc.cjs
├── .gitignore
├── .prettierrc.json
├── .prettierignore
├── Dockerfile
├── docker-compose.yml
├── nginx.conf
├── index.html
├── package.json
├── package-lock.json
├── playwright.config.ts
├── README.md
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts
└── vitest.config.ts
```

## File Count Summary

- **Configuration files**: ~15
- **Source files**: ~30-50 (depending on features)
- **Test files**: ~10-20 (1:1 ratio with implementation)
- **Total**: ~60-85 files for a medium-sized production app

---

## Navigation

- **Up**: `00-overview.md`
