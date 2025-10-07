# Blob Storage Concurrency Control

> **File Purpose**: Comprehensive guide to ETags, conditional operations, optimistic concurrency, and lease management for blob storage
> **Prerequisites**: `blob-fundamentals.md`, `02-core-concepts/storage-accounts.md`
> **Agent Use Case**: Reference when implementing concurrent access patterns, preventing lost updates, and coordinating blob operations

## Quick Context

Azure Blob Storage provides multiple concurrency control mechanisms to prevent lost updates and coordinate access across distributed systems. ETags enable optimistic concurrency, while leases provide exclusive write access for coordinated operations.

**Key principle**: Use ETags for optimistic concurrency (most common), use leases for exclusive access (distributed coordination), never assume single-writer scenarios in production.

## Concurrency Challenges

### The Lost Update Problem

```
Time    Client A              Blob Storage        Client B
----    --------              ------------        --------
T0      Read blob (v1)                            Read blob (v1)
T1      Modify locally                            Modify locally
T2      Write blob (v2) ✓     [v2 stored]
T3                                                Write blob (v3) ✓
                                                  ⚠️ Client A's changes lost!
```

**Without concurrency control**: Client B overwrites Client A's changes, causing data loss.

**With optimistic concurrency**: Client B's write fails with 412 Precondition Failed, preserving Client A's changes.

## ETags and Conditional Operations

### What are ETags?

**ETag** (Entity Tag) is a unique identifier that changes whenever a blob is modified. Azure generates ETags automatically for all blobs.

**Format**: `"0x8DCFD12345ABCDEF"` (opaque string, don't parse)

**Properties**:
- Changes on every write operation
- Unique per blob version
- Can be used for conditional operations
- Returned in response headers and blob properties

### How ETags Work

```
┌─────────────────────────────────────────────────────────┐
│  Blob Lifecycle                                         │
├─────────────────────────────────────────────────────────┤
│  Upload blob                                            │
│  └─> ETag: "0x8D1"                                      │
│                                                          │
│  Modify blob                                            │
│  └─> ETag: "0x8D2" (changed)                            │
│                                                          │
│  Modify again                                           │
│  └─> ETag: "0x8D3" (changed again)                      │
└─────────────────────────────────────────────────────────┘
```

### C# Implementation: Basic ETag Usage

#### Get Current ETag

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure;

var blobClient = containerClient.GetBlobClient("important-document.json");

// Get current ETag
var properties = await blobClient.GetPropertiesAsync();
var currentETag = properties.Value.ETag;

Console.WriteLine($"Current ETag: {currentETag}");
// Output: Current ETag: "0x8DCFD12345ABCDEF"
```

#### Upload with ETag Return

```csharp
// Upload returns ETag in response
var uploadResponse = await blobClient.UploadAsync(
    new BinaryData("{\"version\": 1}"),
    overwrite: true
);

var eTag = uploadResponse.Value.ETag;
Console.WriteLine($"Upload ETag: {eTag}");
```

## Optimistic Concurrency with If-Match

### If-Match Header

**Purpose**: "Proceed only if the ETag matches the current blob version"

**Use case**: Prevent lost updates when multiple clients may modify the same blob

### Pattern: Read-Modify-Write with If-Match

```csharp
public async Task<bool> UpdateDocumentSafelyAsync(
    BlobClient blobClient,
    Func<string, string> updateFunction)
{
    try
    {
        // Step 1: Read current content and ETag
        var downloadResponse = await blobClient.DownloadContentAsync();
        var currentContent = downloadResponse.Value.Content.ToString();
        var currentETag = downloadResponse.Value.Details.ETag;

        Console.WriteLine($"Read ETag: {currentETag}");

        // Step 2: Apply business logic
        var updatedContent = updateFunction(currentContent);

        // Step 3: Write only if ETag hasn't changed
        var uploadOptions = new BlobUploadOptions
        {
            Conditions = new BlobRequestConditions
            {
                IfMatch = currentETag  // ⚠️ Fail if ETag changed
            }
        };

        await blobClient.UploadAsync(
            new BinaryData(updatedContent),
            uploadOptions
        );

        Console.WriteLine("Update succeeded - no concurrent modifications");
        return true;
    }
    catch (RequestFailedException ex) when (ex.Status == 412)
    {
        Console.WriteLine("Update failed - blob was modified by another client");
        return false;
    }
}

// Usage
var blobClient = containerClient.GetBlobClient("counter.json");

bool success = await UpdateDocumentSafelyAsync(
    blobClient,
    content =>
    {
        var doc = JsonDocument.Parse(content);
        var counter = doc.RootElement.GetProperty("counter").GetInt32();
        return $"{{\"counter\": {counter + 1}}}";
    }
);

if (!success)
{
    Console.WriteLine("Retry with fresh ETag");
}
```

**Why it works**: If another client modifies the blob between read and write, the ETag changes, and the upload fails with 412 (Precondition Failed).

### Retry Pattern with Exponential Backoff

```csharp
public async Task<T> OptimisticUpdateWithRetryAsync<T>(
    BlobClient blobClient,
    Func<T, T> updateFunction,
    int maxRetries = 5)
    where T : class
{
    int retryCount = 0;
    var random = new Random();

    while (retryCount < maxRetries)
    {
        try
        {
            // Read current state
            var downloadResponse = await blobClient.DownloadContentAsync();
            var currentData = downloadResponse.Value.Content.ToObjectFromJson<T>();
            var currentETag = downloadResponse.Value.Details.ETag;

            // Apply update
            var updatedData = updateFunction(currentData);

            // Write with ETag check
            var uploadOptions = new BlobUploadOptions
            {
                Conditions = new BlobRequestConditions { IfMatch = currentETag }
            };

            await blobClient.UploadAsync(
                BinaryData.FromObjectAsJson(updatedData),
                uploadOptions
            );

            Console.WriteLine($"Update succeeded after {retryCount} retries");
            return updatedData;
        }
        catch (RequestFailedException ex) when (ex.Status == 412)
        {
            retryCount++;
            if (retryCount >= maxRetries)
            {
                throw new InvalidOperationException(
                    $"Failed to update after {maxRetries} retries due to concurrent modifications",
                    ex
                );
            }

            // Exponential backoff with jitter
            var backoffMs = (int)(Math.Pow(2, retryCount) * 100 + random.Next(0, 100));
            Console.WriteLine($"Retry {retryCount}/{maxRetries} after {backoffMs}ms");
            await Task.Delay(backoffMs);
        }
    }

    throw new InvalidOperationException("Update failed unexpectedly");
}

// Usage: Increment counter with automatic retry
public class Counter
{
    public int Value { get; set; }
    public DateTime LastUpdated { get; set; }
}

var blobClient = containerClient.GetBlobClient("distributed-counter.json");

var updatedCounter = await OptimisticUpdateWithRetryAsync(
    blobClient,
    (Counter counter) =>
    {
        counter.Value++;
        counter.LastUpdated = DateTime.UtcNow;
        return counter;
    },
    maxRetries: 10
);

Console.WriteLine($"New counter value: {updatedCounter.Value}");
```

**Performance characteristics**:
- Average retries under moderate contention: 1-2
- Average retries under high contention: 3-5
- Exponential backoff prevents thundering herd

## If-None-Match for Conditional Downloads

### If-None-Match Header

**Purpose**: "Proceed only if the ETag does NOT match the current blob version"

**Use case**: Efficient polling, caching, avoiding redundant downloads

### Pattern: Conditional Download (Cache Validation)

```csharp
public class BlobCache
{
    private readonly Dictionary<string, (ETag ETag, string Content)> _cache = new();

    public async Task<string> GetContentAsync(BlobClient blobClient)
    {
        var blobName = blobClient.Name;

        // Check if we have cached version
        if (_cache.TryGetValue(blobName, out var cached))
        {
            try
            {
                // Download only if blob changed (ETag doesn't match)
                var downloadOptions = new BlobDownloadOptions
                {
                    Conditions = new BlobRequestConditions
                    {
                        IfNoneMatch = cached.ETag  // 304 if not modified
                    }
                };

                var response = await blobClient.DownloadContentAsync(downloadOptions);

                // Blob changed, update cache
                var newContent = response.Value.Content.ToString();
                var newETag = response.Value.Details.ETag;
                _cache[blobName] = (newETag, newContent);

                Console.WriteLine("Cache miss - blob was modified");
                return newContent;
            }
            catch (RequestFailedException ex) when (ex.Status == 304)
            {
                // Not modified, use cached version
                Console.WriteLine("Cache hit - blob unchanged");
                return cached.Content;
            }
        }

        // First download, populate cache
        var downloadResponse = await blobClient.DownloadContentAsync();
        var content = downloadResponse.Value.Content.ToString();
        var eTag = downloadResponse.Value.Details.ETag;
        _cache[blobName] = (eTag, content);

        Console.WriteLine("Initial download");
        return content;
    }
}

// Usage: Efficient polling
var cache = new BlobCache();
var configBlob = containerClient.GetBlobClient("app-config.json");

while (true)
{
    var config = await cache.GetContentAsync(configBlob);
    // Only downloads if config changed
    await Task.Delay(TimeSpan.FromSeconds(30));
}
```

**Benefits**:
- Reduces bandwidth (no download if blob unchanged)
- Reduces storage costs (no egress for 304 responses)
- Faster response (304 returns immediately)

## Conditional Headers Reference

### All Conditional Headers

| Header | Condition | Success | Failure | Use Case |
|--------|-----------|---------|---------|----------|
| **IfMatch** | ETag matches current | 200 OK | 412 Precondition Failed | Optimistic concurrency |
| **IfNoneMatch** | ETag doesn't match | 200 OK | 304 Not Modified | Caching, avoid redundant downloads |
| **IfModifiedSince** | Modified after date | 200 OK | 304 Not Modified | Time-based caching |
| **IfUnmodifiedSince** | Not modified since date | 200 OK | 412 Precondition Failed | Ensure staleness threshold |

### C# Examples for All Headers

```csharp
// IfMatch - Update only if not changed
var uploadOptions = new BlobUploadOptions
{
    Conditions = new BlobRequestConditions
    {
        IfMatch = currentETag
    }
};

// IfNoneMatch - Download only if changed
var downloadOptions = new BlobDownloadOptions
{
    Conditions = new BlobRequestConditions
    {
        IfNoneMatch = cachedETag
    }
};

// IfModifiedSince - Download only if modified after date
var downloadOptions2 = new BlobDownloadOptions
{
    Conditions = new BlobRequestConditions
    {
        IfModifiedSince = lastCheckTime
    }
};

// IfUnmodifiedSince - Delete only if not modified after date
var deleteOptions = new BlobRequestConditions
{
    IfUnmodifiedSince = maxAllowedAge
};
await blobClient.DeleteIfExistsAsync(conditions: deleteOptions);

// Wildcard - Create only if doesn't exist
var uploadOptions2 = new BlobUploadOptions
{
    Conditions = new BlobRequestConditions
    {
        IfNoneMatch = ETag.All  // Fails if blob exists
    }
};
```

## Blob Leases

### What are Leases?

**Lease**: An exclusive lock on a blob that prevents other clients from modifying or deleting it.

**Key characteristics**:
- Duration: 15-60 seconds (finite) or infinite
- Renewable before expiration
- Breakable by authorized clients
- Prevents writes and deletes (reads are still allowed)

### Lease States

```
┌─────────────────────────────────────────────────────┐
│  Lease Lifecycle                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  [Available]                                        │
│      │                                              │
│      │ Acquire(60s)                                 │
│      ▼                                              │
│  [Leased] ──Renew──> [Leased]                      │
│      │                                              │
│      │ Expires after 60s (if not renewed)          │
│      │ OR Release()                                 │
│      │ OR Break()                                   │
│      ▼                                              │
│  [Available]                                        │
│                                                      │
│  [Broken] (intermediate state during break)         │
└─────────────────────────────────────────────────────┘
```

### C# Implementation: Basic Lease Operations

#### Acquire Lease

```csharp
using Azure.Storage.Blobs.Specialized;

var blobClient = containerClient.GetBlobClient("shared-resource.txt");
var leaseClient = blobClient.GetBlobLeaseClient();

// Acquire 60-second lease
var leaseResponse = await leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));
var leaseId = leaseResponse.Value.LeaseId;

Console.WriteLine($"Acquired lease: {leaseId}");

try
{
    // Perform exclusive operations
    var uploadOptions = new BlobUploadOptions
    {
        Conditions = new BlobRequestConditions
        {
            LeaseId = leaseId  // Required for write operations
        }
    };

    await blobClient.UploadAsync(
        new BinaryData("Updated content"),
        uploadOptions
    );

    Console.WriteLine("Successfully modified blob with lease");
}
finally
{
    // Always release lease
    await leaseClient.ReleaseAsync();
    Console.WriteLine("Lease released");
}
```

#### Renew Lease (Keep-Alive Pattern)

```csharp
public async Task PerformLongOperationWithLeaseAsync(
    BlobClient blobClient,
    Func<Task> longOperation,
    CancellationToken cancellationToken)
{
    var leaseClient = blobClient.GetBlobLeaseClient();
    var leaseResponse = await leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));
    var leaseId = leaseResponse.Value.LeaseId;

    // Renew lease every 45 seconds (before 60-second expiration)
    using var renewTimer = new Timer(
        async _ =>
        {
            try
            {
                await leaseClient.RenewAsync();
                Console.WriteLine($"Lease renewed: {DateTime.UtcNow}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Failed to renew lease: {ex.Message}");
            }
        },
        null,
        TimeSpan.FromSeconds(45),
        TimeSpan.FromSeconds(45)
    );

    try
    {
        // Perform long operation
        await longOperation();
    }
    finally
    {
        await leaseClient.ReleaseAsync();
        Console.WriteLine("Lease released after long operation");
    }
}

// Usage
await PerformLongOperationWithLeaseAsync(
    blobClient,
    async () =>
    {
        // Simulate long processing (5 minutes)
        for (int i = 0; i < 10; i++)
        {
            Console.WriteLine($"Processing step {i + 1}/10");
            await Task.Delay(TimeSpan.FromSeconds(30));
        }
    },
    CancellationToken.None
);
```

#### Break Lease (Force Release)

```csharp
// Break lease immediately (for emergency situations)
var leaseClient = blobClient.GetBlobLeaseClient();

try
{
    await leaseClient.BreakAsync(breakPeriod: TimeSpan.Zero);
    Console.WriteLine("Lease broken successfully");
}
catch (RequestFailedException ex) when (ex.Status == 409)
{
    Console.WriteLine("No active lease to break");
}
```

### Infinite Lease Pattern

```csharp
// Acquire infinite lease (requires manual release)
var leaseClient = blobClient.GetBlobLeaseClient();
var leaseResponse = await leaseClient.AcquireAsync(TimeSpan.FromSeconds(-1));
var leaseId = leaseResponse.Value.LeaseId;

Console.WriteLine($"Acquired infinite lease: {leaseId}");

// Infinite lease never expires, must be explicitly released
// Useful for administrative locks or long-running workflows

// ... perform operations ...

// Must release manually
await leaseClient.ReleaseAsync();
```

**When to use infinite lease**:
- Administrative locks (prevent accidental deletion)
- Long-running batch operations
- Manual coordination between teams

## Distributed Coordination Patterns

### Pattern 1: Leader Election

```csharp
public class BlobLeaderElection
{
    private readonly BlobClient _leaderBlob;
    private readonly string _instanceId;
    private BlobLeaseClient? _leaseClient;
    private Timer? _renewTimer;
    private bool _isLeader;

    public BlobLeaderElection(BlobContainerClient container, string instanceId)
    {
        _leaderBlob = container.GetBlobClient("leader-election");
        _instanceId = instanceId;
    }

    public async Task<bool> TryBecomeLeaderAsync()
    {
        try
        {
            // Ensure blob exists
            await _leaderBlob.UploadAsync(
                new BinaryData(_instanceId),
                overwrite: false
            );
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            // Blob already exists, that's fine
        }

        _leaseClient = _leaderBlob.GetBlobLeaseClient(_instanceId);

        try
        {
            // Try to acquire lease
            var leaseResponse = await _leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));
            _isLeader = true;

            // Update blob content with leader info
            var uploadOptions = new BlobUploadOptions
            {
                Conditions = new BlobRequestConditions
                {
                    LeaseId = leaseResponse.Value.LeaseId
                }
            };

            await _leaderBlob.UploadAsync(
                new BinaryData($"Leader: {_instanceId}, Time: {DateTime.UtcNow:o}"),
                uploadOptions
            );

            // Start renewal timer
            StartLeaseRenewal();

            Console.WriteLine($"[{_instanceId}] Became leader");
            return true;
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            Console.WriteLine($"[{_instanceId}] Failed to become leader - lease held by another instance");
            return false;
        }
    }

    private void StartLeaseRenewal()
    {
        _renewTimer = new Timer(
            async _ =>
            {
                try
                {
                    if (_isLeader && _leaseClient != null)
                    {
                        await _leaseClient.RenewAsync();
                        Console.WriteLine($"[{_instanceId}] Lease renewed");
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"[{_instanceId}] Lost leadership: {ex.Message}");
                    _isLeader = false;
                }
            },
            null,
            TimeSpan.FromSeconds(45),
            TimeSpan.FromSeconds(45)
        );
    }

    public async Task StepDownAsync()
    {
        if (_leaseClient != null && _isLeader)
        {
            _renewTimer?.Dispose();
            await _leaseClient.ReleaseAsync();
            _isLeader = false;
            Console.WriteLine($"[{_instanceId}] Stepped down as leader");
        }
    }

    public bool IsLeader => _isLeader;
}

// Usage: Multiple instances compete for leadership
var leaderElection = new BlobLeaderElection(containerClient, $"instance-{Environment.MachineName}");

while (true)
{
    if (!leaderElection.IsLeader)
    {
        await leaderElection.TryBecomeLeaderAsync();
    }

    if (leaderElection.IsLeader)
    {
        // Perform leader-only tasks
        Console.WriteLine("Executing leader responsibilities...");
        await Task.Delay(TimeSpan.FromSeconds(10));
    }
    else
    {
        // Wait before retry
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
}
```

### Pattern 2: Distributed Lock

```csharp
public class DistributedBlobLock : IAsyncDisposable
{
    private readonly BlobClient _lockBlob;
    private BlobLeaseClient? _leaseClient;
    private string? _leaseId;

    public DistributedBlobLock(BlobContainerClient container, string lockName)
    {
        _lockBlob = container.GetBlobClient($"locks/{lockName}");
    }

    public async Task<bool> AcquireAsync(TimeSpan timeout)
    {
        var deadline = DateTime.UtcNow.Add(timeout);

        // Ensure lock blob exists
        try
        {
            await _lockBlob.UploadAsync(new BinaryData("lock"), overwrite: false);
        }
        catch (RequestFailedException ex) when (ex.Status == 409) { }

        _leaseClient = _lockBlob.GetBlobLeaseClient();

        while (DateTime.UtcNow < deadline)
        {
            try
            {
                var leaseResponse = await _leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));
                _leaseId = leaseResponse.Value.LeaseId;
                Console.WriteLine($"Lock acquired: {_lockBlob.Name}");
                return true;
            }
            catch (RequestFailedException ex) when (ex.Status == 409)
            {
                // Lock held by another client, retry
                await Task.Delay(TimeSpan.FromMilliseconds(100));
            }
        }

        Console.WriteLine($"Failed to acquire lock within {timeout}");
        return false;
    }

    public async ValueTask DisposeAsync()
    {
        if (_leaseClient != null && _leaseId != null)
        {
            try
            {
                await _leaseClient.ReleaseAsync();
                Console.WriteLine($"Lock released: {_lockBlob.Name}");
            }
            catch { }
        }
    }
}

// Usage: Ensure only one instance processes a resource
await using var lock = new DistributedBlobLock(containerClient, "process-invoices");

if (await lock.AcquireAsync(TimeSpan.FromSeconds(30)))
{
    // Critical section - only one instance executes this
    Console.WriteLine("Processing invoices...");
    await ProcessInvoicesAsync();
}
else
{
    Console.WriteLine("Another instance is processing invoices");
}
```

### Pattern 3: Work Queue with Lease-Based Visibility

```csharp
public class BlobWorkQueue
{
    private readonly BlobContainerClient _containerClient;

    public BlobWorkQueue(BlobContainerClient containerClient)
    {
        _containerClient = containerClient;
    }

    public async Task EnqueueAsync(string workItemId, object data)
    {
        var blobClient = _containerClient.GetBlobClient($"queue/{workItemId}");
        await blobClient.UploadAsync(
            BinaryData.FromObjectAsJson(new
            {
                Id = workItemId,
                Data = data,
                EnqueuedAt = DateTime.UtcNow
            }),
            overwrite: false
        );
    }

    public async Task<WorkItem?> DequeueAsync(TimeSpan visibilityTimeout)
    {
        // List available work items
        await foreach (var blobItem in _containerClient.GetBlobsAsync(prefix: "queue/"))
        {
            var blobClient = _containerClient.GetBlobClient(blobItem.Name);
            var leaseClient = blobClient.GetBlobLeaseClient();

            try
            {
                // Try to acquire lease (claim work item)
                var leaseResponse = await leaseClient.AcquireAsync(visibilityTimeout);
                var leaseId = leaseResponse.Value.LeaseId;

                // Download work item data
                var downloadResponse = await blobClient.DownloadContentAsync();
                var content = downloadResponse.Value.Content.ToString();

                return new WorkItem
                {
                    BlobName = blobItem.Name,
                    LeaseId = leaseId,
                    Content = content,
                    BlobClient = blobClient,
                    LeaseClient = leaseClient
                };
            }
            catch (RequestFailedException ex) when (ex.Status == 409)
            {
                // Already leased by another worker, try next item
                continue;
            }
        }

        return null; // No available work items
    }

    public async Task CompleteAsync(WorkItem workItem)
    {
        // Delete work item after processing
        var deleteOptions = new BlobRequestConditions
        {
            LeaseId = workItem.LeaseId
        };

        await workItem.BlobClient.DeleteAsync(conditions: deleteOptions);
        Console.WriteLine($"Work item completed: {workItem.BlobName}");
    }

    public async Task AbandonAsync(WorkItem workItem)
    {
        // Release lease without deleting (item becomes available again)
        await workItem.LeaseClient.ReleaseAsync();
        Console.WriteLine($"Work item abandoned: {workItem.BlobName}");
    }
}

public class WorkItem
{
    public string BlobName { get; set; } = "";
    public string LeaseId { get; set; } = "";
    public string Content { get; set; } = "";
    public BlobClient BlobClient { get; set; } = null!;
    public BlobLeaseClient LeaseClient { get; set; } = null!;
}

// Usage: Distributed worker pool
var workQueue = new BlobWorkQueue(containerClient);

// Producer
await workQueue.EnqueueAsync("job-001", new { Task = "Process invoice", InvoiceId = 123 });
await workQueue.EnqueueAsync("job-002", new { Task = "Generate report", ReportId = 456 });

// Consumer
while (true)
{
    var workItem = await workQueue.DequeueAsync(TimeSpan.FromSeconds(60));

    if (workItem != null)
    {
        try
        {
            Console.WriteLine($"Processing: {workItem.Content}");
            // Process work item...
            await Task.Delay(TimeSpan.FromSeconds(5));

            await workQueue.CompleteAsync(workItem);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error processing work item: {ex.Message}");
            await workQueue.AbandonAsync(workItem);
        }
    }
    else
    {
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
}
```

## Performance and Best Practices

### Concurrency Control Performance

| Mechanism | Latency Overhead | Throughput Impact | Best For |
|-----------|------------------|-------------------|----------|
| **ETag (IfMatch)** | ~5ms | Minimal | Optimistic concurrency, low contention |
| **Lease (60s)** | ~10ms acquire, ~5ms renew | Moderate | Distributed coordination, moderate contention |
| **Infinite Lease** | ~10ms acquire | High (blocks others) | Administrative locks, single-writer |

### Best Practices

#### 1. Prefer Optimistic Concurrency (ETags)

✅ **Use ETags when**:
- Multiple readers, occasional writers
- Conflicts are rare (<10% of operations)
- Can retry on conflict

❌ **Don't use ETags when**:
- High contention (>50% conflicts)
- Need exclusive access guarantee
- Can't handle retries

#### 2. Use Leases for Coordination

✅ **Use Leases when**:
- Need exclusive write access
- Distributed worker pools
- Leader election scenarios
- Prevent concurrent processing

❌ **Don't use Leases when**:
- Single-threaded application
- Low contention scenarios
- Need read-write locking (leases don't block reads)

#### 3. Always Release Leases

```csharp
// ✅ Good: Always release in finally block
var leaseClient = blobClient.GetBlobLeaseClient();
var leaseResponse = await leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));

try
{
    // Critical section
}
finally
{
    await leaseClient.ReleaseAsync();
}

// ✅ Better: Use IAsyncDisposable pattern
public class LeasedBlobOperation : IAsyncDisposable
{
    private readonly BlobLeaseClient _leaseClient;

    public async Task<LeasedBlobOperation> CreateAsync(BlobClient blobClient)
    {
        var leaseClient = blobClient.GetBlobLeaseClient();
        await leaseClient.AcquireAsync(TimeSpan.FromSeconds(60));
        return new LeasedBlobOperation(leaseClient);
    }

    private LeasedBlobOperation(BlobLeaseClient leaseClient)
    {
        _leaseClient = leaseClient;
    }

    public async ValueTask DisposeAsync()
    {
        await _leaseClient.ReleaseAsync();
    }
}

// Usage
await using var leasedOperation = await LeasedBlobOperation.CreateAsync(blobClient);
// Lease automatically released on dispose
```

#### 4. Set Appropriate Lease Duration

- **Short operations (<30s)**: 30-60 second lease
- **Long operations (1-5 min)**: 60 second lease with renewal
- **Very long operations (>5 min)**: Consider work queue pattern instead
- **Administrative locks**: Infinite lease (manual release)

#### 5. Implement Exponential Backoff

```csharp
public static async Task<T> RetryWithBackoffAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 5)
{
    var random = new Random();

    for (int retry = 0; retry < maxRetries; retry++)
    {
        try
        {
            return await operation();
        }
        catch (RequestFailedException ex) when (ex.Status == 412 || ex.Status == 409)
        {
            if (retry == maxRetries - 1)
                throw;

            var backoffMs = (int)(Math.Pow(2, retry) * 100 + random.Next(0, 100));
            await Task.Delay(backoffMs);
        }
    }

    throw new InvalidOperationException("Unreachable code");
}

// Usage
var result = await RetryWithBackoffAsync(async () =>
{
    var props = await blobClient.GetPropertiesAsync();
    var uploadOptions = new BlobUploadOptions
    {
        Conditions = new BlobRequestConditions { IfMatch = props.Value.ETag }
    };
    return await blobClient.UploadAsync(content, uploadOptions);
});
```

## Error Handling

### Common Error Codes

| Status Code | Error | Cause | Solution |
|-------------|-------|-------|----------|
| **412** | Precondition Failed | ETag mismatch, IfMatch failed | Retry with fresh ETag |
| **304** | Not Modified | IfNoneMatch matched | Use cached version |
| **409** | Conflict | Lease already held, blob exists | Wait and retry, or break lease |
| **404** | Not Found | Blob doesn't exist | Create blob first |

### Robust Error Handling Pattern

```csharp
public async Task<BlobUploadResult> UploadWithConcurrencyControlAsync(
    BlobClient blobClient,
    BinaryData content,
    ETag? expectedETag = null)
{
    try
    {
        var uploadOptions = new BlobUploadOptions();

        if (expectedETag.HasValue)
        {
            uploadOptions.Conditions = new BlobRequestConditions
            {
                IfMatch = expectedETag.Value
            };
        }

        var response = await blobClient.UploadAsync(content, uploadOptions);

        return new BlobUploadResult
        {
            Success = true,
            ETag = response.Value.ETag
        };
    }
    catch (RequestFailedException ex) when (ex.Status == 412)
    {
        return new BlobUploadResult
        {
            Success = false,
            Error = "Blob was modified by another client",
            ErrorType = ConcurrencyErrorType.ETagMismatch
        };
    }
    catch (RequestFailedException ex) when (ex.Status == 409)
    {
        return new BlobUploadResult
        {
            Success = false,
            Error = "Blob is locked by another client",
            ErrorType = ConcurrencyErrorType.LeaseConflict
        };
    }
    catch (RequestFailedException ex) when (ex.Status == 404)
    {
        return new BlobUploadResult
        {
            Success = false,
            Error = "Blob or container not found",
            ErrorType = ConcurrencyErrorType.NotFound
        };
    }
}

public class BlobUploadResult
{
    public bool Success { get; set; }
    public ETag? ETag { get; set; }
    public string? Error { get; set; }
    public ConcurrencyErrorType? ErrorType { get; set; }
}

public enum ConcurrencyErrorType
{
    ETagMismatch,
    LeaseConflict,
    NotFound
}
```

## Real-World Scenarios

### Scenario 1: Configuration Management

**Requirement**: Multiple instances read config, single admin updates, prevent inconsistent reads

```csharp
public class DistributedConfigManager
{
    private readonly BlobClient _configBlob;
    private (ETag ETag, AppConfig Config)? _cachedConfig;

    public async Task<AppConfig> GetConfigAsync()
    {
        try
        {
            var downloadOptions = new BlobDownloadOptions();

            if (_cachedConfig.HasValue)
            {
                downloadOptions.Conditions = new BlobRequestConditions
                {
                    IfNoneMatch = _cachedConfig.Value.ETag
                };
            }

            var response = await _configBlob.DownloadContentAsync(downloadOptions);
            var config = response.Value.Content.ToObjectFromJson<AppConfig>();
            _cachedConfig = (response.Value.Details.ETag, config);

            return config;
        }
        catch (RequestFailedException ex) when (ex.Status == 304)
        {
            // Not modified, return cached
            return _cachedConfig!.Value.Config;
        }
    }

    public async Task<bool> UpdateConfigAsync(Func<AppConfig, AppConfig> updateFunc)
    {
        return await RetryWithBackoffAsync(async () =>
        {
            var downloadResponse = await _configBlob.DownloadContentAsync();
            var currentConfig = downloadResponse.Value.Content.ToObjectFromJson<AppConfig>();
            var currentETag = downloadResponse.Value.Details.ETag;

            var updatedConfig = updateFunc(currentConfig);

            var uploadOptions = new BlobUploadOptions
            {
                Conditions = new BlobRequestConditions { IfMatch = currentETag }
            };

            await _configBlob.UploadAsync(
                BinaryData.FromObjectAsJson(updatedConfig),
                uploadOptions
            );

            _cachedConfig = null; // Invalidate cache
            return true;
        });
    }
}

public class AppConfig
{
    public string Environment { get; set; } = "";
    public Dictionary<string, string> Settings { get; set; } = new();
}
```

### Scenario 2: Distributed Counter

**Requirement**: Increment counter across distributed system

```csharp
public class DistributedCounter
{
    private readonly BlobClient _counterBlob;

    public async Task<long> IncrementAsync(long delta = 1)
    {
        return await RetryWithBackoffAsync(async () =>
        {
            var downloadResponse = await _counterBlob.DownloadContentAsync();
            var currentValue = long.Parse(downloadResponse.Value.Content.ToString());
            var currentETag = downloadResponse.Value.Details.ETag;

            var newValue = currentValue + delta;

            var uploadOptions = new BlobUploadOptions
            {
                Conditions = new BlobRequestConditions { IfMatch = currentETag }
            };

            await _counterBlob.UploadAsync(
                new BinaryData(newValue.ToString()),
                uploadOptions
            );

            return newValue;
        }, maxRetries: 10);
    }

    public async Task<long> GetValueAsync()
    {
        var response = await _counterBlob.DownloadContentAsync();
        return long.Parse(response.Value.Content.ToString());
    }
}
```

## References

**Microsoft Learn Documentation**:
- [Optimistic Concurrency](https://learn.microsoft.com/azure/storage/blobs/concurrency-manage)
- [Blob Leases](https://learn.microsoft.com/rest/api/storageservices/lease-blob)
- [Conditional Headers](https://learn.microsoft.com/azure/storage/common/storage-concurrency)
- [ETags](https://learn.microsoft.com/rest/api/storageservices/specifying-conditional-headers-for-blob-service-operations)

## Navigation

- **Previous**: `blob-fundamentals.md`
- **Next**: `blob-lifecycle-management.md`
- **Related**: `06-operations/data-access-layer.md` - Client implementation patterns

## See Also

- `blob-implementation.md` - Upload/download patterns
- `06-operations/performance-optimization.md` - Throughput optimization
- `02-core-concepts/security.md` - Access control and authentication
