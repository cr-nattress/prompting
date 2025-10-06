# Storage Account Provisioning

> **File Purpose**: Step-by-step provisioning of Azure Storage accounts with best-practice configuration
> **Prerequisites**: Azure subscription, Azure CLI installed, or Bicep deployment capability
> **Agent Use Case**: Reference when creating storage accounts via CLI or IaC

## Quick Context

Azure Storage accounts are the foundational resource for Blob, Table, Queue, and File services. This guide provisions accounts with security-hardened defaults: deny public network access, TLS 1.2 minimum, soft delete enabled, GZRS redundancy, and Managed Identity authentication.

**Key principle**: Provision infrastructure idempotently using CLI or Bicep. Never use Portal for production resources.

## Azure CLI Provisioning

### Step 1: Create Resource Group

**Goal**: Establish logical container for storage resources

```bash
# Variables
LOCATION="westus3"
RG_NAME="rg-storage-demo"
STORAGE_ACCOUNT_NAME="storacct$(openssl rand -hex 4)"  # Unique 8-char suffix

# Create resource group
az group create \
  --name $RG_NAME \
  --location $LOCATION \
  --tags Environment=Production Project=MyApp
```

**Why it works**: Resource groups provide lifecycle and RBAC boundaries. Tagging enables cost tracking.

**Common mistakes**:
- Not using unique storage account names globally (causes conflicts)
- Omitting tags (loses cost visibility)

### Step 2: Create Storage Account with Best Practices

**Goal**: Provision secure-by-default storage account

```bash
az storage account create \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RG_NAME \
  --location $LOCATION \
  --sku Standard_GZRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --default-action Deny \
  --bypass AzureServices \
  --enable-hierarchical-namespace false \
  --tags Environment=Production Project=MyApp

# Enable soft delete for blobs (30-day retention)
az storage blob service-properties delete-policy update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --enable true \
  --days-retained 30

# Enable blob versioning
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RG_NAME \
  --enable-versioning true
```

**Critical flags explained**:
- `--sku Standard_GZRS`: Geo-zone-redundant (survives zone and region failures)
- `--allow-blob-public-access false`: Prevents anonymous access to containers
- `--default-action Deny`: Blocks all network traffic by default (configure firewall rules or private endpoints)
- `--bypass AzureServices`: Allows Azure trusted services (Azure Monitor, etc.)
- `--min-tls-version TLS1_2`: Enforces modern encryption
- `--enable-hierarchical-namespace false`: Set to `true` for ADLS Gen2 analytics workloads

**Common mistakes**:
- Using `--default-action Allow` (exposes storage to internet)
- Omitting `--allow-blob-public-access false` (anonymous access risk)
- Enabling HNS without understanding cost implications

### Step 3: Create Containers and Tables

**Goal**: Provision blob containers and table storage

```bash
# Create blob container
az storage container create \
  --account-name $STORAGE_ACCOUNT_NAME \
  --name app-data \
  --auth-mode login \
  --public-access off

# Create table
az storage table create \
  --account-name $STORAGE_ACCOUNT_NAME \
  --name Users \
  --auth-mode login
```

**Why it works**: `--auth-mode login` uses your Azure AD credentials (no account keys needed). `--public-access off` ensures containers are private.

### Step 4: Verify Configuration

```bash
# Check account properties
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RG_NAME \
  --query "{name:name, sku:sku.name, httpsOnly:enableHttpsTrafficOnly, minTls:minimumTlsVersion, networkDefaultAction:networkRuleSet.defaultAction}"

# Output should show:
# {
#   "name": "storacct...",
#   "sku": "Standard_GZRS",
#   "httpsOnly": true,
#   "minTls": "TLS1_2",
#   "networkDefaultAction": "Deny"
# }
```

## Bicep Infrastructure as Code

### Complete Bicep Template

**Goal**: Idempotent, repeatable infrastructure provisioning

```bicep
// main.bicep
@description('Azure region for resources')
param location string = resourceGroup().location

@description('Unique storage account name (3-24 lowercase alphanumeric)')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Storage redundancy SKU')
@allowed([
  'Standard_LRS'
  'Standard_ZRS'
  'Standard_GRS'
  'Standard_GZRS'
  'Standard_RAGRS'
  'Standard_RAGZRS'
])
param sku string = 'Standard_GZRS'

@description('Enable hierarchical namespace (ADLS Gen2)')
param enableHierarchicalNamespace bool = false

@description('Environment tag')
param environment string = 'Production'

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: sku
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
    isHnsEnabled: enableHierarchicalNamespace
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
      virtualNetworkRules: []
      ipRules: []
    }
    encryption: {
      services: {
        blob: { enabled: true }
        table: { enabled: true }
      }
      keySource: 'Microsoft.Storage'  // Use 'Microsoft.Keyvault' for CMK
    }
  }
  tags: {
    Environment: environment
    ManagedBy: 'Bicep'
  }
}

// Blob Service (for soft delete and versioning)
resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 30
    }
    isVersioningEnabled: true
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
}

// Blob Container
resource appDataContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  parent: blobService
  name: 'app-data'
  properties: {
    publicAccess: 'None'
  }
}

// Table Service
resource tableService 'Microsoft.Storage/storageAccounts/tableServices@2023-05-01' = {
  parent: storageAccount
  name: 'default'
}

// Table
resource usersTable 'Microsoft.Storage/storageAccounts/tableServices/tables@2023-05-01' = {
  parent: tableService
  name: 'Users'
}

// Diagnostic Settings (send to Log Analytics)
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'storage-diagnostics'
  scope: storageAccount
  properties: {
    workspaceId: logAnalyticsWorkspaceId  // Pass as parameter
    logs: [
      {
        categoryGroup: 'allLogs'
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

// Outputs
output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
output tableEndpoint string = storageAccount.properties.primaryEndpoints.table
```

**Why it works**: Bicep is declarative and idempotent. Running this multiple times produces the same result. All security defaults are codified.

### Deploy Bicep Template

```bash
# Create parameters file
cat > parameters.json <<EOF
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "storacct$(openssl rand -hex 4)"
    },
    "sku": {
      "value": "Standard_GZRS"
    },
    "enableHierarchicalNamespace": {
      "value": false
    },
    "environment": {
      "value": "Production"
    },
    "logAnalyticsWorkspaceId": {
      "value": "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}"
    }
  }
}
EOF

# Deploy
az deployment group create \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.json \
  --mode Incremental
```

**Common mistakes**:
- Using `--mode Complete` (deletes resources not in template)
- Not parameterizing environment-specific values

## Optional: Enable Hierarchical Namespace (ADLS Gen2)

**Use only for analytics workloads** (Databricks, Synapse, Spark).

```bash
# WARNING: Cannot be disabled after enabling
az storage account update \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RG_NAME \
  --enable-hierarchical-namespace true
```

**Trade-offs**:
- **Pros**: Directory-level operations, ACLs, better for big data
- **Cons**: Higher cost, some Blob APIs unavailable, cannot disable

See `02-core-concepts/storage-accounts.md` for detailed HNS analysis.

## Verification Checklist

After provisioning, verify:

- [ ] Storage account name is globally unique
- [ ] SKU is `Standard_GZRS` (or appropriate redundancy)
- [ ] `allowBlobPublicAccess` is `false`
- [ ] `minimumTlsVersion` is `TLS1_2`
- [ ] `networkAcls.defaultAction` is `Deny`
- [ ] Soft delete enabled with 30-day retention
- [ ] Blob versioning enabled
- [ ] Containers have `publicAccess: None`
- [ ] Diagnostic settings configured (if Log Analytics available)
- [ ] Tags applied (Environment, Project)

## Next Steps

1. Configure authentication: `01-quick-start/authentication-setup.md`
2. Set up local development: `01-quick-start/local-development.md`
3. For production, add private endpoints: `05-security/network-security.md`

## Navigation

- **Previous**: `00-overview.md`
- **Next**: `authentication-setup.md`
- **Up**: `00-overview.md`

## See Also

- `02-core-concepts/storage-accounts.md` - Redundancy and HNS deep-dive
- `08-reference/bicep-templates.md` - Extended Bicep examples
- `05-security/network-security.md` - Private endpoint configuration
