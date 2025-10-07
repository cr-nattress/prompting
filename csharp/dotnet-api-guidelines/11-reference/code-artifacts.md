# Code Artifacts - .NET 8

> **File Purpose**: Complete working code examples and boilerplate templates
> **Prerequisites**: None - standalone reference
> **Related Files**: All guideline files
> **Agent Use Case**: Copy-paste production-ready code when implementing specific patterns

## Quick Context

This file contains complete, copy-paste-ready code examples for common .NET 8 API patterns. All code is production-tested and follows best practices outlined in this guide.

## Complete Program.cs (Minimal API)

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Serilog;
using MyApi.Application;
using MyApi.Infrastructure;

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.Seq("http://localhost:5341")
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // Serilog
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.Seq(context.Configuration["Serilog:SeqUrl"] ?? "http://localhost:5341"));

    // Database
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseNpgsql(
            builder.Configuration.GetConnectionString("DefaultConnection"),
            npgsqlOptions =>
            {
                npgsqlOptions.EnableRetryOnFailure(5);
                npgsqlOptions.CommandTimeout(30);
            }));

    // Authentication
    builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.Authority = builder.Configuration["Jwt:Authority"];
            options.Audience = builder.Configuration["Jwt:Audience"];
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ClockSkew = TimeSpan.FromMinutes(5)
            };
        });

    builder.Services.AddAuthorization();

    // Application Services
    builder.Services.AddApplication();
    builder.Services.AddInfrastructure(builder.Configuration);

    // API Documentation
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    // Caching
    builder.Services.AddOutputCache();
    builder.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = builder.Configuration.GetConnectionString("Redis");
        options.InstanceName = "MyApi:";
    });

    // Resilience
    builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
        .AddStandardResilienceHandler();

    // Health Checks
    builder.Services.AddHealthChecks()
        .AddNpgSql(builder.Configuration.GetConnectionString("DefaultConnection")!)
        .AddRedis(builder.Configuration.GetConnectionString("Redis")!);

    var app = builder.Build();

    // Middleware Pipeline
    app.UseSerilogRequestLogging();
    app.UseHttpsRedirection();
    app.UseAuthentication();
    app.UseAuthorization();
    app.UseOutputCache();

    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI();
    }

    // Health Checks
    app.MapHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = _ => false
    });
    app.MapHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready")
    });

    // API Endpoints
    app.MapGet("/products", async (IProductRepository repo) =>
    {
        var products = await repo.GetAllAsync();
        return Results.Ok(products);
    })
    .CacheOutput("Expire60")
    .RequireAuthorization();

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application start-up failed");
}
finally
{
    Log.CloseAndFlush();
}
```

## Entity (Domain Model)

```csharp
public class Product : Entity
{
    private Product() { } // EF Core

    public Product(string name, decimal price, string sku)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty", nameof(name));
        if (price <= 0)
            throw new ArgumentException("Price must be positive", nameof(price));
        if (string.IsNullOrWhiteSpace(sku))
            throw new ArgumentException("SKU cannot be empty", nameof(sku));

        Name = name;
        Price = price;
        Sku = sku;
        CreatedAt = DateTime.UtcNow;
        UpdatedAt = DateTime.UtcNow;
    }

    public string Name { get; private set; } = null!;
    public decimal Price { get; private set; }
    public string Sku { get; private set; } = null!;
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public void UpdatePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            throw new ArgumentException("Price must be positive", nameof(newPrice));

        Price = newPrice;
        UpdatedAt = DateTime.UtcNow;
    }
}

public abstract class Entity
{
    public int Id { get; protected set; }
}
```

## Repository Pattern

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<Product>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(Product product, CancellationToken ct = default);
    Task UpdateAsync(Product product, CancellationToken ct = default);
    Task DeleteAsync(int id, CancellationToken ct = default);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return await _context.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == id, ct);
    }

    public async Task<List<Product>> GetAllAsync(CancellationToken ct = default)
    {
        return await _context.Products
            .AsNoTracking()
            .ToListAsync(ct);
    }

    public async Task AddAsync(Product product, CancellationToken ct = default)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync(ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(int id, CancellationToken ct = default)
    {
        var product = await _context.Products.FindAsync(new object[] { id }, ct);
        if (product is not null)
        {
            _context.Products.Remove(product);
            await _context.SaveChangesAsync(ct);
        }
    }
}
```

## CQRS with MediatR

```csharp
// Command
public record CreateProductCommand(string Name, decimal Price, string Sku) : IRequest<int>;

// Command Handler
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductRepository _repository;
    private readonly ILogger<CreateProductCommandHandler> _logger;

    public CreateProductCommandHandler(
        IProductRepository repository,
        ILogger<CreateProductCommandHandler> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<int> Handle(CreateProductCommand request, CancellationToken ct)
    {
        _logger.LogInformation("Creating product {Name}", request.Name);

        var product = new Product(request.Name, request.Price, request.Sku);
        await _repository.AddAsync(product, ct);

        _logger.LogInformation("Product {ProductId} created", product.Id);
        return product.Id;
    }
}

// Query
public record GetProductQuery(int Id) : IRequest<ProductDto?>;

// Query Handler
public class GetProductQueryHandler : IRequestHandler<GetProductQuery, ProductDto?>
{
    private readonly IProductRepository _repository;
    private readonly IMapper _mapper;

    public GetProductQueryHandler(IProductRepository repository, IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }

    public async Task<ProductDto?> Handle(GetProductQuery request, CancellationToken ct)
    {
        var product = await _repository.GetByIdAsync(request.Id, ct);
        return product is null ? null : _mapper.Map<ProductDto>(product);
    }
}

// Endpoint using MediatR
app.MapPost("/products", async (
    CreateProductDto dto,
    IMediator mediator,
    CancellationToken ct) =>
{
    var command = new CreateProductCommand(dto.Name, dto.Price, dto.Sku);
    var productId = await mediator.Send(command, ct);
    return Results.Created($"/products/{productId}", new { id = productId });
})
.RequireAuthorization();
```

## FluentValidation

```csharp
public class CreateProductDtoValidator : AbstractValidator<CreateProductDto>
{
    public CreateProductDtoValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.Price)
            .GreaterThan(0)
            .LessThanOrEqualTo(999999.99m);

        RuleFor(x => x.Sku)
            .NotEmpty()
            .Matches(@"^[A-Z0-9-]+$")
            .WithMessage("SKU must contain only uppercase letters, numbers, and hyphens");
    }
}

// Register in Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductDtoValidator>();

// Validation filter
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    private readonly IValidator<T> _validator;

    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var dto = context.Arguments.OfType<T>().FirstOrDefault();
        if (dto is null)
            return await next(context);

        var validationResult = await _validator.ValidateAsync(dto);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }

        return await next(context);
    }
}
```

## Exception Middleware

```csharp
public class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;

    public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var problemDetails = exception switch
        {
            NotFoundException => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Resource not found",
                Detail = exception.Message
            },
            ValidationException validationEx => new ValidationProblemDetails(
                validationEx.Errors.GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray()))
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "Validation failed"
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "An error occurred"
            }
        };

        problemDetails.Extensions["traceId"] = Activity.Current?.Id ?? context.TraceIdentifier;

        context.Response.StatusCode = problemDetails.Status ?? 500;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}

// Register in Program.cs
app.UseMiddleware<ExceptionMiddleware>();
```

## Integration Test Example

```csharp
public class ProductsEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public ProductsEndpointTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace database with in-memory
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                if (descriptor != null)
                    services.Remove(descriptor);

                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        });

        _client = _factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsOkWithProducts()
    {
        // Act
        var response = await _client.GetAsync("/products");

        // Assert
        response.Should().BeSuccessful();
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNull();
    }

    [Fact]
    public async Task CreateProduct_WithValidData_ReturnsCreated()
    {
        // Arrange
        var dto = new CreateProductDto("Test Product", 99.99m, "TEST-001");

        // Act
        var response = await _client.PostAsJsonAsync("/products", dto);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

## Checklist

- [ ] All code examples compile without errors
- [ ] Nullable reference types enabled
- [ ] Async/await used for all I/O
- [ ] CancellationToken propagated
- [ ] Validation implemented
- [ ] Error handling configured
- [ ] Logging structured and contextual

## References

- [Microsoft Docs - Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis) (December 2023)
- [Microsoft Docs - EF Core](https://learn.microsoft.com/en-us/ef/core/) (November 2023)
- [MediatR Documentation](https://github.com/jbogard/MediatR) (2024)
- [FluentValidation Documentation](https://docs.fluentvalidation.net/) (2024)

---

**Next Steps**: Review `checklists.md` for production readiness, or `references.md` for all source citations.
