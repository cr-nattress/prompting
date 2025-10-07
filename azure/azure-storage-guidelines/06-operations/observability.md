# Azure Storage Observability and Monitoring

> **File Purpose**: Comprehensive guide to diagnostics, logging, metrics, Azure Monitor alerts, KQL queries, and request correlation for Azure Storage
> **Prerequisites**: `../02-core-concepts/storage-accounts.md`, any storage service guide
> **Agent Use Case**: Reference when implementing monitoring, troubleshooting performance issues, or setting up alerting for storage operations

## Quick Context

Effective observability is critical for production Azure Storage workloads. Azure provides rich diagnostic capabilities through metrics, logs, and distributed tracing.

**Key principle**: Enable diagnostic settings early; use KQL for deep analysis; set up proactive alerts for SLA breaches; correlate requests end-to-end for troubleshooting.

## Monitoring Architecture

### Azure Storage Telemetry Stack

```
┌─────────────────────────────────────────────────────┐
│  Azure Storage Services                             │
│  (Blob, Table, Queue, File)                         │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Emits
                 ▼
┌─────────────────────────────────────────────────────┐
│  Metrics + Logs + Traces                            │
│  - Platform metrics (automatic, free)               │
│  - Resource logs (requires diagnostic settings)     │
│  - Distributed traces (Application Insights)        │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Stored in
                 ▼
┌─────────────────────────────────────────────────────┐
│  Azure Monitor                                      │
│  - Metrics Database (93 days retention)             │
│  - Log Analytics (configurable retention)           │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Queried via
                 ▼
┌─────────────────────────────────────────────────────┐
│  Analysis Tools                                     │
│  - Azure Portal Metrics Explorer                    │
│  - Log Analytics (KQL)                              │
│  - Azure Monitor Alerts                             │
│  - Workbooks, Dashboards                            │
└─────────────────────────────────────────────────────┘
```

## Diagnostic Settings Configuration

### Enable Diagnostic Logging

#### Azure CLI

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group myresourcegroup \
  --workspace-name storage-logs

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group myresourcegroup \
  --workspace-name storage-logs \
  --query id --output tsv)

# Enable diagnostics for Blob service
az monitor diagnostic-settings create \
  --name blob-diagnostics \
  --resource /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
  --workspace $WORKSPACE_ID \
  --logs '[
    {
      "category": "StorageRead",
      "enabled": true,
      "retentionPolicy": {"enabled": true, "days": 30}
    },
    {
      "category": "StorageWrite",
      "enabled": true,
      "retentionPolicy": {"enabled": true, "days": 30}
    },
    {
      "category": "StorageDelete",
      "enabled": true,
      "retentionPolicy": {"enabled": true, "days": 30}
    }
  ]' \
  --metrics '[
    {
      "category": "Transaction",
      "enabled": true,
      "retentionPolicy": {"enabled": true, "days": 30}
    }
  ]'

# Enable for Table, Queue, File services similarly
az monitor diagnostic-settings create \
  --name table-diagnostics \
  --resource /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/tableServices/default \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "StorageRead", "enabled": true}, {"category": "StorageWrite", "enabled": true}, {"category": "StorageDelete", "enabled": true}]' \
  --metrics '[{"category": "Transaction", "enabled": true}]'
```

#### Bicep

```bicep
@description('Storage account name')
param storageAccountName string

@description('Log Analytics workspace ID')
param workspaceId string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' existing = {
  parent: storageAccount
  name: 'default'
}

resource blobDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'blob-diagnostics'
  scope: blobService
  properties: {
    workspaceId: workspaceId
    logs: [
      {
        category: 'StorageRead'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
      {
        category: 'StorageWrite'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
      {
        category: 'StorageDelete'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
    ]
    metrics: [
      {
        category: 'Transaction'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
    ]
  }
}

resource tableService 'Microsoft.Storage/storageAccounts/tableServices@2023-01-01' existing = {
  parent: storageAccount
  name: 'default'
}

resource tableDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'table-diagnostics'
  scope: tableService
  properties: {
    workspaceId: workspaceId
    logs: [
      {
        category: 'StorageRead'
        enabled: true
      }
      {
        category: 'StorageWrite'
        enabled: true
      }
      {
        category: 'StorageDelete'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'Transaction'
        enabled: true
      }
    ]
  }
}
```

### Log Categories

| Category | Operations Logged | Use Case |
|----------|------------------|----------|
| **StorageRead** | GetBlob, DownloadBlob, ListBlobs, QueryEntities | Read performance, access patterns |
| **StorageWrite** | PutBlob, UploadBlob, UpdateEntity, Enqueue | Write performance, throughput analysis |
| **StorageDelete** | DeleteBlob, DeleteEntity, DeleteMessage | Audit trail, retention compliance |

## Key Performance Indicators (KPIs)

### Platform Metrics (Automatic)

#### Availability Metrics

```kql
// Average availability for Blob service (last 24 hours)
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where ResourceId contains "blobServices"
| where MetricName == "Availability"
| where TimeGenerated > ago(24h)
| summarize AvgAvailability = avg(Average) by bin(TimeGenerated, 1h)
| render timechart
```

**SLA targets**:
- LRS/ZRS: 99.9% (read), 99.9% (write)
- GRS/GZRS: 99.9% (read), 99.9% (write), 99.99% (RA-GRS read)

#### Latency Metrics

```kql
// P50, P95, P99 latency for read operations
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where MetricName in ("SuccessE2ELatency", "SuccessServerLatency")
| where TimeGenerated > ago(1h)
| summarize
    P50 = percentile(Average, 50),
    P95 = percentile(Average, 95),
    P99 = percentile(Average, 99)
    by MetricName, bin(TimeGenerated, 5m)
| render timechart
```

**Latency targets**:
- Hot tier: <10ms (P99 server latency)
- Cool tier: <10ms (P99 server latency)
- Archive tier: Hours (rehydration)

#### Throughput Metrics

```kql
// Ingress and egress bandwidth
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where MetricName in ("Ingress", "Egress")
| where TimeGenerated > ago(1h)
| summarize TotalBytes = sum(Total) by MetricName, bin(TimeGenerated, 5m)
| extend TotalMB = TotalBytes / (1024.0 * 1024.0)
| render timechart
```

**Throughput limits**:
- LRS: 50 Gbps ingress, 100 Gbps egress
- GZRS: 25 Gbps ingress, 50 Gbps egress

### Transaction Metrics

```kql
// Transaction count by API operation
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where MetricName == "Transactions"
| where TimeGenerated > ago(1h)
| extend ApiName = tostring(parse_json(DimensionValue)["ApiName"])
| summarize TransactionCount = sum(Total) by ApiName, bin(TimeGenerated, 5m)
| render timechart
```

## Resource Logs Analysis

### Common KQL Queries

#### 1. Failed Requests Analysis

```kql
// Find all failed requests with error details
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where StatusCode >= 400
| summarize
    ErrorCount = count(),
    SampleUri = any(Uri),
    SampleError = any(StatusText)
    by StatusCode, OperationName
| order by ErrorCount desc
```

**Common error codes**:
- 404: Blob not found
- 409: Conflict (lease held, blob exists)
- 412: Precondition failed (ETag mismatch)
- 429: Throttling (too many requests)
- 500: Server error

#### 2. Slow Requests (P99 Latency)

```kql
// Identify slowest requests (P99)
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where StatusCode < 400  // Successful requests only
| extend E2ELatencyMs = DurationMs
| summarize
    P99Latency = percentile(E2ELatencyMs, 99),
    AvgLatency = avg(E2ELatencyMs),
    RequestCount = count()
    by OperationName
| where P99Latency > 100  // Requests slower than 100ms
| order by P99Latency desc
```

#### 3. Throttling Analysis

```kql
// Identify throttled requests (429 errors)
StorageBlobLogs
| where TimeGenerated > ago(24h)
| where StatusCode == 429
| summarize
    ThrottleCount = count(),
    UniqueCallers = dcount(CallerIpAddress)
    by bin(TimeGenerated, 5m), OperationName
| render timechart
```

**Throttling mitigation**:
- Implement exponential backoff
- Distribute load across partitions
- Increase provisioned throughput (Cosmos DB)
- Enable client-side retry policies

#### 4. Request Correlation (End-to-End Tracing)

```kql
// Trace entire request flow using CorrelationId
let correlationId = "your-correlation-id-here";
union StorageBlobLogs, StorageTableLogs, StorageQueueLogs
| where CorrelationId == correlationId
| project
    TimeGenerated,
    ServiceType = Type,
    OperationName,
    StatusCode,
    DurationMs,
    Uri,
    CallerIpAddress
| order by TimeGenerated asc
```

#### 5. Access Pattern Analysis

```kql
// Most accessed blobs (top 100)
StorageBlobLogs
| where TimeGenerated > ago(24h)
| where OperationName in ("GetBlob", "GetBlobProperties")
| extend BlobPath = extract(@"/([^?]+)", 1, Uri)
| summarize
    AccessCount = count(),
    UniqueCallers = dcount(CallerIpAddress)
    by BlobPath
| top 100 by AccessCount desc
```

#### 6. Cost Analysis by Operation

```kql
// Transaction costs by operation type
StorageBlobLogs
| where TimeGenerated > ago(30d)
| summarize
    TransactionCount = count()
    by OperationName
| extend
    // Pricing: Write ops $0.05/10K, Read ops $0.004/10K, List ops $0.05/10K
    Cost = case(
        OperationName in ("PutBlob", "PutBlock", "PutBlockList"), TransactionCount / 10000.0 * 0.05,
        OperationName in ("GetBlob", "GetBlobProperties"), TransactionCount / 10000.0 * 0.004,
        OperationName in ("ListBlobs"), TransactionCount / 10000.0 * 0.05,
        0.0
    )
| project OperationName, TransactionCount, EstimatedCost = round(Cost, 2)
| order by EstimatedCost desc
```

#### 7. Geographic Distribution

```kql
// Requests by geographic location
StorageBlobLogs
| where TimeGenerated > ago(24h)
| summarize
    RequestCount = count(),
    AvgLatency = avg(DurationMs)
    by CallerIpAddress
| extend Country = geo_info_from_ip_address(CallerIpAddress).country
| summarize
    TotalRequests = sum(RequestCount),
    AvgLatency = avg(AvgLatency)
    by Country
| order by TotalRequests desc
```

## Azure Monitor Alerts

### Alert Rules

#### 1. Availability Alert (SLA Breach)

```bash
# Create availability alert (< 99.9%)
az monitor metrics alert create \
  --name storage-availability-alert \
  --resource-group myresourcegroup \
  --scopes /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
  --condition "avg Availability < 99.9" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1 \
  --description "Blob storage availability below 99.9%" \
  --action email me@example.com
```

#### 2. Latency Alert (P95 > 100ms)

```bash
# Create latency alert
az monitor metrics alert create \
  --name storage-latency-alert \
  --resource-group myresourcegroup \
  --scopes /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
  --condition "avg SuccessE2ELatency > 100" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Blob storage E2E latency > 100ms" \
  --action email me@example.com
```

#### 3. Error Rate Alert (>1% Errors)

```kql
// Log Analytics alert query
StorageBlobLogs
| where TimeGenerated > ago(5m)
| summarize
    TotalRequests = count(),
    FailedRequests = countif(StatusCode >= 400)
| extend ErrorRate = (FailedRequests * 100.0) / TotalRequests
| where ErrorRate > 1.0
```

**Create alert**:

```bash
az monitor scheduled-query create \
  --name storage-error-rate-alert \
  --resource-group myresourcegroup \
  --scopes /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.OperationalInsights/workspaces/storage-logs \
  --condition "count 'Heartbeat' > 0" \
  --condition-query "StorageBlobLogs | where TimeGenerated > ago(5m) | summarize TotalRequests = count(), FailedRequests = countif(StatusCode >= 400) | extend ErrorRate = (FailedRequests * 100.0) / TotalRequests | where ErrorRate > 1.0" \
  --window-size 5m \
  --evaluation-frequency 5m \
  --severity 2 \
  --description "Error rate > 1%" \
  --action email me@example.com
```

#### 4. Throttling Alert (429 Errors)

```kql
// Throttling detection
StorageBlobLogs
| where TimeGenerated > ago(5m)
| where StatusCode == 429
| summarize ThrottleCount = count()
| where ThrottleCount > 10
```

### Action Groups

```bash
# Create action group for email and SMS
az monitor action-group create \
  --name storage-critical-alerts \
  --resource-group myresourcegroup \
  --short-name StorageCrit \
  --email-receiver \
    name=AdminEmail \
    email-address=admin@example.com \
  --sms-receiver \
    name=AdminSMS \
    country-code=1 \
    phone-number=5551234567

# Create action group for webhook (PagerDuty, Slack, etc.)
az monitor action-group create \
  --name storage-webhook-alerts \
  --resource-group myresourcegroup \
  --short-name StorageHook \
  --webhook-receiver \
    name=PagerDuty \
    service-uri=https://events.pagerduty.com/integration/{key}/enqueue
```

## Request Correlation and Distributed Tracing

### C# Client-Side Instrumentation

```csharp
using Azure.Storage.Blobs;
using Azure.Identity;
using System.Diagnostics;
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;

public class InstrumentedBlobService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly TelemetryClient _telemetryClient;

    public InstrumentedBlobService(
        string storageAccountUrl,
        TelemetryClient telemetryClient)
    {
        _blobServiceClient = new BlobServiceClient(
            new Uri(storageAccountUrl),
            new DefaultAzureCredential()
        );
        _telemetryClient = telemetryClient;
    }

    public async Task<string> UploadBlobWithTracingAsync(
        string containerName,
        string blobName,
        Stream content)
    {
        // Start operation tracking
        using var operation = _telemetryClient.StartOperation<DependencyTelemetry>("UploadBlob");
        operation.Telemetry.Type = "Azure Storage";
        operation.Telemetry.Target = containerName;
        operation.Telemetry.Data = blobName;

        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobClient = containerClient.GetBlobClient(blobName);

            // Custom properties for correlation
            var metadata = new Dictionary<string, string>
            {
                ["CorrelationId"] = Activity.Current?.Id ?? Guid.NewGuid().ToString(),
                ["UploadedBy"] = Environment.UserName,
                ["UploadTime"] = DateTime.UtcNow.ToString("o")
            };

            var uploadOptions = new Azure.Storage.Blobs.Models.BlobUploadOptions
            {
                Metadata = metadata
            };

            var uploadResponse = await blobClient.UploadAsync(content, uploadOptions);

            // Track success
            operation.Telemetry.Success = true;
            operation.Telemetry.ResultCode = "200";

            _telemetryClient.TrackEvent("BlobUploaded", new Dictionary<string, string>
            {
                ["ContainerName"] = containerName,
                ["BlobName"] = blobName,
                ["SizeBytes"] = content.Length.ToString(),
                ["ETag"] = uploadResponse.Value.ETag.ToString()
            });

            return uploadResponse.Value.ETag.ToString();
        }
        catch (Exception ex)
        {
            // Track failure
            operation.Telemetry.Success = false;
            operation.Telemetry.ResultCode = "500";

            _telemetryClient.TrackException(ex, new Dictionary<string, string>
            {
                ["ContainerName"] = containerName,
                ["BlobName"] = blobName
            });

            throw;
        }
    }
}

// Usage with Application Insights
var telemetryConfig = TelemetryConfiguration.CreateDefault();
telemetryConfig.InstrumentationKey = "your-app-insights-key";
var telemetryClient = new TelemetryClient(telemetryConfig);

var blobService = new InstrumentedBlobService(
    "https://mystorageaccount.blob.core.windows.net",
    telemetryClient
);

using var fileStream = File.OpenRead("document.pdf");
var eTag = await blobService.UploadBlobWithTracingAsync(
    "documents",
    "important-doc.pdf",
    fileStream
);
```

### Cross-Service Correlation

```kql
// Correlate Application Insights traces with Storage logs
let correlationId = "your-correlation-id";
union
    (
        // Application Insights dependencies
        dependencies
        | where cloud_RoleName == "MyApp"
        | where customDimensions.CorrelationId == correlationId
        | project
            TimeGenerated = timestamp,
            Source = "AppInsights",
            Operation = name,
            Duration = duration,
            Success = success
    ),
    (
        // Storage logs
        StorageBlobLogs
        | where CorrelationId == correlationId
        | project
            TimeGenerated,
            Source = "Storage",
            Operation = OperationName,
            Duration = DurationMs,
            Success = StatusCode < 400
    )
| order by TimeGenerated asc
```

## Performance Dashboards

### Bicep: Create Workbook

```bicep
resource storageWorkbook 'Microsoft.Insights/workbooks@2022-04-01' = {
  name: guid('storage-performance-workbook')
  location: resourceGroup().location
  kind: 'shared'
  properties: {
    displayName: 'Storage Performance Dashboard'
    category: 'storage'
    serializedData: '''
    {
      "version": "Notebook/1.0",
      "items": [
        {
          "type": 3,
          "content": {
            "version": "KqlItem/1.0",
            "query": "AzureMetrics\\n| where ResourceProvider == \\"MICROSOFT.STORAGE\\"\\n| where MetricName == \\"Availability\\"\\n| summarize avg(Average) by bin(TimeGenerated, 5m)\\n| render timechart",
            "size": 0,
            "title": "Availability Over Time"
          }
        },
        {
          "type": 3,
          "content": {
            "version": "KqlItem/1.0",
            "query": "StorageBlobLogs\\n| where StatusCode >= 400\\n| summarize count() by StatusCode, OperationName\\n| render barchart",
            "size": 0,
            "title": "Error Distribution"
          }
        }
      ]
    }
    '''
  }
}
```

## Real-Time Monitoring with CLI

### Live Metrics

```bash
# Monitor metrics in real-time (1-minute intervals)
while true; do
    az monitor metrics list \
      --resource /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
      --metric Availability SuccessE2ELatency Transactions \
      --interval PT1M \
      --aggregation Average \
      --output table
    sleep 60
done
```

### Live Log Streaming

```bash
# Stream logs in real-time
az monitor log-analytics query \
  --workspace storage-logs \
  --analytics-query "StorageBlobLogs | where TimeGenerated > ago(1m) | project TimeGenerated, OperationName, StatusCode, DurationMs, Uri" \
  --output table

# Repeat every 10 seconds
```

## Best Practices

### 1. Enable Diagnostic Settings Early

```bash
# Enable at storage account creation
az storage account create \
  --name mystorageaccount \
  --resource-group myresourcegroup \
  --sku Standard_LRS \
  --location eastus

# Immediately enable diagnostics (don't wait for issues)
az monitor diagnostic-settings create \
  --name initial-diagnostics \
  --resource /subscriptions/{sub-id}/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "StorageRead", "enabled": true}, {"category": "StorageWrite", "enabled": true}, {"category": "StorageDelete", "enabled": true}]'
```

### 2. Set Up Proactive Alerts

**Alert priority**:
1. **Critical (Severity 0)**: Availability < 99%
2. **High (Severity 1)**: Availability < 99.9%, Latency > 500ms
3. **Medium (Severity 2)**: Error rate > 5%, Throttling detected
4. **Low (Severity 3)**: Latency > 100ms, Warning thresholds

### 3. Implement Request Correlation

**Guidelines**:
- Always include `x-ms-client-request-id` header
- Log correlation IDs in application logs
- Use distributed tracing (Application Insights, OpenTelemetry)

### 4. Monitor Costs

```kql
// Monthly cost projection
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where MetricName == "Transactions"
| where TimeGenerated > ago(7d)
| summarize DailyTransactions = sum(Total) by bin(TimeGenerated, 1d)
| summarize AvgDailyTransactions = avg(DailyTransactions)
| extend
    MonthlyTransactions = AvgDailyTransactions * 30,
    MonthlyTransactionCost = (AvgDailyTransactions * 30) / 10000.0 * 0.0004
| project MonthlyTransactions, EstimatedMonthlyCost = round(MonthlyTransactionCost, 2)
```

### 5. Retention Policies

**Recommendations**:
- Hot data (1-7 days): Keep in Log Analytics
- Warm data (7-90 days): Keep in Log Analytics with compression
- Cold data (90-365 days): Archive to Blob Storage
- Compliance data (>1 year): Archive with immutability

## References

**Microsoft Learn Documentation**:
- [Azure Storage Monitoring](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage)
- [Storage Analytics Logging](https://learn.microsoft.com/azure/storage/common/storage-analytics-logging)
- [Azure Monitor Metrics](https://learn.microsoft.com/azure/azure-monitor/essentials/metrics-supported#microsoftstoragestorageaccounts)
- [KQL Quick Reference](https://learn.microsoft.com/azure/data-explorer/kql-quick-reference)
- [Application Insights](https://learn.microsoft.com/azure/azure-monitor/app/app-insights-overview)

## Navigation

- **Previous**: `../05-queue-storage/queue-fundamentals.md`
- **Next**: `performance-optimization.md`
- **Related**: `data-access-layer.md` - Client instrumentation

## See Also

- `performance-optimization.md` - Performance tuning based on metrics
- `governance.md` - Cost monitoring and optimization
- `../02-core-concepts/security.md` - Security monitoring and audit logs
