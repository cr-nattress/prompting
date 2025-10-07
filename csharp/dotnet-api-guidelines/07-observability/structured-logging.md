# Structured Logging with Serilog - .NET 8

> **File Purpose**: Configure Serilog for structured logging with enrichers, request logging, and correlation IDs
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Program.cs setup
> **Related Files**: `opentelemetry.md`, `health-checks.md`, `../06-error-handling/exception-middleware.md`
> **Agent Use Case**: Reference when implementing structured logging for production observability

## Quick Context

Structured logging captures log events as structured data (JSON) rather than plain text, enabling powerful querying and analysis. Serilog is the most popular structured logging library for .NET, supporting multiple sinks (console, file, Seq, Elasticsearch) and rich contextual data.

**Microsoft References**:
- [Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging)
- [Serilog](https://serilog.net/)

## Install Packages

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
dotnet add package Serilog.Enrichers.Process
dotnet add package Serilog.Expressions  # For filtering
```

## Basic Serilog Configuration

```csharp
// Program.cs
using Serilog;
using Serilog.Events;

// Create bootstrap logger for startup errors
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting web application");

    var builder = WebApplication.CreateBuilder(args);

    // Configure Serilog from appsettings.json
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithEnvironmentName()
        .Enrich.WithMachineName()
        .Enrich.WithThreadId()
        .Enrich.WithProcessId()
        .WriteTo.Console(
            outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        )
        .WriteTo.File(
            "logs/log-.txt",
            rollingInterval: RollingInterval.Day,
            outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        )
    );

    var app = builder.Build();

    // Use Serilog request logging
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
            diagnosticContext.Set("CorrelationId", httpContext.Items["CorrelationId"]);
        };
    });

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
```

## appsettings.json Configuration

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}",
          "retainedFileCountLimit": 31
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"],
    "Properties": {
      "Application": "MyApi"
    }
  }
}
```

## Structured Logging Patterns

### Using ILogger

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;

    public UserService(
        IUserRepository repository,
        ILogger<UserService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<User> GetUserByIdAsync(int id, CancellationToken ct)
    {
        _logger.LogInformation("Fetching user with ID {UserId}", id);

        var user = await _repository.GetByIdAsync(id, ct);

        if (user is null)
        {
            _logger.LogWarning("User with ID {UserId} not found", id);
            throw new NotFoundException(nameof(User), id);
        }

        _logger.LogInformation("Successfully retrieved user {UserId} ({UserEmail})", user.Id, user.Email);

        return user;
    }

    public async Task<User> CreateUserAsync(CreateUserRequest request, CancellationToken ct)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["UserEmail"] = request.Email
        }))
        {
            _logger.LogInformation("Creating new user");

            var user = request.Adapt<User>();
            user.CreatedAt = DateTime.UtcNow;

            await _repository.AddAsync(user, ct);

            _logger.LogInformation("User created successfully with ID {UserId}", user.Id);

            return user;
        }
    }

    public async Task DeleteUserAsync(int id, CancellationToken ct)
    {
        try
        {
            _logger.LogInformation("Attempting to delete user {UserId}", id);

            var user = await _repository.GetByIdAsync(id, ct);
            if (user is null)
            {
                _logger.LogWarning("Cannot delete user {UserId}: not found", id);
                throw new NotFoundException(nameof(User), id);
            }

            await _repository.DeleteAsync(user, ct);

            _logger.LogInformation("User {UserId} deleted successfully", id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting user {UserId}", id);
            throw;
        }
    }
}
```

### Log Levels

```csharp
// Trace - Very detailed, typically only in development
_logger.LogTrace("Entering method GetUserByIdAsync with ID {UserId}", id);

// Debug - Internal system events
_logger.LogDebug("Cache miss for user {UserId}, fetching from database", id);

// Information - General flow events
_logger.LogInformation("User {UserId} logged in successfully", user.Id);

// Warning - Abnormal or unexpected events
_logger.LogWarning("Login attempt failed for user {UserEmail}", email);

// Error - Errors and exceptions
_logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);

// Critical - Critical failures requiring immediate attention
_logger.LogCritical("Database connection failed after {RetryCount} retries", retryCount);
```

## Custom Enrichers

### Correlation ID Enricher

```csharp
// MyApi.Api/Logging/CorrelationIdEnricher.cs
using Serilog.Core;
using Serilog.Events;

public class CorrelationIdEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CorrelationIdEnricher(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var correlationId = _httpContextAccessor.HttpContext?.Items["CorrelationId"]?.ToString()
            ?? _httpContextAccessor.HttpContext?.TraceIdentifier;

        if (!string.IsNullOrEmpty(correlationId))
        {
            var property = propertyFactory.CreateProperty("CorrelationId", correlationId);
            logEvent.AddPropertyIfAbsent(property);
        }
    }
}

// Register
builder.Host.UseSerilog((context, services, configuration) => configuration
    .Enrich.With(new CorrelationIdEnricher(services.GetRequiredService<IHttpContextAccessor>()))
);
```

### User Context Enricher

```csharp
public class UserContextEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public UserContextEnricher(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext?.User?.Identity?.IsAuthenticated == true)
        {
            var userId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            var userEmail = httpContext.User.FindFirst(ClaimTypes.Email)?.Value;

            if (userId is not null)
            {
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("UserId", userId));
            }

            if (userEmail is not null)
            {
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("UserEmail", userEmail));
            }
        }
    }
}
```

## Request Logging

```csharp
// Enhanced request logging
app.UseSerilogRequestLogging(options =>
{
    // Customize message template
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";

    // Customize log level based on status code
    options.GetLevel = (httpContext, elapsed, ex) =>
    {
        if (ex is not null) return LogEventLevel.Error;
        if (httpContext.Response.StatusCode >= 500) return LogEventLevel.Error;
        if (httpContext.Response.StatusCode >= 400) return LogEventLevel.Warning;
        if (elapsed > 5000) return LogEventLevel.Warning; // Slow requests
        return LogEventLevel.Information;
    };

    // Enrich with additional properties
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());
        diagnosticContext.Set("CorrelationId", httpContext.Items["CorrelationId"]);

        if (httpContext.User.Identity?.IsAuthenticated == true)
        {
            diagnosticContext.Set("UserId", httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value);
        }
    };
});
```

## External Sinks

### Seq (Development)

```bash
dotnet add package Serilog.Sinks.Seq
```

```json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://localhost:5341"
        }
      }
    ]
  }
}
```

### Elasticsearch (Production)

```bash
dotnet add package Serilog.Sinks.Elasticsearch
```

```csharp
builder.Host.UseSerilog((context, configuration) => configuration
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
    {
        IndexFormat = "myapi-logs-{0:yyyy.MM.dd}",
        AutoRegisterTemplate = true,
        NumberOfShards = 2,
        NumberOfReplicas = 1
    })
);
```

## Complete Production Example

```csharp
// Program.cs
using Serilog;
using Serilog.Events;
using Serilog.Exceptions;
using Serilog.Formatting.Compact;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting web application");

    var builder = WebApplication.CreateBuilder(args);

    builder.Services.AddHttpContextAccessor();

    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithProcessId()
        .Enrich.WithThreadId()
        .Enrich.WithExceptionDetails()
        .Enrich.With(new CorrelationIdEnricher(services.GetRequiredService<IHttpContextAccessor>()))
        .Enrich.With(new UserContextEnricher(services.GetRequiredService<IHttpContextAccessor>()))
        .WriteTo.Console(new CompactJsonFormatter())
        .WriteTo.File(
            new CompactJsonFormatter(),
            "logs/log-.json",
            rollingInterval: RollingInterval.Day,
            retainedFileCountLimit: 31
        )
        .WriteTo.Conditional(
            evt => context.HostingEnvironment.IsDevelopment(),
            wt => wt.Seq("http://localhost:5341")
        )
    );

    var app = builder.Build();

    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
        options.GetLevel = (ctx, elapsed, ex) =>
        {
            if (ex is not null) return LogEventLevel.Error;
            if (ctx.Response.StatusCode >= 500) return LogEventLevel.Error;
            if (ctx.Response.StatusCode >= 400) return LogEventLevel.Warning;
            return LogEventLevel.Information;
        };
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("CorrelationId", httpContext.Items["CorrelationId"]);
            diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());
        };
    });

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
```

## Logging Best Practices Checklist

- [ ] Serilog configured as early as possible in Program.cs
- [ ] Bootstrap logger created for startup errors
- [ ] Log.CloseAndFlush() called in finally block
- [ ] Structured logging used (no string interpolation in messages)
- [ ] Correlation IDs included in all log entries
- [ ] Request logging enabled with enrichment
- [ ] Sensitive data (passwords, tokens) never logged
- [ ] Appropriate log levels used consistently
- [ ] Logs written to multiple sinks (console + file/cloud)
- [ ] JSON format used for machine-readable logs
- [ ] Log retention policies configured

## Navigation

- **Previous**: `../06-error-handling/exception-middleware.md` - Exception handling
- **Next**: `opentelemetry.md` - Distributed tracing
- **Related**: `health-checks.md` - Health monitoring

## References

- [Serilog Documentation](https://serilog.net/)
- [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore)
- [Microsoft: Logging in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging)
- [Structured Logging Best Practices](https://stackify.com/what-is-structured-logging-and-why-developers-need-it/)
