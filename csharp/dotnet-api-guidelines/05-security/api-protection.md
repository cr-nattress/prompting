# API Protection in .NET 8

> **File Purpose**: Implement comprehensive API protection including rate limiting, security headers, CORS, and input validation
> **Prerequisites**: `authentication-jwt.md`, `authorization.md`
> **Related Files**: `owasp-checklist.md`, `../06-error-handling/problem-details.md`
> **Agent Use Case**: Reference when implementing API hardening and protection mechanisms

## Quick Context

API protection encompasses multiple defense layers beyond authentication and authorization. This includes rate limiting, security headers, HTTPS enforcement, CORS policies, and input validation. .NET 8 introduces built-in rate limiting middleware, making it easier to protect APIs from abuse.

**OWASP References**:
- [OWASP API Security Top 10 - API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)
- [OWASP API Security Top 10 - API8:2023 Security Misconfiguration](https://owasp.org/API-Security/editions/2023/en/0xa8-security-misconfiguration/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)

**Microsoft References**:
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [HTTPS enforcement](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl)

## Rate Limiting (.NET 8 Built-in)

### Install Package

```bash
# Built into .NET 8, no package needed
# For .NET 7, install:
# dotnet add package Microsoft.AspNetCore.RateLimiting
```

### Basic Rate Limiting Configuration

```csharp
// Program.cs
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// Add rate limiting services
builder.Services.AddRateLimiter(options =>
{
    // Fixed Window Limiter
    options.AddFixedWindowLimiter("fixed", options =>
    {
        options.PermitLimit = 100; // 100 requests
        options.Window = TimeSpan.FromMinutes(1); // per minute
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 5; // Queue up to 5 requests when limit is hit
    });

    // Sliding Window Limiter (more precise)
    options.AddSlidingWindowLimiter("sliding", options =>
    {
        options.PermitLimit = 100;
        options.Window = TimeSpan.FromMinutes(1);
        options.SegmentsPerWindow = 6; // 6 segments of 10 seconds each
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 5;
    });

    // Token Bucket (bursty traffic)
    options.AddTokenBucketLimiter("token", options =>
    {
        options.TokenLimit = 100; // Bucket capacity
        options.ReplenishmentPeriod = TimeSpan.FromMinutes(1);
        options.TokensPerPeriod = 100; // Tokens added per period
        options.AutoReplenishment = true;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 5;
    });

    // Concurrency Limiter (limit concurrent requests)
    options.AddConcurrencyLimiter("concurrency", options =>
    {
        options.PermitLimit = 10; // Max 10 concurrent requests
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 20;
    });

    // Global rate limiter fallback
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        // Get user ID or IP address
        var userId = context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: userId,
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window = TimeSpan.FromHours(1)
            });
    });

    // Customize rejection response
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString();
        }

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status429TooManyRequests,
            Type = "https://tools.ietf.org/html/rfc6585#section-4",
            Title = "Too Many Requests",
            Detail = "Rate limit exceeded. Please try again later.",
            Instance = context.HttpContext.Request.Path
        };

        problemDetails.Extensions["retryAfter"] = retryAfter?.TotalSeconds ?? 60;

        await context.HttpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);
    };
});

var app = builder.Build();

// IMPORTANT: Add rate limiting middleware
app.UseRateLimiter();

app.Run();
```

### Apply Rate Limiting to Endpoints

```csharp
// Apply specific rate limiter to endpoint
app.MapGet("/api/products", GetProducts)
    .RequireRateLimiting("fixed");

// Apply to endpoint group
var api = app.MapGroup("/api")
    .RequireRateLimiting("sliding");

api.MapGet("/products", GetProducts);
api.MapPost("/products", CreateProduct);

// Disable rate limiting for specific endpoint
app.MapGet("/health", () => "Healthy")
    .DisableRateLimiting();
```

### Per-User Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("perUser", context =>
    {
        // Get authenticated user ID
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";

        return RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: userId,
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            });
    });

    // Different limits for different roles
    options.AddPolicy("roleBasedLimit", context =>
    {
        var isAdmin = context.User.IsInRole("Admin");
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: userId,
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = isAdmin ? 10000 : 100, // Admins get higher limit
                Window = TimeSpan.FromMinutes(1)
            });
    });
});

// Usage
app.MapGet("/api/data", GetData)
    .RequireAuthorization()
    .RequireRateLimiting("perUser");
```

### Per-IP Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("perIp", context =>
    {
        var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        return RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: ipAddress,
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 1000,
                ReplenishmentPeriod = TimeSpan.FromHours(1),
                TokensPerPeriod = 1000,
                AutoReplenishment = true
            });
    });
});
```

### Rate Limiting Decorator Pattern

```csharp
// Demo.Api/RateLimiting/RateLimitPolicies.cs
namespace Demo.Api.RateLimiting;

public static class RateLimitPolicies
{
    public const string Public = "public";
    public const string Authenticated = "authenticated";
    public const string Admin = "admin";
    public const string Api = "api";
    public const string Upload = "upload";

    public static IServiceCollection AddApiRateLimiting(this IServiceCollection services)
    {
        services.AddRateLimiter(options =>
        {
            // Public endpoints (strict limits)
            options.AddFixedWindowLimiter(Public, opt =>
            {
                opt.PermitLimit = 20;
                opt.Window = TimeSpan.FromMinutes(1);
            });

            // Authenticated users (generous limits)
            options.AddSlidingWindowLimiter(Authenticated, opt =>
            {
                opt.PermitLimit = 200;
                opt.Window = TimeSpan.FromMinutes(1);
                opt.SegmentsPerWindow = 6;
            });

            // Admin users (very high limits)
            options.AddFixedWindowLimiter(Admin, opt =>
            {
                opt.PermitLimit = 10000;
                opt.Window = TimeSpan.FromMinutes(1);
            });

            // API endpoints (per-user)
            options.AddPolicy(Api, context =>
            {
                var userId = context.User.FindFirst("sub")?.Value ??
                            context.Connection.RemoteIpAddress?.ToString() ??
                            "anonymous";

                var isAdmin = context.User.IsInRole("Admin");

                return RateLimitPartition.GetSlidingWindowLimiter(
                    userId,
                    _ => new SlidingWindowRateLimiterOptions
                    {
                        PermitLimit = isAdmin ? 10000 : 500,
                        Window = TimeSpan.FromMinutes(1),
                        SegmentsPerWindow = 6
                    });
            });

            // Upload endpoints (concurrency limit)
            options.AddConcurrencyLimiter(Upload, opt =>
            {
                opt.PermitLimit = 5; // Max 5 concurrent uploads
                opt.QueueLimit = 10;
            });

            options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
        });

        return services;
    }
}
```

## Request Size Limits and Throttling

### Limit Request Body Size

```csharp
// Program.cs
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024; // 10 MB
});

builder.Services.Configure<KestrelServerOptions>(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
});

// Per-endpoint limit
app.MapPost("/upload", async (IFormFile file) =>
{
    // Process file
    return Results.Ok();
})
.DisableRequestSizeLimit() // Disable global limit
.WithRequestSizeLimit(50 * 1024 * 1024); // Set to 50 MB
```

### Request Timeout

```csharp
// Program.cs
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
});

// Per-request timeout using middleware
app.Use(async (context, next) =>
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        cts.Token,
        context.RequestAborted
    );

    context.RequestAborted = linkedCts.Token;

    try
    {
        await next(context);
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        context.Response.StatusCode = StatusCodes.Status408RequestTimeout;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = StatusCodes.Status408RequestTimeout,
            Title = "Request Timeout",
            Detail = "The request took too long to process"
        });
    }
});
```

## HTTPS Enforcement and HSTS

### Force HTTPS Redirection

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Configure HTTPS redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443; // Production port
});

var app = builder.Build();

// Enforce HTTPS (OWASP recommendation)
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();

    // HSTS (HTTP Strict Transport Security)
    app.UseHsts();
}

app.Run();
```

### HSTS Configuration

```csharp
// Program.cs
builder.Services.AddHsts(options =>
{
    options.Preload = true; // Include in browser preload list
    options.IncludeSubDomains = true; // Apply to all subdomains
    options.MaxAge = TimeSpan.FromDays(365); // Cache for 1 year
    options.ExcludedHosts.Add("localhost"); // Exclude development
});
```

### Development HTTPS Certificate

```bash
# Trust development certificate
dotnet dev-certs https --trust

# Configure in appsettings.Development.json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://localhost:7001"
      }
    }
  }
}
```

## Security Headers

### Comprehensive Security Headers Middleware

```csharp
// Demo.Api/Middleware/SecurityHeadersMiddleware.cs
using Microsoft.Extensions.Primitives;

namespace Demo.Api.Middleware;

public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // OWASP Recommended Security Headers

        // Content Security Policy (CSP)
        context.Response.Headers.Append(
            "Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "font-src 'self'; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none'; " +
            "base-uri 'self'; " +
            "form-action 'self'"
        );

        // X-Content-Type-Options (prevent MIME sniffing)
        context.Response.Headers.Append("X-Content-Type-Options", "nosniff");

        // X-Frame-Options (prevent clickjacking)
        context.Response.Headers.Append("X-Frame-Options", "DENY");

        // X-XSS-Protection (legacy, but still useful)
        context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");

        // Referrer-Policy (control referrer information)
        context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

        // Permissions-Policy (formerly Feature-Policy)
        context.Response.Headers.Append(
            "Permissions-Policy",
            "geolocation=(), microphone=(), camera=()"
        );

        // Remove server header (security through obscurity)
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");

        // Cache-Control for sensitive endpoints
        if (context.Request.Path.StartsWithSegments("/api"))
        {
            context.Response.Headers.Append("Cache-Control", "no-store, no-cache, must-revalidate");
            context.Response.Headers.Append("Pragma", "no-cache");
        }

        await _next(context);
    }
}

public static class SecurityHeadersMiddlewareExtensions
{
    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<SecurityHeadersMiddleware>();
    }
}
```

### Register Security Headers

```csharp
// Program.cs
using Demo.Api.Middleware;

var app = builder.Build();

// Add security headers (OWASP best practice)
app.UseSecurityHeaders();

app.Run();
```

### NWebsec Alternative (NuGet Package)

```bash
dotnet add package NWebsec.AspNetCore.Middleware
```

```csharp
// Program.cs
using NWebsec.AspNetCore.Middleware;

var app = builder.Build();

// X-Content-Type-Options
app.UseXContentTypeOptions();

// X-Frame-Options
app.UseXfo(options => options.Deny());

// X-XSS-Protection
app.UseXXssProtection(options => options.EnabledWithBlockMode());

// Referrer-Policy
app.UseReferrerPolicy(options => options.StrictOriginWhenCrossOrigin());

// CSP
app.UseCsp(options => options
    .DefaultSources(s => s.Self())
    .ScriptSources(s => s.Self().UnsafeInline())
    .StyleSources(s => s.Self().UnsafeInline())
    .ImageSources(s => s.Self().CustomSources("data:", "https:"))
    .FrameAncestors(s => s.None())
);

app.Run();
```

## CORS Configuration

### Basic CORS Setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    // Development policy (permissive)
    options.AddPolicy("Development", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });

    // Production policy (restrictive)
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins(
                "https://example.com",
                "https://app.example.com"
            )
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Content-Type", "Authorization")
            .WithExposedHeaders("X-Pagination", "X-Total-Count")
            .SetIsOriginAllowedToAllowWildcardSubdomains()
            .AllowCredentials() // Allow cookies/auth headers
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10)); // Cache preflight
    });

    // Specific API policy
    options.AddPolicy("ApiPolicy", policy =>
    {
        policy.WithOrigins(builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>()!)
              .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
              .WithHeaders("Content-Type", "Authorization", "X-Api-Key")
              .AllowCredentials();
    });
});

var app = builder.Build();

// Apply CORS policy
var corsPolicy = app.Environment.IsDevelopment() ? "Development" : "Production";
app.UseCors(corsPolicy);

app.Run();
```

### CORS Configuration in appsettings.json

```json
{
  "Cors": {
    "AllowedOrigins": [
      "https://example.com",
      "https://app.example.com"
    ]
  }
}
```

### Per-Endpoint CORS

```csharp
// Apply to specific endpoints
app.MapGet("/api/public", () => "Public data")
    .RequireCors("Development");

// Apply to endpoint group
var api = app.MapGroup("/api")
    .RequireCors("ApiPolicy");

api.MapGet("/data", GetData);
```

### Dynamic CORS Policy

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.SetIsOriginAllowed(origin =>
        {
            // Allow localhost in development
            if (builder.Environment.IsDevelopment() &&
                origin.StartsWith("http://localhost"))
            {
                return true;
            }

            // Check against whitelist
            var allowedOrigins = builder.Configuration
                .GetSection("Cors:AllowedOrigins")
                .Get<string[]>() ?? Array.Empty<string>();

            return allowedOrigins.Contains(origin);
        })
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials();
    });
});
```

## Input Validation and Sanitization

### FluentValidation Integration

```csharp
// Demo.Api/Validators/CreateProductRequestValidator.cs
using FluentValidation;

namespace Demo.Api.Validators;

public class CreateProductRequest
{
    public required string Name { get; init; }
    public required string Description { get; init; }
    public decimal Price { get; init; }
    public string? Category { get; init; }
}

public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        // Required fields
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters")
            .Must(BeValidProductName).WithMessage("Name contains invalid characters");

        RuleFor(x => x.Description)
            .NotEmpty().WithMessage("Description is required")
            .MaximumLength(2000).WithMessage("Description must not exceed 2000 characters");

        // Numeric validation
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero")
            .LessThanOrEqualTo(1000000).WithMessage("Price must not exceed 1,000,000");

        // Optional field validation
        RuleFor(x => x.Category)
            .MaximumLength(100).WithMessage("Category must not exceed 100 characters")
            .When(x => !string.IsNullOrEmpty(x.Category));
    }

    private bool BeValidProductName(string name)
    {
        // Prevent XSS and SQL injection patterns
        var invalidChars = new[] { '<', '>', '&', '"', '\'', '/', '\\', ';' };
        return !name.Any(c => invalidChars.Contains(c));
    }
}
```

### Validation Endpoint Filter

```csharp
// Demo.Api/Filters/ValidationFilter.cs
using FluentValidation;

namespace Demo.Api.Filters;

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
        var requestObject = context.Arguments.OfType<T>().FirstOrDefault();
        if (requestObject is null)
        {
            return Results.BadRequest("Request object is null");
        }

        var validationResult = await _validator.ValidateAsync(requestObject);
        if (!validationResult.IsValid)
        {
            var errors = validationResult.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray()
                );

            return Results.ValidationProblem(
                errors,
                title: "One or more validation errors occurred"
            );
        }

        return await next(context);
    }
}
```

### Register and Use Validation

```csharp
// Program.cs
using FluentValidation;
using Demo.Api.Filters;
using Demo.Api.Validators;

builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Apply validation to endpoint
app.MapPost("/api/products", CreateProduct)
    .AddEndpointFilter<ValidationFilter<CreateProductRequest>>()
    .ProducesValidationProblem();
```

### Sanitize HTML Input

```bash
dotnet add package HtmlSanitizer
```

```csharp
// Demo.Api/Services/InputSanitizer.cs
using Ganss.Xss;

namespace Demo.Api.Services;

public interface IInputSanitizer
{
    string SanitizeHtml(string input);
    string SanitizeString(string input);
}

public class InputSanitizer : IInputSanitizer
{
    private readonly HtmlSanitizer _htmlSanitizer;

    public InputSanitizer()
    {
        _htmlSanitizer = new HtmlSanitizer();

        // Configure allowed tags
        _htmlSanitizer.AllowedTags.Clear();
        _htmlSanitizer.AllowedTags.Add("p");
        _htmlSanitizer.AllowedTags.Add("br");
        _htmlSanitizer.AllowedTags.Add("strong");
        _htmlSanitizer.AllowedTags.Add("em");
        _htmlSanitizer.AllowedTags.Add("ul");
        _htmlSanitizer.AllowedTags.Add("li");
    }

    public string SanitizeHtml(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
        {
            return string.Empty;
        }

        return _htmlSanitizer.Sanitize(input);
    }

    public string SanitizeString(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
        {
            return string.Empty;
        }

        // Remove control characters
        return new string(input.Where(c => !char.IsControl(c) || c == '\n' || c == '\r').ToArray());
    }
}
```

### SQL Injection Prevention

```csharp
// NEVER concatenate user input into SQL
// DON'T
var sql = $"SELECT * FROM Products WHERE Name = '{productName}'"; // VULNERABLE!

// DO - Use parameterized queries
var products = await _context.Products
    .Where(p => p.Name == productName) // EF Core parameterizes this
    .ToListAsync();

// DO - Use parameters with Dapper
var products = await connection.QueryAsync<Product>(
    "SELECT * FROM Products WHERE Name = @Name",
    new { Name = productName }
);
```

## Complete Security Configuration Example

```csharp
// Program.cs
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;
using Demo.Api.Middleware;
using Demo.Api.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// Authentication & Authorization
builder.Services.AddAuthentication(/* JWT config */);
builder.Services.AddAuthorization();

// Rate Limiting
builder.Services.AddApiRateLimiting();

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>()!)
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// HSTS
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

// Request limits
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024;
});

builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024;
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);
});

// Validation
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Input sanitization
builder.Services.AddSingleton<IInputSanitizer, InputSanitizer>();

var app = builder.Build();

// SECURITY MIDDLEWARE ORDER (CRITICAL!)
if (!app.Environment.IsDevelopment())
{
    app.UseHsts(); // 1. HSTS
}

app.UseHttpsRedirection(); // 2. Force HTTPS
app.UseSecurityHeaders(); // 3. Security headers

app.UseCors(); // 4. CORS

app.UseRateLimiter(); // 5. Rate limiting

app.UseAuthentication(); // 6. Authentication
app.UseAuthorization(); // 7. Authorization

// Exception handling
app.UseExceptionHandler();

app.MapEndpoints();

app.Run();
```

## Security Checklist

- [ ] Enable HTTPS redirection in production
- [ ] Configure HSTS with `Preload` and `IncludeSubDomains`
- [ ] Implement rate limiting on all public endpoints
- [ ] Set request size limits (10-50 MB recommended)
- [ ] Configure request timeouts
- [ ] Add security headers (CSP, X-Frame-Options, etc.)
- [ ] Configure CORS with explicit allowed origins
- [ ] Never use `AllowAnyOrigin()` with `AllowCredentials()`
- [ ] Validate all user input with FluentValidation
- [ ] Sanitize HTML content
- [ ] Use parameterized queries (prevent SQL injection)
- [ ] Remove `Server` and `X-Powered-By` headers
- [ ] Set `Cache-Control: no-store` on sensitive endpoints
- [ ] Implement different rate limits for authenticated vs anonymous users
- [ ] Log security events (failed auth, rate limit hits)

## Common Mistakes

### Mistake: Allowing any origin with credentials

```csharp
// DON'T - Security vulnerability!
policy.AllowAnyOrigin()
      .AllowCredentials(); // Can't use both!
```

**Solution**: Specify allowed origins

```csharp
// DO
policy.WithOrigins("https://example.com")
      .AllowCredentials();
```

### Mistake: No rate limiting on authentication endpoints

```csharp
// DON'T - Vulnerable to brute force
app.MapPost("/auth/login", Login); // No rate limit!
```

**Solution**: Add strict rate limiting

```csharp
// DO
app.MapPost("/auth/login", Login)
    .RequireRateLimiting("strictAuth"); // 5 attempts per minute
```

### Mistake: Not validating file uploads

```csharp
// DON'T
app.MapPost("/upload", async (IFormFile file) =>
{
    await file.CopyToAsync(stream); // No validation!
});
```

**Solution**: Validate file type, size, and content

```csharp
// DO
app.MapPost("/upload", async (IFormFile file) =>
{
    var allowedExtensions = new[] { ".jpg", ".png", ".pdf" };
    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();

    if (!allowedExtensions.Contains(extension))
    {
        return Results.BadRequest("Invalid file type");
    }

    if (file.Length > 10 * 1024 * 1024)
    {
        return Results.BadRequest("File too large");
    }

    // Additional content validation...
});
```

## Navigation

- **Previous**: `authorization.md`
- **Next**: `owasp-checklist.md`
- **Related**: `authentication-jwt.md`, `../06-error-handling/problem-details.md`

## References

- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Microsoft: Rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Microsoft: CORS](https://learn.microsoft.com/en-us/aspnet/core/security/cors)
- [Microsoft: HTTPS enforcement](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl)
- [Content Security Policy Reference](https://content-security-policy.com/)
