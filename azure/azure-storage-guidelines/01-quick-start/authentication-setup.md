# Authentication Setup - Managed Identity & RBAC

> **File Purpose**: Configure secure authentication using Managed Identity and DefaultAzureCredential
> **Prerequisites**: `01-quick-start/provisioning.md` (storage account created)
> **Agent Use Case**: Reference when implementing authentication in applications

## Quick Context

**Critical principle**: Never use storage account keys in application code. Always use Managed Identity with RBAC for production workloads. DefaultAzureCredential provides seamless authentication across local dev (Azure CLI), CI/CD (service principals), and production (Managed Identity).

This guide configures Azure AD-based authentication, eliminating secrets management burden and adhering to least-privilege principles.

## Authentication Hierarchy (Recommended Priority)

1. **Managed Identity** (production apps on Azure)
2. **Workload Identity** (Kubernetes/AKS workloads)
3. **Service Principal** (CI/CD pipelines, non-Azure hosting)
4. **Azure CLI** (local development only)
5. **Account Keys** (avoid; legacy compatibility only)

## Step 1: Assign RBAC Roles

### For Blob Storage Access

**Goal**: Grant least-privilege access to Managed Identity or user principal

```bash
# Variables
STORAGE_ACCOUNT_NAME="storacct..."
RESOURCE_GROUP="rg-storage-demo"
PRINCIPAL_ID="<managed-identity-or-user-object-id>"

# Get storage account resource ID
STORAGE_ACCOUNT_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Assign Blob Data Contributor (read/write/delete)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID

# Or assign Blob Data Reader (read-only)
# az role assignment create \
#   --assignee $PRINCIPAL_ID \
#   --role "Storage Blob Data Reader" \
#   --scope $STORAGE_ACCOUNT_ID
```

**RBAC Roles for Blob Storage**:
- `Storage Blob Data Owner` - Full control (read/write/delete/ACLs)
- `Storage Blob Data Contributor` - Read/write/delete (most common)
- `Storage Blob Data Reader` - Read-only
- `Storage Blob Delegator` - Generate user delegation SAS tokens

**Scope best practices**:
- Scope to specific container: `--scope "$STORAGE_ACCOUNT_ID/blobServices/default/containers/app-data"`
- Scope to account: `--scope $STORAGE_ACCOUNT_ID`
- Avoid subscription-level scopes unless necessary

### For Table Storage Access

```bash
# Assign Table Data Contributor (read/write/delete)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Table Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID

# Or Table Data Reader (read-only)
# az role assignment create \
#   --assignee $PRINCIPAL_ID \
#   --role "Storage Table Data Reader" \
#   --scope $STORAGE_ACCOUNT_ID
```

**RBAC Roles for Table Storage**:
- `Storage Table Data Contributor` - Read/write/delete
- `Storage Table Data Reader` - Read-only

**Common mistakes**:
- Assigning `Contributor` (full ARM control) instead of `Storage Blob Data Contributor` (data plane only)
- Using subscription scope when resource-level scope suffices
- Forgetting to wait 5-10 minutes for RBAC propagation

### Verify Role Assignment

```bash
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $STORAGE_ACCOUNT_ID \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
```

## Step 2: Configure Managed Identity (Azure Resources)

### For Azure App Service / Azure Functions

```bash
# Enable system-assigned Managed Identity
APP_NAME="myapp"
az webapp identity assign \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP

# Get the Managed Identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Assign RBAC role (use commands from Step 1)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

**Why it works**: System-assigned Managed Identity is tied to App Service lifecycle. No credential rotation needed.

### For User-Assigned Managed Identity (Shared Across Resources)

```bash
# Create user-assigned identity
IDENTITY_NAME="id-storage-app"
az identity create \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP

# Get principal ID
PRINCIPAL_ID=$(az identity show \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Assign to App Service
az webapp identity assign \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --identities $(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)

# Assign RBAC role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

**Use user-assigned when**:
- Identity is shared across multiple resources
- Identity lifecycle is independent of resource lifecycle

## Step 3: Implement DefaultAzureCredential in .NET

### Install NuGet Packages

```bash
dotnet add package Azure.Storage.Blobs --version 12.22.0
dotnet add package Azure.Data.Tables --version 12.9.0
dotnet add package Azure.Identity --version 1.13.0
```

### Basic Client Initialization

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;

// DefaultAzureCredential tries in order:
// 1. Environment variables (CI/CD)
// 2. Managed Identity (Azure resources)
// 3. Visual Studio / VS Code
// 4. Azure CLI (local dev)
// 5. Azure PowerShell
var credential = new DefaultAzureCredential();

// Blob Service Client
var blobServiceUri = new Uri("https://storacct....blob.core.windows.net");
var blobServiceClient = new BlobServiceClient(blobServiceUri, credential);

// Table Service Client
var tableServiceUri = new Uri("https://storacct....table.core.windows.net");
var tableServiceClient = new TableServiceClient(tableServiceUri, credential);
```

**Why it works**: DefaultAzureCredential abstracts authentication across environments. No code changes between local, staging, and production.

**Common mistakes**:
- Hardcoding URIs (use configuration)
- Not handling authentication failures (credential chain exhausted)

### Dependency Injection Setup (ASP.NET Core)

```csharp
// Program.cs or Startup.cs
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;

var builder = WebApplication.CreateBuilder(args);

// Register DefaultAzureCredential as singleton
builder.Services.AddSingleton<DefaultAzureCredential>();

// Register BlobServiceClient
builder.Services.AddSingleton(sp =>
{
    var credential = sp.GetRequiredService<DefaultAzureCredential>();
    var blobUri = builder.Configuration["Azure:Storage:BlobServiceUri"]
        ?? throw new InvalidOperationException("Missing Azure:Storage:BlobServiceUri");
    return new BlobServiceClient(new Uri(blobUri), credential);
});

// Register TableServiceClient
builder.Services.AddSingleton(sp =>
{
    var credential = sp.GetRequiredService<DefaultAzureCredential>();
    var tableUri = builder.Configuration["Azure:Storage:TableServiceUri"]
        ?? throw new InvalidOperationException("Missing Azure:Storage:TableServiceUri");
    return new TableServiceClient(new Uri(tableUri), credential);
});

var app = builder.Build();
```

**Configuration (appsettings.json)**:

```json
{
  "Azure": {
    "Storage": {
      "BlobServiceUri": "https://storacct....blob.core.windows.net",
      "TableServiceUri": "https://storacct....table.core.windows.net"
    }
  }
}
```

**Why it works**: Configuration is environment-specific; credentials are implicit. No secrets in config files.

### Advanced: Custom Credential Options

```csharp
// For user-assigned Managed Identity, specify client ID
var credentialOptions = new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = "00000000-0000-0000-0000-000000000000"
};
var credential = new DefaultAzureCredential(credentialOptions);

// For timeout and retry customization
var clientOptions = new BlobClientOptions
{
    Retry = {
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(16),
        Mode = Azure.Core.RetryMode.Exponential
    },
    Diagnostics = {
        IsLoggingEnabled = true,
        IsDistributedTracingEnabled = true
    }
};

var blobServiceClient = new BlobServiceClient(blobServiceUri, credential, clientOptions);
```

## Step 4: Local Development Authentication

### Azure CLI Login (Recommended)

```bash
# Login with your Azure AD account
az login

# Set default subscription
az account set --subscription "<subscription-id>"

# Verify identity
az account show --query "{User:user.name, Subscription:name}"
```

**Why it works**: DefaultAzureCredential detects Azure CLI credentials automatically. No code changes needed.

**Common mistakes**:
- Not setting default subscription (DefaultAzureCredential fails)
- CLI token expiration (re-run `az login`)

### Grant Your User RBAC Access

```bash
# Get your user object ID
USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Assign roles for local development
az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID

az role assignment create \
  --assignee $USER_OBJECT_ID \
  --role "Storage Table Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

## Step 5: CI/CD Authentication (GitHub Actions)

### Configure Workload Identity Federation (OIDC)

**One-time Azure setup**:

```bash
# Create service principal for GitHub Actions
APP_NAME="github-actions-sp"
az ad app create --display-name $APP_NAME

APP_ID=$(az ad app list --display-name $APP_NAME --query "[0].appId" -o tsv)

# Create federated credential for GitHub
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-oidc",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Assign RBAC roles
PRINCIPAL_ID=$(az ad sp list --display-name $APP_NAME --query "[0].id" -o tsv)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

**GitHub Actions workflow**:

```yaml
name: Deploy
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

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Upload blob
        run: |
          az storage blob upload \
            --account-name storacct... \
            --container-name app-data \
            --name deployment-$(date +%s).txt \
            --file README.md \
            --auth-mode login
```

See `07-deployment/cicd-patterns.md` for complete CI/CD examples.

## Troubleshooting Authentication

### Diagnostic Commands

```bash
# Test blob access with Azure CLI
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name app-data \
  --auth-mode login

# Test table access
az storage table list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login
```

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `AuthorizationPermissionMismatch` | Missing RBAC role | Assign `Storage Blob Data Contributor` role |
| `CredentialUnavailable` | No credentials in chain | Run `az login` or configure Managed Identity |
| `403 Forbidden` | RBAC not propagated | Wait 5-10 minutes; verify role assignment |
| `Network access denied` | Firewall blocks traffic | Add IP to firewall or use private endpoint |

### Enable Diagnostic Logging

```csharp
// Add to Program.cs for detailed auth logs
using Azure.Core.Diagnostics;

using var listener = AzureEventSourceListener.CreateConsoleLogger(EventLevel.Verbose);
```

## Security Checklist

- [ ] Managed Identity enabled for Azure resources
- [ ] RBAC roles assigned (not `Contributor`, use `Storage *Data*` roles)
- [ ] Account keys disabled (optional but recommended): `az storage account update --allow-shared-key-access false`
- [ ] DefaultAzureCredential used in application code
- [ ] No connection strings or account keys in config files
- [ ] User-assigned identity documented if used
- [ ] CI/CD uses OIDC (not service principal secrets)
- [ ] Local developers granted RBAC access via Azure CLI

## Navigation

- **Previous**: `provisioning.md`
- **Next**: `local-development.md`
- **Up**: `00-overview.md`

## See Also

- `05-security/identity-authentication.md` - Deep dive on authentication patterns
- `05-security/sas-patterns.md` - When SAS is required (rare)
- `07-deployment/cicd-patterns.md` - Complete CI/CD authentication
