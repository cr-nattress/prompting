# .NET 8 Web API Implementation Guidelines - AI Agent Reference Guide

> **Last Updated**: 2025-10-06
> **Original Source**: Deep-Research Implementation Specification
> **Optimization Version**: 1.0
> **Target**: .NET 8, C# 12, Production-Grade APIs

## What This Is

This comprehensive guide defines implementation guidelines and industry best practices for building production-ready C# / ASP.NET Core 8 Web APIs. It synthesizes primary sources from Microsoft Learn, .NET Blog, OWASP, and IETF into actionable, copy-paste-ready implementations with runnable code, checklists, and authoritative references.

The content follows Clean/Hexagonal architecture principles with minimal hosting model, focusing on **security-first, testable-by-default, and observable** systems with defined performance budgets.

**Optimization**: This guide has been split into focused files to minimize token usage. AI agents can load only relevant sections based on the current task, reducing context window consumption by ~85% for targeted queries.

## File Structure Guide

This research has been organized into the following structure for optimal agent workflow:

### 01-quick-start/
**Purpose**: Get implementing fast - project scaffolding and basic setup
- `project-bootstrap.md` - CLI commands, solution layout, EditorConfig, analyzers
- `minimal-program-setup.md` - Program.cs composition, DI, middleware pipeline
- `first-endpoint.md` - Basic minimal API endpoint with validation

### 02-architecture/
**Purpose**: Understand foundational architectural patterns
- `clean-architecture.md` - Layers, boundaries, dependency rules, Mermaid diagram
- `domain-layer.md` - Entities, aggregates, value objects, domain services
- `application-layer.md` - CQRS patterns, commands/queries, validation strategy

### 03-infrastructure/
**Purpose**: Data persistence and infrastructure concerns
- `ef-core-setup.md` - DbContext, configurations, migrations, seeding
- `data-patterns.md` - Repositories vs DbContext trade-offs, compiled queries
- `connection-management.md` - Pooling, resiliency, transactions, NodaTime handling

### 04-api-design/
**Purpose**: API design standards and conventions
- `endpoints-and-routing.md` - Minimal APIs vs Controllers, endpoint conventions
- `versioning.md` - Asp.Versioning.Http configuration and strategies
- `openapi-swagger.md` - Swashbuckle setup, filters, example providers
- `api-standards.md` - Pagination, filtering, sorting, idempotency, status codes

### 05-security/
**Purpose**: Authentication, authorization, and API protection
- `authentication-jwt.md` - JWT/OIDC setup with Auth0/Azure AD
- `authorization.md` - Role/policy-based authorization, scopes, claims
- `api-protection.md` - Rate limiting, request size limits, HTTPS/HSTS, security headers
- `owasp-checklist.md` - OWASP API Top 10 alignment with mitigations

### 06-error-handling/
**Purpose**: Centralized error handling and problem details
- `exception-middleware.md` - Exception mapping strategy and middleware
- `problem-details.md` - RFC 7807 implementation, validation errors, correlation IDs

### 07-observability/
**Purpose**: Logging, tracing, metrics, and health monitoring
- `structured-logging.md` - Serilog configuration, enrichers, request logging
- `opentelemetry.md` - Traces, metrics, logs, OTLP exporters
- `health-checks.md` - Health checks, readiness/liveness probes, HealthChecks UI

### 08-performance/
**Purpose**: Optimization and resilience patterns
- `async-patterns.md` - Async best practices, pooled object patterns
- `caching.md` - Output caching policies, cache headers
- `resilience.md` - HttpClientFactory + Polly (retry, circuit breaker, timeout)
- `benchmarking.md` - BenchmarkDotNet samples for critical paths

### 09-testing/
**Purpose**: Testing strategies and patterns
- `unit-testing.md` - xUnit + FluentAssertions, fakes vs mocks
- `integration-testing.md` - WebApplicationFactory, Testcontainers
- `testing-strategy.md` - Test pyramid, contract tests, API approval testing

### 10-deployment/
**Purpose**: Containerization and CI/CD
- `docker.md` - Multi-stage Dockerfile, .dockerignore
- `kubernetes.md` - Probes, environment config, zero-downtime deployments
- `ci-cd.md` - GitHub Actions workflow for build/test/publish

### 11-reference/
**Purpose**: Quick lookup for code samples and checklists
- `code-artifacts.md` - All runnable code samples consolidated
- `checklists.md` - Security, observability, performance, quality checklists
- `references.md` - Primary sources with citations and dates

## Agent Workflow Paths

### For Quick Implementation (< 1 hour)
1. Read `01-quick-start/project-bootstrap.md`
2. Follow `01-quick-start/minimal-program-setup.md`
3. Implement `01-quick-start/first-endpoint.md`
4. Reference `11-reference/code-artifacts.md` as needed

### For Production-Ready API (Full implementation)
1. Review `02-architecture/` (all files) for foundational understanding
2. Complete `01-quick-start/` setup
3. Implement domain/application layers from `02-architecture/domain-layer.md` and `application-layer.md`
4. Set up infrastructure following `03-infrastructure/` in order
5. Design API endpoints using `04-api-design/` guidelines
6. Secure application following `05-security/` (all files)
7. Implement error handling from `06-error-handling/`
8. Add observability using `07-observability/` patterns
9. Optimize using `08-performance/` techniques
10. Test using `09-testing/` strategies
11. Deploy following `10-deployment/` guides

### For Security Hardening
1. Start with `05-security/owasp-checklist.md`
2. Implement `05-security/authentication-jwt.md`
3. Configure `05-security/authorization.md`
4. Apply `05-security/api-protection.md`
5. Verify with `11-reference/checklists.md` (Security section)

### For Performance Optimization
1. Review `08-performance/async-patterns.md` for foundational practices
2. Implement `08-performance/caching.md` strategies
3. Add `08-performance/resilience.md` for external calls
4. Benchmark hotspots using `08-performance/benchmarking.md`
5. Verify with `11-reference/checklists.md` (Performance section)

### For Testing Implementation
1. Read `09-testing/testing-strategy.md` for overall approach
2. Implement `09-testing/unit-testing.md` patterns
3. Add `09-testing/integration-testing.md` coverage
4. Verify with `11-reference/checklists.md` (Quality section)

### For Deployment Setup
1. Create `10-deployment/docker.md` multi-stage builds
2. Configure `10-deployment/kubernetes.md` probes
3. Set up `10-deployment/ci-cd.md` pipeline
4. Test with `07-observability/health-checks.md`

## Quick Lookup Index

**I need to...**
- Scaffold a new project → `01-quick-start/project-bootstrap.md`
- Configure Program.cs → `01-quick-start/minimal-program-setup.md`
- Create my first endpoint → `01-quick-start/first-endpoint.md`
- Understand Clean Architecture → `02-architecture/clean-architecture.md`
- Model my domain → `02-architecture/domain-layer.md`
- Implement CQRS → `02-architecture/application-layer.md`
- Set up EF Core → `03-infrastructure/ef-core-setup.md`
- Choose repository pattern → `03-infrastructure/data-patterns.md`
- Manage connections → `03-infrastructure/connection-management.md`
- Design API endpoints → `04-api-design/endpoints-and-routing.md`
- Version my API → `04-api-design/versioning.md`
- Configure Swagger → `04-api-design/openapi-swagger.md`
- Implement pagination → `04-api-design/api-standards.md`
- Add JWT authentication → `05-security/authentication-jwt.md`
- Configure authorization → `05-security/authorization.md`
- Add rate limiting → `05-security/api-protection.md`
- Check OWASP compliance → `05-security/owasp-checklist.md`
- Handle exceptions → `06-error-handling/exception-middleware.md`
- Return ProblemDetails → `06-error-handling/problem-details.md`
- Add logging → `07-observability/structured-logging.md`
- Implement tracing → `07-observability/opentelemetry.md`
- Configure health checks → `07-observability/health-checks.md`
- Optimize async code → `08-performance/async-patterns.md`
- Add caching → `08-performance/caching.md`
- Use Polly for resilience → `08-performance/resilience.md`
- Benchmark performance → `08-performance/benchmarking.md`
- Write unit tests → `09-testing/unit-testing.md`
- Write integration tests → `09-testing/integration-testing.md`
- Plan testing strategy → `09-testing/testing-strategy.md`
- Containerize my app → `10-deployment/docker.md`
- Deploy to Kubernetes → `10-deployment/kubernetes.md`
- Set up CI/CD → `10-deployment/ci-cd.md`
- Find code samples → `11-reference/code-artifacts.md`
- Use checklists → `11-reference/checklists.md`
- Read citations → `11-reference/references.md`

## Critical Reading

**Must-read files** (minimum for production implementation):
- [ ] `01-quick-start/project-bootstrap.md`
- [ ] `01-quick-start/minimal-program-setup.md`
- [ ] `02-architecture/clean-architecture.md`
- [ ] `03-infrastructure/ef-core-setup.md`
- [ ] `05-security/authentication-jwt.md`
- [ ] `05-security/owasp-checklist.md`
- [ ] `06-error-handling/exception-middleware.md`
- [ ] `07-observability/structured-logging.md`
- [ ] `09-testing/integration-testing.md`
- [ ] `11-reference/checklists.md`

**Highly recommended** (production hardening):
- [ ] `02-architecture/application-layer.md` (for CQRS)
- [ ] `04-api-design/api-standards.md` (for consistency)
- [ ] `05-security/api-protection.md` (rate limiting, headers)
- [ ] `08-performance/resilience.md` (Polly patterns)
- [ ] `10-deployment/ci-cd.md` (automation)

**Optional but valuable**:
- `08-performance/` (all files for high-traffic optimization)
- `09-testing/testing-strategy.md` (comprehensive test planning)

## Complexity Overview

| Component | Complexity | Time Est. | Priority |
|-----------|------------|-----------|----------|
| Project bootstrap | Low | 15 min | Critical |
| Program.cs setup | Medium | 30 min | Critical |
| Clean architecture | Medium | 1 hour | Critical |
| Domain modeling | Medium | 2 hours | Critical |
| EF Core setup | Medium | 1 hour | Critical |
| API design | Medium | 2 hours | High |
| JWT authentication | Medium | 1 hour | Critical |
| Authorization | Medium | 1 hour | High |
| Error handling | Low | 30 min | Critical |
| Structured logging | Low | 30 min | Critical |
| OpenTelemetry | High | 2 hours | Medium |
| Caching | Medium | 1 hour | Medium |
| Resilience patterns | Medium | 1 hour | Medium |
| Testing setup | Medium | 2 hours | High |
| Docker setup | Low | 30 min | High |
| CI/CD pipeline | Medium | 2 hours | Medium |

## Principles & Non-Goals

**Core Principles:**
- **Security first**: Authentication, authorization, and OWASP compliance from the start
- **Testable by default**: Design for testability with clear boundaries
- **Observable**: Structured logging, tracing, and metrics built-in
- **Performance budgets**: Define and measure performance targets
- **Pragmatic over dogmatic**: Choose patterns based on actual needs

**Non-Goals:**
- This is NOT a beginner's tutorial (assumes .NET experience)
- This is NOT framework-agnostic (opinionated about .NET 8 patterns)
- This is NOT exhaustive (focuses on most common production scenarios)

## Key Architectural Decisions

This guide makes opinionated choices based on industry best practices:

1. **Minimal APIs** over Controllers (simpler, faster, .NET 8 native)
2. **Clean Architecture** layers (maintainability, testability)
3. **FluentValidation** for complex rules (expressiveness)
4. **JWT Bearer** with OIDC providers (standard, secure)
5. **EF Core** with migrations (type-safe, maintainable)
6. **Serilog** for structured logging (query-friendly)
7. **OpenTelemetry** for tracing (vendor-neutral)
8. **xUnit** + WebApplicationFactory (integration test standard)
9. **Docker** multi-stage builds (production-optimized)

See individual files for trade-off discussions and alternatives.

## Primary Sources

All recommendations cite primary sources:
- Microsoft Learn (.NET 8, ASP.NET Core, EF Core)
- .NET Blog (release notes, announcements)
- OWASP (API Security Top 10, ASVS)
- RFC 7807 (Problem Details)
- IETF RFCs (JWT, OAuth2, OIDC)

Full citations in `11-reference/references.md`.

---

**Next Steps**: Start with `01-quick-start/project-bootstrap.md` to scaffold your project, or jump to specific topics using the Quick Lookup Index above.
