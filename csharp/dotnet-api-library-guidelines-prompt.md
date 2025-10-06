Prompt: Deep-Research Implementation Guidelines & Best Practices (C# / .NET 8 Web API)

Role: You are a principal .NET architect and standards author.
Mission: Produce a single, publication-ready Markdown document that defines implementation guidelines and industry best practices for building C# / ASP.NET Core 8 Web APIs, including runnable code, checklists, and references.

Scope (cover each item thoroughly)

Platform & Language: .NET 8, C# 12 (nullable enabled, implicit usings)

Project Style: Minimal hosting model; Clean/Hexagonal architecture (API → Application → Domain → Infrastructure)

Data & Persistence: EF Core 8, migrations, seeding, transactions; SQL Server or PostgreSQL

Validation: DataAnnotations + FluentValidation

Error Handling: Centralized exception middleware + RFC 7807 ProblemDetails

AuthN/Z & Security: JWT Bearer with OIDC (Auth0/Azure AD), role/policy-based authorization, input limits, rate limiting, HTTPS/HSTS, security headers, OWASP alignment

API Design: Versioning (Asp.Versioning), OpenAPI/Swagger (Swashbuckle), idempotency, pagination/filtering, consistency of status codes

Observability: Serilog (structured logging), OpenTelemetry (traces/metrics/logs), health checks + HealthChecks UI

Resilience & Networking: HttpClientFactory + Polly (retry, circuit breaker, timeout, fallback)

Performance: async end-to-end, output caching (ASP.NET Core 8), pooling, compiled EF queries, BenchmarkDotNet for hotspots

Testing: xUnit + FluentAssertions; WebApplicationFactory for integration tests; test containers (optional) for DB

Tooling: dotnet CLI, analyzers (IDEs + Roslyn analyzers, FxCop analyzers), EditorConfig, code style rules

Packaging & Deploy: Docker multi-stage, .dockerignore, K8s readiness/liveness probes, GitHub Actions CI (build/test/scan/publish), environment configuration

Optional Patterns: CQRS (with/without MediatR), Decorator, Specification, Outbox (reliability), Saga basics

Output Contract

Deliverable: One cohesive Markdown guide.

Audience: Senior engineers & tech leads.

Tone: Clear, pragmatic, opinionated where trade-offs matter; justify choices.

Citations: Prefer primary sources (Microsoft Docs/Learn, .NET Blog, EF Core Docs, OWASP, IETF). Use inline footnotes like [1] and a References section with Title — Publisher — Date — URL.

Code: Must build on .NET 8. Tag blocks with csharp, bash, json, yaml, http, dockerfile.

Required Structure (use this exact order)

Title, Author, Date (ISO), Abstract

Table of Contents (with anchors)

Principles & Non-Goals (security first, testable by default, observability, performance budgets)

Project Bootstrap

CLI commands to scaffold solution & projects

Recommended solution layout (API/Application/Domain/Infrastructure + Tests)

EditorConfig, analyzers, code-style rules

Architecture & Folder Structure

Mermaid diagram (Clean/Hexagonal)

Dependency rules and boundaries; DTOs vs Domain models

Program Startup (Minimal Hosting)

Program.cs composition: DI, config, options validation, Serilog bootstrap, OpenTelemetry, health checks, rate limiting, output caching, ProblemDetails, Swagger, API versioning

Domain & Application Layers

Entities/aggregates, value objects, domain services

Commands/queries (CQRS) with pros/cons; with and without MediatR

Validation strategy (FluentValidation + pipeline/decorator)

Infrastructure & Data

EF Core 8 DbContext, configurations, migrations, seeding

Transactions, repositories vs direct DbContext (trade-offs), compiled queries

Connection resiliency, pooling, NodaTime/DateTime handling

API Design

Minimal APIs vs Controllers trade-offs; endpoint conventions

Versioning (Asp.Versioning.Http), OpenAPI filters, example providers

Pagination/filtering/sorting standards; idempotency keys for POST/PUT

Error Handling & ProblemDetails

Exception mapping strategy and examples

Validation failure format; correlation IDs

Security

JWT/OIDC setup; scopes/roles/policies; claims mapping

Rate limiting; request size limits; HTTPS/HSTS; security headers

OWASP API Top 10 checklist; secrets via environment/Key Vault

Observability

Serilog enrichers, structured fields, request logging

OpenTelemetry traces/metrics/logs; exporters (OTLP)

Health checks (DB, external deps), readiness/liveness

Performance & Resilience

Async best practices; pooled object patterns; output caching policies

HttpClientFactory + Polly (retry, circuit breaker, timeout) configuration

BenchmarkDotNet sample for critical paths

Testing Strategy

Unit tests (xUnit + FluentAssertions), fakes vs mocks

Integration tests with WebApplicationFactory; Testcontainers (SQL) optional

Contract tests (sketch) and API approval testing (optional)

Deployment

Dockerfile (multi-stage), .dockerignore

K8s probes; environment config; zero-downtime considerations

GitHub Actions CI: build, test, coverage, publish artifacts, Docker build/push

Checklists

Security, Observability, Performance, Quality (lint, analyzers, tests)

References (primary sources, dates, URLs)

Must-Include Artifacts (runnable / copy-paste)

Scaffold commands

dotnet new sln -n Demo
dotnet new webapi -n Demo.Api
dotnet new classlib -n Demo.Application
dotnet new classlib -n Demo.Domain
dotnet new classlib -n Demo.Infrastructure
dotnet sln Demo.sln add Demo.*
dotnet add Demo.Api reference Demo.Application Demo.Infrastructure Demo.Domain
dotnet add Demo.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add Demo.Infrastructure package Microsoft.EntityFrameworkCore.Design
dotnet add Demo.Api package Swashbuckle.AspNetCore
dotnet add Demo.Api package Asp.Versioning.Http
dotnet add Demo.Api package Microsoft.AspNetCore.RateLimiting
dotnet add Demo.Api package Microsoft.AspNetCore.OutputCaching
dotnet add Demo.Api package FluentValidation.AspNetCore
dotnet add Demo.Api package Serilog.AspNetCore
dotnet add Demo.Api package OpenTelemetry.Extensions.Hosting
dotnet add Demo.Api package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add Demo.Api package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add Demo.Api package Polly.Extensions.Http


Program.cs: Minimal API wiring for DI, options, Serilog, OpenTelemetry, health checks, rate limiting, output caching, ProblemDetails, Swagger, API versioning, JWT auth.

Options pattern with validation (IOptions<T> + ValidateDataAnnotations).

Entity + DbContext: TodoItem, AppDbContext, OnModelCreating with required fields & indexes.

EF Core: migration commands (dotnet ef migrations add Init, dotnet ef database update) and seeding example.

Validation: FluentValidation example + pipeline behavior (if CQRS) or endpoint filter.

Error middleware: Map domain/validation/unknown exceptions → ProblemDetails.

Auth: JWT Bearer config with authority/audience; [Authorize(Policy="...")] example.

Rate Limiting: Fixed/Token-bucket example using built-in middleware.

Output Caching: Per-route policy config demonstrating cache hints.

OpenAPI: Swashbuckle setup with operation/Schema filters & example payloads.

Resilience: HttpClientFactory with Polly (retry + circuit breaker) handler.

Testing:

xUnit unit test for domain service

Integration test with WebApplicationFactory<Program>

Benchmarking: Minimal BenchmarkDotNet sample for a hot path.

Dockerfile: Multi-stage build (SDK → ASP.NET runtime) + .dockerignore.

GitHub Actions: CI workflow YAML (restore, build, test, publish test results, build/push Docker).

Example Snippets (required)

Program.cs

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext());

builder.Services.AddEndpointsApiExplorer()
                .AddSwaggerGen()
                .AddProblemDetails()
                .AddHealthChecks();

builder.Services.AddOutputCache();
builder.Services.AddRateLimiter(_ => { /* fixed/token bucket */ });

builder.Services.AddApiVersioning(o => o.DefaultApiVersion = new(1,0))
                .AddApiExplorer();

builder.Services.AddAuthentication("Bearer")
                .AddJwtBearer("Bearer", o =>
                {
                    o.Authority = builder.Configuration["Auth:Authority"];
                    o.Audience  = builder.Configuration["Auth:Audience"];
                    o.RequireHttpsMetadata = true;
                });
builder.Services.AddAuthorization();

// EF Core, Application services, Validators, HttpClientFactory+Polly, etc.

var app = builder.Build();
app.UseSerilogRequestLogging();
app.UseExceptionHandler(); // maps to ProblemDetails
app.UseRateLimiter();
app.UseOutputCache();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapHealthChecks("/health");
app.MapSwagger().RequireAuthorization(); // or public in non-prod
app.UseSwaggerUI();

app.MapGet("/v1/todos/{id:int}", async (int id, AppDbContext db) =>
    await db.Todos.FindAsync(id) is { } todo
      ? Results.Ok(todo)
      : Results.Problem(statusCode: StatusCodes.Status404NotFound, title: "Not Found"))
   .CacheOutput(p => p.Expire(TimeSpan.FromSeconds(30)))
   .RequireAuthorization();

app.Run();


FluentValidation + Endpoint Filter

public record CreateTodo(string Title);

public class CreateTodoValidator : AbstractValidator<CreateTodo>
{
    public CreateTodoValidator() => RuleFor(x => x.Title).NotEmpty().MaximumLength(120);
}

public sealed class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var validator = ctx.HttpContext.RequestServices.GetService<IValidator<T>>();
        var model = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (validator is not null && model is not null)
        {
            var result = await validator.ValidateAsync(model);
            if (!result.IsValid)
                return Results.ValidationProblem(result.ToDictionary());
        }
        return await next(ctx);
    }
}


DbContext & Entity

public class AppDbContext(DbContextOptions<AppDbContext> opts) : DbContext(opts)
{
    public DbSet<TodoItem> Todos => Set<TodoItem>();
    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<TodoItem>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Title).IsRequired().HasMaxLength(120);
            e.HasIndex(x => x.IsDone);
        });
    }
}
public class TodoItem { public int Id { get; set; } public string Title { get; set; } = ""; public bool IsDone { get; set; } }


HttpClientFactory + Polly

builder.Services.AddHttpClient("external")
    .AddPolicyHandler(GetResilience());

static IAsyncPolicy<HttpResponseMessage> GetResilience() =>
    HttpPolicyExtensions.HandleTransientHttpError()
        .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(200 * i))
        .WrapAsync(HttpPolicyExtensions.HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));


Integration Test with WebApplicationFactory

public class TodosApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public TodosApiTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task Get_Returns404_WhenMissing()
    {
        var res = await _client.GetAsync("/v1/todos/999999");
        res.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}


Dockerfile (multi-stage)

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore && dotnet publish Demo.Api/Demo.Api.csproj -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Demo.Api.dll"]

Research & Citations

For each major section (Security, Observability, Data, Versioning, Performance, Testing, Deployment), consult ≥3 primary sources; note disagreements, choose a default, justify it.

Include publish/update dates; avoid pre-.NET 8 content unless flagged as legacy.

Final Checks

Builds on .NET 8; no obsolete APIs.

Every guideline includes a code sample, a checklist, or both.

Document is self-contained and immediately actionable.