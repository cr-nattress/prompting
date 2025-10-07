# Authorization in .NET 8

> **File Purpose**: Implement comprehensive authorization strategies including role-based, policy-based, and resource-based authorization
> **Prerequisites**: `authentication-jwt.md` - JWT authentication setup
> **Related Files**: `api-protection.md`, `../06-error-handling/problem-details.md`
> **Agent Use Case**: Reference when implementing granular access control and authorization policies

## Quick Context

Authorization determines what authenticated users can access. .NET 8 provides multiple authorization strategies: simple role-based, flexible policy-based, and granular resource-based authorization. This guide covers all approaches with OWASP security best practices.

**OWASP References**:
- [OWASP API Security Top 10 - API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP API Security Top 10 - API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)

**Microsoft References**:
- [Authorization in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/)
- [Policy-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)

## Role-Based Authorization (Simple)

### Basic Role Checks

```csharp
// Program.cs
using Microsoft.AspNetCore.Authorization;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(/* JWT config */);
builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Single role requirement
app.MapGet("/admin/dashboard", () => "Admin Dashboard")
    .RequireAuthorization(new AuthorizeAttribute { Roles = "Admin" });

// Multiple roles (OR logic)
app.MapGet("/admin/users", () => "User Management")
    .RequireAuthorization(new AuthorizeAttribute { Roles = "Admin,SuperAdmin" });

// Shorthand syntax
app.MapGet("/admin/settings", () => "Settings")
    .RequireAuthorization("Admin"); // Policy name = role name

app.Run();
```

### Controller-Based Authorization

```csharp
// Demo.Api/Controllers/AdminController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Demo.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize(Roles = "Admin")] // Apply to entire controller
public class AdminController : ControllerBase
{
    [HttpGet("users")]
    public IActionResult GetUsers()
    {
        return Ok(new[] { "User1", "User2" });
    }

    [HttpGet("settings")]
    [Authorize(Roles = "SuperAdmin")] // Override for specific action
    public IActionResult GetSettings()
    {
        return Ok(new { Setting = "Value" });
    }

    [AllowAnonymous] // Allow anonymous access to specific action
    [HttpGet("public")]
    public IActionResult GetPublicInfo()
    {
        return Ok("Public information");
    }
}
```

### Minimal API with Role Authorization

```csharp
// Demo.Api/Endpoints/ProductEndpoints.cs
using Microsoft.AspNetCore.Authorization;

namespace Demo.Api.Endpoints;

public static class ProductEndpoints
{
    public static RouteGroupBuilder MapProductEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/products")
            .WithTags("Products")
            .RequireAuthorization(); // All endpoints require authentication

        // Anyone authenticated can view
        group.MapGet("/", GetAllProducts);

        // Only Admins can create
        group.MapPost("/", CreateProduct)
            .RequireAuthorization("Admin");

        // Only Admins or Managers can update
        group.MapPut("/{id:guid}", UpdateProduct)
            .RequireAuthorization(new AuthorizeAttribute { Roles = "Admin,Manager" });

        // Only SuperAdmin can delete
        group.MapDelete("/{id:guid}", DeleteProduct)
            .RequireAuthorization("SuperAdmin");

        return group;
    }

    private static IResult GetAllProducts() => Results.Ok(new[] { "Product1", "Product2" });
    private static IResult CreateProduct() => Results.Created("/products/1", new { Id = 1 });
    private static IResult UpdateProduct(Guid id) => Results.NoContent();
    private static IResult DeleteProduct(Guid id) => Results.NoContent();
}
```

## Policy-Based Authorization

### Define Authorization Policies

```csharp
// Program.cs
using Microsoft.AspNetCore.Authorization;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(/* JWT config */);

builder.Services.AddAuthorization(options =>
{
    // Role-based policies
    options.AddPolicy("RequireAdminRole", policy =>
        policy.RequireRole("Admin", "SuperAdmin"));

    // Claim-based policies
    options.AddPolicy("RequireEmailVerified", policy =>
        policy.RequireClaim("email_verified", "true"));

    // Multiple requirements (AND logic)
    options.AddPolicy("ManagerWithVerifiedEmail", policy =>
    {
        policy.RequireRole("Manager");
        policy.RequireClaim("email_verified", "true");
    });

    // Custom requirement
    options.AddPolicy("AtLeast21", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(21)));

    // Combine multiple conditions
    options.AddPolicy("SeniorManager", policy =>
    {
        policy.RequireRole("Manager");
        policy.RequireClaim("department", "Engineering", "Operations");
        policy.RequireClaim("seniority_level", "Senior", "Principal");
    });

    // Assertion-based policy (inline logic)
    options.AddPolicy("ActiveUser", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c =>
                c.Type == "account_status" &&
                c.Value == "active")));

    // Default fallback policy (require authentication for all endpoints)
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Use policies
app.MapGet("/admin", () => "Admin only")
    .RequireAuthorization("RequireAdminRole");

app.MapGet("/verified", () => "Email verified users")
    .RequireAuthorization("RequireEmailVerified");

app.MapGet("/legal", () => "21+ content")
    .RequireAuthorization("AtLeast21");

app.Run();
```

### Custom Authorization Requirements

#### MinimumAgeRequirement

```csharp
// Demo.Api/Authorization/Requirements/MinimumAgeRequirement.cs
using Microsoft.AspNetCore.Authorization;

namespace Demo.Api.Authorization.Requirements;

public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}
```

#### MinimumAgeHandler

```csharp
// Demo.Api/Authorization/Handlers/MinimumAgeHandler.cs
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Demo.Api.Authorization.Requirements;

namespace Demo.Api.Authorization.Handlers;

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        // Get date of birth claim
        var dobClaim = context.User.FindFirst(c => c.Type == "date_of_birth");
        if (dobClaim is null)
        {
            return Task.CompletedTask; // Fail silently (requirement not met)
        }

        // Parse and calculate age
        if (!DateOnly.TryParse(dobClaim.Value, out var dateOfBirth))
        {
            return Task.CompletedTask;
        }

        var age = CalculateAge(dateOfBirth);

        // Check if user meets minimum age
        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }

    private static int CalculateAge(DateOnly dateOfBirth)
    {
        var today = DateOnly.FromDateTime(DateTime.UtcNow);
        var age = today.Year - dateOfBirth.Year;

        if (dateOfBirth > today.AddYears(-age))
        {
            age--;
        }

        return age;
    }
}
```

#### Register Custom Handler

```csharp
// Program.cs
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### Advanced Custom Requirement - Same Author

```csharp
// Demo.Api/Authorization/Requirements/SameAuthorRequirement.cs
namespace Demo.Api.Authorization.Requirements;

public class SameAuthorRequirement : IAuthorizationRequirement { }
```

```csharp
// Demo.Api/Authorization/Handlers/SameAuthorHandler.cs
using Microsoft.AspNetCore.Authorization;
using Demo.Api.Authorization.Requirements;
using Demo.Application.Interfaces;

namespace Demo.Api.Authorization.Handlers;

public class SameAuthorHandler : AuthorizationHandler<SameAuthorRequirement, Document>
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ILogger<SameAuthorHandler> _logger;

    public SameAuthorHandler(
        IHttpContextAccessor httpContextAccessor,
        ILogger<SameAuthorHandler> logger)
    {
        _httpContextAccessor = httpContextAccessor;
        _logger = logger;
    }

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameAuthorRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
        {
            _logger.LogWarning("User ID not found in claims");
            return Task.CompletedTask;
        }

        // Check if user is the author
        if (resource.AuthorId == userId)
        {
            context.Succeed(requirement);
        }
        else
        {
            _logger.LogWarning(
                "User {UserId} attempted to access document {DocumentId} authored by {AuthorId}",
                userId,
                resource.Id,
                resource.AuthorId
            );
        }

        return Task.CompletedTask;
    }
}
```

## Resource-Based Authorization

### Authorization Service Pattern

```csharp
// Demo.Api/Endpoints/DocumentEndpoints.cs
using Microsoft.AspNetCore.Authorization;
using Demo.Api.Authorization.Requirements;
using Demo.Application.Interfaces;

namespace Demo.Api.Endpoints;

public static class DocumentEndpoints
{
    public static RouteGroupBuilder MapDocumentEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/documents")
            .WithTags("Documents")
            .RequireAuthorization();

        group.MapGet("/{id:guid}", GetDocument);
        group.MapPut("/{id:guid}", UpdateDocument);
        group.MapDelete("/{id:guid}", DeleteDocument);

        return group;
    }

    private static async Task<IResult> GetDocument(
        Guid id,
        IDocumentService documentService,
        IAuthorizationService authorizationService,
        ClaimsPrincipal user,
        CancellationToken ct)
    {
        var document = await documentService.GetByIdAsync(id, ct);
        if (document is null)
        {
            return Results.NotFound();
        }

        // Check if user can view this document
        var authResult = await authorizationService.AuthorizeAsync(
            user,
            document,
            "ViewDocument"
        );

        if (!authResult.Succeeded)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status403Forbidden,
                title: "Forbidden",
                detail: "You do not have permission to view this document"
            );
        }

        return Results.Ok(document);
    }

    private static async Task<IResult> UpdateDocument(
        Guid id,
        UpdateDocumentRequest request,
        IDocumentService documentService,
        IAuthorizationService authorizationService,
        ClaimsPrincipal user,
        CancellationToken ct)
    {
        var document = await documentService.GetByIdAsync(id, ct);
        if (document is null)
        {
            return Results.NotFound();
        }

        // OWASP: Broken Object Level Authorization mitigation
        // Verify user owns the resource before allowing modification
        var authResult = await authorizationService.AuthorizeAsync(
            user,
            document,
            new SameAuthorRequirement()
        );

        if (!authResult.Succeeded)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status403Forbidden,
                title: "Forbidden",
                detail: "You can only edit your own documents"
            );
        }

        await documentService.UpdateAsync(id, request, ct);
        return Results.NoContent();
    }

    private static async Task<IResult> DeleteDocument(
        Guid id,
        IDocumentService documentService,
        IAuthorizationService authorizationService,
        ClaimsPrincipal user,
        CancellationToken ct)
    {
        var document = await documentService.GetByIdAsync(id, ct);
        if (document is null)
        {
            return Results.NotFound();
        }

        // Only author or admin can delete
        var authResult = await authorizationService.AuthorizeAsync(
            user,
            document,
            "DeleteDocument"
        );

        if (!authResult.Succeeded)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status403Forbidden,
                title: "Forbidden",
                detail: "You do not have permission to delete this document"
            );
        }

        await documentService.DeleteAsync(id, ct);
        return Results.NoContent();
    }
}
```

### Document Authorization Policies

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ViewDocument", policy =>
        policy.Requirements.Add(new DocumentViewRequirement()));

    options.AddPolicy("DeleteDocument", policy =>
        policy.Requirements.Add(new DocumentDeleteRequirement()));
});

builder.Services.AddSingleton<IAuthorizationHandler, SameAuthorHandler>();
builder.Services.AddSingleton<IAuthorizationHandler, DocumentViewHandler>();
builder.Services.AddSingleton<IAuthorizationHandler, DocumentDeleteHandler>();
```

### Document View Requirement

```csharp
// Demo.Api/Authorization/Handlers/DocumentViewHandler.cs
using Microsoft.AspNetCore.Authorization;

namespace Demo.Api.Authorization.Handlers;

public class DocumentViewRequirement : IAuthorizationRequirement { }

public class DocumentViewHandler : AuthorizationHandler<DocumentViewRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentViewRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;

        // Owner can always view
        if (resource.AuthorId == userId)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Admins can view all documents
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Users in the same department can view
        var userDepartment = context.User.FindFirst("department")?.Value;
        if (!string.IsNullOrEmpty(userDepartment) &&
            resource.Department == userDepartment)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Document is public
        if (resource.IsPublic)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

### Document Delete Requirement

```csharp
// Demo.Api/Authorization/Handlers/DocumentDeleteHandler.cs
namespace Demo.Api.Authorization.Handlers;

public class DocumentDeleteRequirement : IAuthorizationRequirement { }

public class DocumentDeleteHandler : AuthorizationHandler<DocumentDeleteRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentDeleteRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;

        // Only author can delete
        if (resource.AuthorId == userId)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Or admins
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

## Scopes and Claims Mapping

### OAuth 2.0 Scopes

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Scope-based policies (common in OAuth 2.0)
    options.AddPolicy("ReadAccess", policy =>
        policy.RequireClaim("scope", "read"));

    options.AddPolicy("WriteAccess", policy =>
        policy.RequireClaim("scope", "write"));

    options.AddPolicy("FullAccess", policy =>
        policy.RequireClaim("scope", "read", "write"));

    // API-specific scopes
    options.AddPolicy("ProductsRead", policy =>
        policy.RequireClaim("scope", "products:read"));

    options.AddPolicy("ProductsWrite", policy =>
        policy.RequireClaim("scope", "products:write"));

    options.AddPolicy("OrdersManage", policy =>
        policy.RequireClaim("scope", "orders:manage"));
});

// Usage
app.MapGet("/products", GetProducts)
    .RequireAuthorization("ProductsRead");

app.MapPost("/products", CreateProduct)
    .RequireAuthorization("ProductsWrite");

app.MapGet("/orders", GetOrders)
    .RequireAuthorization("OrdersManage");
```

### Transform Scopes to Claims

```csharp
// Demo.Api/Security/ScopeClaimsTransformer.cs
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;

namespace Demo.Api.Security;

public class ScopeClaimsTransformer : IClaimsTransformation
{
    public Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var claimsIdentity = principal.Identity as ClaimsIdentity;
        if (claimsIdentity is null)
        {
            return Task.FromResult(principal);
        }

        // Auth0/Azure AD often send scopes as space-separated string
        var scopeClaim = principal.FindFirst("scope");
        if (scopeClaim is not null)
        {
            var scopes = scopeClaim.Value.Split(' ', StringSplitOptions.RemoveEmptyEntries);

            foreach (var scope in scopes)
            {
                // Add individual scope claims
                claimsIdentity.AddClaim(new Claim("scope", scope));

                // Map scopes to roles for easier authorization
                var role = MapScopeToRole(scope);
                if (!string.IsNullOrEmpty(role))
                {
                    claimsIdentity.AddClaim(new Claim(ClaimTypes.Role, role));
                }
            }
        }

        return Task.FromResult(principal);
    }

    private static string? MapScopeToRole(string scope)
    {
        return scope switch
        {
            "admin" => "Admin",
            "products:write" => "ProductManager",
            "orders:manage" => "OrderManager",
            _ => null
        };
    }
}
```

## Combining Multiple Policies

### Multiple Policies (AND Logic)

```csharp
// Require ALL policies to succeed
app.MapPost("/sensitive-operation", () => "Success")
    .RequireAuthorization("RequireAdminRole")
    .RequireAuthorization("RequireEmailVerified")
    .RequireAuthorization("AtLeast21");
```

### Complex Authorization Logic

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanApproveExpense", policy =>
    {
        policy.RequireAuthenticatedUser();

        // Must be manager OR admin
        policy.RequireAssertion(context =>
            context.User.IsInRole("Manager") ||
            context.User.IsInRole("Admin"));

        // AND must have verified email
        policy.RequireClaim("email_verified", "true");

        // AND must not be suspended
        policy.RequireAssertion(context =>
            !context.User.HasClaim("account_status", "suspended"));
    });
});
```

### Custom Policy Provider (Advanced)

```csharp
// Demo.Api/Authorization/CustomPolicyProvider.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.Options;

namespace Demo.Api.Authorization;

public class CustomPolicyProvider : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallbackPolicyProvider;

    public CustomPolicyProvider(IOptions<AuthorizationOptions> options)
    {
        _fallbackPolicyProvider = new DefaultAuthorizationPolicyProvider(options);
    }

    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        // Dynamic policy: MinimumAge:21
        if (policyName.StartsWith("MinimumAge:", StringComparison.OrdinalIgnoreCase))
        {
            var ageString = policyName.Substring("MinimumAge:".Length);
            if (int.TryParse(ageString, out var age))
            {
                var policy = new AuthorizationPolicyBuilder();
                policy.AddRequirements(new MinimumAgeRequirement(age));
                return Task.FromResult<AuthorizationPolicy?>(policy.Build());
            }
        }

        // Dynamic policy: Department:Engineering
        if (policyName.StartsWith("Department:", StringComparison.OrdinalIgnoreCase))
        {
            var department = policyName.Substring("Department:".Length);
            var policy = new AuthorizationPolicyBuilder();
            policy.RequireClaim("department", department);
            return Task.FromResult<AuthorizationPolicy?>(policy.Build());
        }

        // Fallback to configured policies
        return _fallbackPolicyProvider.GetPolicyAsync(policyName);
    }

    public Task<AuthorizationPolicy> GetDefaultPolicyAsync()
        => _fallbackPolicyProvider.GetDefaultPolicyAsync();

    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync()
        => _fallbackPolicyProvider.GetFallbackPolicyAsync();
}
```

```csharp
// Program.cs - Register custom policy provider
builder.Services.AddSingleton<IAuthorizationPolicyProvider, CustomPolicyProvider>();

// Usage with dynamic policies
app.MapGet("/engineering", () => "Engineering Department")
    .RequireAuthorization("Department:Engineering");

app.MapGet("/legal", () => "Legal Content")
    .RequireAuthorization("MinimumAge:21");
```

## Security Checklist

- [ ] Always authenticate before authorizing (`UseAuthentication()` before `UseAuthorization()`)
- [ ] Use policy-based authorization for complex logic
- [ ] Implement resource-based authorization for ownership checks
- [ ] Log authorization failures for security monitoring
- [ ] Return 403 (Forbidden) for authenticated but unauthorized users
- [ ] Return 401 (Unauthorized) for unauthenticated users
- [ ] Never expose sensitive data in authorization error messages
- [ ] Test all authorization paths in integration tests
- [ ] Use `[AllowAnonymous]` sparingly and intentionally
- [ ] Implement tenant isolation in multi-tenant applications
- [ ] Validate ownership before modifying resources (OWASP BOLA mitigation)
- [ ] Use scopes for API-to-API authorization
- [ ] Implement rate limiting per user/role
- [ ] Audit authorization decisions for compliance

## Common Mistakes

### Mistake: Forgetting UseAuthentication

```csharp
// DON'T
app.UseAuthorization(); // Authentication middleware missing!
```

**Solution**: Always use both in correct order

```csharp
// DO
app.UseAuthentication();
app.UseAuthorization();
```

### Mistake: Not checking resource ownership

```csharp
// DON'T - Vulnerable to BOLA (Broken Object Level Authorization)
app.MapDelete("/documents/{id}", async (Guid id, IDocumentService svc) =>
{
    await svc.DeleteAsync(id); // Anyone can delete any document!
    return Results.NoContent();
});
```

**Solution**: Verify ownership

```csharp
// DO
app.MapDelete("/documents/{id}", async (
    Guid id,
    ClaimsPrincipal user,
    IDocumentService svc,
    IAuthorizationService authService) =>
{
    var document = await svc.GetByIdAsync(id);
    var authResult = await authService.AuthorizeAsync(
        user,
        document,
        new SameAuthorRequirement()
    );

    if (!authResult.Succeeded)
    {
        return Results.Forbid();
    }

    await svc.DeleteAsync(id);
    return Results.NoContent();
});
```

### Mistake: Using string comparisons for roles

```csharp
// DON'T
if (user.FindFirst(ClaimTypes.Role)?.Value == "Admin") // Only checks first role
```

**Solution**: Use IsInRole

```csharp
// DO
if (user.IsInRole("Admin"))
```

## Navigation

- **Previous**: `authentication-jwt.md`
- **Next**: `api-protection.md`
- **Related**: `owasp-checklist.md`, `../06-error-handling/problem-details.md`

## References

- [OWASP API Security - Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP API Security - Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [Microsoft: Policy-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
- [Microsoft: Resource-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased)
- [Microsoft: Claims-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/claims)
