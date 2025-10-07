# Testing Strategy for Azure Storage

> **File Purpose**: Comprehensive testing patterns for Azure Storage applications from unit to production validation
> **Prerequisites**: .NET 8 SDK, Docker, xUnit knowledge
> **Agent Use Case**: Reference when building test suites and CI/CD pipelines

## Quick Context

Effective Azure Storage testing requires a multi-layered approach: Azurite emulator for fast unit/integration tests, contract tests to verify API behavior, performance tests for throughput validation, and pre-production smoke tests. This guide provides complete, runnable test examples using C#, xUnit, and Docker.

**Key principle**: Test pyramid applies - many fast Azurite tests, fewer integration tests against real Azure, minimal manual validation. Automate everything.

## Azurite Emulator Setup

### Docker Compose for Consistent Environments

**docker-compose.test.yml**:

```yaml
version: '3.8'

services:
  azurite:
    image: mcr.microsoft.com/azure-storage/azurite:latest
    container_name: azurite-test
    ports:
      - "10000:10000"  # Blob service
      - "10001:10001"  # Queue service
      - "10002:10002"  # Table service
    volumes:
      - azurite-data:/data
    command: azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0 --location /data --loose --debug /data/debug.log
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "10000"]
      interval: 10s
      timeout: 5s
      retries: 3

  azurite-init:
    image: mcr.microsoft.com/azure-cli:latest
    container_name: azurite-init
    depends_on:
      azurite:
        condition: service_healthy
    volumes:
      - ./test-data:/test-data
    environment:
      AZURE_STORAGE_CONNECTION_STRING: "DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://azurite:10000/devstoreaccount1;QueueEndpoint=http://azurite:10001/devstoreaccount1;TableEndpoint=http://azurite:10002/devstoreaccount1;"
    entrypoint: /bin/bash
    command:
      - -c
      - |
        # Wait for Azurite to be ready
        sleep 5

        # Create containers
        az storage container create --name test-blobs --connection-string "$$AZURE_STORAGE_CONNECTION_STRING"
        az storage container create --name test-images --connection-string "$$AZURE_STORAGE_CONNECTION_STRING"

        # Create tables
        az storage table create --name TestUsers --connection-string "$$AZURE_STORAGE_CONNECTION_STRING"
        az storage table create --name TestSessions --connection-string "$$AZURE_STORAGE_CONNECTION_STRING"

        # Upload seed data
        echo "Test blob content" > /test-data/sample.txt
        az storage blob upload \
          --container-name test-blobs \
          --name sample.txt \
          --file /test-data/sample.txt \
          --connection-string "$$AZURE_STORAGE_CONNECTION_STRING"

        echo "Azurite initialized successfully"

volumes:
  azurite-data:
    driver: local
```

**Why it works**:
- `--loose` mode relaxes validation for faster testing
- Health check ensures Azurite is ready before init runs
- Init container seeds test data automatically
- Volume persistence maintains data between runs

### Starting Test Environment

```bash
# Start Azurite and seed data
docker-compose -f docker-compose.test.yml up -d

# Verify services are running
docker-compose -f docker-compose.test.yml ps

# View Azurite logs
docker-compose -f docker-compose.test.yml logs azurite

# Stop and clean up
docker-compose -f docker-compose.test.yml down -v
```

## Unit Tests with Azurite

### Test Project Setup

**Azure.Storage.Tests.csproj**:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Azure.Data.Tables" Version="12.8.3" />
    <PackageReference Include="Azure.Storage.Blobs" Version="12.19.1" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="Testcontainers.Azurite" Version="3.6.0" />
    <PackageReference Include="xunit" Version="2.6.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.5" />
  </ItemGroup>
</Project>
```

### Blob Storage Unit Tests

**BlobStorageTests.cs**:

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using FluentAssertions;
using Xunit;

namespace Azure.Storage.Tests;

public class BlobStorageTests : IAsyncLifetime
{
    private const string ConnectionString = "UseDevelopmentStorage=true";
    private BlobServiceClient _blobServiceClient = null!;
    private BlobContainerClient _containerClient = null!;

    public async Task InitializeAsync()
    {
        _blobServiceClient = new BlobServiceClient(ConnectionString);
        _containerClient = _blobServiceClient.GetBlobContainerClient($"test-{Guid.NewGuid()}");
        await _containerClient.CreateAsync();
    }

    public async Task DisposeAsync()
    {
        await _containerClient.DeleteAsync();
    }

    [Fact]
    public async Task UploadBlob_WithTextContent_ShouldSucceed()
    {
        // Arrange
        var blobName = "test-document.txt";
        var content = "This is test content";
        var blobClient = _containerClient.GetBlobClient(blobName);

        // Act
        await blobClient.UploadAsync(new BinaryData(content), overwrite: true);

        // Assert
        var exists = await blobClient.ExistsAsync();
        exists.Value.Should().BeTrue();

        var downloadResult = await blobClient.DownloadContentAsync();
        downloadResult.Value.Content.ToString().Should().Be(content);
    }

    [Fact]
    public async Task UploadBlob_WithMetadata_ShouldPersistMetadata()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("metadata-test.txt");
        var metadata = new Dictionary<string, string>
        {
            { "author", "test-user" },
            { "version", "1.0" },
            { "category", "test-data" }
        };

        var uploadOptions = new BlobUploadOptions
        {
            Metadata = metadata
        };

        // Act
        await blobClient.UploadAsync(new BinaryData("test"), uploadOptions);

        // Assert
        var properties = await blobClient.GetPropertiesAsync();
        properties.Value.Metadata.Should().ContainKeys("author", "version", "category");
        properties.Value.Metadata["author"].Should().Be("test-user");
    }

    [Fact]
    public async Task DownloadBlob_WhenNotExists_ShouldThrowException()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("non-existent.txt");

        // Act
        Func<Task> act = async () => await blobClient.DownloadContentAsync();

        // Assert
        await act.Should().ThrowAsync<global::Azure.RequestFailedException>()
            .Where(e => e.Status == 404);
    }

    [Fact]
    public async Task ListBlobs_WithPrefix_ShouldReturnFilteredBlobs()
    {
        // Arrange
        await _containerClient.GetBlobClient("docs/file1.txt").UploadAsync(new BinaryData("test"));
        await _containerClient.GetBlobClient("docs/file2.txt").UploadAsync(new BinaryData("test"));
        await _containerClient.GetBlobClient("images/pic1.jpg").UploadAsync(new BinaryData("test"));

        // Act
        var blobs = new List<string>();
        await foreach (var blobItem in _containerClient.GetBlobsAsync(prefix: "docs/"))
        {
            blobs.Add(blobItem.Name);
        }

        // Assert
        blobs.Should().HaveCount(2);
        blobs.Should().Contain("docs/file1.txt");
        blobs.Should().Contain("docs/file2.txt");
    }

    [Fact]
    public async Task DeleteBlob_ShouldRemoveBlobCompletely()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("to-delete.txt");
        await blobClient.UploadAsync(new BinaryData("test"));

        // Act
        await blobClient.DeleteAsync();

        // Assert
        var exists = await blobClient.ExistsAsync();
        exists.Value.Should().BeFalse();
    }

    [Fact]
    public async Task CopyBlob_ShouldCreateDuplicate()
    {
        // Arrange
        var sourceBlobClient = _containerClient.GetBlobClient("source.txt");
        var destBlobClient = _containerClient.GetBlobClient("destination.txt");
        var content = "Original content";

        await sourceBlobClient.UploadAsync(new BinaryData(content));

        // Act
        var copyOperation = await destBlobClient.StartCopyFromUriAsync(sourceBlobClient.Uri);
        await copyOperation.WaitForCompletionAsync();

        // Assert
        var downloadResult = await destBlobClient.DownloadContentAsync();
        downloadResult.Value.Content.ToString().Should().Be(content);
    }
}
```

### Table Storage Unit Tests

**TableStorageTests.cs**:

```csharp
using Azure;
using Azure.Data.Tables;
using FluentAssertions;
using Xunit;

namespace Azure.Storage.Tests;

public class UserEntity : ITableEntity
{
    public string PartitionKey { get; set; } = default!;
    public string RowKey { get; set; } = default!;
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string Email { get; set; } = default!;
    public string DisplayName { get; set; } = default!;
    public int LoginCount { get; set; }
    public bool IsActive { get; set; }
}

public class TableStorageTests : IAsyncLifetime
{
    private const string ConnectionString = "UseDevelopmentStorage=true";
    private TableServiceClient _tableServiceClient = null!;
    private TableClient _tableClient = null!;

    public async Task InitializeAsync()
    {
        _tableServiceClient = new TableServiceClient(ConnectionString);
        _tableClient = _tableServiceClient.GetTableClient($"test{Guid.NewGuid():N}");
        await _tableClient.CreateAsync();
    }

    public async Task DisposeAsync()
    {
        await _tableClient.DeleteAsync();
    }

    [Fact]
    public async Task InsertEntity_ShouldSucceed()
    {
        // Arrange
        var entity = new UserEntity
        {
            PartitionKey = "tenant-001",
            RowKey = "user-123",
            Email = "test@example.com",
            DisplayName = "Test User",
            LoginCount = 0,
            IsActive = true
        };

        // Act
        await _tableClient.AddEntityAsync(entity);

        // Assert
        var retrieved = await _tableClient.GetEntityAsync<UserEntity>(
            entity.PartitionKey,
            entity.RowKey);

        retrieved.Value.Email.Should().Be("test@example.com");
        retrieved.Value.DisplayName.Should().Be("Test User");
    }

    [Fact]
    public async Task UpdateEntity_WithETag_ShouldSucceed()
    {
        // Arrange
        var entity = new UserEntity
        {
            PartitionKey = "tenant-001",
            RowKey = "user-456",
            Email = "original@example.com",
            DisplayName = "Original Name",
            LoginCount = 5,
            IsActive = true
        };

        await _tableClient.AddEntityAsync(entity);
        var retrieved = await _tableClient.GetEntityAsync<UserEntity>(
            entity.PartitionKey,
            entity.RowKey);

        // Act
        retrieved.Value.DisplayName = "Updated Name";
        retrieved.Value.LoginCount = 10;
        await _tableClient.UpdateEntityAsync(retrieved.Value, retrieved.Value.ETag, TableUpdateMode.Replace);

        // Assert
        var updated = await _tableClient.GetEntityAsync<UserEntity>(
            entity.PartitionKey,
            entity.RowKey);

        updated.Value.DisplayName.Should().Be("Updated Name");
        updated.Value.LoginCount.Should().Be(10);
    }

    [Fact]
    public async Task UpdateEntity_WithStaleETag_ShouldThrowException()
    {
        // Arrange
        var entity = new UserEntity
        {
            PartitionKey = "tenant-001",
            RowKey = "user-789",
            Email = "test@example.com",
            DisplayName = "Test",
            LoginCount = 0,
            IsActive = true
        };

        await _tableClient.AddEntityAsync(entity);
        var first = await _tableClient.GetEntityAsync<UserEntity>(entity.PartitionKey, entity.RowKey);
        var second = await _tableClient.GetEntityAsync<UserEntity>(entity.PartitionKey, entity.RowKey);

        // Update via first reference
        first.Value.LoginCount = 1;
        await _tableClient.UpdateEntityAsync(first.Value, first.Value.ETag, TableUpdateMode.Replace);

        // Act - Try to update with stale ETag
        second.Value.LoginCount = 2;
        Func<Task> act = async () => await _tableClient.UpdateEntityAsync(
            second.Value,
            second.Value.ETag,
            TableUpdateMode.Replace);

        // Assert
        await act.Should().ThrowAsync<RequestFailedException>()
            .Where(e => e.Status == 412); // Precondition Failed
    }

    [Fact]
    public async Task QueryEntities_WithFilter_ShouldReturnMatchingEntities()
    {
        // Arrange
        var entities = new[]
        {
            new UserEntity { PartitionKey = "tenant-001", RowKey = "user-1", Email = "active1@example.com", IsActive = true },
            new UserEntity { PartitionKey = "tenant-001", RowKey = "user-2", Email = "active2@example.com", IsActive = true },
            new UserEntity { PartitionKey = "tenant-001", RowKey = "user-3", Email = "inactive@example.com", IsActive = false }
        };

        foreach (var entity in entities)
        {
            await _tableClient.AddEntityAsync(entity);
        }

        // Act
        var activeUsers = new List<UserEntity>();
        await foreach (var user in _tableClient.QueryAsync<UserEntity>(
            filter: $"PartitionKey eq 'tenant-001' and IsActive eq true"))
        {
            activeUsers.Add(user);
        }

        // Assert
        activeUsers.Should().HaveCount(2);
        activeUsers.Should().AllSatisfy(u => u.IsActive.Should().BeTrue());
    }

    [Fact]
    public async Task UpsertEntity_WhenNotExists_ShouldInsert()
    {
        // Arrange
        var entity = new UserEntity
        {
            PartitionKey = "tenant-002",
            RowKey = "user-100",
            Email = "upsert@example.com",
            DisplayName = "Upsert User",
            LoginCount = 0,
            IsActive = true
        };

        // Act
        await _tableClient.UpsertEntityAsync(entity, TableUpdateMode.Replace);

        // Assert
        var retrieved = await _tableClient.GetEntityAsync<UserEntity>(
            entity.PartitionKey,
            entity.RowKey);

        retrieved.Value.Email.Should().Be("upsert@example.com");
    }

    [Fact]
    public async Task DeleteEntity_ShouldRemoveEntity()
    {
        // Arrange
        var entity = new UserEntity
        {
            PartitionKey = "tenant-003",
            RowKey = "user-delete",
            Email = "delete@example.com",
            DisplayName = "Delete Me",
            LoginCount = 0,
            IsActive = true
        };

        await _tableClient.AddEntityAsync(entity);

        // Act
        await _tableClient.DeleteEntityAsync(entity.PartitionKey, entity.RowKey);

        // Assert
        Func<Task> act = async () => await _tableClient.GetEntityAsync<UserEntity>(
            entity.PartitionKey,
            entity.RowKey);

        await act.Should().ThrowAsync<RequestFailedException>()
            .Where(e => e.Status == 404);
    }

    [Fact]
    public async Task BatchTransaction_ShouldExecuteAtomically()
    {
        // Arrange
        var partitionKey = "tenant-batch";
        var entities = new[]
        {
            new UserEntity { PartitionKey = partitionKey, RowKey = "user-1", Email = "batch1@example.com", DisplayName = "User 1", IsActive = true },
            new UserEntity { PartitionKey = partitionKey, RowKey = "user-2", Email = "batch2@example.com", DisplayName = "User 2", IsActive = true },
            new UserEntity { PartitionKey = partitionKey, RowKey = "user-3", Email = "batch3@example.com", DisplayName = "User 3", IsActive = true }
        };

        // Act
        var batch = new List<TableTransactionAction>();
        foreach (var entity in entities)
        {
            batch.Add(new TableTransactionAction(TableTransactionActionType.Add, entity));
        }

        await _tableClient.SubmitTransactionAsync(batch);

        // Assert
        var count = 0;
        await foreach (var _ in _tableClient.QueryAsync<UserEntity>(
            filter: $"PartitionKey eq '{partitionKey}'"))
        {
            count++;
        }

        count.Should().Be(3);
    }
}
```

### Blob Concurrency Tests

**BlobConcurrencyTests.cs**:

```csharp
using Azure;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using FluentAssertions;
using Xunit;

namespace Azure.Storage.Tests;

public class BlobConcurrencyTests : IAsyncLifetime
{
    private const string ConnectionString = "UseDevelopmentStorage=true";
    private BlobServiceClient _blobServiceClient = null!;
    private BlobContainerClient _containerClient = null!;

    public async Task InitializeAsync()
    {
        _blobServiceClient = new BlobServiceClient(ConnectionString);
        _containerClient = _blobServiceClient.GetBlobContainerClient($"concurrency-{Guid.NewGuid()}");
        await _containerClient.CreateAsync();
    }

    public async Task DisposeAsync()
    {
        await _containerClient.DeleteAsync();
    }

    [Fact]
    public async Task Upload_WithIfNoneMatch_ShouldPreventOverwrite()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("exclusive.txt");
        await blobClient.UploadAsync(new BinaryData("Original content"));

        // Act - Try to upload with If-None-Match: * (only if blob doesn't exist)
        var conditions = new BlobRequestConditions { IfNoneMatch = ETag.All };
        var uploadOptions = new BlobUploadOptions { Conditions = conditions };

        Func<Task> act = async () => await blobClient.UploadAsync(
            new BinaryData("New content"),
            uploadOptions);

        // Assert
        await act.Should().ThrowAsync<RequestFailedException>()
            .Where(e => e.Status == 409); // Conflict - blob already exists
    }

    [Fact]
    public async Task Update_WithMatchingETag_ShouldSucceed()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("optimistic.txt");
        await blobClient.UploadAsync(new BinaryData("Version 1"));

        var properties = await blobClient.GetPropertiesAsync();
        var etag = properties.Value.ETag;

        // Act
        var conditions = new BlobRequestConditions { IfMatch = etag };
        var uploadOptions = new BlobUploadOptions { Conditions = conditions };

        await blobClient.UploadAsync(new BinaryData("Version 2"), uploadOptions);

        // Assert
        var download = await blobClient.DownloadContentAsync();
        download.Value.Content.ToString().Should().Be("Version 2");
    }

    [Fact]
    public async Task Update_WithStaleETag_ShouldFail()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("stale-etag.txt");
        await blobClient.UploadAsync(new BinaryData("Version 1"));

        var firstRead = await blobClient.GetPropertiesAsync();
        var staleETag = firstRead.Value.ETag;

        // Someone else updates the blob
        await blobClient.UploadAsync(new BinaryData("Version 2"), overwrite: true);

        // Act - Try to update with stale ETag
        var conditions = new BlobRequestConditions { IfMatch = staleETag };
        var uploadOptions = new BlobUploadOptions { Conditions = conditions };

        Func<Task> act = async () => await blobClient.UploadAsync(
            new BinaryData("Version 3"),
            uploadOptions);

        // Assert
        await act.Should().ThrowAsync<RequestFailedException>()
            .Where(e => e.Status == 412); // Precondition Failed
    }

    [Fact]
    public async Task LeaseBlob_ShouldPreventConcurrentModifications()
    {
        // Arrange
        var blobClient = _containerClient.GetBlobClient("leased.txt");
        await blobClient.UploadAsync(new BinaryData("Leased content"));

        var leaseClient = blobClient.GetBlobLeaseClient();

        // Act - Acquire lease
        var lease = await leaseClient.AcquireAsync(TimeSpan.FromSeconds(30));

        // Try to modify without lease ID - should fail
        Func<Task> actWithoutLease = async () => await blobClient.UploadAsync(
            new BinaryData("Modified"),
            overwrite: true);

        await actWithoutLease.Should().ThrowAsync<RequestFailedException>()
            .Where(e => e.Status == 412);

        // Modify with lease ID - should succeed
        var conditions = new BlobRequestConditions { LeaseId = lease.Value.LeaseId };
        var uploadOptions = new BlobUploadOptions { Conditions = conditions };

        await blobClient.UploadAsync(new BinaryData("Modified with lease"), uploadOptions);

        // Assert
        await leaseClient.ReleaseAsync();
    }
}
```

## Integration Tests with Real Azure Storage

### Test Configuration

**appsettings.IntegrationTests.json**:

```json
{
  "AzureStorage": {
    "BlobServiceUri": "https://stintegrationtest.blob.core.windows.net",
    "TableServiceUri": "https://stintegrationtest.table.core.windows.net",
    "TestContainerPrefix": "integration-test",
    "TestTablePrefix": "IntegrationTest"
  }
}
```

### Integration Test Base Class

**IntegrationTestBase.cs**:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using Microsoft.Extensions.Configuration;
using Xunit;

namespace Azure.Storage.IntegrationTests;

public abstract class IntegrationTestBase : IAsyncLifetime
{
    protected BlobServiceClient BlobServiceClient { get; private set; } = null!;
    protected TableServiceClient TableServiceClient { get; private set; } = null!;
    protected IConfiguration Configuration { get; private set; } = null!;

    protected string TestRunId { get; } = Guid.NewGuid().ToString("N")[..8];

    public virtual Task InitializeAsync()
    {
        Configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.IntegrationTests.json")
            .AddEnvironmentVariables()
            .Build();

        var credential = new DefaultAzureCredential();

        var blobUri = new Uri(Configuration["AzureStorage:BlobServiceUri"]!);
        BlobServiceClient = new BlobServiceClient(blobUri, credential);

        var tableUri = new Uri(Configuration["AzureStorage:TableServiceUri"]!);
        TableServiceClient = new TableServiceClient(tableUri, credential);

        return Task.CompletedTask;
    }

    public virtual Task DisposeAsync()
    {
        // Cleanup test resources
        return Task.CompletedTask;
    }

    protected async Task<BlobContainerClient> CreateTestContainerAsync()
    {
        var containerName = $"{Configuration["AzureStorage:TestContainerPrefix"]}-{TestRunId}";
        var container = BlobServiceClient.GetBlobContainerClient(containerName);
        await container.CreateAsync();
        return container;
    }

    protected async Task<TableClient> CreateTestTableAsync()
    {
        var tableName = $"{Configuration["AzureStorage:TestTablePrefix"]}{TestRunId}";
        var table = TableServiceClient.GetTableClient(tableName);
        await table.CreateAsync();
        return table;
    }
}
```

### Integration Test Example

**RealAzureStorageTests.cs**:

```csharp
using Azure.Storage.Blobs.Models;
using FluentAssertions;
using Xunit;

namespace Azure.Storage.IntegrationTests;

[Trait("Category", "Integration")]
public class RealAzureStorageTests : IntegrationTestBase
{
    private BlobContainerClient _container = null!;
    private TableClient _table = null!;

    public override async Task InitializeAsync()
    {
        await base.InitializeAsync();
        _container = await CreateTestContainerAsync();
        _table = await CreateTestTableAsync();
    }

    public override async Task DisposeAsync()
    {
        await _container.DeleteAsync();
        await _table.DeleteAsync();
        await base.DisposeAsync();
    }

    [Fact]
    public async Task RealStorage_UploadLargeBlob_ShouldSucceed()
    {
        // Arrange
        var blobClient = _container.GetBlobClient("large-file.bin");
        var dataSize = 50 * 1024 * 1024; // 50 MB
        var data = new byte[dataSize];
        new Random().NextBytes(data);

        // Act
        var startTime = DateTime.UtcNow;
        await blobClient.UploadAsync(new BinaryData(data), overwrite: true);
        var uploadDuration = DateTime.UtcNow - startTime;

        // Assert
        var exists = await blobClient.ExistsAsync();
        exists.Value.Should().BeTrue();

        var properties = await blobClient.GetPropertiesAsync();
        properties.Value.ContentLength.Should().Be(dataSize);

        // Performance assertion
        uploadDuration.Should().BeLessThan(TimeSpan.FromSeconds(30));
    }

    [Fact]
    public async Task RealStorage_AccessTierTransition_ShouldWork()
    {
        // Arrange
        var blobClient = _container.GetBlobClient("tier-test.txt");
        await blobClient.UploadAsync(new BinaryData("Test content"));

        // Act - Move to Cool tier
        await blobClient.SetAccessTierAsync(AccessTier.Cool);

        // Assert
        var properties = await blobClient.GetPropertiesAsync();
        properties.Value.AccessTier.Should().Be(AccessTier.Cool.ToString());
    }

    [Fact]
    public async Task RealStorage_TableBatchPerformance_ShouldMeetSLA()
    {
        // Arrange
        var entities = Enumerable.Range(0, 100)
            .Select(i => new TableEntity("perf-test", $"row-{i:D4}")
            {
                { "Data", $"Test data {i}" }
            })
            .ToList();

        // Act
        var startTime = DateTime.UtcNow;

        var batch = new List<TableTransactionAction>();
        foreach (var entity in entities)
        {
            batch.Add(new TableTransactionAction(TableTransactionActionType.Add, entity));

            // Table batch max 100 operations
            if (batch.Count == 100)
            {
                await _table.SubmitTransactionAsync(batch);
                batch.Clear();
            }
        }

        if (batch.Any())
        {
            await _table.SubmitTransactionAsync(batch);
        }

        var duration = DateTime.UtcNow - startTime;

        // Assert - Should complete in under 2 seconds
        duration.Should().BeLessThan(TimeSpan.FromSeconds(2));
    }
}
```

## Performance Testing

### Performance Test Suite

**PerformanceTests.cs**:

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace Azure.Storage.PerformanceTests;

[MemoryDiagnoser]
[SimpleJob(iterationCount: 10)]
public class BlobPerformanceBenchmarks
{
    private BlobServiceClient _serviceClient = null!;
    private BlobContainerClient _containerClient = null!;
    private byte[] _testData = null!;

    [GlobalSetup]
    public async Task Setup()
    {
        _serviceClient = new BlobServiceClient("UseDevelopmentStorage=true");
        _containerClient = _serviceClient.GetBlobContainerClient("perf-test");
        await _containerClient.CreateIfNotExistsAsync();

        _testData = new byte[1024 * 1024]; // 1 MB
        new Random(42).NextBytes(_testData);
    }

    [GlobalCleanup]
    public async Task Cleanup()
    {
        await _containerClient.DeleteAsync();
    }

    [Benchmark]
    public async Task UploadSmallBlob_1MB()
    {
        var blobClient = _containerClient.GetBlobClient($"small-{Guid.NewGuid()}.bin");
        await blobClient.UploadAsync(new BinaryData(_testData), overwrite: true);
    }

    [Benchmark]
    public async Task UploadWithOptions_Parallel()
    {
        var blobClient = _containerClient.GetBlobClient($"parallel-{Guid.NewGuid()}.bin");

        var uploadOptions = new BlobUploadOptions
        {
            TransferOptions = new global::Azure.Storage.StorageTransferOptions
            {
                MaximumConcurrency = 8,
                InitialTransferSize = 4 * 1024 * 1024,
                MaximumTransferSize = 4 * 1024 * 1024
            }
        };

        await blobClient.UploadAsync(new BinaryData(_testData), uploadOptions);
    }

    [Benchmark]
    public async Task ListBlobs_1000Items()
    {
        var count = 0;
        await foreach (var _ in _containerClient.GetBlobsAsync())
        {
            count++;
            if (count >= 1000) break;
        }
    }
}

public class Program
{
    public static void Main(string[] args)
    {
        BenchmarkRunner.Run<BlobPerformanceBenchmarks>();
    }
}
```

### Load Testing Script

**load-test.ps1**:

```powershell
# Load test for Azure Storage
param(
    [int]$Threads = 10,
    [int]$DurationSeconds = 60,
    [string]$StorageAccount = "stloadtest001",
    [string]$ContainerName = "load-test"
)

$jobs = @()
$startTime = Get-Date

for ($i = 1; $i -le $Threads; $i++) {
    $jobs += Start-Job -ScriptBlock {
        param($Account, $Container, $Duration, $ThreadId)

        $endTime = (Get-Date).AddSeconds($Duration)
        $uploads = 0
        $downloads = 0
        $errors = 0

        while ((Get-Date) -lt $endTime) {
            try {
                $blobName = "thread-$ThreadId-$(New-Guid).txt"
                $content = "Test data from thread $ThreadId"

                # Upload
                az storage blob upload `
                    --account-name $Account `
                    --container-name $Container `
                    --name $blobName `
                    --data $content `
                    --auth-mode login `
                    --only-show-errors

                $uploads++

                # Download
                az storage blob download `
                    --account-name $Account `
                    --container-name $Container `
                    --name $blobName `
                    --file "temp-$ThreadId.txt" `
                    --auth-mode login `
                    --only-show-errors

                $downloads++

                Remove-Item "temp-$ThreadId.txt" -ErrorAction SilentlyContinue
            }
            catch {
                $errors++
            }
        }

        @{
            ThreadId = $ThreadId
            Uploads = $uploads
            Downloads = $downloads
            Errors = $errors
        }
    } -ArgumentList $StorageAccount, $ContainerName, $DurationSeconds, $i
}

Write-Host "Started $Threads threads, running for $DurationSeconds seconds..."

$results = $jobs | Wait-Job | Receive-Job
$jobs | Remove-Job

$totalUploads = ($results | Measure-Object -Property Uploads -Sum).Sum
$totalDownloads = ($results | Measure-Object -Property Downloads -Sum).Sum
$totalErrors = ($results | Measure-Object -Property Errors -Sum).Sum
$duration = ((Get-Date) - $startTime).TotalSeconds

Write-Host "`nLoad Test Results:"
Write-Host "==================="
Write-Host "Duration: $duration seconds"
Write-Host "Total Uploads: $totalUploads ($($totalUploads / $duration) ops/sec)"
Write-Host "Total Downloads: $totalDownloads ($($totalDownloads / $duration) ops/sec)"
Write-Host "Total Errors: $totalErrors"
Write-Host "Error Rate: $($totalErrors / ($totalUploads + $totalDownloads) * 100)%"
```

## Pre-Production Validation Checklist

### Smoke Test Suite

**SmokeTests.cs**:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using FluentAssertions;
using Xunit;

namespace Azure.Storage.SmokeTests;

[Trait("Category", "Smoke")]
public class PreProductionSmokeTests
{
    private readonly DefaultAzureCredential _credential = new();

    [Theory]
    [InlineData("https://stprodapp001.blob.core.windows.net", "smoke-test")]
    public async Task Blob_ConnectivityCheck_ShouldSucceed(string serviceUri, string containerName)
    {
        // Arrange
        var blobService = new BlobServiceClient(new Uri(serviceUri), _credential);
        var container = blobService.GetBlobContainerClient(containerName);

        // Act & Assert
        var exists = await container.ExistsAsync();
        exists.Value.Should().BeTrue("Container must exist for smoke test");

        // Test write
        var testBlob = container.GetBlobClient($"smoke-test-{DateTime.UtcNow:yyyyMMddHHmmss}.txt");
        await testBlob.UploadAsync(new BinaryData("Smoke test"));

        // Test read
        var download = await testBlob.DownloadContentAsync();
        download.Value.Content.ToString().Should().Contain("Smoke test");

        // Cleanup
        await testBlob.DeleteAsync();
    }

    [Theory]
    [InlineData("https://stprodapp001.table.core.windows.net", "SmokeTest")]
    public async Task Table_ConnectivityCheck_ShouldSucceed(string serviceUri, string tableName)
    {
        // Arrange
        var tableService = new TableServiceClient(new Uri(serviceUri), _credential);
        var table = tableService.GetTableClient(tableName);

        // Act & Assert
        var exists = await table.CreateIfNotExistsAsync();

        // Test write
        var entity = new TableEntity("smoke-test", Guid.NewGuid().ToString())
        {
            { "Timestamp", DateTime.UtcNow },
            { "TestData", "Smoke test" }
        };

        await table.AddEntityAsync(entity);

        // Test read
        var retrieved = await table.GetEntityAsync<TableEntity>(
            entity.PartitionKey,
            entity.RowKey);

        retrieved.Value["TestData"].ToString().Should().Be("Smoke test");

        // Cleanup
        await table.DeleteEntityAsync(entity.PartitionKey, entity.RowKey);
    }

    [Fact]
    public async Task PrivateEndpoint_ShouldBeAccessible()
    {
        // This test should run from a VM inside the VNET
        var serviceUri = "https://stprodapp001.blob.core.windows.net";
        var blobService = new BlobServiceClient(new Uri(serviceUri), _credential);

        // Verify can access via private endpoint
        var accountInfo = await blobService.GetAccountInfoAsync();
        accountInfo.Should().NotBeNull();
    }
}
```

### Validation Checklist

```markdown
## Pre-Production Checklist

### Infrastructure Validation
- [ ] Storage account exists and accessible
- [ ] Correct redundancy configured (GZRS for prod)
- [ ] Private endpoints provisioned and healthy
- [ ] Private DNS zones configured
- [ ] Network rules deny public access
- [ ] Diagnostic settings enabled (Log Analytics)

### Security Validation
- [ ] Managed Identity assigned to application
- [ ] RBAC roles granted (Storage Blob Data Contributor, Storage Table Data Contributor)
- [ ] Account keys disabled or rotated
- [ ] TLS 1.2+ enforced
- [ ] No anonymous access on containers
- [ ] Soft delete enabled (7-30 days retention)

### Performance Validation
- [ ] Throughput targets meet requirements (see performance tests)
- [ ] Latency < 50ms for 95th percentile
- [ ] Large file uploads use parallel transfers
- [ ] Table partitioning avoids hot partitions
- [ ] CDN configured for static content (if applicable)

### Monitoring Validation
- [ ] Metrics flowing to Azure Monitor
- [ ] Alerts configured (throttling, availability, latency)
- [ ] Log Analytics queries tested
- [ ] Runbook for incident response documented

### Application Validation
- [ ] DefaultAzureCredential authentication works
- [ ] Retry policies configured (exponential backoff)
- [ ] Error handling covers transient failures
- [ ] Integration tests pass against staging
- [ ] Smoke tests pass against production
```

## Test Data Management

### Test Data Factory

**TestDataFactory.cs**:

```csharp
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using Bogus;

namespace Azure.Storage.Tests.TestData;

public static class TestDataFactory
{
    public static async Task SeedBlobContainerAsync(
        BlobContainerClient container,
        int blobCount = 100)
    {
        var faker = new Faker();

        for (int i = 0; i < blobCount; i++)
        {
            var blobName = $"test-data/{faker.System.FileName("txt")}";
            var content = faker.Lorem.Paragraphs(3);
            var blobClient = container.GetBlobClient(blobName);

            await blobClient.UploadAsync(
                new BinaryData(content),
                overwrite: true);
        }
    }

    public static async Task SeedTableAsync(
        TableClient table,
        int entityCount = 1000)
    {
        var faker = new Faker();

        var batch = new List<TableTransactionAction>();
        var currentPartition = Guid.NewGuid().ToString();

        for (int i = 0; i < entityCount; i++)
        {
            // New partition every 100 entities (batch limit)
            if (i % 100 == 0 && i > 0)
            {
                await table.SubmitTransactionAsync(batch);
                batch.Clear();
                currentPartition = Guid.NewGuid().ToString();
            }

            var entity = new TableEntity(currentPartition, Guid.NewGuid().ToString())
            {
                { "Name", faker.Name.FullName() },
                { "Email", faker.Internet.Email() },
                { "CreatedAt", faker.Date.Past() }
            };

            batch.Add(new TableTransactionAction(
                TableTransactionActionType.Add,
                entity));
        }

        if (batch.Any())
        {
            await table.SubmitTransactionAsync(batch);
        }
    }
}
```

### Cleanup Strategy

**TestCleanup.cs**:

```csharp
using Azure.Storage.Blobs;
using Azure.Data.Tables;

namespace Azure.Storage.Tests;

public static class TestCleanup
{
    public static async Task CleanupOldTestResourcesAsync(
        BlobServiceClient blobService,
        TableServiceClient tableService,
        TimeSpan olderThan)
    {
        var cutoffDate = DateTimeOffset.UtcNow - olderThan;

        // Cleanup old test containers
        await foreach (var container in blobService.GetBlobContainersAsync(
            prefix: "test-"))
        {
            if (container.Properties.LastModified < cutoffDate)
            {
                await blobService.GetBlobContainerClient(container.Name)
                    .DeleteAsync();
            }
        }

        // Cleanup old test tables
        await foreach (var table in tableService.QueryAsync(
            filter: $"TableName ge 'Test'"))
        {
            var tableClient = tableService.GetTableClient(table.Name);

            // Check if table has old data
            var hasRecentData = false;
            await foreach (var entity in tableClient.QueryAsync<TableEntity>(
                maxPerPage: 1))
            {
                if (entity.Timestamp > cutoffDate)
                {
                    hasRecentData = true;
                    break;
                }
            }

            if (!hasRecentData)
            {
                await tableClient.DeleteAsync();
            }
        }
    }
}
```

## Chaos Engineering for Resilience

### Chaos Test Examples

**ChaosTests.cs**:

```csharp
using Azure.Storage.Blobs;
using FluentAssertions;
using Polly;
using Polly.Retry;
using Xunit;

namespace Azure.Storage.ChaosTests;

public class ResilienceTests
{
    [Fact]
    public async Task Upload_WithTransientFailures_ShouldRetry()
    {
        // Arrange
        var blobClient = new BlobServiceClient("UseDevelopmentStorage=true")
            .GetBlobContainerClient("chaos-test")
            .GetBlobClient("resilient-upload.txt");

        await blobClient.GetParentBlobContainerClient().CreateIfNotExistsAsync();

        var retryPolicy = Policy
            .Handle<global::Azure.RequestFailedException>(e => e.Status == 503)
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
                onRetry: (exception, timespan, attempt, context) =>
                {
                    Console.WriteLine($"Retry {attempt} after {timespan.TotalSeconds}s due to {exception.Message}");
                });

        // Act
        var result = await retryPolicy.ExecuteAsync(async () =>
        {
            await blobClient.UploadAsync(new BinaryData("Test content"), overwrite: true);
            return true;
        });

        // Assert
        result.Should().BeTrue();
        var exists = await blobClient.ExistsAsync();
        exists.Value.Should().BeTrue();
    }

    [Fact]
    public async Task BulkOperation_WithPartialFailures_ShouldContinue()
    {
        // Arrange
        var containerClient = new BlobServiceClient("UseDevelopmentStorage=true")
            .GetBlobContainerClient("bulk-chaos");

        await containerClient.CreateIfNotExistsAsync();

        var blobs = Enumerable.Range(0, 100)
            .Select(i => $"bulk-{i:D4}.txt")
            .ToList();

        // Act
        var successCount = 0;
        var failureCount = 0;

        var tasks = blobs.Select(async blobName =>
        {
            try
            {
                var blobClient = containerClient.GetBlobClient(blobName);
                await blobClient.UploadAsync(new BinaryData($"Content {blobName}"));
                Interlocked.Increment(ref successCount);
            }
            catch
            {
                Interlocked.Increment(ref failureCount);
            }
        });

        await Task.WhenAll(tasks);

        // Assert - At least 95% success rate
        var successRate = (double)successCount / blobs.Count;
        successRate.Should().BeGreaterThan(0.95);
    }
}
```

## CI/CD Integration

### GitHub Actions Test Workflow

**.github/workflows/test.yml**:

```yaml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  unit-tests:
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

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run unit tests
        env:
          AZURE_STORAGE_CONNECTION_STRING: "UseDevelopmentStorage=true;BlobEndpoint=http://localhost:10000/devstoreaccount1;QueueEndpoint=http://localhost:10001/devstoreaccount1;TableEndpoint=http://localhost:10002/devstoreaccount1;"
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            --logger "trx;LogFileName=test-results.trx" \
            --collect:"XPlat Code Coverage" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/test-results.trx'

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: '**/coverage.opencover.xml'

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.ref == 'refs/heads/main'
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Run integration tests
        run: |
          dotnet test \
            --configuration Release \
            --filter "Category=Integration" \
            --logger "trx;LogFileName=integration-results.trx"

  smoke-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Run smoke tests
        run: |
          dotnet test \
            --configuration Release \
            --filter "Category=Smoke" \
            --logger "console;verbosity=detailed"
```

## Testing Checklist

### Development
- [ ] Azurite running locally (Docker Compose)
- [ ] Unit tests cover all CRUD operations
- [ ] Concurrency tests validate ETags and leases
- [ ] Mock/stub external dependencies
- [ ] Test coverage > 80%

### CI Pipeline
- [ ] Azurite container in GitHub Actions
- [ ] Unit tests run on every push
- [ ] Integration tests run on main branch
- [ ] Test results uploaded as artifacts
- [ ] Code coverage tracked

### Pre-Production
- [ ] Integration tests pass against staging
- [ ] Performance benchmarks meet SLAs
- [ ] Load tests validate scalability
- [ ] Smoke tests validate production access
- [ ] Chaos tests validate resilience

### Production
- [ ] Smoke tests run post-deployment
- [ ] Health checks verify connectivity
- [ ] Monitoring alerts configured
- [ ] Rollback procedure tested

## Navigation

- **Previous**: `cicd-patterns.md`
- **Next**: `08-reference/code-examples.md`
- **Up**: `00-overview.md`

## See Also

- `01-quick-start/local-development.md` - Azurite setup basics
- `cicd-patterns.md` - GitHub Actions workflows
- `06-operations/observability.md` - Production monitoring
- `03-blob-storage/blob-concurrency.md` - ETag patterns

## References

- [Azurite Emulator Documentation](https://learn.microsoft.com/azure/storage/common/storage-use-azurite) - Microsoft Learn
- [Testing with Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-use-emulator) - Microsoft Learn
- [Testcontainers for .NET](https://dotnet.testcontainers.org/) - Testcontainers
- [xUnit Best Practices](https://xunit.net/docs/getting-started/netcore/cmdline) - xUnit Documentation
- [BenchmarkDotNet](https://benchmarkdotnet.org/articles/overview.html) - Performance Testing
