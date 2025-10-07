# Resilience - .NET 8

> **File Purpose**: Implement HttpClientFactory + Polly for retry, circuit breaker, timeout, and fallback patterns
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Basic API setup
> **Related Files**: `async-patterns.md`, `caching.md`, `../03-infrastructure/connection-management.md`
> **Agent Use Case**: Reference when implementing resilient HTTP calls, handling transient failures, and preventing cascade failures

## Quick Context

Resilience patterns protect your API from failures in downstream services (databases, external APIs, queues). .NET 8 uses **HttpClientFactory** with **Microsoft.Extensions.Resilience** for retry logic, circuit breakers, timeouts, and fallbacks. This prevents cascading failures and improves overall system reliability.

## HttpClientFactory Setup

### Basic Configuration

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Named client
builder.Services.AddHttpClient("ExternalApi", client =>
{
    client.BaseAddress = new Uri("https://api.external.com");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
});

// Typed client (recommended)
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ExternalApi:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

**Source**: [Microsoft Docs - HttpClientFactory](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory) (December 2023)

### Typed Client Pattern

```csharp
public interface IExternalApiClient
{
    Task<ExternalData> GetDataAsync(string id, CancellationToken ct);
}

public class ExternalApiClient : IExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiClient> _logger;

    public ExternalApiClient(HttpClient httpClient, ILogger<ExternalApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<ExternalData> GetDataAsync(string id, CancellationToken ct)
    {
        var response = await _httpClient.GetAsync($"/api/data/{id}", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<ExternalData>(ct)
            ?? throw new InvalidOperationException("Response was null");
    }
}
```

## Polly Resilience Policies (.NET 8)

### Install Packages

```bash
# Modern resilience extensions (.NET 8+)
dotnet add package Microsoft.Extensions.Http.Resilience
```

### Standard Resilience Handler

```csharp
// Program.cs - comprehensive resilience
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .AddStandardResilienceHandler(options =>
    {
        // Retry configuration
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromSeconds(1);
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        options.Retry.UseJitter = true;

        // Circuit breaker configuration
        options.CircuitBreaker.FailureRatio = 0.5;
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        options.CircuitBreaker.MinimumThroughput = 10;
        options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);

        // Timeout configuration
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
    });
```

**Source**: [Microsoft Docs - HTTP resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/) (November 2023)

### Custom Retry Policy

```csharp
builder.Services.AddHttpClient<IPaymentApiClient, PaymentApiClient>()
    .AddResilienceHandler("payment-retry", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(2),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .Handle<TimeoutException>()
                .HandleResult(response =>
                    response.StatusCode >= HttpStatusCode.InternalServerError ||
                    response.StatusCode == HttpStatusCode.RequestTimeout),
            OnRetry = args =>
            {
                var logger = args.Context.ServiceProvider.GetRequiredService<ILogger<Program>>();
                logger.LogWarning("Retry {Attempt} after {Delay}ms",
                    args.AttemptNumber, args.RetryDelay.TotalMilliseconds);
                return ValueTask.CompletedTask;
            }
        });
    });
```

### Circuit Breaker Pattern

```csharp
builder.Services.AddHttpClient<IPaymentApiClient, PaymentApiClient>()
    .AddResilienceHandler("circuit-breaker", builder =>
    {
        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5, // Break if 50% of requests fail
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(30),
            OnOpened = args =>
            {
                var logger = args.Context.ServiceProvider.GetRequiredService<ILogger<Program>>();
                logger.LogError("Circuit breaker OPENED - too many failures");
                return ValueTask.CompletedTask;
            }
        });
    });
```

**Source**: [Microsoft Docs - Circuit breaker](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience#circuit-breaker) (October 2023)

## Database Resilience

### EF Core with Retry Policy

```csharp
// SQL Server with connection resiliency
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        }));

// PostgreSQL with retry
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorCodesToAdd: null);
        }));
```

**Source**: [Microsoft Docs - Connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) (September 2023)

## Bulkhead Pattern

### Isolating Downstream Failures

```csharp
// Separate HttpClient for each external dependency
builder.Services.AddHttpClient<IPaymentApiClient, PaymentApiClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.CircuitBreaker.BreakDuration = TimeSpan.FromMinutes(1);
    });

builder.Services.AddHttpClient<IShippingApiClient, ShippingApiClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.CircuitBreaker.BreakDuration = TimeSpan.FromMinutes(5);
    });

// Different policies prevent cascading failures
```

## Checklist

- [ ] HttpClientFactory used for all HTTP calls (never `new HttpClient()`)
- [ ] Retry policy configured with exponential backoff and jitter
- [ ] Circuit breaker prevents cascade failures to unhealthy dependencies
- [ ] Timeout policies prevent indefinite waits
- [ ] Database connections use EF Core retry on failure
- [ ] Separate resilience policies per external dependency (bulkhead)
- [ ] Circuit breaker state monitored via health checks
- [ ] Resilience events logged for debugging
- [ ] CancellationTokens propagated through resilience pipelines

## References

- [Microsoft Docs - HTTP resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/) (November 2023)
- [Microsoft Docs - HttpClientFactory](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory) (December 2023)
- [Microsoft Docs - Circuit breaker](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience#circuit-breaker) (October 2023)
- [Microsoft Docs - EF Core connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) (September 2023)
- [Polly Documentation](https://www.pollydocs.org/) (2024)
- [.NET Blog - Resilience in .NET 8](https://devblogs.microsoft.com/dotnet/building-resilient-cloud-services-with-dotnet-8/) (November 2023)

---

**Next Steps**: Review `async-patterns.md` for CancellationToken usage, or `benchmarking.md` to measure resilience impact.
