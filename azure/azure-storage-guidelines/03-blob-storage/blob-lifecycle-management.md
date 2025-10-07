# Blob Lifecycle Management

> **File Purpose**: Comprehensive guide to lifecycle policies, automated tiering, soft delete, versioning, and data retention strategies
> **Prerequisites**: `blob-fundamentals.md`, `02-core-concepts/storage-accounts.md`
> **Agent Use Case**: Reference when implementing automated data lifecycle, cost optimization, and compliance retention policies

## Quick Context

Blob lifecycle management automates data transitions between access tiers and deletion based on age, access patterns, or custom rules. This reduces storage costs while maintaining compliance with retention policies.

**Key principle**: Automate tier transitions to minimize manual intervention; use versioning + soft delete for data protection; implement retention policies for compliance.

## Lifecycle Management Overview

### What is Lifecycle Management?

**Lifecycle management** is a rule-based policy engine that automatically:
- Transitions blobs between access tiers (Hot → Cool → Cold → Archive)
- Deletes blobs after specified retention periods
- Manages blob versions and snapshots
- Applies actions based on last modification time or last access time

### Cost Optimization Impact

**Example savings** (1 TB data, 90-day retention):

| Strategy | Storage Cost/Month | Notes |
|----------|-------------------|-------|
| **All Hot** | $18.40 | No lifecycle management |
| **Manual tiering** | $12.20 | Error-prone, operational overhead |
| **Automated lifecycle** | $8.30 | Hot (30d) → Cool (60d) → Archive |
| **Savings** | **55%** | Automated, policy-driven |

## Lifecycle Policy Structure

### Policy Components

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "rule-name",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": { /* Actions for current blob version */ },
          "snapshot": { /* Actions for snapshots */ },
          "version": { /* Actions for non-current versions */ }
        },
        "filters": {
          "blobTypes": ["blockBlob", "appendBlob"],
          "prefixMatch": ["container/prefix"],
          "blobIndexMatch": [{"name": "tag", "op": "==", "value": "value"}]
        }
      }
    }
  ]
}
```

### Action Types

| Action | Description | Applies To | Trigger |
|--------|-------------|------------|---------|
| **tierToCool** | Move to Cool tier | Block blobs | Days since modification/access |
| **tierToCold** | Move to Cold tier | Block blobs | Days since modification/access |
| **tierToArchive** | Move to Archive tier | Block blobs | Days since modification/access |
| **tierToHot** | Move to Hot tier | Block blobs | Days since modification/access |
| **delete** | Delete blob | All types | Days since modification/creation |
| **enableAutoTierToHotFromCool** | Auto-promote on access | Block blobs | On first access |

### Filter Types

| Filter | Purpose | Example |
|--------|---------|---------|
| **blobTypes** | Target specific blob types | `["blockBlob", "appendBlob"]` |
| **prefixMatch** | Target blobs by path prefix | `["logs/", "backups/"]` |
| **blobIndexMatch** | Target blobs by index tags | `[{"name": "status", "op": "==", "value": "archived"}]` |

## Basic Lifecycle Policies

### Example 1: Simple Age-Based Tiering

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "tier-by-age",
      "type": "Lifecycle",
      "definition": {
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
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    }
  ]
}
```

**Timeline**:
```
Day 0    →   Day 30   →   Day 90    →   Day 365
Upload       Cool         Archive       Delete
(Hot)        tier         tier
```

### Example 2: Log Retention Policy

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "application-logs-retention",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 7
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 30
            },
            "delete": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["appendBlob"],
          "prefixMatch": ["logs/application/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "audit-logs-long-term",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 1
            },
            "delete": {
              "daysAfterModificationGreaterThan": 2555
            }
          }
        },
        "filters": {
          "blobTypes": ["appendBlob"],
          "prefixMatch": ["logs/audit/"]
        }
      }
    }
  ]
}
```

**Why this works**:
- Application logs: Short retention (90 days), fast transition to Archive
- Audit logs: 7-year retention (compliance), immediate Archive

### Example 3: Backup Retention (GFS Strategy)

**Grandfather-Father-Son** backup retention:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "daily-backups",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 1
            },
            "delete": {
              "daysAfterModificationGreaterThan": 7
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/daily/"],
          "blobIndexMatch": [
            {"name": "backupType", "op": "==", "value": "daily"}
          ]
        }
      }
    },
    {
      "enabled": true,
      "name": "weekly-backups",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 7
            },
            "delete": {
              "daysAfterModificationGreaterThan": 28
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/weekly/"],
          "blobIndexMatch": [
            {"name": "backupType", "op": "==", "value": "weekly"}
          ]
        }
      }
    },
    {
      "enabled": true,
      "name": "monthly-backups",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 1
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/monthly/"],
          "blobIndexMatch": [
            {"name": "backupType", "op": "==", "value": "monthly"}
          ]
        }
      }
    }
  ]
}
```

**Retention summary**:
- Daily: 7 days (Hot → Cool → Delete)
- Weekly: 28 days (Hot → Archive → Delete)
- Monthly: 365 days (Hot → Archive → Delete)

## Advanced Lifecycle Policies

### Example 4: Last Access Time-Based Tiering

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "tier-on-last-access",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterLastAccessTimeGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterLastAccessTimeGreaterThan": 90
            },
            "enableAutoTierToHotFromCool": {}
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["media/"]
        }
      }
    }
  ]
}
```

**How it works**:
1. Blob not accessed for 30 days → Cool tier
2. Blob not accessed for 90 days → Archive tier
3. On next access from Cool → Auto-promote to Hot

**Enable last access tracking**:

```bash
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-last-access-tracking true \
  --resource-group myresourcegroup
```

**Cost consideration**: Last access tracking incurs read transaction costs (~$0.0004 per 10K reads).

### Example 5: Versioning + Lifecycle Management

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "current-version-lifecycle",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "old-versions-cleanup",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "version": {
            "tierToCool": {
              "daysAfterCreationGreaterThan": 7
            },
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 30
            },
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    }
  ]
}
```

**Version lifecycle**:
- Current version: Hot (30 days) → Cool
- Old versions: Hot (7 days) → Cool (30 days) → Archive (90 days) → Delete

### Example 6: Snapshot Management

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "snapshot-retention",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "snapshot": {
            "tierToCool": {
              "daysAfterCreationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 90
            },
            "delete": {
              "daysAfterCreationGreaterThan": 180
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["database-backups/"]
        }
      }
    }
  ]
}
```

## Deploying Lifecycle Policies

### Azure CLI

```bash
# Create policy file
cat > lifecycle-policy.json <<'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "move-old-data-to-archive",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["data/"]
        }
      }
    }
  ]
}
EOF

# Apply policy
az storage account management-policy create \
  --account-name mystorageaccount \
  --policy @lifecycle-policy.json \
  --resource-group myresourcegroup

# View current policy
az storage account management-policy show \
  --account-name mystorageaccount \
  --resource-group myresourcegroup

# Delete policy
az storage account management-policy delete \
  --account-name mystorageaccount \
  --resource-group myresourcegroup
```

### Bicep Deployment

```bicep
@description('Storage account name')
param storageAccountName string

@description('Resource location')
param location string = resourceGroup().location

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}

resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          enabled: true
          name: 'tier-old-data'
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
                  daysAfterModificationGreaterThan: 365
                }
              }
              version: {
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
              snapshot: {
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
            }
            filters: {
              blobTypes: [
                'blockBlob'
              ]
              prefixMatch: [
                'data/'
              ]
            }
          }
        }
        {
          enabled: true
          name: 'delete-old-logs'
          type: 'Lifecycle'
          definition: {
            actions: {
              baseBlob: {
                delete: {
                  daysAfterModificationGreaterThan: 7
                }
              }
            }
            filters: {
              blobTypes: [
                'appendBlob'
              ]
              prefixMatch: [
                'logs/'
              ]
            }
          }
        }
      ]
    }
  }
}

output storageAccountId string = storageAccount.id
output lifecyclePolicyId string = lifecyclePolicy.id
```

**Deploy**:

```bash
az deployment group create \
  --resource-group myresourcegroup \
  --template-file lifecycle-policy.bicep \
  --parameters storageAccountName=mystorageaccount
```

### PowerShell

```powershell
# Define policy rules
$rules = @(
    @{
        enabled = $true
        name = "tier-old-data"
        type = "Lifecycle"
        definition = @{
            actions = @{
                baseBlob = @{
                    tierToCool = @{
                        daysAfterModificationGreaterThan = 30
                    }
                    tierToArchive = @{
                        daysAfterModificationGreaterThan = 90
                    }
                }
            }
            filters = @{
                blobTypes = @("blockBlob")
                prefixMatch = @("data/")
            }
        }
    }
)

# Create policy object
$policy = @{
    rules = $rules
}

# Apply policy
Set-AzStorageAccountManagementPolicy `
    -ResourceGroupName "myresourcegroup" `
    -AccountName "mystorageaccount" `
    -Policy ($policy | ConvertTo-Json -Depth 10)
```

## Soft Delete

### What is Soft Delete?

**Soft delete** provides a recovery mechanism for accidentally deleted blobs and blob versions. Deleted data is retained for a specified period before permanent deletion.

**Benefits**:
- Recover from accidental deletions
- Protection against ransomware (if immutability enabled)
- Compliance requirements for data retention

### Soft Delete Types

| Type | Protects | Retention | Use Case |
|------|----------|-----------|----------|
| **Blob soft delete** | Individual blobs | 1-365 days | Recover deleted files |
| **Container soft delete** | Entire containers | 1-365 days | Recover deleted containers |
| **Versioning** | Blob overwrites | Lifecycle policy | Recover previous versions |

### Enable Soft Delete

#### Azure CLI

```bash
# Enable blob soft delete (7-day retention)
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-delete-retention true \
  --delete-retention-days 7 \
  --resource-group myresourcegroup

# Enable container soft delete (14-day retention)
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-container-delete-retention true \
  --container-delete-retention-days 14 \
  --resource-group myresourcegroup

# View soft delete settings
az storage account blob-service-properties show \
  --account-name mystorageaccount \
  --resource-group myresourcegroup \
  --query "{blobRetention: deleteRetentionPolicy, containerRetention: containerDeleteRetentionPolicy}"
```

#### Bicep

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 14
    }
    isVersioningEnabled: true
    changeFeed: {
      enabled: true
      retentionInDays: 90
    }
  }
}
```

### C# Implementation: Work with Soft-Deleted Blobs

#### List Soft-Deleted Blobs

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

var containerClient = blobServiceClient.GetBlobContainerClient("documents");

// List including soft-deleted blobs
await foreach (var blobItem in containerClient.GetBlobsAsync(
    states: BlobStates.Deleted))
{
    Console.WriteLine($"Deleted blob: {blobItem.Name}");
    Console.WriteLine($"  Deleted on: {blobItem.Properties.DeletedOn}");
    Console.WriteLine($"  Remaining retention: {blobItem.Properties.RemainingRetentionDays} days");
}
```

#### Undelete Soft-Deleted Blob

```csharp
var blobClient = containerClient.GetBlobClient("important-document.pdf");

try
{
    // Undelete blob
    await blobClient.UndeleteAsync();
    Console.WriteLine("Blob restored successfully");
}
catch (RequestFailedException ex)
{
    Console.WriteLine($"Failed to restore: {ex.Message}");
}
```

#### Undelete All Soft-Deleted Blobs in Container

```csharp
public async Task UndeleteAllBlobsAsync(BlobContainerClient containerClient)
{
    var deletedBlobs = containerClient.GetBlobsAsync(states: BlobStates.Deleted);

    await foreach (var blobItem in deletedBlobs)
    {
        var blobClient = containerClient.GetBlobClient(blobItem.Name);

        try
        {
            await blobClient.UndeleteAsync();
            Console.WriteLine($"Restored: {blobItem.Name}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Failed to restore {blobItem.Name}: {ex.Message}");
        }
    }
}
```

### Soft Delete + Lifecycle Management

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "cleanup-soft-deleted-blobs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 90
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

**Timeline**:
```
Day 0       Day 7        Day 90       Day 97
Delete  →   Soft-delete  Lifecycle    Permanent
            expires      triggers     deletion
            (recoverable)(if still deleted)
```

## Blob Versioning

### Enable Versioning

```bash
# Enable versioning at storage account level
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-versioning true \
  --resource-group myresourcegroup
```

### Versioning Behavior

**Every write creates a new version**:

```
Upload v1    →  Upload v2    →  Upload v3
ETag: 0x8D1     ETag: 0x8D2     ETag: 0x8D3
Version: v1     Version: v2     Version: v3
(current)       (current)       (current)
                v1 (old)        v2 (old)
                                v1 (old)
```

### C# Implementation: Work with Versions

#### List All Versions

```csharp
await foreach (var blobItem in containerClient.GetBlobsAsync(
    states: BlobStates.Version))
{
    Console.WriteLine($"Blob: {blobItem.Name}");
    Console.WriteLine($"  Version: {blobItem.VersionId}");
    Console.WriteLine($"  Is current: {blobItem.IsLatestVersion}");
    Console.WriteLine($"  Created: {blobItem.Properties.CreatedOn}");
}
```

#### Download Specific Version

```csharp
var blobClient = containerClient.GetBlobClient("document.pdf");

// Get list of versions
var versions = new List<string>();
await foreach (var blobItem in containerClient.GetBlobsAsync(
    states: BlobStates.Version,
    prefix: "document.pdf"))
{
    versions.Add(blobItem.VersionId);
}

// Download version from 2 versions ago
if (versions.Count >= 2)
{
    var oldVersionId = versions[versions.Count - 2];
    var versionedBlob = blobClient.WithVersion(oldVersionId);

    var downloadResponse = await versionedBlob.DownloadContentAsync();
    var content = downloadResponse.Value.Content;

    Console.WriteLine($"Downloaded version: {oldVersionId}");
}
```

#### Restore Previous Version

```csharp
public async Task RestoreVersionAsync(
    BlobClient blobClient,
    string versionId)
{
    // Download specific version
    var versionedBlob = blobClient.WithVersion(versionId);
    var downloadResponse = await versionedBlob.DownloadContentAsync();

    // Upload as current version
    await blobClient.UploadAsync(downloadResponse.Value.Content, overwrite: true);

    Console.WriteLine($"Restored version {versionId} as current version");
}
```

### Versioning + Lifecycle Management

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "version-lifecycle",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "version": {
            "tierToCool": {
              "daysAfterCreationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 90
            },
            "delete": {
              "daysAfterCreationGreaterThan": 180,
              "daysAfterLastTierChangeGreaterThan": 7
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    }
  ]
}
```

**Version retention strategy**:
- Keep 5 most recent versions in Hot (manual cleanup)
- Older versions: Cool (30d) → Archive (90d) → Delete (180d)

## Immutability Policies (WORM)

### What is Immutability?

**WORM** (Write Once, Read Many) ensures data cannot be modified or deleted for a specified period.

**Use cases**:
- Regulatory compliance (SEC 17a-4, FINRA, HIPAA)
- Legal holds
- Ransomware protection

### Immutability Types

| Type | Deletable | Extendable | Use Case |
|------|-----------|------------|----------|
| **Time-based retention** | No (until expiration) | Yes | Compliance retention |
| **Legal hold** | No (until removed) | N/A | Litigation, investigations |

### Enable Immutability

```bash
# Enable version-level immutability
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-versioning true \
  --resource-group myresourcegroup

# Set default time-based retention policy (container level)
az storage container immutability-policy create \
  --account-name mystorageaccount \
  --container-name compliance-data \
  --period 2555 \
  --allow-protected-append-writes true
```

### C# Implementation: Immutability

```csharp
// Set immutability policy on blob version
var blobClient = containerClient.GetBlobClient("compliance-record.pdf");

// Upload with immutability policy
var uploadResponse = await blobClient.UploadAsync(
    new BinaryData("Sensitive compliance data"),
    overwrite: true
);

// Set time-based retention (7 years)
var retentionPolicy = new BlobImmutabilityPolicy
{
    ExpiresOn = DateTimeOffset.UtcNow.AddYears(7),
    PolicyMode = BlobImmutabilityPolicyMode.Unlocked
};

await blobClient.SetImmutabilityPolicyAsync(retentionPolicy);

// Lock policy (cannot be deleted until expiration)
await blobClient.SetImmutabilityPolicyAsync(
    new BlobImmutabilityPolicy
    {
        ExpiresOn = retentionPolicy.ExpiresOn,
        PolicyMode = BlobImmutabilityPolicyMode.Locked
    }
);

Console.WriteLine("Immutability policy locked - blob protected until 2032");
```

## Complete Production Example

### Multi-Tier Data Management System

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "hot-data-lifecycle",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterLastAccessTimeGreaterThan": 30
            },
            "enableAutoTierToHotFromCool": {}
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["production/active/"],
          "blobIndexMatch": [
            {"name": "tier", "op": "==", "value": "hot"}
          ]
        }
      }
    },
    {
      "enabled": true,
      "name": "archive-old-backups",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 7
            },
            "delete": {
              "daysAfterCreationGreaterThan": 2555
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/"],
          "blobIndexMatch": [
            {"name": "retention", "op": "==", "value": "long-term"}
          ]
        }
      }
    },
    {
      "enabled": true,
      "name": "cleanup-temp-files",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterCreationGreaterThan": 1
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["temp/"],
          "blobIndexMatch": [
            {"name": "temporary", "op": "==", "value": "true"}
          ]
        }
      }
    },
    {
      "enabled": true,
      "name": "version-management",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "version": {
            "tierToCool": {
              "daysAfterCreationGreaterThan": 7
            },
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 30
            },
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "snapshot-retention",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "snapshot": {
            "tierToArchive": {
              "daysAfterCreationGreaterThan": 30
            },
            "delete": {
              "daysAfterCreationGreaterThan": 180
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

### C# Lifecycle Management Helper

```csharp
public class BlobLifecycleManager
{
    private readonly BlobServiceClient _serviceClient;

    public BlobLifecycleManager(BlobServiceClient serviceClient)
    {
        _serviceClient = serviceClient;
    }

    public async Task TagBlobForRetentionAsync(
        string containerName,
        string blobName,
        string retentionPolicy)
    {
        var containerClient = _serviceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var tags = new Dictionary<string, string>
        {
            ["retention"] = retentionPolicy,
            ["taggedDate"] = DateTime.UtcNow.ToString("o")
        };

        await blobClient.SetTagsAsync(tags);
        Console.WriteLine($"Tagged {blobName} with retention: {retentionPolicy}");
    }

    public async Task<Dictionary<string, int>> GetTierDistributionAsync(
        string containerName)
    {
        var distribution = new Dictionary<string, int>
        {
            ["Hot"] = 0,
            ["Cool"] = 0,
            ["Cold"] = 0,
            ["Archive"] = 0
        };

        var containerClient = _serviceClient.GetBlobContainerClient(containerName);

        await foreach (var blobItem in containerClient.GetBlobsAsync(
            traits: BlobTraits.Metadata))
        {
            var tier = blobItem.Properties.AccessTier?.ToString() ?? "Hot";
            distribution[tier]++;
        }

        return distribution;
    }

    public async Task GenerateLifecycleReportAsync(string containerName)
    {
        var containerClient = _serviceClient.GetBlobContainerClient(containerName);
        var report = new
        {
            TotalBlobs = 0,
            TotalSize = 0L,
            TierDistribution = new Dictionary<string, long>(),
            OldestBlob = DateTime.MaxValue,
            NewestBlob = DateTime.MinValue
        };

        await foreach (var blobItem in containerClient.GetBlobsAsync(
            traits: BlobTraits.Metadata))
        {
            report.TotalBlobs++;
            report.TotalSize += blobItem.Properties.ContentLength ?? 0;

            var tier = blobItem.Properties.AccessTier?.ToString() ?? "Hot";
            if (!report.TierDistribution.ContainsKey(tier))
                report.TierDistribution[tier] = 0;

            report.TierDistribution[tier] += blobItem.Properties.ContentLength ?? 0;

            if (blobItem.Properties.CreatedOn < report.OldestBlob)
                report.OldestBlob = blobItem.Properties.CreatedOn.Value.DateTime;

            if (blobItem.Properties.CreatedOn > report.NewestBlob)
                report.NewestBlob = blobItem.Properties.CreatedOn.Value.DateTime;
        }

        Console.WriteLine($"Lifecycle Report for {containerName}:");
        Console.WriteLine($"  Total blobs: {report.TotalBlobs}");
        Console.WriteLine($"  Total size: {report.TotalSize / (1024.0 * 1024 * 1024):F2} GB");
        Console.WriteLine($"  Tier distribution:");

        foreach (var (tier, size) in report.TierDistribution)
        {
            Console.WriteLine($"    {tier}: {size / (1024.0 * 1024 * 1024):F2} GB");
        }
    }
}
```

## Monitoring and Validation

### Verify Lifecycle Actions

```bash
# Check lifecycle policy execution
az monitor activity-log list \
  --resource-group myresourcegroup \
  --query "[?contains(operationName.value, 'lifecycleManagement')]" \
  --output table
```

### Azure Monitor Alerts

```bash
# Create alert when lifecycle management fails
az monitor metrics alert create \
  --name lifecycle-failure-alert \
  --resource-group myresourcegroup \
  --scopes /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account} \
  --condition "count transactions where ResponseType = 'ServerError' and ApiName contains 'lifecycle'" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action email me@example.com
```

## Best Practices

### 1. Test Policies Before Production

```bash
# Test with small prefix first
{
  "filters": {
    "prefixMatch": ["test/lifecycle/"]
  }
}

# Expand to production after validation
{
  "filters": {
    "prefixMatch": ["production/"]
  }
}
```

### 2. Use Blob Index Tags for Complex Rules

```csharp
// Tag blobs on upload
var uploadOptions = new BlobUploadOptions
{
    Tags = new Dictionary<string, string>
    {
        ["retention"] = "long-term",
        ["department"] = "finance",
        ["compliance"] = "sox"
    }
};

await blobClient.UploadAsync(content, uploadOptions);
```

### 3. Combine with Soft Delete

**Recommended settings**:
- Soft delete: 7-14 days
- Lifecycle delete: After soft delete period
- Versioning: Enabled for critical data

### 4. Monitor Storage Costs

```bash
# View cost analysis by access tier
az consumption usage list \
  --start-date 2025-09-01 \
  --end-date 2025-09-30 \
  --query "[?contains(instanceName, 'mystorageaccount')]" \
  --output table
```

## References

**Microsoft Learn Documentation**:
- [Lifecycle Management](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
- [Lifecycle Policy Actions](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-policy-configure)
- [Soft Delete](https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview)
- [Blob Versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
- [Immutability Policies](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview)
- [Last Access Time](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview#move-data-based-on-last-accessed-time)

## Navigation

- **Previous**: `blob-concurrency.md`
- **Next**: `../04-table-storage/table-fundamentals.md`
- **Related**: `06-operations/governance.md` - Cost optimization and retention

## See Also

- `blob-fundamentals.md` - Access tier details
- `06-operations/governance.md` - Naming conventions and tagging strategies
- `06-operations/observability.md` - Monitoring lifecycle operations
