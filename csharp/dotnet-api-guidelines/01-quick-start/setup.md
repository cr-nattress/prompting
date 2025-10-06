# Project Setup - .NET 8 Web API

> **File Purpose**: Scaffold a production-ready .NET 8 Web API project structure
> **Prerequisites**: .NET 8 SDK installed, basic CLI familiarity
> **Related Files**: `project-structure.md`, `minimal-api.md`
> **Agent Use Case**: Reference when creating a new Web API project from scratch

## 🎯 Quick Context

This guide walks through scaffolding a .NET 8 Web API using Clean Architecture with four layers: API, Application, Domain, and Infrastructure. You'll create a solution with proper project references and modern C# defaults (nullable reference types, implicit usings).

## Prerequisites

**Required**:
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) (verify with `dotnet --version`)
- Code editor (VS Code, Visual Studio 2022, Rider)

**Verify Installation**:
```bash
dotnet --version
# Expected output: 8.0.x or higher
```

## Project Scaffolding Commands

### Step 1: Create Solution and Projects

```bash
# Create solution directory
mkdir Demo.Api
cd Demo.Api

# Create solution file
dotnet new sln -n Demo

# Create API project (entry point)
dotnet new webapi -n Demo.Api --use-minimal-apis

# Create class libraries for layers
dotnet new classlib -n Demo.Application
dotnet new classlib -n Demo.Domain
dotnet new classlib -n Demo.Infrastructure

# Add projects to solution
dotnet sln add Demo.Api
dotnet sln add Demo.Application
dotnet sln add Demo.Domain
dotnet sln add Demo.Infrastructure
```

**Why**: This creates a Clean Architecture structure where dependencies flow inward (API → Application → Domain, Infrastructure → Domain).

### Step 2: Configure Project References

```bash
# API references Application and Infrastructure (composition root)
dotnet add Demo.Api reference Demo.Application Demo.Infrastructure

# Application references Domain (business logic orchestration)
dotnet add Demo.Application reference Demo.Domain

# Infrastructure references Domain (implements domain interfaces)
dotnet add Demo.Infrastructure reference Demo.Domain
```

**Dependency Flow**:
```
Domain (no dependencies - pure business logic)
   ↑
   ├─ Application (orchestrates use cases)
   ↑
   └─ Infrastructure (implements data access, external services)
   ↑
   API (composes everything, handles HTTP)
```

### Step 3: Enable Modern C# Features

Edit each `.csproj` file to ensure these properties:

**Demo.Api/Demo.Api.csproj**:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

**Demo.Application/Demo.Application.csproj** (and Domain, Infrastructure):
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

**What these do**:
- `Nullable>enable`: Enables nullable reference types (prevents null reference exceptions)
- `ImplicitUsings>enable`: Automatically imports common namespaces
- `TreatWarningsAsErrors>true`: Forces clean code (no warnings tolerated)

### Step 4: Add Essential NuGet Packages

**API Layer**:
```bash
cd Demo.Api
dotnet add package Swashbuckle.AspNetCore --version 6.5.0
dotnet add package Serilog.AspNetCore --version 8.0.1
dotnet add package Serilog.Sinks.Console --version 5.0.1
```

**Infrastructure Layer** (if using EF Core):
```bash
cd ../Demo.Infrastructure
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 8.0.0
```

**Application Layer** (if using MediatR for CQRS):
```bash
cd ../Demo.Application
dotnet add package MediatR --version 12.2.0
dotnet add package FluentValidation --version 11.9.0
```

## Initial Program.cs

Replace the generated `Demo.Api/Program.cs` with this minimal setup:

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateLogger();

builder.Host.UseSerilog();

// Add services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure HTTP pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Health check endpoint
app.MapGet("/health", () => Results.Ok(new { status = "Healthy" }))
   .WithName("HealthCheck")
   .WithTags("Health");

app.Run();

// Required for WebApplicationFactory in integration tests
public partial class Program { }
```

**Key Features**:
- Serilog for structured logging
- Swagger in development mode only
- Health check endpoint at `/health`
- `public partial class Program` enables integration testing

## appsettings.json Configuration

Replace `Demo.Api/appsettings.json`:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    }
  },
  "AllowedHosts": "*"
}
```

## Verify Setup

```bash
# Build solution
dotnet build

# Run API
cd Demo.Api
dotnet run

# Expected output:
# info: Microsoft.Hosting.Lifetime[14]
#       Now listening on: https://localhost:7001
# info: Microsoft.Hosting.Lifetime[14]
#       Now listening on: http://localhost:5000
```

**Test Endpoints**:
- Health: `https://localhost:7001/health`
- Swagger UI: `https://localhost:7001/swagger`

## Common Mistakes

❌ **Mistake 1**: Circular dependencies (e.g., Domain references Infrastructure)
✅ **Solution**: Always follow dependency rule (outer layers depend on inner layers, never reverse)

❌ **Mistake 2**: Putting business logic in API controllers/endpoints
✅ **Solution**: Keep endpoints thin - delegate to Application layer services

❌ **Mistake 3**: Not enabling nullable reference types
✅ **Solution**: Always set `<Nullable>enable</Nullable>` in all projects

## Next Steps

1. **Define Domain Entities**: Create models in `Demo.Domain/Entities/`
2. **Add First Endpoint**: Follow `minimal-api.md` to create your first API endpoint
3. **Configure Database**: See `../04-data-persistence/ef-core-setup.md`

## File Structure Created

```
Demo.Api/
├── Demo.sln
├── Demo.Api/
│   ├── Demo.Api.csproj
│   ├── Program.cs
│   ├── appsettings.json
│   └── Properties/
│       └── launchSettings.json
├── Demo.Application/
│   └── Demo.Application.csproj
├── Demo.Domain/
│   └── Demo.Domain.csproj
└── Demo.Infrastructure/
    └── Demo.Infrastructure.csproj
```

---

## Navigation
- **Next**: `minimal-api.md` - Create your first endpoint
- **Up**: `../00-overview.md`

## See Also
- `project-structure.md` - Detailed folder organization
- `../02-core-concepts/dependency-injection.md` - DI patterns
- `../12-reference/file-tree.md` - Complete project structure
