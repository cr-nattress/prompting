# OpenAPI and Swagger - .NET 8

> **File Purpose**: Configure Swashbuckle for OpenAPI documentation with operation filters, schema filters, and examples
> **Prerequisites**: `endpoints-and-routing.md` - Endpoint organization
> **Related Files**: `versioning.md`, `api-standards.md`
> **Agent Use Case**: Reference when setting up comprehensive API documentation with Swagger UI

## Quick Context

OpenAPI (formerly Swagger) provides machine-readable API documentation. .NET 8 uses Swashbuckle.AspNetCore to generate OpenAPI specifications and provides an interactive Swagger UI for testing endpoints.

**Microsoft References**:
- [Get started with Swashbuckle](https://learn.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle)
- [OpenAPI specification](https://spec.openapis.org/oas/v3.1.0)

## Install Packages

```bash
dotnet add package Swashbuckle.AspNetCore
dotnet add package Swashbuckle.AspNetCore.Annotations
```

## Basic Configuration

```csharp
// Program.cs
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Version = "v1",
        Title = "My API",
        Description = "Production-ready .NET 8 API",
        TermsOfService = new Uri("https://example.com/terms"),
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "support@example.com",
            Url = new Uri("https://example.com/support")
        },
        License = new OpenApiLicense
        {
            Name = "MIT",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
        options.RoutePrefix = string.Empty; // Serve UI at root
    });
}

app.Run();
```

## JWT Bearer Authentication in Swagger

```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. Enter 'Bearer' [space] and then your token.",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer",
        BearerFormat = "JWT"
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
                },
                Scheme = "oauth2",
                Name = "Bearer",
                In = ParameterLocation.Header
            },
            Array.Empty<string>()
        }
    });
});
```

## XML Documentation Comments

```csharp
// Enable in .csproj
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn>
</PropertyGroup>

// Use in code
/// <summary>
/// Retrieves a user by their unique identifier
/// </summary>
/// <param name="id">The user's unique identifier</param>
/// <param name="ct">Cancellation token</param>
/// <returns>The user if found, otherwise NotFound</returns>
/// <response code="200">Returns the requested user</response>
/// <response code="404">User not found</response>
app.MapGet("/api/users/{id:int}", async (
    int id,
    IUserRepository repo,
    CancellationToken ct) =>
{
    var user = await repo.GetByIdAsync(id, ct);
    return user is null ? Results.NotFound() : Results.Ok(user);
})
.WithName("GetUser")
.WithTags("Users")
.Produces<UserResponse>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);
```

## Operation Filters

```csharp
// MyApi.Api/Swagger/AddCorrelationIdHeaderFilter.cs
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

public class AddCorrelationIdHeaderFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters ??= new List<OpenApiParameter>();

        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Correlation-ID",
            In = ParameterLocation.Header,
            Required = false,
            Description = "Correlation ID for request tracing",
            Schema = new OpenApiSchema
            {
                Type = "string",
                Format = "uuid"
            }
        });
    }
}

// Register
builder.Services.AddSwaggerGen(options =>
{
    options.OperationFilter<AddCorrelationIdHeaderFilter>();
});
```

## Schema Filters

```csharp
// MyApi.Api/Swagger/EnumSchemaFilter.cs
using Microsoft.OpenApi.Any;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type.IsEnum)
        {
            schema.Enum.Clear();
            schema.Type = "string";
            schema.Format = null;

            foreach (var name in Enum.GetNames(context.Type))
            {
                schema.Enum.Add(new OpenApiString(name));
            }
        }
    }
}

// Register
builder.Services.AddSwaggerGen(options =>
{
    options.SchemaFilter<EnumSchemaFilter>();
});
```

## Request/Response Examples

```csharp
// MyApi.Api/Swagger/UserExamplesFilter.cs
using Microsoft.OpenApi.Any;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

public class UserExamplesFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (context.MethodInfo.Name == "CreateUser")
        {
            operation.RequestBody.Content["application/json"].Example =
                new OpenApiObject
                {
                    ["name"] = new OpenApiString("John Doe"),
                    ["email"] = new OpenApiString("john@example.com"),
                    ["password"] = new OpenApiString("SecurePass123!")
                };
        }

        if (operation.Responses.TryGetValue("200", out var response))
        {
            response.Content["application/json"].Example =
                new OpenApiObject
                {
                    ["id"] = new OpenApiInteger(1),
                    ["name"] = new OpenApiString("John Doe"),
                    ["email"] = new OpenApiString("john@example.com"),
                    ["createdAt"] = new OpenApiString("2024-01-15T10:30:00Z")
                };
        }
    }
}
```

## Versioned Swagger Documentation

```csharp
// See versioning.md for complete ConfigureSwaggerOptions implementation

builder.Services.AddSwaggerGen();
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>();

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
```

## Customization Options

```csharp
builder.Services.AddSwaggerGen(options =>
{
    // Custom schema IDs to avoid conflicts
    options.CustomSchemaIds(type => type.FullName);

    // Order actions alphabetically
    options.OrderActionsBy(apiDesc =>
        $"{apiDesc.ActionDescriptor.RouteValues["controller"]}_{apiDesc.HttpMethod}");

    // Group by tags
    options.TagActionsBy(apiDesc =>
        new[] { apiDesc.GroupName ?? "Default" });

    // Custom document filters
    options.DocumentFilter<LowercaseDocumentFilter>();
});

app.UseSwaggerUI(options =>
{
    options.DisplayRequestDuration();
    options.DocExpansion(Docfx.Plugins.DocExpansion.None);
    options.EnableDeepLinking();
    options.EnableFilter();
    options.ShowExtensions();
});
```

## Complete Production Example

```csharp
using System.Reflection;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Version = "v1",
        Title = "Production API",
        Description = "Comprehensive .NET 8 API with Swagger documentation",
        Contact = new OpenApiContact { Name = "Support", Email = "support@example.com" },
        License = new OpenApiLicense { Name = "MIT", Url = new Uri("https://opensource.org/licenses/MIT") }
    });

    // XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // JWT authentication
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });

    // Filters
    options.OperationFilter<AddCorrelationIdHeaderFilter>();
    options.SchemaFilter<EnumSchemaFilter>();
    options.CustomSchemaIds(type => type.FullName);
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger(options =>
    {
        options.RouteTemplate = "swagger/{documentName}/swagger.json";
    });

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1");
        options.RoutePrefix = string.Empty;
        options.DocumentTitle = "My API Documentation";
        options.DisplayRequestDuration();
    });
}

app.Run();
```

## References

- [Swashbuckle Documentation](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)
- [Microsoft: Get started with Swashbuckle](https://learn.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle)
