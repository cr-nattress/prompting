# Async Patterns - .NET 8

> **File Purpose**: Implement async/await best practices for high-performance, non-blocking APIs
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Basic API setup
> **Related Files**: `resilience.md`, `benchmarking.md`, `../03-infrastructure/connection-management.md`
> **Agent Use Case**: Reference when optimizing async operations, avoiding deadlocks, and minimizing allocations

## Quick Context

Proper async/await patterns are critical for building scalable .NET 8 APIs. Asynchronous code allows threads to process other requests while waiting for I/O operations (database, HTTP, file system), dramatically improving throughput. This guide covers best practices for async methods, CancellationToken propagation, ValueTask optimization, and common pitfalls to avoid.

## Core Async Principles

### The Golden Rules

```csharp
// 1. Async all the way - avoid mixing sync and async
//  GOOD
public async Task<User> GetUserAsync(int id, CancellationToken ct)
{
    return await _dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id, ct);
}

// L BAD - blocking async code
public User GetUser(int id)
{
    return _dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id)
        .Result; // Deadlock risk!
}

// 2. Always accept and propagate CancellationToken
//  GOOD
public async Task<IEnumerable<Product>> GetProductsAsync(CancellationToken ct)
{
    var products = await _dbContext.Products.ToListAsync(ct);
    await _cache.SetAsync("products", products, ct);
    return products;
}

// L BAD - ignores cancellation
public async Task<IEnumerable<Product>> GetProductsAsync()
{
    return await _dbContext.Products.ToListAsync(); // Cannot cancel
}

// 3. Use ConfigureAwait(false) in libraries, NOT in ASP.NET Core
//  GOOD - ASP.NET Core 8 (no context needed)
public async Task<Order> ProcessOrderAsync(Order order, CancellationToken ct)
{
    await _repository.SaveAsync(order, ct);
    await _eventBus.PublishAsync(new OrderCreated(order.Id), ct);
    return order;
}

//   LEGACY - ConfigureAwait(false) was needed in .NET Framework
// .NET 8 minimal APIs don't capture SynchronizationContext
```

**Source**: [Microsoft Docs - Async in depth](https://learn.microsoft.com/en-us/dotnet/csharp/async) (January 2024)

### Async Method Signatures

```csharp
// Task<T> - standard async method with result
public async Task<UserDto> GetUserAsync(int id, CancellationToken ct)
{
    var user = await _repository.GetByIdAsync(id, ct);
    return _mapper.Map<UserDto>(user);
}

// Task - async method without result
public async Task UpdateUserAsync(int id, UpdateUserDto dto, CancellationToken ct)
{
    var user = await _repository.GetByIdAsync(id, ct);
    user.Update(dto.Name, dto.Email);
    await _repository.UpdateAsync(user, ct);
}

// ValueTask<T> - optimized for hot paths (see next section)
public async ValueTask<User?> GetCachedUserAsync(int id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out User? cached))
        return cached; // Synchronous completion - no allocation

    var user = await _repository.GetByIdAsync(id, ct);
    _cache.Set(id, user);
    return user;
}

// IAsyncEnumerable<T> - streaming results
public async IAsyncEnumerable<Product> GetProductsStreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var product in _repository.StreamProductsAsync(ct))
    {
        yield return product;
    }
}
```

**Source**: [Microsoft Docs - Task-based asynchronous pattern](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) (December 2023)

## CancellationToken Patterns

### Proper Token Propagation

```csharp
// Minimal API - automatically injected
app.MapGet("/users/{id:int}", async (
    int id,
    IUserRepository repo,
    CancellationToken ct) => // ASP.NET Core binds from request abort
{
    var user = await repo.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

// Service layer - always accept and pass through
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly IEventPublisher _eventPublisher;

    public async Task<User> CreateUserAsync(CreateUserDto dto, CancellationToken ct)
    {
        var user = new User(dto.Name, dto.Email);

        // Propagate to all async calls
        await _repository.AddAsync(user, ct);
        await _eventPublisher.PublishAsync(new UserCreated(user.Id), ct);

        return user;
    }
}

// Repository layer - pass to EF Core
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context;

    public async Task<User?> GetByIdAsync(int id, CancellationToken ct)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, ct);
    }

    public async Task AddAsync(User user, CancellationToken ct)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync(ct);
    }
}
```

### Timeout with CancellationTokenSource

```csharp
public class ExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly TimeSpan _defaultTimeout = TimeSpan.FromSeconds(30);

    public async Task<ApiResponse> CallExternalApiAsync(
        string endpoint,
        CancellationToken ct)
    {
        // Combine request cancellation with timeout
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(_defaultTimeout);

        try
        {
            var response = await _httpClient.GetAsync(endpoint, cts.Token);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<ApiResponse>(cts.Token);
        }
        catch (OperationCanceledException) when (!ct.IsCancellationRequested)
        {
            // Timeout occurred (not request cancellation)
            throw new TimeoutException($"External API call timed out after {_defaultTimeout}");
        }
    }
}
```

**Source**: [Microsoft Docs - Cancellation in managed threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) (November 2023)

### Graceful Shutdown

```csharp
// Program.cs - configure shutdown timeout
var builder = WebApplication.CreateBuilder(args);

builder.Host.ConfigureHostOptions(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

var app = builder.Build();

// Background service respecting cancellation
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processing service starting");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();

                await orderService.ProcessPendingOrdersAsync(stoppingToken);

                // Wait with cancellation support
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                _logger.LogInformation("Order processing cancelled");
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing orders");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }

        _logger.LogInformation("Order processing service stopped");
    }
}
```

## ValueTask Optimization

### When to Use ValueTask<T>

```csharp
//  GOOD - synchronous completion path (cache hit)
public ValueTask<User?> GetUserAsync(int id, CancellationToken ct)
{
    if (_memoryCache.TryGetValue(id, out User? cached))
        return new ValueTask<User?>(cached); // No Task allocation

    return new ValueTask<User?>(LoadUserFromDbAsync(id, ct));
}

private async Task<User?> LoadUserFromDbAsync(int id, CancellationToken ct)
{
    var user = await _context.Users.FindAsync(new object[] { id }, ct);
    if (user is not null)
        _memoryCache.Set(id, user, TimeSpan.FromMinutes(5));
    return user;
}

//  GOOD - pooled buffer pattern
public async ValueTask<int> ReadDataAsync(Memory<byte> buffer, CancellationToken ct)
{
    return await _stream.ReadAsync(buffer, ct);
}

// L BAD - always asynchronous (use Task<T> instead)
public async ValueTask<List<User>> GetAllUsersAsync(CancellationToken ct)
{
    // Always hits database - no sync path benefit
    return await _context.Users.ToListAsync(ct);
}

// L BAD - awaiting ValueTask multiple times (undefined behavior)
public async Task ProcessUserAsync(int id)
{
    var task = GetUserAsync(id, default); // ValueTask
    var user1 = await task; // First await - OK
    var user2 = await task; // WRONG! ValueTask consumed
}
```

**Rules for ValueTask**:
1. Await only once
2. Don't call `.Result` or `.GetAwaiter().GetResult()`
3. Use when synchronous completion is common (>30% of calls)
4. Default to `Task<T>` when in doubt

**Source**: [Microsoft Docs - Understanding ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) (March 2023)

### Pooled ValueTask Pattern

```csharp
public class BufferPool
{
    private readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;

    public async ValueTask<ProcessResult> ProcessDataAsync(
        Stream source,
        CancellationToken ct)
    {
        byte[] buffer = _pool.Rent(4096);
        try
        {
            int bytesRead = await source.ReadAsync(buffer.AsMemory(0, 4096), ct);
            return ProcessBuffer(buffer.AsSpan(0, bytesRead));
        }
        finally
        {
            _pool.Return(buffer);
        }
    }

    private ProcessResult ProcessBuffer(ReadOnlySpan<byte> data)
    {
        // Process data without allocations
        return new ProcessResult { BytesProcessed = data.Length };
    }
}
```

## Async Antipatterns to Avoid

### 1. Blocking on Async Code

```csharp
// L DEADLOCK RISK - never block on async
public User GetUser(int id)
{
    return _repository.GetByIdAsync(id, default).Result; // DEADLOCK!
}

public List<User> GetUsers()
{
    return _repository.GetAllAsync(default).GetAwaiter().GetResult(); // DEADLOCK!
}

//  CORRECT - async all the way
public async Task<User> GetUserAsync(int id, CancellationToken ct)
{
    return await _repository.GetByIdAsync(id, ct);
}
```

### 2. Async Void (Except Event Handlers)

```csharp
// L BAD - unhandled exceptions crash app
public async void ProcessOrderAsync(Order order)
{
    await _repository.SaveAsync(order, default);
    throw new Exception("Unhandled!"); // Crash!
}

//  GOOD - exceptions can be caught
public async Task ProcessOrderAsync(Order order, CancellationToken ct)
{
    await _repository.SaveAsync(order, ct);
}

//   EXCEPTION - event handlers only
private async void OnDataReceived(object sender, DataEventArgs e)
{
    try
    {
        await ProcessDataAsync(e.Data);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error in event handler");
    }
}
```

### 3. Unnecessary Task.Run in ASP.NET Core

```csharp
// L BAD - wastes thread pool threads
app.MapGet("/cpu-bound", async () =>
{
    // Already on thread pool thread!
    return await Task.Run(() => PerformCalculation());
});

//  GOOD - direct synchronous work (or use dedicated queue)
app.MapGet("/cpu-bound", () =>
{
    return PerformCalculation(); // Sync endpoint for CPU work
});

//  BETTER - offload to background queue for long operations
app.MapPost("/process", (ProcessRequest req, IBackgroundQueue queue) =>
{
    queue.Enqueue(req);
    return Results.Accepted();
});
```

### 4. Missing CancellationToken

```csharp
// L BAD - cannot cancel long-running operation
public async Task<Report> GenerateReportAsync(int userId)
{
    var data = await _repository.GetDataAsync(userId);
    return ProcessData(data);
}

//  GOOD - supports cancellation
public async Task<Report> GenerateReportAsync(int userId, CancellationToken ct)
{
    var data = await _repository.GetDataAsync(userId, ct);
    ct.ThrowIfCancellationRequested();
    return ProcessData(data);
}
```

## Async Enumerable (IAsyncEnumerable<T>)

### Streaming Large Datasets

```csharp
// Endpoint using streaming
app.MapGet("/products/stream", async (
    IProductRepository repo,
    CancellationToken ct) =>
{
    return Results.Stream(repo.StreamProductsAsync(ct), "application/json");
});

// Repository streaming implementation
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public async IAsyncEnumerable<Product> StreamProductsAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var product in _context.Products.AsAsyncEnumerable().WithCancellation(ct))
        {
            yield return product;
        }
    }
}

// Consumer with backpressure
public class ProductExporter
{
    public async Task ExportProductsAsync(
        IAsyncEnumerable<Product> products,
        Stream outputStream,
        CancellationToken ct)
    {
        await using var writer = new StreamWriter(outputStream);

        await foreach (var product in products.WithCancellation(ct))
        {
            await writer.WriteLineAsync(JsonSerializer.Serialize(product));
        }
    }
}
```

**Source**: [Microsoft Docs - IAsyncEnumerable](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream) (October 2023)

### Parallel Processing with SemaphoreSlim

```csharp
public class BatchProcessor
{
    private readonly SemaphoreSlim _semaphore = new(10); // Max 10 concurrent
    private readonly IHttpClientFactory _httpClientFactory;

    public async Task<List<Result>> ProcessBatchAsync(
        IEnumerable<string> urls,
        CancellationToken ct)
    {
        var tasks = urls.Select(url => ProcessUrlAsync(url, ct));
        return (await Task.WhenAll(tasks)).ToList();
    }

    private async Task<Result> ProcessUrlAsync(string url, CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var client = _httpClientFactory.CreateClient();
            var response = await client.GetAsync(url, ct);
            return new Result { Url = url, Success = response.IsSuccessStatusCode };
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Async Lazy Initialization

### Thread-Safe Async Initialization

```csharp
public class ConfigurationService
{
    private readonly SemaphoreSlim _initLock = new(1, 1);
    private readonly IHttpClientFactory _httpClientFactory;
    private AppConfiguration? _config;

    public async Task<AppConfiguration> GetConfigurationAsync(CancellationToken ct)
    {
        if (_config is not null)
            return _config;

        await _initLock.WaitAsync(ct);
        try
        {
            // Double-check after acquiring lock
            if (_config is not null)
                return _config;

            var client = _httpClientFactory.CreateClient();
            var response = await client.GetAsync("/api/config", ct);
            response.EnsureSuccessStatusCode();

            _config = await response.Content.ReadFromJsonAsync<AppConfiguration>(ct);
            return _config!;
        }
        finally
        {
            _initLock.Release();
        }
    }
}

// Alternative: AsyncLazy<T> (custom implementation)
public class AsyncLazy<T>
{
    private readonly Lazy<Task<T>> _instance;

    public AsyncLazy(Func<Task<T>> factory)
    {
        _instance = new Lazy<Task<T>>(factory);
    }

    public Task<T> Value => _instance.Value;
}

// Usage
private readonly AsyncLazy<AppConfiguration> _config;

public ConfigurationService(IHttpClientFactory httpClientFactory)
{
    _config = new AsyncLazy<AppConfiguration>(async () =>
    {
        var client = httpClientFactory.CreateClient();
        var response = await client.GetAsync("/api/config");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<AppConfiguration>();
    });
}

public Task<AppConfiguration> GetConfigurationAsync() => _config.Value;
```

## Performance Best Practices

### Minimize Allocations

```csharp
//  GOOD - single await, no state machine overhead
public async Task<User> GetUserAsync(int id, CancellationToken ct)
{
    return await _repository.GetByIdAsync(id, ct);
}

// L BAD - unnecessary async/await (adds state machine)
public async Task<User> GetUserAsync(int id, CancellationToken ct)
{
    return await _repository.GetByIdAsync(id, ct); // Just return the task!
}

//  BETTER - elide async/await when just passing through
public Task<User> GetUserAsync(int id, CancellationToken ct)
{
    return _repository.GetByIdAsync(id, ct);
}

//   BUT - use async/await when using try/catch or using statements
public async Task<User> GetUserWithLoggingAsync(int id, CancellationToken ct)
{
    try
    {
        return await _repository.GetByIdAsync(id, ct);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to get user {UserId}", id);
        throw;
    }
}
```

### Parallel Operations

```csharp
//  GOOD - parallel independent operations
public async Task<OrderSummary> GetOrderSummaryAsync(int orderId, CancellationToken ct)
{
    var orderTask = _orderRepo.GetByIdAsync(orderId, ct);
    var itemsTask = _itemRepo.GetByOrderIdAsync(orderId, ct);
    var customerTask = _customerRepo.GetByOrderIdAsync(orderId, ct);

    await Task.WhenAll(orderTask, itemsTask, customerTask);

    return new OrderSummary
    {
        Order = orderTask.Result,
        Items = itemsTask.Result,
        Customer = customerTask.Result
    };
}

// L BAD - sequential when could be parallel
public async Task<OrderSummary> GetOrderSummaryAsync(int orderId, CancellationToken ct)
{
    var order = await _orderRepo.GetByIdAsync(orderId, ct);
    var items = await _itemRepo.GetByOrderIdAsync(orderId, ct);
    var customer = await _customerRepo.GetByOrderIdAsync(orderId, ct);

    return new OrderSummary { Order = order, Items = items, Customer = customer };
}
```

## Testing Async Code

### Unit Testing Async Methods

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _mockRepo = new Mock<IUserRepository>();
        _sut = new UserService(_mockRepo.Object);
    }

    [Fact]
    public async Task GetUserAsync_WhenUserExists_ReturnsUser()
    {
        // Arrange
        var expectedUser = new User { Id = 1, Name = "John" };
        _mockRepo.Setup(r => r.GetByIdAsync(1, It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedUser);

        // Act
        var result = await _sut.GetUserAsync(1, CancellationToken.None);

        // Assert
        result.Should().Be(expectedUser);
    }

    [Fact]
    public async Task CreateUserAsync_WhenCancelled_ThrowsOperationCancelledException()
    {
        // Arrange
        var cts = new CancellationTokenSource();
        cts.Cancel();

        // Act & Assert
        await Assert.ThrowsAsync<OperationCanceledException>(
            () => _sut.CreateUserAsync(new CreateUserDto(), cts.Token));
    }

    [Fact(Timeout = 5000)] // Fail if test takes > 5s
    public async Task ProcessOrderAsync_CompletesWithinTimeout()
    {
        // Arrange
        var order = new Order { Id = 1 };

        // Act
        var result = await _sut.ProcessOrderAsync(order, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
    }
}
```

## Checklist

- [ ] All I/O methods are async (database, HTTP, file)
- [ ] CancellationToken accepted and propagated in all async methods
- [ ] No blocking calls (.Result, .Wait(), .GetAwaiter().GetResult())
- [ ] No async void (except event handlers)
- [ ] ValueTask used only for hot paths with sync completion
- [ ] Task.Run avoided in ASP.NET Core endpoints
- [ ] IAsyncEnumerable used for streaming large datasets
- [ ] Parallel operations used when operations are independent
- [ ] SemaphoreSlim used to limit concurrency for external calls
- [ ] Background services respect CancellationToken for graceful shutdown
- [ ] Async code tested with cancellation scenarios

## References

- [Microsoft Docs - Async/await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) (January 2024)
- [Microsoft Docs - Task-based asynchronous pattern](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) (December 2023)
- [.NET Blog - Understanding ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) (March 2023)
- [Microsoft Docs - Cancellation](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) (November 2023)
- [Microsoft Docs - IAsyncEnumerable](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream) (October 2023)
- [Stephen Cleary - Async Best Practices](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) (March 2013, still relevant)

---

**Next Steps**: Review `caching.md` for output caching patterns, or `resilience.md` for Polly integration with async HttpClient calls.
