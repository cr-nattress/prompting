# JWT Authentication in .NET 8

> **File Purpose**: Implement JWT-based authentication with OIDC providers (Auth0, Azure AD)
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Basic API setup
> **Related Files**: `authorization.md`, `api-protection.md`, `../06-error-handling/problem-details.md`
> **Agent Use Case**: Reference when implementing JWT authentication and identity provider integration

## Quick Context

JWT (JSON Web Tokens) authentication provides stateless, scalable authentication for .NET 8 APIs. This guide covers JWT bearer token configuration, OIDC integration with popular identity providers, token validation, and refresh token patterns following Microsoft and OWASP best practices.

**OWASP References**:
- [OWASP API Security Top 10 - API2:2023 Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/)
- [OWASP ASVS v4.0 - Authentication](https://github.com/OWASP/ASVS/blob/master/4.0/en/0x11-V2-Authentication.md)

**Microsoft References**:
- [ASP.NET Core Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [JWT Bearer Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn)

## Install Required Packages

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```

## Basic JWT Configuration (Self-Signed Tokens)

### appsettings.json

```json
{
  "Jwt": {
    "Issuer": "https://api.example.com",
    "Audience": "https://api.example.com",
    "SecretKey": "your-256-bit-secret-key-min-32-chars-long!!",
    "ExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 7
  },
  "Logging": {
    "LogLevel": {
      "Microsoft.AspNetCore.Authentication": "Information"
    }
  }
}
```

**Security Notes**:
- Store `SecretKey` in **Azure Key Vault** or **AWS Secrets Manager** in production
- Never commit secrets to source control
- Use minimum 256-bit (32 character) keys
- Rotate keys regularly (every 90 days recommended)

### Program.cs - Basic JWT Setup

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// Configure JWT authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtSettings = builder.Configuration.GetSection("Jwt");
        var secretKey = jwtSettings["SecretKey"]!;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            // Validate issuer (who created the token)
            ValidateIssuer = true,
            ValidIssuer = jwtSettings["Issuer"],

            // Validate audience (who the token is intended for)
            ValidateAudience = true,
            ValidAudience = jwtSettings["Audience"],

            // Validate token signature
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(secretKey)
            ),

            // Validate token expiration
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5), // Allow 5 min clock skew

            // OWASP recommendation: Require expiration claim
            RequireExpirationTime = true,
            RequireSignedTokens = true
        };

        // Configure events for logging and custom handling
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();

                logger.LogWarning(
                    "Authentication failed: {Exception}",
                    context.Exception.Message
                );

                return Task.CompletedTask;
            },
            OnTokenValidated = context =>
            {
                var logger = context.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();

                var userId = context.Principal?.FindFirst("sub")?.Value;
                logger.LogInformation(
                    "Token validated for user: {UserId}",
                    userId
                );

                return Task.CompletedTask;
            },
            OnChallenge = context =>
            {
                // Override default 401 response with ProblemDetails
                context.HandleResponse();

                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                context.Response.ContentType = "application/problem+json";

                var problemDetails = new ProblemDetails
                {
                    Status = StatusCodes.Status401Unauthorized,
                    Type = "https://tools.ietf.org/html/rfc7235#section-3.1",
                    Title = "Unauthorized",
                    Detail = "Valid authentication credentials required"
                };

                return context.Response.WriteAsJsonAsync(problemDetails);
            }
        };

        // HTTPS requirement (OWASP recommendation)
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
    });

builder.Services.AddAuthorization();

var app = builder.Build();

// IMPORTANT: Order matters!
app.UseAuthentication(); // Must come before UseAuthorization
app.UseAuthorization();

app.MapGet("/secure", () => "Secure data")
    .RequireAuthorization();

app.Run();
```

## Token Generation Service

### JwtSettings.cs

```csharp
// Demo.Api/Configuration/JwtSettings.cs
namespace Demo.Api.Configuration;

public class JwtSettings
{
    public const string SectionName = "Jwt";

    public required string Issuer { get; init; }
    public required string Audience { get; init; }
    public required string SecretKey { get; init; }
    public int ExpirationMinutes { get; init; } = 60;
    public int RefreshTokenExpirationDays { get; init; } = 7;
}
```

### ITokenService.cs

```csharp
// Demo.Application/Interfaces/ITokenService.cs
namespace Demo.Application.Interfaces;

public interface ITokenService
{
    string GenerateAccessToken(string userId, IEnumerable<string> roles, Dictionary<string, string>? claims = null);
    string GenerateRefreshToken();
    ClaimsPrincipal? ValidateToken(string token);
}
```

### TokenService.cs

```csharp
// Demo.Infrastructure/Services/TokenService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using Demo.Api.Configuration;
using Demo.Application.Interfaces;

namespace Demo.Infrastructure.Services;

public class TokenService : ITokenService
{
    private readonly JwtSettings _jwtSettings;
    private readonly JwtSecurityTokenHandler _tokenHandler;
    private readonly SigningCredentials _signingCredentials;

    public TokenService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
        _tokenHandler = new JwtSecurityTokenHandler();

        var secretKey = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);
        var symmetricKey = new SymmetricSecurityKey(secretKey);
        _signingCredentials = new SigningCredentials(
            symmetricKey,
            SecurityAlgorithms.HmacSha256Signature
        );
    }

    public string GenerateAccessToken(
        string userId,
        IEnumerable<string> roles,
        Dictionary<string, string>? claims = null)
    {
        var tokenClaims = new List<Claim>
        {
            // Standard JWT claims (OIDC-compliant)
            new(JwtRegisteredClaimNames.Sub, userId), // Subject (user ID)
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()), // JWT ID
            new(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString()), // Issued at

            // User identity
            new(ClaimTypes.NameIdentifier, userId)
        };

        // Add roles
        foreach (var role in roles)
        {
            tokenClaims.Add(new Claim(ClaimTypes.Role, role));
        }

        // Add custom claims
        if (claims is not null)
        {
            foreach (var (key, value) in claims)
            {
                tokenClaims.Add(new Claim(key, value));
            }
        }

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(tokenClaims),
            Expires = DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            Issuer = _jwtSettings.Issuer,
            Audience = _jwtSettings.Audience,
            SigningCredentials = _signingCredentials,

            // OWASP recommendation: Use strong encryption
            EncryptingCredentials = null // Add encryption for sensitive claims if needed
        };

        var token = _tokenHandler.CreateToken(tokenDescriptor);
        return _tokenHandler.WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        // Generate cryptographically secure random token
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }

    public ClaimsPrincipal? ValidateToken(string token)
    {
        try
        {
            var secretKey = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidIssuer = _jwtSettings.Issuer,
                ValidateAudience = true,
                ValidAudience = _jwtSettings.Audience,
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(secretKey),
                ValidateLifetime = false, // Don't validate lifetime for refresh token scenarios
                ClockSkew = TimeSpan.Zero
            };

            var principal = _tokenHandler.ValidateToken(
                token,
                validationParameters,
                out var validatedToken
            );

            // Verify token is JWT with correct algorithm
            if (validatedToken is not JwtSecurityToken jwtToken ||
                !jwtToken.Header.Alg.Equals(
                    SecurityAlgorithms.HmacSha256,
                    StringComparison.InvariantCultureIgnoreCase))
            {
                return null;
            }

            return principal;
        }
        catch
        {
            return null;
        }
    }
}
```

## Authentication Endpoints

### Login Request/Response DTOs

```csharp
// Demo.Api/Contracts/Auth/LoginRequest.cs
namespace Demo.Api.Contracts.Auth;

public record LoginRequest(
    string Email,
    string Password
);

public record LoginResponse(
    string AccessToken,
    string RefreshToken,
    int ExpiresIn,
    string TokenType = "Bearer"
);

public record RefreshTokenRequest(
    string RefreshToken
);
```

### Login Endpoint

```csharp
// Demo.Api/Endpoints/AuthEndpoints.cs
using Microsoft.AspNetCore.Identity;
using Demo.Api.Contracts.Auth;
using Demo.Application.Interfaces;

namespace Demo.Api.Endpoints;

public static class AuthEndpoints
{
    public static RouteGroupBuilder MapAuthEndpoints(this IEndpointRouteBuilder routes)
    {
        var group = routes.MapGroup("/auth")
            .WithTags("Authentication");

        group.MapPost("/login", LoginAsync)
            .AllowAnonymous()
            .Produces<LoginResponse>(StatusCodes.Status200OK)
            .ProducesProblem(StatusCodes.Status401Unauthorized);

        group.MapPost("/refresh", RefreshTokenAsync)
            .AllowAnonymous()
            .Produces<LoginResponse>(StatusCodes.Status200OK)
            .ProducesProblem(StatusCodes.Status401Unauthorized);

        group.MapPost("/logout", LogoutAsync)
            .RequireAuthorization()
            .Produces(StatusCodes.Status204NoContent);

        return group;
    }

    private static async Task<IResult> LoginAsync(
        LoginRequest request,
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ITokenService tokenService,
        IConfiguration configuration,
        CancellationToken ct)
    {
        // Find user by email
        var user = await userManager.FindByEmailAsync(request.Email);
        if (user is null)
        {
            return Results.Problem(
                statusCode: StatusCodes.Status401Unauthorized,
                title: "Invalid credentials",
                detail: "Email or password is incorrect"
            );
        }

        // Verify password
        var result = await signInManager.CheckPasswordSignInAsync(
            user,
            request.Password,
            lockoutOnFailure: true // OWASP: Enable account lockout
        );

        if (!result.Succeeded)
        {
            if (result.IsLockedOut)
            {
                return Results.Problem(
                    statusCode: StatusCodes.Status423Locked,
                    title: "Account locked",
                    detail: "Account has been locked due to multiple failed login attempts"
                );
            }

            return Results.Problem(
                statusCode: StatusCodes.Status401Unauthorized,
                title: "Invalid credentials",
                detail: "Email or password is incorrect"
            );
        }

        // Get user roles
        var roles = await userManager.GetRolesAsync(user);

        // Generate tokens
        var accessToken = tokenService.GenerateAccessToken(
            user.Id,
            roles,
            new Dictionary<string, string>
            {
                ["email"] = user.Email!,
                ["email_verified"] = user.EmailConfirmed.ToString()
            }
        );

        var refreshToken = tokenService.GenerateRefreshToken();

        // Store refresh token (implement your storage mechanism)
        // await _refreshTokenRepository.SaveAsync(user.Id, refreshToken, ct);

        var jwtSettings = configuration.GetSection("Jwt");
        var expiresIn = int.Parse(jwtSettings["ExpirationMinutes"]!);

        return Results.Ok(new LoginResponse(
            accessToken,
            refreshToken,
            expiresIn * 60 // Convert to seconds
        ));
    }

    private static async Task<IResult> RefreshTokenAsync(
        RefreshTokenRequest request,
        ITokenService tokenService,
        IConfiguration configuration,
        CancellationToken ct)
    {
        // Validate refresh token (implement your validation logic)
        // var storedToken = await _refreshTokenRepository.GetAsync(request.RefreshToken, ct);
        // if (storedToken is null || storedToken.IsRevoked || storedToken.ExpiresAt < DateTime.UtcNow)
        // {
        //     return Results.Unauthorized();
        // }

        // Generate new tokens
        // var accessToken = tokenService.GenerateAccessToken(storedToken.UserId, storedToken.Roles);
        // var newRefreshToken = tokenService.GenerateRefreshToken();

        // Rotate refresh token (OWASP best practice)
        // await _refreshTokenRepository.RevokeAsync(request.RefreshToken, ct);
        // await _refreshTokenRepository.SaveAsync(storedToken.UserId, newRefreshToken, ct);

        return Results.Ok(new LoginResponse(
            "new-access-token",
            "new-refresh-token",
            3600
        ));
    }

    private static IResult LogoutAsync(HttpContext context)
    {
        // Implement token revocation/blacklisting
        // var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        // await _refreshTokenRepository.RevokeAllAsync(userId, ct);

        return Results.NoContent();
    }
}
```

## OIDC Integration - Auth0

### appsettings.json - Auth0

```json
{
  "Auth0": {
    "Domain": "your-tenant.auth0.com",
    "Audience": "https://api.example.com",
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret"
  }
}
```

### Program.cs - Auth0 Configuration

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

var auth0Domain = builder.Configuration["Auth0:Domain"]!;
var auth0Audience = builder.Configuration["Auth0:Audience"]!;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = $"https://{auth0Domain}/";
        options.Audience = auth0Audience;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = $"https://{auth0Domain}/",
            ValidateAudience = true,
            ValidAudience = auth0Audience,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromMinutes(5)
        };

        // Configure metadata endpoint for OIDC discovery
        options.MetadataAddress = $"https://{auth0Domain}/.well-known/openid-configuration";

        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = context =>
            {
                // Custom claims transformation
                var claimsIdentity = context.Principal?.Identity as ClaimsIdentity;

                // Extract Auth0 permissions and convert to roles
                var permissions = context.Principal?
                    .FindAll("permissions")
                    .Select(c => c.Value)
                    .ToList() ?? new List<string>();

                foreach (var permission in permissions)
                {
                    claimsIdentity?.AddClaim(new Claim(ClaimTypes.Role, permission));
                }

                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/secure", (ClaimsPrincipal user) =>
    new
    {
        user.Identity?.Name,
        Claims = user.Claims.Select(c => new { c.Type, c.Value })
    })
    .RequireAuthorization();

app.Run();
```

## OIDC Integration - Azure AD (Microsoft Entra ID)

### appsettings.json - Azure AD

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "your-tenant-id",
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret",
    "Audience": "api://your-api-client-id"
  }
}
```

### Program.cs - Azure AD Configuration

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Identity.Web;

var builder = WebApplication.CreateBuilder(args);

// Use Microsoft.Identity.Web for Azure AD integration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    // Require authenticated users by default
    options.FallbackPolicy = options.DefaultPolicy;
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/user-info", (ClaimsPrincipal user) =>
{
    var userId = user.FindFirst("oid")?.Value; // Object ID
    var email = user.FindFirst("preferred_username")?.Value;
    var name = user.FindFirst("name")?.Value;
    var roles = user.FindAll("roles").Select(c => c.Value);

    return new
    {
        userId,
        email,
        name,
        roles
    };
})
.RequireAuthorization();

app.Run();
```

## Claims Transformation

### Custom Claims Transformer

```csharp
// Demo.Api/Security/CustomClaimsTransformer.cs
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;

namespace Demo.Api.Security;

public class CustomClaimsTransformer : IClaimsTransformation
{
    private readonly IUserService _userService;

    public CustomClaimsTransformer(IUserService userService)
    {
        _userService = userService;
    }

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        // Clone current identity
        var claimsIdentity = new ClaimsIdentity(
            principal.Identity,
            principal.Claims,
            principal.Identity?.AuthenticationType,
            ClaimTypes.Name,
            ClaimTypes.Role
        );

        // Get user ID from token
        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(userId))
        {
            return principal;
        }

        // Fetch additional claims from database
        var userClaims = await _userService.GetUserClaimsAsync(userId);

        foreach (var claim in userClaims)
        {
            // Add custom claims
            claimsIdentity.AddClaim(new Claim(claim.Type, claim.Value));
        }

        // Add custom computed claims
        var isManager = userClaims.Any(c =>
            c.Type == ClaimTypes.Role && c.Value == "Manager");

        if (isManager)
        {
            claimsIdentity.AddClaim(new Claim("can_approve", "true"));
        }

        return new ClaimsPrincipal(claimsIdentity);
    }
}
```

### Register Claims Transformer

```csharp
// Program.cs
builder.Services.AddTransient<IClaimsTransformation, CustomClaimsTransformer>();
```

## Refresh Token Implementation

### RefreshToken Entity

```csharp
// Demo.Domain/Entities/RefreshToken.cs
namespace Demo.Domain.Entities;

public class RefreshToken
{
    public Guid Id { get; set; }
    public required string Token { get; set; }
    public required string UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsRevoked { get; set; }
    public string? RevokedReason { get; set; }
    public string? ReplacedByToken { get; set; }
    public string? CreatedByIp { get; set; }
    public string? RevokedByIp { get; set; }

    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    public bool IsActive => !IsRevoked && !IsExpired;
}
```

### Refresh Token Repository

```csharp
// Demo.Infrastructure/Repositories/RefreshTokenRepository.cs
using Microsoft.EntityFrameworkCore;
using Demo.Domain.Entities;

namespace Demo.Infrastructure.Repositories;

public class RefreshTokenRepository
{
    private readonly ApplicationDbContext _context;

    public RefreshTokenRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<RefreshToken?> GetByTokenAsync(string token, CancellationToken ct = default)
    {
        return await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token, ct);
    }

    public async Task SaveAsync(
        string userId,
        string token,
        string ipAddress,
        int expirationDays,
        CancellationToken ct = default)
    {
        var refreshToken = new RefreshToken
        {
            Token = token,
            UserId = userId,
            ExpiresAt = DateTime.UtcNow.AddDays(expirationDays),
            CreatedAt = DateTime.UtcNow,
            CreatedByIp = ipAddress
        };

        _context.RefreshTokens.Add(refreshToken);
        await _context.SaveChangesAsync(ct);
    }

    public async Task RevokeAsync(
        string token,
        string ipAddress,
        string? reason = null,
        string? replacedByToken = null,
        CancellationToken ct = default)
    {
        var refreshToken = await GetByTokenAsync(token, ct);
        if (refreshToken is null || !refreshToken.IsActive)
        {
            return;
        }

        refreshToken.IsRevoked = true;
        refreshToken.RevokedByIp = ipAddress;
        refreshToken.RevokedReason = reason;
        refreshToken.ReplacedByToken = replacedByToken;

        await _context.SaveChangesAsync(ct);
    }

    public async Task RevokeAllForUserAsync(
        string userId,
        string ipAddress,
        CancellationToken ct = default)
    {
        var tokens = await _context.RefreshTokens
            .Where(rt => rt.UserId == userId && rt.IsActive)
            .ToListAsync(ct);

        foreach (var token in tokens)
        {
            token.IsRevoked = true;
            token.RevokedByIp = ipAddress;
            token.RevokedReason = "User logout";
        }

        await _context.SaveChangesAsync(ct);
    }

    public async Task RemoveExpiredTokensAsync(CancellationToken ct = default)
    {
        var expiredTokens = await _context.RefreshTokens
            .Where(rt => rt.ExpiresAt < DateTime.UtcNow)
            .ToListAsync(ct);

        _context.RefreshTokens.RemoveRange(expiredTokens);
        await _context.SaveChangesAsync(ct);
    }
}
```

## Complete Working Example

### Full Program.cs with Security Best Practices

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Demo.Api.Configuration;
using Demo.Api.Security;
using Demo.Application.Interfaces;
using Demo.Infrastructure.Services;

var builder = WebApplication.CreateBuilder(args);

// Bind JWT settings
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName)
);

// Register services
builder.Services.AddScoped<ITokenService, TokenService>();
builder.Services.AddTransient<IClaimsTransformation, CustomClaimsTransformer>();

// Configure authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    var jwtSettings = builder.Configuration.GetSection("Jwt");
    var secretKey = jwtSettings["SecretKey"]!;

    options.SaveToken = true;
    options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();

    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidateAudience = true,
        ValidAudience = jwtSettings["Audience"],
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(secretKey)
        ),
        ValidateLifetime = true,
        ClockSkew = TimeSpan.FromMinutes(5),
        RequireExpirationTime = true,
        RequireSignedTokens = true
    };

    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            var logger = context.HttpContext.RequestServices
                .GetRequiredService<ILogger<Program>>();

            logger.LogWarning(
                "JWT authentication failed: {Error}",
                context.Exception.Message
            );

            return Task.CompletedTask;
        },
        OnTokenValidated = context =>
        {
            var logger = context.HttpContext.RequestServices
                .GetRequiredService<ILogger<Program>>();

            var userId = context.Principal?.FindFirst("sub")?.Value;
            logger.LogInformation("Token validated for user: {UserId}", userId);

            return Task.CompletedTask;
        },
        OnChallenge = context =>
        {
            context.HandleResponse();

            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            context.Response.ContentType = "application/problem+json";

            var problemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status401Unauthorized,
                Type = "https://tools.ietf.org/html/rfc7235#section-3.1",
                Title = "Unauthorized",
                Detail = context.ErrorDescription ?? "Valid JWT token required",
                Instance = context.Request.Path
            };

            problemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;

            return context.Response.WriteAsJsonAsync(problemDetails);
        }
    };
});

builder.Services.AddAuthorization();

var app = builder.Build();

// Security middleware order
app.UseHttpsRedirection(); // Force HTTPS
app.UseAuthentication(); // Authenticate user
app.UseAuthorization(); // Authorize access

// Public endpoint
app.MapGet("/", () => "API is running")
    .AllowAnonymous();

// Protected endpoint
app.MapGet("/secure", (ClaimsPrincipal user) => new
{
    userId = user.FindFirst("sub")?.Value,
    email = user.FindFirst("email")?.Value,
    roles = user.FindAll(ClaimTypes.Role).Select(c => c.Value)
})
.RequireAuthorization();

app.Run();
```

## Testing JWT Authentication

### Test with HTTP Client

```http
### Login
POST https://localhost:7001/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}

### Use access token
@accessToken = {{login.response.body.accessToken}}

GET https://localhost:7001/secure
Authorization: Bearer {{accessToken}}

### Refresh token
POST https://localhost:7001/auth/refresh
Content-Type: application/json

{
  "refreshToken": "{{login.response.body.refreshToken}}"
}
```

## Security Checklist

- [ ] Use HTTPS in production (`RequireHttpsMetadata = true`)
- [ ] Store secrets in Azure Key Vault/AWS Secrets Manager
- [ ] Use minimum 256-bit signing keys
- [ ] Set appropriate token expiration (15-60 minutes)
- [ ] Implement refresh token rotation
- [ ] Enable account lockout after failed attempts
- [ ] Validate all token claims (issuer, audience, expiration)
- [ ] Log authentication failures
- [ ] Use secure random generator for refresh tokens
- [ ] Implement token revocation/blacklisting
- [ ] Set `ClockSkew` to reasonable value (5 minutes max)
- [ ] Never expose stack traces in production
- [ ] Rotate signing keys regularly
- [ ] Use `RequireSignedTokens = true`
- [ ] Use `RequireExpirationTime = true`

## Common Mistakes

### Mistake: Storing secrets in appsettings.json

```csharp
// DON'T
"Jwt": {
  "SecretKey": "my-secret-key" // Committed to git!
}
```

**Solution**: Use User Secrets locally, Azure Key Vault in production

```bash
dotnet user-secrets set "Jwt:SecretKey" "your-secret-key"
```

### Mistake: Not validating token lifetime

```csharp
// DON'T
ValidateLifetime = false // Tokens never expire!
```

**Solution**: Always validate lifetime

```csharp
// DO
ValidateLifetime = true,
RequireExpirationTime = true
```

### Mistake: Accepting unsigned tokens

```csharp
// DON'T
RequireSignedTokens = false // Anyone can create tokens!
```

**Solution**: Always require signed tokens

```csharp
// DO
RequireSignedTokens = true
```

## Navigation

- **Previous**: `../04-api-design/api-standards.md`
- **Next**: `authorization.md`
- **Related**: `api-protection.md`, `owasp-checklist.md`

## References

- [OWASP API Security Top 10 - Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/)
- [Microsoft: JWT Bearer Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn)
- [Auth0: .NET Core API Authentication](https://auth0.com/docs/quickstart/backend/aspnet-core-webapi)
- [Microsoft Identity Platform](https://learn.microsoft.com/en-us/azure/active-directory/develop/)
- [RFC 6749: OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7519: JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
