# References - .NET 8

> **File Purpose**: Comprehensive list of primary sources, Microsoft Docs, OWASP, and RFC specifications
> **Prerequisites**: None - standalone reference
> **Related Files**: All guideline files
> **Agent Use Case**: Reference when citing sources or finding authoritative documentation

## Quick Context

This file contains all primary sources cited throughout the .NET 8 API guidelines. All URLs and dates are current as of October 2025.

## Microsoft Official Documentation

### .NET 8 and C# 12
- [.NET 8 Release Notes](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8) (November 2023)
- [C# 12 What's New](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12) (November 2023)
- [Async Programming](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) (January 2024)
- [Task-based Asynchronous Pattern](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) (December 2023)
- [Cancellation in Managed Threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) (November 2023)
- [IAsyncEnumerable](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream) (October 2023)

### ASP.NET Core 8
- [ASP.NET Core 8 Overview](https://learn.microsoft.com/en-us/aspnet/core/introduction-to-aspnet-core) (December 2023)
- [Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis) (December 2023)
- [Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/) (November 2023)
- [Dependency Injection](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) (October 2023)
- [Configuration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/) (November 2023)

### Authentication & Authorization
- [Authentication in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/) (December 2023)
- [JWT Bearer Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn) (November 2023)
- [Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/introduction) (October 2023)
- [Policy-based Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies) (November 2023)

### API Security & Protection
- [HTTPS Enforcement](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl) (October 2023)
- [CORS](https://learn.microsoft.com/en-us/aspnet/core/security/cors) (November 2023)
- [Rate Limiting](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) (November 2023)
- [Data Protection](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/introduction) (September 2023)

### Entity Framework Core 8
- [EF Core 8 Overview](https://learn.microsoft.com/en-us/ef/core/) (November 2023)
- [DbContext Configuration](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/) (October 2023)
- [Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) (November 2023)
- [Connection Resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) (September 2023)
- [Query Performance](https://learn.microsoft.com/en-us/ef/core/performance/) (December 2023)

### Performance & Caching
- [Performance Best Practices](https://learn.microsoft.com/en-us/aspnet/core/performance/performance-best-practices) (November 2023)
- [Output Caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) (November 2023)
- [Distributed Caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) (December 2023)
- [In-Memory Caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory) (November 2023)
- [Response Caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response) (October 2023)

### Resilience & HttpClient
- [HTTP Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/) (November 2023)
- [HttpClientFactory](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory) (December 2023)
- [Circuit Breaker](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience#circuit-breaker) (October 2023)

### Observability
- [Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging) (December 2023)
- [Diagnostics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) (December 2023)
- [Health Checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) (November 2023)
- [OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) (October 2023)

### Testing
- [Testing in .NET](https://learn.microsoft.com/en-us/dotnet/core/testing/) (October 2023)
- [Integration Tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) (November 2023)
- [Unit Testing Best Practices](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices) (September 2023)

### Deployment & Containers
- [Docker with .NET](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container) (December 2023)
- [Container Images](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/container-docker-introduction/docker-containers-images-registries) (November 2023)
- [Kubernetes Deployment](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/deploy-application-kubernetes) (October 2023)

## .NET Blog Posts

- [Announcing .NET 8](https://devblogs.microsoft.com/dotnet/announcing-dotnet-8/) (November 2023)
- [ASP.NET Core 8 Features](https://devblogs.microsoft.com/dotnet/announcing-asp-net-core-in-dotnet-8/) (November 2023)
- [Performance Improvements in .NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/) (November 2023)
- [Understanding ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) (March 2023)
- [Output Caching in .NET 8](https://devblogs.microsoft.com/dotnet/announcing-asp-net-core-in-dotnet-8/#output-caching) (November 2023)
- [Building Resilient Cloud Services with .NET 8](https://devblogs.microsoft.com/dotnet/building-resilient-cloud-services-with-dotnet-8/) (November 2023)

## OWASP (Open Web Application Security Project)

- [OWASP API Security Top 10 (2023)](https://owasp.org/www-project-api-security/) (2023)
  - API1:2023 - Broken Object Level Authorization
  - API2:2023 - Broken Authentication
  - API3:2023 - Broken Object Property Level Authorization
  - API4:2023 - Unrestricted Resource Consumption
  - API5:2023 - Broken Function Level Authorization
  - API6:2023 - Unrestricted Access to Sensitive Business Flows
  - API7:2023 - Server Side Request Forgery
  - API8:2023 - Security Misconfiguration
  - API9:2023 - Improper Inventory Management
  - API10:2023 - Unsafe Consumption of APIs

- [OWASP Application Security Verification Standard (ASVS)](https://owasp.org/www-project-application-security-verification-standard/) (2021)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) (2024)
  - [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
  - [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
  - [Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
  - [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## IETF RFCs (Internet Engineering Task Force)

- [RFC 7807 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807) (March 2016)
- [RFC 7519 - JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519) (May 2015)
- [RFC 6749 - OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749) (October 2012)
- [RFC 8414 - OAuth 2.0 Authorization Server Metadata](https://www.rfc-editor.org/rfc/rfc8414) (June 2018)
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://www.rfc-editor.org/rfc/rfc7231) (June 2014)
- [RFC 7232 - HTTP/1.1 Conditional Requests](https://www.rfc-editor.org/rfc/rfc7232) (June 2014)
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) (June 2022)

## Third-Party Libraries & Tools

### BenchmarkDotNet
- [BenchmarkDotNet Documentation](https://benchmarkdotnet.org/) (2024)
- [BenchmarkDotNet Best Practices](https://benchmarkdotnet.org/articles/guides/good-practices.html) (2024)

### Serilog
- [Serilog Documentation](https://serilog.net/) (2024)
- [Serilog ASP.NET Core](https://github.com/serilog/serilog-aspnetcore) (2024)

### FluentValidation
- [FluentValidation Documentation](https://docs.fluentvalidation.net/) (2024)

### MediatR
- [MediatR GitHub](https://github.com/jbogard/MediatR) (2024)

### Polly
- [Polly Documentation](https://www.pollydocs.org/) (2024)

### StackExchange.Redis
- [StackExchange.Redis Documentation](https://stackexchange.github.io/StackExchange.Redis/) (2024)

### Testcontainers
- [Testcontainers for .NET](https://dotnet.testcontainers.org/) (2024)

## Docker & Kubernetes

- [Docker Documentation](https://docs.docker.com/) (2024)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) (2024)
- [Kubernetes Documentation](https://kubernetes.io/docs/) (2024)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/) (2024)

## CI/CD

- [GitHub Actions Documentation](https://docs.github.com/en/actions) (2024)
- [Azure DevOps Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/) (December 2023)
- [Docker Build Push Action](https://github.com/docker/build-push-action) (2024)
- [Kubernetes Deploy Action](https://github.com/Azure/k8s-deploy) (2024)

## Architecture & Design Patterns

- [Clean Architecture (Robert C. Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) (August 2012)
- [Domain-Driven Design Reference (Eric Evans)](https://www.domainlanguage.com/ddd/reference/) (2015)
- [Microsoft Architecture Guides](https://learn.microsoft.com/en-us/dotnet/architecture/) (2024)
- [.NET Microservices Architecture](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/) (November 2023)

## Community Resources

- [.NET Foundation](https://dotnetfoundation.org/) (2024)
- [ASP.NET Community Standup](https://www.youtube.com/playlist?list=PL1rZQsJPBU2StolNg0aqvQswETPcYnNKL) (Ongoing)
- [Weekly .NET Newsletter](https://www.dotnetweekly.com/) (Ongoing)

## Standards & Specifications

- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/v3.1.0) (February 2021)
- [JSON Schema](https://json-schema.org/) (2024)
- [REST API Design Rulebook](https://www.oreilly.com/library/view/rest-api-design/9781449317904/) (2011)

## Cloud Provider Specific

### Azure
- [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/) (December 2023)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/) (November 2023)
- [Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/) (December 2023)
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/) (October 2023)

### AWS
- [AWS Elastic Container Service (ECS)](https://docs.aws.amazon.com/ecs/) (2024)
- [AWS Elastic Kubernetes Service (EKS)](https://docs.aws.amazon.com/eks/) (2024)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/) (2024)

## Version Control

All URLs and dates verified as of: **October 6, 2025**

## Citation Format

Throughout this guide, sources are cited using the following format:

```
**Source**: [Title](URL) (Month Year)
```

Example:
```
**Source**: [Microsoft Docs - Async/await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) (January 2024)
```

## Checklist for Source Verification

- [ ] All Microsoft Learn URLs use `learn.microsoft.com` (not deprecated `docs.microsoft.com`)
- [ ] All dates include Month and Year
- [ ] OWASP links point to current versions (2023 for API Top 10)
- [ ] RFC links use `rfc-editor.org` canonical URLs
- [ ] Third-party library links verified and active
- [ ] GitHub links point to official repositories
- [ ] Blog posts from authoritative sources (.NET Blog, official team members)

---

**Next Steps**: Review `checklists.md` for production readiness validation, or `code-artifacts.md` for copy-paste code examples.
