# Endpoints and Routing - .NET 8

> **File Purpose**: Design API endpoints using Minimal APIs vs Controllers, route conventions, and endpoint filters
> **Prerequisites**: `../01-quick-start/first-endpoint.md` - Basic endpoint creation
> **Related Files**: `versioning.md`, `api-standards.md`, `openapi-swagger.md`
> **Agent Use Case**: Reference when designing API route structure and choosing between Minimal APIs and Controllers

## Quick Context

.NET 8 offers two approaches for defining endpoints: Minimal APIs (lightweight, functional) and Controllers (traditional, object-oriented). This guide covers when to use each, route organization patterns, endpoint filters, route constraints, and best practices for scalable API design.

**Microsoft References**:
- [Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Controllers](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [Routing](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing)

## Minimal APIs vs Controllers

### When to Use Minimal APIs

**Best For**:
- Small to medium APIs (< 50 endpoints)
- Microservices
- Simple CRUD operations
- Function-based programming
- Low ceremony, high performance

```csharp
// Minimal API Example
app.MapGet("/api/products/{id:int}", async (
    int id,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var product = await repo.GetByIdAsync(id, ct);
    return product is null ? Results.NotFound() : Results.Ok(product);
})
.WithName("GetProduct")
.WithTags("Products")
.Produces<Product>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);
```

**Advantages**:
- Less boilerplate code
- ~30% faster startup time
- Direct routing definition
- Easy to understand for simple cases
- Native to .NET 6+

### When to Use Controllers

**Best For**:
- Large APIs (50+ endpoints)
- Complex business logic
- Heavy use of attributes/filters
- Team familiar with MVC pattern
- Need for action method overloading

```csharp
// Controller Example
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(Product), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<Product>> GetProduct(
        int id,
        CancellationToken ct)
    {
        var product = await _repository.GetByIdAsync(id, ct);
        return product is null ? NotFound() : Ok(product);
    }
}
```

**Advantages**:
- Better organization for large codebases
- Familiar pattern for ASP.NET developers
- Built-in model binding
- Easier to apply filters/attributes
- Better IDE support

**Recommendation**: Use Minimal APIs for new .NET 8 projects unless you have specific requirements for Controllers.

## Route Organization Patterns

### Pattern 1: Resource-Based Routes

```csharp
// RESTful resource routes
app.MapGet("/api/users", GetUsers);                      // GET all
app.MapGet("/api/users/{id:int}", GetUser);              // GET by ID
app.MapPost("/api/users", CreateUser);                   // CREATE
app.MapPut("/api/users/{id:int}", UpdateUser);           // UPDATE
app.MapDelete("/api/users/{id:int}", DeleteUser);        // DELETE

// Nested resources
app.MapGet("/api/users/{userId:int}/orders", GetUserOrders);
app.MapGet("/api/users/{userId:int}/orders/{orderId:int}", GetUserOrder);
```

### Pattern 2: Route Groups

```csharp
// Group related endpoints
var usersApi = app.MapGroup("/api/users")
    .WithTags("Users")
    .WithOpenApi();

usersApi.MapGet("/", GetUsers);
usersApi.MapGet("/{id:int}", GetUser);
usersApi.MapPost("/", CreateUser);
usersApi.MapPut("/{id:int}", UpdateUser);
usersApi.MapDelete("/{id:int}", DeleteUser);

// Nested groups
var userOrdersApi = usersApi.MapGroup("/{userId:int}/orders")
    .WithTags("Orders");

userOrdersApi.MapGet("/", GetUserOrders);
userOrdersApi.MapGet("/{orderId:int}", GetUserOrder);
```

### Pattern 3: Versioned Groups

```csharp
var v1 = app.MapGroup("/api/v1")
    .WithOpenApi();

var v1Users = v1.MapGroup("/users")
    .WithTags("Users - V1");

v1Users.MapGet("/", GetUsersV1);
v1Users.MapGet("/{id:int}", GetUserV1);

var v2 = app.MapGroup("/api/v2")
    .WithOpenApi();

var v2Users = v2.MapGroup("/users")
    .WithTags("Users - V2");

v2Users.MapGet("/", GetUsersV2);
v2Users.MapGet("/{id:int}", GetUserV2);
```

### Pattern 4: Feature-Based Organization

```csharp
// MyApi.Api/Features/Users/UserEndpoints.cs
namespace MyApi.Api.Features.Users;

public static class UserEndpoints
{
    public static IEndpointRouteBuilder MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/users")
            .WithTags("Users");

        group.MapGet("/", GetAllAsync);
        group.MapGet("/{id:int}", GetByIdAsync);
        group.MapPost("/", CreateAsync);
        group.MapPut("/{id:int}", UpdateAsync);
        group.MapDelete("/{id:int}", DeleteAsync);

        return app;
    }

    private static async Task<IResult> GetAllAsync(
        IUserService service,
        CancellationToken ct)
    {
        var users = await service.GetAllUsersAsync(ct);
        return Results.Ok(users);
    }

    // Other endpoint handlers...
}

// Program.cs
app.MapUserEndpoints();
app.MapProductEndpoints();
app.MapOrderEndpoints();
```

## Route Constraints

### Built-in Constraints

```csharp
// Integer
app.MapGet("/api/users/{id:int}", handler);

// GUID
app.MapGet("/api/orders/{orderId:guid}", handler);

// String with minimum length
app.MapGet("/api/products/{sku:minlength(5)}", handler);

// Range
app.MapGet("/api/items/{quantity:range(1,100)}", handler);

// Regular expression
app.MapGet("/api/products/{code:regex(^[A-Z]{{3}}-\\d{{4}}$)}", handler);

// Multiple constraints
app.MapGet("/api/users/{id:int:min(1)}", handler);

// Optional parameter (nullable)
app.MapGet("/api/search/{query?}", handler);
```

**Common Constraints**:
- `int`, `bool`, `datetime`, `decimal`, `double`, `float`, `guid`, `long`
- `alpha` - Alphabetic characters
- `min(value)` - Minimum integer value
- `max(value)` - Maximum integer value
- `range(min,max)` - Integer range
- `length(value)` - Exact string length
- `minlength(value)` - Minimum string length
- `maxlength(value)` - Maximum string length
- `regex(expression)` - Regular expression match

### Custom Route Constraints

```csharp
// MyApi.Api/Routing/SlugConstraint.cs
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

public class SlugConstraint : IRouteConstraint
{
    private static readonly Regex SlugRegex = new(@"^[a-z0-9]+(?:-[a-z0-9]+)*$");

    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value) && value is string slug)
        {
            return SlugRegex.IsMatch(slug);
        }

        return false;
    }
}

// Register in Program.cs
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("slug", typeof(SlugConstraint));
});

// Usage
app.MapGet("/api/posts/{slug:slug}", async (string slug) =>
{
    // slug is guaranteed to match pattern
    return Results.Ok(new { Slug = slug });
});
```

## Endpoint Filters

### Built-in Filters

```csharp
// Add filters to route groups
var api = app.MapGroup("/api")
    .AddEndpointFilter(async (context, next) =>
    {
        // Before endpoint execution
        var stopwatch = Stopwatch.StartNew();

        var result = await next(context);

        // After endpoint execution
        stopwatch.Stop();
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILogger<Program>>();

        logger.LogInformation(
            "Endpoint {Endpoint} executed in {ElapsedMs}ms",
            context.HttpContext.GetEndpoint()?.DisplayName,
            stopwatch.ElapsedMilliseconds
        );

        return result;
    });
```

### Custom Endpoint Filter

```csharp
// MyApi.Api/Filters/ValidationFilter.cs
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

        var model = context.Arguments
            .OfType<T>()
            .FirstOrDefault();

        if (model is null)
            return await next(context);

        var validationResult = await validator.ValidateAsync(model);

        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(
                validationResult.ToDictionary()
            );
        }

        return await next(context);
    }
}

// Usage
app.MapPost("/api/users", CreateUser)
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

### Filter Factory Pattern

```csharp
public class LoggingFilterFactory : IEndpointFilterFactory
{
    public EndpointFilterDelegate CreateFilterDelegate(
        EndpointFilterFactoryContext context,
        EndpointFilterDelegate next)
    {
        return async invocationContext =>
        {
            var logger = invocationContext.HttpContext.RequestServices
                .GetRequiredService<ILogger<LoggingFilterFactory>>();

            logger.LogInformation(
                "Executing {Endpoint}",
                invocationContext.HttpContext.GetEndpoint()?.DisplayName
            );

            return await next(invocationContext);
        };
    }
}

// Apply to group
var api = app.MapGroup("/api")
    .AddEndpointFilterFactory(new LoggingFilterFactory());
```

## Request/Response Binding

### Parameter Binding Sources

```csharp
app.MapPost("/api/products/{id:int}", async (
    // Route parameter
    int id,

    // Query string
    [FromQuery] int? categoryId,

    // Request body
    [FromBody] UpdateProductRequest request,

    // Header
    [FromHeader(Name = "X-API-Key")] string apiKey,

    // Services (DI)
    IProductRepository repository,

    // HttpContext
    HttpContext context,

    // ClaimsPrincipal (if authenticated)
    ClaimsPrincipal user,

    // CancellationToken
    CancellationToken ct) =>
{
    // Implementation
    return Results.Ok();
});
```

**Binding Rules**:
- Route parameters: `{parameter}` in route template
- Query parameters: primitive types without attributes
- Request body: complex types with `[FromBody]`
- Services: registered in DI container
- Special types: `HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken`

### Custom Binding

```csharp
public class PaginationParameters
{
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 10;

    public static ValueTask<PaginationParameters?> BindAsync(
        HttpContext context,
        ParameterInfo parameter)
    {
        int.TryParse(context.Request.Query["page"], out var page);
        int.TryParse(context.Request.Query["pageSize"], out var pageSize);

        var result = new PaginationParameters
        {
            Page = page > 0 ? page : 1,
            PageSize = pageSize > 0 && pageSize <= 100 ? pageSize : 10
        };

        return ValueTask.FromResult<PaginationParameters?>(result);
    }
}

// Usage
app.MapGet("/api/products", async (
    PaginationParameters pagination,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var products = await repo.GetPagedAsync(
        pagination.Page,
        pagination.PageSize,
        ct
    );

    return Results.Ok(products);
});
```

## Response Patterns

### Typed Results

```csharp
// Explicit return types for better OpenAPI documentation
app.MapGet("/api/products/{id:int}", async Task<Results<Ok<Product>, NotFound>> (
    int id,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var product = await repo.GetByIdAsync(id, ct);

    return product is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(product);
})
.WithName("GetProduct");
```

### Common Response Helpers

```csharp
// Success responses
Results.Ok(data);                           // 200
Results.Created($"/api/resource/{id}", data); // 201
Results.Accepted($"/api/jobs/{jobId}");     // 202
Results.NoContent();                        // 204

// Client error responses
Results.BadRequest();                       // 400
Results.ValidationProblem(errors);          // 400
Results.Unauthorized();                     // 401
Results.Forbid();                          // 403
Results.NotFound();                        // 404
Results.Conflict();                        // 409
Results.UnprocessableEntity();             // 422

// Server error responses
Results.Problem();                         // 500

// File responses
Results.File(bytes, "application/pdf");
Results.Stream(stream, "application/json");

// Redirect responses
Results.Redirect("/new-location");         // 302
Results.RedirectPermanent("/new-location"); // 301
```

## Complete Endpoint Organization Example

```csharp
// MyApi.Api/Features/Products/ProductEndpoints.cs
namespace MyApi.Api.Features.Products;

public static class ProductEndpoints
{
    public static IEndpointRouteBuilder MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var products = app.MapGroup("/api/products")
            .WithTags("Products")
            .WithOpenApi()
            .AddEndpointFilter<ValidationFilter<CreateProductRequest>>();

        // Public endpoints
        products.MapGet("/", GetAll)
            .WithName("GetProducts")
            .Produces<PagedResponse<ProductResponse>>();

        products.MapGet("/{id:int}", GetById)
            .WithName("GetProduct")
            .Produces<ProductResponse>(StatusCodes.Status200OK)
            .Produces(StatusCodes.Status404NotFound);

        products.MapGet("/slug/{slug:slug}", GetBySlug)
            .WithName("GetProductBySlug")
            .Produces<ProductResponse>(StatusCodes.Status200OK)
            .Produces(StatusCodes.Status404NotFound);

        // Protected endpoints
        var protectedProducts = products
            .RequireAuthorization()
            .AddEndpointFilter<AuditFilter>();

        protectedProducts.MapPost("/", Create)
            .WithName("CreateProduct")
            .Produces<ProductResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();

        protectedProducts.MapPut("/{id:int}", Update)
            .WithName("UpdateProduct")
            .Produces<ProductResponse>(StatusCodes.Status200OK)
            .Produces(StatusCodes.Status404NotFound)
            .ProducesValidationProblem();

        protectedProducts.MapDelete("/{id:int}", Delete)
            .WithName("DeleteProduct")
            .Produces(StatusCodes.Status204NoContent)
            .Produces(StatusCodes.Status404NotFound)
            .RequireAuthorization("Admin");

        return app;
    }

    private static async Task<IResult> GetAll(
        [AsParameters] PaginationParameters pagination,
        [AsParameters] ProductFilters filters,
        IProductService service,
        CancellationToken ct)
    {
        var result = await service.GetProductsAsync(
            pagination.Page,
            pagination.PageSize,
            filters,
            ct
        );

        return Results.Ok(result);
    }

    private static async Task<Results<Ok<ProductResponse>, NotFound>> GetById(
        int id,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.GetProductByIdAsync(id, ct);

        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<Ok<ProductResponse>, NotFound>> GetBySlug(
        string slug,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.GetProductBySlugAsync(slug, ct);

        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<Created<ProductResponse>, ValidationProblem>> Create(
        CreateProductRequest request,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.CreateProductAsync(request, ct);

        return TypedResults.Created(
            $"/api/products/{product.Id}",
            product
        );
    }

    private static async Task<Results<Ok<ProductResponse>, NotFound, ValidationProblem>> Update(
        int id,
        UpdateProductRequest request,
        IProductService service,
        CancellationToken ct)
    {
        var product = await service.UpdateProductAsync(id, request, ct);

        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<NoContent, NotFound>> Delete(
        int id,
        IProductService service,
        CancellationToken ct)
    {
        var deleted = await service.DeleteProductAsync(id, ct);

        return deleted
            ? TypedResults.NoContent()
            : TypedResults.NotFound();
    }
}

// Program.cs
app.MapProductEndpoints();
```

## Routing Best Practices Checklist

- [ ] Use resource-based URL structure (`/api/resources/{id}`)
- [ ] Apply appropriate route constraints to parameters
- [ ] Group related endpoints with `MapGroup()`
- [ ] Use semantic HTTP methods (GET, POST, PUT, DELETE)
- [ ] Include OpenAPI metadata (`.WithTags()`, `.Produces()`)
- [ ] Implement endpoint filters for cross-cutting concerns
- [ ] Use typed results for better documentation
- [ ] Accept CancellationToken in all async endpoints
- [ ] Apply authorization requirements explicitly
- [ ] Avoid deep nesting (max 2 levels: `/users/{id}/orders/{id}`)
- [ ] Use lowercase URLs with hyphens for readability
- [ ] Version APIs when making breaking changes

## Navigation

- **Previous**: `../01-quick-start/first-endpoint.md` - Basic endpoints
- **Next**: `versioning.md` - API versioning strategies
- **Related**: `api-standards.md` - REST conventions

## References

- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Routing in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing)
- [Route Constraints](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing#route-constraint-reference)
- [Endpoint Filters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters)
- [Parameter Binding](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding)
