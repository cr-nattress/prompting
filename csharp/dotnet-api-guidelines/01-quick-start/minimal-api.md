# Minimal API Implementation

> **File Purpose**: Create a basic CRUD endpoint with validation using Minimal APIs
> **Prerequisites**: `setup.md` - Project scaffolded
> **Related Files**: `../05-validation-errors/validation.md`, `../02-core-concepts/dependency-injection.md`
> **Agent Use Case**: Reference when implementing HTTP endpoints in .NET 8

## 🎯 Quick Context

Minimal APIs (introduced in .NET 6, refined in .NET 8) provide a streamlined approach to building HTTP APIs without controllers. They reduce boilerplate while maintaining testability and performance. This guide shows a complete CRUD example with validation.

## Basic GET Endpoint

```csharp
// Add to Program.cs
app.MapGet("/todos", () => Results.Ok(new { message = "Todo list" }))
   .WithName("GetTodos")
   .WithTags("Todos")
   .Produces<object>(StatusCodes.Status200OK);
```

**Key Components**:
- `MapGet`: Registers HTTP GET handler
- `WithName`: Provides endpoint name for URL generation
- `WithTags`: Groups endpoints in Swagger UI
- `Produces`: Documents response type for OpenAPI

## POST Endpoint with Validation

### Step 1: Create Request DTO

```csharp
// Demo.Application/DTOs/CreateTodoRequest.cs
namespace Demo.Application.DTOs;

public record CreateTodoRequest(
    string Title,
    string? Description,
    DateTime? DueDate
);
```

**Why `record`**: Immutable by default, built-in equality, concise syntax.

### Step 2: Add FluentValidation

```csharp
// Demo.Application/Validators/CreateTodoValidator.cs
using FluentValidation;

namespace Demo.Application.Validators;

public class CreateTodoValidator : AbstractValidator<CreateTodoRequest>
{
    public CreateTodoValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("Title is required")
            .MaximumLength(200).WithMessage("Title must not exceed 200 characters");

        RuleFor(x => x.Description)
            .MaximumLength(1000).WithMessage("Description must not exceed 1000 characters");

        RuleFor(x => x.DueDate)
            .GreaterThan(DateTime.UtcNow).WithMessage("Due date must be in the future")
            .When(x => x.DueDate.HasValue);
    }
}
```

### Step 3: Create Validation Filter

```csharp
// Demo.Api/Filters/ValidationFilter.cs
using FluentValidation;

namespace Demo.Api.Filters;

public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<T>>();

        if (validator is null)
            return await next(context);

        var entity = context.Arguments
            .OfType<T>()
            .FirstOrDefault();

        if (entity is null)
            return Results.BadRequest("Request body is required");

        var validation = await validator.ValidateAsync(entity);

        if (!validation.IsValid)
        {
            return Results.ValidationProblem(
                validation.ToDictionary(),
                title: "Validation Failed"
            );
        }

        return await next(context);
    }
}
```

### Step 4: Register Validators in DI

```csharp
// Program.cs
using FluentValidation;

builder.Services.AddValidatorsFromAssemblyContaining<CreateTodoValidator>();
```

### Step 5: Implement POST Endpoint

```csharp
// Program.cs
app.MapPost("/todos", async (
    CreateTodoRequest request,
    ITodoService todoService,
    CancellationToken ct) =>
{
    var id = await todoService.CreateAsync(request, ct);
    return Results.Created($"/todos/{id}", new { id });
})
.AddEndpointFilter<ValidationFilter<CreateTodoRequest>>()
.WithName("CreateTodo")
.WithTags("Todos")
.Produces<object>(StatusCodes.Status201Created)
.ProducesValidationProblem();
```

**Key Features**:
- `AddEndpointFilter`: Applies validation before handler executes
- `CancellationToken`: Enables request cancellation
- `Results.Created`: Returns 201 with Location header
- `ProducesValidationProblem`: Documents 400 responses in Swagger

## Complete CRUD Example

```csharp
// Program.cs (add after builder.Build())

var todos = app.MapGroup("/todos")
    .WithTags("Todos");

// GET /todos
todos.MapGet("/", async (ITodoService svc, CancellationToken ct) =>
{
    var items = await svc.GetAllAsync(ct);
    return Results.Ok(items);
})
.WithName("GetTodos")
.Produces<IEnumerable<TodoResponse>>(StatusCodes.Status200OK);

// GET /todos/{id}
todos.MapGet("/{id:guid}", async (
    Guid id,
    ITodoService svc,
    CancellationToken ct) =>
{
    var todo = await svc.GetByIdAsync(id, ct);
    return todo is not null
        ? Results.Ok(todo)
        : Results.NotFound(new { message = $"Todo {id} not found" });
})
.WithName("GetTodoById")
.Produces<TodoResponse>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);

// POST /todos
todos.MapPost("/", async (
    CreateTodoRequest request,
    ITodoService svc,
    CancellationToken ct) =>
{
    var id = await svc.CreateAsync(request, ct);
    return Results.CreatedAtRoute("GetTodoById", new { id }, new { id });
})
.AddEndpointFilter<ValidationFilter<CreateTodoRequest>>()
.WithName("CreateTodo")
.Produces<object>(StatusCodes.Status201Created)
.ProducesValidationProblem();

// PUT /todos/{id}
todos.MapPut("/{id:guid}", async (
    Guid id,
    UpdateTodoRequest request,
    ITodoService svc,
    CancellationToken ct) =>
{
    var updated = await svc.UpdateAsync(id, request, ct);
    return updated
        ? Results.NoContent()
        : Results.NotFound(new { message = $"Todo {id} not found" });
})
.AddEndpointFilter<ValidationFilter<UpdateTodoRequest>>()
.WithName("UpdateTodo")
.Produces(StatusCodes.Status204NoContent)
.Produces(StatusCodes.Status404NotFound)
.ProducesValidationProblem();

// DELETE /todos/{id}
todos.MapDelete("/{id:guid}", async (
    Guid id,
    ITodoService svc,
    CancellationToken ct) =>
{
    var deleted = await svc.DeleteAsync(id, ct);
    return deleted
        ? Results.NoContent()
        : Results.NotFound(new { message = $"Todo {id} not found" });
})
.WithName("DeleteTodo")
.Produces(StatusCodes.Status204NoContent)
.Produces(StatusCodes.Status404NotFound);
```

## Service Interface Example

```csharp
// Demo.Application/Services/ITodoService.cs
namespace Demo.Application.Services;

public interface ITodoService
{
    Task<IEnumerable<TodoResponse>> GetAllAsync(CancellationToken ct);
    Task<TodoResponse?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<Guid> CreateAsync(CreateTodoRequest request, CancellationToken ct);
    Task<bool> UpdateAsync(Guid id, UpdateTodoRequest request, CancellationToken ct);
    Task<bool> DeleteAsync(Guid id, CancellationToken ct);
}
```

## Service Implementation (Stub)

```csharp
// Demo.Application/Services/TodoService.cs
namespace Demo.Application.Services;

public class TodoService : ITodoService
{
    private readonly ILogger<TodoService> _logger;
    private static readonly List<TodoResponse> _todos = new();

    public TodoService(ILogger<TodoService> logger)
    {
        _logger = logger;
    }

    public Task<IEnumerable<TodoResponse>> GetAllAsync(CancellationToken ct)
    {
        _logger.LogInformation("Fetching all todos");
        return Task.FromResult<IEnumerable<TodoResponse>>(_todos);
    }

    public Task<TodoResponse?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var todo = _todos.FirstOrDefault(t => t.Id == id);
        return Task.FromResult(todo);
    }

    public Task<Guid> CreateAsync(CreateTodoRequest request, CancellationToken ct)
    {
        var id = Guid.NewGuid();
        _todos.Add(new TodoResponse(id, request.Title, request.Description, request.DueDate, false));
        _logger.LogInformation("Created todo {TodoId}", id);
        return Task.FromResult(id);
    }

    public Task<bool> UpdateAsync(Guid id, UpdateTodoRequest request, CancellationToken ct)
    {
        var todo = _todos.FirstOrDefault(t => t.Id == id);
        if (todo is null) return Task.FromResult(false);

        _todos.Remove(todo);
        _todos.Add(new TodoResponse(id, request.Title, request.Description, request.DueDate, request.IsCompleted));
        return Task.FromResult(true);
    }

    public Task<bool> DeleteAsync(Guid id, CancellationToken ct)
    {
        var todo = _todos.FirstOrDefault(t => t.Id == id);
        if (todo is null) return Task.FromResult(false);

        _todos.Remove(todo);
        return Task.FromResult(true);
    }
}

public record TodoResponse(Guid Id, string Title, string? Description, DateTime? DueDate, bool IsCompleted);
```

## Register Service

```csharp
// Program.cs
builder.Services.AddScoped<ITodoService, TodoService>();
```

## Testing with HTTP File

```http
### Create Todo
POST https://localhost:7001/todos
Content-Type: application/json

{
  "title": "Learn Minimal APIs",
  "description": "Build a production-ready API",
  "dueDate": "2025-12-31T23:59:59Z"
}

### Get All Todos
GET https://localhost:7001/todos

### Get Todo by ID
GET https://localhost:7001/todos/{{todoId}}

### Update Todo
PUT https://localhost:7001/todos/{{todoId}}
Content-Type: application/json

{
  "title": "Learn Minimal APIs - Updated",
  "isCompleted": true
}

### Delete Todo
DELETE https://localhost:7001/todos/{{todoId}}
```

## Common Patterns

### Route Groups for Versioning
```csharp
var v1 = app.MapGroup("/api/v1");
v1.MapGet("/todos", GetTodosV1);

var v2 = app.MapGroup("/api/v2");
v2.MapGet("/todos", GetTodosV2);
```

### Filters for Cross-Cutting Concerns
```csharp
todos.AddEndpointFilter(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    var result = await next(context);
    sw.Stop();
    context.HttpContext.Response.Headers.Add("X-Response-Time", $"{sw.ElapsedMilliseconds}ms");
    return result;
});
```

## Common Mistakes

❌ **Mistake**: Putting business logic in endpoint handlers
```csharp
app.MapPost("/todos", (CreateTodoRequest req) =>
{
    // DON'T: Business logic here
    if (req.Title.Contains("bad"))
        return Results.BadRequest();
});
```

✅ **Solution**: Delegate to Application layer
```csharp
app.MapPost("/todos", async (CreateTodoRequest req, ITodoService svc, CancellationToken ct) =>
{
    var id = await svc.CreateAsync(req, ct); // Business logic in service
    return Results.Created($"/todos/{id}", new { id });
});
```

---

## Navigation
- **Previous**: `setup.md`
- **Next**: `../04-data-persistence/ef-core-setup.md`
- **Up**: `../00-overview.md`

## See Also
- `../05-validation-errors/validation.md` - Detailed validation strategies
- `../02-core-concepts/async-patterns.md` - CancellationToken usage
- `../12-reference/code-examples.md` - Complete working examples
