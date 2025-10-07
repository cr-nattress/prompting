# Minimal Program Setup - .NET 8

> **File Purpose**: Configure Program.cs with dependency injection, middleware pipeline, and options pattern
> **Prerequisites**: `project-bootstrap.md` - Project scaffolding completed
> **Related Files**: `first-endpoint.md`, `../06-error-handling/exception-middleware.md`, `../07-observability/structured-logging.md`
> **Agent Use Case**: Reference when setting up Program.cs composition and middleware pipeline

## Quick Context

.NET 8 uses top-level statements and minimal hosting model for cleaner Program.cs composition. This guide covers dependency injection configuration, middleware pipeline ordering, options pattern for configuration, and production-ready Program.cs structure following Microsoft best practices.

**Microsoft References**:
- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview)
- [Dependency Injection](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Options Pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)

## Basic Program.cs Structure

### Minimal Setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add services to DI container
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Define endpoints
app.MapGet("/", () => "Hello World!");

app.Run();
```

## Dependency Injection Configuration

### Service Registration Patterns

```csharp
// Program.cs
using MyApi.Application.Common.Interfaces;
using MyApi.Infrastructure.Services;
using MyApi.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

// Register services with different lifetimes

// Transient - Created each time requested
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddTransient<IDateTime, DateTimeService>();

// Scoped - Created once per request
builder.Services.AddScoped<IApplicationDbContext, ApplicationDbContext>();
builder.Services.AddScoped<IOrderService, OrderService>();

// Singleton - Created once for application lifetime
builder.Services.AddSingleton<IMetricsCollector, MetricsCollector>();
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();

var app = builder.Build();
app.Run();
```

**Lifetime Guidelines**:
- **Transient**: Lightweight, stateless services
- **Scoped**: DbContext, request-specific services, UnitOfWork
- **Singleton**: Caching, logging, configuration, metrics
- **Never**: Register DbContext as Singleton (causes threading issues)

### Service Registration Extensions

Create extension methods for clean organization:

```csharp
// MyApi.Infrastructure/DependencyInjection.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using MyApi.Application.Common.Interfaces;
using MyApi.Infrastructure.Persistence;
using MyApi.Infrastructure.Services;

namespace MyApi.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)
            )
        );

        services.AddScoped<IApplicationDbContext>(provider =>
            provider.GetRequiredService<ApplicationDbContext>());

        // Services
        services.AddTransient<IDateTime, DateTimeService>();
        services.AddTransient<IEmailService, EmailService>();

        return services;
    }
}
```

```csharp
// MyApi.Application/DependencyInjection.cs
using System.Reflection;
using FluentValidation;
using MediatR;
using Microsoft.Extensions.DependencyInjection;

namespace MyApi.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // MediatR
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly())
        );

        // FluentValidation
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        // AutoMapper/Mapster
        services.AddMapster();

        return services;
    }
}
```

**Program.cs with Extensions**:
```csharp
using MyApi.Application;
using MyApi.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// Layer-based service registration
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();
app.Run();
```

## Options Pattern Configuration

### Configuration Class

```csharp
// MyApi.Api/Configuration/JwtSettings.cs
namespace MyApi.Api.Configuration;

public class JwtSettings
{
    public const string SectionName = "Jwt";

    public required string Issuer { get; init; }
    public required string Audience { get; init; }
    public required string SecretKey { get; init; }
    public int ExpirationMinutes { get; init; } = 60;
}
```

### appsettings.json

```json
{
  "Jwt": {
    "Issuer": "https://api.example.com",
    "Audience": "https://api.example.com",
    "SecretKey": "your-secret-key-here",
    "ExpirationMinutes": 60
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApiDb;Trusted_Connection=true;TrustServerCertificate=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  }
}
```

### Register Options

```csharp
// Program.cs
using MyApi.Api.Configuration;

var builder = WebApplication.CreateBuilder(args);

// Bind configuration section to strongly-typed options
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName)
);

// Validate options on startup
builder.Services.AddOptions<JwtSettings>()
    .Bind(builder.Configuration.GetSection(JwtSettings.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();

var app = builder.Build();
```

### Use Options in Services

```csharp
// MyApi.Infrastructure/Services/TokenService.cs
using Microsoft.Extensions.Options;
using MyApi.Api.Configuration;

public class TokenService : ITokenService
{
    private readonly JwtSettings _jwtSettings;

    public TokenService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }

    public string GenerateToken(string userId)
    {
        // Use _jwtSettings.Issuer, _jwtSettings.SecretKey, etc.
        return "generated-token";
    }
}
```

### Named Options

```csharp
// For multiple configurations of same type
builder.Services.Configure<EmailSettings>("SendGrid",
    builder.Configuration.GetSection("Email:SendGrid"));

builder.Services.Configure<EmailSettings>("Smtp",
    builder.Configuration.GetSection("Email:Smtp"));

// Inject with IOptionsSnapshot
public class EmailService
{
    private readonly IOptionsSnapshot<EmailSettings> _emailSettings;

    public EmailService(IOptionsSnapshot<EmailSettings> emailSettings)
    {
        _emailSettings = emailSettings;
    }

    public void SendViaSendGrid()
    {
        var settings = _emailSettings.Get("SendGrid");
    }
}
```

## Middleware Pipeline

### Correct Middleware Order

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddAuthentication();
builder.Services.AddAuthorization();

var app = builder.Build();

// IMPORTANT: Middleware order matters!

// 1. Exception handling (must be first to catch all errors)
app.UseExceptionHandler("/error");

// 2. HSTS (HTTP Strict Transport Security)
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

// 3. HTTPS redirection
app.UseHttpsRedirection();

// 4. Static files (if any)
// app.UseStaticFiles();

// 5. Routing
app.UseRouting();

// 6. CORS (must be after routing, before auth)
app.UseCors();

// 7. Authentication (must be before authorization)
app.UseAuthentication();

// 8. Authorization
app.UseAuthorization();

// 9. Custom middleware
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<RequestLoggingMiddleware>();

// 10. Endpoints (must be last)
app.MapControllers();

app.Run();
```

**Critical Order Rules**:
1. Exception handling FIRST
2. HTTPS redirection BEFORE authentication
3. CORS AFTER routing, BEFORE authentication
4. Authentication BEFORE authorization
5. Endpoints LAST

### Custom Middleware

```csharp
// MyApi.Api/Middleware/CorrelationIdMiddleware.cs
namespace MyApi.Api.Middleware;

public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Get or generate correlation ID
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // Add to response headers
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        // Add to HttpContext for logging
        context.Items["CorrelationId"] = correlationId;

        await _next(context);
    }
}

// Extension method for clean registration
public static class CorrelationIdMiddlewareExtensions
{
    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder app)
    {
        return app.UseMiddleware<CorrelationIdMiddleware>();
    }
}
```

**Usage**:
```csharp
// Program.cs
app.UseCorrelationId();
```

## Complete Production Program.cs

```csharp
// Program.cs
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using Microsoft.OpenApi.Models;
using Serilog;
using MyApi.Api.Configuration;
using MyApi.Api.Middleware;
using MyApi.Application;
using MyApi.Infrastructure;

// Configure Serilog early
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting web application");

    var builder = WebApplication.CreateBuilder(args);

    // Configure Serilog
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    );

    // Configuration options
    builder.Services.Configure<JwtSettings>(
        builder.Configuration.GetSection(JwtSettings.SectionName)
    );

    // Application layers
    builder.Services.AddApplication();
    builder.Services.AddInfrastructure(builder.Configuration);

    // Authentication
    var jwtSettings = builder.Configuration.GetSection("Jwt");
    builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidIssuer = jwtSettings["Issuer"],
                ValidateAudience = true,
                ValidAudience = jwtSettings["Audience"],
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(jwtSettings["SecretKey"]!)
                ),
                ValidateLifetime = true,
                ClockSkew = TimeSpan.FromMinutes(5)
            };
        });

    builder.Services.AddAuthorization();

    // CORS
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("AllowAll", policy =>
        {
            policy.AllowAnyOrigin()
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
    });

    // API versioning
    builder.Services.AddApiVersioning(options =>
    {
        options.DefaultApiVersion = new ApiVersion(1, 0);
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.ReportApiVersions = true;
    });

    // Swagger/OpenAPI
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo
        {
            Title = "My API",
            Version = "v1",
            Description = "Production API with .NET 8"
        });

        // JWT authentication in Swagger
        c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
        {
            Description = "JWT Authorization header using the Bearer scheme",
            Name = "Authorization",
            In = ParameterLocation.Header,
            Type = SecuritySchemeType.ApiKey,
            Scheme = "Bearer"
        });

        c.AddSecurityRequirement(new OpenApiSecurityRequirement
        {
            {
                new OpenApiSecurityScheme
                {
                    Reference = new OpenApiReference
                    {
                        Type = ReferenceType.SecurityScheme,
                        Id = "Bearer"
                    }
                },
                Array.Empty<string>()
            }
        });
    });

    // Health checks
    builder.Services.AddHealthChecks()
        .AddDbContextCheck<ApplicationDbContext>();

    // Response compression
    builder.Services.AddResponseCompression(options =>
    {
        options.EnableForHttps = true;
    });

    // Output caching
    builder.Services.AddOutputCache(options =>
    {
        options.AddBasePolicy(builder => builder.Cache());
    });

    var app = builder.Build();

    // Middleware pipeline
    app.UseSerilogRequestLogging();

    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI();
    }
    else
    {
        app.UseExceptionHandler("/error");
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseResponseCompression();

    // CORS
    app.UseCors("AllowAll");

    // Custom middleware
    app.UseCorrelationId();

    // Security
    app.UseAuthentication();
    app.UseAuthorization();

    // Caching
    app.UseOutputCache();

    // Health checks
    app.MapHealthChecks("/health");

    // Map endpoints
    app.MapGet("/", () => "API is running")
        .WithName("Root")
        .AllowAnonymous();

    // Run database migrations (dev only)
    if (app.Environment.IsDevelopment())
    {
        using var scope = app.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        await dbContext.Database.MigrateAsync();
    }

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}

// Make Program class accessible for integration tests
public partial class Program { }
```

## Environment-Specific Configuration

### appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApiDb_Dev;Trusted_Connection=true;TrustServerCertificate=true"
  }
}
```

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "#{ConnectionString}#"
  },
  "Jwt": {
    "SecretKey": "#{JwtSecretKey}#"
  }
}
```

### User Secrets (Development)

```bash
# Initialize user secrets
dotnet user-secrets init --project src/MyApi.Api

# Set secrets
dotnet user-secrets set "Jwt:SecretKey" "super-secret-key-min-32-chars-long!!" --project src/MyApi.Api
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=MyApiDb;..." --project src/MyApi.Api

# List secrets
dotnet user-secrets list --project src/MyApi.Api
```

## Service Scope Patterns

### Scoped Service in Singleton

```csharp
// WRONG - Don't inject scoped into singleton!
public class MySingletonService
{
    private readonly ApplicationDbContext _context; // WRONG!

    public MySingletonService(ApplicationDbContext context)
    {
        _context = context; // Captive dependency!
    }
}

// CORRECT - Create scope when needed
public class MySingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MySingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task DoWork()
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        // Use context within this scope
        await context.SaveChangesAsync();
    }
}
```

### Background Service with Scoped Dependencies

```csharp
// MyApi.Api/BackgroundServices/DataSyncService.cs
public class DataSyncService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<DataSyncService> _logger;

    public DataSyncService(
        IServiceScopeFactory scopeFactory,
        ILogger<DataSyncService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

                // Perform work with scoped service
                await PerformSync(dbContext, stoppingToken);

                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in data sync service");
            }
        }
    }

    private async Task PerformSync(ApplicationDbContext context, CancellationToken ct)
    {
        // Sync logic
    }
}

// Register in Program.cs
builder.Services.AddHostedService<DataSyncService>();
```

## Configuration Validation

### Using Data Annotations

```csharp
using System.ComponentModel.DataAnnotations;

public class EmailSettings
{
    public const string SectionName = "Email";

    [Required]
    [EmailAddress]
    public required string FromAddress { get; init; }

    [Required]
    public required string SmtpServer { get; init; }

    [Range(1, 65535)]
    public int SmtpPort { get; init; } = 587;

    [Required]
    public required string Username { get; init; }

    [Required]
    public required string Password { get; init; }
}

// Program.cs - Validates on startup
builder.Services.AddOptions<EmailSettings>()
    .Bind(builder.Configuration.GetSection(EmailSettings.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### Custom Validation

```csharp
public class DatabaseSettings
{
    public required string ConnectionString { get; init; }
    public int MaxRetryCount { get; init; }
    public int CommandTimeout { get; init; }
}

public class DatabaseSettingsValidator : IValidateOptions<DatabaseSettings>
{
    public ValidateOptionsResult Validate(string? name, DatabaseSettings options)
    {
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(options.ConnectionString))
        {
            errors.Add("ConnectionString is required");
        }

        if (options.MaxRetryCount < 0)
        {
            errors.Add("MaxRetryCount must be non-negative");
        }

        if (options.CommandTimeout < 0)
        {
            errors.Add("CommandTimeout must be non-negative");
        }

        return errors.Count > 0
            ? ValidateOptionsResult.Fail(errors)
            : ValidateOptionsResult.Success;
    }
}

// Register validator
builder.Services.AddSingleton<IValidateOptions<DatabaseSettings>, DatabaseSettingsValidator>();
```

## Common Patterns Checklist

- [ ] Services registered with correct lifetime (Transient/Scoped/Singleton)
- [ ] Configuration bound to strongly-typed options classes
- [ ] Options validated on startup with ValidateOnStart()
- [ ] Middleware pipeline in correct order
- [ ] Authentication before Authorization
- [ ] CORS after routing, before authentication
- [ ] Exception handling first in pipeline
- [ ] Serilog configured for structured logging
- [ ] User secrets for development configuration
- [ ] Environment-specific appsettings files
- [ ] Health checks configured
- [ ] Swagger/OpenAPI enabled for development
- [ ] HTTPS redirection enabled
- [ ] HSTS enabled for production
- [ ] Program class partial for test access

## Navigation

- **Previous**: `project-bootstrap.md` - Project scaffolding
- **Next**: `first-endpoint.md` - Create first API endpoint
- **Related**: `../07-observability/structured-logging.md`, `../06-error-handling/exception-middleware.md`

## References

- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview)
- [Dependency Injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Options Pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)
- [Configuration in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
- [Service Lifetimes](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-lifetimes)
