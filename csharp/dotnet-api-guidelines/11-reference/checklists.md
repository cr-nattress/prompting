# Checklists - .NET 8

> **File Purpose**: Production readiness checklists for security, performance, quality, and deployment
> **Prerequisites**: All previous sections completed
> **Related Files**: All guideline files
> **Agent Use Case**: Reference when validating production readiness before deployment

## Quick Context

Use these checklists to ensure your .NET 8 API meets production standards before deployment. Each checklist corresponds to critical areas: security, performance, observability, quality, and deployment.

## Security Checklist

### Authentication & Authorization
- [ ] JWT Bearer authentication configured with OIDC provider (Auth0/Azure AD)
- [ ] Token expiration set appropriately (15-60 minutes)
- [ ] Refresh tokens implemented for long-lived sessions
- [ ] Authorization policies defined (role-based or claims-based)
- [ ] API endpoints protected with `[Authorize]` or `.RequireAuthorization()`
- [ ] Minimal permissions granted (principle of least privilege)

### API Protection
- [ ] HTTPS enforced in production (HSTS enabled)
- [ ] Rate limiting configured per endpoint/user
- [ ] Request size limits set (default 30MB max)
- [ ] CORS policy defined (not wildcard `*` in production)
- [ ] Security headers set (CSP, X-Frame-Options, X-Content-Type-Options)
- [ ] API keys/secrets stored in environment variables (not appsettings.json)

### OWASP API Security Top 10
- [ ] **API1**: Broken Object Level Authorization - verified with integration tests
- [ ] **API2**: Broken Authentication - JWT/OIDC properly configured
- [ ] **API3**: Broken Object Property Level Authorization - DTOs prevent over-posting
- [ ] **API4**: Unrestricted Resource Consumption - rate limiting + pagination
- [ ] **API5**: Broken Function Level Authorization - endpoint policies enforced
- [ ] **API6**: Unrestricted Access to Sensitive Business Flows - critical flows protected
- [ ] **API7**: Server Side Request Forgery - input validation on URLs
- [ ] **API8**: Security Misconfiguration - production config reviewed
- [ ] **API9**: Improper Inventory Management - API versioning implemented
- [ ] **API10**: Unsafe Consumption of APIs - external APIs validated

### Data Protection
- [ ] Sensitive data encrypted at rest (database encryption)
- [ ] Sensitive data encrypted in transit (TLS 1.2+)
- [ ] PII logged with masking/redaction
- [ ] Database connection strings use environment variables
- [ ] Secrets managed with Azure Key Vault/AWS Secrets Manager

**Source**: [OWASP API Security Top 10](https://owasp.org/www-project-api-security/) (2023)

## Performance Checklist

### Async Patterns
- [ ] All I/O operations are async (database, HTTP, file system)
- [ ] CancellationToken accepted and propagated in all async methods
- [ ] No blocking calls (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`)
- [ ] No `async void` methods (except event handlers)
- [ ] ValueTask used for hot paths with synchronous completion
- [ ] IAsyncEnumerable used for streaming large datasets

### Caching
- [ ] Output caching configured for GET endpoints
- [ ] Distributed cache (Redis) configured with connection resilience
- [ ] Cache-aside pattern implemented with proper error handling
- [ ] Cache invalidation strategy defined (time-based or event-based)
- [ ] Cache keys follow consistent naming convention
- [ ] Response caching headers (Cache-Control, ETag) set

### Database
- [ ] EF Core queries use `.AsNoTracking()` for read-only data
- [ ] Indexes created for frequently queried columns
- [ ] N+1 queries eliminated (`.Include()` or compiled queries)
- [ ] Connection pooling enabled
- [ ] EF Core retry on failure configured
- [ ] Query performance benchmarked with BenchmarkDotNet

### Resilience
- [ ] HttpClientFactory used for all HTTP calls
- [ ] Polly retry policy configured with exponential backoff
- [ ] Circuit breaker prevents cascade failures
- [ ] Timeout policies prevent indefinite waits
- [ ] Separate resilience policies per external dependency

**Source**: [Microsoft Docs - Performance](https://learn.microsoft.com/en-us/aspnet/core/performance/) (November 2023)

## Observability Checklist

### Structured Logging
- [ ] Serilog configured with structured logging
- [ ] Log levels set appropriately (Information for production)
- [ ] Correlation IDs logged for request tracing
- [ ] Sensitive data excluded from logs (passwords, tokens, PII)
- [ ] Request/response logging middleware configured
- [ ] Logs exported to centralized system (Seq, Application Insights)

### Tracing
- [ ] OpenTelemetry configured for distributed tracing
- [ ] Traces exported to backend (Jaeger, Zipkin, Azure Monitor)
- [ ] Custom spans created for critical operations
- [ ] Trace context propagated across services

### Metrics
- [ ] Application metrics exposed (Prometheus format)
- [ ] Custom meters for business metrics
- [ ] ASP.NET Core metrics enabled
- [ ] Database query metrics tracked

### Health Checks
- [ ] Liveness probe endpoint `/health/live` (process is running)
- [ ] Readiness probe endpoint `/health/ready` (dependencies healthy)
- [ ] Startup probe endpoint `/health/startup` (application started)
- [ ] Database health check registered
- [ ] Redis health check registered (if used)
- [ ] Health check UI configured for monitoring

**Source**: [Microsoft Docs - Observability](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) (December 2023)

## Quality Checklist

### Testing
- [ ] Unit tests cover business logic (>80% coverage)
- [ ] Integration tests cover critical user flows
- [ ] WebApplicationFactory used for integration tests
- [ ] Testcontainers used for database/Redis in tests
- [ ] Tests run in CI pipeline
- [ ] Code coverage reported and tracked

### Code Quality
- [ ] .editorconfig enforces style rules
- [ ] Nullable reference types enabled (`<Nullable>enable</Nullable>`)
- [ ] Code analyzers enabled (Microsoft.CodeAnalysis.NetAnalyzers)
- [ ] Static code analysis passes (no warnings in Release build)
- [ ] FluentValidation used for complex validation rules
- [ ] Domain models validate invariants in constructors

### Error Handling
- [ ] Global exception middleware configured
- [ ] ProblemDetails (RFC 7807) returned for all errors
- [ ] Validation errors return 400 with field-level details
- [ ] Correlation IDs included in error responses
- [ ] Sensitive details excluded from error messages

### API Design
- [ ] RESTful conventions followed (correct HTTP verbs/status codes)
- [ ] API versioning implemented (URL or header-based)
- [ ] OpenAPI/Swagger documentation generated
- [ ] Pagination implemented for list endpoints
- [ ] Filtering and sorting supported where appropriate
- [ ] Idempotency keys supported for critical operations

**Source**: [Microsoft Docs - Testing](https://learn.microsoft.com/en-us/dotnet/core/testing/) (October 2023)

## Deployment Checklist

### Docker
- [ ] Multi-stage Dockerfile separates build and runtime
- [ ] Alpine or chiseled base image used (smaller size)
- [ ] Non-root user used in runtime
- [ ] .dockerignore excludes unnecessary files
- [ ] Health check endpoint configured in Dockerfile
- [ ] Security updates installed in image
- [ ] Image scanned for vulnerabilities (Trivy/Snyk)

### Kubernetes
- [ ] Deployment with 3+ replicas for high availability
- [ ] Rolling update strategy for zero-downtime
- [ ] Liveness, readiness, and startup probes configured
- [ ] Resource requests and limits set
- [ ] ConfigMap for non-sensitive configuration
- [ ] Secrets for sensitive data (base64 encoded)
- [ ] Horizontal Pod Autoscaler configured
- [ ] Ingress with TLS termination
- [ ] Network policies restrict traffic
- [ ] Read-only root filesystem enabled

### CI/CD
- [ ] CI workflow runs on every push and PR
- [ ] Unit and integration tests run automatically
- [ ] Code coverage tracked and reported
- [ ] Security scanning integrated
- [ ] Docker images built and pushed to registry
- [ ] Automated deployment to staging
- [ ] Manual approval required for production
- [ ] Health checks after deployment
- [ ] Automatic rollback on deployment failure

**Source**: [Microsoft Docs - Deployment](https://learn.microsoft.com/en-us/dotnet/core/docker/) (December 2023)

## Configuration Checklist

### Environment Configuration
- [ ] `appsettings.json` for default configuration
- [ ] `appsettings.Production.json` for production overrides
- [ ] Secrets stored in environment variables (not in appsettings)
- [ ] Configuration validated on startup
- [ ] Configuration hot-reloaded when possible

### Production Settings
- [ ] `ASPNETCORE_ENVIRONMENT=Production`
- [ ] Detailed errors disabled (`UseDeveloperExceptionPage` removed)
- [ ] HTTPS redirection enabled
- [ ] HSTS enabled with appropriate max-age
- [ ] Log level set to Information or Warning
- [ ] Connection pooling enabled
- [ ] Graceful shutdown timeout configured (30 seconds)

## Final Pre-Production Checklist

### Before Deployment
- [ ] All tests passing in CI
- [ ] Code coverage meets targets (>80%)
- [ ] Security scan shows no critical vulnerabilities
- [ ] Load testing completed
- [ ] Database migrations tested
- [ ] Rollback plan documented
- [ ] Monitoring dashboards configured
- [ ] On-call rotation scheduled

### After Deployment
- [ ] Health checks passing
- [ ] Logs flowing to centralized system
- [ ] Metrics being collected
- [ ] Alerts configured for critical errors
- [ ] Smoke tests passed
- [ ] User acceptance testing completed

## References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/) (2023)
- [Microsoft Docs - Performance](https://learn.microsoft.com/en-us/aspnet/core/performance/) (November 2023)
- [Microsoft Docs - Observability](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) (December 2023)
- [Microsoft Docs - Testing](https://learn.microsoft.com/en-us/dotnet/core/testing/) (October 2023)
- [Microsoft Docs - Deployment](https://learn.microsoft.com/en-us/dotnet/core/docker/) (December 2023)

---

**Next Steps**: Review `code-artifacts.md` for complete code examples, or `references.md` for all primary source citations.
