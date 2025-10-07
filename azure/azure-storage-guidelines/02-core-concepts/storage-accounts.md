# Storage Account Architecture and Configuration

> **File Purpose**: Comprehensive guide to storage account types, redundancy options, performance tiers, and architectural decisions
> **Prerequisites**: Basic understanding of Azure Storage services
> **Agent Use Case**: Reference when designing storage architecture, selecting redundancy, or planning migration

## Quick Context

Azure Storage accounts are the foundational container for all Azure Storage services (Blob, Table, Queue, File). Selecting the right account type, redundancy option, and performance tier is critical for meeting availability, durability, performance, and cost requirements.

**Key principle**: Storage account architecture decisions (SKU, hierarchical namespace) are difficult or impossible to change post-creation. Design carefully upfront.

## Storage Account Types

### StorageV2 (General Purpose v2) - Recommended

**Use case**: Default choice for all modern workloads

```bash
# Create StorageV2 account
az storage account create \
  --name storagev2demo \
  --resource-group rg-storage \
  --location westus3 \
  --kind StorageV2 \
  --sku Standard_GZRS
```

**Features**:
- Supports all storage services (Blob, Table, Queue, File)
- Access tiers (Hot, Cool, Archive) for cost optimization
- Lifecycle management policies
- Advanced security features (CMK, immutability, soft delete)
- Best price/performance ratio

**When to use**:
- New projects (always start here)
- Production applications requiring cost optimization
- When you need blob tiering (Hot/Cool/Archive)

### BlobStorage (Legacy) - Deprecated

**DO NOT USE**: Use StorageV2 instead. BlobStorage accounts only support blob storage and lack lifecycle management features.

### BlockBlobStorage (Premium)

**Use case**: Ultra-low latency blob workloads

```bash
# Create Premium Block Blob account
az storage account create \
  --name premiumblobdemo \
  --resource-group rg-storage \
  --location westus3 \
  --kind BlockBlobStorage \
  --sku Premium_LRS
```

**Features**:
- Single-digit millisecond latency
- SSD-backed storage
- Higher transaction throughput (100,000+ IOPS)
- No tiering (Hot/Cool/Archive not available)

**When to use**:
- High transaction workloads (analytics, IoT telemetry)
- Low-latency requirements (< 10ms P99)
- Small objects with frequent reads/writes

**Trade-offs**:
- ❌ Higher cost per GB (3-5x Standard)
- ❌ Only LRS and ZRS redundancy (no geo-redundancy)
- ❌ No lifecycle policies or tiering
- ✅ Predictable low latency
- ✅ Higher IOPS and throughput

### FileStorage (Premium)

**Use case**: SMB/NFS file shares requiring low latency

```bash
# Create Premium File account
az storage account create \
  --name premiumfiledemo \
  --resource-group rg-storage \
  --location westus3 \
  --kind FileStorage \
  --sku Premium_LRS
```

**When to use**:
- Database workloads (SQL Server, MySQL)
- Containerized applications needing persistent volumes
- Lift-and-shift Windows/Linux file servers

**Not covered in this guide** (focus is Blob/Table).

## Performance Tiers Decision Matrix

| Workload | Account Type | SKU | Latency | Cost | Use Case |
|----------|-------------|-----|---------|------|----------|
| Web application media | StorageV2 | Standard_GZRS | 10-50ms | Low | Images, videos, static assets |
| IoT telemetry (high volume) | BlockBlobStorage | Premium_ZRS | <10ms | High | Real-time analytics, append logs |
| Document storage | StorageV2 | Standard_ZRS | 10-50ms | Low | PDF, Office files, archives |
| ML training datasets | StorageV2 + ADLS Gen2 | Standard_GRS | 10-50ms | Low | Large datasets, Spark/Databricks |
| Table Storage (key-value) | StorageV2 | Standard_GZRS | <15ms | Very Low | User profiles, metadata, sessions |
| Backup/Archive | StorageV2 | Standard_GRS | Hours (Archive tier) | Lowest | Compliance, long-term retention |

## Redundancy Options

### Local Redundancy (LRS)

**Configuration**: 3 synchronous copies within a single datacenter

```bash
az storage account create \
  --name storagelrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_LRS \
  --kind StorageV2
```

**Durability**: 11 nines (99.999999999%) annually
**Availability SLA**:
- Read requests: 99.9% (LRS)
- Write requests: 99.9% (LRS)

**Use cases**:
- Development/test environments
- Non-critical data with external backups
- Cost-sensitive workloads where regional failure is acceptable

**Failures protected against**:
- ✅ Disk/rack failures
- ❌ Datacenter/zone failures
- ❌ Regional disasters

**Trade-offs**:
- ✅ Lowest cost (baseline pricing)
- ✅ Available for Premium accounts
- ❌ No zone or regional protection
- ❌ Data loss if datacenter destroyed

### Zone-Redundant Storage (ZRS)

**Configuration**: 3 synchronous copies across 3 availability zones

```bash
az storage account create \
  --name storagezrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_ZRS \
  --kind StorageV2
```

**Durability**: 12 nines (99.9999999999%) annually
**Availability SLA**: 99.9%

**Use cases**:
- Production applications in regions with availability zones
- High availability without geo-replication cost
- When regional failover not required

**Failures protected against**:
- ✅ Disk/rack failures
- ✅ Datacenter/zone failures (within region)
- ❌ Regional disasters

**Trade-offs**:
- ✅ Higher availability than LRS (~10% cost increase)
- ✅ Automatic failover within region
- ✅ Available for Premium Block Blob
- ❌ No regional protection
- ❌ Only available in regions with availability zones

**Important**: Check region support with:
```bash
az provider show --namespace Microsoft.Storage \
  --query "resourceTypes[?resourceType=='storageAccounts'].zoneMappings"
```

### Geo-Redundant Storage (GRS)

**Configuration**: 3 copies in primary region (LRS) + 3 copies in paired secondary region (LRS)

```bash
az storage account create \
  --name storagegrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_GRS \
  --kind StorageV2
```

**Durability**: 16 nines (99.99999999999999%) annually
**Availability SLA**: 99.9% (primary region)

**Replication**: Asynchronous to secondary region (RPO typically < 15 minutes)

**Use cases**:
- Business-critical data requiring regional disaster recovery
- Compliance requirements for geo-redundancy
- When RPO of 15 minutes is acceptable

**Failures protected against**:
- ✅ Disk/rack failures
- ✅ Datacenter failures (within primary region)
- ✅ Regional disasters (failover to secondary)

**Trade-offs**:
- ✅ Survives regional outages
- ✅ 16 nines durability
- ❌ Higher cost (~2x LRS)
- ❌ No automatic failover (requires manual initiation)
- ❌ Secondary region is read-only by default (use RA-GRS for read access)
- ❌ Asynchronous replication (data loss possible during failover)

**Region pairs**: Azure pairs regions (e.g., West US 3 ↔ East US). See:
```bash
az account list-locations --query "[?metadata.regionType=='Physical'].{Name:name, Pair:metadata.pairedRegion[0].name}"
```

### Read-Access Geo-Redundant Storage (RA-GRS)

**Configuration**: GRS + read access to secondary region

```bash
az storage account create \
  --name storageragrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_RAGRS \
  --kind StorageV2
```

**Durability**: 16 nines (same as GRS)
**Availability SLA**:
- Primary reads: 99.9%
- Secondary reads: 99.9%

**Secondary endpoint**: `https://{accountname}-secondary.blob.core.windows.net`

**Use cases**:
- Applications requiring read access during regional outages
- Global read distribution (primary for writes, secondary for reads)
- Reporting/analytics from secondary region

**Code example** (C# with secondary endpoint):
```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

// Primary endpoint (read/write)
var primaryClient = new BlobServiceClient(
    new Uri("https://storageragrs.blob.core.windows.net"),
    new DefaultAzureCredential());

// Secondary endpoint (read-only)
var secondaryClient = new BlobServiceClient(
    new Uri("https://storageragrs-secondary.blob.core.windows.net"),
    new DefaultAzureCredential());

// Fallback pattern
BlobContainerClient GetContainerWithFallback(string containerName)
{
    try
    {
        var primary = primaryClient.GetBlobContainerClient(containerName);
        primary.GetProperties(); // Check accessibility
        return primary;
    }
    catch (Azure.RequestFailedException)
    {
        // Fallback to secondary
        return secondaryClient.GetBlobContainerClient(containerName);
    }
}
```

**Trade-offs**:
- ✅ Read availability during primary region outage
- ✅ Global read distribution
- ✅ Same durability as GRS
- ❌ Slightly higher cost than GRS (~5%)
- ❌ Secondary data may lag primary (check `LastSyncTime`)
- ❌ Secondary is read-only (writes always go to primary)

**Check replication status**:
```bash
az storage account show \
  --name storageragrs \
  --resource-group rg-storage \
  --query "{status:geoReplicationStats.status, lastSync:geoReplicationStats.lastSyncTime}"
```

### Geo-Zone-Redundant Storage (GZRS)

**Configuration**: 3 copies across zones in primary region + 3 copies in secondary region (LRS)

```bash
az storage account create \
  --name storagegzrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_GZRS \
  --kind StorageV2
```

**Durability**: 16 nines (99.99999999999999%) annually
**Availability SLA**: 99.99% (highest for standard tier)

**Use cases**:
- Mission-critical production workloads
- Maximum availability and durability requirements
- When both zone and regional failures must be tolerated

**Failures protected against**:
- ✅ Disk/rack failures
- ✅ Zone failures (within primary region)
- ✅ Regional disasters

**Trade-offs**:
- ✅ Highest availability/durability
- ✅ Automatic failover within zones
- ✅ Survives regional disasters
- ❌ Highest cost (~2.5x LRS)
- ❌ Manual failover required for regional disasters (unless RA-GZRS)

**Recommended for**:
- Financial services (high availability required)
- Healthcare systems (durability and compliance)
- E-commerce platforms (customer data protection)

### Read-Access Geo-Zone-Redundant Storage (RA-GZRS)

**Configuration**: GZRS + read access to secondary region

```bash
az storage account create \
  --name storagezrs \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_RAGZRS \
  --kind StorageV2
```

**Durability**: 16 nines
**Availability SLA**: 99.99% (primary), 99.99% (secondary reads)

**Use cases**:
- All GZRS use cases + global read distribution
- Highest availability for read workloads
- When primary region outage cannot impact read operations

**Trade-offs**:
- ✅ Maximum availability for reads
- ✅ Highest durability
- ❌ Highest cost (~2.6x LRS)

## Redundancy Selection Decision Matrix

| Requirement | Recommended SKU | Justification |
|-------------|----------------|---------------|
| Dev/test environment | Standard_LRS | Lowest cost, sufficient for non-production |
| Production (single region) | Standard_ZRS | Zone redundancy, automatic failover |
| Production (geo-redundant) | Standard_GZRS | Zone + region protection |
| Global read distribution | Standard_RAGZRS | Read from secondary region |
| Ultra-low latency | Premium_ZRS | SSD-backed, single-digit ms latency |
| Compliance (disaster recovery) | Standard_GRS or GZRS | Survives regional disasters |
| Cost-optimized production | Standard_ZRS | Best balance of cost and availability |

## Hierarchical Namespace (ADLS Gen2)

### What Is Hierarchical Namespace?

HNS enables Azure Data Lake Storage Gen2 features:
- True directory semantics (rename directory in O(1))
- POSIX-compliant ACLs (rwx permissions)
- Better performance for big data workloads (Spark, Databricks)

### Enabling HNS

**CRITICAL**: Cannot be disabled after enabling. Plan carefully.

```bash
# Create account with HNS enabled
az storage account create \
  --name storagehns \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_GZRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true
```

**Bicep example**:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  sku: { name: 'Standard_GZRS' }
  kind: 'StorageV2'
  properties: {
    isHnsEnabled: true  // Enable ADLS Gen2
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}
```

### When to Enable HNS

| Enable HNS If... | Keep HNS Disabled If... |
|------------------|------------------------|
| Using Databricks, Synapse, Spark | Simple blob storage (images, videos, documents) |
| Need directory-level ACLs | Using RBAC at container level is sufficient |
| Rename directory operations required | Flat namespace is adequate |
| Big data analytics workloads | Cost sensitivity (HNS has higher transaction costs) |
| HDFS compatibility needed | Using Azure Blob SDKs only |

### HNS Trade-offs

**Advantages**:
- ✅ O(1) directory rename (vs O(n) blob-by-blob copy)
- ✅ POSIX ACLs (read/write/execute per file/directory)
- ✅ Better performance for Hadoop/Spark workloads
- ✅ HDFS-compatible APIs

**Disadvantages**:
- ❌ Cannot disable after enabling
- ❌ Some Blob APIs unavailable (e.g., SetBlobTier for individual blobs)
- ❌ Higher transaction costs (especially list operations)
- ❌ Lifecycle policies work differently
- ❌ Premium Block Blob not supported with HNS

### HNS API Differences

**Without HNS** (standard Blob API):
```csharp
// Standard blob client
BlobClient blobClient = new BlobClient(connectionString, "container", "path/to/file.txt");
await blobClient.UploadAsync(stream);
```

**With HNS** (Data Lake API):
```csharp
using Azure.Storage.Files.DataLake;

// Data Lake client (HNS-enabled account)
DataLakeFileSystemClient fileSystemClient = new DataLakeFileSystemClient(
    new Uri("https://storagehns.dfs.core.windows.net/container"),
    new DefaultAzureCredential());

DataLakeFileClient fileClient = fileSystemClient.GetFileClient("path/to/file.txt");
await fileClient.UploadAsync(stream);

// Set ACLs (only available with HNS)
var acl = "user::rwx,group::r-x,other::---";
await fileClient.SetPermissionsAsync(permissions: acl);
```

### Migration to HNS

**Cannot convert existing accounts**. Must create new account and copy data.

```bash
# 1. Create new HNS-enabled account
az storage account create \
  --name storagehns \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_GZRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true

# 2. Use AzCopy to migrate data
azcopy copy \
  "https://sourceaccount.blob.core.windows.net/container/*" \
  "https://storagehns.dfs.core.windows.net/container/" \
  --recursive=true
```

**Migration checklist**:
- [ ] Update application code to use Data Lake APIs (`dfs.core.windows.net` endpoint)
- [ ] Update connection strings and DNS entries
- [ ] Re-configure RBAC and ACLs on new account
- [ ] Test all data access patterns
- [ ] Update lifecycle policies (syntax differs for HNS)
- [ ] Monitor transaction costs (may increase)

## Account Limits and Quotas

| Limit | Value | Notes |
|-------|-------|-------|
| Storage accounts per subscription per region | 250 | Soft limit, can request increase |
| Max storage account capacity | 5 PiB | Standard tier (default) |
| Max ingress per account (GRS/GZRS) | 20 Gbps | US regions |
| Max egress per account (GRS/GZRS) | 50 Gbps | US regions |
| Max IOPS (Standard) | 20,000 | Per storage account |
| Max IOPS (Premium Block Blob) | 100,000+ | Per storage account |
| Max blob size (Block Blob) | ~190.7 TiB | 50,000 blocks × 4000 MiB |
| Max blob size (Page Blob) | 8 TiB | VHD maximum |

**Check current limits**:
```bash
az storage account show-usage --location westus3
```

## Configuration Best Practices

### 1. Default to GZRS for Production

```bash
# Production storage account template
az storage account create \
  --name prodstorageacct \
  --resource-group rg-prod \
  --location westus3 \
  --sku Standard_GZRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --default-action Deny \
  --bypass AzureServices
```

**Justification**:
- Survives zone and regional failures
- 99.99% availability SLA
- Industry-standard for production
- Cost justified by availability benefits

### 2. Use ZRS for Single-Region Workloads

If geo-redundancy not required (e.g., data can be regenerated):

```bash
az storage account create \
  --name cachestorageacct \
  --resource-group rg-prod \
  --location westus3 \
  --sku Standard_ZRS \
  --kind StorageV2
```

**Use cases**:
- Cache or CDN origin data
- Logs (shipped to central repository)
- Temporary processing data

### 3. LRS for Development Only

```bash
# NEVER use in production
az storage account create \
  --name devstorageacct \
  --resource-group rg-dev \
  --location westus3 \
  --sku Standard_LRS \
  --kind StorageV2
```

### 4. Enable Soft Delete and Versioning

Regardless of redundancy, enable data protection:

```bash
# Soft delete (30-day retention)
az storage blob service-properties delete-policy update \
  --account-name prodstorageacct \
  --enable true \
  --days-retained 30

# Versioning (keeps all versions)
az storage account blob-service-properties update \
  --account-name prodstorageacct \
  --resource-group rg-prod \
  --enable-versioning true
```

**Cost impact**: Versioning increases storage costs. Use lifecycle policies to expire old versions.

## Migration Considerations

### Changing Redundancy (Supported)

**Supported conversions** (no downtime):

| From | To | Method |
|------|-----|--------|
| LRS | ZRS | `az storage account update` |
| LRS | GRS/GZRS | `az storage account update` |
| GRS | GZRS | `az storage account update` |
| ZRS | GZRS | `az storage account update` |

**Not supported**:
- Premium to Standard (or vice versa)
- HNS-enabled to HNS-disabled
- BlobStorage to StorageV2 (migrate data manually)

```bash
# Upgrade LRS to GZRS (no downtime)
az storage account update \
  --name storagelrs \
  --resource-group rg-storage \
  --sku Standard_GZRS
```

**Conversion time**: Typically completes within hours, but can take up to 72 hours.

**Check conversion status**:
```bash
az storage account show \
  --name storagelrs \
  --resource-group rg-storage \
  --query "{sku:sku.name, provisioningState:provisioningState}"
```

### Changing Account Kind (Not Supported)

**Cannot convert** BlobStorage → StorageV2 or Standard → Premium.

**Migration steps**:
1. Create new account with desired configuration
2. Use AzCopy to copy data
3. Update application connection strings
4. Delete old account

```bash
# 1. Create new account
az storage account create \
  --name newstoragev2 \
  --resource-group rg-storage \
  --location westus3 \
  --sku Standard_GZRS \
  --kind StorageV2

# 2. Copy data with AzCopy
azcopy sync \
  "https://oldaccount.blob.core.windows.net/" \
  "https://newstoragev2.blob.core.windows.net/" \
  --recursive
```

## Cost Optimization Strategies

### 1. Use Lifecycle Policies

Automatically tier blobs to Cool/Archive based on age:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "tier-to-cool",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          }
        }
      }
    }
  ]
}
```

See `03-blob-storage/blob-lifecycle-management.md` for detailed policies.

### 2. Right-Size Redundancy

**Don't over-provision**:
- Development: LRS (sufficient)
- Staging: ZRS (zone-redundant)
- Production: GZRS (geo-zone redundant)

**Cost comparison** (relative to LRS = 1.0x):
- LRS: 1.0x
- ZRS: 1.25x
- GRS: 2.0x
- GZRS: 2.5x
- RA-GZRS: 2.6x

### 3. Use Premium Sparingly

Premium Block Blob is 3-5x more expensive than Standard. Use only when:
- P99 latency < 10ms required
- Transaction rate > 20,000 IOPS
- Small objects with frequent access

## Monitoring and Alerts

### Key Metrics to Monitor

```bash
# View transaction metrics
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{name} \
  --metric Transactions \
  --interval PT1H
```

**Critical metrics**:
- **Availability**: Target 99.9%+ (depends on SKU SLA)
- **SuccessE2ELatency**: Monitor P95/P99
- **Ingress/Egress**: Track against limits
- **UsedCapacity**: Set alerts at 80% of quota

### Alert on Geo-Replication Lag

```bash
# Alert if LastSyncTime > 1 hour
az monitor metrics alert create \
  --name geo-replication-lag \
  --resource-group rg-storage \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{name} \
  --condition "avg GeoReplicationDataLag > 3600" \
  --description "Geo-replication lag exceeds 1 hour"
```

See `06-operations/observability.md` for comprehensive monitoring setup.

## Security Considerations

### 1. Deny Public Network Access

```bash
az storage account update \
  --name prodstorageacct \
  --resource-group rg-prod \
  --default-action Deny
```

Then configure private endpoints or firewall rules. See `02-core-concepts/networking-architecture.md`.

### 2. Enforce TLS 1.2+

```bash
az storage account update \
  --name prodstorageacct \
  --resource-group rg-prod \
  --min-tls-version TLS1_2
```

### 3. Disable Account Keys (Use Managed Identity)

```bash
# Disable shared key access (forces Azure AD authentication)
az storage account update \
  --name prodstorageacct \
  --resource-group rg-prod \
  --allow-shared-key-access false
```

See `02-core-concepts/security-model.md` for authentication strategies.

## Navigation

- **Previous**: `01-quick-start/local-development.md`
- **Next**: `security-model.md`
- **Up**: `00-overview.md`

## See Also

- `01-quick-start/provisioning.md` - Step-by-step provisioning
- `02-core-concepts/networking-architecture.md` - Private endpoints and network security
- `03-blob-storage/blob-lifecycle-management.md` - Lifecycle policies for cost optimization
- `06-operations/governance.md` - Naming conventions and tagging strategies

## References

[1] "Redundancy in Azure Storage" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/common/storage-redundancy
[2] "Storage account overview" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/common/storage-account-overview
[3] "Azure Data Lake Storage Gen2 hierarchical namespace" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-namespace
[4] "Azure Storage scalability and performance targets" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/common/scalability-targets-standard-account
[5] "Change the redundancy configuration" - Microsoft Learn - 2024-07 - https://learn.microsoft.com/azure/storage/common/redundancy-migration
