# Azure Storage Performance Optimization

> **File Purpose**: Comprehensive guide to throughput targets, parallelism strategies, CDN integration, chunk sizing, and performance benchmarking
> **Prerequisites**: `../03-blob-storage/blob-fundamentals.md`, `observability.md`
> **Agent Use Case**: Reference when optimizing storage performance, troubleshooting slow operations, or designing high-throughput applications

## Quick Context

Azure Storage performance depends on partition design, parallelism, chunk sizing, and network configuration. Understanding scalability targets and optimization techniques is critical for production workloads.

**Key principle**: Design for parallelism; leverage CDN for read-heavy workloads; tune chunk sizes for your data; monitor and adjust based on metrics.

## Performance Targets and Limits

### Storage Account Limits

| Metric | Standard (LRS/ZRS) | Standard (GRS/GZRS) | Premium (Block Blob) |
|--------|-------------------|---------------------|---------------------|
| **Max ingress** | 50 Gbps | 25 Gbps | 100 Gbps |
| **Max egress** | 100 Gbps | 50 Gbps | 200 Gbps |
| **Max IOPS** | 20,000 req/s | 20,000 req/s | 100,000+ req/s |
| **Total capacity** | 5 PiB | 5 PiB | 4.75 TiB per blob |

### Blob Storage Limits

| Blob Type | Max Size | Max Throughput | Max Request Rate | Optimal Chunk |
|-----------|----------|----------------|------------------|---------------|
| **Block Blob** | 190.7 TiB | 60 GiB/s per blob | 2,000 req/s | 8-16 MB |
| **Append Blob** | 195 GiB | 60 MiB/s | 100 appends/s | 4 MB |
| **Page Blob** | 8 TiB | 60 MiB/s | 500 IOPS (std)<br/>20,000 IOPS (premium) | 512 bytes |

### Table Storage Limits

| Metric | Limit | Notes |
|--------|-------|-------|
| **Partition throughput** | 2,000 entities/s | Per partition |
| **Table throughput** | 20,000 entities/s | Across all partitions |
| **Entity size** | 1 MB | Including all properties |
| **Batch size** | 100 entities, 4 MB | Same partition key only |

### Queue Storage Limits

| Metric | Limit | Notes |
|--------|-------|-------|
| **Message size** | 64 KB | Per message |
| **Queue throughput** | 2,000 msg/s | Per queue |
| **Batch size** | 32 messages | GetMessages operation |
| **Visibility timeout** | 7 days | Max message lock duration |

## Blob Storage Performance Optimization

### 1. Parallel Upload Strategies

#### Small Files (<4 MB)

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public class BlobUploader
{
    private readonly BlobContainerClient _containerClient;

    // Upload small file (single PUT)
    public async Task UploadSmallFileAsync(string blobName, Stream content)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);

        // Single operation, no chunking needed
        var uploadOptions = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = "application/pdf"
            },
            TransferOptions = new StorageTransferOptions
            {
                MaximumTransferSize = 4 * 1024 * 1024  // 4 MB (single upload)
            }
        };

        await blobClient.UploadAsync(content, uploadOptions);
        // ~50ms for 4 MB file
    }
}
```

**Performance**: ~50-100ms for <4 MB files, single HTTP request

#### Medium Files (4-256 MB)

```csharp
// Optimal chunking for medium files
public async Task UploadMediumFileAsync(string blobName, Stream content)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    var uploadOptions = new BlobUploadOptions
    {
        TransferOptions = new StorageTransferOptions
        {
            // 8 MB chunks, 4 concurrent uploads
            InitialTransferSize = 8 * 1024 * 1024,
            MaximumTransferSize = 8 * 1024 * 1024,
            MaximumConcurrency = 4
        },
        ProgressHandler = new Progress<long>(bytes =>
        {
            Console.WriteLine($"Uploaded: {bytes / (1024.0 * 1024.0):F2} MB");
        })
    };

    await blobClient.UploadAsync(content, uploadOptions);
}
```

**Performance**: ~500 Mbps throughput for 100 MB file (4 concurrent streams)

#### Large Files (256 MB - 1 GB)

```csharp
// Optimized for large files
public async Task UploadLargeFileAsync(string blobName, Stream content)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    var uploadOptions = new BlobUploadOptions
    {
        TransferOptions = new StorageTransferOptions
        {
            // 16 MB chunks, 8 concurrent uploads
            InitialTransferSize = 16 * 1024 * 1024,
            MaximumTransferSize = 16 * 1024 * 1024,
            MaximumConcurrency = 8
        },
        HttpHeaders = new BlobHttpHeaders
        {
            ContentType = "video/mp4"
        }
    };

    await blobClient.UploadAsync(content, uploadOptions);
}
```

**Performance**: ~1-2 Gbps throughput for 1 GB file (8 concurrent streams)

#### Huge Files (>1 GB)

```csharp
// Maximum throughput for huge files
public async Task UploadHugeFileAsync(string blobName, string filePath)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    var uploadOptions = new BlobUploadOptions
    {
        TransferOptions = new StorageTransferOptions
        {
            // 100 MB chunks, 16 concurrent uploads
            InitialTransferSize = 100 * 1024 * 1024,
            MaximumTransferSize = 100 * 1024 * 1024,
            MaximumConcurrency = 16
        },
        ProgressHandler = new Progress<long>(bytes =>
        {
            var mbUploaded = bytes / (1024.0 * 1024.0);
            var progress = (double)bytes / new FileInfo(filePath).Length * 100;
            Console.WriteLine($"Uploaded: {mbUploaded:F2} MB ({progress:F1}%)");
        })
    };

    using var fileStream = File.OpenRead(filePath);
    await blobClient.UploadAsync(fileStream, uploadOptions);
}
```

**Performance**: ~5-10 Gbps throughput for 10 GB file (16 concurrent streams)

### 2. Parallel Download Strategies

#### Sequential Download (Small Files)

```csharp
// Simple download for small files
public async Task<byte[]> DownloadSmallFileAsync(string blobName)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    var downloadResponse = await blobClient.DownloadContentAsync();
    return downloadResponse.Value.Content.ToArray();
}
```

#### Parallel Download (Large Files)

```csharp
public async Task DownloadLargeFileParallelAsync(string blobName, string localPath)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    // Get blob properties to determine size
    var properties = await blobClient.GetPropertiesAsync();
    var blobSize = properties.Value.ContentLength;

    // Download options with parallelism
    var downloadOptions = new BlobDownloadToOptions
    {
        TransferOptions = new StorageTransferOptions
        {
            InitialTransferSize = 16 * 1024 * 1024,  // 16 MB chunks
            MaximumTransferSize = 16 * 1024 * 1024,
            MaximumConcurrency = 8  // 8 parallel downloads
        },
        ProgressHandler = new Progress<long>(bytes =>
        {
            var progress = (double)bytes / blobSize * 100;
            Console.WriteLine($"Downloaded: {bytes / (1024.0 * 1024.0):F2} MB ({progress:F1}%)");
        })
    };

    await blobClient.DownloadToAsync(localPath, downloadOptions);
}
```

**Performance**: ~2-5 Gbps throughput with 8 parallel streams

#### Range Download (Partial Content)

```csharp
// Download specific byte range (efficient for large files)
public async Task<byte[]> DownloadRangeAsync(
    string blobName,
    long offset,
    long length)
{
    var blobClient = _containerClient.GetBlobClient(blobName);

    var downloadOptions = new BlobDownloadOptions
    {
        Range = new Azure.HttpRange(offset, length)
    };

    var downloadResponse = await blobClient.DownloadContentAsync(downloadOptions);
    return downloadResponse.Value.Content.ToArray();
}

// Example: Download first 10 MB of video for preview
var preview = await DownloadRangeAsync("movie.mp4", offset: 0, length: 10 * 1024 * 1024);
```

### 3. Chunk Size Tuning

#### Performance Benchmarks

| Chunk Size | Network Overhead | Memory Usage | Optimal For |
|------------|-----------------|--------------|-------------|
| **1 MB** | High (many requests) | Low | Very slow networks |
| **4 MB** | Moderate | Low | Standard networks |
| **8 MB** | Low | Moderate | Fast networks (100 Mbps) |
| **16 MB** | Low | Moderate | Very fast networks (1 Gbps) |
| **32-100 MB** | Very low | High | Ultra-fast networks (10 Gbps) |

#### Dynamic Chunk Sizing

```csharp
public class AdaptiveBlobUploader
{
    private readonly BlobContainerClient _containerClient;

    private (int chunkSize, int concurrency) GetOptimalSettings(long fileSize)
    {
        return fileSize switch
        {
            < 4 * 1024 * 1024 => (4 * 1024 * 1024, 1),      // <4 MB: single upload
            < 256 * 1024 * 1024 => (8 * 1024 * 1024, 4),    // <256 MB: 8 MB chunks, 4 concurrent
            < 1024 * 1024 * 1024 => (16 * 1024 * 1024, 8),  // <1 GB: 16 MB chunks, 8 concurrent
            _ => (100 * 1024 * 1024, 16)                     // >1 GB: 100 MB chunks, 16 concurrent
        };
    }

    public async Task UploadWithOptimalSettingsAsync(string blobName, string filePath)
    {
        var fileInfo = new FileInfo(filePath);
        var (chunkSize, concurrency) = GetOptimalSettings(fileInfo.Length);

        Console.WriteLine($"Uploading {fileInfo.Length / (1024.0 * 1024.0):F2} MB with {concurrency} concurrent streams, {chunkSize / (1024 * 1024)} MB chunks");

        var blobClient = _containerClient.GetBlobClient(blobName);

        var uploadOptions = new BlobUploadOptions
        {
            TransferOptions = new StorageTransferOptions
            {
                InitialTransferSize = chunkSize,
                MaximumTransferSize = chunkSize,
                MaximumConcurrency = concurrency
            }
        };

        using var fileStream = File.OpenRead(filePath);
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        await blobClient.UploadAsync(fileStream, uploadOptions);
        stopwatch.Stop();

        var throughputMbps = (fileInfo.Length / (1024.0 * 1024.0)) / stopwatch.Elapsed.TotalSeconds;
        Console.WriteLine($"Upload complete: {throughputMbps:F2} MB/s");
    }
}
```

### 4. Batch Operations

```csharp
// Batch upload multiple small files
public async Task BatchUploadAsync(string[] filePaths)
{
    var uploadTasks = filePaths.Select(async filePath =>
    {
        var blobName = Path.GetFileName(filePath);
        var blobClient = _containerClient.GetBlobClient(blobName);

        using var fileStream = File.OpenRead(filePath);
        await blobClient.UploadAsync(fileStream, overwrite: true);

        return blobName;
    });

    var results = await Task.WhenAll(uploadTasks);
    Console.WriteLine($"Uploaded {results.Length} files in parallel");
}
```

**Performance**: 10x faster than sequential uploads for 100 small files

## Table Storage Performance Optimization

### 1. Partition Key Design

#### ❌ Bad: Single Partition (Hot Partition)

```csharp
// All entities in single partition (max 2,000 ops/s)
public class BadOrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "all-orders";  // ❌ Bottleneck!
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }
}
```

#### ✅ Good: Distributed Partitions

```csharp
// Hash-based partitioning (scales to 20,000+ ops/s)
public class GoodOrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";  // customer-{hash}
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public static string GetPartitionKey(string customerId)
    {
        var hash = Math.Abs(customerId.GetHashCode()) % 100;
        return $"customer-{hash:D2}";  // 100 partitions
    }
}

// Usage
var order = new GoodOrderEntity
{
    PartitionKey = GoodOrderEntity.GetPartitionKey(customerId),
    RowKey = orderId
};

await tableClient.AddEntityAsync(order);
```

**Performance improvement**: 100x throughput (2,000 ops/s → 200,000 ops/s)

### 2. Batch Operations

```csharp
public async Task BatchInsertAsync(List<OrderEntity> orders)
{
    // Group by partition key (batch requirement)
    var grouped = orders.GroupBy(o => o.PartitionKey);

    foreach (var group in grouped)
    {
        var batch = new List<TableTransactionAction>();

        foreach (var order in group.Take(100))  // Max 100 per batch
        {
            batch.Add(new TableTransactionAction(TableTransactionActionType.Add, order));
        }

        // Single transaction (atomic)
        await tableClient.SubmitTransactionAsync(batch);
    }
}
```

**Performance**: ~20x faster than individual inserts (1 request vs 100 requests)

### 3. Query Optimization

#### Point Query (Fastest)

```csharp
// ~5ms latency
var entity = await tableClient.GetEntityAsync<OrderEntity>(
    partitionKey: "customer-123",
    rowKey: "order-456"
);
```

#### Partition Query (Fast)

```csharp
// ~10-50ms latency depending on partition size
var query = tableClient.QueryAsync<OrderEntity>(
    filter: $"PartitionKey eq 'customer-123'"
);
```

#### Table Scan (Slow - Avoid!)

```csharp
// ❌ Avoid: Scans entire table (seconds to minutes)
var query = tableClient.QueryAsync<OrderEntity>(
    filter: $"Status eq 'pending'"
);

// ✅ Better: Use secondary index table
var indexTable = tableServiceClient.GetTableClient("OrdersByStatus");
var query = indexTable.QueryAsync<OrderEntity>(
    filter: $"PartitionKey eq 'pending'"
);
```

### 4. Parallel Queries

```csharp
public async Task<List<OrderEntity>> ParallelPartitionQueryAsync(string[] customerIds)
{
    var queryTasks = customerIds.Select(async customerId =>
    {
        var partitionKey = OrderEntity.GetPartitionKey(customerId);
        var query = tableClient.QueryAsync<OrderEntity>(
            filter: $"PartitionKey eq '{partitionKey}'"
        );

        var results = new List<OrderEntity>();
        await foreach (var entity in query)
        {
            results.Add(entity);
        }

        return results;
    });

    var allResults = await Task.WhenAll(queryTasks);
    return allResults.SelectMany(r => r).ToList();
}
```

## CDN Integration (Blob Storage)

### Why Use CDN?

**Benefits**:
- **Reduced latency**: Content served from edge locations (50-200ms → 10-50ms)
- **Lower storage egress costs**: Cached content reduces origin requests
- **Higher availability**: CDN provides additional redundancy
- **DDoS protection**: CDN absorbs attack traffic

### Setup Azure CDN

#### Azure CLI

```bash
# Create CDN profile
az cdn profile create \
  --name mycdnprofile \
  --resource-group myresourcegroup \
  --sku Standard_Microsoft \
  --location global

# Create CDN endpoint
az cdn endpoint create \
  --name mystoragecdn \
  --profile-name mycdnprofile \
  --resource-group myresourcegroup \
  --origin mystorageaccount.blob.core.windows.net \
  --origin-host-header mystorageaccount.blob.core.windows.net \
  --enable-compression true \
  --content-types-to-compress "text/html" "text/css" "application/javascript" "application/json"

# Configure caching rules
az cdn endpoint rule add \
  --name mystoragecdn \
  --profile-name mycdnprofile \
  --resource-group myresourcegroup \
  --order 1 \
  --rule-name CacheStaticAssets \
  --match-variable UrlPath \
  --operator Contains \
  --match-values "/images/" "/css/" "/js/" \
  --action-name CacheExpiration \
  --cache-behavior Override \
  --cache-duration "7.00:00:00"  # 7 days
```

#### Bicep

```bicep
resource cdnProfile 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: 'mycdnprofile'
  location: 'global'
  sku: {
    name: 'Standard_Microsoft'
  }
}

resource cdnEndpoint 'Microsoft.Cdn/profiles/endpoints@2023-05-01' = {
  parent: cdnProfile
  name: 'mystoragecdn'
  location: 'global'
  properties: {
    originHostHeader: '${storageAccountName}.blob.core.windows.net'
    isHttpAllowed: true
    isHttpsAllowed: true
    queryStringCachingBehavior: 'IgnoreQueryString'
    contentTypesToCompress: [
      'text/plain'
      'text/html'
      'text/css'
      'application/javascript'
      'application/json'
      'image/svg+xml'
    ]
    isCompressionEnabled: true
    origins: [
      {
        name: 'storage-origin'
        properties: {
          hostName: '${storageAccountName}.blob.core.windows.net'
          httpPort: 80
          httpsPort: 443
        }
      }
    ]
    deliveryPolicy: {
      rules: [
        {
          name: 'CacheStaticAssets'
          order: 1
          conditions: [
            {
              name: 'UrlPath'
              parameters: {
                operator: 'Contains'
                matchValues: [
                  '/images/'
                  '/css/'
                  '/js/'
                ]
                '@odata.type': '#Microsoft.Azure.Cdn.Models.DeliveryRuleUrlPathConditionParameters'
              }
            }
          ]
          actions: [
            {
              name: 'CacheExpiration'
              parameters: {
                cacheBehavior: 'Override'
                cacheDuration: '7.00:00:00'
                '@odata.type': '#Microsoft.Azure.Cdn.Models.DeliveryRuleCacheExpirationActionParameters'
              }
            }
          ]
        }
      ]
    }
  }
}
```

### C# CDN-Aware Access

```csharp
public class CdnBlobService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly string _cdnEndpoint;

    public CdnBlobService(
        BlobServiceClient blobServiceClient,
        string cdnEndpoint)
    {
        _blobServiceClient = blobServiceClient;
        _cdnEndpoint = cdnEndpoint;
    }

    // Upload with cache-friendly headers
    public async Task UploadWithCachingAsync(
        string containerName,
        string blobName,
        Stream content,
        TimeSpan cacheMaxAge)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var uploadOptions = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                CacheControl = $"public, max-age={cacheMaxAge.TotalSeconds}",
                ContentType = GetContentType(blobName)
            }
        };

        await blobClient.UploadAsync(content, uploadOptions);
    }

    // Get CDN URL for blob
    public string GetCdnUrl(string containerName, string blobName)
    {
        return $"https://{_cdnEndpoint}/{containerName}/{blobName}";
    }

    // Purge CDN cache when blob updated
    public async Task PurgeCdnCacheAsync(string containerName, string blobName)
    {
        var cdnUrl = GetCdnUrl(containerName, blobName);

        // Azure CLI command (execute via Process.Start)
        var purgeCommand = $"az cdn endpoint purge " +
            $"--content-paths \"/{containerName}/{blobName}\" " +
            $"--name mystoragecdn " +
            $"--profile-name mycdnprofile " +
            $"--resource-group myresourcegroup";

        Console.WriteLine($"Purging CDN cache for: {cdnUrl}");
        // Execute purge...
    }

    private string GetContentType(string blobName)
    {
        var extension = Path.GetExtension(blobName).ToLower();
        return extension switch
        {
            ".jpg" or ".jpeg" => "image/jpeg",
            ".png" => "image/png",
            ".css" => "text/css",
            ".js" => "application/javascript",
            ".json" => "application/json",
            ".html" => "text/html",
            _ => "application/octet-stream"
        };
    }
}

// Usage
var cdnBlobService = new CdnBlobService(
    blobServiceClient,
    cdnEndpoint: "mystoragecdn.azureedge.net"
);

// Upload image with 7-day cache
using var imageStream = File.OpenRead("logo.png");
await cdnBlobService.UploadWithCachingAsync(
    "images",
    "logo.png",
    imageStream,
    cacheMaxAge: TimeSpan.FromDays(7)
);

// Get CDN URL
var cdnUrl = cdnBlobService.GetCdnUrl("images", "logo.png");
Console.WriteLine($"Access via CDN: {cdnUrl}");
// https://mystoragecdn.azureedge.net/images/logo.png
```

### CDN Performance Metrics

```kql
// CDN cache hit ratio
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.CDN"
| where OperationName == "Microsoft.Cdn/Profiles/Endpoints/contentDelivery"
| summarize
    TotalRequests = count(),
    CacheHits = countif(CacheStatus == "HIT"),
    CacheMisses = countif(CacheStatus == "MISS")
| extend CacheHitRatio = (CacheHits * 100.0) / TotalRequests
| project TotalRequests, CacheHits, CacheMisses, CacheHitRatio
```

**Target**: >80% cache hit ratio for static content

## Performance Benchmarking

### C# Benchmark Harness

```csharp
using System.Diagnostics;

public class StoragePerformanceBenchmark
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly TableServiceClient _tableServiceClient;

    public async Task RunBlobUploadBenchmarkAsync(int fileSizeMB, int concurrency)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient("benchmark");
        await containerClient.CreateIfNotExistsAsync();

        var fileSize = fileSizeMB * 1024 * 1024;
        var testData = new byte[fileSize];
        new Random().NextBytes(testData);

        var uploadOptions = new BlobUploadOptions
        {
            TransferOptions = new StorageTransferOptions
            {
                MaximumConcurrency = concurrency
            }
        };

        var stopwatch = Stopwatch.StartNew();

        using var stream = new MemoryStream(testData);
        var blobClient = containerClient.GetBlobClient($"benchmark-{Guid.NewGuid()}.bin");
        await blobClient.UploadAsync(stream, uploadOptions);

        stopwatch.Stop();

        var throughputMBps = (fileSize / (1024.0 * 1024.0)) / stopwatch.Elapsed.TotalSeconds;
        var throughputGbps = throughputMBps * 8 / 1024.0;

        Console.WriteLine($"File: {fileSizeMB} MB, Concurrency: {concurrency}");
        Console.WriteLine($"Time: {stopwatch.ElapsedMilliseconds}ms");
        Console.WriteLine($"Throughput: {throughputMBps:F2} MB/s ({throughputGbps:F2} Gbps)");
    }

    public async Task RunTableInsertBenchmarkAsync(int entityCount, int batchSize)
    {
        var tableClient = _tableServiceClient.GetTableClient("benchmark");
        await tableClient.CreateIfNotExistsAsync();

        var stopwatch = Stopwatch.StartNew();

        for (int i = 0; i < entityCount; i += batchSize)
        {
            var batch = new List<TableTransactionAction>();

            for (int j = 0; j < Math.Min(batchSize, entityCount - i); j++)
            {
                var entity = new TableEntity("partition-1", $"entity-{i + j}")
                {
                    ["Data"] = $"Benchmark data {i + j}"
                };

                batch.Add(new TableTransactionAction(TableTransactionActionType.Add, entity));
            }

            await tableClient.SubmitTransactionAsync(batch);
        }

        stopwatch.Stop();

        var entitiesPerSecond = entityCount / stopwatch.Elapsed.TotalSeconds;

        Console.WriteLine($"Entities: {entityCount}, Batch size: {batchSize}");
        Console.WriteLine($"Time: {stopwatch.ElapsedMilliseconds}ms");
        Console.WriteLine($"Throughput: {entitiesPerSecond:F0} entities/s");
    }
}

// Run benchmarks
var benchmark = new StoragePerformanceBenchmark(blobServiceClient, tableServiceClient);

// Blob upload benchmarks
await benchmark.RunBlobUploadBenchmarkAsync(fileSizeMB: 100, concurrency: 1);
await benchmark.RunBlobUploadBenchmarkAsync(fileSizeMB: 100, concurrency: 4);
await benchmark.RunBlobUploadBenchmarkAsync(fileSizeMB: 100, concurrency: 8);
await benchmark.RunBlobUploadBenchmarkAsync(fileSizeMB: 100, concurrency: 16);

// Table insert benchmarks
await benchmark.RunTableInsertBenchmarkAsync(entityCount: 1000, batchSize: 1);
await benchmark.RunTableInsertBenchmarkAsync(entityCount: 1000, batchSize: 100);
```

### Sample Benchmark Results

```
Blob Upload Benchmarks (100 MB file):
------------------------------------
Concurrency 1:  200 MB/s (1.6 Gbps)
Concurrency 4:  600 MB/s (4.8 Gbps)
Concurrency 8:  900 MB/s (7.2 Gbps)
Concurrency 16: 1200 MB/s (9.6 Gbps)

Table Insert Benchmarks (1,000 entities):
---------------------------------------
Batch size 1:   100 entities/s
Batch size 100: 2000 entities/s
```

## Best Practices Summary

### 1. Blob Storage

- ✅ Use parallel uploads/downloads (4-16 concurrent streams)
- ✅ Tune chunk sizes (8-100 MB depending on file size)
- ✅ Enable CDN for static content (images, CSS, JS)
- ✅ Use Premium Blob Storage for high IOPS workloads
- ✅ Monitor throttling (429 errors) and adjust parallelism

### 2. Table Storage

- ✅ Design partition keys to distribute load evenly
- ✅ Use batch operations (up to 100 entities per batch)
- ✅ Avoid table scans (use partition queries or secondary indexes)
- ✅ Keep entity size <100 KB for optimal performance
- ✅ Use point queries (PK + RK) whenever possible

### 3. General

- ✅ Monitor metrics (latency, throughput, throttling)
- ✅ Test different concurrency levels for your workload
- ✅ Use Azure regions close to users
- ✅ Enable compression for text-based content
- ✅ Implement retry policies with exponential backoff

## References

**Microsoft Learn Documentation**:
- [Blob Storage Performance](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-scalability-targets)
- [Table Storage Performance](https://learn.microsoft.com/azure/storage/tables/scalability-targets)
- [Azure CDN](https://learn.microsoft.com/azure/cdn/cdn-overview)
- [Performance Tuning](https://learn.microsoft.com/azure/storage/blobs/storage-performance-checklist)
- [Parallelism Best Practices](https://learn.microsoft.com/azure/storage/blobs/storage-blob-scalable-app-upload-files)

## Navigation

- **Previous**: `observability.md`
- **Next**: `data-access-layer.md`
- **Related**: `../03-blob-storage/blob-fundamentals.md` - Performance characteristics

## See Also

- `observability.md` - Monitor performance metrics
- `data-access-layer.md` - Client-side optimization
- `governance.md` - Cost optimization through performance tuning
