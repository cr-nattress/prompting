# Azure Storage Data Access Layer Patterns

> **File Purpose**: Production-ready C# client patterns, dependency injection, retry policies, connection pooling, and SDK best practices
> **Prerequisites**: `../02-core-concepts/storage-accounts.md`, `performance-optimization.md`
> **Agent Use Case**: Reference when implementing storage clients in applications, setting up DI containers, or designing resilient data access layers

## Quick Context

A well-designed data access layer (DAL) abstracts storage operations, handles retries, manages connections, and provides testability. Proper client configuration is critical for production reliability.

**Key principle**: Use dependency injection; configure retry policies; implement circuit breakers; handle transient failures gracefully; design for testability.

## Client Architecture

### Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│  Application Layer (Controllers, Services)          │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Depends on abstractions
                 ▼
┌─────────────────────────────────────────────────────┐
│  Data Access Layer (Repositories, Interfaces)       │
│  - IBlobRepository, ITableRepository                │
│  - Retry policies, error handling                   │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Uses Azure SDK clients
                 ▼
┌─────────────────────────────────────────────────────┐
│  Azure SDK Clients (BlobServiceClient, etc.)        │
│  - Configured in DI container                       │
│  - Shared across requests (singleton)               │
└─────────────────────────────────────────────────────┘
```

## Dependency Injection Setup

### ASP.NET Core Configuration

#### Program.cs (Minimal API / .NET 6+)

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using Azure.Storage.Queues;
using Microsoft.Extensions.Azure;

var builder = WebApplication.CreateBuilder(args);

// Add Azure clients using Microsoft.Extensions.Azure
builder.Services.AddAzureClients(clientBuilder =>
{
    var storageAccountUrl = builder.Configuration["Azure:Storage:AccountUrl"]!;
    var credential = new DefaultAzureCredential();

    // Blob Service Client (singleton)
    clientBuilder.AddBlobServiceClient(new Uri(storageAccountUrl))
        .WithCredential(credential)
        .WithName("BlobService")
        .ConfigureOptions(options =>
        {
            options.Retry.MaxRetries = 5;
            options.Retry.Delay = TimeSpan.FromMilliseconds(100);
            options.Retry.MaxDelay = TimeSpan.FromSeconds(10);
            options.Retry.Mode = Azure.Core.RetryMode.Exponential;
            options.Diagnostics.IsLoggingEnabled = true;
            options.Diagnostics.IsDistributedTracingEnabled = true;
        });

    // Table Service Client (singleton)
    clientBuilder.AddTableServiceClient(new Uri(storageAccountUrl))
        .WithCredential(credential)
        .WithName("TableService")
        .ConfigureOptions(options =>
        {
            options.Retry.MaxRetries = 5;
            options.Retry.Mode = Azure.Core.RetryMode.Exponential;
        });

    // Queue Service Client (singleton)
    clientBuilder.AddQueueServiceClient(new Uri(storageAccountUrl))
        .WithCredential(credential)
        .WithName("QueueService")
        .ConfigureOptions(options =>
        {
            options.MessageEncoding = Azure.Storage.Queues.QueueMessageEncoding.Base64;
            options.Retry.MaxRetries = 5;
        });
});

// Register repositories
builder.Services.AddScoped<IBlobRepository, BlobRepository>();
builder.Services.AddScoped<ITableRepository, TableRepository>();
builder.Services.AddScoped<IQueueRepository, QueueRepository>();

var app = builder.Build();
app.Run();
```

#### appsettings.json

```json
{
  "Azure": {
    "Storage": {
      "AccountUrl": "https://mystorageaccount.blob.core.windows.net",
      "TableUrl": "https://mystorageaccount.table.core.windows.net",
      "QueueUrl": "https://mystorageaccount.queue.core.windows.net"
    }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Azure": "Warning"
    }
  }
}
```

### Manual Configuration (Non-DI Scenarios)

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Core;

public static class StorageClientFactory
{
    public static BlobServiceClient CreateBlobServiceClient(string accountUrl)
    {
        var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
        {
            ExcludeEnvironmentCredential = false,
            ExcludeWorkloadIdentityCredential = false,
            ExcludeManagedIdentityCredential = false,
            ExcludeSharedTokenCacheCredential = true,
            ExcludeVisualStudioCredential = true,
            ExcludeVisualStudioCodeCredential = true,
            ExcludeAzureCliCredential = false,
            ExcludeAzurePowerShellCredential = true,
            ExcludeInteractiveBrowserCredential = true
        });

        var clientOptions = new BlobClientOptions
        {
            Retry =
            {
                MaxRetries = 5,
                Delay = TimeSpan.FromMilliseconds(100),
                MaxDelay = TimeSpan.FromSeconds(10),
                Mode = RetryMode.Exponential,
                NetworkTimeout = TimeSpan.FromSeconds(100)
            },
            Diagnostics =
            {
                IsLoggingEnabled = true,
                IsDistributedTracingEnabled = true,
                ApplicationId = "MyApp/1.0"
            },
            // Connection pooling (default is good)
            Transport = new HttpClientTransport(new HttpClient
            {
                Timeout = TimeSpan.FromMinutes(5),
                DefaultRequestHeaders =
                {
                    { "x-ms-client-request-id", Guid.NewGuid().ToString() }
                }
            })
        };

        return new BlobServiceClient(new Uri(accountUrl), credential, clientOptions);
    }
}

// Usage
var blobServiceClient = StorageClientFactory.CreateBlobServiceClient(
    "https://mystorageaccount.blob.core.windows.net"
);
```

## Repository Pattern

### IBlobRepository Interface

```csharp
public interface IBlobRepository
{
    Task<string> UploadAsync(string containerName, string blobName, Stream content, string contentType);
    Task<Stream> DownloadAsync(string containerName, string blobName);
    Task<bool> ExistsAsync(string containerName, string blobName);
    Task DeleteAsync(string containerName, string blobName);
    Task<IEnumerable<string>> ListBlobsAsync(string containerName, string prefix = "");
    Task<BlobMetadata> GetMetadataAsync(string containerName, string blobName);
}

public record BlobMetadata(
    string Name,
    long Size,
    DateTimeOffset LastModified,
    string ContentType,
    string ETag
);
```

### BlobRepository Implementation

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure;
using Microsoft.Extensions.Logging;
using Polly;
using Polly.CircuitBreaker;

public class BlobRepository : IBlobRepository
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<BlobRepository> _logger;
    private readonly AsyncCircuitBreakerPolicy _circuitBreakerPolicy;

    public BlobRepository(
        BlobServiceClient blobServiceClient,
        ILogger<BlobRepository> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;

        // Circuit breaker: Open after 5 failures, stay open for 30s
        _circuitBreakerPolicy = Policy
            .Handle<RequestFailedException>(ex => ex.Status >= 500)
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) =>
                {
                    _logger.LogWarning(
                        "Circuit breaker opened due to {ExceptionType}. Duration: {Duration}",
                        ex.GetType().Name,
                        duration
                    );
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit breaker reset");
                }
            );
    }

    public async Task<string> UploadAsync(
        string containerName,
        string blobName,
        Stream content,
        string contentType)
    {
        try
        {
            return await _circuitBreakerPolicy.ExecuteAsync(async () =>
            {
                var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
                await containerClient.CreateIfNotExistsAsync();

                var blobClient = containerClient.GetBlobClient(blobName);

                var uploadOptions = new BlobUploadOptions
                {
                    HttpHeaders = new BlobHttpHeaders
                    {
                        ContentType = contentType
                    },
                    Metadata = new Dictionary<string, string>
                    {
                        ["UploadedBy"] = "BlobRepository",
                        ["UploadTime"] = DateTime.UtcNow.ToString("o")
                    }
                };

                var response = await blobClient.UploadAsync(content, uploadOptions);

                _logger.LogInformation(
                    "Uploaded blob {BlobName} to container {ContainerName}. ETag: {ETag}",
                    blobName,
                    containerName,
                    response.Value.ETag
                );

                return response.Value.ETag.ToString();
            });
        }
        catch (BrokenCircuitException ex)
        {
            _logger.LogError("Circuit breaker is open. Upload failed for {BlobName}", blobName);
            throw new InvalidOperationException("Storage service is temporarily unavailable", ex);
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            _logger.LogWarning("Blob {BlobName} already exists in {ContainerName}", blobName, containerName);
            throw new InvalidOperationException($"Blob '{blobName}' already exists", ex);
        }
    }

    public async Task<Stream> DownloadAsync(string containerName, string blobName)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobClient = containerClient.GetBlobClient(blobName);

            var response = await blobClient.DownloadStreamingAsync();

            _logger.LogInformation(
                "Downloaded blob {BlobName} from container {ContainerName}",
                blobName,
                containerName
            );

            return response.Value.Content;
        }
        catch (RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogWarning(
                "Blob {BlobName} not found in container {ContainerName}",
                blobName,
                containerName
            );
            throw new FileNotFoundException($"Blob '{blobName}' not found", ex);
        }
    }

    public async Task<bool> ExistsAsync(string containerName, string blobName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var response = await blobClient.ExistsAsync();
        return response.Value;
    }

    public async Task DeleteAsync(string containerName, string blobName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        await blobClient.DeleteIfExistsAsync();

        _logger.LogInformation(
            "Deleted blob {BlobName} from container {ContainerName}",
            blobName,
            containerName
        );
    }

    public async Task<IEnumerable<string>> ListBlobsAsync(string containerName, string prefix = "")
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);

        var blobs = new List<string>();

        await foreach (var blobItem in containerClient.GetBlobsAsync(prefix: prefix))
        {
            blobs.Add(blobItem.Name);
        }

        return blobs;
    }

    public async Task<BlobMetadata> GetMetadataAsync(string containerName, string blobName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var properties = await blobClient.GetPropertiesAsync();

        return new BlobMetadata(
            Name: blobName,
            Size: properties.Value.ContentLength,
            LastModified: properties.Value.LastModified,
            ContentType: properties.Value.ContentType,
            ETag: properties.Value.ETag.ToString()
        );
    }
}
```

### ITableRepository Interface

```csharp
public interface ITableRepository<T> where T : class, ITableEntity, new()
{
    Task<T> GetAsync(string partitionKey, string rowKey);
    Task<IEnumerable<T>> QueryAsync(string filter);
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(string partitionKey, string rowKey);
    Task<int> BatchInsertAsync(IEnumerable<T> entities);
}
```

### TableRepository Implementation

```csharp
using Azure.Data.Tables;
using Azure;
using Microsoft.Extensions.Logging;

public class TableRepository<T> : ITableRepository<T> where T : class, ITableEntity, new()
{
    private readonly TableClient _tableClient;
    private readonly ILogger<TableRepository<T>> _logger;

    public TableRepository(
        TableServiceClient tableServiceClient,
        string tableName,
        ILogger<TableRepository<T>> logger)
    {
        _tableClient = tableServiceClient.GetTableClient(tableName);
        _tableClient.CreateIfNotExists();
        _logger = logger;
    }

    public async Task<T> GetAsync(string partitionKey, string rowKey)
    {
        try
        {
            var response = await _tableClient.GetEntityAsync<T>(partitionKey, rowKey);
            return response.Value;
        }
        catch (RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogWarning(
                "Entity not found: PK={PartitionKey}, RK={RowKey}",
                partitionKey,
                rowKey
            );
            throw new KeyNotFoundException($"Entity with PK '{partitionKey}' and RK '{rowKey}' not found", ex);
        }
    }

    public async Task<IEnumerable<T>> QueryAsync(string filter)
    {
        var entities = new List<T>();

        await foreach (var entity in _tableClient.QueryAsync<T>(filter))
        {
            entities.Add(entity);
        }

        _logger.LogInformation("Query returned {Count} entities", entities.Count);
        return entities;
    }

    public async Task AddAsync(T entity)
    {
        try
        {
            await _tableClient.AddEntityAsync(entity);

            _logger.LogInformation(
                "Added entity: PK={PartitionKey}, RK={RowKey}",
                entity.PartitionKey,
                entity.RowKey
            );
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            _logger.LogWarning(
                "Entity already exists: PK={PartitionKey}, RK={RowKey}",
                entity.PartitionKey,
                entity.RowKey
            );
            throw new InvalidOperationException("Entity already exists", ex);
        }
    }

    public async Task UpdateAsync(T entity)
    {
        await _tableClient.UpdateEntityAsync(entity, ETag.All, TableUpdateMode.Replace);

        _logger.LogInformation(
            "Updated entity: PK={PartitionKey}, RK={RowKey}",
            entity.PartitionKey,
            entity.RowKey
        );
    }

    public async Task DeleteAsync(string partitionKey, string rowKey)
    {
        await _tableClient.DeleteEntityAsync(partitionKey, rowKey);

        _logger.LogInformation(
            "Deleted entity: PK={PartitionKey}, RK={RowKey}",
            partitionKey,
            rowKey
        );
    }

    public async Task<int> BatchInsertAsync(IEnumerable<T> entities)
    {
        var grouped = entities.GroupBy(e => e.PartitionKey);
        var totalInserted = 0;

        foreach (var group in grouped)
        {
            var batches = group.Chunk(100); // Max 100 per batch

            foreach (var batch in batches)
            {
                var transactionActions = batch.Select(entity =>
                    new TableTransactionAction(TableTransactionActionType.Add, entity)
                ).ToList();

                await _tableClient.SubmitTransactionAsync(transactionActions);
                totalInserted += transactionActions.Count;
            }
        }

        _logger.LogInformation("Batch inserted {Count} entities", totalInserted);
        return totalInserted;
    }
}
```

## Retry Policies

### Built-In Azure SDK Retry

```csharp
// Configured in BlobClientOptions (recommended)
var clientOptions = new BlobClientOptions
{
    Retry =
    {
        MaxRetries = 5,
        Delay = TimeSpan.FromMilliseconds(100),
        MaxDelay = TimeSpan.FromSeconds(10),
        Mode = RetryMode.Exponential,  // Exponential backoff
        NetworkTimeout = TimeSpan.FromSeconds(100)
    }
};

var blobServiceClient = new BlobServiceClient(accountUri, credential, clientOptions);
```

**Retry behavior**:
```
Attempt 1: Immediate
Attempt 2: 100ms delay
Attempt 3: 200ms delay
Attempt 4: 400ms delay
Attempt 5: 800ms delay
Attempt 6: 1600ms delay (capped at 10s MaxDelay)
```

### Polly Retry Policy (Advanced)

```csharp
using Polly;
using Polly.Retry;

public class ResilientBlobRepository : IBlobRepository
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<ResilientBlobRepository> _logger;
    private readonly AsyncRetryPolicy _retryPolicy;

    public ResilientBlobRepository(
        BlobServiceClient blobServiceClient,
        ILogger<ResilientBlobRepository> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;

        // Custom retry policy with Polly
        _retryPolicy = Policy
            .Handle<RequestFailedException>(ex =>
                ex.Status == 429 ||  // Throttling
                ex.Status == 503 ||  // Service unavailable
                ex.Status >= 500)    // Server errors
            .WaitAndRetryAsync(
                retryCount: 5,
                sleepDurationProvider: retryAttempt =>
                {
                    // Exponential backoff with jitter
                    var baseDelay = TimeSpan.FromMilliseconds(100);
                    var exponentialDelay = TimeSpan.FromMilliseconds(
                        baseDelay.TotalMilliseconds * Math.Pow(2, retryAttempt)
                    );
                    var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100));
                    return exponentialDelay + jitter;
                },
                onRetry: (exception, sleepDuration, retryCount, context) =>
                {
                    _logger.LogWarning(
                        "Retry {RetryCount} after {SleepDuration}ms due to {Exception}",
                        retryCount,
                        sleepDuration.TotalMilliseconds,
                        exception.GetType().Name
                    );
                }
            );
    }

    public async Task<string> UploadAsync(
        string containerName,
        string blobName,
        Stream content,
        string contentType)
    {
        return await _retryPolicy.ExecuteAsync(async () =>
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobClient = containerClient.GetBlobClient(blobName);

            var response = await blobClient.UploadAsync(content, overwrite: true);
            return response.Value.ETag.ToString();
        });
    }
}
```

## Connection Pooling

### HttpClient Configuration

```csharp
// ✅ Good: Singleton HttpClient with connection pooling
public static class HttpClientConfiguration
{
    private static readonly Lazy<HttpClient> _httpClient = new(() =>
    {
        var handler = new SocketsHttpHandler
        {
            PooledConnectionLifetime = TimeSpan.FromMinutes(5),
            PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
            MaxConnectionsPerServer = 100,
            EnableMultipleHttp2Connections = true
        };

        return new HttpClient(handler)
        {
            Timeout = TimeSpan.FromMinutes(5)
        };
    });

    public static HttpClient Instance => _httpClient.Value;
}

// Use in Azure SDK clients
var transport = new HttpClientTransport(HttpClientConfiguration.Instance);

var clientOptions = new BlobClientOptions
{
    Transport = transport
};

var blobServiceClient = new BlobServiceClient(accountUri, credential, clientOptions);
```

**Benefits**:
- Reuses TCP connections (avoids socket exhaustion)
- Reduces connection establishment overhead (~50ms per request)
- Improves throughput by 2-5x

## Testing Strategies

### Unit Testing with Mocks

```csharp
using Moq;
using Xunit;

public class BlobRepositoryTests
{
    [Fact]
    public async Task UploadAsync_ShouldReturnETag_WhenSuccessful()
    {
        // Arrange
        var mockBlobServiceClient = new Mock<BlobServiceClient>();
        var mockContainerClient = new Mock<BlobContainerClient>();
        var mockBlobClient = new Mock<BlobClient>();

        var expectedETag = new ETag("\"0x8DCFD12345\"");

        mockBlobServiceClient
            .Setup(x => x.GetBlobContainerClient(It.IsAny<string>()))
            .Returns(mockContainerClient.Object);

        mockContainerClient
            .Setup(x => x.GetBlobClient(It.IsAny<string>()))
            .Returns(mockBlobClient.Object);

        mockBlobClient
            .Setup(x => x.UploadAsync(It.IsAny<Stream>(), It.IsAny<BlobUploadOptions>(), default))
            .ReturnsAsync(Response.FromValue(
                BlobsModelFactory.BlobContentInfo(expectedETag, DateTime.UtcNow, new byte[0], "", 0),
                Mock.Of<Response>()
            ));

        var logger = Mock.Of<ILogger<BlobRepository>>();
        var repository = new BlobRepository(mockBlobServiceClient.Object, logger);

        // Act
        var eTag = await repository.UploadAsync(
            "test-container",
            "test-blob.txt",
            new MemoryStream(),
            "text/plain"
        );

        // Assert
        Assert.Equal(expectedETag.ToString(), eTag);
    }
}
```

### Integration Testing with Azurite

```csharp
using Azure.Storage.Blobs;
using Xunit;

// Azurite connection string (local emulator)
// docker run -p 10000:10000 mcr.microsoft.com/azure-storage/azurite azurite-blob --blobHost 0.0.0.0

public class BlobRepositoryIntegrationTests : IDisposable
{
    private const string AzuriteConnectionString =
        "DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;";

    private readonly BlobServiceClient _blobServiceClient;
    private readonly BlobRepository _repository;

    public BlobRepositoryIntegrationTests()
    {
        _blobServiceClient = new BlobServiceClient(AzuriteConnectionString);
        var logger = Mock.Of<ILogger<BlobRepository>>();
        _repository = new BlobRepository(_blobServiceClient, logger);
    }

    [Fact]
    public async Task UploadAndDownload_ShouldWorkEndToEnd()
    {
        // Arrange
        var containerName = $"test-{Guid.NewGuid()}";
        var blobName = "test-file.txt";
        var content = "Hello, Azurite!";

        using var uploadStream = new MemoryStream(Encoding.UTF8.GetBytes(content));

        // Act: Upload
        var eTag = await _repository.UploadAsync(
            containerName,
            blobName,
            uploadStream,
            "text/plain"
        );

        // Act: Download
        using var downloadStream = await _repository.DownloadAsync(containerName, blobName);
        using var reader = new StreamReader(downloadStream);
        var downloadedContent = await reader.ReadToEndAsync();

        // Assert
        Assert.NotNull(eTag);
        Assert.Equal(content, downloadedContent);
    }

    public void Dispose()
    {
        // Cleanup test containers
        foreach (var container in _blobServiceClient.GetBlobContainers(prefix: "test-"))
        {
            _blobServiceClient.DeleteBlobContainer(container.Name);
        }
    }
}
```

## Best Practices

### 1. Use Singleton Clients

```csharp
// ✅ Good: Singleton (DI container lifetime)
services.AddSingleton(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var accountUrl = config["Azure:Storage:AccountUrl"];
    return new BlobServiceClient(new Uri(accountUrl!), new DefaultAzureCredential());
});

// ❌ Bad: Creating new client per request
public async Task UploadAsync()
{
    var client = new BlobServiceClient(...);  // ❌ Creates new HttpClient
    await client.GetBlobContainerClient("test").GetBlobClient("file").UploadAsync(...);
}
```

### 2. Configure Retry Policies

```csharp
// Always configure retry policies
var clientOptions = new BlobClientOptions
{
    Retry =
    {
        MaxRetries = 5,
        Mode = RetryMode.Exponential
    }
};
```

### 3. Enable Distributed Tracing

```csharp
var clientOptions = new BlobClientOptions
{
    Diagnostics =
    {
        IsDistributedTracingEnabled = true,
        ApplicationId = "MyApp/1.0"
    }
};
```

### 4. Implement Circuit Breakers

```csharp
// Use Polly circuit breaker for resilience
var circuitBreakerPolicy = Policy
    .Handle<RequestFailedException>(ex => ex.Status >= 500)
    .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
```

### 5. Log Correlation IDs

```csharp
// Include correlation IDs for troubleshooting
_logger.LogInformation(
    "Operation {OperationName} completed. CorrelationId: {CorrelationId}",
    "UploadBlob",
    Activity.Current?.Id ?? Guid.NewGuid().ToString()
);
```

## References

**Microsoft Learn Documentation**:
- [Azure SDK for .NET](https://learn.microsoft.com/dotnet/azure/sdk/azure-sdk-for-dotnet)
- [Dependency Injection](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
- [Retry Policies](https://learn.microsoft.com/azure/architecture/best-practices/retry-service-specific)
- [Connection Pooling](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)
- [Testing with Azurite](https://learn.microsoft.com/azure/storage/common/storage-use-azurite)

## Navigation

- **Previous**: `performance-optimization.md`
- **Next**: `governance.md`
- **Related**: `../02-core-concepts/security.md` - Authentication setup

## See Also

- `observability.md` - Client instrumentation and logging
- `performance-optimization.md` - Client-side optimization
- `../03-blob-storage/blob-concurrency.md` - Retry and error handling patterns
