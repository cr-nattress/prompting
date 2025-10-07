# Storage Security Model and Authentication

> **File Purpose**: Comprehensive comparison of authentication methods (RBAC, SAS, Account Keys) with security decision framework
> **Prerequisites**: Basic understanding of Azure AD and identity concepts
> **Agent Use Case**: Reference when designing authentication strategy, implementing least privilege, or troubleshooting access issues

## Quick Context

Azure Storage supports three authentication methods: Azure AD (RBAC), Shared Access Signatures (SAS), and Account Keys. **Default to Azure AD with Managed Identity for all production workloads.** Use SAS only for time-limited external access. Never use Account Keys in application code.

**Key principle**: Authentication is defense-in-depth. Layer network security (private endpoints, firewall rules) with identity-based access control (RBAC) and least privilege principles.

## Authentication Methods Overview

### Comparison Table

| Feature | RBAC (Recommended) | SAS (Limited Use) | Account Keys (Avoid) |
|---------|-------------------|-------------------|---------------------|
| **Authentication** | Azure AD identity | Signed URL token | Shared secret |
| **Granularity** | Service/container/blob | Service/container/blob | Full account access |
| **Expiration** | Token-based (1 hour default) | Configurable (minutes to years) | Never expires |
| **Revocation** | Instant (via Azure AD) | Requires regeneration | Requires regeneration |
| **Audit trail** | Full (Azure AD logs) | Limited (storage logs) | Limited (storage logs) |
| **Least privilege** | Yes (granular roles) | Yes (limited permissions) | No (all or nothing) |
| **Zero-trust ready** | Yes | Partial | No |
| **Complexity** | Medium | Medium-High | Low |
| **Security posture** | Strongest | Moderate | Weakest |

### Decision Matrix

| Use Case | Recommended Method | Justification |
|----------|-------------------|---------------|
| Azure service (App Service, AKS, VM) | RBAC + Managed Identity | Zero credential management, automatic rotation |
| On-premises application | RBAC + Service Principal | Azure AD integration, audit trail |
| Third-party client (limited time) | User Delegation SAS | Time-boxed access, revocable |
| Anonymous public read (CDN) | SAS (container-level) | Read-only, IP-scoped |
| Local development | RBAC + Azure CLI | Uses developer Azure AD credentials |
| CI/CD pipeline | RBAC + OIDC (GitHub Actions) | No secrets in repo |
| Legacy application (cannot modify) | Account Key (rotate frequently) | Last resort, plan migration |
| Mobile/web frontend (browser) | SAS from backend API | Backend generates short-lived tokens |

## Azure AD (RBAC) - Recommended

### What Is RBAC?

**Role-Based Access Control (RBAC)** uses Azure AD identities (users, service principals, managed identities) with built-in or custom roles to grant least privilege access to storage resources.

### Built-In Roles

| Role | Scope | Permissions | Use Case |
|------|-------|-------------|----------|
| **Storage Blob Data Owner** | Blob containers | Full control (read/write/delete + ACLs) | Admin accounts, ADLS Gen2 ACL management |
| **Storage Blob Data Contributor** | Blob containers | Read/write/delete blobs | Application write access |
| **Storage Blob Data Reader** | Blob containers | Read blobs only | Read-only applications, analytics |
| **Storage Table Data Contributor** | Tables | Read/write/delete entities | Application CRUD operations |
| **Storage Table Data Reader** | Tables | Read entities only | Read-only applications |
| **Reader and Data Access** | Account | Read data + list keys | Legacy compatibility (avoid) |

**Critical**: Do NOT use `Contributor` or `Owner` roles for data access. Use data-plane roles only.

### Setup: Managed Identity (Recommended)

**System-assigned managed identity** (preferred for single resource):

```bash
# 1. Enable system-assigned managed identity on App Service
az webapp identity assign \
  --name myapp \
  --resource-group rg-apps

# 2. Get the principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --name myapp \
  --resource-group rg-apps \
  --query principalId -o tsv)

# 3. Assign role to storage account
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
```

**User-assigned managed identity** (preferred for shared identity):

```bash
# 1. Create user-assigned identity
az identity create \
  --name myapp-identity \
  --resource-group rg-identities

# 2. Assign to App Service
az webapp identity assign \
  --name myapp \
  --resource-group rg-apps \
  --identities /subscriptions/{sub}/resourceGroups/rg-identities/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myapp-identity

# 3. Get principal ID
PRINCIPAL_ID=$(az identity show \
  --name myapp-identity \
  --resource-group rg-identities \
  --query principalId -o tsv)

# 4. Assign role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
```

**Bicep template** (complete setup):

```bicep
// User-assigned managed identity
resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'myapp-identity'
  location: location
}

// Storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: storageAccountName
}

// Role assignment (Storage Blob Data Contributor)
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, identity.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // GUID for role
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // Storage Blob Data Contributor
    principalId: identity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

// App Service with managed identity
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
    // ... app service configuration
  }
}
```

**Built-in role GUIDs** (for Bicep):
- Storage Blob Data Owner: `b7e6dc6d-f1e8-4753-8033-0f276bb0955b`
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Storage Blob Data Reader: `2a2b9908-6b94-4a23-b4d1-a27a9f1cef3f`
- Storage Table Data Contributor: `0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3`
- Storage Table Data Reader: `76199698-9eea-4c19-bc75-cec21354c6b1`

### Application Code: DefaultAzureCredential

**C# example** (works in Azure and locally):

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

// DefaultAzureCredential chain:
// 1. Environment variables (for local dev with service principal)
// 2. Managed Identity (when running in Azure)
// 3. Azure CLI (when running locally with `az login`)
var credential = new DefaultAzureCredential();

var blobServiceClient = new BlobServiceClient(
    new Uri("https://mystorageacct.blob.core.windows.net"),
    credential);

// Upload blob
var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
var blobClient = containerClient.GetBlobClient("myfile.txt");
await blobClient.UploadAsync(stream, overwrite: true);
```

**Table Storage example**:

```csharp
using Azure.Data.Tables;
using Azure.Identity;

var credential = new DefaultAzureCredential();

var tableServiceClient = new TableServiceClient(
    new Uri("https://mystorageacct.table.core.windows.net"),
    credential);

var tableClient = tableServiceClient.GetTableClient("Users");
await tableClient.CreateIfNotExistsAsync();

// Insert entity
var entity = new TableEntity("partition1", "row1")
{
    { "Email", "user@example.com" },
    { "Name", "John Doe" }
};
await tableClient.AddEntityAsync(entity);
```

### Local Development with Azure CLI

```bash
# Login with your Azure AD account
az login

# Set default subscription
az account set --subscription "My Subscription"

# Now DefaultAzureCredential uses your Azure CLI credentials
dotnet run
```

**Important**: Ensure your Azure AD user has `Storage Blob Data Contributor` role on the storage account.

```bash
# Assign role to yourself (for local development)
MY_PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --assignee $MY_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
```

### Advanced: Conditional Access and PIM

**Conditional Access**: Enforce MFA, device compliance, IP restrictions at Azure AD level.

```bash
# Example: Create conditional access policy (requires Azure AD Premium)
az ad conditional-access policy create \
  --display-name "Require MFA for Storage Access" \
  --conditions "{ ... }" \
  --grant-controls "{ 'operator': 'OR', 'builtInControls': ['mfa'] }"
```

**Privileged Identity Management (PIM)**: Just-in-time RBAC elevation.

```bash
# Request Storage Blob Data Owner role for 8 hours
az pim role assignment create \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --role "Storage Blob Data Owner" \
  --principal-id $MY_PRINCIPAL_ID \
  --duration PT8H \
  --justification "Incident response: investigate data corruption"
```

See Azure AD documentation for conditional access policies.

## Shared Access Signatures (SAS)

### SAS Types Comparison

| Type | Secured By | Use Case | Security Posture |
|------|-----------|----------|------------------|
| **User Delegation SAS** | Azure AD credentials | Short-lived external access | Strong (recommended) |
| **Service SAS** | Account key | Container/blob-level access | Moderate |
| **Account SAS** | Account key | Cross-service access | Weak (avoid) |

### User Delegation SAS (Recommended)

**Secured by Azure AD** (not account keys). Revocable by revoking the user delegation key.

```bash
# Generate user delegation SAS (Azure CLI)
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry "2025-10-07T00:00:00Z" \
  --auth-mode login \
  --as-user
```

**C# example** (backend generates SAS for frontend):

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Sas;
using Azure.Identity;

public async Task<Uri> GenerateUserDelegationSasAsync(
    string accountName,
    string containerName,
    string blobName,
    TimeSpan validity)
{
    var credential = new DefaultAzureCredential();
    var blobServiceClient = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        credential);

    // Get user delegation key (valid for up to 7 days)
    var userDelegationKey = await blobServiceClient.GetUserDelegationKeyAsync(
        startsOn: DateTimeOffset.UtcNow,
        expiresOn: DateTimeOffset.UtcNow.Add(validity));

    var blobClient = blobServiceClient
        .GetBlobContainerClient(containerName)
        .GetBlobClient(blobName);

    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = containerName,
        BlobName = blobName,
        Resource = "b", // blob
        StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5), // Clock skew
        ExpiresOn = DateTimeOffset.UtcNow.Add(validity)
    };
    sasBuilder.SetPermissions(BlobSasPermissions.Read);

    var sasUri = blobClient.GenerateSasUri(sasBuilder, userDelegationKey);
    return sasUri;
}

// Usage (backend API endpoint)
[HttpGet("download/{blobName}")]
public async Task<IActionResult> GetDownloadUrl(string blobName)
{
    var sasUri = await GenerateUserDelegationSasAsync(
        accountName: "mystorageacct",
        containerName: "downloads",
        blobName: blobName,
        validity: TimeSpan.FromMinutes(15)); // Short-lived

    return Ok(new { downloadUrl = sasUri.ToString() });
}
```

**Best practices**:
- ✅ Use shortest expiry possible (minutes to hours, not days)
- ✅ Scope to specific blob or container (not account-level)
- ✅ Use `StartsOn` with clock skew buffer (-5 minutes)
- ✅ Set IP restrictions if known (`IPAddressOrRange`)
- ✅ Log SAS generation (request ID correlation)
- ❌ Never embed SAS in client-side code
- ❌ Never use long-lived SAS (> 1 day)

### Service SAS

**Secured by account key**. Use only if user delegation SAS not feasible.

```csharp
using Azure.Storage.Sas;
using Azure.Storage.Blobs;

public Uri GenerateServiceSas(
    string accountName,
    string accountKey,
    string containerName,
    string blobName,
    TimeSpan validity)
{
    var blobClient = new BlobClient(
        new Uri($"https://{accountName}.blob.core.windows.net/{containerName}/{blobName}"),
        new StorageSharedKeyCredential(accountName, accountKey));

    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = containerName,
        BlobName = blobName,
        Resource = "b",
        StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
        ExpiresOn = DateTimeOffset.UtcNow.Add(validity)
    };
    sasBuilder.SetPermissions(BlobSasPermissions.Read);

    // Optional: IP restriction
    sasBuilder.IPRange = new SasIPRange(IPAddress.Parse("203.0.113.0"), IPAddress.Parse("203.0.113.255"));

    return blobClient.GenerateSasUri(sasBuilder);
}
```

**Drawbacks**:
- Requires storing account key (secret management burden)
- Cannot revoke without regenerating account key (impacts all clients)
- No Azure AD audit trail

### Stored Access Policies

**Allows SAS revocation** by deleting the policy.

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public async Task CreateStoredAccessPolicyAsync(
    BlobContainerClient containerClient,
    string policyName)
{
    var accessPolicy = new BlobSignedIdentifier
    {
        Id = policyName,
        AccessPolicy = new BlobAccessPolicy
        {
            PolicyStartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
            PolicyExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
            Permissions = "r" // Read-only
        }
    };

    var policies = new List<BlobSignedIdentifier> { accessPolicy };
    await containerClient.SetAccessPolicyAsync(permissions: policies);
}

public Uri GenerateSasWithPolicy(
    BlobClient blobClient,
    string policyName)
{
    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = blobClient.BlobContainerName,
        BlobName = blobClient.Name,
        Resource = "b",
        Identifier = policyName // Reference stored policy
    };

    return blobClient.GenerateSasUri(sasBuilder);
}
```

**Revoke SAS**:

```csharp
// Delete stored access policy (invalidates all SAS tokens using it)
await containerClient.SetAccessPolicyAsync(permissions: new List<BlobSignedIdentifier>());
```

**Benefits**:
- ✅ Centralized SAS management
- ✅ Revoke without regenerating account keys
- ✅ Change expiry/permissions without reissuing SAS

**Limits**:
- Max 5 stored access policies per container

### SAS Security Checklist

- [ ] Use User Delegation SAS (not service/account SAS)
- [ ] Shortest expiry possible (< 1 hour preferred)
- [ ] Scope to specific resource (blob, not container/account)
- [ ] Set `StartsOn` with clock skew buffer (-5 minutes)
- [ ] Restrict to HTTPS only (default)
- [ ] Add IP restrictions if client IP known
- [ ] Use stored access policies for revocability
- [ ] Log SAS generation with correlation IDs
- [ ] Never log SAS tokens (PII/secret)
- [ ] Rotate account keys quarterly (if using service/account SAS)

## Account Keys (Avoid)

### Why Avoid Account Keys?

Account keys provide **full administrative access** to the entire storage account. Compromised keys = data breach.

**Risks**:
- ❌ No granular permissions (all or nothing)
- ❌ Never expire (must manually rotate)
- ❌ Difficult to revoke (requires regeneration, breaks all clients)
- ❌ No audit trail (cannot determine which client used key)
- ❌ Not zero-trust compliant

**When unavoidable**:
- Legacy applications that cannot be modified
- Third-party tools requiring connection strings
- Short-term development/testing (rotate frequently)

### Secure Handling of Account Keys

**DO NOT** hardcode in application code:

```csharp
// ❌ NEVER DO THIS
var connectionString = "DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...";
```

**DO** use Key Vault:

```bash
# 1. Store account key in Key Vault
ACCOUNT_KEY=$(az storage account keys list \
  --account-name mystorageacct \
  --resource-group rg-storage \
  --query "[0].value" -o tsv)

az keyvault secret set \
  --vault-name myvault \
  --name StorageAccountKey \
  --value "$ACCOUNT_KEY"

# 2. Grant App Service access to Key Vault
az keyvault set-policy \
  --name myvault \
  --object-id $PRINCIPAL_ID \
  --secret-permissions get
```

**Application code** (retrieve from Key Vault):

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var credential = new DefaultAzureCredential();
var secretClient = new SecretClient(
    new Uri("https://myvault.vault.azure.net"),
    credential);

KeyVaultSecret secret = await secretClient.GetSecretAsync("StorageAccountKey");
string accountKey = secret.Value;

// Use accountKey to construct connection string
var connectionString = $"DefaultEndpointsProtocol=https;AccountName=mystorageacct;AccountKey={accountKey}";
```

**Better**: Use connection string reference in App Service:

```bash
# App Service configuration (references Key Vault)
az webapp config appsettings set \
  --name myapp \
  --resource-group rg-apps \
  --settings StorageConnectionString="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/StorageConnectionString/)"
```

### Rotate Account Keys

**Rotate every 90 days** (compliance requirement).

```bash
# 1. Generate new key2 (key1 still active)
az storage account keys renew \
  --account-name mystorageacct \
  --resource-group rg-storage \
  --key key2

# 2. Update applications to use key2 (deployment rollout)

# 3. Verify all clients using key2

# 4. Regenerate key1
az storage account keys renew \
  --account-name mystorageacct \
  --resource-group rg-storage \
  --key key1

# 5. Update Key Vault secret
NEW_KEY=$(az storage account keys list \
  --account-name mystorageacct \
  --resource-group rg-storage \
  --query "[0].value" -o tsv)

az keyvault secret set \
  --vault-name myvault \
  --name StorageAccountKey \
  --value "$NEW_KEY"
```

### Disable Account Keys (Recommended)

**Force Azure AD authentication** by disabling shared key access:

```bash
az storage account update \
  --name mystorageacct \
  --resource-group rg-storage \
  --allow-shared-key-access false
```

**Impact**:
- ✅ Blocks all account key and connection string access
- ✅ Forces RBAC or user delegation SAS
- ❌ Breaks legacy applications using connection strings
- ❌ Some Azure services require shared key (e.g., Azure Backup, Azure Site Recovery)

**Check compatibility** before disabling:
- Azure Monitor diagnostics: ✅ Works with Azure AD
- Azure Backup: ❌ Requires shared key
- AzCopy: ✅ Works with `--login-mode`
- Azure Data Factory: ✅ Works with managed identity

## Least Privilege Principles

### 1. Scope Roles to Minimum Necessary

**BAD** (overly permissive):
```bash
# ❌ Grants access to entire subscription
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub-id}
```

**GOOD** (container-level scope):
```bash
# ✅ Grants access to specific container
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub-id}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct/blobServices/default/containers/public-assets
```

### 2. Use Read-Only Roles When Possible

If application only reads data, use `Storage Blob Data Reader`:

```bash
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub-id}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct
```

### 3. Separate Identities for Different Services

```bash
# Web API: read/write to app-data container
az role assignment create \
  --assignee $WEB_API_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope .../containers/app-data

# Background worker: read-only from app-data, write to processed-data
az role assignment create \
  --assignee $WORKER_PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope .../containers/app-data

az role assignment create \
  --assignee $WORKER_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope .../containers/processed-data
```

### 4. Time-Limited Access with PIM

For administrative tasks, use PIM instead of permanent role assignments:

```bash
# Temporary elevation (expires after 8 hours)
az pim role assignment request create \
  --scope /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --role "Storage Blob Data Owner" \
  --principal-id $MY_PRINCIPAL_ID \
  --assignment-type active \
  --duration PT8H \
  --justification "Data migration task"
```

## Threat Model and Mitigations

### Threat: Credential Theft

**Attack vector**: Attacker steals account key or SAS token.

**Mitigations**:
- ✅ Disable account keys (use RBAC)
- ✅ Use user delegation SAS (revocable via Azure AD)
- ✅ Set short SAS expiry (< 1 hour)
- ✅ IP restrict SAS tokens
- ✅ Store keys in Key Vault (if unavoidable)
- ✅ Rotate keys every 90 days

### Threat: Overprivileged Identity

**Attack vector**: Compromised service with excessive permissions.

**Mitigations**:
- ✅ Assign read-only roles when possible
- ✅ Scope roles to container/blob level
- ✅ Use separate identities per service
- ✅ Audit role assignments quarterly

### Threat: Network Exposure

**Attack vector**: Public internet access to storage account.

**Mitigations**:
- ✅ Deny public network access (`--default-action Deny`)
- ✅ Use private endpoints (see `networking-architecture.md`)
- ✅ Firewall rules for known IPs only
- ✅ Disable anonymous blob access (`--allow-blob-public-access false`)

### Threat: Data Exfiltration

**Attack vector**: Malicious insider or compromised account downloads all data.

**Mitigations**:
- ✅ Azure AD conditional access (MFA, device compliance)
- ✅ Azure Monitor alerts on large egress
- ✅ Diagnostic logging to Log Analytics
- ✅ Soft delete and versioning (recovery)
- ✅ Immutability policies (WORM)

## Compliance Considerations

### SOC 2 / ISO 27001

**Requirements**:
- Rotate credentials every 90 days
- Audit access logs quarterly
- Enforce MFA for administrative access
- Implement least privilege

**Implementation**:
```bash
# 1. Enable diagnostic logs
az monitor diagnostic-settings create \
  --name storage-audit-logs \
  --resource /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true},{"category":"StorageDelete","enabled":true}]' \
  --workspace /subscriptions/{sub}/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/myworkspace

# 2. Query audit logs (KQL)
StorageBlobLogs
| where TimeGenerated > ago(7d)
| where OperationName in ("PutBlob", "DeleteBlob")
| summarize count() by Identity, OperationName, bin(TimeGenerated, 1h)
```

### HIPAA / GDPR

**Requirements**:
- Encrypt at rest (default)
- Encrypt in transit (TLS 1.2+)
- Access control (RBAC)
- Audit logging
- Data residency (geo-replication pairing)

**Implementation**:
```bash
# Enforce TLS 1.2+
az storage account update \
  --name mystorageacct \
  --resource-group rg-storage \
  --min-tls-version TLS1_2

# Enable soft delete (data recoverability)
az storage blob service-properties delete-policy update \
  --account-name mystorageacct \
  --enable true \
  --days-retained 30

# Customer-managed keys (HIPAA requirement)
# See 05-security/encryption.md
```

### PCI DSS

**Requirements**:
- No shared credentials (account keys)
- Network segmentation (private endpoints)
- Encryption at rest and in transit
- Access logging and monitoring

**Implementation**:
```bash
# Disable account keys
az storage account update \
  --name mystorageacct \
  --resource-group rg-storage \
  --allow-shared-key-access false

# Private endpoint (no internet exposure)
# See 02-core-concepts/networking-architecture.md

# Alert on policy violations
az monitor metrics alert create \
  --name shared-key-access-detected \
  --resource-group rg-storage \
  --scopes /subscriptions/{sub}/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/mystorageacct \
  --condition "total Transactions where AuthenticationType == 'AccountKey' > 0" \
  --description "Alert if account key used (should be disabled)"
```

## Security Checklist

Before production deployment:

- [ ] RBAC configured with least privilege roles
- [ ] Managed Identity assigned to all Azure services
- [ ] Account keys disabled (`--allow-shared-key-access false`)
- [ ] If SAS required, using User Delegation SAS only
- [ ] SAS tokens expire within 1 hour
- [ ] Network access denied by default (`--default-action Deny`)
- [ ] Private endpoints configured (see `networking-architecture.md`)
- [ ] TLS 1.2+ enforced (`--min-tls-version TLS1_2`)
- [ ] Soft delete enabled (30-day retention)
- [ ] Diagnostic logs sent to Log Analytics
- [ ] Azure Monitor alerts configured
- [ ] Role assignments reviewed quarterly
- [ ] Compliance requirements met (SOC 2, HIPAA, etc.)

## Navigation

- **Previous**: `storage-accounts.md`
- **Next**: `networking-architecture.md`
- **Up**: `00-overview.md`

## See Also

- `01-quick-start/authentication-setup.md` - Hands-on RBAC setup
- `05-security/identity-authentication.md` - DefaultAzureCredential deep-dive
- `05-security/sas-patterns.md` - Advanced SAS patterns
- `05-security/network-security.md` - Firewall and private endpoints

## References

[1] "Authorize access to data in Azure Storage" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/common/authorize-data-access
[2] "Assign Azure roles for access to blob data" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/storage/blobs/assign-azure-role-data-access
[3] "Create a user delegation SAS" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/blobs/storage-blob-user-delegation-sas-create-dotnet
[4] "Prevent authorization with Shared Key" - Microsoft Learn - 2024-07 - https://learn.microsoft.com/azure/storage/common/shared-key-authorization-prevent
[5] "Azure Storage security recommendations" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/blobs/security-recommendations
