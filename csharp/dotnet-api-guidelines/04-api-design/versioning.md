# API Versioning - .NET 8

> **File Purpose**: Implement API versioning using Asp.Versioning.Http with URL and header-based strategies
> **Prerequisites**: `endpoints-and-routing.md` - Endpoint organization
> **Related Files**: `openapi-swagger.md`, `api-standards.md`
> **Agent Use Case**: Reference when implementing API versioning and managing breaking changes

## Quick Context

API versioning allows you to evolve your API while maintaining backward compatibility for existing clients. .NET 8 uses the `Asp.Versioning.Http` package (formerly Microsoft.AspNetCore.Mvc.Versioning) to support URL-based, header-based, and query string versioning strategies.

**Microsoft References**:
- [API Versioning in ASP.NET Core](https://github.com/dotnet/aspnet-api-versioning)
- [Versioning Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api)

## Install Package

```bash
dotnet add package Asp.Versioning.Http
```

## URL-Based Versioning (Recommended)

### Basic Setup

```csharp
// Program.cs
using Asp.Versioning;
using Asp.Versioning.Builder;

var builder = WebApplication.CreateBuilder(args);

// Configure API versioning
builder.Services.AddApiVersioning(options =>
{
    // Default version if not specified
    options.DefaultApiVersion = new ApiVersion(1, 0);

    // Assume default version when client doesn't specify
    options.AssumeDefaultVersionWhenUnspecified = true;

    // Report supported versions in response headers
    options.ReportApiVersions = true;

    // Read version from URL path
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddApiExplorer(options =>
{
    // Format version as "'v'major[.minor][-status]"
    options.GroupNameFormat = "'v'VVV";

    // Substitute version in route template
    options.SubstituteApiVersionInUrl = true;
});

var app = builder.Build();

// Create version sets
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .ReportApiVersions()
    .Build();

// Version 1.0 endpoints
var v1 = app.MapGroup("/api/v{version:apiVersion}")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(new ApiVersion(1, 0))
    .WithOpenApi();

v1.MapGet("/users", () => new { Message = "V1 Users", Version = "1.0" })
    .WithName("GetUsersV1");

v1.MapGet("/users/{id:int}", (int id) => new { Id = id, Version = "1.0" })
    .WithName("GetUserV1");

// Version 2.0 endpoints
var v2 = app.MapGroup("/api/v{version:apiVersion}")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(new ApiVersion(2, 0))
    .WithOpenApi();

v2.MapGet("/users", () => new
{
    Message = "V2 Users",
    Version = "2.0",
    Features = new[] { "Enhanced filtering", "Pagination" }
})
.WithName("GetUsersV2");

v2.MapGet("/users/{id:int}", (int id) => new
{
    Id = id,
    Version = "2.0",
    AdditionalData = "New in V2"
})
.WithName("GetUserV2");

app.Run();
```

**Response Headers**:
```
api-supported-versions: 1.0, 2.0
api-deprecated-versions: (empty if no deprecations)
```

### Organized Version Structure

```csharp
// MyApi.Api/Features/Users/V1/UserEndpointsV1.cs
namespace MyApi.Api.Features.Users.V1;

public static class UserEndpointsV1
{
    public static RouteGroupBuilder MapUserEndpointsV1(
        this IVersionedEndpointRouteBuilder builder)
    {
        var group = builder.MapGroup("/api/v{version:apiVersion}/users")
            .WithApiVersionSet(builder.VersionSet)
            .MapToApiVersion(1.0)
            .WithTags("Users - V1");

        group.MapGet("/", GetAllAsync);
        group.MapGet("/{id:int}", GetByIdAsync);
        group.MapPost("/", CreateAsync);

        return group;
    }

    private static async Task<IResult> GetAllAsync(
        IUserServiceV1 service,
        CancellationToken ct)
    {
        var users = await service.GetUsersAsync(ct);
        return Results.Ok(users);
    }

    private static async Task<IResult> GetByIdAsync(
        int id,
        IUserServiceV1 service,
        CancellationToken ct)
    {
        var user = await service.GetUserByIdAsync(id, ct);
        return user is null ? Results.NotFound() : Results.Ok(user);
    }

    private static async Task<IResult> CreateAsync(
        CreateUserRequestV1 request,
        IUserServiceV1 service,
        CancellationToken ct)
    {
        var user = await service.CreateUserAsync(request, ct);
        return Results.Created($"/api/v1/users/{user.Id}", user);
    }
}

// MyApi.Api/Features/Users/V2/UserEndpointsV2.cs
namespace MyApi.Api.Features.Users.V2;

public static class UserEndpointsV2
{
    public static RouteGroupBuilder MapUserEndpointsV2(
        this IVersionedEndpointRouteBuilder builder)
    {
        var group = builder.MapGroup("/api/v{version:apiVersion}/users")
            .WithApiVersionSet(builder.VersionSet)
            .MapToApiVersion(2.0)
            .WithTags("Users - V2");

        group.MapGet("/", GetAllAsync);
        group.MapGet("/{id:int}", GetByIdAsync);
        group.MapPost("/", CreateAsync);
        group.MapPut("/{id:int}", UpdateAsync); // New in V2

        return group;
    }

    // Implementation with V2-specific features...
}

// Program.cs
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(1.0)
    .HasApiVersion(2.0)
    .Build();

var versionedApi = app.NewVersionedApi();
versionedApi.VersionSet = versionSet;

versionedApi.MapUserEndpointsV1();
versionedApi.MapUserEndpointsV2();
```

## Header-Based Versioning

### Configuration

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;

    // Read version from custom header
    options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");

    // Or from multiple sources
    options.ApiVersionReader = ApiVersionReader.Combine(
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version"),
        new UrlSegmentApiVersionReader()
    );
});

// Endpoints
app.MapGet("/api/users", () => new { Message = "Users" })
    .HasApiVersion(1.0)
    .HasApiVersion(2.0);
```

**Client Request**:
```http
GET /api/users HTTP/1.1
Host: api.example.com
X-API-Version: 2.0
```

## Query String Versioning

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
});

// Client request: /api/users?api-version=2.0
```

## Version Deprecation

### Mark Versions as Deprecated

```csharp
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasDeprecatedApiVersion(new ApiVersion(1, 0)) // Mark V1 as deprecated
    .HasApiVersion(new ApiVersion(2, 0))
    .Build();

var v1 = app.MapGroup("/api/v{version:apiVersion}")
    .WithApiVersionSet(versionSet)
    .MapToApiVersion(1.0)
    .WithTags("V1 - Deprecated");

v1.MapGet("/users", () =>
{
    return Results.Ok(new
    {
        Message = "This version is deprecated. Please use V2.",
        DeprecationNotice = "V1 will be removed on 2025-12-31"
    });
});
```

**Response Headers**:
```
api-supported-versions: 2.0
api-deprecated-versions: 1.0
```

### Sunset Header

```csharp
app.Use(async (context, next) =>
{
    var endpoint = context.GetEndpoint();
    var apiVersion = endpoint?.Metadata.GetMetadata<ApiVersionMetadata>();

    if (apiVersion?.DeprecatedApiVersions.Any() == true)
    {
        // RFC 8594 Sunset header
        context.Response.Headers["Sunset"] = "Sat, 31 Dec 2025 23:59:59 GMT";
        context.Response.Headers["Link"] = "</api/v2/users>; rel=\"successor-version\"";
    }

    await next();
});
```

## Complete Production Example

```csharp
// Program.cs
using Asp.Versioning;
using Asp.Versioning.Builder;
using MyApi.Api.Features.Users.V1;
using MyApi.Api.Features.Users.V2;

var builder = WebApplication.CreateBuilder(args);

// Configure versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(2, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;

    // Support multiple version readers
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new QueryStringApiVersionReader("api-version")
    );
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Swagger with versioning
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        var descriptions = app.DescribeApiVersions();

        foreach (var description in descriptions)
        {
            var url = $"/swagger/{description.GroupName}/swagger.json";
            var name = description.GroupName.ToUpperInvariant();
            options.SwaggerEndpoint(url, name);
        }
    });
}

// Create version set
var versionSet = app.NewApiVersionSet()
    .HasDeprecatedApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .HasApiVersion(new ApiVersion(3, 0, "beta")) // Pre-release
    .ReportApiVersions()
    .Build();

// Map versioned endpoints
var versionedApi = app.NewVersionedApi("API");
versionedApi.VersionSet = versionSet;

versionedApi.MapUserEndpointsV1();
versionedApi.MapUserEndpointsV2();
versionedApi.MapUserEndpointsV3Beta();

app.Run();
```

### Swagger Configuration for Versioning

```csharp
// MyApi.Api/Configuration/ConfigureSwaggerOptions.cs
using Asp.Versioning.ApiExplorer;
using Microsoft.Extensions.Options;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

namespace MyApi.Api.Configuration;

public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;

    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }

    public void Configure(SwaggerGenOptions options)
    {
        foreach (var description in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(
                description.GroupName,
                CreateInfoForApiVersion(description)
            );
        }
    }

    private static OpenApiInfo CreateInfoForApiVersion(
        ApiVersionDescription description)
    {
        var info = new OpenApiInfo
        {
            Title = "My API",
            Version = description.ApiVersion.ToString(),
            Description = "Production API with versioning",
            Contact = new OpenApiContact
            {
                Name = "API Support",
                Email = "support@example.com"
            }
        };

        if (description.IsDeprecated)
        {
            info.Description += " (DEPRECATED - Will be removed on 2025-12-31)";
        }

        return info;
    }
}
```

## Version-Specific DTOs

```csharp
// V1 DTO (simple)
namespace MyApi.Api.Contracts.Users.V1;

public record UserResponseV1(
    int Id,
    string Name,
    string Email
);

// V2 DTO (enhanced)
namespace MyApi.Api.Contracts.Users.V2;

public record UserResponseV2(
    int Id,
    string Name,
    string Email,
    DateTime CreatedAt,
    DateTime? UpdatedAt,
    UserPreferences Preferences  // New in V2
);

public record UserPreferences(
    string Language,
    string Timezone,
    bool EmailNotifications
);
```

## Migration Strategies

### Strategy 1: Gradual Migration

```csharp
// Shared service, version-specific adapters
public class UserService : IUserService
{
    public async Task<User> GetUserByIdAsync(int id, CancellationToken ct)
    {
        // Single implementation
        return await _repository.GetByIdAsync(id, ct);
    }
}

// V1 adapter
public class UserServiceV1Adapter : IUserServiceV1
{
    private readonly IUserService _userService;

    public async Task<UserResponseV1> GetUserByIdAsync(int id, CancellationToken ct)
    {
        var user = await _userService.GetUserByIdAsync(id, ct);
        return user.Adapt<UserResponseV1>(); // Simple mapping
    }
}

// V2 adapter with additional data
public class UserServiceV2Adapter : IUserServiceV2
{
    private readonly IUserService _userService;
    private readonly IPreferencesService _preferencesService;

    public async Task<UserResponseV2> GetUserByIdAsync(int id, CancellationToken ct)
    {
        var user = await _userService.GetUserByIdAsync(id, ct);
        var preferences = await _preferencesService.GetByUserIdAsync(id, ct);

        return new UserResponseV2(
            user.Id,
            user.Name,
            user.Email,
            user.CreatedAt,
            user.UpdatedAt,
            preferences.Adapt<UserPreferences>()
        );
    }
}
```

### Strategy 2: Version-Specific Services

```csharp
// Completely separate implementations
public class UserServiceV1 : IUserServiceV1
{
    // V1-specific logic
}

public class UserServiceV2 : IUserServiceV2
{
    // V2-specific logic with new features
}

// Register both
builder.Services.AddScoped<IUserServiceV1, UserServiceV1>();
builder.Services.AddScoped<IUserServiceV2, UserServiceV2>();
```

## API Versioning Best Practices

### Semantic Versioning

```csharp
// Major version: Breaking changes
new ApiVersion(2, 0)

// Minor version: New features, backward compatible
new ApiVersion(1, 1)

// Pre-release versions
new ApiVersion(3, 0, "beta")
new ApiVersion(3, 0, "rc1")
```

### Version Lifecycle

```csharp
public static class ApiVersions
{
    // Active versions
    public static readonly ApiVersion V2 = new(2, 0);
    public static readonly ApiVersion V3 = new(3, 0);

    // Deprecated versions (sunset in 6 months)
    public static readonly ApiVersion V1Deprecated = new(1, 0);

    // Beta versions
    public static readonly ApiVersion V4Beta = new(4, 0, "beta");
}

var versionSet = app.NewApiVersionSet()
    .HasDeprecatedApiVersion(ApiVersions.V1Deprecated)
    .HasApiVersion(ApiVersions.V2)
    .HasApiVersion(ApiVersions.V3)
    .HasApiVersion(ApiVersions.V4Beta)
    .Build();
```

## Testing Versioned Endpoints

```csharp
// MyApi.IntegrationTests/VersioningTests.cs
public class VersioningTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public VersioningTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetUsers_V1_ReturnsV1Response()
    {
        var response = await _client.GetAsync("/api/v1/users");

        response.EnsureSuccessStatusCode();

        var headers = response.Headers;
        headers.Should().ContainKey("api-supported-versions");
        headers.Should().ContainKey("api-deprecated-versions");
    }

    [Fact]
    public async Task GetUsers_WithHeaderVersion_ReturnsCorrectVersion()
    {
        _client.DefaultRequestHeaders.Add("X-API-Version", "2.0");

        var response = await _client.GetAsync("/api/users");

        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        content.Should().Contain("\"version\":\"2.0\"");
    }

    [Fact]
    public async Task GetUsers_DeprecatedVersion_ReturnsSunsetHeader()
    {
        var response = await _client.GetAsync("/api/v1/users");

        response.Headers.Should().ContainKey("Sunset");
    }
}
```

## Versioning Checklist

- [ ] URL-based versioning configured in route template
- [ ] Default version specified for unversioned requests
- [ ] API versions reported in response headers
- [ ] Version sets defined for each endpoint group
- [ ] Deprecated versions marked explicitly
- [ ] Sunset header included for deprecated endpoints
- [ ] Swagger configured to document all versions
- [ ] Version-specific DTOs created when schemas change
- [ ] Breaking changes increment major version
- [ ] New features use minor version increment
- [ ] Migration strategy defined for clients
- [ ] Version sunset timeline communicated
- [ ] Tests cover all supported versions

## Navigation

- **Previous**: `endpoints-and-routing.md` - Route organization
- **Next**: `openapi-swagger.md` - API documentation
- **Related**: `api-standards.md` - REST conventions

## References

- [ASP.NET Core API Versioning](https://github.com/dotnet/aspnet-api-versioning)
- [API Versioning Wiki](https://github.com/dotnet/aspnet-api-versioning/wiki)
- [Semantic Versioning](https://semver.org/)
- [RFC 8594 - Sunset Header](https://datatracker.ietf.org/doc/html/rfc8594)
- [Azure API Design - Versioning](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api)
