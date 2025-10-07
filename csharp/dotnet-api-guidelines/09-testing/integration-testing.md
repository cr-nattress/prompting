# Integration Testing with WebApplicationFactory and Testcontainers

> **Last Updated**: 2025-10-06
> **Target**: .NET 8, xUnit, Testcontainers, WebApplicationFactory
> **Complexity**: Medium-High
> **Time to Implement**: 2-3 hours

## Overview

This guide covers integration testing for .NET 8 Web APIs using WebApplicationFactory for in-memory API testing and Testcontainers for real database integration. It includes API contract tests, test isolation strategies, custom factories, database seeding, and complete working examples.

**Key Principles:**
- **Real Dependencies**: Use actual database, not mocks
- **Isolated**: Each test runs independently
- **Fast**: Docker containers start quickly with Testcontainers
- **Realistic**: Tests mimic production environment
- **Reliable**: Deterministic and repeatable

---

## Table of Contents

1. [WebApplicationFactory Setup](#webapplicationfactory-setup)
2. [Testcontainers for Database](#testcontainers-for-database)
3. [API Contract Tests](#api-contract-tests)
4. [Test Isolation Strategies](#test-isolation-strategies)
5. [Custom WebApplicationFactory](#custom-webapplicationfactory)
6. [Database Seeding for Tests](#database-seeding-for-tests)
7. [Complete Working Example](#complete-working-example)
8. [References](#references)

---

## WebApplicationFactory Setup

WebApplicationFactory creates an in-memory test server to test your API without deploying it.

### Project Setup

```bash
# Create integration test project
dotnet new xunit -n YourApi.IntegrationTests -o tests/YourApi.IntegrationTests

# Add necessary packages
cd tests/YourApi.IntegrationTests
dotnet add package Microsoft.AspNetCore.Mvc.Testing --version 8.0.0
dotnet add package FluentAssertions --version 6.12.0
dotnet add package Testcontainers --version 3.7.0
dotnet add package Testcontainers.MsSql --version 3.7.0
dotnet add package Testcontainers.PostgreSql --version 3.7.0
dotnet add package Respawn --version 6.2.0

# Add project reference
dotnet add reference ../../src/YourApi/YourApi.csproj
```

### Project File Configuration

**YourApi.IntegrationTests.csproj**:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <!-- Test Framework -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.4" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.6">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>

    <!-- Integration Testing -->
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />

    <!-- Testcontainers -->
    <PackageReference Include="Testcontainers" Version="3.7.0" />
    <PackageReference Include="Testcontainers.MsSql" Version="3.7.0" />
    <PackageReference Include="Testcontainers.PostgreSql" Version="3.7.0" />

    <!-- Database Reset -->
    <PackageReference Include="Respawn" Version="6.2.0" />

    <!-- Code Coverage -->
    <PackageReference Include="coverlet.collector" Version="6.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\YourApi\YourApi.csproj" />
  </ItemGroup>

  <!-- Global Usings -->
  <ItemGroup>
    <Using Include="Xunit" />
    <Using Include="FluentAssertions" />
    <Using Include="Microsoft.AspNetCore.Mvc.Testing" />
    <Using Include="System.Net.Http.Json" />
  </ItemGroup>
</Project>
```

### Basic WebApplicationFactory Test

```csharp
using Microsoft.AspNetCore.Mvc.Testing;

namespace YourApi.IntegrationTests;

public class BasicApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public BasicApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetHealth_ShouldReturnOk()
    {
        // Act
        var response = await _client.GetAsync("/health");

        // Assert
        response.Should().BeSuccessful();
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task GetSwagger_ShouldReturnSwaggerDocument()
    {
        // Act
        var response = await _client.GetAsync("/swagger/v1/swagger.json");

        // Assert
        response.Should().BeSuccessful();
        var content = await response.Content.ReadAsStringAsync();
        content.Should().Contain("openapi");
    }
}
```

### Making Program.cs Testable

Your API's `Program.cs` needs to be accessible to tests:

**Program.cs**:
```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapGet("/health", () => Results.Ok(new { status = "Healthy" }))
    .WithName("GetHealth");

app.Run();

// Make Program accessible to tests
public partial class Program { }
```

**Source**: [Microsoft Integration Testing](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) (2024)

---

## Testcontainers for Database

Testcontainers spin up real Docker containers for databases, ensuring your tests run against actual infrastructure.

### SQL Server Testcontainer

```csharp
using Testcontainers.MsSql;

namespace YourApi.IntegrationTests.Fixtures;

public class SqlServerFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container;

    public SqlServerFixture()
    {
        _container = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .WithPassword("YourStrong@Passw0rd")
            .WithCleanUp(true)
            .Build();
    }

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

### PostgreSQL Testcontainer

```csharp
using Testcontainers.PostgreSql;

namespace YourApi.IntegrationTests.Fixtures;

public class PostgreSqlFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container;

    public PostgreSqlFixture()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("testdb")
            .WithUsername("postgres")
            .WithPassword("postgres")
            .WithCleanUp(true)
            .Build();
    }

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

### Using Testcontainer in Tests

```csharp
namespace YourApi.IntegrationTests;

public class DatabaseTests : IClassFixture<SqlServerFixture>
{
    private readonly SqlServerFixture _dbFixture;

    public DatabaseTests(SqlServerFixture dbFixture)
    {
        _dbFixture = dbFixture;
    }

    [Fact]
    public async Task Database_ShouldBeReachable()
    {
        // Arrange
        await using var connection = new SqlConnection(_dbFixture.ConnectionString);

        // Act
        await connection.OpenAsync();

        // Assert
        connection.State.Should().Be(ConnectionState.Open);
    }
}
```

**Source**: [Testcontainers for .NET](https://dotnet.testcontainers.org/) (2024)

---

## API Contract Tests

API contract tests verify your endpoints return the correct HTTP status codes, response shapes, and content.

### GET Endpoint Tests

```csharp
namespace YourApi.IntegrationTests.Endpoints;

public class TodoEndpointsTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public TodoEndpointsTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetTodos_ShouldReturnOkWithEmptyArray()
    {
        // Act
        var response = await _client.GetAsync("/api/todos");

        // Assert
        response.Should().BeSuccessful();
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var todos = await response.Content.ReadFromJsonAsync<List<TodoResponse>>();
        todos.Should().NotBeNull();
        todos.Should().BeEmpty();
    }

    [Fact]
    public async Task GetTodoById_WhenExists_ShouldReturnOkWithTodo()
    {
        // Arrange - Create a todo first
        var createRequest = new CreateTodoRequest
        {
            Title = "Test Todo",
            Description = "Test Description"
        };
        var createResponse = await _client.PostAsJsonAsync("/api/todos", createRequest);
        var createdId = await createResponse.Content.ReadFromJsonAsync<Guid>();

        // Act
        var response = await _client.GetAsync($"/api/todos/{createdId}");

        // Assert
        response.Should().BeSuccessful();
        var todo = await response.Content.ReadFromJsonAsync<TodoResponse>();
        todo.Should().NotBeNull();
        todo!.Id.Should().Be(createdId);
        todo.Title.Should().Be("Test Todo");
        todo.Description.Should().Be("Test Description");
    }

    [Fact]
    public async Task GetTodoById_WhenNotExists_ShouldReturnNotFound()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();

        // Act
        var response = await _client.GetAsync($"/api/todos/{nonExistentId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

### POST Endpoint Tests

```csharp
public class CreateTodoTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public CreateTodoTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateTodo_WithValidRequest_ShouldReturnCreated()
    {
        // Arrange
        var request = new CreateTodoRequest
        {
            Title = "New Todo",
            Description = "New Description",
            Priority = Priority.High
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/todos", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var todoId = await response.Content.ReadFromJsonAsync<Guid>();
        todoId.Should().NotBeEmpty();

        // Verify the todo was actually created
        var getResponse = await _client.GetAsync($"/api/todos/{todoId}");
        getResponse.Should().BeSuccessful();
    }

    [Fact]
    public async Task CreateTodo_WithInvalidRequest_ShouldReturnBadRequest()
    {
        // Arrange - Missing required Title
        var request = new CreateTodoRequest
        {
            Title = "", // Invalid
            Description = "Description"
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/todos", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

        var problemDetails = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problemDetails.Should().NotBeNull();
        problemDetails!.Errors.Should().ContainKey("Title");
    }

    [Theory]
    [InlineData(null)]
    [InlineData("")]
    [InlineData("   ")]
    public async Task CreateTodo_WithInvalidTitle_ShouldReturnBadRequest(string invalidTitle)
    {
        // Arrange
        var request = new CreateTodoRequest { Title = invalidTitle };

        // Act
        var response = await _client.PostAsJsonAsync("/api/todos", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### PUT/PATCH Endpoint Tests

```csharp
public class UpdateTodoTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public UpdateTodoTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task UpdateTodo_WhenExists_ShouldReturnNoContent()
    {
        // Arrange - Create a todo
        var createRequest = new CreateTodoRequest { Title = "Original Title" };
        var createResponse = await _client.PostAsJsonAsync("/api/todos", createRequest);
        var todoId = await createResponse.Content.ReadFromJsonAsync<Guid>();

        var updateRequest = new UpdateTodoRequest
        {
            Title = "Updated Title",
            Description = "Updated Description"
        };

        // Act
        var response = await _client.PutAsJsonAsync($"/api/todos/{todoId}", updateRequest);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify the update
        var getResponse = await _client.GetAsync($"/api/todos/{todoId}");
        var todo = await getResponse.Content.ReadFromJsonAsync<TodoResponse>();
        todo!.Title.Should().Be("Updated Title");
        todo.Description.Should().Be("Updated Description");
    }

    [Fact]
    public async Task CompleteTodo_WhenExists_ShouldMarkAsCompleted()
    {
        // Arrange
        var createRequest = new CreateTodoRequest { Title = "Todo to Complete" };
        var createResponse = await _client.PostAsJsonAsync("/api/todos", createRequest);
        var todoId = await createResponse.Content.ReadFromJsonAsync<Guid>();

        // Act
        var response = await _client.PutAsync($"/api/todos/{todoId}/complete", null);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify completion
        var getResponse = await _client.GetAsync($"/api/todos/{todoId}");
        var todo = await getResponse.Content.ReadFromJsonAsync<TodoResponse>();
        todo!.Status.Should().Be("Completed");
        todo.CompletedAt.Should().NotBeNull();
    }
}
```

### DELETE Endpoint Tests

```csharp
public class DeleteTodoTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public DeleteTodoTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task DeleteTodo_WhenExists_ShouldReturnNoContent()
    {
        // Arrange - Create a todo
        var createRequest = new CreateTodoRequest { Title = "Todo to Delete" };
        var createResponse = await _client.PostAsJsonAsync("/api/todos", createRequest);
        var todoId = await createResponse.Content.ReadFromJsonAsync<Guid>();

        // Act
        var response = await _client.DeleteAsync($"/api/todos/{todoId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify deletion
        var getResponse = await _client.GetAsync($"/api/todos/{todoId}");
        getResponse.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task DeleteTodo_WhenNotExists_ShouldReturnNotFound()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();

        // Act
        var response = await _client.DeleteAsync($"/api/todos/{nonExistentId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

**Source**: [API Testing Best Practices](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests#basic-tests-with-the-default-webapplicationfactory) (2024)

---

## Test Isolation Strategies

Ensure each test runs independently without affecting others.

### Strategy 1: Respawn (Database Reset)

Respawn resets the database to a clean state between tests.

**Installation**:
```bash
dotnet add package Respawn --version 6.2.0
```

**Implementation**:
```csharp
using Respawn;
using Npgsql;

namespace YourApi.IntegrationTests.Fixtures;

public class DatabaseResetFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container;
    private Respawner _respawner = null!;

    public DatabaseResetFixture()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();
    }

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        // Initialize Respawner
        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();

        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = new[] { "public" },
            TablesToIgnore = new[] { new Table("__EFMigrationsHistory") }
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();
        await _respawner.ResetAsync(connection);
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

**Using Respawn in Tests**:
```csharp
public class IsolatedTodoTests : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    private readonly CustomWebApplicationFactory _factory;
    private readonly HttpClient _client;

    public IsolatedTodoTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public Task InitializeAsync() => Task.CompletedTask;

    public async Task DisposeAsync()
    {
        // Reset database after each test
        await _factory.ResetDatabaseAsync();
    }

    [Fact]
    public async Task Test1_CreatesData()
    {
        // This test creates data
        var request = new CreateTodoRequest { Title = "Test 1" };
        await _client.PostAsJsonAsync("/api/todos", request);
    }

    [Fact]
    public async Task Test2_StartsClean()
    {
        // This test starts with a clean database
        var response = await _client.GetAsync("/api/todos");
        var todos = await response.Content.ReadFromJsonAsync<List<TodoResponse>>();
        todos.Should().BeEmpty(); // Database was reset
    }
}
```

### Strategy 2: Unique Test Data per Test

```csharp
public class UniqueDataTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly string _testPrefix;

    public UniqueDataTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
        _testPrefix = Guid.NewGuid().ToString(); // Unique per test class
    }

    [Fact]
    public async Task Test1_WithUniqueData()
    {
        var request = new CreateTodoRequest
        {
            Title = $"{_testPrefix}-Todo-{Guid.NewGuid()}"
        };

        var response = await _client.PostAsJsonAsync("/api/todos", request);
        response.Should().BeSuccessful();
    }

    [Fact]
    public async Task Test2_WithDifferentUniqueData()
    {
        var request = new CreateTodoRequest
        {
            Title = $"{_testPrefix}-Todo-{Guid.NewGuid()}"
        };

        var response = await _client.PostAsJsonAsync("/api/todos", request);
        response.Should().BeSuccessful();
    }
}
```

### Strategy 3: Transaction Rollback (Not Recommended for API Tests)

```csharp
// This works for repository tests, but not for full API tests
// because WebApplicationFactory uses a separate scope
public class TransactionalTests : IDisposable
{
    private readonly DbContext _context;
    private readonly IDbContextTransaction _transaction;

    public TransactionalTests()
    {
        _context = CreateDbContext();
        _transaction = _context.Database.BeginTransaction();
    }

    public void Dispose()
    {
        _transaction.Rollback();
        _transaction.Dispose();
        _context.Dispose();
    }

    [Fact]
    public void Test_WithRollback()
    {
        // Changes will be rolled back
    }
}
```

**Source**: [Respawn Documentation](https://github.com/jbogard/Respawn) (2024)

---

## Custom WebApplicationFactory

Customize the factory to replace services, configure test database, and apply test-specific settings.

### Basic Custom Factory

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace YourApi.IntegrationTests.Fixtures;

public class CustomWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer;

    public CustomWebApplicationFactory()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithCleanUp(true)
            .Build();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the existing DbContext registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            // Add DbContext with test database
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseNpgsql(_dbContainer.GetConnectionString());
            });

            // Build service provider and apply migrations
            var serviceProvider = services.BuildServiceProvider();
            using var scope = serviceProvider.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.Migrate();
        });

        builder.UseEnvironment("Testing");
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

### Advanced Custom Factory with Service Overrides

```csharp
public class AdvancedWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer;
    private Respawner _respawner = null!;

    public AdvancedWebApplicationFactory()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();
    }

    public string ConnectionString => _dbContainer.GetConnectionString();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace DbContext
            RemoveService<DbContextOptions<ApplicationDbContext>>(services);
            services.AddDbContext<ApplicationDbContext>(options =>
                options.UseNpgsql(ConnectionString));

            // Replace external services with test doubles
            RemoveService<IEmailService>(services);
            services.AddScoped<IEmailService, FakeEmailService>();

            RemoveService<IBlobStorageService>(services);
            services.AddScoped<IBlobStorageService, InMemoryBlobStorageService>();

            // Override configuration
            services.Configure<AppSettings>(settings =>
            {
                settings.EnableEmailNotifications = false;
                settings.MaxUploadSizeMb = 1; // Smaller for tests
            });

            // Apply migrations
            var serviceProvider = services.BuildServiceProvider();
            using var scope = serviceProvider.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.Migrate();
        });

        builder.UseEnvironment("Testing");
    }

    private static void RemoveService<T>(IServiceCollection services)
    {
        var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(T));
        if (descriptor != null)
        {
            services.Remove(descriptor);
        }
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        // Initialize Respawner
        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();

        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = new[] { "public" },
            TablesToIgnore = new[] { new Table("__EFMigrationsHistory") }
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();
        await _respawner.ResetAsync(connection);
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

### Test Doubles for External Services

```csharp
// Fake email service for testing
public class FakeEmailService : IEmailService
{
    public List<EmailMessage> SentEmails { get; } = new();

    public Task SendEmailAsync(string to, string subject, string body, CancellationToken ct = default)
    {
        SentEmails.Add(new EmailMessage
        {
            To = to,
            Subject = subject,
            Body = body,
            SentAt = DateTime.UtcNow
        });

        return Task.CompletedTask;
    }

    public void Clear() => SentEmails.Clear();
}

// In-memory blob storage for testing
public class InMemoryBlobStorageService : IBlobStorageService
{
    private readonly Dictionary<string, byte[]> _storage = new();

    public Task<string> UploadAsync(Stream stream, string fileName, CancellationToken ct = default)
    {
        using var memoryStream = new MemoryStream();
        stream.CopyTo(memoryStream);
        _storage[fileName] = memoryStream.ToArray();
        return Task.FromResult(fileName);
    }

    public Task<Stream> DownloadAsync(string fileName, CancellationToken ct = default)
    {
        if (!_storage.ContainsKey(fileName))
            throw new FileNotFoundException();

        var stream = new MemoryStream(_storage[fileName]);
        return Task.FromResult<Stream>(stream);
    }

    public Task DeleteAsync(string fileName, CancellationToken ct = default)
    {
        _storage.Remove(fileName);
        return Task.CompletedTask;
    }
}
```

**Source**: [Customize WebApplicationFactory](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests#customize-webapplicationfactory) (2024)

---

## Database Seeding for Tests

Seed test data to set up specific scenarios for your integration tests.

### Seeding Extension Methods

```csharp
namespace YourApi.IntegrationTests.Helpers;

public static class DatabaseSeeder
{
    public static async Task SeedTodosAsync(this ApplicationDbContext context, params Todo[] todos)
    {
        await context.Todos.AddRangeAsync(todos);
        await context.SaveChangesAsync();
    }

    public static async Task SeedUsersAsync(this ApplicationDbContext context, params User[] users)
    {
        await context.Users.AddRangeAsync(users);
        await context.SaveChangesAsync();
    }

    public static async Task<ApplicationDbContext> GetDbContextAsync(this WebApplicationFactory<Program> factory)
    {
        var scope = factory.Services.CreateScope();
        return scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    }
}
```

### Using Seeded Data in Tests

```csharp
public class SeededDataTests : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    private readonly CustomWebApplicationFactory _factory;
    private readonly HttpClient _client;
    private ApplicationDbContext _context = null!;

    public SeededDataTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public async Task InitializeAsync()
    {
        _context = await _factory.GetDbContextAsync();
    }

    public async Task DisposeAsync()
    {
        await _factory.ResetDatabaseAsync();
    }

    [Fact]
    public async Task GetTodos_WithSeededData_ShouldReturnAll()
    {
        // Arrange - Seed test data
        await _context.SeedTodosAsync(
            Todo.Create("Todo 1", null, Priority.High),
            Todo.Create("Todo 2", "Description", Priority.Low),
            Todo.Create("Todo 3", null, Priority.Medium)
        );

        // Act
        var response = await _client.GetAsync("/api/todos");

        // Assert
        response.Should().BeSuccessful();
        var todos = await response.Content.ReadFromJsonAsync<List<TodoResponse>>();
        todos.Should().HaveCount(3);
        todos.Should().Contain(t => t.Title == "Todo 1");
        todos.Should().Contain(t => t.Title == "Todo 2");
        todos.Should().Contain(t => t.Title == "Todo 3");
    }

    [Fact]
    public async Task GetTodoById_WithSeededData_ShouldReturnSpecificTodo()
    {
        // Arrange
        var todo = Todo.Create("Specific Todo", "Find me", Priority.High);
        await _context.SeedTodosAsync(todo);

        // Act
        var response = await _client.GetAsync($"/api/todos/{todo.Id}");

        // Assert
        response.Should().BeSuccessful();
        var result = await response.Content.ReadFromJsonAsync<TodoResponse>();
        result!.Title.Should().Be("Specific Todo");
        result.Description.Should().Be("Find me");
    }

    [Fact]
    public async Task UpdateTodo_WithExistingTodo_ShouldModifyCorrectly()
    {
        // Arrange
        var todo = Todo.Create("Original Title", null, Priority.Low);
        await _context.SeedTodosAsync(todo);

        var updateRequest = new UpdateTodoRequest
        {
            Title = "Updated Title",
            Description = "New Description",
            Priority = Priority.High
        };

        // Act
        var response = await _client.PutAsJsonAsync($"/api/todos/{todo.Id}", updateRequest);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify in database
        var updated = await _context.Todos.FindAsync(todo.Id);
        updated!.Title.Should().Be("Updated Title");
        updated.Description.Should().Be("New Description");
        updated.Priority.Should().Be(Priority.High);
    }
}
```

### Test Data Builders for Seeding

```csharp
public class TodoTestDataBuilder
{
    public static Todo CreateDefault() =>
        Todo.Create("Default Todo", "Default Description", Priority.Medium);

    public static Todo CreateHighPriority() =>
        Todo.Create("High Priority Todo", null, Priority.High);

    public static Todo CreateCompleted()
    {
        var todo = Todo.Create("Completed Todo", null, Priority.Low);
        todo.Complete();
        return todo;
    }

    public static List<Todo> CreateMultiple(int count) =>
        Enumerable.Range(1, count)
            .Select(i => Todo.Create($"Todo {i}", $"Description {i}", Priority.Medium))
            .ToList();
}

// Usage
[Fact]
public async Task GetTodos_WithMultipleItems_ShouldReturnPagedResults()
{
    // Arrange
    var todos = TodoTestDataBuilder.CreateMultiple(50);
    await _context.SeedTodosAsync(todos.ToArray());

    // Act
    var response = await _client.GetAsync("/api/todos?page=1&pageSize=10");

    // Assert
    response.Should().BeSuccessful();
    var result = await response.Content.ReadFromJsonAsync<PagedResult<TodoResponse>>();
    result!.Items.Should().HaveCount(10);
    result.TotalCount.Should().Be(50);
    result.PageNumber.Should().Be(1);
}
```

---

## Complete Working Example

Here's a complete end-to-end example with all components integrated.

### 1. Custom Factory

**CustomWebApplicationFactory.cs**:
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;
using Respawn;
using Testcontainers.PostgreSql;

namespace YourApi.IntegrationTests.Fixtures;

public class CustomWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer;
    private Respawner _respawner = null!;

    public CustomWebApplicationFactory()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("testdb")
            .WithUsername("postgres")
            .WithPassword("postgres")
            .WithCleanUp(true)
            .Build();
    }

    public string ConnectionString => _dbContainer.GetConnectionString();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove existing DbContext
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            // Add test DbContext
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseNpgsql(ConnectionString);
                options.EnableSensitiveDataLogging();
                options.EnableDetailedErrors();
            });

            // Replace external services
            ReplaceService<IEmailService, FakeEmailService>(services);
            ReplaceService<IBlobStorageService, InMemoryBlobStorageService>(services);

            // Build and apply migrations
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.Migrate();
        });

        builder.UseEnvironment("Testing");
    }

    private static void ReplaceService<TService, TImplementation>(IServiceCollection services)
        where TService : class
        where TImplementation : class, TService
    {
        var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(TService));
        if (descriptor != null)
        {
            services.Remove(descriptor);
        }
        services.AddScoped<TService, TImplementation>();
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();

        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = new[] { "public" },
            TablesToIgnore = new[] { new Table("__EFMigrationsHistory") }
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var connection = new NpgsqlConnection(ConnectionString);
        await connection.OpenAsync();
        await _respawner.ResetAsync(connection);
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

### 2. Base Test Class

**IntegrationTestBase.cs**:
```csharp
namespace YourApi.IntegrationTests;

public abstract class IntegrationTestBase : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    protected readonly CustomWebApplicationFactory Factory;
    protected readonly HttpClient Client;
    protected ApplicationDbContext DbContext = null!;

    protected IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false,
            BaseAddress = new Uri("http://localhost")
        });
    }

    public async Task InitializeAsync()
    {
        var scope = Factory.Services.CreateScope();
        DbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        await Task.CompletedTask;
    }

    public async Task DisposeAsync()
    {
        await Factory.ResetDatabaseAsync();
    }

    protected async Task<T> GetDbEntityAsync<T>(Guid id) where T : class
    {
        return (await DbContext.Set<T>().FindAsync(id))!;
    }

    protected async Task SeedAsync<T>(params T[] entities) where T : class
    {
        await DbContext.Set<T>().AddRangeAsync(entities);
        await DbContext.SaveChangesAsync();
    }
}
```

### 3. Complete Test Suite

**TodoEndpointsIntegrationTests.cs**:
```csharp
namespace YourApi.IntegrationTests.Endpoints;

public class TodoEndpointsIntegrationTests : IntegrationTestBase
{
    public TodoEndpointsIntegrationTests(CustomWebApplicationFactory factory)
        : base(factory)
    {
    }

    public class GetEndpoints : TodoEndpointsIntegrationTests
    {
        public GetEndpoints(CustomWebApplicationFactory factory) : base(factory) { }

        [Fact]
        public async Task GetAllTodos_WithNoData_ShouldReturnEmptyArray()
        {
            // Act
            var response = await Client.GetAsync("/api/todos");

            // Assert
            response.Should().BeSuccessful();
            var todos = await response.Content.ReadFromJsonAsync<List<TodoResponse>>();
            todos.Should().NotBeNull().And.BeEmpty();
        }

        [Fact]
        public async Task GetAllTodos_WithData_ShouldReturnAllTodos()
        {
            // Arrange
            await SeedAsync(
                Todo.Create("Todo 1", null, Priority.High),
                Todo.Create("Todo 2", "Desc 2", Priority.Low),
                Todo.Create("Todo 3", null, Priority.Medium)
            );

            // Act
            var response = await Client.GetAsync("/api/todos");

            // Assert
            response.Should().BeSuccessful();
            var todos = await response.Content.ReadFromJsonAsync<List<TodoResponse>>();
            todos.Should().HaveCount(3);
        }

        [Fact]
        public async Task GetTodoById_WhenExists_ShouldReturnTodo()
        {
            // Arrange
            var todo = Todo.Create("Existing Todo", "Description", Priority.High);
            await SeedAsync(todo);

            // Act
            var response = await Client.GetAsync($"/api/todos/{todo.Id}");

            // Assert
            response.Should().BeSuccessful();
            var result = await response.Content.ReadFromJsonAsync<TodoResponse>();
            result.Should().NotBeNull();
            result!.Id.Should().Be(todo.Id);
            result.Title.Should().Be("Existing Todo");
            result.Priority.Should().Be("High");
        }

        [Fact]
        public async Task GetTodoById_WhenNotExists_ShouldReturn404()
        {
            // Arrange
            var nonExistentId = Guid.NewGuid();

            // Act
            var response = await Client.GetAsync($"/api/todos/{nonExistentId}");

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.NotFound);
        }
    }

    public class CreateEndpoints : TodoEndpointsIntegrationTests
    {
        public CreateEndpoints(CustomWebApplicationFactory factory) : base(factory) { }

        [Fact]
        public async Task CreateTodo_WithValidData_ShouldReturn201()
        {
            // Arrange
            var request = new CreateTodoRequest
            {
                Title = "New Todo",
                Description = "New Description",
                Priority = Priority.High
            };

            // Act
            var response = await Client.PostAsJsonAsync("/api/todos", request);

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.Created);
            response.Headers.Location.Should().NotBeNull();

            var todoId = await response.Content.ReadFromJsonAsync<Guid>();
            todoId.Should().NotBeEmpty();

            // Verify in database
            var todo = await GetDbEntityAsync<Todo>(todoId);
            todo.Should().NotBeNull();
            todo.Title.Should().Be("New Todo");
            todo.Priority.Should().Be(Priority.High);
        }

        [Theory]
        [InlineData(null, "Description")]
        [InlineData("", "Description")]
        [InlineData("   ", "Description")]
        public async Task CreateTodo_WithInvalidTitle_ShouldReturn400(
            string invalidTitle, string description)
        {
            // Arrange
            var request = new CreateTodoRequest
            {
                Title = invalidTitle,
                Description = description
            };

            // Act
            var response = await Client.PostAsJsonAsync("/api/todos", request);

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
            var problemDetails = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
            problemDetails!.Errors.Should().ContainKey("Title");
        }
    }

    public class UpdateEndpoints : TodoEndpointsIntegrationTests
    {
        public UpdateEndpoints(CustomWebApplicationFactory factory) : base(factory) { }

        [Fact]
        public async Task UpdateTodo_WithValidData_ShouldReturn204()
        {
            // Arrange
            var todo = Todo.Create("Original", "Original Desc", Priority.Low);
            await SeedAsync(todo);

            var request = new UpdateTodoRequest
            {
                Title = "Updated",
                Description = "Updated Desc",
                Priority = Priority.High
            };

            // Act
            var response = await Client.PutAsJsonAsync($"/api/todos/{todo.Id}", request);

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.NoContent);

            // Verify in database
            var updated = await GetDbEntityAsync<Todo>(todo.Id);
            updated.Title.Should().Be("Updated");
            updated.Description.Should().Be("Updated Desc");
            updated.Priority.Should().Be(Priority.High);
        }

        [Fact]
        public async Task CompleteTodo_WhenNotCompleted_ShouldMarkAsCompleted()
        {
            // Arrange
            var todo = Todo.Create("Todo to Complete", null, Priority.Medium);
            await SeedAsync(todo);

            // Act
            var response = await Client.PutAsync($"/api/todos/{todo.Id}/complete", null);

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.NoContent);

            // Verify in database
            var completed = await GetDbEntityAsync<Todo>(todo.Id);
            completed.Status.Should().Be(TodoStatus.Completed);
            completed.CompletedAt.Should().NotBeNull();
        }

        [Fact]
        public async Task CompleteTodo_WhenAlreadyCompleted_ShouldReturn400()
        {
            // Arrange
            var todo = Todo.Create("Already Completed", null, Priority.High);
            todo.Complete();
            await SeedAsync(todo);

            // Act
            var response = await Client.PutAsync($"/api/todos/{todo.Id}/complete", null);

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        }
    }

    public class DeleteEndpoints : TodoEndpointsIntegrationTests
    {
        public DeleteEndpoints(CustomWebApplicationFactory factory) : base(factory) { }

        [Fact]
        public async Task DeleteTodo_WhenExists_ShouldReturn204()
        {
            // Arrange
            var todo = Todo.Create("To Delete", null, Priority.Low);
            await SeedAsync(todo);

            // Act
            var response = await Client.DeleteAsync($"/api/todos/{todo.Id}");

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.NoContent);

            // Verify deletion
            var deleted = await DbContext.Todos.FindAsync(todo.Id);
            deleted.Should().BeNull();
        }

        [Fact]
        public async Task DeleteTodo_WhenNotExists_ShouldReturn404()
        {
            // Act
            var response = await Client.DeleteAsync($"/api/todos/{Guid.NewGuid()}");

            // Assert
            response.StatusCode.Should().Be(HttpStatusCode.NotFound);
        }
    }
}
```

### 4. Running the Tests

```bash
# Ensure Docker is running
docker --version

# Run all integration tests
dotnet test --filter "Category=Integration"

# Run specific test class
dotnet test --filter "FullyQualifiedName~TodoEndpointsIntegrationTests"

# Run with detailed output
dotnet test --logger "console;verbosity=detailed"

# Generate coverage report
dotnet test --collect:"XPlat Code Coverage"
```

**Source**: [Integration Testing in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) (2024)

---

## References

### Primary Sources

1. **Microsoft Integration Testing**
   https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests
   Last accessed: 2024-10
   *Comprehensive guide to integration testing in ASP.NET Core*

2. **WebApplicationFactory Documentation**
   https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1
   Last accessed: 2024-10
   *Official WebApplicationFactory API documentation*

3. **Testcontainers for .NET**
   https://dotnet.testcontainers.org/
   Last accessed: 2024-10
   *Official Testcontainers library for .NET*

4. **Respawn Documentation**
   https://github.com/jbogard/Respawn
   Last accessed: 2024-10
   *Database reset library for integration tests*

5. **xUnit Documentation**
   https://xunit.net/docs/getting-started/netcore/cmdline
   Last accessed: 2024-10
   *xUnit testing framework documentation*

6. **FluentAssertions**
   https://fluentassertions.com/
   Last accessed: 2024-10
   *Assertion library for more readable tests*

### Related Documentation

- [Testing Strategy](./testing-strategy.md) - Overall testing approach
- [Unit Testing](./unit-testing.md) - Unit testing patterns with xUnit
- [Clean Architecture](../02-architecture/clean-architecture.md) - Testable architecture design
- [EF Core Setup](../03-infrastructure/ef-core-setup.md) - Database configuration
- [CI/CD Pipeline](../10-deployment/ci-cd.md) - Automated test execution

---

**Next Steps**:
- Review [testing-strategy.md](./testing-strategy.md) for test distribution guidelines
- Read [unit-testing.md](./unit-testing.md) for unit testing patterns
- Check [../10-deployment/ci-cd.md](../10-deployment/ci-cd.md) for CI/CD integration
