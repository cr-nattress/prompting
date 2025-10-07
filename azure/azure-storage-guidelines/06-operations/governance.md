# Azure Storage Governance and Operations

> **File Purpose**: Enterprise governance including naming conventions, tagging strategies, cost optimization, retention policies, and compliance
> **Prerequisites**: `../02-core-concepts/storage-accounts.md`, `observability.md`
> **Agent Use Case**: Reference when establishing governance policies, implementing cost controls, or ensuring compliance across storage infrastructure

## Quick Context

Effective governance ensures consistent naming, cost control, compliance, and operational excellence across Azure Storage resources. Proactive governance prevents technical debt and cost overruns.

**Key principle**: Establish naming conventions early; tag all resources for cost tracking; implement lifecycle policies for retention; monitor costs continuously; automate governance with Azure Policy.

## Naming Conventions

### Storage Account Naming

**Requirements**:
- 3-24 characters
- Lowercase letters and numbers only
- Globally unique across Azure

**Recommended pattern**:
```
st{environment}{workload}{region}{instance}
```

**Examples**:
```
stprodappeastus01  - Production, app workload, East US, instance 01
stdevdatawesteu02  - Development, data workload, West Europe, instance 02
stqalogscentral01  - QA, logs workload, Central US, instance 01
```

#### Naming Convention Code

```csharp
public static class StorageAccountNaming
{
    public static string Generate(
        string environment,
        string workload,
        string region,
        int instance)
    {
        // Validate inputs
        if (string.IsNullOrWhiteSpace(environment) ||
            string.IsNullOrWhiteSpace(workload) ||
            string.IsNullOrWhiteSpace(region))
        {
            throw new ArgumentException("Environment, workload, and region are required");
        }

        // Generate name
        var name = $"st{environment}{workload}{region}{instance:D2}";

        // Validate length (max 24 characters)
        if (name.Length > 24)
        {
            throw new ArgumentException($"Generated name '{name}' exceeds 24 characters");
        }

        // Ensure lowercase
        return name.ToLowerInvariant();
    }
}

// Usage
var storageAccountName = StorageAccountNaming.Generate(
    environment: "prod",
    workload: "app",
    region: "eastus",
    instance: 1
);
// Result: stprodappeastus01
```

### Container/Table/Queue Naming

**Requirements**:
- 3-63 characters
- Lowercase letters, numbers, and hyphens
- Cannot start or end with hyphen

**Recommended patterns**:

```
Containers:
{workload}-{datatype}-{environment}
  Examples:
  - app-documents-prod
  - web-images-dev
  - backup-databases-prod

Tables:
{Entity}{Purpose}
  Examples:
  - OrdersByCustomer
  - UserProfiles
  - ProductInventory

Queues:
{workload}-{action}-{environment}
  Examples:
  - app-processorders-prod
  - web-sendnotifications-dev
  - analytics-exportdata-qa
```

### Blob Naming

**Recommended patterns**:

```
Documents:
{year}/{month}/{day}/{filename}
  Example: 2025/10/06/invoice-12345.pdf

Logs:
{service}/{year}/{month}/{day}/{hour}/{logfile}
  Example: api/2025/10/06/14/app-20251006-140523.log

Backups:
{environment}/{service}/{timestamp}/{backup-type}.bak
  Example: prod/database/20251006-140000/full.bak

Media:
{category}/{subcategory}/{filename}
  Example: products/electronics/laptop-dell-xps13.jpg
```

## Tagging Strategy

### Required Tags

| Tag Key | Purpose | Example Values | Applied To |
|---------|---------|----------------|------------|
| **Environment** | Deployment stage | prod, dev, qa, staging | All resources |
| **CostCenter** | Billing allocation | finance, engineering, marketing | All resources |
| **Owner** | Responsible team/person | jane.doe@company.com, platform-team | All resources |
| **Application** | Application name | customer-portal, analytics-pipeline | All resources |
| **Criticality** | Business impact | critical, high, medium, low | All resources |

### Optional Tags

| Tag Key | Purpose | Example Values |
|---------|---------|----------------|
| **DataClassification** | Data sensitivity | public, internal, confidential, restricted |
| **ComplianceRequirement** | Regulatory requirements | gdpr, hipaa, pci-dss, sox |
| **BackupPolicy** | Backup retention | daily-30d, weekly-90d, monthly-7y |
| **LifecyclePolicy** | Data lifecycle | hot-30d-cool-90d-archive |
| **Project** | Project identifier | project-phoenix, migration-2025 |

### Tagging Implementation

#### Azure CLI

```bash
# Tag storage account
az storage account update \
  --name stprodappeastus01 \
  --resource-group rg-prod-app-eastus \
  --tags \
    Environment=prod \
    CostCenter=engineering \
    Owner=platform-team@company.com \
    Application=customer-portal \
    Criticality=high \
    DataClassification=confidential \
    ComplianceRequirement=gdpr \
    BackupPolicy=daily-30d

# Tag container (via blob service properties)
az storage container metadata update \
  --account-name stprodappeastus01 \
  --name app-documents-prod \
  --metadata \
    Environment=prod \
    DataClassification=confidential
```

#### Bicep

```bicep
param storageAccountName string
param location string = resourceGroup().location

@description('Common tags applied to all resources')
var commonTags = {
  Environment: 'prod'
  CostCenter: 'engineering'
  Owner: 'platform-team@company.com'
  Application: 'customer-portal'
  Criticality: 'high'
  ManagedBy: 'Terraform'
  CreatedDate: utcNow('yyyy-MM-dd')
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: commonTags
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

output storageAccountId string = storageAccount.id
output tags object = storageAccount.tags
```

#### C# SDK

```csharp
using Azure.ResourceManager;
using Azure.ResourceManager.Storage;

public async Task ApplyTagsAsync(
    string subscriptionId,
    string resourceGroupName,
    string storageAccountName,
    Dictionary<string, string> tags)
{
    var armClient = new ArmClient(new DefaultAzureCredential());

    var subscription = armClient.GetSubscriptionResource(
        new ResourceIdentifier($"/subscriptions/{subscriptionId}")
    );

    var resourceGroup = await subscription.GetResourceGroupAsync(resourceGroupName);

    var storageAccount = await resourceGroup.Value
        .GetStorageAccountAsync(storageAccountName);

    // Update tags
    var updateOptions = new Azure.ResourceManager.Storage.Models.StorageAccountPatch
    {
        Tags =
        {
            ["Environment"] = tags["Environment"],
            ["CostCenter"] = tags["CostCenter"],
            ["Owner"] = tags["Owner"],
            ["Application"] = tags["Application"],
            ["LastUpdated"] = DateTime.UtcNow.ToString("o")
        }
    };

    await storageAccount.Value.UpdateAsync(updateOptions);

    Console.WriteLine($"Applied {tags.Count} tags to {storageAccountName}");
}
```

## Cost Optimization

### Cost Breakdown

**Azure Storage costs** (simplified):

```
Total Cost = Storage Cost + Transaction Cost + Data Transfer Cost

Storage Cost = (GB stored) × ($/GB/month)
  - Hot: $0.0184/GB/month
  - Cool: $0.0100/GB/month
  - Cold: $0.0045/GB/month
  - Archive: $0.0020/GB/month

Transaction Cost = (Operations / 10,000) × ($/10K ops)
  - Write: $0.05/10K
  - Read: $0.004/10K
  - List: $0.05/10K

Data Transfer Cost = (GB egress) × ($/GB)
  - First 100 GB: Free
  - 100 GB - 10 TB: $0.087/GB
```

### Cost Optimization Strategies

#### 1. Lifecycle Management (Automated Tiering)

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "optimize-costs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToCold": {
              "daysAfterModificationGreaterThan": 90
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"]
        }
      }
    }
  ]
}
```

**Cost savings example** (1 TB data):
```
No lifecycle:
  - 1 TB Hot for 1 year = $221/year

With lifecycle:
  - 1 month Hot (30 days) = $18.40
  - 2 months Cool (60 days) = $16.67
  - 3 months Cold (90 days) = $11.25
  - 6 months Archive (180 days) = $9.86
  Total = $56.18/year

Savings: $164.82 (75% reduction)
```

#### 2. Reserved Capacity

```bash
# Purchase 1-year reserved capacity (10-20% discount)
az storage account create \
  --name stprodappeastus01 \
  --resource-group rg-prod-app-eastus \
  --sku Standard_LRS \
  --reservation-term OneYear \
  --reservation-scope Shared

# Cost comparison:
# Pay-as-you-go: $221/year for 1 TB Hot
# 1-year reserved: $199/year (10% savings)
# 3-year reserved: $177/year (20% savings)
```

#### 3. Redundancy Optimization

```bash
# Choose appropriate redundancy level
# LRS (cheapest): $0.0184/GB/month
# ZRS: $0.023/GB/month (25% more)
# GRS: $0.0368/GB/month (100% more)
# GZRS: $0.046/GB/month (150% more)

# For non-critical data, use LRS
az storage account create \
  --name stdevdataeastus01 \
  --sku Standard_LRS  # Cheapest option

# For critical data, use ZRS (regional redundancy)
az storage account create \
  --name stproddataeastus01 \
  --sku Standard_ZRS  # Balance of cost and durability
```

#### 4. Minimize Transactions

```csharp
// ❌ Bad: Individual operations (expensive)
for (int i = 0; i < 100; i++)
{
    await tableClient.AddEntityAsync(entities[i]);
}
// Cost: 100 write operations × $0.05/10K = $0.0005

// ✅ Good: Batch operations (cheaper)
var batch = entities.Take(100).Select(e =>
    new TableTransactionAction(TableTransactionActionType.Add, e)
).ToList();

await tableClient.SubmitTransactionAsync(batch);
// Cost: 1 write operation × $0.05/10K = $0.000005 (100x cheaper)
```

#### 5. Reduce Data Transfer Costs

```csharp
// Use CDN for frequently accessed content
// Direct blob access: $0.087/GB egress after 100 GB
// CDN access: $0.08/GB (slightly cheaper, plus edge caching)

// Enable compression to reduce transfer size
var uploadOptions = new BlobUploadOptions
{
    HttpHeaders = new BlobHttpHeaders
    {
        ContentEncoding = "gzip"  // Compress before upload
    }
};

// Result: 10 MB file compressed to 2 MB → 80% cost reduction
```

### Cost Monitoring

#### Azure Cost Management + KQL

```kql
// Monthly cost by storage account
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.STORAGE"
| summarize
    TotalCost = sum(CostInBillingCurrency)
    by ResourceId, bin(TimeGenerated, 1d)
| summarize MonthlyCost = sum(TotalCost) by ResourceId
| order by MonthlyCost desc
```

#### Cost Alerts

```bash
# Create budget alert ($500/month)
az consumption budget create \
  --budget-name storage-budget-prod \
  --amount 500 \
  --time-grain Monthly \
  --start-date 2025-10-01 \
  --end-date 2026-09-30 \
  --resource-group rg-prod-app-eastus \
  --notifications \
    operator=GreaterThan \
    threshold=80 \
    contactEmails='["admin@company.com"]'

# Cost alert triggers at $400 (80% of $500 budget)
```

## Retention Policies

### Data Retention Framework

| Data Type | Hot Tier | Cool Tier | Cold Tier | Archive Tier | Total Retention | Compliance |
|-----------|----------|-----------|-----------|--------------|-----------------|------------|
| **Application logs** | 7 days | - | - | - | 7 days | - |
| **Audit logs** | 30 days | 60 days | - | 7 years | 7+ years | SOX, GDPR |
| **Backups** | 7 days | 23 days | - | 11 months | 1 year | - |
| **Customer data** | 30 days | 60 days | 90 days | - | 180 days | GDPR |
| **Financial records** | 90 days | - | - | 7 years | 7+ years | SOX |
| **User-uploaded content** | 90 days | Indefinite | - | - | Indefinite | - |

### Retention Policy Implementation

#### Bicep Policy Template

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' existing = {
  parent: storageAccount
  name: 'default'
}

resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          enabled: true
          name: 'audit-logs-retention'
          type: 'Lifecycle'
          definition: {
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 30
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 90
                }
                delete: {
                  daysAfterModificationGreaterThan: 2555  // 7 years
                }
              }
            }
            filters: {
              blobTypes: ['appendBlob']
              prefixMatch: ['logs/audit/']
              blobIndexMatch: [
                {
                  name: 'retention'
                  op: '=='
                  value: 'compliance-7y'
                }
              ]
            }
          }
        }
        {
          enabled: true
          name: 'gdpr-right-to-delete'
          type: 'Lifecycle'
          definition: {
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 180
                }
              }
            }
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['customer-data/']
              blobIndexMatch: [
                {
                  name: 'gdpr-retain'
                  op: '=='
                  value: 'false'
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

## Compliance and Audit

### Compliance Controls

#### 1. Immutability Policies (WORM)

```bash
# Enable version-level immutability for compliance
az storage account blob-service-properties update \
  --account-name stprodcomplianceeastus01 \
  --resource-group rg-prod-compliance-eastus \
  --enable-versioning true

# Set container-level immutability policy (7-year retention)
az storage container immutability-policy create \
  --account-name stprodcomplianceeastus01 \
  --container-name financial-records \
  --period 2555 \
  --policy-mode Locked
```

#### 2. Audit Logging

```bash
# Enable storage analytics logging
az storage logging update \
  --account-name stprodappeastus01 \
  --services blob table queue \
  --log rwd \
  --retention 90 \
  --version 2.0

# r = read operations
# w = write operations
# d = delete operations
```

#### 3. Access Reviews

```kql
// Review all access to sensitive data
StorageBlobLogs
| where TimeGenerated > ago(30d)
| where Uri contains "confidential"
| summarize
    AccessCount = count(),
    UniqueUsers = dcount(CallerIpAddress),
    FirstAccess = min(TimeGenerated),
    LastAccess = max(TimeGenerated)
    by CallerIpAddress, AuthenticationType
| order by AccessCount desc
```

### Compliance Checklist

**GDPR Compliance**:
- [ ] Implement data retention policies (max 180 days for personal data)
- [ ] Enable soft delete (30-day recovery window)
- [ ] Implement right to be forgotten (delete on request)
- [ ] Enable audit logging for all access
- [ ] Tag personal data with `DataClassification=personal`
- [ ] Encrypt data at rest and in transit

**HIPAA Compliance**:
- [ ] Enable encryption at rest (AES-256)
- [ ] Require TLS 1.2+ for all connections
- [ ] Implement access controls (Azure RBAC)
- [ ] Enable audit logging (retain for 6 years)
- [ ] Implement immutability policies
- [ ] Regular access reviews (quarterly)

**SOX Compliance**:
- [ ] Financial data retention: 7 years minimum
- [ ] Immutability policies (WORM storage)
- [ ] Audit logging enabled
- [ ] Change tracking (versioning)
- [ ] Separation of duties (RBAC)

## Azure Policy Governance

### Policy Definitions

#### Enforce Naming Convention

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "field": "name",
          "notMatch": "st[a-z]{3,6}[a-z]{3,10}[a-z]{4,10}[0-9]{2}"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {},
  "metadata": {
    "displayName": "Enforce Storage Account Naming Convention",
    "description": "Storage account names must follow pattern: st{env}{workload}{region}{instance}"
  }
}
```

#### Require Specific Tags

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "anyOf": [
            {
              "field": "tags['Environment']",
              "exists": "false"
            },
            {
              "field": "tags['CostCenter']",
              "exists": "false"
            },
            {
              "field": "tags['Owner']",
              "exists": "false"
            }
          ]
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

#### Enforce Minimum TLS Version

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Storage/storageAccounts"
        },
        {
          "field": "Microsoft.Storage/storageAccounts/minimumTlsVersion",
          "notEquals": "TLS1_2"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

### Deploy Policies

```bash
# Create policy definition
az policy definition create \
  --name enforce-storage-naming \
  --display-name "Enforce Storage Account Naming Convention" \
  --description "Deny creation of storage accounts that don't follow naming convention" \
  --rules @policy-naming.json \
  --mode Indexed

# Assign policy to subscription
az policy assignment create \
  --name enforce-storage-naming-sub \
  --policy enforce-storage-naming \
  --scope /subscriptions/{subscription-id}

# Assign policy to resource group
az policy assignment create \
  --name enforce-storage-naming-rg \
  --policy enforce-storage-naming \
  --scope /subscriptions/{subscription-id}/resourceGroups/rg-prod-app-eastus
```

## Operational Excellence

### 1. Automated Backups

```bash
# Backup storage account configuration
az storage account show \
  --name stprodappeastus01 \
  --resource-group rg-prod-app-eastus \
  > backup-config-$(date +%Y%m%d).json

# Backup lifecycle policies
az storage account management-policy show \
  --account-name stprodappeastus01 \
  --resource-group rg-prod-app-eastus \
  > backup-lifecycle-$(date +%Y%m%d).json
```

### 2. Disaster Recovery Planning

```bash
# Enable GRS replication for critical storage
az storage account update \
  --name stprodcriticaleastus01 \
  --sku Standard_GZRS  # Geo-zone-redundant

# Test failover (RA-GRS only)
az storage account failover \
  --name stprodcriticaleastus01 \
  --resource-group rg-prod-critical-eastus \
  --no-wait
```

### 3. Health Monitoring

```kql
// Storage account health check
AzureMetrics
| where ResourceProvider == "MICROSOFT.STORAGE"
| where TimeGenerated > ago(1h)
| summarize
    AvgAvailability = avg(todouble(Average)),
    AvgLatency = avg(todouble(Average)),
    TotalErrors = countif(MetricName == "Transactions" and StatusCode >= 400)
    by ResourceId
| where AvgAvailability < 99.9 or AvgLatency > 100 or TotalErrors > 100
```

## References

**Microsoft Learn Documentation**:
- [Azure Storage Governance](https://learn.microsoft.com/azure/storage/common/storage-governance)
- [Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Tagging Strategy](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging)
- [Cost Optimization](https://learn.microsoft.com/azure/storage/common/storage-plan-manage-costs)
- [Azure Policy](https://learn.microsoft.com/azure/governance/policy/overview)
- [Compliance](https://learn.microsoft.com/azure/compliance/)

## Navigation

- **Previous**: `data-access-layer.md`
- **Next**: `../08-reference/troubleshooting.md`
- **Related**: `observability.md` - Cost monitoring

## See Also

- `../03-blob-storage/blob-lifecycle-management.md` - Automated tiering policies
- `observability.md` - Monitoring and alerting
- `../02-core-concepts/cost-optimization.md` - Cost management strategies
