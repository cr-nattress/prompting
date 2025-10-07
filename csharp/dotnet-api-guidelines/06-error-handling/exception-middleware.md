# Global Exception Middleware - .NET 8

> **File Purpose**: Implement global exception handler with ProblemDetails mapping and correlation IDs
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Program.cs setup
> **Related Files**: `problem-details.md`, `../07-observability/structured-logging.md`
> **Agent Use Case**: Reference when implementing centralized exception handling with RFC 7807 ProblemDetails

## Quick Context

Global exception handling provides a centralized location for mapping exceptions to appropriate HTTP responses, logging errors, and returning standardized ProblemDetails responses. This prevents exception details from leaking to clients while maintaining observability.

**Microsoft References**:
- [Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807)

## Global Exception Handler (.NET 8+)

### Using IExceptionHandler

```csharp
// MyApi.Api/Middleware/GlobalExceptionHandler.cs
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

namespace MyApi.Api.Middleware;

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var correlationId = httpContext.Items["CorrelationId"]?.ToString()
            ?? httpContext.TraceIdentifier;

        _logger.LogError(
            exception,
            "Unhandled exception occurred. CorrelationId: {CorrelationId}",
            correlationId
        );

        var problemDetails = exception switch
        {
            ValidationException validationEx => new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
                Title = "Validation Error",
                Detail = validationEx.Message,
                Instance = httpContext.Request.Path,
                Extensions =
                {
                    ["errors"] = validationEx.Errors,
                    ["traceId"] = correlationId
                }
            },
            NotFoundException notFoundEx => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4",
                Title = "Resource Not Found",
                Detail = notFoundEx.Message,
                Instance = httpContext.Request.Path,
                Extensions = { ["traceId"] = correlationId }
            },
            UnauthorizedAccessException => new ProblemDetails
            {
                Status = StatusCodes.Status401Unauthorized,
                Type = "https://tools.ietf.org/html/rfc7235#section-3.1",
                Title = "Unauthorized",
                Detail = "Authentication required",
                Instance = httpContext.Request.Path,
                Extensions = { ["traceId"] = correlationId }
            },
            ForbiddenException forbiddenEx => new ProblemDetails
            {
                Status = StatusCodes.Status403Forbidden,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.3",
                Title = "Forbidden",
                Detail = forbiddenEx.Message,
                Instance = httpContext.Request.Path,
                Extensions = { ["traceId"] = correlationId }
            },
            ConflictException conflictEx => new ProblemDetails
            {
                Status = StatusCodes.Status409Conflict,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.8",
                Title = "Conflict",
                Detail = conflictEx.Message,
                Instance = httpContext.Request.Path,
                Extensions = { ["traceId"] = correlationId }
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1",
                Title = "Internal Server Error",
                Detail = "An unexpected error occurred",
                Instance = httpContext.Request.Path,
                Extensions = { ["traceId"] = correlationId }
            }
        };

        httpContext.Response.StatusCode = problemDetails.Status ?? 500;
        httpContext.Response.ContentType = "application/problem+json";

        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}

// Register in Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// Add to middleware pipeline
app.UseExceptionHandler();
```

## Custom Domain Exceptions

```csharp
// MyApi.Domain/Exceptions/DomainException.cs
namespace MyApi.Domain.Exceptions;

public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception innerException)
        : base(message, innerException) { }
}

// MyApi.Domain/Exceptions/NotFoundException.cs
public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object key)
        : base($"{entityName} with key '{key}' was not found") { }
}

// MyApi.Domain/Exceptions/ValidationException.cs
public class ValidationException : DomainException
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred")
    {
        Errors = errors;
    }

    public ValidationException(string propertyName, string errorMessage)
        : base($"Validation failed for {propertyName}")
    {
        Errors = new Dictionary<string, string[]>
        {
            [propertyName] = new[] { errorMessage }
        };
    }
}

// MyApi.Domain/Exceptions/ConflictException.cs
public class ConflictException : DomainException
{
    public ConflictException(string message) : base(message) { }
}

// MyApi.Domain/Exceptions/ForbiddenException.cs
public class ForbiddenException : DomainException
{
    public ForbiddenException(string message) : base(message) { }
}

// Usage in application layer
public async Task<User> GetUserByIdAsync(int id, CancellationToken ct)
{
    var user = await _repository.GetByIdAsync(id, ct);

    if (user is null)
        throw new NotFoundException(nameof(User), id);

    return user;
}
```

## Legacy Exception Handler Middleware

```csharp
// Alternative approach for .NET 6/7 or custom requirements
public class ExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlerMiddleware> _logger;
    private readonly IHostEnvironment _environment;

    public ExceptionHandlerMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlerMiddleware> logger,
        IHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var correlationId = context.Items["CorrelationId"]?.ToString()
            ?? context.TraceIdentifier;

        _logger.LogError(
            exception,
            "Unhandled exception. CorrelationId: {CorrelationId}, Path: {Path}",
            correlationId,
            context.Request.Path
        );

        var (statusCode, title, detail) = exception switch
        {
            ValidationException validationEx => (
                StatusCodes.Status400BadRequest,
                "Validation Error",
                validationEx.Message
            ),
            NotFoundException notFoundEx => (
                StatusCodes.Status404NotFound,
                "Resource Not Found",
                notFoundEx.Message
            ),
            UnauthorizedAccessException => (
                StatusCodes.Status401Unauthorized,
                "Unauthorized",
                "Authentication required"
            ),
            ForbiddenException forbiddenEx => (
                StatusCodes.Status403Forbidden,
                "Forbidden",
                forbiddenEx.Message
            ),
            ConflictException conflictEx => (
                StatusCodes.Status409Conflict,
                "Conflict",
                conflictEx.Message
            ),
            _ => (
                StatusCodes.Status500InternalServerError,
                "Internal Server Error",
                _environment.IsDevelopment()
                    ? exception.Message
                    : "An unexpected error occurred"
            )
        };

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Instance = context.Request.Path,
            Type = GetRfcType(statusCode),
            Extensions =
            {
                ["traceId"] = correlationId
            }
        };

        // Include stack trace in development
        if (_environment.IsDevelopment() && exception is not DomainException)
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
        }

        // Include validation errors
        if (exception is ValidationException validationException)
        {
            problemDetails.Extensions["errors"] = validationException.Errors;
        }

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problemDetails);
    }

    private static string GetRfcType(int statusCode) => statusCode switch
    {
        400 => "https://tools.ietf.org/html/rfc7231#section-6.5.1",
        401 => "https://tools.ietf.org/html/rfc7235#section-3.1",
        403 => "https://tools.ietf.org/html/rfc7231#section-6.5.3",
        404 => "https://tools.ietf.org/html/rfc7231#section-6.5.4",
        409 => "https://tools.ietf.org/html/rfc7231#section-6.5.8",
        _ => "https://tools.ietf.org/html/rfc7231#section-6.6.1"
    };
}

// Register
app.UseMiddleware<ExceptionHandlerMiddleware>();
```

## FluentValidation Integration

```csharp
// Endpoint filter for validation
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<T>>();

        if (validator is null)
            return await next(context);

        var model = context.Arguments.OfType<T>().FirstOrDefault();
        if (model is null)
            return await next(context);

        var validationResult = await validator.ValidateAsync(model);

        if (!validationResult.IsValid)
        {
            throw new ValidationException(
                validationResult.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray()
                    )
            );
        }

        return await next(context);
    }
}

// Apply to endpoint
app.MapPost("/api/users", CreateUser)
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

## Complete Production Example

```csharp
// Program.cs
using MyApi.Api.Middleware;
using MyApi.Domain.Exceptions;

var builder = WebApplication.CreateBuilder(args);

// Register exception handler
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["instance"] = context.HttpContext.Request.Path;
        context.ProblemDetails.Extensions["traceId"] =
            context.HttpContext.Items["CorrelationId"]?.ToString()
            ?? context.HttpContext.TraceIdentifier;
    };
});

var app = builder.Build();

// Exception handling must be first
app.UseExceptionHandler();

// Status code pages for non-exception errors
app.UseStatusCodePages(async context =>
{
    var problemDetails = new ProblemDetails
    {
        Status = context.HttpContext.Response.StatusCode,
        Title = context.HttpContext.Response.StatusCode switch
        {
            404 => "Not Found",
            405 => "Method Not Allowed",
            _ => "Error"
        },
        Instance = context.HttpContext.Request.Path,
        Type = $"https://httpstatuses.com/{context.HttpContext.Response.StatusCode}",
        Extensions =
        {
            ["traceId"] = context.HttpContext.Items["CorrelationId"]?.ToString()
                ?? context.HttpContext.TraceIdentifier
        }
    };

    context.HttpContext.Response.ContentType = "application/problem+json";
    await context.HttpContext.Response.WriteAsJsonAsync(problemDetails);
});

app.MapGet("/api/users/{id:int}", async (int id, IUserService service, CancellationToken ct) =>
{
    // Will throw NotFoundException if not found
    var user = await service.GetUserByIdAsync(id, ct);
    return Results.Ok(user);
});

app.Run();
```

## Error Handling Checklist

- [ ] Global exception handler registered first in pipeline
- [ ] Custom domain exceptions defined for business errors
- [ ] ProblemDetails returned for all error responses
- [ ] Correlation IDs included in error responses
- [ ] Sensitive information not exposed in production
- [ ] Stack traces only included in development
- [ ] All exceptions logged with structured data
- [ ] HTTP status codes match error type
- [ ] RFC 7807 ProblemDetails format used
- [ ] Validation errors mapped to 400 Bad Request

## Navigation

- **Previous**: `../04-api-design/api-standards.md` - API standards
- **Next**: `problem-details.md` - ProblemDetails implementation
- **Related**: `../07-observability/structured-logging.md` - Logging

## References

- [Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
- [Use ProblemDetails](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling#problem-details)
- [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807)
- [IExceptionHandler](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.diagnostics.iexceptionhandler)
