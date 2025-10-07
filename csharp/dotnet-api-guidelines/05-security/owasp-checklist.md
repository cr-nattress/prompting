# OWASP API Security Top 10 Checklist

> **File Purpose**: Comprehensive security checklist based on OWASP API Security Top 10 (2023) with .NET-specific mitigations
> **Prerequisites**: `authentication-jwt.md`, `authorization.md`, `api-protection.md`
> **Related Files**: `../06-error-handling/problem-details.md`, `../09-testing/integration-testing.md`
> **Agent Use Case**: Reference for security audits, code reviews, and compliance requirements

## Quick Context

The OWASP API Security Top 10 identifies the most critical security risks for APIs. This guide provides .NET 8-specific mitigations, code examples, and a comprehensive checklist for securing your APIs against these vulnerabilities.

**OWASP References**:
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP ASVS (Application Security Verification Standard)](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

**Compliance Frameworks**:
- GDPR (General Data Protection Regulation)
- SOC 2 Type II
- PCI DSS (Payment Card Industry Data Security Standard)
- HIPAA (Health Insurance Portability and Accountability Act)

## API1:2023 - Broken Object Level Authorization (BOLA)

### Description

APIs fail to validate that users can only access objects they own. Attackers manipulate object IDs to access unauthorized resources.

### Risk Level: CRITICAL

### .NET Mitigation

```csharp
// Demo.Api/Endpoints/OrderEndpoints.cs
using Microsoft.AspNetCore.Authorization;

namespace Demo.Api.Endpoints;

public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/{id:guid}", GetOrder);
        group.MapPut("/{id:guid}", UpdateOrder);
        group.MapDelete("/{id:guid}", DeleteOrder);

        return group;
    }

    // VULNERABLE CODE (DON'T DO THIS)
    private static async Task<IResult> GetOrderVulnerable(
        Guid id,
        IOrderService orderService,
        CancellationToken ct)
    {
        var order = await orderService.GetByIdAsync(id, ct);
        return order is not null ? Results.Ok(order) : Results.NotFound();
        // L No ownership check! User can access any order by guessing IDs
    }

    // SECURE CODE (DO THIS)
    private static async Task<IResult> GetOrder(
        Guid id,
        ClaimsPrincipal user,
        IOrderService orderService,
        IAuthorizationService authorizationService,
        CancellationToken ct)
    {
        var order = await orderService.GetByIdAsync(id, ct);
        if (order is null)
        {
            return Results.NotFound();
        }

        //  Verify ownership before returning data
        var authResult = await authorizationService.AuthorizeAsync(
            user,
            order,
            "OwnerOrAdmin"
        );

        if (!authResult.Succeeded)
        {
            // Return 404 instead of 403 to prevent information disclosure
            return Results.NotFound();
        }

        return Results.Ok(order);
    }

    private static async Task<IResult> UpdateOrder(
        Guid id,
        UpdateOrderRequest request,
        ClaimsPrincipal user,
        IOrderService orderService,
        IAuthorizationService authorizationService,
        CancellationToken ct)
    {
        var order = await orderService.GetByIdAsync(id, ct);
        if (order is null)
        {
            return Results.NotFound();
        }

        //  Verify ownership before modification
        var authResult = await authorizationService.AuthorizeAsync(
            user,
            order,
            new OwnerRequirement()
        );

        if (!authResult.Succeeded)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status403Forbidden,
                title: "Forbidden",
                detail: "You can only modify your own orders"
            );
        }

        await orderService.UpdateAsync(id, request, ct);
        return Results.NoContent();
    }

    private static async Task<IResult> DeleteOrder(
        Guid id,
        ClaimsPrincipal user,
        IOrderService orderService,
        ILogger<Program> logger,
        CancellationToken ct)
    {
        var userId = user.FindFirst("sub")?.Value;
        var order = await orderService.GetByIdAsync(id, ct);

        if (order is null)
        {
            return Results.NotFound();
        }

        //  Verify ownership
        if (order.UserId != userId && !user.IsInRole("Admin"))
        {
            logger.LogWarning(
                "User {UserId} attempted to delete order {OrderId} owned by {OwnerId}",
                userId,
                id,
                order.UserId
            );
            return Results.NotFound(); // Don't leak existence
        }

        await orderService.DeleteAsync(id, ct);
        return Results.NoContent();
    }
}
```

### Authorization Handler for Ownership

```csharp
// Demo.Api/Authorization/Handlers/OwnerHandler.cs
using Microsoft.AspNetCore.Authorization;

namespace Demo.Api.Authorization.Handlers;

public class OwnerRequirement : IAuthorizationRequirement { }

public class OwnerHandler : AuthorizationHandler<OwnerRequirement, IOwnable>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OwnerRequirement requirement,
        IOwnable resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
        {
            return Task.CompletedTask;
        }

        // Allow owner or admin
        if (resource.UserId == userId || context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Interface for ownable resources
public interface IOwnable
{
    string UserId { get; }
}
```

### Checklist

- [ ] Every data access endpoint validates ownership
- [ ] Use `IAuthorizationService.AuthorizeAsync()` for resource-based auth
- [ ] Return 404 instead of 403 to prevent information disclosure
- [ ] Log unauthorized access attempts
- [ ] Use GUIDs instead of sequential IDs
- [ ] Implement row-level security in database queries
- [ ] Never trust client-provided object IDs without validation

---

## API2:2023 - Broken Authentication

### Description

Authentication mechanisms are improperly implemented, allowing attackers to compromise tokens or exploit implementation flaws.

### Risk Level: CRITICAL

### .NET Mitigation

```csharp
// Program.cs - Secure JWT Configuration
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtSettings = builder.Configuration.GetSection("Jwt");

        options.TokenValidationParameters = new TokenValidationParameters
        {
            //  Validate ALL claims
            ValidateIssuer = true,
            ValidIssuer = jwtSettings["Issuer"],

            ValidateAudience = true,
            ValidAudience = jwtSettings["Audience"],

            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings["SecretKey"]!)
            ),

            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5),

            //  OWASP recommendations
            RequireExpirationTime = true,
            RequireSignedTokens = true
        };

        //  Require HTTPS in production
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();

        //  Log authentication failures
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();

                logger.LogWarning(
                    "Authentication failed: {Reason}. IP: {IP}",
                    context.Exception.Message,
                    context.HttpContext.Connection.RemoteIpAddress
                );

                return Task.CompletedTask;
            },
            OnChallenge = context =>
            {
                context.HandleResponse();
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                context.Response.ContentType = "application/problem+json";

                return context.Response.WriteAsJsonAsync(new ProblemDetails
                {
                    Status = StatusCodes.Status401Unauthorized,
                    Title = "Unauthorized",
                    Detail = "Valid authentication credentials required"
                });
            }
        };
    });

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

### Secure Login Implementation

```csharp
// Demo.Api/Endpoints/AuthEndpoints.cs
public static class AuthEndpoints
{
    public static RouteGroupBuilder MapAuthEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/auth")
            .WithTags("Authentication");

        group.MapPost("/login", LoginAsync)
            .RequireRateLimiting("strictAuth") //  Rate limit authentication
            .AllowAnonymous();

        return group;
    }

    private static async Task<IResult> LoginAsync(
        LoginRequest request,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ITokenService tokenService,
        ILogger<Program> logger,
        HttpContext httpContext,
        CancellationToken ct)
    {
        //  Validate input
        if (string.IsNullOrWhiteSpace(request.Email) ||
            string.IsNullOrWhiteSpace(request.Password))
        {
            return Results.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Invalid request",
                detail: "Email and password are required"
            );
        }

        //  Find user
        var user = await userManager.FindByEmailAsync(request.Email);
        if (user is null)
        {
            //  Log failed login attempt
            logger.LogWarning(
                "Login failed for email {Email} - user not found. IP: {IP}",
                request.Email,
                httpContext.Connection.RemoteIpAddress
            );

            //  Return generic error (don't reveal user existence)
            await Task.Delay(Random.Shared.Next(100, 300)); // Timing attack mitigation
            return Results.Problem(
                statusCode: StatusCodes.Status401Unauthorized,
                title: "Invalid credentials",
                detail: "Email or password is incorrect"
            );
        }

        //  Verify password with lockout
        var result = await signInManager.CheckPasswordSignInAsync(
            user,
            request.Password,
            lockoutOnFailure: true // Enable account lockout
        );

        if (!result.Succeeded)
        {
            if (result.IsLockedOut)
            {
                logger.LogWarning(
                    "User {UserId} is locked out. IP: {IP}",
                    user.Id,
                    httpContext.Connection.RemoteIpAddress
                );

                return Results.Problem(
                    statusCode: StatusCodes.Status423Locked,
                    title: "Account locked",
                    detail: "Account has been locked due to multiple failed login attempts"
                );
            }

            logger.LogWarning(
                "Login failed for user {UserId} - invalid password. IP: {IP}",
                user.Id,
                httpContext.Connection.RemoteIpAddress
            );

            await Task.Delay(Random.Shared.Next(100, 300));
            return Results.Problem(
                statusCode: StatusCodes.Status401Unauthorized,
                title: "Invalid credentials",
                detail: "Email or password is incorrect"
            );
        }

        //  Generate secure tokens
        var roles = await userManager.GetRolesAsync(user);
        var accessToken = tokenService.GenerateAccessToken(user.Id, roles);
        var refreshToken = tokenService.GenerateRefreshToken();

        //  Store refresh token securely
        await StoreRefreshTokenAsync(user.Id, refreshToken, httpContext, ct);

        //  Log successful login
        logger.LogInformation(
            "User {UserId} logged in successfully. IP: {IP}",
            user.Id,
            httpContext.Connection.RemoteIpAddress
        );

        return Results.Ok(new LoginResponse(accessToken, refreshToken, 3600));
    }

    private static Task StoreRefreshTokenAsync(
        string userId,
        string refreshToken,
        HttpContext context,
        CancellationToken ct)
    {
        // Implement secure refresh token storage
        // - Hash the token before storage
        // - Store with expiration timestamp
        // - Associate with user ID and IP address
        return Task.CompletedTask;
    }
}
```

### Password Policy Configuration

```csharp
// Program.cs
builder.Services.Configure<IdentityOptions>(options =>
{
    //  Strong password requirements
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 12; // Minimum 12 characters
    options.Password.RequiredUniqueChars = 4;

    //  Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    //  User settings
    options.User.RequireUniqueEmail = true;

    //  Sign-in settings
    options.SignIn.RequireConfirmedEmail = true;
    options.SignIn.RequireConfirmedAccount = true;
});
```

### Rate Limiting for Auth Endpoints

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("strictAuth", opt =>
    {
        opt.PermitLimit = 5; // 5 login attempts
        opt.Window = TimeSpan.FromMinutes(15); // per 15 minutes
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 0; // No queueing
    });
});
```

### Checklist

- [ ] Use strong JWT validation (all parameters)
- [ ] Require HTTPS for authentication endpoints
- [ ] Implement account lockout (5 attempts in 15 minutes)
- [ ] Enforce strong password policy (12+ chars, complexity)
- [ ] Rate limit authentication endpoints (5 requests/15 min)
- [ ] Log all authentication failures with IP addresses
- [ ] Use timing-attack-resistant comparisons
- [ ] Implement refresh token rotation
- [ ] Require email/phone verification
- [ ] Never expose whether email exists
- [ ] Hash refresh tokens before storage
- [ ] Set appropriate token expiration (15-60 minutes)
- [ ] Implement MFA for sensitive operations

---

## API3:2023 - Broken Object Property Level Authorization

### Description

APIs allow users to modify or access object properties they shouldn't have access to.

### Risk Level: HIGH

### .NET Mitigation

```csharp
// Demo.Api/Contracts/UpdateUserRequest.cs
namespace Demo.Api.Contracts;

// L VULNERABLE: Allows updating admin status
public record UpdateUserRequestVulnerable
{
    public string? Name { get; init; }
    public string? Email { get; init; }
    public bool IsAdmin { get; init; } // User could set themselves as admin!
    public string? Role { get; init; } // User could change their role!
}

//  SECURE: Only allows updating safe properties
public record UpdateUserProfileRequest
{
    public string? Name { get; init; }
    public string? Bio { get; init; }
    public string? AvatarUrl { get; init; }
    // Admin-only properties are NOT included
}

//  Separate DTO for admin operations
public record UpdateUserRoleRequest
{
    public string Role { get; init; } = string.Empty;
}
```

### Endpoint Implementation

```csharp
// Demo.Api/Endpoints/UserEndpoints.cs
public static class UserEndpoints
{
    public static RouteGroupBuilder MapUserEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/users")
            .WithTags("Users")
            .RequireAuthorization();

        //  Regular users can update their own profile (safe properties only)
        group.MapPut("/me", UpdateProfile);

        //  Only admins can update user roles
        group.MapPut("/{id}/role", UpdateUserRole)
            .RequireAuthorization("Admin");

        return group;
    }

    private static async Task<IResult> UpdateProfile(
        UpdateUserProfileRequest request,
        ClaimsPrincipal user,
        IUserService userService,
        CancellationToken ct)
    {
        var userId = user.FindFirst("sub")?.Value;
        if (string.IsNullOrEmpty(userId))
        {
            return Results.Unauthorized();
        }

        //  Only update safe properties
        await userService.UpdateProfileAsync(userId, request, ct);
        return Results.NoContent();
    }

    private static async Task<IResult> UpdateUserRole(
        string id,
        UpdateUserRoleRequest request,
        ClaimsPrincipal user,
        IUserService userService,
        ILogger<Program> logger,
        CancellationToken ct)
    {
        var adminId = user.FindFirst("sub")?.Value;

        //  Log sensitive operation
        logger.LogInformation(
            "Admin {AdminId} changing role for user {UserId} to {Role}",
            adminId,
            id,
            request.Role
        );

        await userService.UpdateRoleAsync(id, request.Role, ct);
        return Results.NoContent();
    }
}
```

### Entity Protection with Mapping

```csharp
// Demo.Application/Services/UserService.cs
public class UserService : IUserService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<UserService> _logger;

    public async Task UpdateProfileAsync(
        string userId,
        UpdateUserProfileRequest request,
        CancellationToken ct)
    {
        var user = await _context.Users.FindAsync(new object[] { userId }, ct);
        if (user is null)
        {
            throw new NotFoundException(nameof(User), userId);
        }

        //  Explicitly map only safe properties
        user.Name = request.Name ?? user.Name;
        user.Bio = request.Bio ?? user.Bio;
        user.AvatarUrl = request.AvatarUrl ?? user.AvatarUrl;

        // L Never do this: AutoMapper with ReverseMap on update operations
        // _mapper.Map(request, user); // Could map unexpected properties!

        await _context.SaveChangesAsync(ct);
    }

    public async Task UpdateRoleAsync(string userId, string role, CancellationToken ct)
    {
        var user = await _context.Users.FindAsync(new object[] { userId }, ct);
        if (user is null)
        {
            throw new NotFoundException(nameof(User), userId);
        }

        //  Validate role before assignment
        var validRoles = new[] { "User", "Manager", "Admin" };
        if (!validRoles.Contains(role))
        {
            throw new ValidationException("Invalid role");
        }

        user.Role = role;
        await _context.SaveChangesAsync(ct);
    }
}
```

### Response Filtering

```csharp
// Demo.Api/Contracts/UserResponse.cs
public record UserResponse
{
    public required string Id { get; init; }
    public required string Name { get; init; }
    public required string Email { get; init; }
    public string? Bio { get; init; }

    //  Sensitive fields are excluded
    // - PasswordHash
    // - RefreshTokens
    // - SecurityStamp
}

// Admin-only response with additional fields
public record AdminUserResponse : UserResponse
{
    public required string Role { get; init; }
    public required bool IsActive { get; init; }
    public required DateTime CreatedAt { get; init; }
    public DateTime? LastLoginAt { get; init; }
}
```

### Checklist

- [ ] Use separate DTOs for input and output
- [ ] Create role-specific DTOs (user vs admin)
- [ ] Never use entity models directly as API responses
- [ ] Explicitly map properties (avoid AutoMapper ReverseMap)
- [ ] Validate property access based on user role
- [ ] Filter response properties based on authorization
- [ ] Log sensitive property modifications
- [ ] Implement field-level authorization
- [ ] Use immutable records for DTOs
- [ ] Never expose internal IDs or sensitive metadata

---

## API4:2023 - Unrestricted Resource Consumption

### Description

APIs lack protection against excessive resource consumption (rate limiting, pagination, payload size).

### Risk Level: HIGH

### .NET Mitigation

```csharp
// Program.cs - Comprehensive Resource Protection
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

//  1. Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    // Public endpoints (strict)
    options.AddFixedWindowLimiter("public", opt =>
    {
        opt.PermitLimit = 20;
        opt.Window = TimeSpan.FromMinutes(1);
    });

    // Authenticated users (generous)
    options.AddSlidingWindowLimiter("authenticated", opt =>
    {
        opt.PermitLimit = 200;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6;
    });

    // Expensive operations (very strict)
    options.AddConcurrencyLimiter("expensive", opt =>
    {
        opt.PermitLimit = 2; // Max 2 concurrent requests
        opt.QueueLimit = 5;
    });
});

//  2. Request Size Limits
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024; // 10 MB
    options.ValueCountLimit = 100; // Max form values
    options.KeyLengthLimit = 2048; // Max key length
});

builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
    options.Limits.MaxRequestLineSize = 8192; // 8 KB
    options.Limits.MaxRequestHeadersTotalSize = 32768; // 32 KB
    options.Limits.MaxRequestHeaderCount = 100;

    //  3. Connection Limits
    options.Limits.MaxConcurrentConnections = 1000;
    options.Limits.MaxConcurrentUpgradedConnections = 100;

    //  4. Timeouts
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
});

var app = builder.Build();

app.UseRateLimiter();

app.Run();
```

### Pagination Implementation

```csharp
// Demo.Application/Common/PaginatedList.cs
namespace Demo.Application.Common;

public class PaginatedList<T>
{
    public List<T> Items { get; }
    public int PageNumber { get; }
    public int PageSize { get; }
    public int TotalCount { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    public PaginatedList(List<T> items, int count, int pageNumber, int pageSize)
    {
        Items = items;
        TotalCount = count;
        PageNumber = pageNumber;
        PageSize = pageSize;
    }

    public static async Task<PaginatedList<T>> CreateAsync(
        IQueryable<T> source,
        int pageNumber,
        int pageSize,
        CancellationToken ct)
    {
        //  Enforce maximum page size
        const int maxPageSize = 100;
        pageSize = Math.Min(pageSize, maxPageSize);

        //  Ensure minimum page number
        pageNumber = Math.Max(pageNumber, 1);

        var count = await source.CountAsync(ct);
        var items = await source
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PaginatedList<T>(items, count, pageNumber, pageSize);
    }
}
```

### Paginated Endpoint

```csharp
// Demo.Api/Endpoints/ProductEndpoints.cs
public static class ProductEndpoints
{
    public static RouteGroupBuilder MapProductEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/products")
            .WithTags("Products");

        //  Always paginate list endpoints
        group.MapGet("/", GetProducts)
            .RequireRateLimiting("authenticated");

        return group;
    }

    private static async Task<IResult> GetProducts(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        IProductService productService,
        CancellationToken ct)
    {
        //  Enforce limits
        const int maxPageSize = 100;
        pageSize = Math.Min(pageSize, maxPageSize);
        page = Math.Max(page, 1);

        var result = await productService.GetPaginatedAsync(page, pageSize, ct);

        //  Add pagination headers
        var metadata = new
        {
            result.TotalCount,
            result.PageSize,
            result.PageNumber,
            result.TotalPages
        };

        return Results.Ok(new
        {
            data = result.Items,
            pagination = metadata
        });
    }
}
```

### Query Complexity Limits

```csharp
// Demo.Api/Filters/QueryComplexityFilter.cs
using Microsoft.AspNetCore.Http.HttpResults;

namespace Demo.Api.Filters;

public class QueryComplexityFilter : IEndpointFilter
{
    private readonly int _maxFilters;

    public QueryComplexityFilter(int maxFilters = 10)
    {
        _maxFilters = maxFilters;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var query = context.HttpContext.Request.Query;

        //  Limit number of query parameters
        if (query.Count > _maxFilters)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Too many filters",
                detail: $"Maximum {_maxFilters} query parameters allowed"
            );
        }

        //  Validate query parameter values
        foreach (var (key, value) in query)
        {
            if (value.ToString().Length > 1000)
            {
                return Results.Problem(
                    statusCode: StatusCodes.Status400BadRequest,
                    title: "Query parameter too long",
                    detail: $"Parameter '{key}' exceeds maximum length"
                );
            }
        }

        return await next(context);
    }
}
```

### Timeout Protection

```csharp
// Demo.Api/Middleware/TimeoutMiddleware.cs
public class TimeoutMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TimeSpan _timeout;

    public TimeoutMiddleware(RequestDelegate next, TimeSpan timeout)
    {
        _next = next;
        _timeout = timeout;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        using var cts = new CancellationTokenSource(_timeout);
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            cts.Token,
            context.RequestAborted
        );

        var originalToken = context.RequestAborted;
        context.RequestAborted = linkedCts.Token;

        try
        {
            await _next(context);
        }
        catch (OperationCanceledException) when (cts.IsCancellationRequested)
        {
            context.Response.StatusCode = StatusCodes.Status408RequestTimeout;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Status = StatusCodes.Status408RequestTimeout,
                Title = "Request Timeout",
                Detail = $"Request exceeded timeout of {_timeout.TotalSeconds} seconds"
            });
        }
        finally
        {
            context.RequestAborted = originalToken;
        }
    }
}
```

### Checklist

- [ ] Implement rate limiting on all endpoints
- [ ] Set maximum page size (100 recommended)
- [ ] Always paginate list endpoints
- [ ] Limit request body size (10 MB default)
- [ ] Set request timeout (30 seconds recommended)
- [ ] Limit query parameter count and length
- [ ] Implement connection limits
- [ ] Use streaming for large responses
- [ ] Add circuit breakers for external dependencies
- [ ] Monitor resource consumption
- [ ] Implement database query timeouts
- [ ] Use async/await to prevent thread pool starvation
- [ ] Limit concurrent uploads
- [ ] Implement exponential backoff for retries

---

## API5:2023 - Broken Function Level Authorization

### Description

APIs fail to validate user permissions for sensitive functions and admin endpoints.

### Risk Level: CRITICAL

### .NET Mitigation

```csharp
// Program.cs - Function-Level Authorization
builder.Services.AddAuthorization(options =>
{
    //  Define explicit policies for functions
    options.AddPolicy("Admin", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("Manager", policy =>
        policy.RequireRole("Manager", "Admin"));

    options.AddPolicy("DeleteUser", policy =>
    {
        policy.RequireRole("Admin");
        policy.RequireClaim("permission", "users:delete");
    });

    options.AddPolicy("ViewAuditLogs", policy =>
    {
        policy.RequireRole("Admin", "Auditor");
        policy.RequireClaim("department", "Security", "Compliance");
    });

    //  Require authentication by default
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

### Secure Admin Endpoints

```csharp
// Demo.Api/Endpoints/AdminEndpoints.cs
public static class AdminEndpoints
{
    public static RouteGroupBuilder MapAdminEndpoints(this IEndpointRouteBuilder routes)
    {
        //  Apply authorization to entire group
        var admin = routes.MapGroup("/admin")
            .WithTags("Administration")
            .RequireAuthorization("Admin"); // All endpoints require Admin role

        admin.MapGet("/users", GetAllUsers);
        admin.MapDelete("/users/{id}", DeleteUser)
            .RequireAuthorization("DeleteUser"); // Additional permission required

        admin.MapGet("/audit-logs", GetAuditLogs)
            .RequireAuthorization("ViewAuditLogs");

        //  Dangerous operations require extra validation
        admin.MapPost("/system/reset", ResetSystem)
            .RequireAuthorization("SuperAdmin");

        return admin;
    }

    private static async Task<IResult> DeleteUser(
        string id,
        ClaimsPrincipal user,
        IUserService userService,
        ILogger<Program> logger,
        CancellationToken ct)
    {
        var adminId = user.FindFirst("sub")?.Value;
        var adminEmail = user.FindFirst("email")?.Value;

        //  Prevent self-deletion
        if (id == adminId)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Invalid operation",
                detail: "Cannot delete your own account"
            );
        }

        //  Log sensitive operation
        logger.LogWarning(
            "Admin {AdminId} ({Email}) is deleting user {UserId}",
            adminId,
            adminEmail,
            id
        );

        await userService.DeleteAsync(id, ct);

        //  Audit trail
        await AuditAsync("User Deleted", new { DeletedUserId = id, AdminId = adminId });

        return Results.NoContent();
    }

    private static Task AuditAsync(string action, object details)
    {
        // Implement audit logging
        return Task.CompletedTask;
    }
}
```

### Prevent Privilege Escalation

```csharp
// Demo.Api/Endpoints/RoleEndpoints.cs
public static class RoleEndpoints
{
    public static RouteGroupBuilder MapRoleEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/roles")
            .WithTags("Roles")
            .RequireAuthorization("Admin");

        group.MapPut("/users/{userId}/role", AssignRole);

        return group;
    }

    private static async Task<IResult> AssignRole(
        string userId,
        AssignRoleRequest request,
        ClaimsPrincipal user,
        IUserService userService,
        ILogger<Program> logger,
        CancellationToken ct)
    {
        var adminRole = user.FindFirst(ClaimTypes.Role)?.Value;

        //  Prevent privilege escalation
        // Admins cannot assign SuperAdmin role
        if (request.Role == "SuperAdmin" && adminRole != "SuperAdmin")
        {
            logger.LogWarning(
                "Admin {AdminId} attempted to assign SuperAdmin role to user {UserId}",
                user.FindFirst("sub")?.Value,
                userId
            );

            return Results.Problem(
                statusCode: StatusCodes.Status403Forbidden,
                title: "Forbidden",
                detail: "You cannot assign this role"
            );
        }

        //  Validate role exists
        var validRoles = new[] { "User", "Manager", "Admin", "SuperAdmin" };
        if (!validRoles.Contains(request.Role))
        {
            return Results.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Invalid role",
                detail: $"Role '{request.Role}' is not valid"
            );
        }

        await userService.AssignRoleAsync(userId, request.Role, ct);

        logger.LogInformation(
            "Admin {AdminId} assigned role {Role} to user {UserId}",
            user.FindFirst("sub")?.Value,
            request.Role,
            userId
        );

        return Results.NoContent();
    }
}
```

### Checklist

- [ ] All admin endpoints require explicit authorization
- [ ] Use role-based or policy-based authorization
- [ ] Implement principle of least privilege
- [ ] Log all sensitive administrative actions
- [ ] Prevent privilege escalation (admins can't create super admins)
- [ ] Require additional validation for dangerous operations
- [ ] Use fallback policy to require authentication by default
- [ ] Separate admin and user endpoints
- [ ] Implement approval workflows for critical functions
- [ ] Regularly audit admin permissions

---

## API6:2023 - Unrestricted Access to Sensitive Business Flows

### Description

APIs expose business workflows without proper flow validation or rate limiting.

### Risk Level: MEDIUM

### .NET Mitigation

```csharp
// Demo.Application/Services/OrderService.cs
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<OrderService> _logger;

    public async Task<OrderResponse> CreateOrderAsync(
        string userId,
        CreateOrderRequest request,
        CancellationToken ct)
    {
        //  1. Validate business prerequisites
        var user = await _context.Users.FindAsync(new object[] { userId }, ct);
        if (user?.IsActive != true)
        {
            throw new ValidationException("Account is not active");
        }

        //  2. Check for duplicate/concurrent orders
        var hasRecentOrder = await _context.Orders
            .AnyAsync(o =>
                o.UserId == userId &&
                o.CreatedAt > DateTime.UtcNow.AddMinutes(-5), ct);

        if (hasRecentOrder)
        {
            _logger.LogWarning(
                "User {UserId} attempted to create multiple orders within 5 minutes",
                userId
            );
            throw new ValidationException("Please wait before placing another order");
        }

        //  3. Validate order limits
        var orderTotal = request.Items.Sum(i => i.Quantity * i.Price);
        const decimal maxOrderAmount = 10000m;
        if (orderTotal > maxOrderAmount)
        {
            throw new ValidationException($"Order amount exceeds maximum of {maxOrderAmount}");
        }

        //  4. Check inventory availability
        foreach (var item in request.Items)
        {
            var product = await _context.Products.FindAsync(new object[] { item.ProductId }, ct);
            if (product is null || product.Stock < item.Quantity)
            {
                throw new ValidationException($"Insufficient stock for product {item.ProductId}");
            }
        }

        //  5. Create order with proper state
        var order = Order.Create(userId, request.Items);

        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Order {OrderId} created for user {UserId} with total {Total}",
            order.Id,
            userId,
            orderTotal
        );

        return MapToResponse(order);
    }

    public async Task<OrderResponse> CancelOrderAsync(
        Guid orderId,
        string userId,
        CancellationToken ct)
    {
        var order = await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == orderId, ct);

        if (order is null)
        {
            throw new NotFoundException(nameof(Order), orderId);
        }

        //  Verify ownership
        if (order.UserId != userId)
        {
            throw new UnauthorizedAccessException("You can only cancel your own orders");
        }

        //  Validate business rules
        if (order.Status == OrderStatus.Shipped)
        {
            throw new ValidationException("Cannot cancel shipped orders");
        }

        if (order.Status == OrderStatus.Cancelled)
        {
            throw new ValidationException("Order is already cancelled");
        }

        //  Check cancellation window
        var cancellationDeadline = order.CreatedAt.AddHours(24);
        if (DateTime.UtcNow > cancellationDeadline)
        {
            throw new ValidationException("Cancellation period has expired");
        }

        //  Apply business logic
        order.Cancel();
        await _context.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Order {OrderId} cancelled by user {UserId}",
            orderId,
            userId
        );

        return MapToResponse(order);
    }
}
```

### State Machine for Order Flow

```csharp
// Demo.Domain/Entities/Order.cs
public class Order
{
    public Guid Id { get; private set; }
    public string UserId { get; private set; } = string.Empty;
    public OrderStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CancelledAt { get; private set; }

    private Order() { } // EF Core

    public static Order Create(string userId, List<OrderItem> items)
    {
        return new Order
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
    }

    //  Business logic encapsulated in entity
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
        {
            throw new InvalidOperationException($"Cannot confirm order in status {Status}");
        }

        Status = OrderStatus.Confirmed;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
        {
            throw new InvalidOperationException($"Cannot ship order in status {Status}");
        }

        Status = OrderStatus.Shipped;
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
        {
            throw new InvalidOperationException("Cannot cancel shipped order");
        }

        if (Status == OrderStatus.Cancelled)
        {
            throw new InvalidOperationException("Order is already cancelled");
        }

        Status = OrderStatus.Cancelled;
        CancelledAt = DateTime.UtcNow;
    }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}
```

### Checklist

- [ ] Implement state machines for business workflows
- [ ] Validate all business prerequisites before operations
- [ ] Prevent concurrent/duplicate operations
- [ ] Enforce business rules at domain level
- [ ] Check time windows for sensitive operations
- [ ] Validate quantity/amount limits
- [ ] Log suspicious business flow violations
- [ ] Rate limit business-critical endpoints
- [ ] Implement idempotency for critical operations
- [ ] Add approval workflows for high-value transactions

---

## API7:2023 - Server Side Request Forgery (SSRF)

### Description

APIs fetch remote resources without validating the user-supplied URL.

### Risk Level: HIGH

### .NET Mitigation

```csharp
// Demo.Api/Services/SafeHttpClient.cs
using System.Net;

namespace Demo.Api.Services;

public interface ISafeHttpClient
{
    Task<string> GetAsync(string url, CancellationToken ct);
}

public class SafeHttpClient : ISafeHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<SafeHttpClient> _logger;

    //  Allowed domains whitelist
    private static readonly HashSet<string> AllowedHosts = new()
    {
        "api.example.com",
        "cdn.example.com",
        "images.example.com"
    };

    //  Blocked IP ranges (internal networks)
    private static readonly string[] BlockedIPRanges = new[]
    {
        "127.0.0.0/8",     // Loopback
        "10.0.0.0/8",      // Private
        "172.16.0.0/12",   // Private
        "192.168.0.0/16",  // Private
        "169.254.0.0/16",  // Link-local
        "::1/128",         // IPv6 loopback
        "fc00::/7"         // IPv6 private
    };

    public SafeHttpClient(HttpClient httpClient, ILogger<SafeHttpClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;

        //  Configure HTTP client security
        _httpClient.Timeout = TimeSpan.FromSeconds(10);
        _httpClient.MaxResponseContentBufferSize = 1 * 1024 * 1024; // 1 MB
    }

    public async Task<string> GetAsync(string url, CancellationToken ct)
    {
        //  1. Validate URL format
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
        {
            _logger.LogWarning("Invalid URL format: {Url}", url);
            throw new ValidationException("Invalid URL format");
        }

        //  2. Only allow HTTPS
        if (uri.Scheme != Uri.UriSchemeHttps)
        {
            _logger.LogWarning("Non-HTTPS URL blocked: {Url}", url);
            throw new ValidationException("Only HTTPS URLs are allowed");
        }

        //  3. Validate against whitelist
        if (!AllowedHosts.Contains(uri.Host))
        {
            _logger.LogWarning("URL host not in whitelist: {Host}", uri.Host);
            throw new ValidationException("URL host is not allowed");
        }

        //  4. Resolve DNS and check IP
        var addresses = await Dns.GetHostAddressesAsync(uri.Host, ct);
        foreach (var address in addresses)
        {
            if (IsBlockedIPAddress(address))
            {
                _logger.LogWarning(
                    "Blocked IP address detected: {IP} for host {Host}",
                    address,
                    uri.Host
                );
                throw new ValidationException("Cannot access internal IP addresses");
            }
        }

        //  5. Make request with timeout
        try
        {
            var response = await _httpClient.GetAsync(uri, ct);
            response.EnsureSuccessStatusCode();

            return await response.Content.ReadAsStringAsync(ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP request failed for URL: {Url}", url);
            throw new InvalidOperationException("Failed to fetch resource");
        }
        catch (TaskCanceledException)
        {
            _logger.LogWarning("HTTP request timeout for URL: {Url}", url);
            throw new InvalidOperationException("Request timed out");
        }
    }

    private static bool IsBlockedIPAddress(IPAddress address)
    {
        // Check if IP is in blocked ranges
        return address.IsIPv4MappedToIPv6 ||
               IsPrivateNetwork(address) ||
               IsLoopback(address) ||
               IsLinkLocal(address);
    }

    private static bool IsPrivateNetwork(IPAddress address)
    {
        var bytes = address.GetAddressBytes();

        if (address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
        {
            // 10.0.0.0/8
            if (bytes[0] == 10) return true;

            // 172.16.0.0/12
            if (bytes[0] == 172 && bytes[1] >= 16 && bytes[1] <= 31) return true;

            // 192.168.0.0/16
            if (bytes[0] == 192 && bytes[1] == 168) return true;
        }

        return false;
    }

    private static bool IsLoopback(IPAddress address) =>
        IPAddress.IsLoopback(address);

    private static bool IsLinkLocal(IPAddress address)
    {
        if (address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
        {
            var bytes = address.GetAddressBytes();
            return bytes[0] == 169 && bytes[1] == 254; // 169.254.0.0/16
        }

        return false;
    }
}
```

### Endpoint with SSRF Protection

```csharp
// Demo.Api/Endpoints/WebhookEndpoints.cs
public static class WebhookEndpoints
{
    public static RouteGroupBuilder MapWebhookEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/webhooks")
            .WithTags("Webhooks")
            .RequireAuthorization("Admin");

        group.MapPost("/validate", ValidateWebhook);

        return group;
    }

    private static async Task<IResult> ValidateWebhook(
        ValidateWebhookRequest request,
        ISafeHttpClient httpClient,
        CancellationToken ct)
    {
        try
        {
            //  Use safe HTTP client with SSRF protection
            var response = await httpClient.GetAsync(request.Url, ct);

            return Results.Ok(new { success = true, response });
        }
        catch (ValidationException ex)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Invalid URL",
                detail: ex.Message
            );
        }
    }
}
```

### Checklist

- [ ] Use whitelist of allowed domains
- [ ] Block internal IP ranges (RFC 1918)
- [ ] Only allow HTTPS URLs
- [ ] Resolve DNS before making requests
- [ ] Check resolved IP against blacklist
- [ ] Set request timeouts (10 seconds max)
- [ ] Limit response size (1 MB recommended)
- [ ] Validate URL format
- [ ] Log SSRF attempts
- [ ] Use dedicated HTTP client with security defaults
- [ ] Never trust user-provided URLs
- [ ] Implement network segmentation

---

## API8:2023 - Security Misconfiguration

### Description

APIs have insecure default configurations, missing security headers, or overly permissive settings.

### Risk Level: HIGH

### .NET Mitigation

See the complete security configuration in `api-protection.md`. Key points:

```csharp
// Program.cs - Secure Configuration
var builder = WebApplication.CreateBuilder(args);

//  1. Remove unnecessary headers
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false; // Remove "Server" header
});

//  2. Configure security headers
app.UseSecurityHeaders(); // See api-protection.md

//  3. Disable unnecessary features
builder.Services.Configure<RouteOptions>(options =>
{
    options.LowercaseUrls = true;
    options.LowercaseQueryStrings = false;
    options.AppendTrailingSlash = false;
});

//  4. Configure error handling
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler(); // Don't expose stack traces
}

//  5. Disable directory browsing (enabled by default in some scenarios)
// Don't use: app.UseDirectoryBrowser();

//  6. Configure Swagger only for development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    // Disable Swagger in production
}

//  7. Configure CORS properly (see api-protection.md)
app.UseCors("Production"); // Never use AllowAnyOrigin in production

app.Run();
```

### Checklist

- [ ] Remove server headers
- [ ] Disable directory browsing
- [ ] Configure security headers (CSP, HSTS, X-Frame-Options)
- [ ] Disable Swagger/OpenAPI in production
- [ ] Use secure default configurations
- [ ] Enable HTTPS enforcement
- [ ] Configure CORS restrictively
- [ ] Don't expose stack traces in production
- [ ] Remove unnecessary middleware
- [ ] Disable unused HTTP methods
- [ ] Set secure cookie flags
- [ ] Configure TLS 1.2+ only
- [ ] Regular dependency updates
- [ ] Remove commented-out code

---

## API9:2023 - Improper Inventory Management

### Description

Lack of documentation, outdated documentation, or exposed debugging endpoints.

### Risk Level: MEDIUM

### .NET Mitigation

```csharp
// Program.cs - API Documentation
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "Production API",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "api@example.com"
        }
    });

    //  Document security schemes
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        Description = "JWT Authorization header using the Bearer scheme"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
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

    //  Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

//  Environment-specific endpoints
if (app.Environment.IsDevelopment())
{
    app.MapGet("/debug/config", (IConfiguration config) =>
    {
        // Only in development!
        return Results.Ok(config.AsEnumerable());
    });
}

// L NEVER expose in production:
// app.MapGet("/debug/env", () => Environment.GetEnvironmentVariables());
// app.MapGet("/debug/secrets", () => /* secrets */);
```

### API Versioning

```bash
dotnet add package Asp.Versioning.Http
```

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.ReportApiVersions = true;
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});

// Version 1 endpoints
var v1 = app.MapGroup("/api/v1")
    .HasApiVersion(1);

v1.MapGet("/products", GetProductsV1);

// Version 2 endpoints
var v2 = app.MapGroup("/api/v2")
    .HasApiVersion(2);

v2.MapGet("/products", GetProductsV2);
```

### Checklist

- [ ] Document all endpoints with OpenAPI/Swagger
- [ ] Version your API
- [ ] Disable Swagger in production (or require authentication)
- [ ] Maintain API changelog
- [ ] Document authentication requirements
- [ ] Remove debug endpoints in production
- [ ] Keep dependencies up to date
- [ ] Document deprecated endpoints
- [ ] Use API gateway for inventory
- [ ] Regular security audits

---

## API10:2023 - Unsafe Consumption of APIs

### Description

APIs interact with third-party services without proper validation or security.

### Risk Level: MEDIUM

### .NET Mitigation

```csharp
// Demo.Infrastructure/Services/ExternalApiClient.cs
public class ExternalApiClient : IExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiClient> _logger;

    public ExternalApiClient(HttpClient httpClient, ILogger<ExternalApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;

        //  Configure timeout
        _httpClient.Timeout = TimeSpan.FromSeconds(30);
    }

    public async Task<ExternalApiResponse> GetDataAsync(string endpoint, CancellationToken ct)
    {
        try
        {
            //  1. Validate endpoint
            if (!IsValidEndpoint(endpoint))
            {
                throw new ArgumentException("Invalid endpoint", nameof(endpoint));
            }

            //  2. Add authentication
            _httpClient.DefaultRequestHeaders.Add("X-API-Key", GetApiKey());

            //  3. Make request with timeout
            var response = await _httpClient.GetAsync(endpoint, ct);

            //  4. Validate response status
            if (!response.IsSuccessStatusCode)
            {
                _logger.LogWarning(
                    "External API returned {StatusCode} for endpoint {Endpoint}",
                    response.StatusCode,
                    endpoint
                );
                throw new HttpRequestException($"API returned {response.StatusCode}");
            }

            //  5. Validate content type
            var contentType = response.Content.Headers.ContentType?.MediaType;
            if (contentType != "application/json")
            {
                throw new InvalidOperationException($"Unexpected content type: {contentType}");
            }

            //  6. Limit response size
            var content = await response.Content.ReadAsStringAsync(ct);
            if (content.Length > 1_000_000) // 1 MB
            {
                throw new InvalidOperationException("Response too large");
            }

            //  7. Validate JSON structure
            var data = JsonSerializer.Deserialize<ExternalApiResponse>(
                content,
                new JsonSerializerOptions
                {
                    PropertyNameCaseInsensitive = true,
                    MaxDepth = 10 // Prevent deeply nested JSON attacks
                }
            );

            if (data is null)
            {
                throw new InvalidOperationException("Failed to deserialize response");
            }

            //  8. Validate business rules
            if (string.IsNullOrEmpty(data.Id))
            {
                throw new ValidationException("Response missing required ID field");
            }

            return data;
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "External API request failed");
            throw;
        }
        catch (TaskCanceledException)
        {
            _logger.LogWarning("External API request timed out");
            throw new TimeoutException("External API request timed out");
        }
    }

    private static bool IsValidEndpoint(string endpoint)
    {
        // Validate against whitelist
        var allowedEndpoints = new[] { "/api/users", "/api/products" };
        return allowedEndpoints.Any(e => endpoint.StartsWith(e));
    }

    private static string GetApiKey()
    {
        // Retrieve from secure configuration
        return "your-api-key";
    }
}
```

### Resilience with Polly

```bash
dotnet add package Microsoft.Extensions.Http.Polly
```

```csharp
// Program.cs
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .ConfigureHttpClient(client =>
    {
        client.BaseAddress = new Uri("https://api.external.com");
        client.Timeout = TimeSpan.FromSeconds(30);
    })
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            3,
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                // Log retry
            }
        );
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)
        );
}
```

### Checklist

- [ ] Validate all external API responses
- [ ] Use TLS/SSL for all external communications
- [ ] Implement timeout for external calls (30s max)
- [ ] Validate response content type
- [ ] Limit response size
- [ ] Validate JSON structure and depth
- [ ] Use circuit breakers for resilience
- [ ] Implement retry with exponential backoff
- [ ] Validate SSL certificates
- [ ] Whitelist allowed endpoints
- [ ] Sanitize external data before storage
- [ ] Log external API errors
- [ ] Monitor external API health

---

## Compliance Requirements

### GDPR (General Data Protection Regulation)

```csharp
// Demo.Application/Services/DataProtectionService.cs
public class DataProtectionService : IDataProtectionService
{
    //  Right to be forgotten (Article 17)
    public async Task DeleteUserDataAsync(string userId, CancellationToken ct)
    {
        // Anonymize or delete all user data
        await AnonymizePersonalDataAsync(userId, ct);
        await DeleteUserAccountAsync(userId, ct);
    }

    //  Data portability (Article 20)
    public async Task<UserDataExport> ExportUserDataAsync(string userId, CancellationToken ct)
    {
        // Export all user data in machine-readable format
        return new UserDataExport
        {
            Profile = await GetProfileAsync(userId, ct),
            Orders = await GetOrdersAsync(userId, ct),
            ActivityLog = await GetActivityLogAsync(userId, ct)
        };
    }

    //  Consent management (Article 7)
    public async Task RecordConsentAsync(string userId, string purpose, CancellationToken ct)
    {
        // Record user consent with timestamp and purpose
    }
}
```

### SOC 2 Compliance

- [ ] Implement audit logging
- [ ] Encrypt data at rest and in transit
- [ ] Access control and authentication
- [ ] Incident response plan
- [ ] Regular security testing
- [ ] Data backup and recovery
- [ ] Change management process
- [ ] Vendor risk management

### PCI DSS (Payment Card Industry)

```csharp
// L NEVER store sensitive cardholder data
// - Full magnetic stripe data
// - Card verification code (CVV2)
// - PIN/PIN block

//  Tokenize payment data
public class PaymentService : IPaymentService
{
    public async Task<PaymentResult> ProcessPaymentAsync(
        PaymentRequest request,
        CancellationToken ct)
    {
        // Use payment gateway tokenization
        var token = await _paymentGateway.TokenizeCardAsync(request.Card, ct);

        // Store only token, never actual card data
        await _repository.SavePaymentTokenAsync(token, ct);

        return new PaymentResult { Success = true };
    }
}
```

---

## Penetration Testing Guidance

### Tools

1. **OWASP ZAP** - Web application security scanner
2. **Burp Suite** - Web vulnerability scanner
3. **Postman** - API testing and security testing
4. **dotnet-counters** - Performance monitoring
5. **dotnet-trace** - Diagnostics tracing

### Test Scenarios

```markdown
## Authentication Tests
- [ ] Test JWT token expiration
- [ ] Test invalid tokens
- [ ] Test token tampering
- [ ] Test account lockout
- [ ] Test password complexity
- [ ] Test session timeout

## Authorization Tests
- [ ] Test vertical privilege escalation
- [ ] Test horizontal privilege escalation
- [ ] Test IDOR (Insecure Direct Object References)
- [ ] Test missing function level access control

## Input Validation Tests
- [ ] SQL injection (parameterized queries)
- [ ] XSS (Cross-Site Scripting)
- [ ] Command injection
- [ ] Path traversal
- [ ] XXE (XML External Entity)

## Business Logic Tests
- [ ] Test race conditions
- [ ] Test workflow bypass
- [ ] Test amount manipulation
- [ ] Test negative numbers
- [ ] Test boundary values

## Rate Limiting Tests
- [ ] Test concurrent requests
- [ ] Test burst traffic
- [ ] Test distributed attacks

## Error Handling Tests
- [ ] Test information disclosure
- [ ] Test stack trace exposure
- [ ] Test error messages
```

### Automated Security Testing

```csharp
// Demo.Tests/Security/SecurityTests.cs
public class SecurityTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Endpoints_WithoutAuth_Return401()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/secure");
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }

    [Fact]
    public async Task RateLimiting_ExceedLimit_Returns429()
    {
        var client = _factory.CreateClient();

        for (int i = 0; i < 100; i++)
        {
            await client.GetAsync("/api/public");
        }

        var response = await client.GetAsync("/api/public");
        Assert.Equal((HttpStatusCode)429, response.StatusCode);
    }

    [Fact]
    public async Task Headers_IncludeSecurityHeaders()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/");

        Assert.True(response.Headers.Contains("X-Content-Type-Options"));
        Assert.True(response.Headers.Contains("X-Frame-Options"));
    }
}
```

---

## Security Audit Checklist

### Pre-Deployment

- [ ] All secrets moved to secure storage (Azure Key Vault)
- [ ] Swagger/OpenAPI disabled or authenticated in production
- [ ] Error messages don't expose sensitive information
- [ ] HTTPS enforced everywhere
- [ ] Security headers configured
- [ ] Rate limiting enabled on all public endpoints
- [ ] All inputs validated
- [ ] SQL injection prevention verified
- [ ] XSS prevention verified
- [ ] CSRF tokens implemented (if using cookies)
- [ ] Dependencies updated to latest secure versions
- [ ] Authentication requires strong passwords
- [ ] Authorization implemented on all endpoints
- [ ] Audit logging enabled for sensitive operations

### Post-Deployment

- [ ] Penetration testing completed
- [ ] Security headers verified in production
- [ ] SSL/TLS configuration verified (A+ rating)
- [ ] Monitoring and alerting configured
- [ ] Incident response plan documented
- [ ] Regular security reviews scheduled
- [ ] Dependency vulnerability scanning automated
- [ ] Log retention policy implemented
- [ ] Backup and recovery tested

---

## Navigation

- **Previous**: `api-protection.md`
- **Related**: `authentication-jwt.md`, `authorization.md`
- **Testing**: `../09-testing/integration-testing.md`

## References

- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Microsoft Security Best Practices](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/secure-net-microservices-web-applications/)
- [GDPR Compliance](https://gdpr.eu/)
- [PCI DSS Requirements](https://www.pcisecuritystandards.org/)
- [SOC 2 Compliance](https://www.aicpa.org/soc)
