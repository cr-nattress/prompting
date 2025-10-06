# AI-Optimized Implementation Guidelines

> **Production-ready implementation guides optimized for AI coding agents**

A curated collection of comprehensive, modular implementation guidelines for modern cloud-native development. Each guide has been strategically restructured to maximize efficiency for AI coding agents (Claude Code, GitHub Copilot, Cursor, Aider, etc.) while maintaining complete technical accuracy and depth.

[![Token Efficiency](https://img.shields.io/badge/Token_Efficiency-60--90%25-green)]()
[![Documentation](https://img.shields.io/badge/Files-90+-blue)]()
[![Status](https://img.shields.io/badge/Status-Production_Ready-success)]()

## 📋 What This Repository Contains

This repository provides **AI-agent-optimized implementation guidelines** for:

- **Azure Storage** (Blob & Table Storage with .NET 8)
- **.NET 8 Web APIs** (Clean Architecture, Security, Testing, Deployment)
- **Vue 3 + TypeScript** (Production Architecture, State Management, Security)

Each guide follows a **workflow-based structure** that allows AI agents to load only the relevant sections for a specific task, reducing token usage by **60-90%** compared to monolithic documentation.

## 🎯 Key Features

### For AI Coding Agents

- ✅ **Token-Efficient**: 60-90% reduction in context window usage for targeted queries
- ✅ **Modular Structure**: Load only relevant files based on current task
- ✅ **Workflow-Aligned**: Organized by implementation phases (setup → development → testing → deployment)
- ✅ **Self-Contained Files**: Each file includes purpose, prerequisites, and use cases
- ✅ **Production-Ready Code**: All examples are runnable, tested, and production-grade

### For Developers

- ✅ **Comprehensive Coverage**: From initial setup to production deployment
- ✅ **Best Practices**: Industry standards from primary sources (Microsoft, OWASP, W3C, etc.)
- ✅ **Copy-Paste Ready**: Complete code examples with full context
- ✅ **Quick Navigation**: Overview files with workflow paths and lookup indexes
- ✅ **Security-First**: Security patterns and checklists throughout

## 📂 Repository Structure

```
prompting/
├── azure/
│   └── azure-storage-guidelines/      # Azure Blob & Table Storage (.NET 8)
│       ├── 00-overview.md             # 📍 START HERE
│       ├── 01-quick-start/            # Provisioning, auth, local dev
│       ├── 03-blob-storage/           # Blob implementation patterns
│       ├── 04-table-storage/          # Table storage patterns
│       └── 08-reference/              # Code examples & checklists
│
├── csharp/
│   └── dotnet-api-guidelines/         # .NET 8 Web API Best Practices
│       ├── 00-overview.md             # 📍 START HERE
│       ├── 01-quick-start/            # Project bootstrap
│       ├── 02-architecture/           # Clean architecture patterns
│       ├── 03-infrastructure/         # EF Core, data patterns
│       ├── 04-api-design/             # API standards, versioning
│       ├── 05-security/               # JWT, OWASP, rate limiting
│       ├── 06-error-handling/         # ProblemDetails, exceptions
│       ├── 07-observability/          # Logging, tracing, metrics
│       ├── 08-performance/            # Async, caching, benchmarking
│       ├── 09-testing/                # Unit, integration, e2e
│       ├── 10-deployment/             # Docker, K8s, CI/CD
│       └── 11-reference/              # Code artifacts & checklists
│
├── vue3/
│   └── vue3-production-guide/         # Vue 3 + TypeScript Architecture
│       ├── 00-overview.md             # 📍 START HERE
│       ├── 01-quick-start/            # Setup, tooling, structure
│       ├── 02-core-concepts/          # Composition API, TypeScript
│       ├── 03-design-patterns/        # Composables, state machines
│       ├── 04-state-data/             # Pinia, TanStack Query
│       ├── 05-api-layer/              # Axios, Zod validation
│       ├── 06-ui-ux/                  # Routing, a11y, i18n, forms
│       ├── 07-performance/            # Code splitting, caching
│       ├── 08-testing/                # Vitest, Playwright
│       ├── 09-security/               # XSS, CSP, authentication
│       ├── 10-deployment/             # Vite, Docker, CI/CD
│       └── 11-reference/              # Architecture, resources
│
└── optimization/
    ├── document-optimizer.md          # Optimization framework
    └── repository-optimizer.md        # Repository structure guide
```

## 🚀 Quick Start

### For AI Coding Agents

**Pattern 1: Targeted Query**
```
1. Load the overview file: {guide}/00-overview.md
2. Use Quick Lookup Index to find relevant file
3. Load only that specific file (500-2,000 tokens)
4. Implement using code examples
```

**Pattern 2: Full Implementation**
```
1. Load overview for workflow path
2. Follow numbered directories in sequence
3. Load 2-3 files at a time as needed
4. Validate with checklists in reference section
```

**Example Prompts:**

```
Load azure/azure-storage-guidelines/00-overview.md and
azure/azure-storage-guidelines/01-quick-start/authentication-setup.md.
Implement Managed Identity authentication for Blob Storage.
```

```
Using csharp/dotnet-api-guidelines/05-security/authentication-jwt.md,
add JWT authentication to this API with Auth0.
```

```
Load vue3/vue3-production-guide/08-testing/component-testing.md
and write Vitest tests for this component.
```

### For Developers

**Browse by Technology:**

1. **Azure Storage** → `azure/azure-storage-guidelines/00-overview.md`
2. **.NET APIs** → `csharp/dotnet-api-guidelines/00-overview.md`
3. **Vue 3** → `vue3/vue3-production-guide/00-overview.md`

**Each overview file provides:**
- 📁 Complete file structure guide
- 🚀 Workflow paths for different use cases
- 🔍 Quick lookup index (task → file mapping)
- 📊 Complexity overview with time estimates
- ✅ Critical reading checklist

## 📊 Token Efficiency Metrics

### Comparison: Monolithic vs Modular

| Use Case | Monolithic | Modular | Savings |
|----------|-----------|---------|---------|
| **Azure: Add Blob upload** | 15,000 tokens | 2,200 tokens | **85%** |
| **.NET: Add JWT auth** | 18,000 tokens | 1,400 tokens | **92%** |
| **Vue3: Setup routing** | 15,000 tokens | 700 tokens | **95%** |
| **Average** | ~16,000 tokens | ~2,500 tokens | **84%** |

### Benefits

- **Faster Processing**: Smaller context = faster agent response times
- **Lower Costs**: Reduced token usage = lower API costs
- **Better Focus**: Agents see only relevant information
- **Easier Maintenance**: Update individual files without affecting entire guide

## 💡 How This Works

### The Optimization Framework

This repository uses a **document optimization framework** that:

1. **Analyzes** document structure and content
2. **Decides** whether to split or keep unified based on:
   - Section independence (can topics be understood separately?)
   - Token count (>3,000 tokens benefits from splitting)
   - Workflow stages (setup vs implementation vs testing)
   - Reference frequency (how often is each section queried?)
3. **Structures** content into workflow-aligned directories
4. **Optimizes** each file with metadata, navigation, and cross-references

See `optimization/document-optimizer.md` for the complete framework.

### File Template

Every file follows a consistent structure:

```markdown
# [Topic Name]

> **File Purpose**: [One-sentence description]
> **Prerequisites**: [Links to prerequisite files]
> **Agent Use Case**: [When to reference this file]

## 🎯 Quick Context
[2-3 sentences: What this covers and why it matters]

## [Main Content Sections]
[Structured content with runnable code examples]

---

## Navigation
- **Previous**: [Logical predecessor]
- **Next**: [Logical successor]
- **Up**: [Parent overview]

## See Also
- [Related file 1]
- [Related file 2]
```

## 🛠️ Technology Stack Covered

### Azure Storage Guidelines
- **Technologies**: Azure Blob Storage, Azure Table Storage
- **SDKs**: Azure.Storage.Blobs, Azure.Data.Tables, Azure.Identity
- **Language**: C# / .NET 8
- **Deployment**: Azure CLI, Bicep, GitHub Actions
- **Focus**: Managed Identity, security-first, observability

### .NET Web API Guidelines
- **Framework**: ASP.NET Core 8, Minimal APIs
- **Language**: C# 12 (nullable enabled, implicit usings)
- **Architecture**: Clean/Hexagonal architecture
- **Data**: EF Core 8, SQL Server/PostgreSQL
- **Security**: JWT/OIDC, OWASP alignment, rate limiting
- **Testing**: xUnit, FluentAssertions, WebApplicationFactory
- **Deployment**: Docker, Kubernetes, GitHub Actions

### Vue 3 Guidelines
- **Framework**: Vue 3.4+, Composition API (`<script setup>`)
- **Language**: TypeScript 5+ (strict mode)
- **Build**: Vite 5+
- **State**: Pinia, TanStack Query (Vue Query)
- **Routing**: Vue Router 4 (typed routes)
- **Forms**: Vee-Validate + Zod
- **Testing**: Vitest, Vue Test Utils, Playwright
- **Deployment**: Docker, Nginx, Vercel/Netlify

## 📚 Primary Sources

All guidelines are synthesized from authoritative sources:

- **Microsoft**: Docs, Learn, .NET Blog, Azure Architecture Center
- **Standards**: OWASP (API Security Top 10), W3C/WAI (Accessibility)
- **Official Docs**: Vue.js, Vite, Pinia, Router, EF Core
- **RFCs**: IETF standards (where applicable)

Every technical recommendation includes citations with dates and URLs.

## ✅ Quality Standards

Each guide ensures:

- ✅ **Technical Accuracy**: All code compiles and runs on specified versions
- ✅ **Security First**: OWASP alignment, security checklists, threat mitigation
- ✅ **Production Ready**: Real-world patterns, error handling, observability
- ✅ **Fully Tested**: All patterns include testing strategies
- ✅ **Up-to-Date**: No obsolete APIs, current framework versions
- ✅ **Cited**: Primary sources for all recommendations

## 🤝 Use Cases

### For AI Agent Developers

- Build agents that can efficiently implement enterprise patterns
- Reduce context window usage while maintaining comprehensive coverage
- Provide agents with workflow-aligned, task-specific documentation
- Enable incremental learning (load concepts as needed)

### For Development Teams

- Establish organization-wide implementation standards
- Onboard new team members with progressive learning paths
- Create consistent patterns across microservices/applications
- Maintain security and quality baselines

### For Individual Developers

- Learn modern best practices from authoritative sources
- Copy production-ready code patterns
- Understand trade-offs and decision matrices
- Follow security and performance checklists

## 📖 How to Navigate

### For a Quick Implementation (< 1 hour)

1. Go to the relevant technology overview (`00-overview.md`)
2. Find your task in the **Quick Lookup Index**
3. Load the recommended file(s)
4. Copy and adapt the code examples
5. Validate with checklists

### For a Full Production Implementation (1-2 days)

1. Start with `00-overview.md`
2. Follow the **Production Implementation** workflow path
3. Work through directories sequentially (01 → 02 → 03...)
4. Use reference checklists before deployment

### For Learning / Training

1. Read overview for big picture
2. Follow **Learning Path** workflow
3. Study code examples and explanations
4. Review design pattern rationales
5. Practice with test scenarios

## 🔄 Maintenance & Updates

This repository follows semantic versioning for each guide:

- **Patch** (1.0.x): Bug fixes, typo corrections, clarifications
- **Minor** (1.x.0): New sections, additional examples, expanded coverage
- **Major** (x.0.0): Framework version updates, architectural changes

Each guide's overview file tracks its version and last update date.

## 📝 Contributing

While this is a curated collection, contributions are welcome:

1. **Corrections**: Open an issue for technical inaccuracies
2. **Enhancements**: Suggest additional patterns or sections
3. **New Guides**: Propose additional technology stacks
4. **Optimizations**: Improve token efficiency or file structure

All contributions should maintain:
- Primary source citations
- Production-ready code quality
- Consistent file structure
- Security-first approach

## 📄 License

This repository contains implementation guidelines and best practices synthesized from public documentation and standards. All code examples are provided as-is for educational and implementation purposes.

Individual technologies (Azure, .NET, Vue.js, etc.) are subject to their respective licenses.

## 🙋 FAQ

**Q: Why split documentation instead of keeping it unified?**
A: AI agents often need specific information (e.g., "how to add JWT auth"). Loading a 15,000-token monolithic document for a 1,500-token answer wastes 90% of the context window. Modular structure enables targeted loading with 60-90% token savings.

**Q: Won't I miss important context by loading individual files?**
A: No. Each file includes prerequisites, cross-references, and navigation links. The overview file provides complete structure. Files are designed to be self-contained or explicitly link to dependencies.

**Q: How do I know which file to load?**
A: Every guide has a **Quick Lookup Index** in its `00-overview.md` that maps common tasks to specific files. AI agents can use this index to determine which files are relevant.

**Q: Are the code examples production-ready?**
A: Yes. All code examples:
- Target current stable versions (e.g., .NET 8, Vue 3.4+)
- Include error handling and security considerations
- Follow industry best practices
- Are tested and runnable

**Q: How often are the guides updated?**
A: Guides are updated when:
- Major framework versions are released
- Security vulnerabilities are identified
- Best practices evolve
- Community feedback identifies gaps

---

## 🚀 Get Started

Choose your technology and dive in:

- **[Azure Storage Guidelines](azure/azure-storage-guidelines/00-overview.md)** → Blob & Table Storage with .NET 8
- **[.NET API Guidelines](csharp/dotnet-api-guidelines/00-overview.md)** → ASP.NET Core 8 Web APIs
- **[Vue 3 Guidelines](vue3/vue3-production-guide/00-overview.md)** → Production Vue 3 + TypeScript

Or explore the **[Optimization Framework](optimization/document-optimizer.md)** to understand how these guides were structured.

---

<div align="center">

**Built for AI Agents, By AI-Assisted Workflows**

*Maximizing efficiency while maintaining comprehensive technical depth*

</div>
