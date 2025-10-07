# EF Core Setup and Configuration

## Overview

This guide covers Entity Framework Core 8 setup, configuration, and best practices for .NET 8 applications, including DbContext configuration, entity mappings, migrations, and seeding strategies.

## Related Documentation
- [Data Patterns](./data-patterns.md) - Repository and query patterns
- [Connection Management](./connection-management.md) - Connection pooling and resiliency
- [Security Best Practices](../05-security/authentication.md) - Secure database access
- [Testing Strategies](../09-testing/integration-testing.md) - Database testing approaches

## Table of Contents
1. [DbContext Configuration](#dbcontext-configuration)
2. [Entity Configuration](#entity-configuration)
3. [Migration Management](#migration-management)
4. [Seeding Strategies](#seeding-strategies)
5. [Connection Strings](#connection-strings)
6. [Multiple DbContext Scenarios](#multiple-dbcontext-scenarios)

---

## DbContext Configuration

### Basic DbContext Setup

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply all configurations from current assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }

    // Optional: Soft delete query filter
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        base.OnConfiguring(optionsBuilder);

        // Enable sensitive data logging only in development
        #if DEBUG
        optionsBuilder.EnableSensitiveDataLogging();
        optionsBuilder.EnableDetailedErrors();
        #endif
    }
}
```

### Dependency Injection Registration

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Basic registration
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                sqlOptions =>
                {
                    // Enable retry on failure
                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 5,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: null);

                    // Set command timeout
                    sqlOptions.CommandTimeout(30);

                    // Migration assembly (if in separate project)
                    sqlOptions.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName);
                });

            // Performance optimizations
            options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
        });

        return services;
    }

    // Alternative: Scoped lifetime (default for AddDbContext)
    public static IServiceCollection AddInfrastructureWithPooling(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Use DbContext pooling for better performance
        services.AddDbContextPool<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                sqlOptions =>
                {
                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: 5,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: null);
                });

            options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
        },
        poolSize: 128); // Default is 1024, adjust based on load

        return services;
    }

    // Factory pattern for advanced scenarios
    public static IServiceCollection AddInfrastructureWithFactory(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContextFactory<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"));
        });

        return services;
    }
}
```

### DbContext Factory Usage

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.API.Services;

public class ProductService
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public ProductService(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task<List<Product>> GetProductsAsync()
    {
        // Create a new context instance per operation
        await using var context = await _contextFactory.CreateDbContextAsync();

        return await context.Products
            .AsNoTracking()
            .ToListAsync();
    }

    // Useful for long-running operations or background services
    public async Task ProcessLargeDatasetAsync()
    {
        await using var context = await _contextFactory.CreateDbContextAsync();

        var products = context.Products
            .AsNoTracking()
            .AsAsyncEnumerable();

        await foreach (var product in products)
        {
            // Process each product
            await ProcessProductAsync(product);
        }
    }

    private Task ProcessProductAsync(Product product) => Task.CompletedTask;
}
```

---

## Entity Configuration

### Fluent API in OnModelCreating

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace MyApp.Infrastructure.Data.Configurations;

// Separate configuration class (recommended approach)
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Table configuration
        builder.ToTable("Products", schema: "catalog");

        // Primary key
        builder.HasKey(p => p.Id);

        // Properties
        builder.Property(p => p.Id)
            .HasDefaultValueSql("NEWSEQUENTIALID()");

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200)
            .IsUnicode(true);

        builder.Property(p => p.Description)
            .HasMaxLength(2000)
            .IsUnicode(true);

        builder.Property(p => p.Price)
            .HasColumnType("decimal(18,2)")
            .IsRequired();

        builder.Property(p => p.Sku)
            .IsRequired()
            .HasMaxLength(50);

        // Indexes
        builder.HasIndex(p => p.Sku)
            .IsUnique()
            .HasDatabaseName("IX_Product_Sku");

        builder.HasIndex(p => p.Name)
            .HasDatabaseName("IX_Product_Name");

        builder.HasIndex(p => new { p.CategoryId, p.Price })
            .HasDatabaseName("IX_Product_Category_Price");

        // Relationships
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // Owned types
        builder.OwnsOne(p => p.Dimensions, dimensions =>
        {
            dimensions.Property(d => d.Width)
                .HasColumnType("decimal(10,2)")
                .HasColumnName("Width");

            dimensions.Property(d => d.Height)
                .HasColumnType("decimal(10,2)")
                .HasColumnName("Height");

            dimensions.Property(d => d.Depth)
                .HasColumnType("decimal(10,2)")
                .HasColumnName("Depth");
        });

        // Value converters for enums
        builder.Property(p => p.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        // Audit fields
        builder.Property(p => p.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        builder.Property(p => p.UpdatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        // Query filters (soft delete)
        builder.HasQueryFilter(p => !p.IsDeleted);

        // Concurrency token
        builder.Property(p => p.RowVersion)
            .IsRowVersion();
    }
}
```

### Complex Entity Configurations

```csharp
using System.Text.Json;

namespace MyApp.Infrastructure.Data.Configurations;

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders", schema: "sales");

        builder.HasKey(o => o.Id);

        // One-to-many relationship with cascade delete
        builder.HasMany(o => o.OrderItems)
            .WithOne(oi => oi.Order)
            .HasForeignKey(oi => oi.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // One-to-one relationship
        builder.HasOne(o => o.ShippingAddress)
            .WithOne()
            .HasForeignKey<ShippingAddress>(sa => sa.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // Many-to-many relationship (EF Core 5+)
        builder.HasMany(o => o.Tags)
            .WithMany(t => t.Orders)
            .UsingEntity<Dictionary<string, object>>(
                "OrderTag",
                j => j.HasOne<Tag>().WithMany().HasForeignKey("TagId"),
                j => j.HasOne<Order>().WithMany().HasForeignKey("OrderId"),
                j =>
                {
                    j.ToTable("OrderTags", schema: "sales");
                    j.HasKey("OrderId", "TagId");
                });

        // Value object as owned type
        builder.OwnsOne(o => o.ShippingAddress, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200).IsRequired();
            address.Property(a => a.City).HasMaxLength(100).IsRequired();
            address.Property(a => a.State).HasMaxLength(50).IsRequired();
            address.Property(a => a.ZipCode).HasMaxLength(20).IsRequired();
            address.Property(a => a.Country).HasMaxLength(100).IsRequired();
        });

        // Complex value converter
        builder.Property(o => o.Metadata)
            .HasConversion(
                v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null),
                v => JsonSerializer.Deserialize<Dictionary<string, string>>(v, (JsonSerializerOptions)null))
            .HasColumnType("nvarchar(max)");

        // Computed columns
        builder.Property(o => o.TotalAmount)
            .HasComputedColumnSql("[SubTotal] + [TaxAmount] + [ShippingCost]", stored: true);

        // Check constraints
        builder.HasCheckConstraint("CK_Order_TotalAmount", "[TotalAmount] >= 0");
        builder.HasCheckConstraint("CK_Order_Status", "[Status] IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled')");
    }
}
```

### Table-Per-Hierarchy (TPH) Inheritance

```csharp
namespace MyApp.Infrastructure.Data.Configurations;

public class PaymentConfiguration : IEntityTypeConfiguration<Payment>
{
    public void Configure(EntityTypeBuilder<Payment> builder)
    {
        builder.ToTable("Payments", schema: "sales");

        // Discriminator for inheritance
        builder.HasDiscriminator<string>("PaymentType")
            .HasValue<CreditCardPayment>("CreditCard")
            .HasValue<PayPalPayment>("PayPal")
            .HasValue<BankTransferPayment>("BankTransfer");

        builder.Property(p => p.Amount)
            .HasColumnType("decimal(18,2)")
            .IsRequired();
    }
}

public class CreditCardPaymentConfiguration : IEntityTypeConfiguration<CreditCardPayment>
{
    public void Configure(EntityTypeBuilder<CreditCardPayment> builder)
    {
        builder.Property(p => p.CardNumber)
            .HasMaxLength(16)
            .IsRequired();

        builder.Property(p => p.CardHolderName)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(p => p.ExpiryDate)
            .IsRequired();
    }
}
```

---

## Migration Management

### Creating Migrations

```bash
# Create a new migration
dotnet ef migrations add InitialCreate --project src/MyApp.Infrastructure --startup-project src/MyApp.API

# Create migration with specific context
dotnet ef migrations add AddProductTable --context ApplicationDbContext --project src/MyApp.Infrastructure

# Create migration with output directory
dotnet ef migrations add AddProductTable --output-dir Data/Migrations --project src/MyApp.Infrastructure
```

### Applying Migrations

```bash
# Update database to latest migration
dotnet ef database update --project src/MyApp.Infrastructure --startup-project src/MyApp.API

# Update to specific migration
dotnet ef database update AddProductTable --project src/MyApp.Infrastructure

# Generate SQL script instead of applying
dotnet ef migrations script --project src/MyApp.Infrastructure --output migrations.sql

# Generate idempotent script (safe for multiple runs)
dotnet ef migrations script --idempotent --project src/MyApp.Infrastructure --output migrations.sql

# Generate script from specific migration range
dotnet ef migrations script AddProductTable AddOrderTable --project src/MyApp.Infrastructure
```

### Removing Migrations

```bash
# Remove last migration (not applied to database)
dotnet ef migrations remove --project src/MyApp.Infrastructure

# Remove last migration (even if applied - WARNING: data loss possible)
dotnet ef migrations remove --force --project src/MyApp.Infrastructure
```

### Programmatic Migration Application

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.API;

public static class MigrationExtensions
{
    // Apply migrations on startup (development only)
    public static async Task ApplyMigrationsAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        // Get pending migrations
        var pendingMigrations = await context.Database.GetPendingMigrationsAsync();

        if (pendingMigrations.Any())
        {
            app.Logger.LogInformation(
                "Applying {Count} pending migrations: {Migrations}",
                pendingMigrations.Count(),
                string.Join(", ", pendingMigrations));

            await context.Database.MigrateAsync();
        }
    }

    // Safe migration check (production)
    public static async Task<bool> EnsureDatabaseUpToDateAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        var pendingMigrations = await context.Database.GetPendingMigrationsAsync();

        if (pendingMigrations.Any())
        {
            app.Logger.LogWarning(
                "Database has {Count} pending migrations. Application may not function correctly.",
                pendingMigrations.Count());

            return false;
        }

        return true;
    }
}

// Usage in Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Development: Auto-apply migrations
if (app.Environment.IsDevelopment())
{
    await app.ApplyMigrationsAsync();
}
// Production: Check and warn
else
{
    var isUpToDate = await app.EnsureDatabaseUpToDateAsync();
    if (!isUpToDate)
    {
        throw new InvalidOperationException("Database is not up to date. Apply migrations before starting the application.");
    }
}

app.Run();
```

### Custom Migration Operations

```csharp
using Microsoft.EntityFrameworkCore.Migrations;

namespace MyApp.Infrastructure.Data.Migrations;

public partial class AddFullTextSearch : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create full-text catalog
        migrationBuilder.Sql(@"
            IF NOT EXISTS (SELECT * FROM sys.fulltext_catalogs WHERE name = 'ProductCatalog')
            BEGIN
                CREATE FULLTEXT CATALOG ProductCatalog AS DEFAULT;
            END
        ");

        // Create full-text index
        migrationBuilder.Sql(@"
            CREATE FULLTEXT INDEX ON catalog.Products(Name, Description)
            KEY INDEX PK_Products
            ON ProductCatalog
            WITH STOPLIST = SYSTEM;
        ");

        // Create stored procedure
        migrationBuilder.Sql(@"
            CREATE PROCEDURE [catalog].[SearchProducts]
                @SearchTerm NVARCHAR(200)
            AS
            BEGIN
                SELECT * FROM catalog.Products
                WHERE CONTAINS((Name, Description), @SearchTerm)
                ORDER BY [Rank] DESC;
            END
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP PROCEDURE IF EXISTS [catalog].[SearchProducts]");
        migrationBuilder.Sql("DROP FULLTEXT INDEX ON catalog.Products");
        migrationBuilder.Sql("DROP FULLTEXT CATALOG ProductCatalog");
    }
}
```

---

## Seeding Strategies

### HasData Method (Simple Static Data)

```csharp
namespace MyApp.Infrastructure.Data.Configurations;

public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.ToTable("Categories", schema: "catalog");

        builder.HasKey(c => c.Id);

        builder.Property(c => c.Name)
            .IsRequired()
            .HasMaxLength(100);

        // Seed data
        builder.HasData(
            new Category { Id = 1, Name = "Electronics", CreatedAt = DateTime.UtcNow },
            new Category { Id = 2, Name = "Clothing", CreatedAt = DateTime.UtcNow },
            new Category { Id = 3, Name = "Books", CreatedAt = DateTime.UtcNow },
            new Category { Id = 4, Name = "Home & Garden", CreatedAt = DateTime.UtcNow }
        );
    }
}

// For related entities
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // ... other configuration ...

        builder.HasData(
            new Product
            {
                Id = 1,
                Name = "Laptop",
                Sku = "LAP-001",
                Price = 999.99m,
                CategoryId = 1, // Foreign key
                CreatedAt = DateTime.UtcNow
            },
            new Product
            {
                Id = 2,
                Name = "T-Shirt",
                Sku = "TSH-001",
                Price = 19.99m,
                CategoryId = 2,
                CreatedAt = DateTime.UtcNow
            }
        );
    }
}
```

### Custom Seeder (Complex Data)

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Data;

public class ApplicationDbContextSeeder
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ApplicationDbContextSeeder> _logger;

    public ApplicationDbContextSeeder(
        ApplicationDbContext context,
        ILogger<ApplicationDbContextSeeder> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task SeedAsync()
    {
        try
        {
            await SeedCategoriesAsync();
            await SeedProductsAsync();
            await SeedCustomersAsync();
            await SeedOrdersAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An error occurred while seeding the database");
            throw;
        }
    }

    private async Task SeedCategoriesAsync()
    {
        if (await _context.Categories.AnyAsync())
        {
            _logger.LogInformation("Categories already seeded");
            return;
        }

        var categories = new List<Category>
        {
            new() { Name = "Electronics", Description = "Electronic devices and accessories" },
            new() { Name = "Clothing", Description = "Apparel and fashion items" },
            new() { Name = "Books", Description = "Books and publications" },
            new() { Name = "Home & Garden", Description = "Home improvement and gardening" }
        };

        _context.Categories.AddRange(categories);
        await _context.SaveChangesAsync();

        _logger.LogInformation("Seeded {Count} categories", categories.Count);
    }

    private async Task SeedProductsAsync()
    {
        if (await _context.Products.AnyAsync())
        {
            _logger.LogInformation("Products already seeded");
            return;
        }

        var electronics = await _context.Categories
            .FirstAsync(c => c.Name == "Electronics");

        var products = new List<Product>();

        for (int i = 1; i <= 100; i++)
        {
            products.Add(new Product
            {
                Name = $"Product {i}",
                Sku = $"PRD-{i:D5}",
                Description = $"Description for product {i}",
                Price = Random.Shared.Next(10, 1000),
                CategoryId = electronics.Id,
                Status = ProductStatus.Active
            });
        }

        _context.Products.AddRange(products);
        await _context.SaveChangesAsync();

        _logger.LogInformation("Seeded {Count} products", products.Count);
    }

    private async Task SeedCustomersAsync()
    {
        if (await _context.Customers.AnyAsync())
        {
            return;
        }

        var customers = new List<Customer>();

        for (int i = 1; i <= 50; i++)
        {
            customers.Add(new Customer
            {
                FirstName = $"FirstName{i}",
                LastName = $"LastName{i}",
                Email = $"customer{i}@example.com",
                Phone = $"+1-555-{i:D4}"
            });
        }

        _context.Customers.AddRange(customers);
        await _context.SaveChangesAsync();

        _logger.LogInformation("Seeded {Count} customers", customers.Count);
    }

    private async Task SeedOrdersAsync()
    {
        if (await _context.Orders.AnyAsync())
        {
            return;
        }

        var customers = await _context.Customers.ToListAsync();
        var products = await _context.Products.Take(20).ToListAsync();

        var orders = new List<Order>();

        foreach (var customer in customers.Take(30))
        {
            var order = new Order
            {
                CustomerId = customer.Id,
                OrderDate = DateTime.UtcNow.AddDays(-Random.Shared.Next(1, 90)),
                Status = OrderStatus.Pending,
                OrderItems = new List<OrderItem>()
            };

            // Add 1-5 random products
            var itemCount = Random.Shared.Next(1, 6);
            for (int i = 0; i < itemCount; i++)
            {
                var product = products[Random.Shared.Next(products.Count)];
                order.OrderItems.Add(new OrderItem
                {
                    ProductId = product.Id,
                    Quantity = Random.Shared.Next(1, 5),
                    UnitPrice = product.Price
                });
            }

            orders.Add(order);
        }

        _context.Orders.AddRange(orders);
        await _context.SaveChangesAsync();

        _logger.LogInformation("Seeded {Count} orders", orders.Count);
    }
}
```

### Seeder Registration and Usage

```csharp
// In DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<ApplicationDbContextSeeder>();

        return services;
    }
}

// In Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Seed database on startup (development only)
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var services = scope.ServiceProvider;

    try
    {
        var context = services.GetRequiredService<ApplicationDbContext>();
        await context.Database.MigrateAsync();

        var seeder = services.GetRequiredService<ApplicationDbContextSeeder>();
        await seeder.SeedAsync();
    }
    catch (Exception ex)
    {
        var logger = services.GetRequiredService<ILogger<Program>>();
        logger.LogError(ex, "An error occurred while migrating or seeding the database");
        throw;
    }
}

app.Run();
```

---

## Connection Strings

### Configuration in appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyAppDb;Trusted_Connection=True;MultipleActiveResultSets=true",
    "ReadOnlyConnection": "Server=(localdb)\\mssqllocaldb;Database=MyAppDb;Trusted_Connection=True;ApplicationIntent=ReadOnly",
    "AzureSql": "Server=tcp:myserver.database.windows.net,1433;Database=MyAppDb;User ID=myuser;Password=mypassword;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
    "PostgreSQL": "Host=localhost;Database=myapp;Username=postgres;Password=password",
    "SQLite": "Data Source=myapp.db"
  }
}
```

### Environment-Specific Configuration

```json
// appsettings.Development.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyAppDb_Dev;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}

// appsettings.Production.json
{
  "ConnectionStrings": {
    "DefaultConnection": "" // Set via environment variable or Azure Key Vault
  }
}
```

### Secure Connection String Management

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Extensions.Configuration;

namespace MyApp.API;

public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Add Azure Key Vault
        if (!builder.Environment.IsDevelopment())
        {
            var keyVaultEndpoint = builder.Configuration["KeyVaultEndpoint"];
            if (!string.IsNullOrEmpty(keyVaultEndpoint))
            {
                builder.Configuration.AddAzureKeyVault(
                    new Uri(keyVaultEndpoint),
                    new DefaultAzureCredential());
            }
        }

        // Add User Secrets in development
        if (builder.Environment.IsDevelopment())
        {
            builder.Configuration.AddUserSecrets<Program>();
        }

        builder.Services.AddInfrastructure(builder.Configuration);

        var app = builder.Build();
        app.Run();
    }
}
```

### Connection String Builder

```csharp
using Microsoft.Data.SqlClient;

namespace MyApp.Infrastructure.Configuration;

public class ConnectionStringBuilder
{
    public static string BuildSqlServerConnectionString(
        string server,
        string database,
        string userId = null,
        string password = null,
        bool integratedSecurity = true,
        int connectionTimeout = 30,
        bool multipleActiveResultSets = true,
        bool encrypt = true)
    {
        var builder = new SqlConnectionStringBuilder
        {
            DataSource = server,
            InitialCatalog = database,
            IntegratedSecurity = integratedSecurity,
            MultipleActiveResultSets = multipleActiveResultSets,
            ConnectTimeout = connectionTimeout,
            Encrypt = encrypt,
            TrustServerCertificate = false
        };

        if (!integratedSecurity)
        {
            builder.UserID = userId;
            builder.Password = password;
        }

        return builder.ConnectionString;
    }

    public static string BuildAzureSqlConnectionString(
        string server,
        string database,
        string userId,
        string password,
        bool useManagedIdentity = false)
    {
        var builder = new SqlConnectionStringBuilder
        {
            DataSource = server,
            InitialCatalog = database,
            Encrypt = true,
            TrustServerCertificate = false,
            ConnectTimeout = 30
        };

        if (useManagedIdentity)
        {
            builder.Authentication = SqlAuthenticationMethod.ActiveDirectoryManagedIdentity;
        }
        else
        {
            builder.UserID = userId;
            builder.Password = password;
        }

        return builder.ConnectionString;
    }
}
```

---

## Multiple DbContext Scenarios

### Multiple Databases

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Data;

// Main application database
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
}

// Identity/authentication database
public class IdentityDbContext : DbContext
{
    public IdentityDbContext(DbContextOptions<IdentityDbContext> options)
        : base(options)
    {
    }

    public DbSet<User> Users => Set<User>();
    public DbSet<Role> Roles => Set<Role>();
}

// Logging/audit database
public class AuditDbContext : DbContext
{
    public AuditDbContext(DbContextOptions<AuditDbContext> options)
        : base(options)
    {
    }

    public DbSet<AuditLog> AuditLogs => Set<AuditLog>();
    public DbSet<PerformanceLog> PerformanceLogs => Set<PerformanceLog>();
}
```

### Registration of Multiple Contexts

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Main database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly("MyApp.Infrastructure")));

        // Identity database
        services.AddDbContext<IdentityDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("IdentityConnection"),
                b => b.MigrationsAssembly("MyApp.Infrastructure")));

        // Audit database (separate server, read-only from app perspective)
        services.AddDbContext<AuditDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("AuditConnection"),
                b => b.MigrationsAssembly("MyApp.Infrastructure")));

        return services;
    }
}
```

### Bounded Context Pattern

```csharp
namespace MyApp.Infrastructure.Data;

// Sales bounded context
public class SalesDbContext : DbContext
{
    public SalesDbContext(DbContextOptions<SalesDbContext> options)
        : base(options)
    {
    }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Invoice> Invoices => Set<Invoice>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.HasDefaultSchema("sales");
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(SalesDbContext).Assembly,
            t => t.Namespace?.Contains("Sales") == true);
    }
}

// Catalog bounded context
public class CatalogDbContext : DbContext
{
    public CatalogDbContext(DbContextOptions<CatalogDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Brand> Brands => Set<Brand>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.HasDefaultSchema("catalog");
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(CatalogDbContext).Assembly,
            t => t.Namespace?.Contains("Catalog") == true);
    }
}
```

### Managing Multiple Context Migrations

```bash
# Create migrations for specific context
dotnet ef migrations add InitialCreate --context ApplicationDbContext --project src/MyApp.Infrastructure --output-dir Data/Migrations/Application

dotnet ef migrations add InitialCreate --context IdentityDbContext --project src/MyApp.Infrastructure --output-dir Data/Migrations/Identity

dotnet ef migrations add InitialCreate --context AuditDbContext --project src/MyApp.Infrastructure --output-dir Data/Migrations/Audit

# Update specific database
dotnet ef database update --context ApplicationDbContext --project src/MyApp.Infrastructure

# Generate script for specific context
dotnet ef migrations script --context IdentityDbContext --idempotent --project src/MyApp.Infrastructure --output identity-migrations.sql
```

### Cross-Context Coordination

```csharp
using Microsoft.EntityFrameworkCore.Storage;

namespace MyApp.Infrastructure.Services;

public class CrossContextService
{
    private readonly ApplicationDbContext _appContext;
    private readonly AuditDbContext _auditContext;

    public CrossContextService(
        ApplicationDbContext appContext,
        AuditDbContext auditContext)
    {
        _appContext = appContext;
        _auditContext = auditContext;
    }

    public async Task CreateOrderWithAuditAsync(Order order)
    {
        // Use execution strategy for retry logic
        var strategy = _appContext.Database.CreateExecutionStrategy();

        await strategy.ExecuteAsync(async () =>
        {
            // Begin transaction in main context
            await using var transaction = await _appContext.Database.BeginTransactionAsync();

            try
            {
                // Save order
                _appContext.Orders.Add(order);
                await _appContext.SaveChangesAsync();

                // Log to audit database (separate transaction)
                _auditContext.AuditLogs.Add(new AuditLog
                {
                    EntityType = nameof(Order),
                    EntityId = order.Id.ToString(),
                    Action = "Create",
                    Timestamp = DateTime.UtcNow,
                    UserId = order.CreatedBy
                });
                await _auditContext.SaveChangesAsync();

                // Commit main transaction
                await transaction.CommitAsync();
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

## Best Practices

### 1. DbContext Lifetime
- Use scoped lifetime (default) for web applications
- Use DbContext pooling for high-performance scenarios
- Use factory pattern for long-running operations or background services

### 2. Configuration
- Separate entity configurations into individual classes
- Use `ApplyConfigurationsFromAssembly` for automatic discovery
- Keep migrations in a separate folder structure

### 3. Migrations
- Always review generated migrations before applying
- Use idempotent scripts for production deployments
- Never modify applied migrations
- Keep migration names descriptive

### 4. Seeding
- Use `HasData` for static reference data
- Use custom seeders for complex or large datasets
- Check for existing data before seeding
- Only seed in development/staging environments

### 5. Performance
- Use `AsNoTracking()` for read-only queries
- Enable query splitting for collections
- Use compiled queries for frequently executed queries
- Monitor and optimize slow queries

---

## References

- [EF Core Documentation](https://docs.microsoft.com/en-us/ef/core/)
- [EF Core DbContext Configuration](https://docs.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [EF Core Migrations](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [EF Core Data Seeding](https://docs.microsoft.com/en-us/ef/core/modeling/data-seeding)
- [Connection Strings](https://docs.microsoft.com/en-us/ef/core/miscellaneous/connection-strings)
- [Multiple DbContext](https://docs.microsoft.com/en-us/ef/core/dbcontext-configuration/#using-more-than-one-dbcontext-instance)

---

**Last Updated**: 2025-10-06
**EF Core Version**: 8.0
**.NET Version**: 8.0
