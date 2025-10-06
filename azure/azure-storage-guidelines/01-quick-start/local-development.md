# Local Development with Azurite

> **File Purpose**: Configure Azurite emulator for local Azure Storage development
> **Prerequisites**: Node.js/npm or Docker installed
> **Agent Use Case**: Reference when setting up local dev environment

## Quick Context

Azurite is the official Azure Storage emulator that runs locally and emulates Blob, Queue, and Table services. It eliminates the need for cloud storage accounts during development, reducing costs and enabling offline work. Azurite supports the same SDKs and APIs as Azure Storage.

**Key principle**: Use Azurite for local dev, DefaultAzureCredential for cloud environments. Configuration should swap seamlessly.

## Installation Options

### Option 1: NPM (Cross-platform)

```bash
# Install globally
npm install -g azurite

# Start all services
azurite --silent --location ./azurite-data --debug ./azurite-debug.log

# Or start specific services
azurite-blob --blobPort 10000 --location ./azurite-data
azurite-table --tablePort 10002 --location ./azurite-data
```

**Why it works**: Runs as background process. Data persists to `./azurite-data` directory.

### Option 2: Docker (Recommended for CI/CD)

```bash
# Run Azurite container
docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 \
  -v ./azurite-data:/data \
  mcr.microsoft.com/azure-storage/azurite \
  azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0 --location /data

# Or use docker-compose
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  azurite:
    image: mcr.microsoft.com/azure-storage/azurite
    ports:
      - "10000:10000"  # Blob
      - "10001:10001"  # Queue
      - "10002:10002"  # Table
    volumes:
      - ./azurite-data:/data
    command: azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0 --location /data --loose
EOF

docker-compose up -d
```

**Why it works**: Isolated environment. `--loose` mode relaxes validation for faster testing.

### Option 3: Visual Studio / VS Code Extension

- Visual Studio: Tools → Extensions → Azurite
- VS Code: Search "Azurite" in Extensions → Install → Open Command Palette → "Azurite: Start"

## Connection Strings

### Default Azurite Connection String

```bash
# Well-known connection string (use for local dev only)
UseDevelopmentStorage=true

# Explicit connection string
DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;QueueEndpoint=http://127.0.0.1:10001/devstoreaccount1;TableEndpoint=http://127.0.0.1:10002/devstoreaccount1;
```

**Security note**: This account key is well-known and publicly documented. Never use in production.

### Environment-Specific Configuration

**appsettings.Development.json** (.NET):

```json
{
  "Azure": {
    "Storage": {
      "UseAzurite": true,
      "ConnectionString": "UseDevelopmentStorage=true"
    }
  }
}
```

**appsettings.Production.json**:

```json
{
  "Azure": {
    "Storage": {
      "UseAzurite": false,
      "BlobServiceUri": "https://storacct....blob.core.windows.net",
      "TableServiceUri": "https://storacct....table.core.windows.net"
    }
  }
}
```

## .NET Client Configuration

### Conditional Client Initialization

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;

var builder = WebApplication.CreateBuilder(args);

// Check if using Azurite
var useAzurite = builder.Configuration.GetValue<bool>("Azure:Storage:UseAzurite");

if (useAzurite)
{
    // Azurite: use connection string
    var connectionString = builder.Configuration["Azure:Storage:ConnectionString"]
        ?? "UseDevelopmentStorage=true";

    builder.Services.AddSingleton(sp => new BlobServiceClient(connectionString));
    builder.Services.AddSingleton(sp => new TableServiceClient(connectionString));
}
else
{
    // Production: use Managed Identity
    builder.Services.AddSingleton<DefaultAzureCredential>();

    builder.Services.AddSingleton(sp =>
    {
        var credential = sp.GetRequiredService<DefaultAzureCredential>();
        var uri = new Uri(builder.Configuration["Azure:Storage:BlobServiceUri"]!);
        return new BlobServiceClient(uri, credential);
    });

    builder.Services.AddSingleton(sp =>
    {
        var credential = sp.GetRequiredService<DefaultAzureCredential>();
        var uri = new Uri(builder.Configuration["Azure:Storage:TableServiceUri"]!);
        return new TableServiceClient(uri, credential);
    });
}
```

**Why it works**: Same application code works in both local and cloud environments. Configuration drives behavior.

### Alternative: Environment Variable Detection

```csharp
// Detect Azurite automatically by checking environment
var blobServiceClient = Environment.GetEnvironmentVariable("AZURE_STORAGE_CONNECTION_STRING") != null
    ? new BlobServiceClient(Environment.GetEnvironmentVariable("AZURE_STORAGE_CONNECTION_STRING"))
    : new BlobServiceClient(new Uri(builder.Configuration["Azure:Storage:BlobServiceUri"]!), new DefaultAzureCredential());
```

## Seeding Test Data

### Create Containers and Tables

```csharp
// Seed.cs
using Azure.Storage.Blobs;
using Azure.Data.Tables;

public static class AzuriteSeeder
{
    public static async Task SeedAsync(BlobServiceClient blobService, TableServiceClient tableService)
    {
        // Create containers
        var containers = new[] { "app-data", "images", "logs" };
        foreach (var containerName in containers)
        {
            await blobService.GetBlobContainerClient(containerName).CreateIfNotExistsAsync();
        }

        // Create tables
        var tables = new[] { "Users", "Sessions", "AuditLogs" };
        foreach (var tableName in tables)
        {
            await tableService.GetTableClient(tableName).CreateIfNotExistsAsync();
        }

        // Upload sample blobs
        var container = blobService.GetBlobContainerClient("app-data");
        var blob = container.GetBlobClient("sample.txt");
        await blob.UploadAsync(new BinaryData("Hello Azurite"), overwrite: true);

        Console.WriteLine("Azurite seeded successfully");
    }
}

// Call from Program.cs
if (app.Environment.IsDevelopment())
{
    var blobService = app.Services.GetRequiredService<BlobServiceClient>();
    var tableService = app.Services.GetRequiredService<TableServiceClient>();
    await AzuriteSeeder.SeedAsync(blobService, tableService);
}
```

### Bash Seeding Script

```bash
#!/bin/bash
# seed-azurite.sh

CONN_STR="UseDevelopmentStorage=true"

# Create containers
az storage container create --name app-data --connection-string "$CONN_STR"
az storage container create --name images --connection-string "$CONN_STR"
az storage container create --name logs --connection-string "$CONN_STR"

# Create tables
az storage table create --name Users --connection-string "$CONN_STR"
az storage table create --name Sessions --connection-string "$CONN_STR"

# Upload sample data
echo "Sample data" > sample.txt
az storage blob upload \
  --container-name app-data \
  --name sample.txt \
  --file sample.txt \
  --connection-string "$CONN_STR"

rm sample.txt
echo "Azurite seeded"
```

## Testing Against Azurite

### Unit Test Example

```csharp
using Xunit;
using Azure.Storage.Blobs;

public class BlobStorageTests
{
    private readonly BlobServiceClient _blobServiceClient;

    public BlobStorageTests()
    {
        // Use Azurite connection string
        _blobServiceClient = new BlobServiceClient("UseDevelopmentStorage=true");
    }

    [Fact]
    public async Task UploadBlob_ShouldSucceed()
    {
        // Arrange
        var containerClient = _blobServiceClient.GetBlobContainerClient("test-container");
        await containerClient.CreateIfNotExistsAsync();
        var blobClient = containerClient.GetBlobClient("test.txt");

        // Act
        await blobClient.UploadAsync(new BinaryData("test content"), overwrite: true);

        // Assert
        var exists = await blobClient.ExistsAsync();
        Assert.True(exists.Value);

        // Cleanup
        await containerClient.DeleteAsync();
    }
}
```

### Integration Test with Test Containers (Advanced)

```csharp
using Testcontainers.Azurite;
using Xunit;

public class AzuriteIntegrationTests : IAsyncLifetime
{
    private readonly AzuriteContainer _azuriteContainer;
    private BlobServiceClient _blobServiceClient = null!;

    public AzuriteIntegrationTests()
    {
        _azuriteContainer = new AzuriteBuilder().Build();
    }

    public async Task InitializeAsync()
    {
        await _azuriteContainer.StartAsync();
        _blobServiceClient = new BlobServiceClient(_azuriteContainer.GetConnectionString());
    }

    public async Task DisposeAsync()
    {
        await _azuriteContainer.DisposeAsync();
    }

    [Fact]
    public async Task Container_ShouldBeCreated()
    {
        // Test code
    }
}
```

## Azurite Limitations

**What Azurite does NOT support**:
- Geo-replication (GZRS, RA-GZRS)
- Private endpoints (test network config separately)
- Managed Identity authentication (use connection strings for Azurite)
- Some premium features (ADLS Gen2 fully, archive tier)
- Diagnostic settings / Azure Monitor integration

**Best practices**:
- Use Azurite for functional testing
- Test authentication and network features in staging cloud environment
- Run performance tests against real Azure Storage

## CI/CD Integration

### GitHub Actions with Azurite

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      azurite:
        image: mcr.microsoft.com/azure-storage/azurite
        ports:
          - 10000:10000
          - 10001:10001
          - 10002:10002
        options: >-
          --health-cmd "nc -z localhost 10000"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Run tests
        env:
          AZURE_STORAGE_CONNECTION_STRING: "UseDevelopmentStorage=true;BlobEndpoint=http://localhost:10000/devstoreaccount1;QueueEndpoint=http://localhost:10001/devstoreaccount1;TableEndpoint=http://localhost:10002/devstoreaccount1;"
        run: dotnet test --configuration Release
```

## Troubleshooting

### Azurite Not Starting

```bash
# Check if ports are in use
lsof -i :10000  # macOS/Linux
netstat -ano | findstr :10000  # Windows

# Kill existing process
pkill -f azurite
```

### Connection Errors

```bash
# Test connectivity
curl http://127.0.0.1:10000/devstoreaccount1?comp=list

# Expected response: XML with container list
```

### Data Corruption

```bash
# Clear Azurite data
rm -rf ./azurite-data/*

# Restart Azurite
azurite --location ./azurite-data
```

## Development Workflow Checklist

- [ ] Azurite installed and running
- [ ] appsettings.Development.json configured with connection string
- [ ] appsettings.Production.json configured with service URIs
- [ ] Client initialization switches based on environment
- [ ] Seed script creates test containers and tables
- [ ] Unit tests run against Azurite
- [ ] .gitignore includes `azurite-data/` and `azurite-debug.log`

## Navigation

- **Previous**: `authentication-setup.md`
- **Next**: `02-core-concepts/storage-accounts.md`
- **Up**: `00-overview.md`

## See Also

- `07-deployment/testing-strategy.md` - Comprehensive testing patterns
- `08-reference/code-examples.md` - Complete examples
