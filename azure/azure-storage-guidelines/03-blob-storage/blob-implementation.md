# Blob Storage Implementation Patterns

> **File Purpose**: Practical upload/download patterns with streaming, chunking, and metadata
> **Prerequisites**: `01-quick-start/authentication-setup.md`, `03-blob-storage/blob-fundamentals.md`
> **Agent Use Case**: Reference when implementing blob upload/download operations

## Quick Context

This guide covers production-ready patterns for blob operations: streaming large files, parallel uploads, conditional operations with ETags, metadata management, and error handling. All examples use .NET 8 with DefaultAzureCredential.

**Key principle**: Use parallel transfers for large files (>4MB), stream data to avoid memory bloat, and always handle transient failures with retries.

## Prerequisites

```bash
dotnet add package Azure.Storage.Blobs --version 12.22.0
dotnet add package Azure.Identity --version 1.13.0
```

## Basic Blob Upload

### Simple Upload (Small Files <256MB)

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

var container = blobServiceClient.GetBlobContainerClient("app-data");
await container.CreateIfNotExistsAsync();

var blobClient = container.GetBlobClient("documents/report.pdf");

// Upload from local file
await blobClient.UploadAsync("./local-report.pdf", overwrite: true);

// Upload from stream
using var stream = File.OpenRead("./local-report.pdf");
await blobClient.UploadAsync(stream, overwrite: true);

// Upload from memory (BinaryData)
var data = new BinaryData("Text content");
await blobClient.UploadAsync(data, overwrite: true);
```

**Why it works**: Simple API for files under 256MB. SDK handles chunking automatically.

**Common mistakes**:
- Not setting `overwrite: true` (fails if blob exists)
- Loading entire file into memory (use streams)

### Upload with Content Type and Metadata

```csharp
var uploadOptions = new BlobUploadOptions
{
    HttpHeaders = new BlobHttpHeaders
    {
        ContentType = "application/pdf",
        ContentLanguage = "en-US",
        CacheControl = "public, max-age=86400"
    },
    Metadata = new Dictionary<string, string>
    {
        ["uploadedBy"] = "user@example.com",
        ["department"] = "finance",
        ["documentType"] = "quarterly-report",
        ["uploadTimestamp"] = DateTimeOffset.UtcNow.ToString("o")
    },
    Tags = new Dictionary<string, string>
    {
        ["classification"] = "internal",
        ["retention"] = "7-years"
    }
};

using var stream = File.OpenRead("./report.pdf");
await blobClient.UploadAsync(stream, uploadOptions);
```

**Why it works**: Content-Type enables proper browser rendering. Metadata enables searchability. Tags enable lifecycle policies.

**Metadata vs Tags**:
- **Metadata**: Custom key-value pairs, max 8KB total, included in GET operations
- **Tags**: Index for lifecycle management, queryable, up to 10 tags, separate API call to retrieve

## Large File Upload (Parallel Chunking)

### Optimized Parallel Upload

```csharp
var uploadOptions = new BlobUploadOptions
{
    TransferOptions = new StorageTransferOptions
    {
        // Upload 8MB chunks in parallel (4 concurrent uploads)
        InitialTransferSize = 8 * 1024 * 1024,
        MaximumTransferSize = 8 * 1024 * 1024,
        MaximumConcurrency = 4
    },
    HttpHeaders = new BlobHttpHeaders
    {
        ContentType = "video/mp4"
    },
    ProgressHandler = new Progress<long>(bytesTransferred =>
    {
        Console.WriteLine($"Uploaded: {bytesTransferred / (1024.0 * 1024.0):F2} MB");
    })
};

using var fileStream = File.OpenRead("./large-video.mp4");
await blobClient.UploadAsync(fileStream, uploadOptions);
```

**Why it works**:
- 8MB chunks maximize throughput without exceeding single-block limit
- 4 concurrent uploads balance speed and resource usage
- Progress handler enables UI updates

**Performance tuning**:
- Files < 4MB: Single upload (no chunking)
- Files 4MB-1GB: 8MB chunks, 4-8 concurrent uploads
- Files > 1GB: 16MB chunks, 8-16 concurrent uploads (monitor network)

**Common mistakes**:
- Setting `MaximumConcurrency` too high (network saturation, 429 throttling)
- Using chunk sizes < 4MB (too many requests) or > 100MB (memory bloat)

### Resume Failed Uploads (Advanced)

```csharp
// Upload with staging blocks for manual resume control
var blockBlobClient = container.GetBlockBlobClient("large-file.bin");

var blockSize = 8 * 1024 * 1024; // 8MB
var blockIds = new List<string>();

using var fileStream = File.OpenRead("./large-file.bin");
var buffer = new byte[blockSize];
int bytesRead;
int blockNumber = 0;

while ((bytesRead = await fileStream.ReadAsync(buffer, 0, blockSize)) > 0)
{
    var blockId = Convert.ToBase64String(Encoding.UTF8.GetBytes($"block-{blockNumber:D6}"));
    blockIds.Add(blockId);

    try
    {
        using var blockStream = new MemoryStream(buffer, 0, bytesRead);
        await blockBlobClient.StageBlockAsync(blockId, blockStream);
        Console.WriteLine($"Staged block {blockNumber}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Failed to stage block {blockNumber}: {ex.Message}");
        // Retry logic here
        throw;
    }

    blockNumber++;
}

// Commit all blocks
await blockBlobClient.CommitBlockListAsync(blockIds);
Console.WriteLine("Upload complete");
```

**Why it works**: Staging blocks individually allows retry of failed chunks without re-uploading the entire file.

**Use when**: Uploading multi-GB files over unreliable networks.

## Blob Download

### Simple Download

```csharp
var blobClient = container.GetBlobClient("documents/report.pdf");

// Download to local file
await blobClient.DownloadToAsync("./downloaded-report.pdf");

// Download to stream
using var downloadStream = await blobClient.OpenReadAsync();
using var fileStream = File.Create("./downloaded-report.pdf");
await downloadStream.CopyToAsync(fileStream);

// Download to memory (BinaryData)
var downloadResult = await blobClient.DownloadContentAsync();
var content = downloadResult.Value.Content;
Console.WriteLine(content.ToString());
```

**Common mistakes**:
- Using `DownloadContentAsync()` for large files (loads entire file into memory)
- Not checking if blob exists before download

### Parallel Download (Large Files)

```csharp
var downloadOptions = new BlobDownloadToOptions
{
    TransferOptions = new StorageTransferOptions
    {
        InitialTransferSize = 8 * 1024 * 1024,
        MaximumTransferSize = 8 * 1024 * 1024,
        MaximumConcurrency = 4
    },
    ProgressHandler = new Progress<long>(bytesTransferred =>
    {
        Console.WriteLine($"Downloaded: {bytesTransferred / (1024.0 * 1024.0):F2} MB");
    })
};

await blobClient.DownloadToAsync("./downloaded-large-video.mp4", downloadOptions);
```

### Conditional Download (If-Modified-Since)

```csharp
var lastDownload = DateTimeOffset.UtcNow.AddDays(-1);

try
{
    var conditions = new BlobRequestConditions
    {
        IfModifiedSince = lastDownload
    };

    var downloadResult = await blobClient.DownloadContentAsync(conditions);
    Console.WriteLine("Blob has been modified, downloaded new version");
}
catch (Azure.RequestFailedException ex) when (ex.Status == 304)
{
    Console.WriteLine("Blob not modified since last download");
}
```

**Why it works**: Avoids downloading unchanged blobs, reducing bandwidth and latency.

## Listing Blobs

### List All Blobs in Container

```csharp
var container = blobServiceClient.GetBlobContainerClient("app-data");

await foreach (var blobItem in container.GetBlobsAsync())
{
    Console.WriteLine($"Blob: {blobItem.Name}, Size: {blobItem.Properties.ContentLength} bytes");
}
```

### List Blobs with Prefix (Folder Simulation)

```csharp
// List all blobs in "images/2025/" folder
await foreach (var blobItem in container.GetBlobsAsync(prefix: "images/2025/"))
{
    Console.WriteLine($"Image: {blobItem.Name}");
}
```

### List Blobs with Metadata and Tags

```csharp
var options = new BlobTraits
{
    Metadata = true,
    Tags = true
};

await foreach (var blobItem in container.GetBlobsAsync(traits: options))
{
    Console.WriteLine($"Blob: {blobItem.Name}");
    foreach (var (key, value) in blobItem.Metadata)
    {
        Console.WriteLine($"  Metadata: {key} = {value}");
    }
    foreach (var (key, value) in blobItem.Tags)
    {
        Console.WriteLine($"  Tag: {key} = {value}");
    }
}
```

### Hierarchical Listing (Folders)

```csharp
// List "folders" by delimiter
var resultSegment = container.GetBlobsByHierarchyAsync(prefix: "documents/", delimiter: "/");

await foreach (var item in resultSegment)
{
    if (item.IsPrefix)
    {
        Console.WriteLine($"Folder: {item.Prefix}");
    }
    else
    {
        Console.WriteLine($"Blob: {item.Blob.Name}");
    }
}
```

**Why it works**: Delimiter `/` simulates folder structure. `IsPrefix` identifies "folders".

## Blob Metadata and Properties

### Set and Update Metadata

```csharp
var blobClient = container.GetBlobClient("documents/report.pdf");

// Set metadata
var metadata = new Dictionary<string, string>
{
    ["status"] = "reviewed",
    ["reviewer"] = "john.doe@example.com",
    ["reviewDate"] = DateTimeOffset.UtcNow.ToString("o")
};
await blobClient.SetMetadataAsync(metadata);

// Get metadata
var properties = await blobClient.GetPropertiesAsync();
foreach (var (key, value) in properties.Value.Metadata)
{
    Console.WriteLine($"{key}: {value}");
}
```

### Set and Query Blob Tags

```csharp
// Set tags (index for lifecycle policies)
var tags = new Dictionary<string, string>
{
    ["classification"] = "confidential",
    ["department"] = "legal",
    ["retention"] = "10-years"
};
await blobClient.SetTagsAsync(tags);

// Query blobs by tags (container-level)
var query = @"classification='confidential' AND department='legal'";
await foreach (var taggedBlob in blobServiceClient.FindBlobsByTagsAsync(query))
{
    Console.WriteLine($"Found: {taggedBlob.BlobName} in {taggedBlob.BlobContainerName}");
}
```

**Why it works**: Tags enable efficient querying across containers. Lifecycle policies can target tags.

## Blob Copy Operations

### Copy Blob Within Same Account

```csharp
var sourceBlob = container.GetBlobClient("source/file.txt");
var destBlob = container.GetBlobClient("destination/file-copy.txt");

// Start async copy
var copyOperation = await destBlob.StartCopyFromUriAsync(sourceBlob.Uri);

// Wait for completion
await copyOperation.WaitForCompletionAsync();

Console.WriteLine($"Copy status: {copyOperation.Value.CopyStatus}");
```

**Why it works**: Server-side copy (no data transfer through client). Efficient for large blobs.

### Copy Blob Across Accounts (with SAS)

```csharp
// Generate SAS for source blob (different account)
var sourceBlobClient = sourceBlobServiceClient
    .GetBlobContainerClient("source-container")
    .GetBlobClient("file.txt");

var sasBuilder = new BlobSasBuilder
{
    BlobContainerName = "source-container",
    BlobName = "file.txt",
    Resource = "b",
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)
};
sasBuilder.SetPermissions(BlobSasPermissions.Read);

var sasToken = sourceBlobClient.GenerateSasUri(sasBuilder);

// Copy to destination account
var destBlobClient = destBlobServiceClient
    .GetBlobContainerClient("dest-container")
    .GetBlobClient("file.txt");

await destBlobClient.StartCopyFromUriAsync(sasToken);
```

## Error Handling

### Retry Policy Configuration

```csharp
var clientOptions = new BlobClientOptions
{
    Retry = {
        Mode = Azure.Core.RetryMode.Exponential,
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(16),
        NetworkTimeout = TimeSpan.FromMinutes(5)
    }
};

var blobServiceClient = new BlobServiceClient(serviceUri, credential, clientOptions);
```

### Common Error Handling

```csharp
using Azure;

try
{
    await blobClient.UploadAsync(stream);
}
catch (RequestFailedException ex) when (ex.Status == 404)
{
    // Container not found
    Console.WriteLine("Container does not exist");
}
catch (RequestFailedException ex) when (ex.Status == 409)
{
    // Blob already exists (and overwrite: false)
    Console.WriteLine("Blob already exists");
}
catch (RequestFailedException ex) when (ex.Status == 412)
{
    // Precondition failed (ETag mismatch)
    Console.WriteLine("Concurrent modification detected");
}
catch (RequestFailedException ex) when (ex.Status == 429)
{
    // Throttling
    Console.WriteLine($"Throttled, retry after: {ex.Headers["Retry-After"]}");
}
catch (RequestFailedException ex) when (ex.Status >= 500)
{
    // Server error (retryable)
    Console.WriteLine($"Server error: {ex.Message}");
}
```

See `06-operations/data-access-layer.md` for centralized error handling patterns.

## Performance Checklist

- [ ] Use streaming APIs for files >10MB
- [ ] Configure parallel transfers (4-8 concurrent for most workloads)
- [ ] Set appropriate chunk sizes (8-16MB)
- [ ] Set Content-Type for browser-served blobs
- [ ] Use progress handlers for long-running uploads
- [ ] Implement retry logic (SDK handles by default)
- [ ] Use conditional operations (If-Match) for concurrency control
- [ ] Query blobs by tags instead of listing + filtering

## Navigation

- **Previous**: `blob-fundamentals.md`
- **Next**: `blob-concurrency.md`
- **Up**: `00-overview.md`

## See Also

- `03-blob-storage/blob-concurrency.md` - ETag patterns
- `06-operations/performance-optimization.md` - Throughput tuning
- `08-reference/code-examples.md` - Complete examples
