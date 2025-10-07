# Caching - .NET 8

> **File Purpose**: Implement output caching, distributed caching, and invalidation strategies
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Basic API setup
> **Related Files**: `async-patterns.md`, `resilience.md`, `../03-infrastructure/connection-management.md`
> **Agent Use Case**: Reference when implementing Redis, output caching, cache-aside patterns, and invalidation

## Quick Context

Caching is critical for reducing latency, database load, and external API calls. .NET 8 introduces native **output caching** middleware for HTTP responses, complementing existing **distributed caching** (Redis, SQL Server) and **in-memory caching**. This guide covers cache policies, invalidation strategies, and production patterns.

## Output Caching (.NET 8)

### Basic Setup

```csharp
// Program.cs - register output caching
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder =>
        builder.Expire(TimeSpan.FromSeconds(10)));

    // Named policy
    options.AddPolicy("Expire60", builder =>
        builder.Expire(TimeSpan.FromSeconds(60)));

    // Policy with cache key variation
    options.AddPolicy("VaryByQuery", builder =>
        builder
            .Expire(TimeSpan.FromMinutes(5))
            .SetVaryByQuery("page", "size"));

    // Policy with authentication awareness
    options.AddPolicy("PerUser", builder =>
        builder
            .Expire(TimeSpan.FromMinutes(10))
            .SetVaryByHeader("Authorization"));
});

var app = builder.Build();

app.UseOutputCache(); // Add middleware

// Apply to endpoint
app.MapGet("/products", async (IProductRepository repo) =>
{
    var products = await repo.GetAllAsync();
    return Results.Ok(products);
})
.CacheOutput("Expire60");

// Vary by route parameter
app.MapGet("/products/{id:int}", async (int id, IProductRepository repo) =>
{
    var product = await repo.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
})
.CacheOutput(builder => builder
    .Expire(TimeSpan.FromMinutes(15))
    .SetVaryByRouteValue("id"));

app.Run();
```

**Source**: [Microsoft Docs - Output caching middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) (November 2023)

### Cache Invalidation

```csharp
// Tag-based eviction
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("ProductCache", builder =>
        builder
            .Expire(TimeSpan.FromMinutes(30))
            .Tag("products"));
});

// Endpoint with tagging
app.MapGet("/products", async (IProductRepository repo) =>
{
    var products = await repo.GetAllAsync();
    return Results.Ok(products);
})
.CacheOutput("ProductCache");

// Invalidate cache on update
app.MapPut("/products/{id:int}", async (
    int id,
    UpdateProductDto dto,
    IProductRepository repo,
    IOutputCacheStore cache) =>
{
    var product = await repo.GetByIdAsync(id);
    if (product is null) return Results.NotFound();

    product.Update(dto.Name, dto.Price);
    await repo.UpdateAsync(product);

    // Evict all cached responses tagged "products"
    await cache.EvictByTagAsync("products", default);

    return Results.NoContent();
});
```

### Custom Cache Key Generation

```csharp
public class TenantCachePolicy : IOutputCachePolicy
{
    public ValueTask CacheRequestAsync(
        OutputCacheContext context,
        CancellationToken cancellationToken)
    {
        var attemptOutputCaching = AttemptOutputCaching(context);
        context.EnableOutputCaching = true;
        context.AllowCacheLookup = attemptOutputCaching;
        context.AllowCacheStorage = attemptOutputCaching;
        context.AllowLocking = true;

        // Vary by tenant ID from claims
        var tenantId = context.HttpContext.User.FindFirst("tenant_id")?.Value;
        context.CacheVaryByRules.VaryByValues.Add("tenant", tenantId ?? "default");

        context.ResponseExpirationTimeSpan = TimeSpan.FromMinutes(10);

        return ValueTask.CompletedTask;
    }

    private static bool AttemptOutputCaching(OutputCacheContext context)
    {
        // Only cache GET/HEAD
        var request = context.HttpContext.Request;
        if (!HttpMethods.IsGet(request.Method) &&
            !HttpMethods.IsHead(request.Method))
        {
            return false;
        }

        return true;
    }
}

// Register custom policy
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("PerTenant", new TenantCachePolicy());
});
```

**Source**: [Microsoft Docs - Output caching policies](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output#policies) (October 2023)

## Distributed Caching (Redis)

### Redis Setup

```bash
# Install Redis package
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs - configure Redis
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:"; // Key prefix
    options.ConfigurationOptions = new ConfigurationOptions
    {
        ConnectTimeout = 5000,
        ConnectRetry = 3,
        AbortOnConnectFail = false, // Don't fail startup if Redis is down
        Password = builder.Configuration["Redis:Password"]
    };
});

// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,ssl=false"
  }
}
```

### Cache-Aside Pattern

```csharp
public class ProductService
{
    private readonly IProductRepository _repository;
    private readonly IDistributedCache _cache;
    private readonly ILogger<ProductService> _logger;

    public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";

        // 1. Try cache first
        var cached = await _cache.GetStringAsync(cacheKey, ct);
        if (cached is not null)
        {
            _logger.LogDebug("Cache hit for {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<Product>(cached);
        }

        // 2. Cache miss - load from database
        _logger.LogDebug("Cache miss for {CacheKey}", cacheKey);
        var product = await _repository.GetByIdAsync(id, ct);

        if (product is not null)
        {
            // 3. Store in cache
            var options = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15),
                SlidingExpiration = TimeSpan.FromMinutes(5)
            };

            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(product),
                options,
                ct);
        }

        return product;
    }

    public async Task UpdateProductAsync(
        int id,
        UpdateProductDto dto,
        CancellationToken ct)
    {
        var product = await _repository.GetByIdAsync(id, ct);
        if (product is null) throw new NotFoundException("Product not found");

        product.Update(dto.Name, dto.Price);
        await _repository.UpdateAsync(product, ct);

        // Invalidate cache
        await _cache.RemoveAsync($"product:{id}", ct);
    }
}
```

### Typed Cache Helper

```csharp
public static class DistributedCacheExtensions
{
    public static async Task<T?> GetAsync<T>(
        this IDistributedCache cache,
        string key,
        CancellationToken ct = default)
    {
        var data = await cache.GetStringAsync(key, ct);
        return data is null ? default : JsonSerializer.Deserialize<T>(data);
    }

    public static async Task SetAsync<T>(
        this IDistributedCache cache,
        string key,
        T value,
        DistributedCacheEntryOptions? options = null,
        CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(value);
        await cache.SetStringAsync(key, json, options ?? new(), ct);
    }

    public static async Task<T> GetOrCreateAsync<T>(
        this IDistributedCache cache,
        string key,
        Func<Task<T>> factory,
        DistributedCacheEntryOptions? options = null,
        CancellationToken ct = default)
    {
        var cached = await cache.GetAsync<T>(key, ct);
        if (cached is not null) return cached;

        var value = await factory();
        await cache.SetAsync(key, value, options, ct);
        return value;
    }
}

// Usage
public class ProductService
{
    public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async () => await _repository.GetByIdAsync(id, ct),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
            },
            ct);
    }
}
```

**Source**: [Microsoft Docs - Distributed caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) (December 2023)

## In-Memory Caching

### IMemoryCache Setup

```csharp
// Program.cs
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // Limit entries (not bytes)
    options.CompactionPercentage = 0.25; // Evict 25% when limit reached
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(1);
});
```

### In-Memory Cache Usage

```csharp
public class ConfigurationService
{
    private readonly IMemoryCache _cache;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<ConfigurationService> _logger;

    public async Task<AppConfiguration> GetConfigurationAsync(CancellationToken ct)
    {
        const string cacheKey = "app_configuration";

        // Try get from cache
        if (_cache.TryGetValue(cacheKey, out AppConfiguration? cached))
        {
            return cached!;
        }

        // Load and cache
        var config = await LoadConfigurationAsync(ct);

        var cacheOptions = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(30))
            .SetPriority(CacheItemPriority.High)
            .SetSize(1) // Count towards size limit
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation(
                    "Configuration evicted from cache. Reason: {Reason}",
                    reason);
            });

        _cache.Set(cacheKey, config, cacheOptions);
        return config;
    }

    private async Task<AppConfiguration> LoadConfigurationAsync(CancellationToken ct)
    {
        var client = _httpClientFactory.CreateClient();
        var response = await client.GetAsync("/api/config", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<AppConfiguration>(ct)
            ?? throw new InvalidOperationException("Failed to load configuration");
    }
}
```

### GetOrCreateAsync Pattern

```csharp
public class UserService
{
    private readonly IMemoryCache _cache;
    private readonly IUserRepository _repository;

    public async Task<User?> GetUserAsync(int id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync(
            $"user:{id}",
            async entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
                entry.SlidingExpiration = TimeSpan.FromMinutes(3);
                entry.SetPriority(CacheItemPriority.Normal);

                return await _repository.GetByIdAsync(id, ct);
            });
    }
}
```

**Source**: [Microsoft Docs - In-memory caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory) (November 2023)

## Cache Invalidation Strategies

### 1. Time-Based Expiration

```csharp
// Absolute expiration - expires at specific time
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1)
};

// Relative expiration - expires X time after set
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
};

// Sliding expiration - reset on access
var options = new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(5)
};
```

### 2. Event-Based Invalidation

```csharp
public class ProductEventHandler : INotificationHandler<ProductUpdatedEvent>
{
    private readonly IDistributedCache _cache;
    private readonly IOutputCacheStore _outputCache;

    public async Task Handle(ProductUpdatedEvent notification, CancellationToken ct)
    {
        // Invalidate distributed cache
        await _cache.RemoveAsync($"product:{notification.ProductId}", ct);

        // Invalidate output cache
        await _outputCache.EvictByTagAsync("products", ct);
    }
}
```

### 3. Multi-Level Caching

```csharp
public class MultiLevelCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly IProductRepository _repository;

    public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";

        // L1: Memory cache (fastest)
        if (_memoryCache.TryGetValue(cacheKey, out Product? cached))
        {
            return cached;
        }

        // L2: Distributed cache (Redis)
        var product = await _distributedCache.GetAsync<Product>(cacheKey, ct);
        if (product is not null)
        {
            // Store in L1 for next request
            _memoryCache.Set(cacheKey, product, TimeSpan.FromMinutes(5));
            return product;
        }

        // L3: Database (slowest)
        product = await _repository.GetByIdAsync(id, ct);
        if (product is not null)
        {
            // Store in both caches
            await _distributedCache.SetAsync(cacheKey, product,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                }, ct);

            _memoryCache.Set(cacheKey, product, TimeSpan.FromMinutes(5));
        }

        return product;
    }

    public async Task InvalidateAsync(int id, CancellationToken ct)
    {
        var cacheKey = $"product:{id}";
        _memoryCache.Remove(cacheKey);
        await _distributedCache.RemoveAsync(cacheKey, ct);
    }
}
```

## Response Caching Headers

### Cache-Control Headers

```csharp
// Program.cs
builder.Services.AddResponseCaching();

var app = builder.Build();

app.UseResponseCaching(); // Add before UseOutputCache

// Endpoint with cache headers
app.MapGet("/products/{id:int}", async (
    int id,
    IProductRepository repo,
    HttpContext context) =>
{
    var product = await repo.GetByIdAsync(id);
    if (product is null) return Results.NotFound();

    // Set cache headers
    context.Response.Headers.CacheControl = "public, max-age=300"; // 5 minutes
    context.Response.Headers.ETag = $"\"{product.Version}\"";
    context.Response.Headers.LastModified = product.UpdatedAt.ToString("R");

    return Results.Ok(product);
});

// Conditional request support
app.MapGet("/products/{id:int}", async (
    int id,
    IProductRepository repo,
    HttpContext context) =>
{
    var product = await repo.GetByIdAsync(id);
    if (product is null) return Results.NotFound();

    var etag = $"\"{product.Version}\"";
    var ifNoneMatch = context.Request.Headers.IfNoneMatch.ToString();

    if (ifNoneMatch == etag)
    {
        return Results.StatusCode(StatusCodes.Status304NotModified);
    }

    context.Response.Headers.ETag = etag;
    context.Response.Headers.CacheControl = "public, max-age=300";

    return Results.Ok(product);
});
```

**Source**: [Microsoft Docs - Response caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response) (October 2023)

## Cache Warming

### Background Service for Cache Warming

```csharp
public class CacheWarmingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<CacheWarmingService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken); // Wait for startup

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var cache = scope.ServiceProvider.GetRequiredService<IDistributedCache>();
                var repo = scope.ServiceProvider.GetRequiredService<IProductRepository>();

                _logger.LogInformation("Starting cache warming");

                // Load popular products
                var popularProducts = await repo.GetPopularProductsAsync(50, stoppingToken);

                foreach (var product in popularProducts)
                {
                    await cache.SetAsync(
                        $"product:{product.Id}",
                        product,
                        new DistributedCacheEntryOptions
                        {
                            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
                        },
                        stoppingToken);
                }

                _logger.LogInformation("Cache warming completed. Warmed {Count} products",
                    popularProducts.Count);

                // Wait 1 hour before next warming
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during cache warming");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
        }
    }
}

// Register in Program.cs
builder.Services.AddHostedService<CacheWarmingService>();
```

## Redis Advanced Patterns

### Redis Connection Resilience

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.ConfigurationOptions = new ConfigurationOptions
    {
        EndPoints = { "redis-primary:6379", "redis-replica:6379" },
        ConnectTimeout = 5000,
        ConnectRetry = 3,
        AbortOnConnectFail = false,
        ReconnectRetryPolicy = new ExponentialRetry(5000),
        SyncTimeout = 5000,
        AsyncTimeout = 5000,
        KeepAlive = 60,
        Password = builder.Configuration["Redis:Password"]
    };
});
```

### Redis Pub/Sub for Cache Invalidation

```csharp
public class RedisCacheInvalidationService : BackgroundService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IMemoryCache _memoryCache;
    private readonly ILogger<RedisCacheInvalidationService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var subscriber = _redis.GetSubscriber();

        await subscriber.SubscribeAsync("cache:invalidate", (channel, message) =>
        {
            var key = message.ToString();
            _memoryCache.Remove(key);
            _logger.LogDebug("Invalidated cache key: {Key}", key);
        });

        _logger.LogInformation("Cache invalidation subscriber started");

        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
}

// Publish invalidation event
public class ProductService
{
    private readonly IConnectionMultiplexer _redis;

    public async Task UpdateProductAsync(int id, UpdateProductDto dto, CancellationToken ct)
    {
        // Update product...

        // Notify all instances to invalidate cache
        var subscriber = _redis.GetSubscriber();
        await subscriber.PublishAsync("cache:invalidate", $"product:{id}");
    }
}
```

## Monitoring and Metrics

### Cache Hit Rate Tracking

```csharp
public class CacheMetricsService
{
    private readonly IMemoryCache _cache;
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<CacheMetricsService> _logger;
    private long _hits;
    private long _misses;

    public async Task<T?> GetWithMetricsAsync<T>(
        string key,
        Func<Task<T>> factory,
        CancellationToken ct)
    {
        var cached = await _distributedCache.GetAsync<T>(key, ct);

        if (cached is not null)
        {
            Interlocked.Increment(ref _hits);
            _logger.LogDebug("Cache hit for {Key}. Hit rate: {HitRate:P2}",
                key, GetHitRate());
            return cached;
        }

        Interlocked.Increment(ref _misses);
        _logger.LogDebug("Cache miss for {Key}. Hit rate: {HitRate:P2}",
            key, GetHitRate());

        var value = await factory();
        await _distributedCache.SetAsync(key, value, ct: ct);
        return value;
    }

    private double GetHitRate()
    {
        var total = _hits + _misses;
        return total == 0 ? 0 : (double)_hits / total;
    }
}
```

## Checklist

- [ ] Output caching configured for GET endpoints
- [ ] Cache policies vary by route, query, or user context
- [ ] Distributed cache (Redis) configured with connection resilience
- [ ] Cache-aside pattern implemented with proper error handling
- [ ] Cache invalidation strategy defined (time-based, event-based, or manual)
- [ ] Cache keys follow consistent naming convention
- [ ] Sensitive data not cached or encrypted before caching
- [ ] Cache hit/miss metrics tracked
- [ ] Response caching headers (Cache-Control, ETag) set appropriately
- [ ] Cache size limits configured to prevent memory exhaustion
- [ ] Cache warming implemented for critical data
- [ ] Multi-level caching used for hot data (memory + Redis)

## References

- [Microsoft Docs - Output caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output) (November 2023)
- [Microsoft Docs - Distributed caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed) (December 2023)
- [Microsoft Docs - In-memory caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory) (November 2023)
- [Microsoft Docs - Response caching](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response) (October 2023)
- [StackExchange.Redis Documentation](https://stackexchange.github.io/StackExchange.Redis/) (2024)
- [.NET Blog - Output caching in .NET 8](https://devblogs.microsoft.com/dotnet/announcing-asp-net-core-in-dotnet-8/#output-caching) (November 2023)

---

**Next Steps**: Review `resilience.md` for Polly patterns, or `async-patterns.md` for proper async cache operations.
