# Table Storage Design Patterns

> **File Purpose**: Advanced design patterns for Azure Table Storage including secondary indexes, hot partition avoidance, denormalization, and time-series data
> **Prerequisites**: `table-fundamentals.md`, `02-core-concepts/nosql-principles.md`
> **Agent Use Case**: Reference when designing table schemas, optimizing queries, and scaling table storage workloads

## Quick Context

Azure Table Storage is a NoSQL key-value store optimized for fast lookups via PartitionKey and RowKey. Proper schema design is critical for performance and scalability.

**Key principle**: Design schema around query patterns, not data structure. Denormalize aggressively. Avoid hot partitions through smart PartitionKey design.

## Table Storage Fundamentals Recap

### Key-Value Structure

```
┌──────────────────────────────────────────────────────┐
│  Table: Orders                                        │
├────────────────┬─────────────┬──────────────────────┤
│ PartitionKey   │ RowKey      │ Properties           │
├────────────────┼─────────────┼──────────────────────┤
│ customer-001   │ order-123   │ Amount, Date, Status │
│ customer-001   │ order-124   │ Amount, Date, Status │
│ customer-002   │ order-125   │ Amount, Date, Status │
└────────────────┴─────────────┴──────────────────────┘
```

**Performance characteristics**:
- **Point query** (PK + RK): <10ms, <$0.0004/10K ops
- **Partition scan** (PK only): 10-100ms, varies by partition size
- **Table scan** (no PK): Seconds to minutes, expensive

**Scalability limits**:
- Single partition: 2,000 entities/sec, 500 KB/sec
- Table: 20,000 entities/sec, 20 GB/sec
- Entity size: Max 1 MB, 252 properties

## Pattern 1: Secondary Indexes

### Problem: Multiple Query Patterns

**Requirement**: Query orders by customer ID AND by order date

**Naive approach** (table scan):
```csharp
// ❌ Bad: Table scan to find orders by date
var query = tableClient.QueryAsync<OrderEntity>(
    filter: $"OrderDate eq datetime'{date:o}'"
);
// Scans entire table - slow and expensive
```

### Solution: Index Table Pattern

**Strategy**: Create multiple tables, each optimized for specific query pattern

#### Primary Table (Query by Customer)

```
Table: OrdersByCustomer
┌──────────────────┬─────────────┬──────────────────────┐
│ PartitionKey     │ RowKey      │ Properties           │
├──────────────────┼─────────────┼──────────────────────┤
│ customer-001     │ order-123   │ Amount, Date, Status │
│ customer-001     │ order-124   │ Amount, Date, Status │
│ customer-002     │ order-125   │ Amount, Date, Status │
└──────────────────┴─────────────┴──────────────────────┘
```

#### Secondary Index Table (Query by Date)

```
Table: OrdersByDate
┌──────────────────┬──────────────┬──────────────────────┐
│ PartitionKey     │ RowKey       │ Properties           │
├──────────────────┼──────────────┼──────────────────────┤
│ 2025-10-06       │ order-123    │ CustomerId, Amount   │
│ 2025-10-06       │ order-125    │ CustomerId, Amount   │
│ 2025-10-07       │ order-124    │ CustomerId, Amount   │
└──────────────────┴──────────────┴──────────────────────┘
```

### C# Implementation

```csharp
using Azure.Data.Tables;
using System.Text.Json;

public class OrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string OrderId { get; set; } = "";
    public string CustomerId { get; set; } = "";
    public decimal Amount { get; set; }
    public DateTime OrderDate { get; set; }
    public string Status { get; set; } = "";
}

public class OrderService
{
    private readonly TableClient _ordersByCustomerTable;
    private readonly TableClient _ordersByDateTable;

    public OrderService(TableServiceClient tableServiceClient)
    {
        _ordersByCustomerTable = tableServiceClient.GetTableClient("OrdersByCustomer");
        _ordersByDateTable = tableServiceClient.GetTableClient("OrdersByDate");

        _ordersByCustomerTable.CreateIfNotExists();
        _ordersByDateTable.CreateIfNotExists();
    }

    public async Task CreateOrderAsync(
        string orderId,
        string customerId,
        decimal amount,
        DateTime orderDate,
        string status)
    {
        // Insert into primary table (query by customer)
        var primaryEntity = new OrderEntity
        {
            PartitionKey = customerId,  // Group by customer
            RowKey = orderId,
            OrderId = orderId,
            CustomerId = customerId,
            Amount = amount,
            OrderDate = orderDate,
            Status = status
        };

        await _ordersByCustomerTable.AddEntityAsync(primaryEntity);

        // Insert into secondary index (query by date)
        var secondaryEntity = new OrderEntity
        {
            PartitionKey = orderDate.ToString("yyyy-MM-dd"),  // Group by date
            RowKey = orderId,
            OrderId = orderId,
            CustomerId = customerId,
            Amount = amount,
            OrderDate = orderDate,
            Status = status
        };

        await _ordersByDateTable.AddEntityAsync(secondaryEntity);

        Console.WriteLine($"Order {orderId} indexed by customer and date");
    }

    // Query by customer (efficient partition query)
    public async Task<List<OrderEntity>> GetOrdersByCustomerAsync(string customerId)
    {
        var query = _ordersByCustomerTable.QueryAsync<OrderEntity>(
            filter: $"PartitionKey eq '{customerId}'"
        );

        var orders = new List<OrderEntity>();
        await foreach (var order in query)
        {
            orders.Add(order);
        }

        return orders;
    }

    // Query by date (efficient partition query)
    public async Task<List<OrderEntity>> GetOrdersByDateAsync(DateTime date)
    {
        var dateKey = date.ToString("yyyy-MM-dd");
        var query = _ordersByDateTable.QueryAsync<OrderEntity>(
            filter: $"PartitionKey eq '{dateKey}'"
        );

        var orders = new List<OrderEntity>();
        await foreach (var order in query)
        {
            orders.Add(order);
        }

        return orders;
    }

    // Update order (update both tables)
    public async Task UpdateOrderStatusAsync(
        string orderId,
        string customerId,
        DateTime orderDate,
        string newStatus)
    {
        // Update primary table
        var primaryEntity = await _ordersByCustomerTable.GetEntityAsync<OrderEntity>(
            customerId, orderId
        );
        primaryEntity.Value.Status = newStatus;
        await _ordersByCustomerTable.UpdateEntityAsync(primaryEntity.Value, ETag.All);

        // Update secondary index
        var dateKey = orderDate.ToString("yyyy-MM-dd");
        var secondaryEntity = await _ordersByDateTable.GetEntityAsync<OrderEntity>(
            dateKey, orderId
        );
        secondaryEntity.Value.Status = newStatus;
        await _ordersByDateTable.UpdateEntityAsync(secondaryEntity.Value, ETag.All);

        Console.WriteLine($"Order {orderId} status updated to {newStatus}");
    }

    // Delete order (delete from both tables)
    public async Task DeleteOrderAsync(
        string orderId,
        string customerId,
        DateTime orderDate)
    {
        // Delete from primary table
        await _ordersByCustomerTable.DeleteEntityAsync(customerId, orderId);

        // Delete from secondary index
        var dateKey = orderDate.ToString("yyyy-MM-dd");
        await _ordersByDateTable.DeleteEntityAsync(dateKey, orderId);

        Console.WriteLine($"Order {orderId} deleted from all indexes");
    }
}

// Usage
var orderService = new OrderService(tableServiceClient);

// Create order (indexes both tables)
await orderService.CreateOrderAsync(
    orderId: "order-001",
    customerId: "customer-123",
    amount: 99.99m,
    orderDate: DateTime.UtcNow,
    status: "pending"
);

// Query by customer (fast)
var customerOrders = await orderService.GetOrdersByCustomerAsync("customer-123");
Console.WriteLine($"Customer has {customerOrders.Count} orders");

// Query by date (fast)
var todayOrders = await orderService.GetOrdersByDateAsync(DateTime.UtcNow.Date);
Console.WriteLine($"Today: {todayOrders.Count} orders");
```

**Trade-offs**:
- ✅ Fast queries for both patterns
- ✅ No table scans
- ❌ 2x storage cost
- ❌ 2x write operations
- ❌ Consistency challenges (eventual consistency)

### Advanced: Batch Operations for Consistency

```csharp
public async Task CreateOrderAtomicallyAsync(
    string orderId,
    string customerId,
    decimal amount,
    DateTime orderDate,
    string status)
{
    var primaryEntity = new OrderEntity
    {
        PartitionKey = customerId,
        RowKey = orderId,
        OrderId = orderId,
        CustomerId = customerId,
        Amount = amount,
        OrderDate = orderDate,
        Status = status
    };

    var secondaryEntity = new OrderEntity
    {
        PartitionKey = orderDate.ToString("yyyy-MM-dd"),
        RowKey = orderId,
        OrderId = orderId,
        CustomerId = customerId,
        Amount = amount,
        OrderDate = orderDate,
        Status = status
    };

    try
    {
        // Write to primary table
        await _ordersByCustomerTable.AddEntityAsync(primaryEntity);

        try
        {
            // Write to secondary index
            await _ordersByDateTable.AddEntityAsync(secondaryEntity);
        }
        catch
        {
            // Rollback primary table insert
            await _ordersByCustomerTable.DeleteEntityAsync(customerId, orderId);
            throw;
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Failed to create order: {ex.Message}");
        throw;
    }
}
```

## Pattern 2: Hot Partition Avoidance

### Problem: Hot Partitions

**Scenario**: Logging system with single PartitionKey

```
Table: Logs (❌ Bad Design)
┌──────────────────┬─────────────────┬────────────────┐
│ PartitionKey     │ RowKey          │ Properties     │
├──────────────────┼─────────────────┼────────────────┤
│ application-logs │ 2025-10-06-001  │ Message, Level │
│ application-logs │ 2025-10-06-002  │ Message, Level │
│ application-logs │ 2025-10-06-003  │ Message, Level │
│ ...              │ ...             │ ...            │
└──────────────────┴─────────────────┴────────────────┘

Problem: All writes go to single partition → 2,000 ops/sec limit
```

### Solution 1: Hash-Based Sharding

**Strategy**: Distribute entities across multiple partitions using hash

```
Table: Logs (✅ Good Design)
┌──────────────────┬─────────────────┬────────────────┐
│ PartitionKey     │ RowKey          │ Properties     │
├──────────────────┼─────────────────┼────────────────┤
│ logs-shard-00    │ 2025-10-06-001  │ Message, Level │
│ logs-shard-01    │ 2025-10-06-002  │ Message, Level │
│ logs-shard-02    │ 2025-10-06-003  │ Message, Level │
│ ...              │ ...             │ ...            │
└──────────────────┴─────────────────┴────────────────┘

Benefit: Distributes load across partitions → 2,000 * N ops/sec
```

### C# Implementation

```csharp
public class LogEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string Message { get; set; } = "";
    public string Level { get; set; } = "";
    public DateTime LogTime { get; set; }
}

public class ShardedLogService
{
    private readonly TableClient _tableClient;
    private const int SHARD_COUNT = 10;  // 10 shards = 20,000 ops/sec

    public ShardedLogService(TableServiceClient tableServiceClient)
    {
        _tableClient = tableServiceClient.GetTableClient("Logs");
        _tableClient.CreateIfNotExists();
    }

    private string GetShardKey(string logId)
    {
        // Consistent hash to distribute across shards
        var hashCode = Math.Abs(logId.GetHashCode());
        var shardIndex = hashCode % SHARD_COUNT;
        return $"logs-shard-{shardIndex:D2}";
    }

    public async Task LogAsync(string logId, string message, string level)
    {
        var entity = new LogEntity
        {
            PartitionKey = GetShardKey(logId),  // Distribute across shards
            RowKey = logId,
            Message = message,
            Level = level,
            LogTime = DateTime.UtcNow
        };

        await _tableClient.AddEntityAsync(entity);
    }

    // Query across all shards
    public async Task<List<LogEntity>> GetAllLogsAsync()
    {
        var allLogs = new List<LogEntity>();

        // Query each shard
        for (int i = 0; i < SHARD_COUNT; i++)
        {
            var shardKey = $"logs-shard-{i:D2}";
            var query = _tableClient.QueryAsync<LogEntity>(
                filter: $"PartitionKey eq '{shardKey}'"
            );

            await foreach (var log in query)
            {
                allLogs.Add(log);
            }
        }

        return allLogs.OrderByDescending(l => l.LogTime).ToList();
    }

    // Query specific log by ID (know the shard)
    public async Task<LogEntity?> GetLogByIdAsync(string logId)
    {
        var shardKey = GetShardKey(logId);

        try
        {
            var response = await _tableClient.GetEntityAsync<LogEntity>(shardKey, logId);
            return response.Value;
        }
        catch (Azure.RequestFailedException ex) when (ex.Status == 404)
        {
            return null;
        }
    }
}

// Usage
var logService = new ShardedLogService(tableServiceClient);

// High-throughput logging (distributed across shards)
for (int i = 0; i < 10000; i++)
{
    await logService.LogAsync(
        logId: $"log-{Guid.NewGuid()}",
        message: $"Request processed: {i}",
        level: "INFO"
    );
}

Console.WriteLine("10,000 logs written across 10 shards");
```

### Solution 2: Time-Based Sharding

**Strategy**: Use time buckets as PartitionKey

```csharp
public class TimeShardedLogService
{
    private readonly TableClient _tableClient;

    private string GetTimeShardKey(DateTime logTime)
    {
        // 1-hour buckets (24 partitions per day)
        return $"logs-{logTime:yyyy-MM-dd-HH}";
    }

    public async Task LogAsync(string message, string level)
    {
        var logTime = DateTime.UtcNow;
        var entity = new LogEntity
        {
            PartitionKey = GetTimeShardKey(logTime),  // Hour bucket
            RowKey = Guid.NewGuid().ToString(),
            Message = message,
            Level = level,
            LogTime = logTime
        };

        await _tableClient.AddEntityAsync(entity);
    }

    // Query logs for specific hour
    public async Task<List<LogEntity>> GetLogsForHourAsync(DateTime hour)
    {
        var shardKey = GetTimeShardKey(hour);
        var query = _tableClient.QueryAsync<LogEntity>(
            filter: $"PartitionKey eq '{shardKey}'"
        );

        var logs = new List<LogEntity>();
        await foreach (var log in query)
        {
            logs.Add(log);
        }

        return logs;
    }

    // Query logs for date range
    public async Task<List<LogEntity>> GetLogsForRangeAsync(
        DateTime start,
        DateTime end)
    {
        var allLogs = new List<LogEntity>();

        // Iterate through each hour in range
        for (var hour = start; hour <= end; hour = hour.AddHours(1))
        {
            var hourLogs = await GetLogsForHourAsync(hour);
            allLogs.AddRange(hourLogs);
        }

        return allLogs.OrderByDescending(l => l.LogTime).ToList();
    }
}

// Usage
var logService = new TimeShardedLogService(tableServiceClient);

// High-throughput logging (auto-sharded by hour)
await logService.LogAsync("Application started", "INFO");
await logService.LogAsync("User login failed", "ERROR");

// Query specific hour (efficient)
var logsThisHour = await logService.GetLogsForHourAsync(DateTime.UtcNow);

// Query date range (queries only relevant partitions)
var todayLogs = await logService.GetLogsForRangeAsync(
    DateTime.UtcNow.Date,
    DateTime.UtcNow
);
```

**Partition strategy comparison**:

| Strategy | Partitions | Query Pattern | Use Case |
|----------|-----------|---------------|----------|
| **Hash-based** | Fixed (10-100) | Random access by ID | General purpose, high throughput |
| **Time-based** | Time buckets | Time-range queries | Logs, time-series, event data |
| **Composite** | Hash + Time | Both patterns | Complex workloads |

## Pattern 3: Denormalization

### Problem: Joins Don't Exist

**Relational model** (❌ Not possible in Table Storage):
```sql
SELECT o.*, c.Name, c.Email
FROM Orders o
JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderId = '123'
```

### Solution: Embed Related Data

**Strategy**: Store customer data directly in order entity

```csharp
public class DenormalizedOrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    // Order data
    public string OrderId { get; set; } = "";
    public decimal Amount { get; set; }
    public DateTime OrderDate { get; set; }
    public string Status { get; set; } = "";

    // ✅ Denormalized customer data (no join needed)
    public string CustomerId { get; set; } = "";
    public string CustomerName { get; set; } = "";
    public string CustomerEmail { get; set; } = "";
    public string CustomerPhone { get; set; } = "";

    // ✅ Denormalized product data
    public string ProductId { get; set; } = "";
    public string ProductName { get; set; } = "";
    public decimal ProductPrice { get; set; }
}

public class DenormalizedOrderService
{
    private readonly TableClient _ordersTable;
    private readonly TableClient _customersTable;

    public async Task CreateOrderAsync(
        string orderId,
        string customerId,
        string productId,
        decimal amount)
    {
        // Fetch customer data
        var customer = await _customersTable.GetEntityAsync<CustomerEntity>(
            customerId, customerId
        );

        // Fetch product data
        var product = await GetProductAsync(productId);

        // Create denormalized order (contains all data)
        var order = new DenormalizedOrderEntity
        {
            PartitionKey = customerId,
            RowKey = orderId,
            OrderId = orderId,
            Amount = amount,
            OrderDate = DateTime.UtcNow,
            Status = "pending",

            // Copy customer data (denormalization)
            CustomerId = customerId,
            CustomerName = customer.Value.Name,
            CustomerEmail = customer.Value.Email,
            CustomerPhone = customer.Value.Phone,

            // Copy product data
            ProductId = productId,
            ProductName = product.Name,
            ProductPrice = product.Price
        };

        await _ordersTable.AddEntityAsync(order);

        Console.WriteLine("Order created with denormalized data - no joins needed");
    }

    // Single query returns all data (no joins)
    public async Task<DenormalizedOrderEntity> GetOrderAsync(
        string customerId,
        string orderId)
    {
        var response = await _ordersTable.GetEntityAsync<DenormalizedOrderEntity>(
            customerId, orderId
        );

        // All data available in single entity
        Console.WriteLine($"Order: {response.Value.OrderId}");
        Console.WriteLine($"Customer: {response.Value.CustomerName}");
        Console.WriteLine($"Product: {response.Value.ProductName}");

        return response.Value;
    }
}
```

**Trade-offs**:
- ✅ Single query for all data (no joins)
- ✅ Fast reads (<10ms)
- ❌ Data duplication (higher storage cost)
- ❌ Stale data if customer/product changes
- ❌ Update complexity

### Handling Stale Data

**Strategy**: Update denormalized data when source changes

```csharp
public class CustomerService
{
    private readonly TableClient _customersTable;
    private readonly TableClient _ordersTable;

    public async Task UpdateCustomerEmailAsync(string customerId, string newEmail)
    {
        // Update customer record
        var customer = await _customersTable.GetEntityAsync<CustomerEntity>(
            customerId, customerId
        );
        customer.Value.Email = newEmail;
        await _customersTable.UpdateEntityAsync(customer.Value, ETag.All);

        // Update all orders with denormalized customer data
        var ordersQuery = _ordersTable.QueryAsync<DenormalizedOrderEntity>(
            filter: $"PartitionKey eq '{customerId}'"
        );

        await foreach (var order in ordersQuery)
        {
            order.CustomerEmail = newEmail;
            await _ordersTable.UpdateEntityAsync(order, ETag.All);
        }

        Console.WriteLine($"Updated customer email in customer record and {ordersQuery} orders");
    }
}
```

**Alternative: Hybrid approach** (store only frequently accessed fields):

```csharp
public class HybridOrderEntity : ITableEntity
{
    // Frequently accessed (denormalized)
    public string CustomerName { get; set; } = "";

    // Rarely accessed (reference only, fetch on demand)
    public string CustomerId { get; set; } = "";
    // Fetch customer details only when needed
}
```

## Pattern 4: Time-Series Data

### Problem: Efficient Time-Range Queries

**Requirement**: Store IoT sensor data, query by device and time range

### Solution: Inverted Timestamp RowKey

**Strategy**: Use inverted timestamp for reverse chronological order

```csharp
public class SensorDataEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";  // DeviceId
    public string RowKey { get; set; } = "";        // Inverted timestamp
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public double Temperature { get; set; }
    public double Humidity { get; set; }
    public DateTime ReadingTime { get; set; }
}

public class SensorDataService
{
    private readonly TableClient _tableClient;

    private string GetInvertedTimestamp(DateTime time)
    {
        // Invert timestamp: DateTime.MaxValue - time
        // Ensures most recent entries appear first
        var ticks = DateTime.MaxValue.Ticks - time.Ticks;
        return $"{ticks:D19}";  // Zero-padded for lexicographic sorting
    }

    public async Task RecordSensorDataAsync(
        string deviceId,
        double temperature,
        double humidity)
    {
        var readingTime = DateTime.UtcNow;
        var entity = new SensorDataEntity
        {
            PartitionKey = deviceId,  // Group by device
            RowKey = GetInvertedTimestamp(readingTime),  // Most recent first
            Temperature = temperature,
            Humidity = humidity,
            ReadingTime = readingTime
        };

        await _tableClient.AddEntityAsync(entity);
    }

    // Get latest N readings for device (efficient, no sorting needed)
    public async Task<List<SensorDataEntity>> GetLatestReadingsAsync(
        string deviceId,
        int count = 10)
    {
        var query = _tableClient.QueryAsync<SensorDataEntity>(
            filter: $"PartitionKey eq '{deviceId}'",
            maxPerPage: count
        );

        var readings = new List<SensorDataEntity>();
        await foreach (var reading in query)
        {
            readings.Add(reading);
            if (readings.Count >= count)
                break;
        }

        return readings;  // Already in reverse chronological order
    }

    // Get readings in time range
    public async Task<List<SensorDataEntity>> GetReadingsInRangeAsync(
        string deviceId,
        DateTime start,
        DateTime end)
    {
        // Inverted timestamps (end comes before start lexicographically)
        var endRowKey = GetInvertedTimestamp(end);
        var startRowKey = GetInvertedTimestamp(start);

        var query = _tableClient.QueryAsync<SensorDataEntity>(
            filter: $"PartitionKey eq '{deviceId}' and " +
                   $"RowKey ge '{endRowKey}' and RowKey le '{startRowKey}'"
        );

        var readings = new List<SensorDataEntity>();
        await foreach (var reading in query)
        {
            readings.Add(reading);
        }

        return readings;
    }
}

// Usage
var sensorService = new SensorDataService(tableServiceClient);

// Record sensor data
for (int i = 0; i < 100; i++)
{
    await sensorService.RecordSensorDataAsync(
        deviceId: "device-001",
        temperature: 20.0 + (i * 0.1),
        humidity: 50.0 + (i * 0.2)
    );
    await Task.Delay(100);
}

// Get latest 10 readings (efficient, no sorting)
var latestReadings = await sensorService.GetLatestReadingsAsync("device-001", 10);
Console.WriteLine($"Latest reading: {latestReadings[0].Temperature}°C at {latestReadings[0].ReadingTime}");

// Get readings for last hour
var oneHourAgo = DateTime.UtcNow.AddHours(-1);
var recentReadings = await sensorService.GetReadingsInRangeAsync(
    "device-001",
    oneHourAgo,
    DateTime.UtcNow
);
Console.WriteLine($"Readings in last hour: {recentReadings.Count}");
```

**Why inverted timestamp?**:
- RowKey is sorted lexicographically (ascending)
- Inverted timestamp makes latest entries appear first
- No need to sort after query
- Efficient for "latest N" queries

### Time-Series with Bucketing

**Strategy**: Combine date buckets (PartitionKey) with inverted timestamp (RowKey)

```csharp
public class BucketedSensorDataService
{
    private readonly TableClient _tableClient;

    private string GetPartitionKey(string deviceId, DateTime time)
    {
        // Daily buckets: device-001-2025-10-06
        return $"{deviceId}-{time:yyyy-MM-dd}";
    }

    private string GetInvertedTimestamp(DateTime time)
    {
        var ticks = DateTime.MaxValue.Ticks - time.Ticks;
        return $"{ticks:D19}";
    }

    public async Task RecordSensorDataAsync(
        string deviceId,
        double temperature,
        double humidity)
    {
        var readingTime = DateTime.UtcNow;
        var entity = new SensorDataEntity
        {
            PartitionKey = GetPartitionKey(deviceId, readingTime),
            RowKey = GetInvertedTimestamp(readingTime),
            Temperature = temperature,
            Humidity = humidity,
            ReadingTime = readingTime
        };

        await _tableClient.AddEntityAsync(entity);
    }

    // Get today's readings for device
    public async Task<List<SensorDataEntity>> GetTodaysReadingsAsync(string deviceId)
    {
        var partitionKey = GetPartitionKey(deviceId, DateTime.UtcNow);
        var query = _tableClient.QueryAsync<SensorDataEntity>(
            filter: $"PartitionKey eq '{partitionKey}'"
        );

        var readings = new List<SensorDataEntity>();
        await foreach (var reading in query)
        {
            readings.Add(reading);
        }

        return readings;
    }

    // Get readings across multiple days
    public async Task<List<SensorDataEntity>> GetReadingsForDateRangeAsync(
        string deviceId,
        DateTime startDate,
        DateTime endDate)
    {
        var allReadings = new List<SensorDataEntity>();

        // Query each day's partition
        for (var date = startDate.Date; date <= endDate.Date; date = date.AddDays(1))
        {
            var partitionKey = GetPartitionKey(deviceId, date);
            var query = _tableClient.QueryAsync<SensorDataEntity>(
                filter: $"PartitionKey eq '{partitionKey}'"
            );

            await foreach (var reading in query)
            {
                if (reading.ReadingTime >= startDate && reading.ReadingTime <= endDate)
                {
                    allReadings.Add(reading);
                }
            }
        }

        return allReadings.OrderByDescending(r => r.ReadingTime).ToList();
    }
}
```

**Benefits**:
- Partitions don't grow indefinitely (daily limit)
- Efficient queries for recent data
- Old partitions can be archived/deleted
- Scales to millions of devices

## Pattern 5: Composite Keys

### Problem: Multi-Dimensional Queries

**Requirement**: Query orders by status AND customer

### Solution: Composite PartitionKey

```csharp
public class CompositeKeyOrderEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";  // status|customerId
    public string RowKey { get; set; } = "";        // orderId
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string OrderId { get; set; } = "";
    public string CustomerId { get; set; } = "";
    public string Status { get; set; } = "";
    public decimal Amount { get; set; }
}

public class CompositeKeyOrderService
{
    private readonly TableClient _tableClient;

    private string GetCompositeKey(string status, string customerId)
    {
        return $"{status}|{customerId}";
    }

    public async Task CreateOrderAsync(
        string orderId,
        string customerId,
        string status,
        decimal amount)
    {
        var entity = new CompositeKeyOrderEntity
        {
            PartitionKey = GetCompositeKey(status, customerId),
            RowKey = orderId,
            OrderId = orderId,
            CustomerId = customerId,
            Status = status,
            Amount = amount
        };

        await _tableClient.AddEntityAsync(entity);
    }

    // Query pending orders for specific customer (efficient)
    public async Task<List<CompositeKeyOrderEntity>> GetPendingOrdersAsync(
        string customerId)
    {
        var partitionKey = GetCompositeKey("pending", customerId);
        var query = _tableClient.QueryAsync<CompositeKeyOrderEntity>(
            filter: $"PartitionKey eq '{partitionKey}'"
        );

        var orders = new List<CompositeKeyOrderEntity>();
        await foreach (var order in query)
        {
            orders.Add(order);
        }

        return orders;
    }

    // Update status (requires re-inserting with new PartitionKey)
    public async Task UpdateOrderStatusAsync(
        string orderId,
        string customerId,
        string oldStatus,
        string newStatus)
    {
        // Delete old entity
        var oldPartitionKey = GetCompositeKey(oldStatus, customerId);
        var oldEntity = await _tableClient.GetEntityAsync<CompositeKeyOrderEntity>(
            oldPartitionKey, orderId
        );

        await _tableClient.DeleteEntityAsync(oldPartitionKey, orderId);

        // Insert with new PartitionKey
        oldEntity.Value.PartitionKey = GetCompositeKey(newStatus, customerId);
        oldEntity.Value.Status = newStatus;
        oldEntity.Value.ETag = default;  // Reset ETag for new insert

        await _tableClient.AddEntityAsync(oldEntity.Value);

        Console.WriteLine($"Order {orderId} moved from {oldStatus} to {newStatus}");
    }
}
```

## Pattern 6: Large Entities (>1 MB)

### Problem: Entity Size Limit

**Azure Table Storage limit**: 1 MB per entity, 252 properties

### Solution: Split Large Entities

```csharp
public class LargeDocumentEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";  // documentId|chunk-0
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string DocumentId { get; set; } = "";
    public int ChunkIndex { get; set; }
    public int TotalChunks { get; set; }
    public string ChunkData { get; set; } = "";  // Base64 encoded
}

public class LargeDocumentService
{
    private readonly TableClient _tableClient;
    private const int CHUNK_SIZE = 900 * 1024;  // 900 KB (leave room for metadata)

    public async Task UploadLargeDocumentAsync(
        string documentId,
        string customerId,
        byte[] documentData)
    {
        var totalChunks = (int)Math.Ceiling((double)documentData.Length / CHUNK_SIZE);

        for (int i = 0; i < totalChunks; i++)
        {
            var offset = i * CHUNK_SIZE;
            var length = Math.Min(CHUNK_SIZE, documentData.Length - offset);
            var chunk = new byte[length];
            Array.Copy(documentData, offset, chunk, 0, length);

            var entity = new LargeDocumentEntity
            {
                PartitionKey = customerId,
                RowKey = $"{documentId}|chunk-{i:D4}",
                DocumentId = documentId,
                ChunkIndex = i,
                TotalChunks = totalChunks,
                ChunkData = Convert.ToBase64String(chunk)
            };

            await _tableClient.AddEntityAsync(entity);
        }

        Console.WriteLine($"Uploaded {totalChunks} chunks for document {documentId}");
    }

    public async Task<byte[]> DownloadLargeDocumentAsync(
        string customerId,
        string documentId)
    {
        // Query all chunks for document
        var query = _tableClient.QueryAsync<LargeDocumentEntity>(
            filter: $"PartitionKey eq '{customerId}' and DocumentId eq '{documentId}'"
        );

        var chunks = new List<LargeDocumentEntity>();
        await foreach (var chunk in query)
        {
            chunks.Add(chunk);
        }

        // Sort by chunk index
        chunks = chunks.OrderBy(c => c.ChunkIndex).ToList();

        // Reconstruct document
        using var ms = new MemoryStream();
        foreach (var chunk in chunks)
        {
            var chunkData = Convert.FromBase64String(chunk.ChunkData);
            ms.Write(chunkData, 0, chunkData.Length);
        }

        return ms.ToArray();
    }
}

// Usage
var docService = new LargeDocumentService(tableServiceClient);

// Upload 5 MB document (split into chunks)
var largeDocument = new byte[5 * 1024 * 1024];  // 5 MB
new Random().NextBytes(largeDocument);

await docService.UploadLargeDocumentAsync(
    documentId: "doc-001",
    customerId: "customer-123",
    documentData: largeDocument
);

// Download and reconstruct
var downloaded = await docService.DownloadLargeDocumentAsync(
    customerId: "customer-123",
    documentId: "doc-001"
);

Console.WriteLine($"Downloaded {downloaded.Length} bytes");
```

**Alternative**: Store large data in Blob Storage, reference in Table Storage

```csharp
public class DocumentEntity : ITableEntity
{
    public string PartitionKey { get; set; } = "";
    public string RowKey { get; set; } = "";
    public DateTimeOffset? Timestamp { get; set; }
    public ETag ETag { get; set; }

    public string DocumentId { get; set; } = "";
    public string BlobUrl { get; set; } = "";  // Reference to Blob Storage
    public long FileSize { get; set; }
    public string ContentType { get; set; } = "";
}
```

## Performance Best Practices

### 1. Partition Design Checklist

- [ ] PartitionKey distributes load evenly (no hot partitions)
- [ ] Single partition stays under 2,000 ops/sec
- [ ] Single partition stays under 20 GB (soft limit)
- [ ] Query patterns match partition boundaries
- [ ] Consider time-based sharding for time-series data

### 2. Query Optimization

```csharp
// ✅ Good: Point query (PK + RK)
var entity = await tableClient.GetEntityAsync<OrderEntity>(
    partitionKey: "customer-123",
    rowKey: "order-456"
);
// Latency: <10ms, Cost: $0.0004/10K ops

// ✅ Good: Partition query (PK only)
var query = tableClient.QueryAsync<OrderEntity>(
    filter: $"PartitionKey eq 'customer-123'"
);
// Latency: 10-100ms depending on partition size

// ❌ Bad: Table scan (no PK)
var query = tableClient.QueryAsync<OrderEntity>(
    filter: $"Status eq 'pending'"
);
// Latency: Seconds to minutes, scans entire table
```

### 3. Batch Operations

```csharp
// Batch insert (same partition, up to 100 entities)
var batch = new List<TableTransactionAction>();

for (int i = 0; i < 100; i++)
{
    var entity = new OrderEntity
    {
        PartitionKey = "customer-123",  // Must be same partition
        RowKey = $"order-{i}",
        Amount = 99.99m,
        OrderDate = DateTime.UtcNow,
        Status = "pending"
    };

    batch.Add(new TableTransactionAction(TableTransactionActionType.Add, entity));
}

// Execute as atomic transaction
var response = await tableClient.SubmitTransactionAsync(batch);

Console.WriteLine($"Batch inserted {batch.Count} entities");
```

**Batch limitations**:
- Max 100 entities per batch
- All entities must have same PartitionKey
- Atomic (all succeed or all fail)
- Max batch size: 4 MB

## References

**Microsoft Learn Documentation**:
- [Table Storage Design Patterns](https://learn.microsoft.com/azure/storage/tables/table-storage-design-patterns)
- [Designing for Scale](https://learn.microsoft.com/azure/storage/tables/table-storage-design-for-query)
- [Performance Targets](https://learn.microsoft.com/azure/storage/tables/scalability-targets)
- [Partitioning Best Practices](https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning)

## Navigation

- **Previous**: `table-fundamentals.md`
- **Next**: `table-vs-cosmos.md`
- **Related**: `02-core-concepts/nosql-principles.md`

## See Also

- `table-fundamentals.md` - Table Storage basics
- `table-vs-cosmos.md` - When to use Table vs Cosmos DB
- `06-operations/performance-optimization.md` - General performance tuning
