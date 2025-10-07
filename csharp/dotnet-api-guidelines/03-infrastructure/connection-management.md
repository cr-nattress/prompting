# Connection Management and Resiliency

## Overview

This guide covers database connection management, pooling, resiliency strategies, and best practices for .NET 8 applications using Entity Framework Core 8. Learn how to handle transient failures, manage connection lifetimes, and implement multi-tenancy patterns.

## Related Documentation
- [EF Core Setup](./ef-core-setup.md) - DbContext configuration and setup
- [Data Patterns](./data-patterns.md) - Repository and query patterns
- [Performance Optimization](../08-performance/database-optimization.md) - Query and connection optimization
- [Error Handling](../06-error-handling/global-exception-handling.md) - Exception handling strategies

## Table of Contents
1. [Connection Pooling](#connection-pooling)
2. [Connection Resiliency](#connection-resiliency)
3. [Transaction Management](#transaction-management)
4. [DbContext Lifetime](#dbcontext-lifetime)
5. [Date/Time Handling](#datetime-handling)
6. [Multi-Tenancy Strategies](#multi-tenancy-strategies)

---

## Connection Pooling

### Understanding Connection Pooling in .NET 8

Connection pooling is automatically managed by ADO.NET and is enabled by default. EF Core 8 also provides DbContext pooling for improved performance.

### ADO.NET Connection Pooling

```csharp
using Microsoft.Data.SqlClient;

namespace MyApp.Infrastructure.Configuration;

public class ConnectionPoolingConfiguration
{
    // Connection pooling is enabled by default
    public static string GetConnectionString(
        string server,
        string database,
        int minPoolSize = 0,
        int maxPoolSize = 100,
        int connectionLifetime = 0,
        bool pooling = true)
    {
        var builder = new SqlConnectionStringBuilder
        {
            DataSource = server,
            InitialCatalog = database,
            IntegratedSecurity = true,

            // Connection pooling settings
            Pooling = pooling,
            MinPoolSize = minPoolSize,      // Minimum connections in pool
            MaxPoolSize = maxPoolSize,      // Maximum connections in pool
            ConnectionLifetime = connectionLifetime, // Max lifetime in seconds (0 = no limit)

            // Performance settings
            MultipleActiveResultSets = true,
            ConnectTimeout = 30,
            ApplicationName = "MyApp"
        };

        return builder.ConnectionString;
    }
}
```

### DbContext Pooling (Recommended for High Performance)

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    // Standard DbContext registration (no pooling)
    public static IServiceCollection AddStandardDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));

        return services;
    }

    // DbContext pooling for better performance
    public static IServiceCollection AddPooledDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContextPool<ApplicationDbContext>(
            options => options.UseSqlServer(connectionString),
            poolSize: 128); // Default is 1024

        return services;
    }

    // Custom pool size based on load
    public static IServiceCollection AddPooledDbContextWithCustomSize(
        this IServiceCollection services,
        string connectionString,
        int poolSize)
    {
        // Pool size should be roughly: max concurrent requests / 2
        // For example: If you expect 500 concurrent requests, use pool size of ~250
        services.AddDbContextPool<ApplicationDbContext>(
            options =>
            {
                options.UseSqlServer(connectionString);
                options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
            },
            poolSize: poolSize);

        return services;
    }
}
```

### DbContext Pooling Considerations

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Data;

// DbContext for pooling - must be stateless
public class PooledApplicationDbContext : DbContext
{
    public PooledApplicationDbContext(DbContextOptions<PooledApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    // DO NOT store state in pooled DbContext
    // private readonly string _userId; // L BAD - causes issues with pooling

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(PooledApplicationDbContext).Assembly);
    }

    // Reset state when context is returned to pool
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        base.OnConfiguring(optionsBuilder);

        #if DEBUG
        optionsBuilder.EnableSensitiveDataLogging();
        optionsBuilder.EnableDetailedErrors();
        #endif
    }
}
```

### Monitoring Connection Pool

```csharp
using System.Diagnostics;
using Microsoft.Data.SqlClient;

namespace MyApp.Infrastructure.Monitoring;

public class ConnectionPoolMonitor
{
    private readonly string _connectionString;

    public ConnectionPoolMonitor(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<ConnectionPoolStats> GetPoolStatsAsync()
    {
        await using var connection = new SqlConnection(_connectionString);

        // Get performance counters
        var category = new PerformanceCounterCategory(".NET Data Provider for SqlServer");
        var instanceName = GetInstanceName();

        var stats = new ConnectionPoolStats
        {
            NumberOfActiveConnections = GetCounterValue(category, "NumberOfActiveConnections", instanceName),
            NumberOfFreeConnections = GetCounterValue(category, "NumberOfFreeConnections", instanceName),
            NumberOfPooledConnections = GetCounterValue(category, "NumberOfPooledConnections", instanceName),
            NumberOfNonPooledConnections = GetCounterValue(category, "NumberOfNonPooledConnections", instanceName)
        };

        return stats;
    }

    private float GetCounterValue(PerformanceCounterCategory category, string counterName, string instanceName)
    {
        try
        {
            using var counter = new PerformanceCounter(category.CategoryName, counterName, instanceName, true);
            return counter.NextValue();
        }
        catch
        {
            return -1;
        }
    }

    private string GetInstanceName()
    {
        var builder = new SqlConnectionStringBuilder(_connectionString);
        return $"{AppDomain.CurrentDomain.FriendlyName}[{builder.DataSource}:{builder.InitialCatalog}]";
    }
}

public class ConnectionPoolStats
{
    public float NumberOfActiveConnections { get; set; }
    public float NumberOfFreeConnections { get; set; }
    public float NumberOfPooledConnections { get; set; }
    public float NumberOfNonPooledConnections { get; set; }
}
```

---

## Connection Resiliency

### Retry Policies and Execution Strategies

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure;

public static class ResiliencyConfiguration
{
    // Basic retry configuration
    public static IServiceCollection AddResilientDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                connectionString,
                sqlOptions =>
                {
                    // Enable retry on transient failures
                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 5,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: null);
                });
        });

        return services;
    }

    // Advanced retry configuration
    public static IServiceCollection AddAdvancedResilientDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                connectionString,
                sqlOptions =>
                {
                    // Custom error numbers to retry
                    var errorNumbersToAdd = new List<int>
                    {
                        // Custom transient errors
                        4060, // Cannot open database
                        40197, // Service processing request
                        40501, // Service busy
                        40613, // Database unavailable
                        49918, // Cannot process request
                        49919, // Too many create/update operations
                        49920  // Too many operations in progress
                    };

                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 6,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: errorNumbersToAdd);

                    // Command timeout
                    sqlOptions.CommandTimeout(60);
                });
        });

        return services;
    }
}
```

### Custom Execution Strategy

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace MyApp.Infrastructure.Resiliency;

public class CustomExecutionStrategy : ExecutionStrategy
{
    private const int DefaultMaxRetryCount = 5;
    private const int DefaultMaxDelay = 30;

    public CustomExecutionStrategy(DbContext context)
        : base(context, DefaultMaxRetryCount, TimeSpan.FromSeconds(DefaultMaxDelay))
    {
    }

    public CustomExecutionStrategy(ExecutionStrategyDependencies dependencies)
        : base(dependencies, DefaultMaxRetryCount, TimeSpan.FromSeconds(DefaultMaxDelay))
    {
    }

    protected override bool ShouldRetryOn(Exception exception)
    {
        // Customize which exceptions trigger retry
        if (exception is SqlException sqlException)
        {
            foreach (SqlError error in sqlException.Errors)
            {
                // Transient error codes
                switch (error.Number)
                {
                    case -2: // Timeout
                    case 20: // Instance not available
                    case 64: // Error from server
                    case 233: // Connection initialization error
                    case 1205: // Deadlock
                    case 4060: // Cannot open database
                    case 40197: // Service processing request
                    case 40501: // Service busy
                    case 40613: // Database unavailable
                        return true;
                }
            }
        }

        return false;
    }

    protected override TimeSpan? GetNextDelay(Exception lastException)
    {
        // Exponential backoff
        var baseDelay = TimeSpan.FromSeconds(Math.Pow(2, RetriesCount));
        var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000));
        return baseDelay + jitter;
    }
}

// Register custom execution strategy
public static class CustomExecutionStrategyExtensions
{
    public static IServiceCollection AddCustomExecutionStrategy(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                connectionString,
                sqlOptions =>
                {
                    sqlOptions.ExecutionStrategy(dependencies => new CustomExecutionStrategy(dependencies));
                });
        });

        return services;
    }
}
```

### Using Execution Strategy

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.API.Services;

public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Execution strategy is applied automatically
    public async Task<Product?> GetProductAsync(int id)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Id == id);
    }

    // Manual execution strategy control
    public async Task<Product> CreateProductWithManualRetryAsync(Product product)
    {
        var strategy = _context.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _context.Database.BeginTransactionAsync();

            try
            {
                _context.Products.Add(product);
                await _context.SaveChangesAsync();

                await transaction.CommitAsync();

                return product;
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        });
    }
}
```

---

## Transaction Management

### Transaction Scopes and Isolation Levels

```csharp
using Microsoft.EntityFrameworkCore;
using System.Data;
using System.Transactions;

namespace MyApp.Infrastructure.Transactions;

public class TransactionService
{
    private readonly ApplicationDbContext _context;

    public TransactionService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Basic transaction
    public async Task ExecuteInTransactionAsync(Func<Task> action)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            await action();
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // Transaction with specific isolation level
    public async Task ExecuteWithIsolationLevelAsync(
        Func<Task> action,
        IsolationLevel isolationLevel)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync(isolationLevel);

        try
        {
            await action();
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }

    // TransactionScope for distributed transactions
    public async Task ExecuteWithTransactionScopeAsync(Func<Task> action)
    {
        var transactionOptions = new TransactionOptions
        {
            IsolationLevel = System.Transactions.IsolationLevel.ReadCommitted,
            Timeout = TimeSpan.FromSeconds(30)
        };

        using var scope = new TransactionScope(
            TransactionScopeOption.Required,
            transactionOptions,
            TransactionScopeAsyncFlowOption.Enabled);

        await action();

        scope.Complete();
    }
}
```

### Isolation Levels Explained

```csharp
namespace MyApp.Infrastructure.Examples;

public class IsolationLevelExamples
{
    private readonly ApplicationDbContext _context;

    public IsolationLevelExamples(ApplicationDbContext context)
    {
        _context = context;
    }

    // READ UNCOMMITTED - Lowest isolation, best performance
    // Allows dirty reads, non-repeatable reads, and phantom reads
    public async Task ReadUncommittedExample()
    {
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.ReadUncommitted);

        // Can read uncommitted data from other transactions
        var products = await _context.Products.ToListAsync();

        await transaction.CommitAsync();
    }

    // READ COMMITTED - Default for SQL Server
    // Prevents dirty reads, allows non-repeatable reads and phantom reads
    public async Task ReadCommittedExample()
    {
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.ReadCommitted);

        // Only reads committed data
        var products = await _context.Products.ToListAsync();

        await transaction.CommitAsync();
    }

    // REPEATABLE READ - Prevents dirty and non-repeatable reads
    // Allows phantom reads
    public async Task RepeatableReadExample()
    {
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.RepeatableRead);

        // First read
        var product1 = await _context.Products.FindAsync(1);

        // Some delay
        await Task.Delay(1000);

        // Second read will return the same data (no non-repeatable reads)
        var product2 = await _context.Products.FindAsync(1);

        await transaction.CommitAsync();
    }

    // SERIALIZABLE - Highest isolation, worst performance
    // Prevents all concurrency issues
    public async Task SerializableExample()
    {
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.Serializable);

        // Complete isolation from other transactions
        var products = await _context.Products.ToListAsync();

        await transaction.CommitAsync();
    }

    // SNAPSHOT - Uses row versioning
    // Prevents all read phenomena without blocking
    public async Task SnapshotExample()
    {
        // Note: Requires ALLOW_SNAPSHOT_ISOLATION to be enabled on database
        await using var transaction = await _context.Database
            .BeginTransactionAsync(IsolationLevel.Snapshot);

        // Reads snapshot of data at transaction start
        var products = await _context.Products.ToListAsync();

        await transaction.CommitAsync();
    }
}
```

### Complex Transaction Scenarios

```csharp
using Microsoft.EntityFrameworkCore.Storage;

namespace MyApp.Infrastructure.Services;

public class OrderProcessingService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(
        ApplicationDbContext context,
        ILogger<OrderProcessingService> logger)
    {
        _context = context;
        _logger = logger;
    }

    // Multiple operations in single transaction
    public async Task ProcessOrderAsync(Order order)
    {
        var strategy = _context.Database.CreateExecutionStrategy();

        await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _context.Database
                .BeginTransactionAsync(IsolationLevel.ReadCommitted);

            try
            {
                // 1. Create order
                _context.Orders.Add(order);
                await _context.SaveChangesAsync();

                // 2. Update product stock
                foreach (var item in order.OrderItems)
                {
                    var product = await _context.Products.FindAsync(item.ProductId);
                    if (product == null)
                    {
                        throw new InvalidOperationException($"Product {item.ProductId} not found");
                    }

                    product.StockQuantity -= item.Quantity;

                    if (product.StockQuantity < 0)
                    {
                        throw new InvalidOperationException(
                            $"Insufficient stock for product {product.Name}");
                    }
                }

                await _context.SaveChangesAsync();

                // 3. Create invoice
                var invoice = new Invoice
                {
                    OrderId = order.Id,
                    TotalAmount = order.TotalAmount,
                    CreatedAt = DateTime.UtcNow
                };

                _context.Invoices.Add(invoice);
                await _context.SaveChangesAsync();

                // 4. Commit transaction
                await transaction.CommitAsync();

                _logger.LogInformation("Successfully processed order {OrderId}", order.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing order {OrderId}", order.Id);
                await transaction.RollbackAsync();
                throw;
            }
        });
    }

    // Savepoint for partial rollback
    public async Task ProcessOrderWithSavepointAsync(Order order)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            // Create order
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();

            // Create savepoint
            await transaction.CreateSavepointAsync("BeforeStockUpdate");

            try
            {
                // Update stock
                foreach (var item in order.OrderItems)
                {
                    var product = await _context.Products.FindAsync(item.ProductId);
                    product!.StockQuantity -= item.Quantity;
                }

                await _context.SaveChangesAsync();
            }
            catch
            {
                // Rollback to savepoint
                await transaction.RollbackToSavepointAsync("BeforeStockUpdate");
                _logger.LogWarning("Stock update failed, order created without stock reduction");
            }

            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

---

## DbContext Lifetime

### Understanding DbContext Lifetimes

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure;

public static class DbContextLifetimeConfiguration
{
    // SCOPED (default and recommended for web apps)
    public static IServiceCollection AddScopedDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(
            options => options.UseSqlServer(connectionString),
            ServiceLifetime.Scoped); // Default

        return services;
    }

    // TRANSIENT (not recommended - creates new instance per injection)
    public static IServiceCollection AddTransientDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(
            options => options.UseSqlServer(connectionString),
            ServiceLifetime.Transient);

        return services;
    }

    // SINGLETON (avoid - causes concurrency issues)
    // DO NOT USE unless you know exactly what you're doing
    public static IServiceCollection AddSingletonDbContext(
        this IServiceCollection services,
        string connectionString)
    {
        // L NOT RECOMMENDED
        services.AddDbContext<ApplicationDbContext>(
            options => options.UseSqlServer(connectionString),
            ServiceLifetime.Singleton);

        return services;
    }

    // FACTORY (for background services and long-running operations)
    public static IServiceCollection AddDbContextFactory(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContextFactory<ApplicationDbContext>(
            options => options.UseSqlServer(connectionString));

        return services;
    }
}
```

### Using DbContext in Different Scenarios

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.API.Examples;

// Web API Controller - Scoped DbContext
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public ProductsController(ApplicationDbContext context)
    {
        _context = context; // Injected as scoped - one instance per request
    }

    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetProducts()
    {
        // DbContext lives for duration of request
        return await _context.Products.ToListAsync();
    }
}

// Background Service - DbContext Factory
public class ProductSyncBackgroundService : BackgroundService
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;
    private readonly ILogger<ProductSyncBackgroundService> _logger;

    public ProductSyncBackgroundService(
        IDbContextFactory<ApplicationDbContext> contextFactory,
        ILogger<ProductSyncBackgroundService> logger)
    {
        _contextFactory = contextFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Create new context for each operation
            await using var context = await _contextFactory.CreateDbContextAsync(stoppingToken);

            try
            {
                // Process data
                var products = await context.Products
                    .Where(p => p.NeedsSync)
                    .ToListAsync(stoppingToken);

                foreach (var product in products)
                {
                    // Sync logic
                    product.LastSyncedAt = DateTime.UtcNow;
                }

                await context.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error syncing products");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

// Parallel Processing - DbContext Factory
public class ParallelProductProcessor
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public ParallelProductProcessor(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task ProcessProductsInParallelAsync(List<int> productIds)
    {
        var tasks = productIds.Select(async id =>
        {
            // Each parallel task gets its own DbContext
            await using var context = await _contextFactory.CreateDbContextAsync();

            var product = await context.Products.FindAsync(id);
            if (product != null)
            {
                // Process product
                product.ProcessedAt = DateTime.UtcNow;
                await context.SaveChangesAsync();
            }
        });

        await Task.WhenAll(tasks);
    }
}
```

---

## Date/Time Handling

### DateTime vs DateTimeOffset vs NodaTime

```csharp
using NodaTime;
using System;

namespace MyApp.Core.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // L BAD: DateTime without explicit kind
    public DateTime CreatedAt { get; set; }

    //  BETTER: DateTime with UTC
    public DateTime CreatedAtUtc { get; set; } = DateTime.UtcNow;

    //  GOOD: DateTimeOffset (includes timezone)
    public DateTimeOffset CreatedAtOffset { get; set; } = DateTimeOffset.UtcNow;

    //  BEST: NodaTime Instant (unambiguous point in time)
    public Instant CreatedAtInstant { get; set; } = SystemClock.Instance.GetCurrentInstant();

    // For dates without time
    public LocalDate? ManufactureDate { get; set; }

    // For time without date
    public LocalTime? PreferredDeliveryTime { get; set; }

    // For dates with specific timezone
    public ZonedDateTime? ScheduledReleaseDate { get; set; }
}
```

### Configuring NodaTime with EF Core

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using NodaTime;

namespace MyApp.Infrastructure;

public static class NodaTimeConfiguration
{
    public static IServiceCollection AddDbContextWithNodaTime(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                connectionString,
                sqlOptions =>
                {
                    // Enable NodaTime support
                    sqlOptions.UseNodaTime();
                });
        });

        return services;
    }
}

// Entity configuration
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Instant is stored as datetime2
        builder.Property(p => p.CreatedAtInstant)
            .HasColumnType("datetime2")
            .IsRequired();

        // LocalDate is stored as date
        builder.Property(p => p.ManufactureDate)
            .HasColumnType("date");

        // LocalTime is stored as time
        builder.Property(p => p.PreferredDeliveryTime)
            .HasColumnType("time");

        // ZonedDateTime requires special handling
        builder.Property(p => p.ScheduledReleaseDate)
            .HasConversion(
                v => v.HasValue ? v.Value.ToInstant() : (Instant?)null,
                v => v.HasValue ? v.Value.InUtc() : (ZonedDateTime?)null);
    }
}
```

### Date/Time Best Practices

```csharp
using NodaTime;
using NodaTime.Extensions;

namespace MyApp.API.Services;

public class DateTimeService
{
    private readonly IClock _clock;
    private readonly IDateTimeZoneProvider _tzProvider;

    public DateTimeService(IClock clock, IDateTimeZoneProvider tzProvider)
    {
        _clock = clock;
        _tzProvider = tzProvider;
    }

    // Get current time (testable)
    public Instant GetCurrentInstant() => _clock.GetCurrentInstant();

    // Convert UTC to user timezone
    public ZonedDateTime ConvertToUserTimezone(Instant instant, string timeZoneId)
    {
        var timezone = _tzProvider.GetZoneOrNull(timeZoneId) ?? DateTimeZone.Utc;
        return instant.InZone(timezone);
    }

    // Parse user input with timezone
    public Instant ParseUserDateTime(string dateTimeString, string timeZoneId)
    {
        var timezone = _tzProvider.GetZoneOrNull(timeZoneId) ?? DateTimeZone.Utc;
        var localDateTime = LocalDateTime.FromDateTime(DateTime.Parse(dateTimeString));
        var zonedDateTime = localDateTime.InZoneStrictly(timezone);
        return zonedDateTime.ToInstant();
    }

    // Get start of day in specific timezone
    public Instant GetStartOfDay(LocalDate date, string timeZoneId)
    {
        var timezone = _tzProvider.GetZoneOrNull(timeZoneId) ?? DateTimeZone.Utc;
        var localDateTime = date.AtMidnight();
        return localDateTime.InZoneStrictly(timezone).ToInstant();
    }

    // Calculate duration between two instants
    public Duration GetDuration(Instant start, Instant end)
    {
        return end - start;
    }

    // Check if instant falls within business hours
    public bool IsWithinBusinessHours(Instant instant, string timeZoneId)
    {
        var timezone = _tzProvider.GetZoneOrNull(timeZoneId) ?? DateTimeZone.Utc;
        var zonedDateTime = instant.InZone(timezone);
        var localTime = zonedDateTime.TimeOfDay;

        var businessStart = new LocalTime(9, 0);
        var businessEnd = new LocalTime(17, 0);

        return localTime >= businessStart && localTime < businessEnd;
    }
}

// Register NodaTime services
public static class NodaTimeServiceExtensions
{
    public static IServiceCollection AddNodaTime(this IServiceCollection services)
    {
        services.AddSingleton<IClock>(SystemClock.Instance);
        services.AddSingleton<IDateTimeZoneProvider>(DateTimeZoneProviders.Tzdb);
        services.AddScoped<DateTimeService>();

        return services;
    }
}
```

---

## Multi-Tenancy Strategies

### Strategy 1: Database Per Tenant

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure.MultiTenancy;

public interface ITenantService
{
    string GetCurrentTenantId();
    string GetTenantConnectionString(string tenantId);
}

public class TenantService : ITenantService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IConfiguration _configuration;

    public TenantService(
        IHttpContextAccessor httpContextAccessor,
        IConfiguration configuration)
    {
        _httpContextAccessor = httpContextAccessor;
        _configuration = configuration;
    }

    public string GetCurrentTenantId()
    {
        // Get tenant from header, subdomain, or JWT claim
        return _httpContextAccessor.HttpContext?.Request.Headers["X-Tenant-Id"].FirstOrDefault()
            ?? throw new InvalidOperationException("Tenant not specified");
    }

    public string GetTenantConnectionString(string tenantId)
    {
        // Get connection string from configuration or database
        return _configuration.GetConnectionString($"Tenant_{tenantId}")
            ?? throw new InvalidOperationException($"Connection string not found for tenant {tenantId}");
    }
}

// Tenant-aware DbContext
public class TenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;
    private readonly string _tenantId;

    public TenantDbContext(
        DbContextOptions<TenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
        _tenantId = _tenantService.GetCurrentTenantId();
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            var connectionString = _tenantService.GetTenantConnectionString(_tenantId);
            optionsBuilder.UseSqlServer(connectionString);
        }

        base.OnConfiguring(optionsBuilder);
    }
}
```

### Strategy 2: Schema Per Tenant

```csharp
namespace MyApp.Infrastructure.MultiTenancy;

public class SchemaPerTenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public SchemaPerTenantDbContext(
        DbContextOptions<SchemaPerTenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Set schema based on tenant
        var tenantId = _tenantService.GetCurrentTenantId();
        var schema = $"tenant_{tenantId}";

        modelBuilder.HasDefaultSchema(schema);

        // Configure entities
        modelBuilder.Entity<Product>().ToTable("Products", schema);
        modelBuilder.Entity<Order>().ToTable("Orders", schema);
    }
}
```

### Strategy 3: Discriminator Column (Shared Database)

```csharp
namespace MyApp.Infrastructure.MultiTenancy;

public interface ITenantEntity
{
    string TenantId { get; set; }
}

public class Product : ITenantEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string TenantId { get; set; } = string.Empty;
}

public class SharedDatabaseDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public SharedDatabaseDbContext(
        DbContextOptions<SharedDatabaseDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply global query filter for tenant isolation
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.GetCurrentTenantId());

        // Enforce tenant index
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.TenantId);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Automatically set TenantId on save
        var tenantId = _tenantService.GetCurrentTenantId();

        foreach (var entry in ChangeTracker.Entries<ITenantEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.TenantId = tenantId;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Multi-Tenancy Registration

```csharp
namespace MyApp.Infrastructure;

public static class MultiTenancyConfiguration
{
    public static IServiceCollection AddMultiTenancy(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHttpContextAccessor();
        services.AddScoped<ITenantService, TenantService>();

        // Choose strategy based on configuration
        var strategy = configuration["MultiTenancy:Strategy"];

        return strategy switch
        {
            "DatabasePerTenant" => services.AddDatabasePerTenant(configuration),
            "SchemaPerTenant" => services.AddSchemaPerTenant(configuration),
            "SharedDatabase" => services.AddSharedDatabase(configuration),
            _ => throw new InvalidOperationException($"Unknown multi-tenancy strategy: {strategy}")
        };
    }

    private static IServiceCollection AddDatabasePerTenant(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<TenantDbContext>();
        return services;
    }

    private static IServiceCollection AddSchemaPerTenant(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<SchemaPerTenantDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        return services;
    }

    private static IServiceCollection AddSharedDatabase(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<SharedDatabaseDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        return services;
    }
}
```

---

## Best Practices

### 1. Connection Pooling
- Use ADO.NET connection pooling (enabled by default)
- Use DbContext pooling for high-traffic applications
- Monitor pool metrics in production
- Adjust pool size based on actual load

### 2. Connection Resiliency
- Enable retry policies for transient failures
- Use execution strategies for complex scenarios
- Implement exponential backoff
- Log retry attempts

### 3. Transactions
- Use appropriate isolation levels
- Keep transactions short
- Use savepoints for partial rollback
- Always use using/await using for proper disposal

### 4. DbContext Lifetime
- Use Scoped lifetime for web applications
- Use Factory pattern for background services
- Never use Singleton lifetime
- Dispose DbContext properly

### 5. Date/Time
- Prefer NodaTime for unambiguous time handling
- Always store UTC in database
- Handle timezone conversions at application boundaries
- Use DateTimeOffset if NodaTime isn't available

### 6. Multi-Tenancy
- Choose strategy based on isolation requirements
- Database per tenant: Best isolation, higher cost
- Schema per tenant: Good isolation, moderate cost
- Shared database: Lowest cost, requires careful implementation
- Always enforce tenant filtering

---

## References

- [Connection Pooling](https://docs.microsoft.com/en-us/ef/core/performance/advanced-performance-topics#connection-pooling)
- [DbContext Pooling](https://docs.microsoft.com/en-us/ef/core/performance/advanced-performance-topics#dbcontext-pooling)
- [Connection Resiliency](https://docs.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency)
- [Transactions](https://docs.microsoft.com/en-us/ef/core/saving/transactions)
- [DbContext Lifetime](https://docs.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [NodaTime](https://nodatime.org/)
- [NodaTime.EntityFrameworkCore](https://github.com/nodatime/nodatime.entityframeworkcore)
- [Multi-Tenancy Patterns](https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)

---

**Last Updated**: 2025-10-06
**EF Core Version**: 8.0
**.NET Version**: 8.0
