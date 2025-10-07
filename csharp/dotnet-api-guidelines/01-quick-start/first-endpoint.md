# First Endpoint - .NET 8 Minimal APIs

> **File Purpose**: Create GET/POST endpoints with route parameters, validation, and proper request/response handling
> **Prerequisites**: `minimal-program-setup.md` - Program.cs configured
> **Related Files**: `../04-api-design/endpoints-and-routing.md`, `../06-error-handling/problem-details.md`
> **Agent Use Case**: Reference when creating first minimal API endpoints with validation

## Quick Context

.NET 8 minimal APIs provide a streamlined approach for defining HTTP endpoints with less ceremony than traditional controllers. This guide covers creating GET/POST endpoints, route parameters, request/response binding, FluentValidation integration, and returning appropriate HTTP status codes following REST conventions.

**Microsoft References**:
- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Route Parameters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding)
- [Model Binding](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding)

## Simple GET Endpoint

### Basic GET Request

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Simple GET endpoint
app.MapGet("/", () => "Hello World!")
    .WithName("GetRoot")
    .WithTags("General");

// GET with typed response
app.MapGet("/health", () => new { Status = "Healthy", Timestamp = DateTime.UtcNow })
    .WithName("GetHealth")
    .WithTags("Health");

// GET with Results helper
app.MapGet("/api/status", () => Results.Ok(new
{
    Version = "1.0.0",
    Environment = builder.Environment.EnvironmentName
}))
.Produces<object>(StatusCodes.Status200OK);

app.Run();
```

### GET with Route Parameters

```csharp
// Single parameter
app.MapGet("/users/{id:int}", (int id) =>
{
    return Results.Ok(new { Id = id, Name = $"User {id}" });
})
.WithName("GetUserById")
.WithTags("Users");

// Multiple parameters
app.MapGet("/users/{userId:int}/posts/{postId:int}", (int userId, int postId) =>
{
    return Results.Ok(new
    {
        UserId = userId,
        PostId = postId,
        Title = $"Post {postId} by User {userId}"
    });
})
.WithName("GetUserPost");

// Optional parameters via query string
app.MapGet("/search", (string? query, int page = 1, int pageSize = 10) =>
{
    return Results.Ok(new
    {
        Query = query ?? "all",
        Page = page,
        PageSize = pageSize,
        Results = Array.Empty<object>()
    });
})
.WithName("Search");
```

**Route Constraints**:
- `{id:int}` - Integer only
- `{id:guid}` - GUID format
- `{slug:alpha}` - Alphabetic characters only
- `{date:datetime}` - DateTime format
- `{id:min(1)}` - Minimum value
- `{id:range(1,100)}` - Value range

## GET with Dependency Injection

### Repository Pattern

```csharp
// MyApi.Application/Common/Interfaces/IUserRepository.cs
namespace MyApi.Application.Common.Interfaces;

public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id, CancellationToken ct);
    Task<IEnumerable<User>> GetAllAsync(CancellationToken ct);
}

// MyApi.Infrastructure/Repositories/UserRepository.cs
using Microsoft.EntityFrameworkCore;

namespace MyApi.Infrastructure.Repositories;

public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<User?> GetByIdAsync(int id, CancellationToken ct)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, ct);
    }

    public async Task<IEnumerable<User>> GetAllAsync(CancellationToken ct)
    {
        return await _context.Users
            .AsNoTracking()
            .ToListAsync(ct);
    }
}

// Program.cs registration
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Endpoint with DI
app.MapGet("/api/users/{id:int}", async (
    int id,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var user = await repository.GetByIdAsync(id, ct);

    return user is null
        ? Results.NotFound(new { Message = $"User with ID {id} not found" })
        : Results.Ok(user);
})
.WithName("GetUser")
.WithTags("Users")
.Produces<User>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);
```

## Response DTOs

### Create Response Models

```csharp
// MyApi.Api/Contracts/Users/UserResponse.cs
namespace MyApi.Api.Contracts.Users;

public record UserResponse(
    int Id,
    string Name,
    string Email,
    DateTime CreatedAt
);

public record UsersListResponse(
    IEnumerable<UserResponse> Users,
    int TotalCount,
    int Page,
    int PageSize
);
```

### Map Entity to DTO

```csharp
// Using Mapster
using Mapster;

app.MapGet("/api/users/{id:int}", async (
    int id,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var user = await repository.GetByIdAsync(id, ct);

    if (user is null)
        return Results.NotFound();

    var response = user.Adapt<UserResponse>();
    return Results.Ok(response);
})
.Produces<UserResponse>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);

// Manual mapping
app.MapGet("/api/users", async (
    IUserRepository repository,
    int page = 1,
    int pageSize = 10,
    CancellationToken ct = default) =>
{
    var users = await repository.GetAllAsync(ct);

    var response = new UsersListResponse(
        Users: users.Select(u => new UserResponse(
            u.Id,
            u.Name,
            u.Email,
            u.CreatedAt
        )),
        TotalCount: users.Count(),
        Page: page,
        PageSize: pageSize
    );

    return Results.Ok(response);
})
.Produces<UsersListResponse>(StatusCodes.Status200OK);
```

## POST Endpoints

### Basic POST with Request Body

```csharp
// MyApi.Api/Contracts/Users/CreateUserRequest.cs
namespace MyApi.Api.Contracts.Users;

public record CreateUserRequest(
    string Name,
    string Email,
    string Password
);

// Endpoint
app.MapPost("/api/users", async (
    CreateUserRequest request,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var user = new User
    {
        Name = request.Name,
        Email = request.Email,
        PasswordHash = HashPassword(request.Password),
        CreatedAt = DateTime.UtcNow
    };

    await repository.AddAsync(user, ct);

    var response = user.Adapt<UserResponse>();

    return Results.Created($"/api/users/{user.Id}", response);
})
.WithName("CreateUser")
.WithTags("Users")
.Produces<UserResponse>(StatusCodes.Status201Created)
.ProducesValidationProblem();
```

### POST with Validation

#### Install FluentValidation

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

#### Create Validator

```csharp
// MyApi.Application/Validators/CreateUserRequestValidator.cs
using FluentValidation;

namespace MyApi.Application.Validators;

public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MinimumLength(2).WithMessage("Name must be at least 2 characters")
            .MaximumLength(100).WithMessage("Name cannot exceed 100 characters");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Email must be valid");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches(@"[A-Z]").WithMessage("Password must contain uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain digit")
            .Matches(@"[\W_]").WithMessage("Password must contain special character");
    }
}

// Register in Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

#### Endpoint with Validation

```csharp
using FluentValidation;

app.MapPost("/api/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserRepository repository,
    CancellationToken ct) =>
{
    // Validate request
    var validationResult = await validator.ValidateAsync(request, ct);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    // Check if email already exists
    var existingUser = await repository.GetByEmailAsync(request.Email, ct);
    if (existingUser is not null)
    {
        return Results.Conflict(new ProblemDetails
        {
            Title = "User already exists",
            Detail = "A user with this email already exists",
            Status = StatusCodes.Status409Conflict
        });
    }

    var user = new User
    {
        Name = request.Name,
        Email = request.Email,
        PasswordHash = HashPassword(request.Password),
        CreatedAt = DateTime.UtcNow
    };

    await repository.AddAsync(user, ct);

    var response = user.Adapt<UserResponse>();

    return Results.Created($"/api/users/{user.Id}", response);
})
.Produces<UserResponse>(StatusCodes.Status201Created)
.ProducesValidationProblem(StatusCodes.Status400BadRequest)
.Produces(StatusCodes.Status409Conflict);
```

### Validation Extension Helper

```csharp
// MyApi.Api/Extensions/ValidationExtensions.cs
using FluentValidation;
using FluentValidation.Results;

namespace MyApi.Api.Extensions;

public static class ValidationExtensions
{
    public static IDictionary<string, string[]> ToDictionary(this ValidationResult validationResult)
    {
        return validationResult.Errors
            .GroupBy(x => x.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(x => x.ErrorMessage).ToArray()
            );
    }
}
```

## PUT/PATCH Endpoints

### PUT (Full Update)

```csharp
// MyApi.Api/Contracts/Users/UpdateUserRequest.cs
public record UpdateUserRequest(
    string Name,
    string Email
);

// Endpoint
app.MapPut("/api/users/{id:int}", async (
    int id,
    UpdateUserRequest request,
    IValidator<UpdateUserRequest> validator,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var validationResult = await validator.ValidateAsync(request, ct);
    if (!validationResult.IsValid)
        return Results.ValidationProblem(validationResult.ToDictionary());

    var user = await repository.GetByIdAsync(id, ct);
    if (user is null)
        return Results.NotFound();

    user.Name = request.Name;
    user.Email = request.Email;
    user.UpdatedAt = DateTime.UtcNow;

    await repository.UpdateAsync(user, ct);

    var response = user.Adapt<UserResponse>();
    return Results.Ok(response);
})
.Produces<UserResponse>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound)
.ProducesValidationProblem();
```

### PATCH (Partial Update)

```csharp
using Microsoft.AspNetCore.JsonPatch;

app.MapPatch("/api/users/{id:int}", async (
    int id,
    JsonPatchDocument<User> patchDoc,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var user = await repository.GetByIdAsync(id, ct);
    if (user is null)
        return Results.NotFound();

    patchDoc.ApplyTo(user);
    user.UpdatedAt = DateTime.UtcNow;

    await repository.UpdateAsync(user, ct);

    return Results.NoContent();
})
.Produces(StatusCodes.Status204NoContent)
.Produces(StatusCodes.Status404NotFound);
```

## DELETE Endpoint

```csharp
app.MapDelete("/api/users/{id:int}", async (
    int id,
    IUserRepository repository,
    CancellationToken ct) =>
{
    var user = await repository.GetByIdAsync(id, ct);
    if (user is null)
        return Results.NotFound();

    await repository.DeleteAsync(user, ct);

    return Results.NoContent();
})
.WithName("DeleteUser")
.WithTags("Users")
.Produces(StatusCodes.Status204NoContent)
.Produces(StatusCodes.Status404NotFound)
.RequireAuthorization("Admin"); // Require authorization
```

## Organizing Endpoints

### Endpoint Group Extension

```csharp
// MyApi.Api/Endpoints/UserEndpoints.cs
namespace MyApi.Api.Endpoints;

public static class UserEndpoints
{
    public static RouteGroupBuilder MapUserEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/api/users")
            .WithTags("Users")
            .WithOpenApi();

        group.MapGet("/", GetAllUsers)
            .WithName("GetAllUsers");

        group.MapGet("/{id:int}", GetUserById)
            .WithName("GetUserById");

        group.MapPost("/", CreateUser)
            .WithName("CreateUser");

        group.MapPut("/{id:int}", UpdateUser)
            .WithName("UpdateUser");

        group.MapDelete("/{id:int}", DeleteUser)
            .WithName("DeleteUser")
            .RequireAuthorization("Admin");

        return group;
    }

    private static async Task<IResult> GetAllUsers(
        IUserRepository repository,
        int page = 1,
        int pageSize = 10,
        CancellationToken ct = default)
    {
        var users = await repository.GetPagedAsync(page, pageSize, ct);
        var response = users.Adapt<UsersListResponse>();
        return Results.Ok(response);
    }

    private static async Task<IResult> GetUserById(
        int id,
        IUserRepository repository,
        CancellationToken ct)
    {
        var user = await repository.GetByIdAsync(id, ct);

        return user is null
            ? Results.NotFound()
            : Results.Ok(user.Adapt<UserResponse>());
    }

    private static async Task<IResult> CreateUser(
        CreateUserRequest request,
        IValidator<CreateUserRequest> validator,
        IUserRepository repository,
        CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
            return Results.ValidationProblem(validationResult.ToDictionary());

        var user = request.Adapt<User>();
        user.CreatedAt = DateTime.UtcNow;

        await repository.AddAsync(user, ct);

        var response = user.Adapt<UserResponse>();
        return Results.Created($"/api/users/{user.Id}", response);
    }

    private static async Task<IResult> UpdateUser(
        int id,
        UpdateUserRequest request,
        IValidator<UpdateUserRequest> validator,
        IUserRepository repository,
        CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
            return Results.ValidationProblem(validationResult.ToDictionary());

        var user = await repository.GetByIdAsync(id, ct);
        if (user is null)
            return Results.NotFound();

        user.Name = request.Name;
        user.Email = request.Email;
        user.UpdatedAt = DateTime.UtcNow;

        await repository.UpdateAsync(user, ct);

        return Results.Ok(user.Adapt<UserResponse>());
    }

    private static async Task<IResult> DeleteUser(
        int id,
        IUserRepository repository,
        CancellationToken ct)
    {
        var user = await repository.GetByIdAsync(id, ct);
        if (user is null)
            return Results.NotFound();

        await repository.DeleteAsync(user, ct);
        return Results.NoContent();
    }
}

// Program.cs
app.MapUserEndpoints();
```

## Complete Working Example

```csharp
// Program.cs
using FluentValidation;
using Microsoft.EntityFrameworkCore;
using MyApi.Api.Endpoints;
using MyApi.Api.Extensions;
using MyApi.Application.Common.Interfaces;
using MyApi.Application.Validators;
using MyApi.Infrastructure.Persistence;
using MyApi.Infrastructure.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// Repositories
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Validation
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();

// API Documentation
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Development middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Map endpoints
app.MapUserEndpoints();

app.Run();
```

## HTTP Status Code Guidelines

**Successful Responses**:
- `200 OK` - GET, PUT successful
- `201 Created` - POST successful (include Location header)
- `204 No Content` - DELETE, PATCH successful (no body)

**Client Errors**:
- `400 Bad Request` - Validation failed
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - Authenticated but not authorized
- `404 Not Found` - Resource doesn't exist
- `409 Conflict` - Resource already exists

**Server Errors**:
- `500 Internal Server Error` - Unhandled exception

## Testing Endpoints

### HTTP File (REST Client)

```http
### Get all users
GET https://localhost:7001/api/users?page=1&pageSize=10
Content-Type: application/json

### Get user by ID
GET https://localhost:7001/api/users/1
Content-Type: application/json

### Create user
POST https://localhost:7001/api/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "SecurePass123!"
}

### Update user
PUT https://localhost:7001/api/users/1
Content-Type: application/json

{
  "name": "John Smith",
  "email": "john.smith@example.com"
}

### Delete user
DELETE https://localhost:7001/api/users/1
```

## Endpoint Checklist

- [ ] Route parameters have appropriate constraints
- [ ] CancellationToken accepted for all async operations
- [ ] Request DTOs validated with FluentValidation
- [ ] Responses use appropriate HTTP status codes
- [ ] Created responses include Location header
- [ ] NotFound returned for missing resources
- [ ] ValidationProblem returned for invalid input
- [ ] Endpoints grouped logically
- [ ] OpenAPI metadata configured (.Produces, .WithTags)
- [ ] Authorization requirements specified
- [ ] Dependency injection used for services
- [ ] DTOs used instead of exposing entities

## Navigation

- **Previous**: `minimal-program-setup.md` - Program.cs setup
- **Next**: `../04-api-design/endpoints-and-routing.md` - Advanced routing patterns
- **Related**: `../06-error-handling/problem-details.md` - Error responses

## References

- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Route Parameters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding)
- [Request/Response](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/responses)
- [FluentValidation](https://docs.fluentvalidation.net/en/latest/aspnet.html)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
