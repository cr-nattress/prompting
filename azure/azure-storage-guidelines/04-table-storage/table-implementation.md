# Table Storage Implementation - CRUD & Queries

> **File Purpose**: Practical CRUD operations, batch transactions, and query patterns for Azure Table Storage
> **Prerequisites**: `01-quick-start/authentication-setup.md`, `04-table-storage/table-fundamentals.md`
> **Agent Use Case**: Reference when implementing table storage operations

## Quick Context

Azure Table Storage is a NoSQL key-value store optimized for fast lookups via PartitionKey and RowKey. This guide covers entity modeling, CRUD operations, batch transactions (same partition only), pagination, and query optimization.

**Key principle**: Design PartitionKey for scale (avoid hot partitions); design RowKey for query patterns. Always use batch operations for same-partition writes.

## Prerequisites

```bash
dotnet add package Azure.Data.Tables --version 12.9.0
dotnet add package Azure.Identity --version 1.13.0
```

## Entity Modeling

### Define Table Entity

```csharp
using Azure;
using Azure.Data.Tables;

// Approach 1: Record with ITableEntity (recommended)
public record UserEntity(string PartitionKey, string RowKey) : ITableEntity
{
    public string? Email { get; init; }
    public string? DisplayName { get; init; }
    public DateTimeOffset? CreatedAt { get; init; }
    public bool IsActive { get; init; } = true;

    // Required by ITableEntity
    public ETag ETag { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
}

// Approach 2: Class inheriting TableEntity (alternative)
public class ProductEntity : ITableEntity
{
    public string PartitionKey { get; set; } = default!;
    public string RowKey { get; set; } = default!;
    public string? Name { get; set; }
    public decimal Price { get; set; }
    public int StockLevel { get; set; }
    public ETag ETag { get; set; }
    public DateTimeOffset? Timestamp { get; set; }
}
```

**PartitionKey and RowKey Design**:
- **PartitionKey**: Logical grouping (e.g., `tenant-acme`, `user-region-west`)
- **RowKey**: Unique identifier within partition (e.g., `user-0001`, `order-2025-10-06-12345`)
- **Composite**: `PartitionKey + RowKey` forms primary key (must be unique)

See `04-table-storage/table-fundamentals.md` for partition design strategies.

## Client Initialization

```csharp
using Azure.Identity;
using Azure.Data.Tables;

var credential = new DefaultAzureCredential();
var tableServiceUri = new Uri("https://storacct....table.core.windows.net");
var tableServiceClient = new TableServiceClient(tableServiceUri, credential);

// Get table client
var tableClient = tableServiceClient.GetTableClient("Users");
await tableClient.CreateIfNotExistsAsync();
```

## CRUD Operations

### Create (Insert) Entity

```csharp
var entity = new UserEntity("tenant-acme", "user-0001")
{
    Email = "alice@example.com",
    DisplayName = "Alice Smith",
    CreatedAt = DateTimeOffset.UtcNow,
    IsActive = true
};

// AddEntity: Fails if entity exists (409 Conflict)
await tableClient.AddEntityAsync(entity);

// UpsertEntity: Insert or replace (overwrite)
await tableClient.UpsertEntityAsync(entity, TableUpdateMode.Replace);

// UpsertEntity: Insert or merge (update only specified properties)
await tableClient.UpsertEntityAsync(entity, TableUpdateMode.Merge);
```

**When to use**:
- `AddEntity`: Insert-only (fail if exists)
- `UpsertEntity(Replace)`: Insert or full overwrite
- `UpsertEntity(Merge)`: Insert or partial update

**Common mistakes**:
- Using `AddEntity` when entity might exist (causes 409 errors)
- Not setting both PartitionKey and RowKey

### Read (Get) Entity

```csharp
// Get by PartitionKey + RowKey
var response = await tableClient.GetEntityAsync<UserEntity>("tenant-acme", "user-0001");
var user = response.Value;

Console.WriteLine($"User: {user.DisplayName}, Email: {user.Email}");

// Handle not found
try
{
    var user = await tableClient.GetEntityAsync<UserEntity>("tenant-acme", "user-9999");
}
catch (Azure.RequestFailedException ex) when (ex.Status == 404)
{
    Console.WriteLine("User not found");
}
```

### Update Entity

```csharp
// Get current entity (for ETag)
var response = await tableClient.GetEntityAsync<UserEntity>("tenant-acme", "user-0001");
var user = response.Value;

// Modify properties
var updatedUser = user with { DisplayName = "Alice Johnson", IsActive = false };

// Update with ETag (optimistic concurrency)
await tableClient.UpdateEntityAsync(updatedUser, user.ETag, TableUpdateMode.Replace);

// Update without ETag check (overwrite regardless)
await tableClient.UpdateEntityAsync(updatedUser, ETag.All, TableUpdateMode.Replace);
```

**UpdateMode**:
- `Replace`: Overwrites entire entity (removes unspecified properties)
- `Merge`: Updates only specified properties (preserves others)

**Common mistakes**:
- Using `Replace` when you want `Merge` (loses unspecified properties)
- Not handling ETag conflicts (412 Precondition Failed)

### Delete Entity

```csharp
// Delete with ETag (safe delete, fails if modified)
var response = await tableClient.GetEntityAsync<UserEntity>("tenant-acme", "user-0001");
await tableClient.DeleteEntityAsync("tenant-acme", "user-0001", response.Value.ETag);

// Delete without ETag check (force delete)
await tableClient.DeleteEntityAsync("tenant-acme", "user-0001", ETag.All);
```

## Batch Operations (Transactions)

### Batch Insert/Upsert (Same Partition)

**Critical constraint**: All entities in batch MUST have the same PartitionKey.

```csharp
var batch = new List<TableTransactionAction>();

// Add multiple entities to batch
batch.Add(new TableTransactionAction(
    TableTransactionActionType.Add,
    new UserEntity("tenant-acme", "user-0002") { Email = "bob@example.com", DisplayName = "Bob" }
));

batch.Add(new TableTransactionAction(
    TableTransactionActionType.UpsertMerge,
    new UserEntity("tenant-acme", "user-0003") { Email = "carol@example.com", DisplayName = "Carol" }
));

batch.Add(new TableTransactionAction(
    TableTransactionActionType.UpdateReplace,
    new UserEntity("tenant-acme", "user-0001") { Email = "alice.new@example.com", DisplayName = "Alice Updated" }
));

// Submit batch (all-or-nothing transaction)
try
{
    var response = await tableClient.SubmitTransactionAsync(batch);
    Console.WriteLine($"Batch completed: {response.Value.Count} operations");
}
catch (Azure.RequestFailedException ex) when (ex.Status == 400)
{
    Console.WriteLine($"Batch failed: {ex.Message}");
    // Entire batch is rolled back
}
```

**Why it works**: Single HTTP request, atomic transaction (all succeed or all fail), up to 100 operations per batch.

**Limitations**:
- All entities must share same PartitionKey
- Max 100 operations per batch
- Max 4MB batch size
- No Query operations in batch

**Common mistakes**:
- Mixing PartitionKeys (causes 400 Bad Request)
- Exceeding 100 operations (split into multiple batches)

### Batch Delete

```csharp
var batch = new List<TableTransactionAction>
{
    new(TableTransactionActionType.Delete, new UserEntity("tenant-acme", "user-0002")),
    new(TableTransactionActionType.Delete, new UserEntity("tenant-acme", "user-0003"))
};

await tableClient.SubmitTransactionAsync(batch);
```

## Query Patterns

### Query All Entities in Partition

```csharp
var partitionKey = "tenant-acme";
var filter = TableClient.CreateQueryFilter($"PartitionKey eq {partitionKey}");

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter))
{
    Console.WriteLine($"{user.RowKey}: {user.DisplayName}");
}
```

**Why it works**: Efficient (single partition scan). Always filter by PartitionKey when possible.

### Query with RowKey Range

```csharp
var filter = TableClient.CreateQueryFilter(
    $"PartitionKey eq {partitionKey} and RowKey ge {"user-0002"} and RowKey lt {"user-1000"}"
);

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter))
{
    Console.WriteLine($"{user.RowKey}: {user.Email}");
}
```

**Why it works**: Uses index on PartitionKey + RowKey. Very fast.

### Query with Property Filters

```csharp
// Filter by non-indexed property (slower)
var filter = TableClient.CreateQueryFilter(
    $"PartitionKey eq {partitionKey} and IsActive eq {true}"
);

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter))
{
    Console.WriteLine($"{user.RowKey}: {user.DisplayName}");
}
```

**Performance note**: Only PartitionKey and RowKey are indexed. Filtering on other properties requires full partition scan.

### Query with Projection (Select Specific Properties)

```csharp
var filter = TableClient.CreateQueryFilter($"PartitionKey eq {partitionKey}");
var select = new[] { "Email", "DisplayName" };

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter, select: select))
{
    Console.WriteLine($"{user.Email}"); // RowKey, PartitionKey, Timestamp still available
    // Other properties will be null/default
}
```

**Why it works**: Reduces bandwidth by fetching only required properties.

### Query with Pagination

```csharp
var filter = TableClient.CreateQueryFilter($"PartitionKey eq {partitionKey}");
var pageSize = 100;

var pages = tableClient.QueryAsync<UserEntity>(filter: filter, maxPerPage: pageSize).AsPages();

await foreach (var page in pages)
{
    Console.WriteLine($"Page with {page.Values.Count} items");
    foreach (var user in page.Values)
    {
        Console.WriteLine($"  {user.RowKey}: {user.DisplayName}");
    }

    // Get continuation token for next page
    var continuationToken = page.ContinuationToken;
    if (continuationToken != null)
    {
        Console.WriteLine("More results available");
    }
}
```

**Why it works**: Pages avoid loading all results into memory. Continuation tokens enable stateless pagination.

**Common mistakes**:
- Not handling pagination (queries may not return all results)
- Storing continuation tokens as strings (use opaque token as-is)

### Query Across All Partitions (Avoid When Possible)

```csharp
// Scan all partitions (expensive)
var filter = TableClient.CreateQueryFilter($"IsActive eq {true}");

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter))
{
    Console.WriteLine($"{user.PartitionKey}/{user.RowKey}: {user.DisplayName}");
}
```

**Performance warning**: Full table scan. Avoid in production unless table is small (<10K entities).

## Advanced Patterns

### Conditional Operations (ETags)

```csharp
// Get entity
var response = await tableClient.GetEntityAsync<UserEntity>("tenant-acme", "user-0001");
var user = response.Value;
var etag = user.ETag;

// Attempt update (fails if entity was modified)
try
{
    var updatedUser = user with { DisplayName = "Alice Modified" };
    await tableClient.UpdateEntityAsync(updatedUser, etag, TableUpdateMode.Replace);
    Console.WriteLine("Update successful");
}
catch (Azure.RequestFailedException ex) when (ex.Status == 412)
{
    Console.WriteLine("Entity was modified by another process, re-fetch and retry");
}
```

See `03-blob-storage/blob-concurrency.md` for ETag patterns (same concepts apply).

### Timestamp-Based Queries

```csharp
// Query entities modified in last hour
var oneHourAgo = DateTimeOffset.UtcNow.AddHours(-1);
var filter = TableClient.CreateQueryFilter(
    $"PartitionKey eq {partitionKey} and Timestamp ge {oneHourAgo}"
);

await foreach (var user in tableClient.QueryAsync<UserEntity>(filter: filter))
{
    Console.WriteLine($"{user.RowKey} modified at {user.Timestamp}");
}
```

**Note**: `Timestamp` is system-managed (read-only). It represents last modification time.

### Secondary Index Pattern (Manual)

**Problem**: Need to query by Email (not indexed).

**Solution**: Maintain a secondary table with Email as PartitionKey.

```csharp
// Primary table: Users (PK: tenant, RK: userId)
await tableClient.AddEntityAsync(new UserEntity("tenant-acme", "user-0001")
{
    Email = "alice@example.com",
    DisplayName = "Alice"
});

// Secondary index table: UsersByEmail (PK: email, RK: userId)
var indexTable = tableServiceClient.GetTableClient("UsersByEmail");
await indexTable.AddEntityAsync(new TableEntity("alice@example.com", "user-0001")
{
    ["TenantId"] = "tenant-acme"
});

// Query by email (fast)
var emailLookup = await indexTable.GetEntityAsync<TableEntity>("alice@example.com", "user-0001");
var userId = emailLookup.Value.RowKey;
var tenantId = emailLookup.Value["TenantId"].ToString();

// Fetch full user from primary table
var user = await tableClient.GetEntityAsync<UserEntity>(tenantId, userId);
```

See `04-table-storage/table-patterns.md` for comprehensive secondary index strategies.

## Error Handling

```csharp
try
{
    await tableClient.AddEntityAsync(entity);
}
catch (RequestFailedException ex) when (ex.Status == 404)
{
    Console.WriteLine("Table does not exist");
}
catch (RequestFailedException ex) when (ex.Status == 409)
{
    Console.WriteLine("Entity already exists");
}
catch (RequestFailedException ex) when (ex.Status == 412)
{
    Console.WriteLine("ETag mismatch (entity was modified)");
}
catch (RequestFailedException ex) when (ex.Status == 400)
{
    Console.WriteLine($"Bad request: {ex.Message}");
}
```

## Performance Checklist

- [ ] Queries filter by PartitionKey (avoid full table scans)
- [ ] PartitionKey distributes load evenly (no hot partitions)
- [ ] RowKey enables range queries for common access patterns
- [ ] Batch operations used for multiple writes to same partition
- [ ] Pagination implemented with continuation tokens
- [ ] Projection (select) used to reduce bandwidth
- [ ] ETags used for optimistic concurrency control
- [ ] Secondary indexes maintained for non-PK/RK queries

## Navigation

- **Previous**: `table-fundamentals.md`
- **Next**: `table-patterns.md`
- **Up**: `00-overview.md`

## See Also

- `04-table-storage/table-fundamentals.md` - Partition/RowKey design
- `04-table-storage/table-patterns.md` - Hot partition avoidance, secondary indexes
- `08-reference/code-examples.md` - Complete examples
