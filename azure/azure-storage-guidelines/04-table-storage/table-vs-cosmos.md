# Table Storage vs Cosmos DB: Decision Guide

> **File Purpose**: Comprehensive comparison of Azure Table Storage and Cosmos DB Table API with feature analysis, cost comparison, and migration strategies
> **Prerequisites**: `table-fundamentals.md`, `table-patterns.md`
> **Agent Use Case**: Reference when choosing between Table Storage and Cosmos DB, or planning migration between services

## Quick Context

Azure Table Storage and Cosmos DB Table API both provide NoSQL key-value storage, but Cosmos DB offers global distribution, low latency SLAs, and advanced features at higher cost.

**Key principle**: Start with Table Storage for cost-effective NoSQL storage; migrate to Cosmos DB only when you need global distribution, single-digit millisecond latency, or advanced features.

## Feature Comparison

### Core Capabilities

| Feature | Table Storage | Cosmos DB Table API | Winner |
|---------|--------------|---------------------|--------|
| **Latency (read)** | <10ms (regional) | <10ms (99th percentile, SLA) | Cosmos DB (guaranteed) |
| **Latency (write)** | <15ms (regional) | <15ms (99th percentile, SLA) | Cosmos DB (guaranteed) |
| **Throughput** | 20,000 ops/sec per account | Unlimited (provisioned or auto-scale) | Cosmos DB |
| **Global distribution** | No | Yes (multi-region writes) | Cosmos DB |
| **Consistency models** | Strong (single region) | 5 models (Strong, Bounded, Session, Consistent Prefix, Eventual) | Cosmos DB |
| **Indexing** | PK/RK only | Automatic indexing of all properties | Cosmos DB |
| **Query capabilities** | PK/RK filters, simple queries | Rich queries, aggregations, joins | Cosmos DB |
| **SLA** | 99.9% availability | 99.99% single-region, 99.999% multi-region | Cosmos DB |
| **Max entity size** | 1 MB | 2 MB | Cosmos DB |
| **Max properties** | 252 | Unlimited (nested JSON) | Cosmos DB |

### Cost Comparison (Monthly, US East)

**Scenario 1: Small workload** (100 GB storage, 1M operations/month)

| Service | Storage | Operations | Total |
|---------|---------|------------|-------|
| **Table Storage** | $10 | $0.40 | **$10.40** |
| **Cosmos DB (400 RU/s)** | $25 | $28.80 | **$53.80** |
| **Cost difference** | - | - | **5.2x more expensive** |

**Scenario 2: Medium workload** (1 TB storage, 100M operations/month)

| Service | Storage | Operations | Total |
|---------|---------|------------|-------|
| **Table Storage** | $100 | $40 | **$140** |
| **Cosmos DB (4,000 RU/s)** | $250 | $288 | **$538** |
| **Cost difference** | - | - | **3.8x more expensive** |

**Scenario 3: Large workload** (10 TB storage, 1B operations/month)

| Service | Storage | Operations | Total |
|---------|---------|------------|-------|
| **Table Storage** | $1,000 | $400 | **$1,400** |
| **Cosmos DB (40,000 RU/s)** | $2,500 | $2,880 | **$5,380** |
| **Cost difference** | - | - | **3.8x more expensive** |

**Key insight**: Cosmos DB costs 3-5x more for similar workloads, but provides advanced features and global distribution.

### Pricing Models

#### Table Storage Pricing

```
Storage: $0.10/GB/month
Transactions: $0.0004/10K operations

Example calculation (1 TB, 100M ops/month):
- Storage: 1,000 GB × $0.10 = $100
- Operations: 100,000,000 / 10,000 × $0.0004 = $40
Total: $140/month
```

#### Cosmos DB Pricing

```
Storage: $0.25/GB/month
Request Units (RU/s):
- Provisioned: $0.008/100 RU/s/hour ($5.76/100 RU/s/month)
- Serverless: $0.25/million RUs consumed

Example calculation (1 TB, 4,000 RU/s provisioned):
- Storage: 1,000 GB × $0.25 = $250
- RU/s: 4,000 / 100 × $5.76 = $230.40
- Multi-region write: +100% = $230.40
Total: $710.80/month (multi-region)
```

## When to Choose Each Service

### Choose Table Storage When:

✅ **Cost is primary concern**
- Budget-constrained projects
- Dev/test environments
- Non-critical workloads

✅ **Simple key-value access patterns**
- Point queries (PK + RK)
- Partition scans (PK only)
- No complex queries or aggregations

✅ **Single-region deployment**
- Users in single geographic region
- No global distribution needed

✅ **Relaxed latency requirements**
- <100ms read latency acceptable
- No strict SLA requirements

✅ **Predictable, moderate scale**
- <20,000 ops/sec
- Storage growth predictable

**Example use cases**:
- Session state storage
- Application logs
- Device telemetry (regional)
- User profile data (single region)
- Job queue metadata

### Choose Cosmos DB When:

✅ **Global distribution required**
- Users across multiple continents
- Multi-region active-active writes
- Low latency worldwide (<50ms)

✅ **Strict SLA requirements**
- 99.99% single-region availability
- 99.999% multi-region availability
- <10ms read/write latency (guaranteed)

✅ **Complex query requirements**
- Filtering on non-key properties
- Aggregations (SUM, AVG, COUNT)
- Cross-partition queries
- Rich indexing

✅ **Unpredictable scale**
- Auto-scaling needed
- Burst workloads (10x spikes)
- Unlimited throughput requirements

✅ **Advanced features needed**
- Change feed for event-driven architectures
- Time-to-Live (TTL) on entities
- Multi-model support (SQL, MongoDB, Cassandra, Gremlin)
- Analytical store (HTAP scenarios)

**Example use cases**:
- Global e-commerce platforms
- Gaming leaderboards (global)
- IoT device state (multi-region)
- Real-time analytics dashboards
- Social media feeds (worldwide)

## Side-by-Side Code Comparison

### Connection and Setup

#### Table Storage

```csharp
using Azure.Data.Tables;
using Azure.Identity;

// Connection
var tableServiceClient = new TableServiceClient(
    new Uri("https://mystorageaccount.table.core.windows.net"),
    new DefaultAzureCredential()
);

var tableClient = tableServiceClient.GetTableClient("Orders");
await tableClient.CreateIfNotExistsAsync();
```

#### Cosmos DB Table API

```csharp
using Azure.Data.Tables;
using Azure.Identity;

// Connection (same SDK, different endpoint)
var tableServiceClient = new TableServiceClient(
    new Uri("https://mycosmosaccount.table.cosmos.azure.com"),
    new DefaultAzureCredential()
);

var tableClient = tableServiceClient.GetTableClient("Orders");
await tableClient.CreateIfNotExistsAsync();
```

**Key insight**: Same SDK (`Azure.Data.Tables`), just different endpoint!

### Basic CRUD Operations

```csharp
public class OrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public decimal Amount { get; set; }
    public string Status { get; set; } = "";
    public DateTime OrderDate { get; set; }
}

// ✅ Insert (identical for both services)
var order = new OrderEntity
{
    PartitionKey = "customer-123",
    RowKey = "order-456",
    Amount = 99.99m,
    Status = "pending",
    OrderDate = DateTime.UtcNow
};

await tableClient.AddEntityAsync(order);

// ✅ Point query (identical for both services)
var result = await tableClient.GetEntityAsync<OrderEntity>(
    partitionKey: "customer-123",
    rowKey: "order-456"
);

Console.WriteLine($"Order amount: {result.Value.Amount}");

// ✅ Update (identical for both services)
order.Status = "completed";
await tableClient.UpdateEntityAsync(order, ETag.All);

// ✅ Delete (identical for both services)
await tableClient.DeleteEntityAsync("customer-123", "order-456");
```

### Query Differences

#### Table Storage: Limited Queries

```csharp
// ✅ Efficient: Partition query
var partitionQuery = tableClient.QueryAsync<OrderEntity>(
    filter: $"PartitionKey eq 'customer-123'"
);

// ❌ Inefficient: Property filter (scans entire table)
var statusQuery = tableClient.QueryAsync<OrderEntity>(
    filter: $"Status eq 'pending'"
);
// No secondary indexes - full table scan!

// ❌ Not supported: Complex queries
// No aggregations, no joins, limited filtering
```

#### Cosmos DB: Rich Queries

```csharp
// ✅ Efficient: Automatic indexing on all properties
var statusQuery = tableClient.QueryAsync<OrderEntity>(
    filter: $"Status eq 'pending'"
);
// Uses automatic index - fast!

// ✅ Supported: Complex filters
var complexQuery = tableClient.QueryAsync<OrderEntity>(
    filter: $"Status eq 'pending' and Amount gt 100.0 and OrderDate ge datetime'{DateTime.UtcNow.AddDays(-7):o}'"
);

// ✅ Supported: Composite filters
var compositeQuery = tableClient.QueryAsync<OrderEntity>(
    filter: $"(Status eq 'pending' or Status eq 'processing') and Amount le 1000.0"
);
```

**Performance impact**:

| Query Type | Table Storage | Cosmos DB |
|------------|--------------|-----------|
| **Point query (PK+RK)** | ~5ms | ~5ms |
| **Partition query (PK)** | ~10-50ms | ~10-50ms |
| **Property filter (no PK)** | Seconds (table scan) | ~10-100ms (indexed) |
| **Complex filter** | Seconds (table scan) | ~50-200ms (indexed) |

### Global Distribution (Cosmos DB Only)

```csharp
// Configure multi-region writes (Cosmos DB only)
var cosmosClient = new CosmosClient(
    accountEndpoint: "https://mycosmosaccount.documents.azure.com",
    authKeyOrResourceToken: new DefaultAzureCredential(),
    clientOptions: new CosmosClientOptions
    {
        ApplicationRegion = Regions.WestUS,  // Preferred region
        AllowBulkExecution = true
    }
);

// Data automatically replicated to all configured regions
// Reads/writes go to nearest region

// Table Storage: Single region only, no global distribution
```

### Change Feed (Cosmos DB Only)

```csharp
// Cosmos DB: Change feed for event-driven architectures
using Microsoft.Azure.Cosmos;
using Microsoft.Azure.Cosmos.Table;

// Track all changes to Orders table
var processor = cosmosClient
    .GetContainer("OrdersDB", "Orders")
    .GetChangeFeedProcessorBuilder<OrderEntity>(
        processorName: "orderChangeProcessor",
        onChangesDelegate: async (changes, cancellationToken) =>
        {
            foreach (var change in changes)
            {
                Console.WriteLine($"Order changed: {change.RowKey}, Status: {change.Status}");
                // Trigger downstream processing
            }
        })
    .Build();

await processor.StartAsync();

// Table Storage: No change feed support
// Must poll for changes or implement custom tracking
```

## Migration Guide

### Table Storage → Cosmos DB Table API

**Migration strategy**: Zero-downtime migration using dual-write pattern

#### Step 1: Setup Cosmos DB

```bash
# Create Cosmos DB account with Table API
az cosmosdb create \
  --name mycosmosaccount \
  --resource-group myresourcegroup \
  --capabilities EnableTable \
  --locations regionName=eastus failoverPriority=0 \
  --default-consistency-level Session

# Get connection strings
az cosmosdb keys list \
  --name mycosmosaccount \
  --resource-group myresourcegroup \
  --type connection-strings
```

#### Step 2: Dual-Write Pattern

```csharp
public class DualWriteOrderService
{
    private readonly TableClient _tableStorageClient;
    private readonly TableClient _cosmosDbClient;
    private readonly bool _enableCosmosWrites;

    public DualWriteOrderService(
        TableServiceClient tableStorageService,
        TableServiceClient cosmosDbService,
        bool enableCosmosWrites = false)
    {
        _tableStorageClient = tableStorageService.GetTableClient("Orders");
        _cosmosDbClient = cosmosDbService.GetTableClient("Orders");
        _enableCosmosWrites = enableCosmosWrites;
    }

    public async Task CreateOrderAsync(OrderEntity order)
    {
        // Always write to Table Storage (primary)
        await _tableStorageClient.AddEntityAsync(order);

        // Optionally write to Cosmos DB (shadow writes)
        if (_enableCosmosWrites)
        {
            try
            {
                await _cosmosDbClient.AddEntityAsync(order);
            }
            catch (Exception ex)
            {
                // Log error but don't fail the operation
                Console.WriteLine($"Failed to write to Cosmos DB: {ex.Message}");
            }
        }
    }

    // Read from Table Storage or Cosmos DB (A/B testing)
    public async Task<OrderEntity> GetOrderAsync(
        string partitionKey,
        string rowKey,
        bool useCosmosDb = false)
    {
        var client = useCosmosDb ? _cosmosDbClient : _tableStorageClient;
        var result = await client.GetEntityAsync<OrderEntity>(partitionKey, rowKey);
        return result.Value;
    }
}

// Phase 1: Shadow writes to Cosmos DB (validate)
var orderService = new DualWriteOrderService(
    tableStorageService,
    cosmosDbService,
    enableCosmosWrites: true  // Shadow writes
);

// Phase 2: A/B test reads from Cosmos DB
var order = await orderService.GetOrderAsync(
    "customer-123",
    "order-456",
    useCosmosDb: Random.Shared.Next(100) < 10  // 10% traffic to Cosmos
);

// Phase 3: Full cutover to Cosmos DB
```

#### Step 3: Bulk Data Migration

```csharp
public class TableStorageMigrator
{
    private readonly TableClient _sourceClient;
    private readonly TableClient _destinationClient;

    public async Task MigrateAllDataAsync()
    {
        var migratedCount = 0;
        var batchSize = 100;
        var batch = new List<OrderEntity>();

        // Read from Table Storage
        await foreach (var entity in _sourceClient.QueryAsync<OrderEntity>())
        {
            batch.Add(entity);

            if (batch.Count >= batchSize)
            {
                await WriteBatchToCosmosAsync(batch);
                migratedCount += batch.Count;
                Console.WriteLine($"Migrated {migratedCount} entities...");
                batch.Clear();
            }
        }

        // Write remaining entities
        if (batch.Any())
        {
            await WriteBatchToCosmosAsync(batch);
            migratedCount += batch.Count;
        }

        Console.WriteLine($"Migration complete: {migratedCount} entities migrated");
    }

    private async Task WriteBatchToCosmosAsync(List<OrderEntity> batch)
    {
        var tasks = batch.Select(entity =>
            _destinationClient.AddEntityAsync(entity)
        );

        await Task.WhenAll(tasks);
    }
}

// Execute migration
var migrator = new TableStorageMigrator(tableStorageClient, cosmosDbClient);
await migrator.MigrateAllDataAsync();
```

#### Step 4: Azure Data Factory Migration (Large Datasets)

```json
{
  "name": "TableToCosmosDBPipeline",
  "properties": {
    "activities": [
      {
        "name": "CopyTableToCosmosDB",
        "type": "Copy",
        "inputs": [
          {
            "referenceName": "AzureTableStorageDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "CosmosDBTableDataset",
            "type": "DatasetReference"
          }
        ],
        "typeProperties": {
          "source": {
            "type": "AzureTableSource"
          },
          "sink": {
            "type": "CosmosDbTableSink",
            "writeBatchSize": 10000
          }
        }
      }
    ]
  }
}
```

**Deploy with Azure CLI**:

```bash
# Create linked services and datasets
az datafactory linked-service create \
  --factory-name myfactory \
  --resource-group myresourcegroup \
  --name TableStorageLinkedService \
  --properties @table-storage-linked-service.json

az datafactory linked-service create \
  --factory-name myfactory \
  --resource-group myresourcegroup \
  --name CosmosDBLinkedService \
  --properties @cosmosdb-linked-service.json

# Create and run pipeline
az datafactory pipeline create \
  --factory-name myfactory \
  --resource-group myresourcegroup \
  --name TableToCosmosDBPipeline \
  --pipeline @pipeline.json

az datafactory pipeline create-run \
  --factory-name myfactory \
  --resource-group myresourcegroup \
  --pipeline-name TableToCosmosDBPipeline
```

### Cosmos DB → Table Storage (Downgrade)

**Warning**: Migrating from Cosmos DB to Table Storage loses advanced features!

**Lost capabilities**:
- Global distribution
- Automatic indexing on all properties
- Change feed
- Consistency models
- TTL (time-to-live)

**Migration considerations**:

```csharp
public class CosmosToTableMigrator
{
    public async Task ValidateCompatibilityAsync(TableClient cosmosClient)
    {
        var warnings = new List<string>();

        await foreach (var entity in cosmosClient.QueryAsync<OrderEntity>())
        {
            // Check entity size (Cosmos: 2 MB, Table: 1 MB)
            var entitySize = CalculateEntitySize(entity);
            if (entitySize > 1 * 1024 * 1024)
            {
                warnings.Add($"Entity {entity.RowKey} exceeds 1 MB limit for Table Storage");
            }

            // Check property count (Cosmos: unlimited, Table: 252)
            var propertyCount = CountProperties(entity);
            if (propertyCount > 252)
            {
                warnings.Add($"Entity {entity.RowKey} has {propertyCount} properties (max 252 for Table Storage)");
            }

            // Check nested properties (not supported in Table Storage)
            if (HasNestedProperties(entity))
            {
                warnings.Add($"Entity {entity.RowKey} has nested properties (flatten required)");
            }
        }

        if (warnings.Any())
        {
            Console.WriteLine("⚠️ Compatibility warnings:");
            warnings.ForEach(w => Console.WriteLine($"  - {w}"));
        }
    }
}
```

## Feature-Specific Comparisons

### Consistency Models

#### Table Storage

```
Single consistency model:
- Strong consistency (single region)
- No tunable consistency levels
```

#### Cosmos DB

```
Five consistency models:

1. Strong: Linearizability guarantee (highest latency)
2. Bounded Staleness: Configurable lag (K versions or T seconds)
3. Session: Consistent within client session (default)
4. Consistent Prefix: Reads never see out-of-order writes
5. Eventual: Lowest latency, eventual convergence
```

```csharp
// Cosmos DB: Configure consistency level
var cosmosClient = new CosmosClient(
    accountEndpoint,
    authToken,
    new CosmosClientOptions
    {
        ConsistencyLevel = ConsistencyLevel.Session  // or Strong, Eventual, etc.
    }
);

// Override per-request
var requestOptions = new ItemRequestOptions
{
    ConsistencyLevel = ConsistencyLevel.Eventual  // Lower latency for non-critical reads
};
```

### Time-to-Live (TTL)

#### Table Storage

```csharp
// ❌ Not supported
// Must implement manual deletion logic
```

#### Cosmos DB

```csharp
// ✅ Automatic TTL
public class TemporaryEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public int ttl { get; set; }  // Auto-delete after N seconds
}

var entity = new TemporaryEntity
{
    PartitionKey = "session",
    RowKey = Guid.NewGuid().ToString(),
    ttl = 3600  // Auto-delete after 1 hour
};

await tableClient.AddEntityAsync(entity);
// Entity automatically deleted after 1 hour
```

### Analytical Store (HTAP)

#### Table Storage

```csharp
// ❌ Not supported
// Must export to separate analytical store
```

#### Cosmos DB

```csharp
// ✅ Analytical store for real-time analytics
// Enable analytical store on container
// Query with Azure Synapse Spark/SQL

// No ETL needed - automatic sync from transactional to analytical store
```

**Use case**: Real-time dashboards, BI reporting, ML training without impacting transactional workload

## Cost Optimization Strategies

### Table Storage Cost Optimization

```csharp
// 1. Batch operations (reduce transaction costs)
var batch = new List<TableTransactionAction>();
for (int i = 0; i < 100; i++)
{
    batch.Add(new TableTransactionAction(
        TableTransactionActionType.Add,
        new OrderEntity { PartitionKey = "customer-123", RowKey = $"order-{i}" }
    ));
}
await tableClient.SubmitTransactionAsync(batch);
// 100 inserts = 1 transaction (vs 100 transactions)

// 2. Reduce storage costs with compression
public class CompressedEntity : ITableEntity
{
    public string Data { get; set; } = "";  // Gzip-compressed JSON

    public void SetData(object obj)
    {
        var json = JsonSerializer.Serialize(obj);
        var bytes = Encoding.UTF8.GetBytes(json);
        using var ms = new MemoryStream();
        using (var gzip = new GZipStream(ms, CompressionMode.Compress))
        {
            gzip.Write(bytes);
        }
        Data = Convert.ToBase64String(ms.ToArray());
    }
}
```

### Cosmos DB Cost Optimization

```csharp
// 1. Use serverless for dev/test (pay per operation)
// No provisioned throughput, pay only for consumed RUs

// 2. Use autoscale for unpredictable workloads
var containerProperties = new ContainerProperties
{
    Id = "Orders",
    PartitionKeyPath = "/PartitionKey"
};

var throughputProperties = ThroughputProperties.CreateAutoscaleThroughput(4000);  // Max 4,000 RU/s
await database.CreateContainerIfNotExistsAsync(containerProperties, throughputProperties);

// 3. Optimize partition key design to avoid hot partitions
// Evenly distribute load across partitions

// 4. Use TTL to auto-delete old data
public class ExpiringEntity : ITableEntity
{
    public int ttl { get; set; } = 86400;  // 24 hours
}

// 5. Use regional endpoints for reads (lower latency, lower cost)
var cosmosClient = new CosmosClient(
    accountEndpoint,
    authToken,
    new CosmosClientOptions
    {
        ApplicationRegion = Regions.WestUS  // Read from nearest region
    }
);
```

## Decision Tree

```
┌─────────────────────────────────────────────────────┐
│ Choose Storage Service                              │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │ Budget < $500 │
         │   per month?  │
         └───┬───────┬───┘
             │       │
         Yes │       │ No
             │       │
             ▼       ▼
    ┌──────────┐  ┌──────────────┐
    │  Table   │  │ Need global  │
    │ Storage  │  │distribution? │
    └──────────┘  └───┬──────┬───┘
                      │      │
                  Yes │      │ No
                      │      │
                      ▼      ▼
              ┌──────────┐ ┌──────────────┐
              │ Cosmos   │ │ Need complex │
              │   DB     │ │   queries?   │
              └──────────┘ └───┬──────┬───┘
                               │      │
                           Yes │      │ No
                               │      │
                               ▼      ▼
                       ┌──────────┐ ┌──────────┐
                       │ Cosmos   │ │  Table   │
                       │   DB     │ │ Storage  │
                       └──────────┘ └──────────┘
```

## Hybrid Approach

**Strategy**: Use both services for different workloads

```csharp
public class HybridStorageService
{
    private readonly TableClient _tableStorageClient;  // Low-cost, high-volume
    private readonly TableClient _cosmosDbClient;      // Low-latency, critical

    public async Task CreateOrderAsync(OrderEntity order)
    {
        if (order.Amount > 10000)
        {
            // High-value orders: Cosmos DB (global distribution, low latency)
            await _cosmosDbClient.AddEntityAsync(order);
        }
        else
        {
            // Standard orders: Table Storage (cost-effective)
            await _tableStorageClient.AddEntityAsync(order);
        }
    }
}
```

**Use cases**:
- Hot data in Cosmos DB, cold data in Table Storage
- Critical transactions in Cosmos DB, logs in Table Storage
- Global customers in Cosmos DB, regional customers in Table Storage

## References

**Microsoft Learn Documentation**:
- [Choose between Table Storage and Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/table/introduction)
- [Cosmos DB Table API](https://learn.microsoft.com/azure/cosmos-db/table/table-api-overview)
- [Migration Guide](https://learn.microsoft.com/azure/cosmos-db/table/migrate-table-storage)
- [Pricing Comparison](https://azure.microsoft.com/pricing/details/cosmos-db/)
- [Table Storage Pricing](https://azure.microsoft.com/pricing/details/storage/tables/)

## Navigation

- **Previous**: `table-patterns.md`
- **Next**: `../05-queue-storage/queue-fundamentals.md`
- **Related**: `02-core-concepts/cost-optimization.md`

## See Also

- `table-fundamentals.md` - Table Storage basics
- `table-patterns.md` - Design patterns for both services
- `06-operations/governance.md` - Cost management strategies
