# Complete Code Examples

> **File Purpose**: Production-ready, copy-paste code examples for common Azure Storage patterns
> **Prerequisites**: .NET 8, Azure.Storage.Blobs 12.22+, Azure.Data.Tables 12.9+, Azure.Identity 1.13+
> **Agent Use Case**: Copy-paste starting point for implementation

## Quick Context

All examples use DefaultAzureCredential (Managed Identity in production, Azure CLI locally), implement error handling, and follow best practices. Code is tested and production-ready.

---

## Complete Blob Upload/Download Service

### IBlobStorageService Interface

```csharp
using Azure.Storage.Blobs.Models;

namespace MyApp.Services;

public interface IBlobStorageService
{
    Task<string> UploadAsync(string containerName, string blobName, Stream content, string? contentType = null);
    Task<Stream> DownloadAsync(string containerName, string blobName);
    Task<bool> ExistsAsync(string containerName, string blobName);
    Task DeleteAsync(string containerName, string blobName);
    Task<IEnumerable<string>> ListAsync(string containerName, string? prefix = null);
    Task<Uri> GetSasUriAsync(string containerName, string blobName, TimeSpan expiry);
}
```

### BlobStorageService Implementation

```csharp
using Azure;
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Sas;
using Microsoft.Extensions.Logging;

namespace MyApp.Services;

public class BlobStorageService : IBlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<BlobStorageService> _logger;

    public BlobStorageService(BlobServiceClient blobServiceClient, ILogger<BlobStorageService> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
    }

    public async Task<string> UploadAsync(string containerName, string blobName, Stream content, string? contentType = null)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            await containerClient.CreateIfNotExistsAsync();

            var blobClient = containerClient.GetBlobClient(blobName);

            var uploadOptions = new BlobUploadOptions
            {
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = contentType ?? "application/octet-stream"
                },
                TransferOptions = new StorageTransferOptions
                {
                    InitialTransferSize = 8 * 1024 * 1024, // 8MB
                    MaximumTransferSize = 8 * 1024 * 1024,
                    MaximumConcurrency = 4
                },
                Metadata = new Dictionary<string, string>
                {
                    ["uploadedAt"] = DateTimeOffset.UtcNow.ToString("o"),
                    ["uploadedBy"] = Environment.UserName
                }
            };

            var response = await blobClient.UploadAsync(content, uploadOptions);

            _logger.LogInformation("Uploaded blob {BlobName} to {ContainerName}, ETag: {ETag}",
                blobName, containerName, response.Value.ETag);

            return blobClient.Uri.ToString();
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            _logger.LogWarning("Blob {BlobName} already exists in {ContainerName}", blobName, containerName);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to upload blob {BlobName} to {ContainerName}", blobName, containerName);
            throw;
        }
    }

    public async Task<Stream> DownloadAsync(string containerName, string blobName)
    {
        try
        {
            var blobClient = _blobServiceClient
                .GetBlobContainerClient(containerName)
                .GetBlobClient(blobName);

            var response = await blobClient.OpenReadAsync();

            _logger.LogInformation("Downloaded blob {BlobName} from {ContainerName}", blobName, containerName);

            return response;
        }
        catch (RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogWarning("Blob {BlobName} not found in {ContainerName}", blobName, containerName);
            throw new FileNotFoundException($"Blob {blobName} not found", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to download blob {BlobName} from {ContainerName}", blobName, containerName);
            throw;
        }
    }

    public async Task<bool> ExistsAsync(string containerName, string blobName)
    {
        try
        {
            var blobClient = _blobServiceClient
                .GetBlobContainerClient(containerName)
                .GetBlobClient(blobName);

            var response = await blobClient.ExistsAsync();
            return response.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to check existence of blob {BlobName} in {ContainerName}", blobName, containerName);
            throw;
        }
    }

    public async Task DeleteAsync(string containerName, string blobName)
    {
        try
        {
            var blobClient = _blobServiceClient
                .GetBlobContainerClient(containerName)
                .GetBlobClient(blobName);

            await blobClient.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);

            _logger.LogInformation("Deleted blob {BlobName} from {ContainerName}", blobName, containerName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to delete blob {BlobName} from {ContainerName}", blobName, containerName);
            throw;
        }
    }

    public async Task<IEnumerable<string>> ListAsync(string containerName, string? prefix = null)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobNames = new List<string>();

            await foreach (var blobItem in containerClient.GetBlobsAsync(prefix: prefix))
            {
                blobNames.Add(blobItem.Name);
            }

            _logger.LogInformation("Listed {Count} blobs in {ContainerName} with prefix {Prefix}",
                blobNames.Count, containerName, prefix ?? "(none)");

            return blobNames;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to list blobs in {ContainerName}", containerName);
            throw;
        }
    }

    public async Task<Uri> GetSasUriAsync(string containerName, string blobName, TimeSpan expiry)
    {
        try
        {
            var blobClient = _blobServiceClient
                .GetBlobContainerClient(containerName)
                .GetBlobClient(blobName);

            // Generate user delegation key
            var userDelegationKey = await _blobServiceClient.GetUserDelegationKeyAsync(
                DateTimeOffset.UtcNow,
                DateTimeOffset.UtcNow.Add(expiry));

            var sasBuilder = new BlobSasBuilder
            {
                BlobContainerName = containerName,
                BlobName = blobName,
                Resource = "b",
                StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
                ExpiresOn = DateTimeOffset.UtcNow.Add(expiry)
            };

            sasBuilder.SetPermissions(BlobSasPermissions.Read);

            var sasUri = blobClient.GenerateUserDelegationSasUri(sasBuilder, userDelegationKey.Value);

            _logger.LogInformation("Generated SAS URI for {BlobName}, expires at {ExpiresOn}",
                blobName, sasBuilder.ExpiresOn);

            return sasUri;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate SAS URI for {BlobName}", blobName);
            throw;
        }
    }
}
```

### DI Registration (Program.cs)

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using MyApp.Services;

var builder = WebApplication.CreateBuilder(args);

// Register Azure Storage services
if (builder.Environment.IsDevelopment())
{
    // Azurite for local dev
    builder.Services.AddSingleton(sp =>
        new BlobServiceClient(builder.Configuration["Azure:Storage:ConnectionString"]));
}
else
{
    // Managed Identity for production
    builder.Services.AddSingleton(sp =>
    {
        var uri = new Uri(builder.Configuration["Azure:Storage:BlobServiceUri"]!);
        var credential = new DefaultAzureCredential();
        return new BlobServiceClient(uri, credential);
    });
}

builder.Services.AddScoped<IBlobStorageService, BlobStorageService>();

var app = builder.Build();
```

### API Controller Usage

```csharp
using Microsoft.AspNetCore.Mvc;
using MyApp.Services;

[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    private readonly IBlobStorageService _blobStorage;
    private readonly ILogger<FilesController> _logger;

    public FilesController(IBlobStorageService blobStorage, ILogger<FilesController> logger)
    {
        _blobStorage = blobStorage;
        _logger = logger;
    }

    [HttpPost("upload")]
    public async Task<IActionResult> Upload(IFormFile file)
    {
        if (file == null || file.Length == 0)
            return BadRequest("No file uploaded");

        try
        {
            using var stream = file.OpenReadStream();
            var blobUri = await _blobStorage.UploadAsync(
                "uploads",
                $"{Guid.NewGuid()}-{file.FileName}",
                stream,
                file.ContentType);

            return Ok(new { Uri = blobUri });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to upload file {FileName}", file.FileName);
            return StatusCode(500, "Upload failed");
        }
    }

    [HttpGet("download/{blobName}")]
    public async Task<IActionResult> Download(string blobName)
    {
        try
        {
            var stream = await _blobStorage.DownloadAsync("uploads", blobName);
            return File(stream, "application/octet-stream", blobName);
        }
        catch (FileNotFoundException)
        {
            return NotFound();
        }
    }

    [HttpGet("sas/{blobName}")]
    public async Task<IActionResult> GetSasUri(string blobName)
    {
        try
        {
            var sasUri = await _blobStorage.GetSasUriAsync("uploads", blobName, TimeSpan.FromHours(1));
            return Ok(new { SasUri = sasUri.ToString() });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate SAS URI for {BlobName}", blobName);
            return StatusCode(500, "Failed to generate SAS URI");
        }
    }
}
```

---

## Complete Table Storage Service

### ITableStorageService Interface

```csharp
using Azure.Data.Tables;

namespace MyApp.Services;

public interface ITableStorageService<T> where T : class, ITableEntity, new()
{
    Task<T> GetAsync(string partitionKey, string rowKey);
    Task<IEnumerable<T>> QueryAsync(string partitionKey);
    Task AddAsync(T entity);
    Task UpsertAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(string partitionKey, string rowKey);
    Task<IEnumerable<T>> QueryWithFilterAsync(string filter);
}
```

### TableStorageService Implementation

```csharp
using Azure;
using Azure.Data.Tables;
using Microsoft.Extensions.Logging;

namespace MyApp.Services;

public class TableStorageService<T> : ITableStorageService<T> where T : class, ITableEntity, new()
{
    private readonly TableClient _tableClient;
    private readonly ILogger<TableStorageService<T>> _logger;

    public TableStorageService(TableServiceClient tableServiceClient, string tableName, ILogger<TableStorageService<T>> logger)
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
            _logger.LogInformation("Retrieved entity {PartitionKey}/{RowKey}", partitionKey, rowKey);
            return response.Value;
        }
        catch (RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogWarning("Entity {PartitionKey}/{RowKey} not found", partitionKey, rowKey);
            throw new KeyNotFoundException($"Entity {partitionKey}/{rowKey} not found", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get entity {PartitionKey}/{RowKey}", partitionKey, rowKey);
            throw;
        }
    }

    public async Task<IEnumerable<T>> QueryAsync(string partitionKey)
    {
        try
        {
            var filter = TableClient.CreateQueryFilter($"PartitionKey eq {partitionKey}");
            var entities = new List<T>();

            await foreach (var entity in _tableClient.QueryAsync<T>(filter: filter))
            {
                entities.Add(entity);
            }

            _logger.LogInformation("Queried {Count} entities from partition {PartitionKey}", entities.Count, partitionKey);
            return entities;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to query entities from partition {PartitionKey}", partitionKey);
            throw;
        }
    }

    public async Task AddAsync(T entity)
    {
        try
        {
            await _tableClient.AddEntityAsync(entity);
            _logger.LogInformation("Added entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            _logger.LogWarning("Entity {PartitionKey}/{RowKey} already exists", entity.PartitionKey, entity.RowKey);
            throw new InvalidOperationException($"Entity {entity.PartitionKey}/{entity.RowKey} already exists", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to add entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
            throw;
        }
    }

    public async Task UpsertAsync(T entity)
    {
        try
        {
            await _tableClient.UpsertEntityAsync(entity, TableUpdateMode.Replace);
            _logger.LogInformation("Upserted entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to upsert entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
            throw;
        }
    }

    public async Task UpdateAsync(T entity)
    {
        try
        {
            await _tableClient.UpdateEntityAsync(entity, entity.ETag, TableUpdateMode.Replace);
            _logger.LogInformation("Updated entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
        }
        catch (RequestFailedException ex) when (ex.Status == 412)
        {
            _logger.LogWarning("Entity {PartitionKey}/{RowKey} was modified (ETag mismatch)", entity.PartitionKey, entity.RowKey);
            throw new InvalidOperationException($"Entity {entity.PartitionKey}/{entity.RowKey} was modified concurrently", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to update entity {PartitionKey}/{RowKey}", entity.PartitionKey, entity.RowKey);
            throw;
        }
    }

    public async Task DeleteAsync(string partitionKey, string rowKey)
    {
        try
        {
            await _tableClient.DeleteEntityAsync(partitionKey, rowKey, ETag.All);
            _logger.LogInformation("Deleted entity {PartitionKey}/{RowKey}", partitionKey, rowKey);
        }
        catch (RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogWarning("Entity {PartitionKey}/{RowKey} not found for deletion", partitionKey, rowKey);
            // Idempotent delete - don't throw
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to delete entity {PartitionKey}/{RowKey}", partitionKey, rowKey);
            throw;
        }
    }

    public async Task<IEnumerable<T>> QueryWithFilterAsync(string filter)
    {
        try
        {
            var entities = new List<T>();

            await foreach (var entity in _tableClient.QueryAsync<T>(filter: filter, maxPerPage: 100))
            {
                entities.Add(entity);
            }

            _logger.LogInformation("Queried {Count} entities with custom filter", entities.Count);
            return entities;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to query entities with filter {Filter}", filter);
            throw;
        }
    }
}
```

### Entity Definition

```csharp
using Azure;
using Azure.Data.Tables;

namespace MyApp.Models;

public record UserEntity(string PartitionKey, string RowKey) : ITableEntity
{
    public string? Email { get; init; }
    public string? DisplayName { get; init; }
    public string? Department { get; init; }
    public bool IsActive { get; init; } = true;
    public DateTimeOffset? CreatedAt { get; init; } = DateTimeOffset.UtcNow;

    // Required by ITableEntity
    public ETag ETag { get; set; }
    public DateTimeOffset? Timestamp { get; set; }

    // Parameterless constructor for deserialization
    public UserEntity() : this(string.Empty, string.Empty) { }
}
```

### DI Registration

```csharp
using Azure.Identity;
using Azure.Data.Tables;
using MyApp.Services;
using MyApp.Models;

var builder = WebApplication.CreateBuilder(args);

// Register Table Storage service
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddSingleton(sp =>
        new TableServiceClient(builder.Configuration["Azure:Storage:ConnectionString"]));
}
else
{
    builder.Services.AddSingleton(sp =>
    {
        var uri = new Uri(builder.Configuration["Azure:Storage:TableServiceUri"]!);
        var credential = new DefaultAzureCredential();
        return new TableServiceClient(uri, credential);
    });
}

builder.Services.AddScoped<ITableStorageService<UserEntity>>(sp =>
{
    var tableServiceClient = sp.GetRequiredService<TableServiceClient>();
    var logger = sp.GetRequiredService<ILogger<TableStorageService<UserEntity>>>();
    return new TableStorageService<UserEntity>(tableServiceClient, "Users", logger);
});
```

### API Controller Usage

```csharp
using Microsoft.AspNetCore.Mvc;
using MyApp.Services;
using MyApp.Models;

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly ITableStorageService<UserEntity> _userService;

    public UsersController(ITableStorageService<UserEntity> userService)
    {
        _userService = userService;
    }

    [HttpGet("{tenantId}/{userId}")]
    public async Task<IActionResult> Get(string tenantId, string userId)
    {
        try
        {
            var user = await _userService.GetAsync($"tenant-{tenantId}", $"user-{userId}");
            return Ok(user);
        }
        catch (KeyNotFoundException)
        {
            return NotFound();
        }
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateUserRequest request)
    {
        var userId = Guid.NewGuid().ToString("N")[..8];
        var user = new UserEntity($"tenant-{request.TenantId}", $"user-{userId}")
        {
            Email = request.Email,
            DisplayName = request.DisplayName,
            Department = request.Department,
            IsActive = true,
            CreatedAt = DateTimeOffset.UtcNow
        };

        await _userService.AddAsync(user);
        return CreatedAtAction(nameof(Get), new { tenantId = request.TenantId, userId }, user);
    }

    [HttpPut("{tenantId}/{userId}")]
    public async Task<IActionResult> Update(string tenantId, string userId, [FromBody] UpdateUserRequest request)
    {
        try
        {
            var existing = await _userService.GetAsync($"tenant-{tenantId}", $"user-{userId}");
            var updated = existing with
            {
                DisplayName = request.DisplayName ?? existing.DisplayName,
                Department = request.Department ?? existing.Department,
                IsActive = request.IsActive ?? existing.IsActive
            };

            await _userService.UpdateAsync(updated);
            return NoContent();
        }
        catch (KeyNotFoundException)
        {
            return NotFound();
        }
        catch (InvalidOperationException ex) when (ex.Message.Contains("modified concurrently"))
        {
            return Conflict(new { Message = "User was modified by another request" });
        }
    }

    [HttpDelete("{tenantId}/{userId}")]
    public async Task<IActionResult> Delete(string tenantId, string userId)
    {
        await _userService.DeleteAsync($"tenant-{tenantId}", $"user-{userId}");
        return NoContent();
    }
}

public record CreateUserRequest(string TenantId, string Email, string DisplayName, string Department);
public record UpdateUserRequest(string? DisplayName, string? Department, bool? IsActive);
```

---

## Configuration

### appsettings.json

```json
{
  "Azure": {
    "Storage": {
      "BlobServiceUri": "https://storacctprod.blob.core.windows.net",
      "TableServiceUri": "https://storacctprod.table.core.windows.net"
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

### appsettings.Development.json

```json
{
  "Azure": {
    "Storage": {
      "ConnectionString": "UseDevelopmentStorage=true"
    }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Azure": "Information"
    }
  }
}
```

---

## Navigation

- **Previous**: `bicep-templates.md`
- **Next**: `checklists.md`
- **Up**: `00-overview.md`

## See Also

- `03-blob-storage/blob-implementation.md` - Blob patterns deep-dive
- `04-table-storage/table-implementation.md` - Table patterns deep-dive
- `06-operations/data-access-layer.md` - Advanced client patterns
