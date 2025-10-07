# Health Checks - .NET 8

> **File Purpose**: Implement health check endpoints for readiness/liveness probes and dependency monitoring
> **Prerequisites**: `structured-logging.md` - Logging configured
> **Related Files**: `opentelemetry.md`, `../10-deployment/kubernetes.md`
> **Agent Use Case**: Reference when implementing health checks for Kubernetes, load balancers, and monitoring

## Quick Context

Health checks provide endpoints for monitoring application health and dependencies (database, cache, external services). Essential for Kubernetes liveness/readiness probes, load balancer health checks, and alerting systems.

**Microsoft References**:
- [Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [Health Checks UI](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)

## Install Packages

```bash
# Core health checks
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks

# Health check UI
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage

# Specific health checks
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.Redis
dotnet add package AspNetCore.HealthChecks.Rabbitmq
dotnet add package AspNetCore.HealthChecks.Elasticsearch
dotnet add package AspNetCore.HealthChecks.Npgsql
```

## Basic Health Check

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health");

app.Run();
```

**Response**:
```json
{
  "status": "Healthy"
}
```

## Dependency Health Checks

### Database Health Check

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "sql-server",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "sql" }
    )
    .AddDbContextCheck<ApplicationDbContext>(
        name: "ef-core",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "ef" }
    );
```

### Redis Health Check

```csharp
builder.Services.AddHealthChecks()
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "cache", "redis" }
    );
```

### Custom Health Check

```csharp
// MyApi.Api/HealthChecks/ExternalApiHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public ExternalApiHealthCheck(
        IHttpClientFactory httpClientFactory,
        ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var client = _httpClientFactory.CreateClient();
            var response = await client.GetAsync(
                "https://api.example.com/health",
                cancellationToken
            );

            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy("External API is healthy");
            }

            return HealthCheckResult.Degraded(
                $"External API returned {response.StatusCode}"
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed");

            return HealthCheckResult.Unhealthy(
                "External API is unreachable",
                ex
            );
        }
    }
}

// Register
builder.Services.AddHealthChecks()
    .AddCheck<ExternalApiHealthCheck>(
        "external-api",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "external", "api" }
    );
```

## Liveness and Readiness Probes

```csharp
var builder = WebApplication.CreateBuilder(args);

// Liveness: Is the application running?
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" });

// Readiness: Can the application serve requests?
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready", "db" }
    )
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "cache",
        tags: new[] { "ready", "cache" }
    )
    .AddCheck<ExternalApiHealthCheck>(
        "external-api",
        tags: new[] { "ready", "external" }
    );

var app = builder.Build();

// Liveness endpoint - lightweight, always returns healthy if app is running
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

// Readiness endpoint - checks all dependencies
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

// Detailed health endpoint with JSON response
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.Run();
```

## Detailed Health Check Response

```csharp
using HealthChecks.UI.Client;
using Microsoft.Extensions.Diagnostics.HealthChecks;

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});
```

**Response**:
```json
{
  "status": "Healthy",
  "totalDuration": "00:00:00.1234567",
  "entries": {
    "sql-server": {
      "status": "Healthy",
      "duration": "00:00:00.0567890",
      "tags": ["db", "sql"]
    },
    "redis": {
      "status": "Healthy",
      "duration": "00:00:00.0123456",
      "tags": ["cache", "redis"]
    },
    "external-api": {
      "status": "Degraded",
      "duration": "00:00:00.0555555",
      "description": "External API returned 429",
      "tags": ["external", "api"]
    }
  }
}
```

## Health Checks UI

### Configuration

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!);

builder.Services
    .AddHealthChecksUI(options =>
    {
        options.SetEvaluationTimeInSeconds(30); // Check every 30 seconds
        options.MaximumHistoryEntriesPerEndpoint(50);
        options.AddHealthCheckEndpoint("My API", "/health");
    })
    .AddInMemoryStorage();

var app = builder.Build();

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Health check UI endpoints
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
    options.ApiPath = "/health-ui-api";
});

app.Run();
```

Access UI at: `https://localhost:7001/health-ui`

### appsettings.json for UI

```json
{
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "My API",
        "Uri": "https://localhost:7001/health"
      },
      {
        "Name": "External Service",
        "Uri": "https://api.external.com/health"
      }
    ],
    "EvaluationTimeinSeconds": 30,
    "MinimumSecondsBetweenFailureNotifications": 60
  }
}
```

## Advanced Health Check Patterns

### Startup Health Check

```csharp
// Delay health checks until app is fully initialized
public class StartupHealthCheck : IHealthCheck
{
    private volatile bool _isReady;

    public void SetReady() => _isReady = true;

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        return Task.FromResult(
            _isReady
                ? HealthCheckResult.Healthy("Startup complete")
                : HealthCheckResult.Unhealthy("Still starting")
        );
    }
}

// Register
builder.Services.AddSingleton<StartupHealthCheck>();
builder.Services.AddHealthChecks()
    .AddCheck<StartupHealthCheck>("startup", tags: new[] { "ready" });

// Set ready after initialization
var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    // Perform initialization
    var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await dbContext.Database.MigrateAsync();

    // Mark as ready
    var startupCheck = scope.ServiceProvider.GetRequiredService<StartupHealthCheck>();
    startupCheck.SetReady();
}
```

### Degraded State

```csharp
public class CacheHealthCheck : IHealthCheck
{
    private readonly IDistributedCache _cache;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _cache.SetStringAsync("health-check", "ok", cancellationToken);
            var value = await _cache.GetStringAsync("health-check", cancellationToken);

            if (value == "ok")
            {
                return HealthCheckResult.Healthy("Cache is working");
            }

            return HealthCheckResult.Degraded("Cache returned unexpected value");
        }
        catch (Exception ex)
        {
            // Return Degraded instead of Unhealthy - cache is optional
            return HealthCheckResult.Degraded(
                "Cache is unavailable, using fallback",
                ex
            );
        }
    }
}
```

## Complete Production Example

```csharp
// Program.cs
using HealthChecks.UI.Client;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Health checks
builder.Services.AddSingleton<StartupHealthCheck>();

builder.Services.AddHealthChecks()
    // Liveness - simple check
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    // Readiness - dependencies
    .AddCheck<StartupHealthCheck>("startup", tags: new[] { "ready" })
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "ready", "db" },
        timeout: TimeSpan.FromSeconds(5)
    )
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis-cache",
        failureStatus: HealthStatus.Degraded, // Degraded - cache is optional
        tags: new[] { "ready", "cache" }
    )
    .AddCheck<ExternalApiHealthCheck>(
        "payment-api",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "ready", "external" }
    );

// Health checks UI
builder.Services
    .AddHealthChecksUI(options =>
    {
        options.SetEvaluationTimeInSeconds(30);
        options.MaximumHistoryEntriesPerEndpoint(100);
        options.AddHealthCheckEndpoint("API Health", "/health");
    })
    .AddInMemoryStorage();

var app = builder.Build();

// Initialize and mark ready
using (var scope = app.Services.CreateScope())
{
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();

    try
    {
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        await dbContext.Database.MigrateAsync();

        var startupCheck = scope.ServiceProvider.GetRequiredService<StartupHealthCheck>();
        startupCheck.SetReady();

        logger.LogInformation("Application initialized successfully");
    }
    catch (Exception ex)
    {
        logger.LogCritical(ex, "Application initialization failed");
        throw;
    }
}

// Health check endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    AllowCachingResponses = false
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    AllowCachingResponses = false,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});

// Health checks UI
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
});

app.Run();
```

## Kubernetes Configuration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapi
        image: myapi:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

## Health Check Best Practices Checklist

- [ ] Liveness probe checks only if app is running (lightweight)
- [ ] Readiness probe checks all critical dependencies
- [ ] Non-critical dependencies marked as Degraded, not Unhealthy
- [ ] Health checks have appropriate timeouts
- [ ] Sensitive information not exposed in health responses
- [ ] Health endpoints excluded from authentication
- [ ] Health checks don't cause cascading failures
- [ ] Startup checks delay readiness until initialization complete
- [ ] Health check UI configured for monitoring dashboard
- [ ] Kubernetes probes configured correctly

## Navigation

- **Previous**: `opentelemetry.md` - Distributed tracing
- **Next**: `../08-performance/async-patterns.md` - Performance optimization
- **Related**: `../10-deployment/kubernetes.md` - Kubernetes deployment

## References

- [Health checks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks)
- [AspNetCore.Diagnostics.HealthChecks](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)
- [Kubernetes Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Health Check Response Format for HTTP APIs](https://datatracker.ietf.org/doc/html/draft-inadarei-api-health-check)
