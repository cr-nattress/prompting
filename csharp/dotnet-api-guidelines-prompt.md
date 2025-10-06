Prompt: Deep-Research + Build — .NET 8 Web API Best Practices (C#)

Role: You are a senior staff-level .NET architect and technical writer.
Mission: Perform deep research and produce a single, publication-ready Markdown document that teaches how to build a production-grade .NET 8 Web API in C#, including best practices, design patterns, and runnable code examples.

Output Contract

Deliverable: One self-contained Markdown document.

Tone: Clear, pragmatic, opinionated where trade-offs matter; avoid fluff.

Audience: Senior developers and tech leads who want a solid default architecture.

Structure (exactly in this order):

Title, Author, Date (ISO), Abstract

Table of Contents (with anchors)

Executive Summary (bullet list of key recommendations)

Architecture Overview (diagram in Mermaid)

Project Setup (CLI steps)

Core Best Practices (C# + .NET 8)

Design Patterns (with when/why trade-offs)

API Walkthrough (runnable minimal API + layered example)

Data & Persistence (EF Core 8, migrations, seeding)

Validation, Errors, and ProblemDetails

Security (JWT/OAuth2/OIDC), Authorization, Rate Limiting

Observability (logging, tracing, metrics, health)

Performance (async, pooling, output caching, benchmarking)

Versioning, OpenAPI/Swagger, and Client Generation

Testing Strategy (unit, integration, contract tests)

Deployment (Dockerfile, CI example, cloud notes)

Appendix A: Full File Tree

Appendix B: References & Further Reading (with links)

Citations: When claiming a recommendation or API detail, cite primary sources (Microsoft docs, .NET blog, OWASP). Use inline footnotes like: [1].

Code: All code must compile against .NET 8; mark blocks with language hints (e.g., csharp, bash, json, yaml, http).

Deep Research Protocol

Prefer primary sources: Microsoft Learn, .NET Blog, ASP.NET Core docs, EF Core docs, OWASP ASVS/Top 10, IETF RFCs when relevant.

Compare at least 3 sources for each critical topic (security, performance, versioning). Note disagreements and justify the chosen approach.

Capture URLs and publish dates; include footnote references at the end.

Avoid stale content (pre-.NET 8 or obsolete APIs) unless noting “legacy” and providing the modern alternative.

Opinionated Defaults (justify if you differ)

Project: dotnet new webapi (minimal hosting), nullable enabled, implicit usings.

Architecture: Clean architecture (API → Application → Domain → Infrastructure), with CQRS (MediatR optional, discuss trade-offs).

Validation: DataAnnotations + FluentValidation for complex rules.

Error Handling: Centralized exception middleware → RFC 7807 ProblemDetails.

Persistence: EF Core 8, DbContext per request, migrations, bounded contexts when warranted.

AuthN/Z: JWT bearer with OIDC provider (e.g., Auth0/Azure AD). Role/Policy-based Authorization.

Observability: Serilog (structured logs), OpenTelemetry for tracing/metrics, HealthChecks UI.

Performance: async all the way, Output Caching (ASP.NET Core 8), IAsyncEnumerable where useful, object pooling when hot paths demand.

API: OpenAPI with Swashbuckle; API Versioning via Asp.Versioning.Http.

Testing: xUnit + FluentAssertions; WebApplicationFactory for integration; Pact (or similar) for consumer-driven contracts.

Hardening: Rate limiting middleware, input size limits, HTTPS only, security headers.

Must-Include Artifacts (copy-paste runnable)

Commands:

dotnet new webapi -n Demo.Api
dotnet new classlib -n Demo.Application
dotnet new classlib -n Demo.Domain
dotnet new classlib -n Demo.Infrastructure
dotnet sln add Demo.*
dotnet add Demo.Api reference Demo.Application Demo.Infrastructure Demo.Domain


Program.cs (minimal API with health checks, versioning, Swagger, output caching, rate limiting, ProblemDetails, Serilog bootstrap).

appsettings.json sample with logging levels and connection strings (no secrets).

Entity example (TodoItem) + DbContext + Migrations (dotnet ef migrations add Init).

Repository vs. EF Core guidance (when to skip repository pattern; pros/cons).

CQRS sample (Query + Command handlers) with MediatR and a variant without MediatR (pure DI), explaining trade-offs.

Validation example with FluentValidation.

Global exception/ProblemDetails middleware.

Auth: JWT bearer configuration + [Authorize(Policy = "...")] example.

Rate Limiting: Fixed/Token bucket example using built-in middleware.

Output Caching: Per-route policy demonstrating ETag/Cache-Control interop.

OpenAPI: Swashbuckle filters (schema, example providers), NSwag client generation snippet.

Tests:

Unit test for a domain service.

Integration test using WebApplicationFactory.

Optional contract test sketch (e.g., Pact) with why/when.

Dockerfile (SDK build stage + ASP.NET runtime stage) and .dockerignore.

GitHub Actions CI example (build, test, publish test results, Docker build).

Mermaid architecture diagram:

flowchart LR
  Client -->|HTTP/JSON| API[ASP.NET Core 8 API]
  API --> App[Application Layer]
  App --> Dom[Domain]
  App --> Infra[Infrastructure: EF Core, External Services]
  Infra --> DB[(PostgreSQL/SQL Server)]
  API --> Obs[OpenTelemetry/Serilog/HealthChecks]

Core Best Practices to Cover (brief bullets + examples)

C# 12/modern idioms: required members, primary constructors (if applicable), pattern matching, spans and memory when warranted.

DI: Avoid service locator; prefer constructor injection; keyed services when variants exist.

Configuration: Options pattern (IOptions<T>), validation, binding pitfalls.

Async: Avoid Task.Result/.Wait(), cancellation tokens, ValueTask for hotspots.

Mapping: When to use Mapster/AutoMapper vs. manual mapping (perf + clarity).

API design: Idempotency, pagination, filtering, consistency of status codes.

Resilience: Polly (retry, circuit-breaker) with HttpClientFactory.

Data: EF Core tracking vs. no-tracking queries, compiled queries, transaction scopes.

Security: OWASP API Top 10 alignment; secrets with environment variables/Key Vault.

Perf checks: Benchmarks with BenchmarkDotNet for critical paths.

Design Patterns (teach when/why, not just “what”)

CQRS (with/without MediatR)

Decorator (cross-cutting concerns like logging/metrics/validation)

Strategy (business rules variants)

Factory (complex object graphs)

Specification (query logic encapsulation)

Outbox/Inbox (reliability with messaging) — outline and link references

Saga/Process Manager (if multi-service workflows are discussed)

Example Snippets (required)

Include concise, runnable examples for:

Minimal API endpoint with validation + output caching:

app.MapPost("/todos", async (CreateTodo cmd, ITodoService svc, CancellationToken ct) =>
{
    var id = await svc.CreateAsync(cmd, ct);
    return Results.Created($"/todos/{id}", new { id });
})
.AddEndpointFilter<ValidationFilter<CreateTodo>>()
.CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));


ProblemDetails mapping in exception middleware.

JWT bearer configuration in Program.cs with AddAuthorizationBuilder().

EF Core 8 DbContext and OnModelCreating with required fields and indexes.

xUnit integration test with WebApplicationFactory<Program>.

Testing & Quality Bars

Provide commands to run all tests.

Include minimal coverage gates strategy or rules of thumb.

Show an example HTTP file (requests.http) with sample requests/responses.

Deployment Notes

Containerize with a multi-stage Dockerfile; mention distroless or Alpine trade-offs.

Readiness/Liveness health probes.

Basic cloud notes for Azure App Service and container orchestrators (resources & links).

References Section

List each citation with: Title — Publisher — Date — URL.

Prioritize: Microsoft Docs/Learn, .NET Blog (release notes), OWASP, IETF RFCs, official GitHub repos.

Final Checks

All code targets .NET 8 and builds.

No obsolete APIs (note replacements if you mention them historically).

Every nontrivial recommendation has a citation.

Reminder: Produce the full document now, not an outline. Where you must choose between multiple viable approaches, make the choice explicit, briefly justify it, and link sources. If something is debated, present the trade-offs and a default recommendation. Keep the entire document cohesive and runnable end-to-end.