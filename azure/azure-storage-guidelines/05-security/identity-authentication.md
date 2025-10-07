# Identity and Authentication for Azure Storage

> **File Purpose**: Deep-dive into Managed Identity, DefaultAzureCredential authentication chain, RBAC roles, and CI/CD authentication patterns
> **Prerequisites**: Understanding of Azure AD concepts from `02-core-concepts/security-model.md`
> **Agent Use Case**: Reference when implementing production authentication, configuring service principals, or troubleshooting credential issues

## Quick Context

**Always use Azure AD authentication with Managed Identity** for Azure Storage access. Never use connection strings or account keys in production code. This file provides production-ready patterns for all environments: Azure-hosted services, CI/CD pipelines, and local development.

**Key principle**: DefaultAzureCredential provides a unified authentication interface that works across all environments without code changes. Configure least-privilege RBAC roles at the most specific scope possible.

## Managed Identity Overview

### System-Assigned vs User-Assigned Identity

| Feature | System-Assigned | User-Assigned |
|---------|----------------|---------------|
| **Lifecycle** | Tied to Azure resource | Independent lifecycle |
| **Sharing** | One resource only | Shared across multiple resources |
| **Management** | Auto-created with resource | Created separately |
| **Use case** | Single service (App Service, VM) | Shared identity (multiple VMs, microservices) |
| **Rotation** | Automatic | Automatic |
| **Cost** | Free | Free |
| **Deletion** | Auto-deleted with resource | Persists after resource deletion |

**Decision matrix**:
- Use **system-assigned** for single-purpose resources (1 App Service, 1 Function App)
- Use **user-assigned** for shared identity across multiple resources (AKS cluster, VM scale set, microservices)

### System-Assigned Managed Identity Setup

**Azure CLI - App Service**:

```bash
# Variables
APP_NAME="myapp"
RG_APP="rg-apps"
STORAGE_ACCOUNT="mystorageacct"
RG_STORAGE="rg-storage"

# 1. Enable system-assigned identity on App Service
az webapp identity assign \
  --name $APP_NAME \
  --resource-group $RG_APP

# 2. Get the principal ID (identity object ID)
PRINCIPAL_ID=$(az webapp identity show \
  --name $APP_NAME \
  --resource-group $RG_APP \
  --query principalId -o tsv)

echo "Principal ID: $PRINCIPAL_ID"

# 3. Assign Storage Blob Data Contributor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{subscription-id}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT

# 4. Assign Storage Table Data Contributor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Table Data Contributor" \
  --scope /subscriptions/{subscription-id}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT
```

**Azure CLI - Azure Function**:

```bash
# Enable system-assigned identity on Function App
az functionapp identity assign \
  --name myfunctionapp \
  --resource-group $RG_APP

# Get principal ID and assign roles (same as App Service)
PRINCIPAL_ID=$(az functionapp identity show \
  --name myfunctionapp \
  --resource-group $RG_APP \
  --query principalId -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT
```

**Azure CLI - Virtual Machine**:

```bash
# Enable system-assigned identity on VM
az vm identity assign \
  --name myvm \
  --resource-group rg-vms

PRINCIPAL_ID=$(az vm identity show \
  --name myvm \
  --resource-group rg-vms \
  --query principalId -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT
```

**Bicep template** (complete setup):

```bicep
param location string = resourceGroup().location
param appServiceName string
param storageAccountName string

// Existing storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: storageAccountName
}

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: '${appServiceName}-plan'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// App Service with system-assigned managed identity
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: true
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
      http20Enabled: true
    }
    httpsOnly: true
  }
}

// Role assignment: Storage Blob Data Contributor
// GUID: ba92f5b4-2d11-453d-a403-e96b0029c9fe
resource blobRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, appService.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Role assignment: Storage Table Data Contributor
// GUID: 0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3
resource tableRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, appService.id, '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output appServicePrincipalId string = appService.identity.principalId
output appServiceName string = appService.name
```

### User-Assigned Managed Identity Setup

**Azure CLI**:

```bash
# Variables
IDENTITY_NAME="myapp-identity"
RG_IDENTITY="rg-identities"
STORAGE_ACCOUNT="mystorageacct"
RG_STORAGE="rg-storage"

# 1. Create user-assigned managed identity
az identity create \
  --name $IDENTITY_NAME \
  --resource-group $RG_IDENTITY \
  --location westus3

# 2. Get identity details
IDENTITY_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RG_IDENTITY \
  --query id -o tsv)

PRINCIPAL_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RG_IDENTITY \
  --query principalId -o tsv)

CLIENT_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RG_IDENTITY \
  --query clientId -o tsv)

echo "Identity Resource ID: $IDENTITY_ID"
echo "Principal ID: $PRINCIPAL_ID"
echo "Client ID: $CLIENT_ID"

# 3. Assign RBAC roles to storage account
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Table Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT

# 4. Assign identity to App Service
az webapp identity assign \
  --name myapp \
  --resource-group rg-apps \
  --identities $IDENTITY_ID

# 5. Configure app to use specific client ID (optional, if multiple identities)
az webapp config appsettings set \
  --name myapp \
  --resource-group rg-apps \
  --settings AZURE_CLIENT_ID=$CLIENT_ID
```

**Bicep template**:

```bicep
param location string = resourceGroup().location
param identityName string = 'myapp-identity'
param storageAccountName string
param appServiceName string

// User-assigned managed identity
resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: identityName
  location: location
}

// Existing storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: storageAccountName
}

// Role assignment: Storage Blob Data Contributor
resource blobRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, identity.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: identity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

// Role assignment: Storage Table Data Contributor
resource tableRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, identity.id, '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3')
    principalId: identity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: '${appServiceName}-plan'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// App Service with user-assigned managed identity
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identity.id}': {}
    }
  }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: true
      appSettings: [
        {
          name: 'AZURE_CLIENT_ID'
          value: identity.properties.clientId
        }
      ]
    }
    httpsOnly: true
  }
}

// Outputs
output identityPrincipalId string = identity.properties.principalId
output identityClientId string = identity.properties.clientId
output identityResourceId string = identity.id
```

**When to use user-assigned identity**:
- Microservices sharing same identity (reduces role assignment overhead)
- AKS workload identity (pod-level identity)
- Multiple VMs in scale set
- Cross-resource group deployments (identity in shared RG)

## RBAC Role Assignments

### Built-In Storage Roles Reference

| Role Name | Role GUID | Permissions | Use Case |
|-----------|-----------|-------------|----------|
| **Storage Blob Data Owner** | `b7e6dc6d-f1e8-4753-8033-0f276bb0955b` | Full blob control + ACLs | ADLS Gen2 admin, ACL management |
| **Storage Blob Data Contributor** | `ba92f5b4-2d11-453d-a403-e96b0029c9fe` | Read/write/delete blobs | Application write access |
| **Storage Blob Data Reader** | `2a2b9908-6b94-4a23-b4d1-a27a9f1cef3f` | Read blobs only | Read-only apps, analytics |
| **Storage Table Data Contributor** | `0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3` | Read/write/delete entities | Application CRUD |
| **Storage Table Data Reader** | `76199698-9eea-4c19-bc75-cec21354c6b1` | Read entities only | Read-only queries |
| **Storage Queue Data Contributor** | `974c5e8b-45b9-4653-ba55-5f855dd0fb88` | Read/write/delete messages | Queue processing |
| **Storage Queue Data Reader** | `19e7f393-937e-4f77-808e-94535e297925` | Read messages only | Monitoring apps |

**Critical**: Never use `Contributor`, `Owner`, or `Reader` roles for data access. These are control-plane roles. Use data-plane roles only.

### Scope Levels (Least Privilege)

**From most permissive to least permissive**:

1. **Subscription** (avoid):
   ```bash
   # ❌ AVOID: Too broad
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Blob Data Contributor" \
     --scope /subscriptions/{subscription-id}
   ```

2. **Resource Group**:
   ```bash
   # ⚠️ Use only if accessing multiple storage accounts
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Blob Data Contributor" \
     --scope /subscriptions/{sub}/resourceGroups/rg-storage
   ```

3. **Storage Account** (recommended for most cases):
   ```bash
   # ✅ RECOMMENDED: Account-level access
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Blob Data Contributor" \
     --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
   ```

4. **Container** (most restrictive):
   ```bash
   # ✅ BEST: Container-level access
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Blob Data Reader" \
     --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct/blobServices/default/containers/public-assets
   ```

**Best practice**: Scope to container level when possible. Use account-level only if accessing multiple containers.

### Read-Only vs Read-Write Separation

**Pattern**: Use read-only roles wherever possible.

```bash
# Web API: Read/write to uploads container
az role assignment create \
  --assignee $WEB_API_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope .../storageAccounts/mystorageacct/blobServices/default/containers/uploads

# Background worker: Read from uploads, write to processed
az role assignment create \
  --assignee $WORKER_PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope .../containers/uploads

az role assignment create \
  --assignee $WORKER_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope .../containers/processed

# Analytics service: Read-only across all containers
az role assignment create \
  --assignee $ANALYTICS_PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope .../storageAccounts/mystorageacct
```

### Verify Role Assignments

```bash
# List all role assignments for a managed identity
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# List all role assignments on a storage account
az role assignment list \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Type:principalType}" -o table

# Check specific principal's access to specific scope
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --query "[].roleDefinitionName" -o tsv
```

## DefaultAzureCredential Authentication Chain

### How DefaultAzureCredential Works

**DefaultAzureCredential** tries authentication methods in this order:

1. **EnvironmentCredential** - Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`)
2. **WorkloadIdentityCredential** - Azure Kubernetes Service (AKS) workload identity
3. **ManagedIdentityCredential** - System or user-assigned managed identity (Azure services)
4. **SharedTokenCacheCredential** - Shared token cache (Visual Studio, Azure CLI)
5. **AzureCliCredential** - Azure CLI (`az login`)
6. **AzurePowerShellCredential** - Azure PowerShell (`Connect-AzAccount`)
7. **AzureDeveloperCliCredential** - Azure Developer CLI (`azd auth login`)
8. **InteractiveBrowserCredential** - Interactive browser login (fallback)

**Why this is powerful**: Same code works in Azure (managed identity), CI/CD (service principal), and local dev (Azure CLI) without changes.

### Application Code: Blob Storage

**Complete C# example with dependency injection**:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;

// Program.cs (ASP.NET Core 8)
var builder = WebApplication.CreateBuilder(args);

// 1. Register DefaultAzureCredential
builder.Services.AddSingleton<TokenCredential>(sp =>
{
    var options = new DefaultAzureCredentialOptions
    {
        // For user-assigned managed identity, specify client ID
        ManagedIdentityClientId = builder.Configuration["AZURE_CLIENT_ID"],

        // Exclude credentials not needed (reduces auth attempts)
        ExcludeInteractiveBrowserCredential = true,
        ExcludeSharedTokenCacheCredential = true,

        // Retry configuration
        Retry = {
            MaxRetries = 3,
            Delay = TimeSpan.FromSeconds(2)
        }
    };

    return new DefaultAzureCredential(options);
});

// 2. Register BlobServiceClient
builder.Services.AddSingleton(sp =>
{
    var credential = sp.GetRequiredService<TokenCredential>();
    var storageAccountName = builder.Configuration["StorageAccountName"];

    return new BlobServiceClient(
        new Uri($"https://{storageAccountName}.blob.core.windows.net"),
        credential,
        new BlobClientOptions
        {
            Retry = {
                MaxRetries = 3,
                Mode = Azure.Core.RetryMode.Exponential,
                Delay = TimeSpan.FromSeconds(2),
                MaxDelay = TimeSpan.FromSeconds(30)
            },
            Diagnostics = {
                IsLoggingEnabled = true,
                IsLoggingContentEnabled = false, // Don't log content (PII)
                IsTelemetryEnabled = true,
                ApplicationId = "MyApp/1.0"
            }
        });
});

var app = builder.Build();

// 3. Use in controller/service
public class FileController : ControllerBase
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<FileController> _logger;

    public FileController(
        BlobServiceClient blobServiceClient,
        ILogger<FileController> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
    }

    [HttpPost("upload")]
    public async Task<IActionResult> UploadFile(IFormFile file)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient("uploads");
            await containerClient.CreateIfNotExistsAsync();

            var blobClient = containerClient.GetBlobClient(file.FileName);

            using var stream = file.OpenReadStream();
            await blobClient.UploadAsync(stream, overwrite: true);

            _logger.LogInformation(
                "Uploaded blob {BlobName} to container {ContainerName}",
                file.FileName,
                "uploads");

            return Ok(new { blobName = file.FileName });
        }
        catch (Azure.RequestFailedException ex)
        {
            _logger.LogError(ex,
                "Failed to upload blob. Status: {Status}, ErrorCode: {ErrorCode}",
                ex.Status,
                ex.ErrorCode);
            return StatusCode(500, "Upload failed");
        }
    }
}
```

**Configuration (appsettings.json)**:

```json
{
  "StorageAccountName": "mystorageacct",
  "AZURE_CLIENT_ID": ""  // Only needed for user-assigned managed identity
}
```

### Application Code: Table Storage

```csharp
using Azure.Data.Tables;
using Azure.Identity;
using Microsoft.Extensions.DependencyInjection;

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 1. Register TokenCredential (same as blob example)
builder.Services.AddSingleton<TokenCredential>(sp =>
    new DefaultAzureCredential(new DefaultAzureCredentialOptions
    {
        ManagedIdentityClientId = builder.Configuration["AZURE_CLIENT_ID"],
        ExcludeInteractiveBrowserCredential = true
    }));

// 2. Register TableServiceClient
builder.Services.AddSingleton(sp =>
{
    var credential = sp.GetRequiredService<TokenCredential>();
    var storageAccountName = builder.Configuration["StorageAccountName"];

    return new TableServiceClient(
        new Uri($"https://{storageAccountName}.table.core.windows.net"),
        credential,
        new TableClientOptions
        {
            Retry = {
                MaxRetries = 3,
                Mode = Azure.Core.RetryMode.Exponential
            }
        });
});

// 3. Register table-specific clients
builder.Services.AddScoped(sp =>
{
    var tableServiceClient = sp.GetRequiredService<TableServiceClient>();
    return tableServiceClient.GetTableClient("Users");
});

var app = builder.Build();

// 4. Use in repository/service
public class UserRepository
{
    private readonly TableClient _tableClient;
    private readonly ILogger<UserRepository> _logger;

    public UserRepository(TableClient tableClient, ILogger<UserRepository> logger)
    {
        _tableClient = tableClient;
        _logger = logger;
    }

    public async Task<TableEntity?> GetUserAsync(string partitionKey, string rowKey)
    {
        try
        {
            var response = await _tableClient.GetEntityAsync<TableEntity>(
                partitionKey,
                rowKey);

            return response.Value;
        }
        catch (Azure.RequestFailedException ex) when (ex.Status == 404)
        {
            _logger.LogInformation(
                "User not found: {PartitionKey}/{RowKey}",
                partitionKey,
                rowKey);
            return null;
        }
        catch (Azure.RequestFailedException ex)
        {
            _logger.LogError(ex,
                "Failed to get user. Status: {Status}, ErrorCode: {ErrorCode}",
                ex.Status,
                ex.ErrorCode);
            throw;
        }
    }

    public async Task AddUserAsync(string email, string name)
    {
        var entity = new TableEntity("users", Guid.NewGuid().ToString())
        {
            { "Email", email },
            { "Name", name },
            { "CreatedAt", DateTimeOffset.UtcNow }
        };

        await _tableClient.AddEntityAsync(entity);

        _logger.LogInformation(
            "Added user: {Email} with RowKey {RowKey}",
            email,
            entity.RowKey);
    }
}
```

### Health Checks Integration

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Azure.Storage.Blobs;

// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<BlobStorageHealthCheck>("blob_storage")
    .AddCheck<TableStorageHealthCheck>("table_storage");

// Health check implementation
public class BlobStorageHealthCheck : IHealthCheck
{
    private readonly BlobServiceClient _blobServiceClient;

    public BlobStorageHealthCheck(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Test authentication and connectivity
            await _blobServiceClient.GetAccountInfoAsync(cancellationToken);

            return HealthCheckResult.Healthy("Blob storage is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Blob storage is not accessible",
                ex);
        }
    }
}
```

## CI/CD Authentication

### GitHub Actions with OIDC (Recommended)

**OIDC (OpenID Connect)** allows GitHub Actions to authenticate to Azure without storing secrets. Azure trusts GitHub's identity provider.

**Setup steps**:

```bash
# 1. Create Azure AD application
az ad app create \
  --display-name "github-actions-myapp"

APP_ID=$(az ad app list \
  --display-name "github-actions-myapp" \
  --query "[0].appId" -o tsv)

# 2. Create service principal
az ad sp create --id $APP_ID

OBJECT_ID=$(az ad sp list \
  --filter "appId eq '$APP_ID'" \
  --query "[0].id" -o tsv)

# 3. Create federated credential (GitHub OIDC)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Assign RBAC roles to service principal
PRINCIPAL_ID=$(az ad sp list \
  --filter "appId eq '$APP_ID'" \
  --query "[0].id" -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct

# 5. Get Azure details for GitHub secrets
TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "AZURE_CLIENT_ID: $APP_ID"
echo "AZURE_TENANT_ID: $TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
```

**GitHub Actions workflow** (.github/workflows/deploy.yml):

```yaml
name: Deploy to Azure

on:
  push:
    branches: [main]

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # 1. Authenticate with Azure using OIDC
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # 2. Upload files to blob storage
      - name: Upload to Blob Storage
        run: |
          az storage blob upload-batch \
            --account-name mystorageacct \
            --destination 'releases' \
            --source ./dist \
            --auth-mode login \
            --overwrite

      # 3. Use Azure CLI with managed identity (no secrets)
      - name: Deploy App Service
        run: |
          az webapp deployment source config-zip \
            --name myapp \
            --resource-group rg-apps \
            --src ./app.zip
```

**GitHub repository secrets** (Settings > Secrets and variables > Actions):
- `AZURE_CLIENT_ID`: Application (client) ID
- `AZURE_TENANT_ID`: Directory (tenant) ID
- `AZURE_SUBSCRIPTION_ID`: Subscription ID

**Benefits**:
- No secrets stored in GitHub (OIDC token-based)
- Automatic credential rotation
- Per-branch federation (separate credentials for main/staging)
- Audit trail via Azure AD logs

### GitHub Actions with Service Principal (Legacy)

**If OIDC not available** (older GitHub Enterprise):

```bash
# 1. Create service principal with password
az ad sp create-for-rbac \
  --name "github-actions-myapp" \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --sdk-auth

# Output (store as AZURE_CREDENTIALS secret):
# {
#   "clientId": "...",
#   "clientSecret": "...",
#   "subscriptionId": "...",
#   "tenantId": "..."
# }
```

**GitHub workflow**:

```yaml
- name: Azure Login
  uses: azure/login@v2
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}
```

**Drawbacks**:
- Secret stored in GitHub (rotation required every 90 days)
- Less secure than OIDC

### Azure DevOps with Service Connection

**Setup**:

1. In Azure DevOps: Project Settings > Service connections > New service connection > Azure Resource Manager
2. Choose "Workload Identity federation (automatic)" for OIDC
3. Select subscription and resource group
4. Name: `azure-storage-connection`

**Azure Pipelines YAML**:

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  storageAccountName: 'mystorageacct'

steps:
  - task: AzureCLI@2
    displayName: 'Upload to Blob Storage'
    inputs:
      azureSubscription: 'azure-storage-connection'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az storage blob upload-batch \
          --account-name $(storageAccountName) \
          --destination 'releases' \
          --source ./dist \
          --auth-mode login \
          --overwrite

  - task: AzureCLI@2
    displayName: 'Verify Upload'
    inputs:
      azureSubscription: 'azure-storage-connection'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az storage blob list \
          --account-name $(storageAccountName) \
          --container-name 'releases' \
          --auth-mode login \
          --query "[].name" -o tsv
```

**C# task using DefaultAzureCredential** (works in Azure DevOps):

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run Upload Script'
  inputs:
    command: 'run'
    projects: '**/*UploadTool.csproj'
  env:
    AZURE_CLIENT_ID: $(servicePrincipalId)
    AZURE_CLIENT_SECRET: $(servicePrincipalKey)
    AZURE_TENANT_ID: $(tenantId)
    STORAGE_ACCOUNT_NAME: $(storageAccountName)
```

## Local Development Authentication

### Azure CLI Authentication (Recommended)

**Setup**:

```bash
# 1. Login with your Azure AD account
az login

# 2. Set default subscription
az account set --subscription "My Subscription"

# 3. Verify login
az account show

# 4. Ensure your user has RBAC roles on storage account
MY_PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --assignee $MY_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct

az role assignment create \
  --assignee $MY_PRINCIPAL_ID \
  --role "Storage Table Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
```

**Application code** (no changes needed):

```csharp
// DefaultAzureCredential automatically uses Azure CLI credentials
var credential = new DefaultAzureCredential();
var blobServiceClient = new BlobServiceClient(
    new Uri("https://mystorageacct.blob.core.windows.net"),
    credential);
```

**How it works**:
1. `DefaultAzureCredential` tries `AzureCliCredential`
2. `AzureCliCredential` invokes `az account get-access-token`
3. Azure CLI returns token from cached credentials
4. Token used for storage API calls

**Benefits**:
- No secrets in code or configuration
- Same code works in Azure and locally
- Automatic token refresh

### Visual Studio / Visual Studio Code

**Visual Studio 2022**:
1. Tools > Options > Azure Service Authentication
2. Sign in with Azure AD account
3. DefaultAzureCredential uses SharedTokenCacheCredential

**Visual Studio Code**:
1. Install "Azure Account" extension
2. Run command: "Azure: Sign In"
3. DefaultAzureCredential uses SharedTokenCacheCredential

**Application code** (same as Azure CLI):

```csharp
var credential = new DefaultAzureCredential();
```

### Service Principal for Local Development (Not Recommended)

**Only use if Azure CLI not feasible**:

```bash
# 1. Create service principal
az ad sp create-for-rbac \
  --name "dev-myapp" \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct

# Output:
# {
#   "appId": "...",
#   "password": "...",
#   "tenant": "..."
# }

# 2. Store in environment variables (user secrets or .env file)
export AZURE_CLIENT_ID="..."
export AZURE_CLIENT_SECRET="..."
export AZURE_TENANT_ID="..."
```

**Application code** (DefaultAzureCredential uses EnvironmentCredential):

```csharp
// DefaultAzureCredential reads AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID
var credential = new DefaultAzureCredential();
```

**User secrets** (.NET):

```bash
# Initialize user secrets
dotnet user-secrets init

# Set secrets
dotnet user-secrets set "AZURE_CLIENT_ID" "..."
dotnet user-secrets set "AZURE_CLIENT_SECRET" "..."
dotnet user-secrets set "AZURE_TENANT_ID" "..."
```

```csharp
// Read from user secrets
var clientId = builder.Configuration["AZURE_CLIENT_ID"];
var clientSecret = builder.Configuration["AZURE_CLIENT_SECRET"];
var tenantId = builder.Configuration["AZURE_TENANT_ID"];

// Explicit ClientSecretCredential (if not using environment variables)
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

## Troubleshooting Authentication Issues

### Issue: ManagedIdentityCredential authentication failed

**Error**:
```
Azure.Identity.AuthenticationFailedException: ManagedIdentityCredential authentication failed:
No managed identity endpoint found.
```

**Diagnosis**:

```bash
# 1. Verify managed identity enabled
az webapp identity show \
  --name myapp \
  --resource-group rg-apps

# Should return principalId and tenantId

# 2. Check environment variables in App Service
az webapp config appsettings list \
  --name myapp \
  --resource-group rg-apps \
  --query "[?name=='IDENTITY_ENDPOINT' || name=='IDENTITY_HEADER']"

# These are auto-set by Azure, should be present
```

**Solutions**:
- Enable managed identity on App Service/VM
- Restart App Service after enabling identity
- Verify running in Azure (not local development)

### Issue: Insufficient privileges

**Error**:
```
Azure.RequestFailedException: (403) This request is not authorized to perform this operation.
Status: 403 (Forbidden)
ErrorCode: AuthorizationPermissionMismatch
```

**Diagnosis**:

```bash
# 1. Get principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name myapp \
  --resource-group rg-apps \
  --query principalId -o tsv)

# 2. Check role assignments
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# 3. Verify scope includes storage account
```

**Solutions**:
- Assign appropriate role (Storage Blob Data Contributor, not Contributor)
- Verify scope includes storage account or container
- Wait 5-10 minutes for RBAC propagation

### Issue: AzureCliCredential fails locally

**Error**:
```
Azure.Identity.CredentialUnavailableException: Azure CLI not installed
OR
Please run 'az login' to set up account
```

**Diagnosis**:

```bash
# 1. Verify Azure CLI installed
az --version

# 2. Verify logged in
az account show

# 3. Test token acquisition
az account get-access-token --resource https://storage.azure.com/
```

**Solutions**:
- Install Azure CLI: https://learn.microsoft.com/cli/azure/install-azure-cli
- Run `az login`
- Set default subscription: `az account set --subscription "..."`
- Verify user has RBAC roles on storage account

### Issue: Multiple managed identities, wrong one used

**Error**:
```
Azure.RequestFailedException: (403) Forbidden
```

**Diagnosis**:

```bash
# Check all identities assigned to App Service
az webapp identity show \
  --name myapp \
  --resource-group rg-apps

# If multiple user-assigned identities, specify which one to use
```

**Solutions**:

```csharp
// Specify client ID for user-assigned managed identity
var options = new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = "YOUR_CLIENT_ID"
};
var credential = new DefaultAzureCredential(options);
```

Or set environment variable in App Service:

```bash
az webapp config appsettings set \
  --name myapp \
  --resource-group rg-apps \
  --settings AZURE_CLIENT_ID="YOUR_CLIENT_ID"
```

### Debug Logging

**Enable verbose logging**:

```csharp
using Azure.Core.Diagnostics;

// Enable Azure SDK logging
using var listener = AzureEventSourceListener.CreateConsoleLogger(EventLevel.Verbose);

var options = new DefaultAzureCredentialOptions
{
    Diagnostics = {
        LoggedHeaderNames = { "x-ms-request-id" },
        LoggedQueryParameters = { "api-version" },
        IsLoggingContentEnabled = true
    }
};

var credential = new DefaultAzureCredential(options);
```

**View logs**:
- Shows which credential succeeded (EnvironmentCredential, ManagedIdentityCredential, etc.)
- Displays authentication errors for each attempted credential
- Includes HTTP request/response details

## Security Best Practices Checklist

### Identity Configuration

- [ ] Use managed identity (system or user-assigned) for all Azure services
- [ ] Never use account keys or connection strings in application code
- [ ] Use DefaultAzureCredential for environment-agnostic authentication
- [ ] Specify `ManagedIdentityClientId` for user-assigned identities
- [ ] Exclude unnecessary credentials in `DefaultAzureCredentialOptions` (reduce attack surface)

### RBAC Configuration

- [ ] Assign data-plane roles only (Storage Blob Data Contributor, not Contributor)
- [ ] Scope roles to container level when possible (not account or subscription)
- [ ] Use read-only roles (Storage Blob Data Reader) wherever possible
- [ ] Separate identities for different services (microservices pattern)
- [ ] Review role assignments quarterly (remove unused)

### CI/CD Security

- [ ] Use OIDC (workload identity federation) for GitHub Actions/Azure DevOps
- [ ] Never store service principal secrets in source control
- [ ] Rotate service principal secrets every 90 days (if not using OIDC)
- [ ] Use separate service principals for dev/staging/production
- [ ] Limit service principal permissions to deployment tasks only

### Local Development

- [ ] Use Azure CLI authentication (az login) for local development
- [ ] Assign RBAC roles to developer accounts (not permanent secrets)
- [ ] Use user secrets or environment variables (never hardcode)
- [ ] Exclude production credentials from local configuration

### Monitoring and Auditing

- [ ] Enable diagnostic logs for authentication events
- [ ] Monitor failed authentication attempts (Azure AD sign-in logs)
- [ ] Alert on role assignment changes
- [ ] Review access logs quarterly (who accessed what)

## Navigation

- **Previous**: `02-core-concepts/security-model.md`
- **Next**: `sas-patterns.md`
- **Up**: `00-overview.md`

## See Also

- `02-core-concepts/security-model.md` - Authentication methods overview
- `01-quick-start/authentication-setup.md` - Quick start guide
- `05-security/sas-patterns.md` - SAS token patterns
- `05-security/network-security.md` - Network security controls

## References

[1] "Use Azure AD to authorize access to blob data" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/blobs/authorize-access-azure-active-directory
[2] "Assign Azure roles for blob access" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/storage/blobs/assign-azure-role-data-access
[3] "DefaultAzureCredential class" - Azure SDK for .NET - 2024-09 - https://learn.microsoft.com/dotnet/api/azure.identity.defaultazurecredential
[4] "Use managed identities for App Service" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/app-service/overview-managed-identity
[5] "Configure workload identity federation" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/developer/github/connect-from-azure
[6] "Azure built-in roles for Storage" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/role-based-access-control/built-in-roles#storage
