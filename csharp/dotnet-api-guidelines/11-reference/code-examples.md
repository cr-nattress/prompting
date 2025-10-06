# Complete Code Examples

> **File Purpose**: Full, runnable code examples for production-ready .NET 8 Web API
> **Prerequisites**: All core concept files
> **Agent Use Case**: Copy-paste starting point for implementing complete features

## 🎯 Quick Context

This file contains complete, production-ready code examples that combine multiple best practices: Clean Architecture, JWT authentication, validation, error handling, logging, caching, and health checks.

## Complete Program.cs

```csharp
using System.Diagnostics;
using Demo.Api.Configuration;
using Demo.Api.Filters;
using Demo.Application.Services;
using Demo.Infrastructure.Data;
using FluentValidation;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// ============================================================================
// LOGGING
// ============================================================================
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "Demo.Api")
    .WriteTo.Console()
    .CreateLogger();

builder.Host.UseSerilog();

// ============================================================================
// CONFIGURATION
// ============================================================================
var authOptions = builder.Configuration
    .GetSection(AuthenticationOptions.SectionName)
    .Get<AuthenticationOptions>()
    ?? throw new InvalidOperationException("Authentication configuration missing");

// ============================================================================
// DATABASE
// ============================================================================
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null
            );
        }
    )
);

// ============================================================================
// AUTHENTICATION & AUTHORIZATION
// ============================================================================
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = authOptions.Authority;
        options.Audience = authOptions.Audience;
        options.RequireHttpsMetadata = authOptions.RequireHttpsMetadata;

        options.TokenValidationParameters = new()
        {
            ValidateIssuer = true,
            ValidIssuer = authOptions.ValidIssuer,
            ValidateAudience = true,
            ValidAudience = authOptions.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(2),
            ValidateIssuerSigningKey = true
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("TodoWrite", policy =>
        policy.RequireClaim("permissions", "write:todos"));
});

// ============================================================================
// RATE LIMITING
// ============================================================================
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 0;
    });

    options.AddTokenBucketLimiter("token", opt =>
    {
        opt.TokenLimit = 1000;
        opt.ReplenishmentPeriod = TimeSpan.FromMinutes(1);
        opt.TokensPerPeriod = 100;
        opt.QueueLimit = 0;
    });
});

// ============================================================================
// OUTPUT CACHING
// ============================================================================
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder =>
        builder.Expire(TimeSpan.FromSeconds(10)));

    options.AddPolicy("todos-list", builder =>
        builder.Expire(TimeSpan.FromMinutes(5))
               .Tag("todos"));

    options.AddPolicy("todo-detail", builder =>
        builder.Expire(TimeSpan.FromMinutes(10))
               .Tag("todos")
               .SetVaryByQuery("id"));
});

// ============================================================================
// HEALTH CHECKS
// ============================================================================
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(
        name: "database",
        tags: new[] { "db", "ready" }
    )
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" });

// ============================================================================
// API DOCUMENTATION
// ============================================================================
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new()
    {
        Title = "Demo API",
        Version = "v1",
        Description = "Production-ready .NET 8 Web API"
    });

    // JWT authentication in Swagger
    options.AddSecurityDefinition("Bearer", new()
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "Enter JWT token"
    });

    options.AddSecurityRequirement(new()
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

// ============================================================================
// PROBLEM DETAILS
// ============================================================================
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? context.HttpContext.TraceIdentifier;

        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        if (builder.Environment.IsDevelopment())
        {
            context.ProblemDetails.Extensions["machine"] = Environment.MachineName;
        }
    };
});

// ============================================================================
// APPLICATION SERVICES
// ============================================================================
builder.Services.AddScoped<ITodoService, TodoService>();
builder.Services.AddValidatorsFromAssemblyContaining<CreateTodoValidator>();

// ============================================================================
// HTTP CLIENT WITH RESILIENCE
// ============================================================================
builder.Services.AddHttpClient("external-api", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddStandardResilienceHandler();

var app = builder.Build();

// ============================================================================
// MIDDLEWARE PIPELINE
// ============================================================================
app.UseSerilogRequestLogging();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        var problemDetails = new ProblemDetails
        {
            Title = "An error occurred",
            Status = StatusCodes.Status500InternalServerError,
            Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1",
            Detail = app.Environment.IsDevelopment()
                ? exception?.Message
                : "An internal server error occurred",
            Instance = context.Request.Path
        };

        problemDetails.Extensions["traceId"] = Activity.Current?.Id ?? context.TraceIdentifier;

        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(exception, "Unhandled exception. TraceId: {TraceId}", problemDetails.Extensions["traceId"]);

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});

app.UseStatusCodePages();
app.UseHttpsRedirection();

app.UseRateLimiter();
app.UseOutputCache();

app.UseAuthentication();
app.UseAuthorization();

// ============================================================================
// HEALTH CHECK ENDPOINTS
// ============================================================================
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
})
.AllowAnonymous()
.WithName("LivenessProbe");

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
})
.AllowAnonymous()
.WithName("ReadinessProbe");

// ============================================================================
// API ENDPOINTS
// ============================================================================
var todos = app.MapGroup("/api/v1/todos")
    .WithTags("Todos")
    .RequireRateLimiting("fixed");

// GET /api/v1/todos
todos.MapGet("/", async (
    ITodoService svc,
    CancellationToken ct) =>
{
    var items = await svc.GetAllAsync(ct);
    return Results.Ok(items);
})
.WithName("GetTodos")
.CacheOutput("todos-list")
.Produces<IEnumerable<TodoResponse>>(StatusCodes.Status200OK)
.RequireAuthorization();

// GET /api/v1/todos/{id}
todos.MapGet("/{id:guid}", async (
    Guid id,
    ITodoService svc,
    CancellationToken ct) =>
{
    var todo = await svc.GetByIdAsync(id, ct);
    return todo is not null
        ? Results.Ok(todo)
        : Results.Problem(
            title: "Not Found",
            statusCode: StatusCodes.Status404NotFound,
            detail: $"Todo {id} not found"
        );
})
.WithName("GetTodoById")
.CacheOutput("todo-detail")
.Produces<TodoResponse>(StatusCodes.Status200OK)
.ProducesProblem(StatusCodes.Status404NotFound)
.RequireAuthorization();

// POST /api/v1/todos
todos.MapPost("/", async (
    CreateTodoRequest request,
    ITodoService svc,
    IOutputCacheStore cache,
    CancellationToken ct) =>
{
    var id = await svc.CreateAsync(request, ct);

    // Invalidate cache
    await cache.EvictByTagAsync("todos", ct);

    return Results.CreatedAtRoute("GetTodoById", new { id }, new { id });
})
.AddEndpointFilter<ValidationFilter<CreateTodoRequest>>()
.WithName("CreateTodo")
.Produces<object>(StatusCodes.Status201Created)
.ProducesValidationProblem()
.RequireAuthorization("TodoWrite");

// PUT /api/v1/todos/{id}
todos.MapPut("/{id:guid}", async (
    Guid id,
    UpdateTodoRequest request,
    ITodoService svc,
    IOutputCacheStore cache,
    CancellationToken ct) =>
{
    var updated = await svc.UpdateAsync(id, request, ct);

    if (updated)
    {
        await cache.EvictByTagAsync("todos", ct);
        return Results.NoContent();
    }

    return Results.Problem(
        title: "Not Found",
        statusCode: StatusCodes.Status404NotFound,
        detail: $"Todo {id} not found"
    );
})
.AddEndpointFilter<ValidationFilter<UpdateTodoRequest>>()
.WithName("UpdateTodo")
.Produces(StatusCodes.Status204NoContent)
.ProducesProblem(StatusCodes.Status404NotFound)
.ProducesValidationProblem()
.RequireAuthorization("TodoWrite");

// DELETE /api/v1/todos/{id}
todos.MapDelete("/{id:guid}", async (
    Guid id,
    ITodoService svc,
    IOutputCacheStore cache,
    CancellationToken ct) =>
{
    var deleted = await svc.DeleteAsync(id, ct);

    if (deleted)
    {
        await cache.EvictByTagAsync("todos", ct);
        return Results.NoContent();
    }

    return Results.Problem(
        title: "Not Found",
        statusCode: StatusCodes.Status404NotFound,
        detail: $"Todo {id} not found"
    );
})
.WithName("DeleteTodo")
.Produces(StatusCodes.Status204NoContent)
.ProducesProblem(StatusCodes.Status404NotFound)
.RequireAuthorization("TodoWrite");

// ============================================================================
// RUN APPLICATION
// ============================================================================
try
{
    Log.Information("Starting Demo.Api");
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

// Required for WebApplicationFactory in integration tests
public partial class Program { }
```

## Complete appsettings.json

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console"],
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
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=DemoDb;Trusted_Connection=true;TrustServerCertificate=true;"
  },
  "Authentication": {
    "Authority": "https://your-tenant.auth0.com/",
    "Audience": "https://api.example.com",
    "ValidIssuer": "https://your-tenant.auth0.com/",
    "RequireHttpsMetadata": true
  },
  "AllowedHosts": "*"
}
```

## Complete DbContext

```csharp
// Demo.Infrastructure/Data/AppDbContext.cs
using Demo.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace Demo.Infrastructure.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Todo> Todos => Set<Todo>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Todo>(entity =>
        {
            entity.ToTable("Todos");

            entity.HasKey(e => e.Id);

            entity.Property(e => e.Title)
                .IsRequired()
                .HasMaxLength(200);

            entity.Property(e => e.Description)
                .HasMaxLength(1000);

            entity.Property(e => e.CreatedAt)
                .IsRequired()
                .HasDefaultValueSql("GETUTCDATE()");

            entity.Property(e => e.UpdatedAt)
                .IsRequired()
                .HasDefaultValueSql("GETUTCDATE()");

            entity.HasIndex(e => e.IsCompleted);
            entity.HasIndex(e => e.DueDate);
        });
    }
}
```

## Complete Entity

```csharp
// Demo.Domain/Entities/Todo.cs
namespace Demo.Domain.Entities;

public class Todo
{
    public Guid Id { get; set; }
    public required string Title { get; set; }
    public string? Description { get; set; }
    public DateTime? DueDate { get; set; }
    public bool IsCompleted { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

## Complete Service Implementation

```csharp
// Demo.Application/Services/TodoService.cs
using Demo.Application.DTOs;
using Demo.Domain.Entities;
using Demo.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

namespace Demo.Application.Services;

public class TodoService : ITodoService
{
    private readonly AppDbContext _dbContext;
    private readonly ILogger<TodoService> _logger;

    public TodoService(AppDbContext dbContext, ILogger<TodoService> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<IEnumerable<TodoResponse>> GetAllAsync(CancellationToken ct)
    {
        _logger.LogInformation("Fetching all todos");

        var todos = await _dbContext.Todos
            .AsNoTracking()
            .OrderByDescending(t => t.CreatedAt)
            .ToListAsync(ct);

        return todos.Select(t => new TodoResponse(
            t.Id,
            t.Title,
            t.Description,
            t.DueDate,
            t.IsCompleted
        ));
    }

    public async Task<TodoResponse?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        _logger.LogInformation("Fetching todo {TodoId}", id);

        var todo = await _dbContext.Todos
            .AsNoTracking()
            .FirstOrDefaultAsync(t => t.Id == id, ct);

        return todo is not null
            ? new TodoResponse(todo.Id, todo.Title, todo.Description, todo.DueDate, todo.IsCompleted)
            : null;
    }

    public async Task<Guid> CreateAsync(CreateTodoRequest request, CancellationToken ct)
    {
        var todo = new Todo
        {
            Id = Guid.NewGuid(),
            Title = request.Title,
            Description = request.Description,
            DueDate = request.DueDate,
            IsCompleted = false,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        _dbContext.Todos.Add(todo);
        await _dbContext.SaveChangesAsync(ct);

        _logger.LogInformation("Created todo {TodoId}", todo.Id);

        return todo.Id;
    }

    public async Task<bool> UpdateAsync(Guid id, UpdateTodoRequest request, CancellationToken ct)
    {
        var todo = await _dbContext.Todos.FindAsync(new object[] { id }, ct);

        if (todo is null)
        {
            _logger.LogWarning("Todo {TodoId} not found for update", id);
            return false;
        }

        todo.Title = request.Title;
        todo.Description = request.Description;
        todo.DueDate = request.DueDate;
        todo.IsCompleted = request.IsCompleted;
        todo.UpdatedAt = DateTime.UtcNow;

        await _dbContext.SaveChangesAsync(ct);

        _logger.LogInformation("Updated todo {TodoId}", id);

        return true;
    }

    public async Task<bool> DeleteAsync(Guid id, CancellationToken ct)
    {
        var todo = await _dbContext.Todos.FindAsync(new object[] { id }, ct);

        if (todo is null)
        {
            _logger.LogWarning("Todo {TodoId} not found for deletion", id);
            return false;
        }

        _dbContext.Todos.Remove(todo);
        await _dbContext.SaveChangesAsync(ct);

        _logger.LogInformation("Deleted todo {TodoId}", id);

        return true;
    }
}
```

---

## Navigation
- **Up**: `../00-overview.md`

## See Also
- All implementation files in directories 01-11
