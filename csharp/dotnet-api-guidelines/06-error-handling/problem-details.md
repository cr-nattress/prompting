# RFC 7807 Problem Details

> **File Purpose**: Implement standardized error responses using RFC 7807 ProblemDetails
> **Prerequisites**: `error-handling.md` - Global exception middleware
> **Related Files**: `validation.md`, `../06-security/api-security.md`
> **Agent Use Case**: Reference when implementing consistent error responses across API

## 🎯 Quick Context

RFC 7807 (Problem Details for HTTP APIs) defines a standard format for machine-readable error responses. ASP.NET Core 8 includes built-in support via `ProblemDetails` and `ValidationProblemDetails`, ensuring consistent error handling across your API.

## Problem Details Structure

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "title": ["Title is required", "Title must not exceed 200 characters"]
  },
  "traceId": "00-abc123-xyz789-00"
}
```

**Standard Fields**:
- `type`: URI reference identifying the problem type
- `title`: Human-readable summary
- `status`: HTTP status code
- `detail`: Specific explanation for this occurrence
- `instance`: URI reference to specific occurrence

## Enable Problem Details in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Problem Details support
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        // Add traceId for correlation with logs
        context.ProblemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? context.HttpContext.TraceIdentifier;

        // Add machine name in development
        if (builder.Environment.IsDevelopment())
        {
            context.ProblemDetails.Extensions["machine"] = Environment.MachineName;
        }

        // Add timestamp
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});

var app = builder.Build();

// Use exception handler middleware (returns ProblemDetails)
app.UseExceptionHandler();

// Handle 404s with ProblemDetails
app.UseStatusCodePages();
```

## Validation Problem Details

### Automatic from FluentValidation

```csharp
// Validation filter automatically returns ValidationProblemDetails
app.MapPost("/todos", async (CreateTodoRequest req, ITodoService svc, CancellationToken ct) =>
{
    var id = await svc.CreateAsync(req, ct);
    return Results.Created($"/todos/{id}", new { id });
})
.AddEndpointFilter<ValidationFilter<CreateTodoRequest>>()
.ProducesValidationProblem(); // Documents 400 response in OpenAPI
```

**Response** (when validation fails):
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "title": ["Title is required"],
    "dueDate": ["Due date must be in the future"]
  },
  "traceId": "00-abc123-xyz789-00",
  "timestamp": "2025-10-06T10:30:00Z"
}
```

### Manual Validation Problem Details

```csharp
app.MapPost("/manual-validation", (CreateTodoRequest req) =>
{
    var errors = new Dictionary<string, string[]>();

    if (string.IsNullOrWhiteSpace(req.Title))
    {
        errors["title"] = new[] { "Title is required" };
    }

    if (errors.Any())
    {
        return Results.ValidationProblem(
            errors,
            title: "Validation Failed",
            statusCode: StatusCodes.Status400BadRequest
        );
    }

    return Results.Ok();
});
```

## Global Exception Handler

### Exception Handler Middleware (Recommended)

```csharp
// Program.cs
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
                : "An internal server error occurred"
        };

        // Add traceId for correlation
        problemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? context.TraceIdentifier;

        // Map known exceptions to specific status codes
        problemDetails = exception switch
        {
            NotFoundException notFound => new ProblemDetails
            {
                Title = "Resource not found",
                Status = StatusCodes.Status404NotFound,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4",
                Detail = notFound.Message,
                Extensions = { ["traceId"] = context.TraceIdentifier }
            },
            UnauthorizedAccessException => new ProblemDetails
            {
                Title = "Unauthorized",
                Status = StatusCodes.Status403Forbidden,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.3",
                Detail = "You do not have permission to access this resource",
                Extensions = { ["traceId"] = context.TraceIdentifier }
            },
            InvalidOperationException invalid => new ProblemDetails
            {
                Title = "Invalid operation",
                Status = StatusCodes.Status400BadRequest,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
                Detail = invalid.Message,
                Extensions = { ["traceId"] = context.TraceIdentifier }
            },
            _ => problemDetails
        };

        // Log the exception
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(
            exception,
            "Unhandled exception occurred. TraceId: {TraceId}",
            problemDetails.Extensions["traceId"]
        );

        context.Response.StatusCode = problemDetails.Status ?? 500;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});
```

## Custom Domain Exceptions

### Define Exception Types

```csharp
// Demo.Domain/Exceptions/NotFoundException.cs
namespace Demo.Domain.Exceptions;

public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }

    public NotFoundException(string name, object key)
        : base($"{name} with key '{key}' was not found") { }
}
```

```csharp
// Demo.Domain/Exceptions/ValidationException.cs
namespace Demo.Domain.Exceptions;

public class ValidationException : Exception
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred")
    {
        Errors = errors;
    }
}
```

### Use in Application Layer

```csharp
// Demo.Application/Services/TodoService.cs
public async Task<TodoResponse> GetByIdAsync(Guid id, CancellationToken ct)
{
    var todo = await _dbContext.Todos.FindAsync(new object[] { id }, ct);

    if (todo is null)
    {
        throw new NotFoundException(nameof(Todo), id);
    }

    return _mapper.Map<TodoResponse>(todo);
}
```

### Exception to ProblemDetails Mapper

```csharp
// Demo.Api/Middleware/ProblemDetailsMapper.cs
public static class ProblemDetailsMapper
{
    public static ProblemDetails MapException(Exception exception, HttpContext context, bool isDevelopment)
    {
        var (title, status, type, detail) = exception switch
        {
            NotFoundException notFound => (
                "Resource not found",
                StatusCodes.Status404NotFound,
                "https://tools.ietf.org/html/rfc7231#section-6.5.4",
                notFound.Message
            ),
            ValidationException validation => (
                "Validation failed",
                StatusCodes.Status400BadRequest,
                "https://tools.ietf.org/html/rfc7231#section-6.5.1",
                validation.Message
            ),
            UnauthorizedAccessException => (
                "Forbidden",
                StatusCodes.Status403Forbidden,
                "https://tools.ietf.org/html/rfc7231#section-6.5.3",
                "You do not have permission to access this resource"
            ),
            _ => (
                "An error occurred",
                StatusCodes.Status500InternalServerError,
                "https://tools.ietf.org/html/rfc7231#section-6.6.1",
                isDevelopment ? exception.Message : "An internal error occurred"
            )
        };

        var problemDetails = new ProblemDetails
        {
            Title = title,
            Status = status,
            Type = type,
            Detail = detail,
            Instance = context.Request.Path
        };

        problemDetails.Extensions["traceId"] = Activity.Current?.Id ?? context.TraceIdentifier;

        if (isDevelopment)
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
        }

        // Add validation errors for ValidationException
        if (exception is ValidationException validationEx)
        {
            problemDetails.Extensions["errors"] = validationEx.Errors;
        }

        return problemDetails;
    }
}
```

## 404 Not Found Handling

```csharp
// Program.cs - handles requests to non-existent routes
app.UseStatusCodePages(async context =>
{
    if (context.HttpContext.Response.StatusCode == 404)
    {
        var problemDetails = new ProblemDetails
        {
            Title = "Not Found",
            Status = StatusCodes.Status404NotFound,
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4",
            Detail = $"The requested path '{context.HttpContext.Request.Path}' was not found",
            Instance = context.HttpContext.Request.Path
        };

        problemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;

        context.HttpContext.Response.ContentType = "application/problem+json";
        await context.HttpContext.Response.WriteAsJsonAsync(problemDetails);
    }
});
```

## Custom Problem Details Types

For domain-specific errors, create custom URIs:

```csharp
app.MapPost("/todos/{id}/complete", async (Guid id, ITodoService svc, CancellationToken ct) =>
{
    try
    {
        await svc.CompleteAsync(id, ct);
        return Results.NoContent();
    }
    catch (TodoAlreadyCompletedException)
    {
        var problemDetails = new ProblemDetails
        {
            Type = "https://api.example.com/problems/todo-already-completed",
            Title = "Todo Already Completed",
            Status = StatusCodes.Status409Conflict,
            Detail = $"Todo {id} has already been marked as completed",
            Instance = $"/todos/{id}/complete"
        };

        problemDetails.Extensions["todoId"] = id;
        problemDetails.Extensions["traceId"] = Activity.Current?.Id;

        return Results.Problem(problemDetails);
    }
});
```

## Testing Problem Details

```http
### Test validation error
POST https://localhost:7001/todos
Content-Type: application/json

{
  "title": "",
  "dueDate": "2020-01-01"
}

# Expected response:
# HTTP/1.1 400 Bad Request
# Content-Type: application/problem+json
# {
#   "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
#   "title": "One or more validation errors occurred.",
#   "status": 400,
#   "errors": {
#     "title": ["Title is required"],
#     "dueDate": ["Due date must be in the future"]
#   },
#   "traceId": "00-..."
# }

### Test 404
GET https://localhost:7001/todos/00000000-0000-0000-0000-000000000000

### Test 500
GET https://localhost:7001/cause-error
```

## OpenAPI Documentation

```csharp
// Ensure Problem Details are documented in Swagger
builder.Services.AddSwaggerGen(options =>
{
    options.MapType<ProblemDetails>(() => new OpenApiSchema
    {
        Type = "object",
        Properties = new Dictionary<string, OpenApiSchema>
        {
            ["type"] = new() { Type = "string" },
            ["title"] = new() { Type = "string" },
            ["status"] = new() { Type = "integer" },
            ["detail"] = new() { Type = "string" },
            ["instance"] = new() { Type = "string" },
            ["traceId"] = new() { Type = "string" }
        }
    });
});

// On endpoints, document all error responses
app.MapGet("/todos/{id}", GetTodoById)
   .Produces<TodoResponse>(StatusCodes.Status200OK)
   .ProducesProblem(StatusCodes.Status404NotFound)
   .ProducesProblem(StatusCodes.Status500InternalServerError);
```

## Common Mistakes

❌ **Mistake**: Exposing stack traces in production
```csharp
Detail = exception.Message + "\n" + exception.StackTrace // DON'T!
```

✅ **Solution**: Only include stack traces in development
```csharp
Detail = isDevelopment ? exception.Message : "An error occurred"
```

❌ **Mistake**: Not setting Content-Type header
```csharp
await context.Response.WriteAsJsonAsync(problemDetails); // Missing content type
```

✅ **Solution**: Set content type explicitly
```csharp
context.Response.ContentType = "application/problem+json";
await context.Response.WriteAsJsonAsync(problemDetails);
```

---

## Navigation
- **Previous**: `error-handling.md`
- **Related**: `validation.md`
- **Up**: `../00-overview.md`

## See Also
- [RFC 7807: Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)
- [Microsoft Docs: Handle errors in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)
