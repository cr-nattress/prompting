# API Standards - REST Conventions

> **File Purpose**: Implement pagination, filtering, sorting, idempotency, and standard HTTP status codes
> **Prerequisites**: `endpoints-and-routing.md` - Endpoint organization
> **Related Files**: `versioning.md`, `openapi-swagger.md`
> **Agent Use Case**: Reference when implementing REST best practices and API conventions

## Quick Context

RESTful API standards ensure consistency, predictability, and developer-friendly interfaces. This guide covers pagination, filtering, sorting, idempotency headers, proper status code usage, and HATEOAS principles.

**Microsoft References**:
- [Web API Design Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md)

## Pagination

### Offset-Based Pagination

```csharp
// MyApi.Api/Contracts/Common/PagedResponse.cs
public record PagedResponse<T>(
    IEnumerable<T> Items,
    int Page,
    int PageSize,
    int TotalCount,
    int TotalPages
)
{
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}

// Endpoint
app.MapGet("/api/products", async (
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 10,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var totalCount = await repo.CountAsync(ct);
    var items = await repo.GetPagedAsync(page, pageSize, ct);

    var response = new PagedResponse<ProductResponse>(
        items.Select(p => p.Adapt<ProductResponse>()),
        page,
        pageSize,
        totalCount,
        (int)Math.Ceiling(totalCount / (double)pageSize)
    );

    return Results.Ok(response);
})
.Produces<PagedResponse<ProductResponse>>();

// Repository implementation
public async Task<IEnumerable<Product>> GetPagedAsync(
    int page,
    int pageSize,
    CancellationToken ct)
{
    return await _context.Products
        .OrderBy(p => p.Id)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .AsNoTracking()
        .ToListAsync(ct);
}
```

### Cursor-Based Pagination

```csharp
public record CursorPagedResponse<T>(
    IEnumerable<T> Items,
    string? NextCursor,
    string? PreviousCursor,
    int PageSize
);

app.MapGet("/api/products/cursor", async (
    [FromQuery] string? cursor,
    [FromQuery] int pageSize = 10,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var (items, nextCursor, previousCursor) = await repo.GetCursorPagedAsync(
        cursor,
        pageSize,
        ct
    );

    return Results.Ok(new CursorPagedResponse<ProductResponse>(
        items.Select(p => p.Adapt<ProductResponse>()),
        nextCursor,
        previousCursor,
        pageSize
    ));
});

// Repository
public async Task<(IEnumerable<Product>, string?, string?)> GetCursorPagedAsync(
    string? cursor,
    int pageSize,
    CancellationToken ct)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(cursor))
    {
        var decodedCursor = Convert.FromBase64String(cursor);
        var cursorId = BitConverter.ToInt32(decodedCursor);
        query = query.Where(p => p.Id > cursorId);
    }

    var items = await query
        .OrderBy(p => p.Id)
        .Take(pageSize + 1)
        .AsNoTracking()
        .ToListAsync(ct);

    var hasMore = items.Count > pageSize;
    var result = hasMore ? items.Take(pageSize) : items;

    var nextCursor = hasMore
        ? Convert.ToBase64String(BitConverter.GetBytes(items[pageSize - 1].Id))
        : null;

    return (result, nextCursor, null);
}
```

## Filtering

```csharp
// Query object pattern
public record ProductFilters(
    string? Name,
    decimal? MinPrice,
    decimal? MaxPrice,
    string? Category,
    bool? InStock
);

app.MapGet("/api/products", async (
    [AsParameters] ProductFilters filters,
    [AsParameters] PaginationParameters pagination,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrWhiteSpace(filters.Name))
    {
        query = query.Where(p => p.Name.Contains(filters.Name));
    }

    if (filters.MinPrice.HasValue)
    {
        query = query.Where(p => p.Price >= filters.MinPrice.Value);
    }

    if (filters.MaxPrice.HasValue)
    {
        query = query.Where(p => p.Price <= filters.MaxPrice.Value);
    }

    if (!string.IsNullOrWhiteSpace(filters.Category))
    {
        query = query.Where(p => p.Category == filters.Category);
    }

    if (filters.InStock.HasValue)
    {
        query = query.Where(p => p.Quantity > 0 == filters.InStock.Value);
    }

    var totalCount = await query.CountAsync(ct);
    var items = await query
        .OrderBy(p => p.Name)
        .Skip((pagination.Page - 1) * pagination.PageSize)
        .Take(pagination.PageSize)
        .AsNoTracking()
        .ToListAsync(ct);

    return Results.Ok(new PagedResponse<Product>(
        items,
        pagination.Page,
        pagination.PageSize,
        totalCount,
        (int)Math.Ceiling(totalCount / (double)pagination.PageSize)
    ));
});

// Example request
// GET /api/products?name=laptop&minPrice=500&maxPrice=2000&category=electronics&inStock=true&page=1&pageSize=20
```

## Sorting

```csharp
public record SortOptions(
    string? SortBy,
    string? SortOrder = "asc"
);

app.MapGet("/api/products", async (
    [AsParameters] SortOptions sort,
    IProductRepository repo,
    CancellationToken ct) =>
{
    var query = _context.Products.AsQueryable();

    query = (sort.SortBy?.ToLower(), sort.SortOrder?.ToLower()) switch
    {
        ("name", "desc") => query.OrderByDescending(p => p.Name),
        ("name", _) => query.OrderBy(p => p.Name),
        ("price", "desc") => query.OrderByDescending(p => p.Price),
        ("price", _) => query.OrderBy(p => p.Price),
        ("createdat", "desc") => query.OrderByDescending(p => p.CreatedAt),
        ("createdat", _) => query.OrderBy(p => p.CreatedAt),
        _ => query.OrderBy(p => p.Id) // Default sort
    };

    var items = await query.ToListAsync(ct);
    return Results.Ok(items);
});

// Example: GET /api/products?sortBy=price&sortOrder=desc
```

## Idempotency

```csharp
// Idempotency middleware
public class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;

    public IdempotencyMiddleware(RequestDelegate next, IDistributedCache cache)
    {
        _next = next;
        _cache = cache;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method == HttpMethods.Post ||
            context.Request.Method == HttpMethods.Put)
        {
            var idempotencyKey = context.Request.Headers["Idempotency-Key"].FirstOrDefault();

            if (!string.IsNullOrEmpty(idempotencyKey))
            {
                var cachedResponse = await _cache.GetStringAsync(idempotencyKey);

                if (cachedResponse is not null)
                {
                    // Return cached response
                    context.Response.StatusCode = StatusCodes.Status200OK;
                    context.Response.ContentType = "application/json";
                    await context.Response.WriteAsync(cachedResponse);
                    return;
                }

                // Capture response
                var originalBody = context.Response.Body;
                using var memoryStream = new MemoryStream();
                context.Response.Body = memoryStream;

                await _next(context);

                memoryStream.Seek(0, SeekOrigin.Begin);
                var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();

                // Cache successful responses
                if (context.Response.StatusCode >= 200 && context.Response.StatusCode < 300)
                {
                    await _cache.SetStringAsync(
                        idempotencyKey,
                        responseBody,
                        new DistributedCacheEntryOptions
                        {
                            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
                        }
                    );
                }

                memoryStream.Seek(0, SeekOrigin.Begin);
                await memoryStream.CopyToAsync(originalBody);
                context.Response.Body = originalBody;

                return;
            }
        }

        await _next(context);
    }
}

// Register
app.UseMiddleware<IdempotencyMiddleware>();
```

## HTTP Status Codes

### Success Codes

```csharp
// 200 OK - Successful GET, PUT, PATCH
app.MapGet("/api/users/{id}", async (int id, IUserRepository repo, CancellationToken ct) =>
{
    var user = await repo.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

// 201 Created - Successful POST with resource creation
app.MapPost("/api/users", async (CreateUserRequest request, IUserRepository repo, CancellationToken ct) =>
{
    var user = await repo.AddAsync(request.Adapt<User>(), ct);
    return Results.Created($"/api/users/{user.Id}", user);
});

// 202 Accepted - Request accepted for async processing
app.MapPost("/api/reports", async (ReportRequest request, IBackgroundQueue queue) =>
{
    var jobId = Guid.NewGuid();
    await queue.EnqueueAsync(new GenerateReportJob(jobId, request));
    return Results.Accepted($"/api/reports/{jobId}", new { JobId = jobId });
});

// 204 No Content - Successful DELETE or PUT with no response body
app.MapDelete("/api/users/{id}", async (int id, IUserRepository repo, CancellationToken ct) =>
{
    await repo.DeleteAsync(id, ct);
    return Results.NoContent();
});
```

### Client Error Codes

```csharp
// 400 Bad Request - Validation errors
app.MapPost("/api/users", async (CreateUserRequest request, IValidator<CreateUserRequest> validator, CancellationToken ct) =>
{
    var result = await validator.ValidateAsync(request, ct);
    if (!result.IsValid)
        return Results.ValidationProblem(result.ToDictionary());

    return Results.Created("/api/users/1", new User());
});

// 401 Unauthorized - Authentication required
app.MapGet("/api/secure", () => Results.Ok("Data"))
    .RequireAuthorization();

// 403 Forbidden - Authenticated but insufficient permissions
app.MapDelete("/api/users/{id}", async (int id) => Results.NoContent())
    .RequireAuthorization("Admin");

// 404 Not Found - Resource doesn't exist
app.MapGet("/api/users/{id}", async (int id, IUserRepository repo, CancellationToken ct) =>
{
    var user = await repo.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

// 409 Conflict - Resource already exists or constraint violation
app.MapPost("/api/users", async (CreateUserRequest request, IUserRepository repo, CancellationToken ct) =>
{
    var exists = await repo.ExistsByEmailAsync(request.Email, ct);
    if (exists)
        return Results.Conflict(new ProblemDetails
        {
            Title = "User already exists",
            Detail = $"A user with email {request.Email} already exists"
        });

    return Results.Created("/api/users/1", new User());
});

// 422 Unprocessable Entity - Semantically incorrect request
// 429 Too Many Requests - Rate limit exceeded (see api-protection.md)
```

## REST Best Practices Checklist

- [ ] Use plural nouns for resource names (`/users`, not `/user`)
- [ ] Use HTTP methods correctly (GET, POST, PUT, PATCH, DELETE)
- [ ] Implement pagination for list endpoints
- [ ] Support filtering via query parameters
- [ ] Support sorting via query parameters
- [ ] Return 201 Created with Location header for POST
- [ ] Return 204 No Content for DELETE
- [ ] Return 404 Not Found for missing resources
- [ ] Return 409 Conflict for duplicate resources
- [ ] Implement idempotency for POST/PUT requests
- [ ] Use proper HTTP status codes consistently
- [ ] Version API for breaking changes
- [ ] Include HATEOAS links for navigation (optional)

## Navigation

- **Previous**: `openapi-swagger.md` - API documentation
- **Next**: `../05-security/authentication-jwt.md` - Security
- **Related**: `versioning.md` - API versioning

## References

- [Microsoft API Design Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [RFC 7231 - HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc7231)
