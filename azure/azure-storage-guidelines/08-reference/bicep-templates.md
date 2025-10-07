# Bicep Templates Reference

> **File Purpose**: Complete, production-ready Bicep templates for Azure Storage infrastructure
> **Prerequisites**: Azure CLI with Bicep support, Azure subscription
> **Agent Use Case**: Copy-paste templates for storage account deployment across environments

## Quick Context

This reference provides deployable Bicep templates covering all Azure Storage scenarios: development, staging, production, private endpoints, diagnostic settings, lifecycle policies, and RBAC. All templates follow security best practices and are environment-parameterized.

**Key principle**: Use modules for reusability. Deploy via CI/CD pipelines with OIDC authentication (no service principal secrets).

---

## Complete Storage Account Template

### Full-Featured Storage Account (main.bicep)

```bicep
// main.bicep - Complete storage account with all production features
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

@description('Environment (Development, Staging, Production)')
@allowed(['Development', 'Staging', 'Production'])
param environment string = 'Production'

@description('Enable hierarchical namespace (ADLS Gen2)')
param enableHierarchicalNamespace bool = false

@description('Disable account key access (Managed Identity only)')
param allowSharedKeyAccess bool = false

@description('Allowed IP addresses or CIDR ranges')
param allowedIpAddresses array = []

@description('Allowed subnet resource IDs')
param allowedSubnetIds array = []

@description('Log Analytics workspace resource ID')
param logAnalyticsWorkspaceId string

@description('Enable customer-managed keys (CMK)')
param enableCustomerManagedKeys bool = false

@description('Key Vault resource ID (required if CMK enabled)')
param keyVaultId string = ''

@description('Key Vault key name (required if CMK enabled)')
param keyVaultKeyName string = ''

@description('Blob containers to create')
param blobContainers array = [
  'app-data'
  'uploads'
  'backups'
]

@description('Tables to create')
param tables array = [
  'Users'
  'Logs'
  'Configuration'
]

@description('Lifecycle management enabled')
param enableLifecycleManagement bool = true

@description('Soft delete retention days')
@minValue(1)
@maxValue(365)
param softDeleteRetentionDays int = 30

@description('Tags to apply to resources')
param tags object = {}

// Variables
var networkDefaultAction = environment == 'Development' ? 'Allow' : 'Deny'
var allowBlobPublicAccess = false
var minimumTlsVersion = 'TLS1_2'

var ipRules = [for ip in allowedIpAddresses: {
  value: ip
  action: 'Allow'
}]

var virtualNetworkRules = [for subnetId in allowedSubnetIds: {
  id: subnetId
  action: 'Allow'
}]

var commonTags = union({
  Environment: environment
  ManagedBy: 'Bicep'
  DeployedOn: utcNow('yyyy-MM-dd')
}, tags)

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: sku
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: allowBlobPublicAccess
    allowSharedKeyAccess: allowSharedKeyAccess
    minimumTlsVersion: minimumTlsVersion
    supportsHttpsTrafficOnly: true
    isHnsEnabled: enableHierarchicalNamespace
    networkAcls: {
      defaultAction: networkDefaultAction
      bypass: 'AzureServices'
      virtualNetworkRules: virtualNetworkRules
      ipRules: ipRules
    }
    encryption: enableCustomerManagedKeys ? {
      services: {
        blob: { enabled: true }
        table: { enabled: true }
        file: { enabled: true }
        queue: { enabled: true }
      }
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
        keyvaulturi: reference(keyVaultId, '2023-07-01').vaultUri
        keyname: keyVaultKeyName
      }
      identity: {
        userAssignedIdentity: null
      }
    } : {
      services: {
        blob: { enabled: true }
        table: { enabled: true }
        file: { enabled: true }
        queue: { enabled: true }
      }
      keySource: 'Microsoft.Storage'
    }
    accessTier: 'Hot'
  }
  identity: enableCustomerManagedKeys ? {
    type: 'SystemAssigned'
  } : null
  tags: commonTags
}

// Managed Identity for CMK (if enabled)
resource keyVaultAccessPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2023-07-01' = if (enableCustomerManagedKeys) {
  name: '${last(split(keyVaultId, '/'))}}/add'
  properties: {
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: storageAccount.identity.principalId
        permissions: {
          keys: [
            'get'
            'wrapKey'
            'unwrapKey'
          ]
        }
      }
    ]
  }
}

// Blob Service (soft delete, versioning, lifecycle)
resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: softDeleteRetentionDays
      allowPermanentDelete: false
    }
    isVersioningEnabled: true
    changeFeed: {
      enabled: environment == 'Production'
      retentionInDays: environment == 'Production' ? 90 : null
    }
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 7
    }
    cors: {
      corsRules: []
    }
    restorePolicy: environment == 'Production' ? {
      enabled: true
      days: 6
    } : null
  }
}

// Blob Containers
resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for containerName in blobContainers: {
  parent: blobService
  name: containerName
  properties: {
    publicAccess: 'None'
    metadata: {}
  }
}]

// Table Service
resource tableService 'Microsoft.Storage/storageAccounts/tableServices@2023-05-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    cors: {
      corsRules: []
    }
  }
}

// Tables
resource tableResources 'Microsoft.Storage/storageAccounts/tableServices/tables@2023-05-01' = [for tableName in tables: {
  parent: tableService
  name: tableName
}]

// Management Policy (Lifecycle rules)
resource managementPolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-05-01' = if (enableLifecycleManagement) {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          enabled: true
          name: 'move-to-cool-after-30-days'
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['app-data/', 'uploads/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 30
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 180
                }
                delete: {
                  daysAfterModificationGreaterThan: 365
                }
              }
              snapshot: {
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
              version: {
                tierToCool: {
                  daysAfterCreationGreaterThan: 30
                }
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
            }
          }
        }
        {
          enabled: true
          name: 'delete-old-backups'
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['backups/']
            }
            actions: {
              baseBlob: {
                tierToArchive: {
                  daysAfterModificationGreaterThan: 7
                }
                delete: {
                  daysAfterModificationGreaterThan: 90
                }
              }
            }
          }
        }
      ]
    }
  }
}

// Diagnostic Settings
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: '${storageAccountName}-diagnostics'
  scope: storageAccount
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: environment == 'Production' ? 90 : 30
        }
      }
    ]
    metrics: [
      {
        category: 'Transaction'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: environment == 'Production' ? 90 : 30
        }
      }
      {
        category: 'Capacity'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: environment == 'Production' ? 90 : 30
        }
      }
    ]
  }
}

// Blob Service Diagnostic Settings
resource blobDiagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'blob-diagnostics'
  scope: blobService
  properties: {
    workspaceId: logAnalyticsWorkspaceId
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
          days: 90
        }
      }
      {
        category: 'StorageDelete'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 90
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

// Table Service Diagnostic Settings
resource tableDiagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'table-diagnostics'
  scope: tableService
  properties: {
    workspaceId: logAnalyticsWorkspaceId
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

// Outputs
output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
output tableEndpoint string = storageAccount.properties.primaryEndpoints.table
output principalId string = enableCustomerManagedKeys ? storageAccount.identity.principalId : ''
```

---

## Private Endpoint Templates

### Private Endpoint Module (modules/private-endpoint.bicep)

```bicep
// modules/private-endpoint.bicep
@description('Name of the private endpoint')
param privateEndpointName string

@description('Location for the private endpoint')
param location string

@description('Storage account resource ID')
param storageAccountId string

@description('Subnet resource ID for private endpoint')
param subnetId string

@description('Private endpoint group ID (blob, table, file, queue)')
@allowed(['blob', 'table', 'file', 'queue', 'dfs'])
param groupId string

@description('Private DNS zone resource ID')
param privateDnsZoneId string

@description('Tags to apply')
param tags object = {}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: privateEndpointName
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: privateEndpointName
        properties: {
          privateLinkServiceId: storageAccountId
          groupIds: [
            groupId
          ]
        }
      }
    ]
  }
  tags: tags
}

// Private DNS Zone Group
resource privateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: privateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'config1'
        properties: {
          privateDnsZoneId: privateDnsZoneId
        }
      }
    ]
  }
}

output privateEndpointId string = privateEndpoint.id
output privateEndpointIp string = privateEndpoint.properties.customDnsConfigs[0].ipAddresses[0]
```

### Private Endpoint Deployment (storage-with-private-endpoints.bicep)

```bicep
// storage-with-private-endpoints.bicep
param location string = resourceGroup().location
param storageAccountName string
param vnetId string
param subnetId string
param environment string = 'Production'

// Deploy storage account (using main template as module)
module storageAccount './main.bicep' = {
  name: 'storage-deployment'
  params: {
    location: location
    storageAccountName: storageAccountName
    environment: environment
    allowedSubnetIds: [subnetId]
    // ... other parameters
  }
}

// Private DNS Zones
var privateDnsZones = {
  blob: 'privatelink.blob.${az.environment().suffixes.storage}'
  table: 'privatelink.table.${az.environment().suffixes.storage}'
  file: 'privatelink.file.${az.environment().suffixes.storage}'
  queue: 'privatelink.queue.${az.environment().suffixes.storage}'
  dfs: 'privatelink.dfs.${az.environment().suffixes.storage}'
}

resource blobPrivateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: privateDnsZones.blob
  location: 'global'
}

resource tablePrivateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: privateDnsZones.table
  location: 'global'
}

// Link DNS zones to VNet
resource blobDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: blobPrivateDnsZone
  name: '${storageAccountName}-blob-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnetId
    }
  }
}

resource tableDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: tablePrivateDnsZone
  name: '${storageAccountName}-table-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnetId
    }
  }
}

// Blob Private Endpoint
module blobPrivateEndpoint './modules/private-endpoint.bicep' = {
  name: 'blob-pe-deployment'
  params: {
    privateEndpointName: '${storageAccountName}-blob-pe'
    location: location
    storageAccountId: storageAccount.outputs.storageAccountId
    subnetId: subnetId
    groupId: 'blob'
    privateDnsZoneId: blobPrivateDnsZone.id
  }
}

// Table Private Endpoint
module tablePrivateEndpoint './modules/private-endpoint.bicep' = {
  name: 'table-pe-deployment'
  params: {
    privateEndpointName: '${storageAccountName}-table-pe'
    location: location
    storageAccountId: storageAccount.outputs.storageAccountId
    subnetId: subnetId
    groupId: 'table'
    privateDnsZoneId: tablePrivateDnsZone.id
  }
}

output storageAccountId string = storageAccount.outputs.storageAccountId
output blobPrivateEndpointIp string = blobPrivateEndpoint.outputs.privateEndpointIp
output tablePrivateEndpointIp string = tablePrivateEndpoint.outputs.privateEndpointIp
```

---

## RBAC Role Assignment Templates

### RBAC Module (modules/rbac.bicep)

```bicep
// modules/rbac.bicep
@description('Storage account name')
param storageAccountName string

@description('Principal ID (user, group, or service principal)')
param principalId string

@description('Principal type')
@allowed(['User', 'Group', 'ServicePrincipal'])
param principalType string = 'ServicePrincipal'

@description('Role definition (built-in roles)')
@allowed([
  'Storage Blob Data Owner'
  'Storage Blob Data Contributor'
  'Storage Blob Data Reader'
  'Storage Table Data Contributor'
  'Storage Table Data Reader'
  'Storage Account Contributor'
])
param roleDefinition string

// Built-in role definition IDs
var roleDefinitionIds = {
  'Storage Blob Data Owner': 'b7e6dc6d-f1e8-4753-8033-0f276bb0955b'
  'Storage Blob Data Contributor': 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'
  'Storage Blob Data Reader': '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1'
  'Storage Table Data Contributor': '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3'
  'Storage Table Data Reader': '76199698-9eea-4c19-bc75-cec21354c6b6'
  'Storage Account Contributor': '17d1049b-9a84-46fb-8f53-869881c3d3ab'
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: storageAccountName
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, principalId, roleDefinitionIds[roleDefinition])
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', roleDefinitionIds[roleDefinition])
    principalId: principalId
    principalType: principalType
  }
}

output roleAssignmentId string = roleAssignment.id
```

### RBAC Deployment Example

```bicep
// rbac-deployment.bicep
param storageAccountName string
param appServicePrincipalId string
param devGroupPrincipalId string

// App gets contributor access to blob and table
module appBlobAccess './modules/rbac.bicep' = {
  name: 'app-blob-rbac'
  params: {
    storageAccountName: storageAccountName
    principalId: appServicePrincipalId
    principalType: 'ServicePrincipal'
    roleDefinition: 'Storage Blob Data Contributor'
  }
}

module appTableAccess './modules/rbac.bicep' = {
  name: 'app-table-rbac'
  params: {
    storageAccountName: storageAccountName
    principalId: appServicePrincipalId
    principalType: 'ServicePrincipal'
    roleDefinition: 'Storage Table Data Contributor'
  }
}

// Dev team gets read access
module devBlobAccess './modules/rbac.bicep' = {
  name: 'dev-blob-rbac'
  params: {
    storageAccountName: storageAccountName
    principalId: devGroupPrincipalId
    principalType: 'Group'
    roleDefinition: 'Storage Blob Data Reader'
  }
}
```

---

## Environment-Specific Parameter Files

### Development (parameters.dev.json)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "storacctdev001"
    },
    "sku": {
      "value": "Standard_LRS"
    },
    "environment": {
      "value": "Development"
    },
    "enableHierarchicalNamespace": {
      "value": false
    },
    "allowSharedKeyAccess": {
      "value": true
    },
    "allowedIpAddresses": {
      "value": []
    },
    "allowedSubnetIds": {
      "value": []
    },
    "logAnalyticsWorkspaceId": {
      "value": "/subscriptions/{sub-id}/resourceGroups/rg-monitoring-dev/providers/Microsoft.OperationalInsights/workspaces/law-dev"
    },
    "enableCustomerManagedKeys": {
      "value": false
    },
    "keyVaultId": {
      "value": ""
    },
    "keyVaultKeyName": {
      "value": ""
    },
    "blobContainers": {
      "value": ["app-data", "uploads", "test-data"]
    },
    "tables": {
      "value": ["Users", "Logs", "TestData"]
    },
    "enableLifecycleManagement": {
      "value": false
    },
    "softDeleteRetentionDays": {
      "value": 7
    },
    "tags": {
      "value": {
        "Project": "MyApp",
        "CostCenter": "Engineering",
        "Owner": "dev-team@company.com"
      }
    }
  }
}
```

### Staging (parameters.staging.json)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "storacctst001"
    },
    "sku": {
      "value": "Standard_ZRS"
    },
    "environment": {
      "value": "Staging"
    },
    "enableHierarchicalNamespace": {
      "value": false
    },
    "allowSharedKeyAccess": {
      "value": false
    },
    "allowedIpAddresses": {
      "value": []
    },
    "allowedSubnetIds": {
      "value": [
        "/subscriptions/{sub-id}/resourceGroups/rg-network-staging/providers/Microsoft.Network/virtualNetworks/vnet-staging/subnets/snet-app"
      ]
    },
    "logAnalyticsWorkspaceId": {
      "value": "/subscriptions/{sub-id}/resourceGroups/rg-monitoring-staging/providers/Microsoft.OperationalInsights/workspaces/law-staging"
    },
    "enableCustomerManagedKeys": {
      "value": false
    },
    "blobContainers": {
      "value": ["app-data", "uploads", "backups"]
    },
    "tables": {
      "value": ["Users", "Logs", "Configuration"]
    },
    "enableLifecycleManagement": {
      "value": true
    },
    "softDeleteRetentionDays": {
      "value": 14
    },
    "tags": {
      "value": {
        "Project": "MyApp",
        "CostCenter": "Engineering",
        "Owner": "platform-team@company.com"
      }
    }
  }
}
```

### Production (parameters.prod.json)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "storacctprod001"
    },
    "sku": {
      "value": "Standard_GZRS"
    },
    "environment": {
      "value": "Production"
    },
    "enableHierarchicalNamespace": {
      "value": false
    },
    "allowSharedKeyAccess": {
      "value": false
    },
    "allowedIpAddresses": {
      "value": []
    },
    "allowedSubnetIds": {
      "value": [
        "/subscriptions/{sub-id}/resourceGroups/rg-network-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-app",
        "/subscriptions/{sub-id}/resourceGroups/rg-network-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-backend"
      ]
    },
    "logAnalyticsWorkspaceId": {
      "value": "/subscriptions/{sub-id}/resourceGroups/rg-monitoring-prod/providers/Microsoft.OperationalInsights/workspaces/law-prod"
    },
    "enableCustomerManagedKeys": {
      "value": true
    },
    "keyVaultId": {
      "value": "/subscriptions/{sub-id}/resourceGroups/rg-security-prod/providers/Microsoft.KeyVault/vaults/kv-prod-001"
    },
    "keyVaultKeyName": {
      "value": "storage-encryption-key"
    },
    "blobContainers": {
      "value": ["app-data", "uploads", "backups", "archive"]
    },
    "tables": {
      "value": ["Users", "Logs", "Configuration", "AuditTrail"]
    },
    "enableLifecycleManagement": {
      "value": true
    },
    "softDeleteRetentionDays": {
      "value": 30
    },
    "tags": {
      "value": {
        "Project": "MyApp",
        "CostCenter": "Production",
        "Owner": "platform-team@company.com",
        "Compliance": "SOC2"
      }
    }
  }
}
```

---

## Deployment Commands

### Azure CLI Deployment

```bash
# Variables
LOCATION="westus3"
RG_NAME="rg-storage-prod"
SUBSCRIPTION_ID="your-subscription-id"

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Create resource group
az group create \
  --name $RG_NAME \
  --location $LOCATION \
  --tags Environment=Production ManagedBy=Bicep

# Deploy development environment
az deployment group create \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.dev.json \
  --mode Incremental \
  --name "storage-dev-$(date +%Y%m%d-%H%M%S)"

# Deploy staging environment
az deployment group create \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.staging.json \
  --mode Incremental \
  --name "storage-staging-$(date +%Y%m%d-%H%M%S)"

# Deploy production environment (with confirmation)
az deployment group create \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.prod.json \
  --mode Incremental \
  --confirm-with-what-if \
  --name "storage-prod-$(date +%Y%m%d-%H%M%S)"

# Validate template before deployment
az deployment group validate \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.prod.json

# Preview changes (what-if)
az deployment group what-if \
  --resource-group $RG_NAME \
  --template-file main.bicep \
  --parameters parameters.prod.json
```

### PowerShell Deployment

```powershell
# Variables
$Location = "westus3"
$ResourceGroupName = "rg-storage-prod"
$SubscriptionId = "your-subscription-id"

# Set subscription
Set-AzContext -SubscriptionId $SubscriptionId

# Create resource group
New-AzResourceGroup `
  -Name $ResourceGroupName `
  -Location $Location `
  -Tag @{Environment="Production"; ManagedBy="Bicep"}

# Deploy with what-if preview
New-AzResourceGroupDeployment `
  -ResourceGroupName $ResourceGroupName `
  -TemplateFile "main.bicep" `
  -TemplateParameterFile "parameters.prod.json" `
  -Mode Incremental `
  -WhatIf

# Deploy production
New-AzResourceGroupDeployment `
  -ResourceGroupName $ResourceGroupName `
  -TemplateFile "main.bicep" `
  -TemplateParameterFile "parameters.prod.json" `
  -Mode Incremental `
  -Name "storage-prod-$(Get-Date -Format 'yyyyMMdd-HHmmss')" `
  -Confirm
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy-storage.yml
name: Deploy Storage Infrastructure

on:
  push:
    branches:
      - main
    paths:
      - 'infrastructure/storage/**'
  pull_request:
    branches:
      - main
    paths:
      - 'infrastructure/storage/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

permissions:
  id-token: write
  contents: read

jobs:
  validate:
    name: Validate Bicep Templates
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Validate Bicep
        run: |
          az bicep build --file infrastructure/storage/main.bicep
          az deployment group validate \
            --resource-group rg-storage-${{ inputs.environment || 'dev' }} \
            --template-file infrastructure/storage/main.bicep \
            --parameters infrastructure/storage/parameters.${{ inputs.environment || 'dev' }}.json

  plan:
    name: Plan Changes (What-If)
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Run What-If
        run: |
          az deployment group what-if \
            --resource-group rg-storage-dev \
            --template-file infrastructure/storage/main.bicep \
            --parameters infrastructure/storage/parameters.dev.json \
            --result-format FullResourcePayloads

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' || inputs.environment == 'development'
    environment: development
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Storage Account
        run: |
          az deployment group create \
            --resource-group rg-storage-dev \
            --template-file infrastructure/storage/main.bicep \
            --parameters infrastructure/storage/parameters.dev.json \
            --mode Incremental \
            --name "storage-dev-${{ github.run_number }}"

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main' || inputs.environment == 'staging'
    environment: staging
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Storage Account
        run: |
          az deployment group create \
            --resource-group rg-storage-staging \
            --template-file infrastructure/storage/main.bicep \
            --parameters infrastructure/storage/parameters.staging.json \
            --mode Incremental \
            --name "storage-staging-${{ github.run_number }}"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' || inputs.environment == 'production'
    environment: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Storage Account
        run: |
          az deployment group create \
            --resource-group rg-storage-prod \
            --template-file infrastructure/storage/main.bicep \
            --parameters infrastructure/storage/parameters.prod.json \
            --mode Incremental \
            --confirm-with-what-if \
            --name "storage-prod-${{ github.run_number }}"

      - name: Verify Deployment
        run: |
          STORAGE_ACCOUNT=$(az deployment group show \
            --resource-group rg-storage-prod \
            --name "storage-prod-${{ github.run_number }}" \
            --query properties.outputs.storageAccountName.value -o tsv)

          echo "Verifying storage account: $STORAGE_ACCOUNT"

          az storage account show \
            --name $STORAGE_ACCOUNT \
            --resource-group rg-storage-prod \
            --query "{name:name, sku:sku.name, publicAccess:allowBlobPublicAccess, minTls:minimumTlsVersion, networkDefault:networkRuleSet.defaultAction}"
```

---

## ARM Template Alternative

For teams requiring ARM JSON instead of Bicep:

### Convert Bicep to ARM

```bash
# Convert Bicep to ARM JSON
az bicep build --file main.bicep --outfile main.json

# Deploy ARM template
az deployment group create \
  --resource-group $RG_NAME \
  --template-file main.json \
  --parameters parameters.prod.json
```

### ARM Template (Simplified Example)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 24,
      "metadata": {
        "description": "Storage account name"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "sku": {
      "type": "string",
      "defaultValue": "Standard_GZRS",
      "allowedValues": [
        "Standard_LRS",
        "Standard_ZRS",
        "Standard_GRS",
        "Standard_GZRS"
      ]
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-05-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('sku')]"
      },
      "kind": "StorageV2",
      "properties": {
        "allowBlobPublicAccess": false,
        "minimumTlsVersion": "TLS1_2",
        "supportsHttpsTrafficOnly": true,
        "networkAcls": {
          "defaultAction": "Deny",
          "bypass": "AzureServices"
        }
      }
    }
  ],
  "outputs": {
    "storageAccountId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
```

---

## Module Patterns for Reusability

### Project Structure

```
infrastructure/
├── modules/
│   ├── storage-account.bicep
│   ├── private-endpoint.bicep
│   ├── rbac.bicep
│   └── diagnostic-settings.bicep
├── environments/
│   ├── dev/
│   │   ├── main.bicep
│   │   └── parameters.json
│   ├── staging/
│   │   ├── main.bicep
│   │   └── parameters.json
│   └── prod/
│       ├── main.bicep
│       └── parameters.json
└── shared/
    └── naming.bicep
```

### Naming Convention Module (shared/naming.bicep)

```bicep
// shared/naming.bicep
@description('Resource type (storage, blob, table)')
param resourceType string

@description('Environment (dev, staging, prod)')
param environment string

@description('Application name')
param appName string

@description('Azure region abbreviation')
param regionAbbreviation string

@description('Instance number')
param instance string = '001'

var environmentAbbreviations = {
  development: 'dev'
  staging: 'st'
  production: 'prod'
}

var resourceTypeAbbreviations = {
  storage: 'st'
  blob: 'blob'
  table: 'table'
  vnet: 'vnet'
  subnet: 'snet'
}

output name string = '${resourceTypeAbbreviations[resourceType]}${appName}${environmentAbbreviations[environment]}${regionAbbreviation}${instance}'
```

---

## Navigation

- **Previous**: `code-examples.md`
- **Next**: `references.md`
- **Up**: `00-overview.md`

## See Also

- `01-quick-start/provisioning.md` - Basic provisioning guide
- `05-security/network-security.md` - Private endpoint patterns
- `06-operations/governance.md` - Tagging and policy standards
- `08-reference/references.md` - Official Bicep documentation links
