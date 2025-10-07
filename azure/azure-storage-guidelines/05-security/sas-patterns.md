# Shared Access Signature (SAS) Patterns and Best Practices

> **File Purpose**: Comprehensive guide to SAS token types, backend API patterns, stored access policies, and security best practices
> **Prerequisites**: Understanding of authentication from `identity-authentication.md` and `02-core-concepts/security-model.md`
> **Agent Use Case**: Reference when implementing time-limited external access, browser-based uploads, or third-party integrations

## Quick Context

**Use SAS tokens sparingly**. Prefer Azure AD authentication (RBAC) for all Azure-hosted services. Use SAS only when:
1. External clients cannot authenticate via Azure AD (browsers, mobile apps, third-party services)
2. Time-limited access required (short-lived tokens for uploads/downloads)
3. Granular permissions needed (read-only access to specific blob)

**Key principle**: Always use **User Delegation SAS** (secured by Azure AD) instead of Account/Service SAS (secured by account keys). Generate SAS tokens server-side via backend API, never in client code. Use shortest expiry possible (minutes to hours, not days).

## SAS Token Types Comparison

### Decision Matrix

| SAS Type | Secured By | Revocability | Scope | Use Case | Security Rating |
|----------|------------|--------------|-------|----------|-----------------|
| **User Delegation SAS** | Azure AD credentials | High (via delegation key) | Blob/container | External clients, browser uploads | Strong (recommended) |
| **Service SAS** | Account key | Low (requires key rotation) | Blob/table/queue/file | Legacy compatibility | Moderate |
| **Account SAS** | Account key | Low (requires key rotation) | Cross-service, account-level | Rarely needed | Weak (avoid) |

### Detailed Comparison

**User Delegation SAS**:
- ✅ Secured by Azure AD (not account keys)
- ✅ Revocable by revoking user delegation key (up to 7 days)
- ✅ Audit trail via Azure AD logs
- ✅ Supports all blob permissions
- ❌ Blob service only (not table/queue/file)
- ❌ Requires RBAC role (Storage Blob Delegator)

**Service SAS**:
- ⚠️ Secured by account key (secret management burden)
- ❌ Cannot revoke without regenerating account key (impacts all clients)
- ⚠️ Supports all services (blob, table, queue, file)
- ⚠️ Can use stored access policies for revocability

**Account SAS**:
- ❌ Secured by account key
- ❌ Broadest permissions (cross-service)
- ❌ Difficult to scope (account-level)
- ❌ Avoid in production

### When to Use Each Type

| Scenario | Recommended SAS Type | Rationale |
|----------|---------------------|-----------|
| Browser-based file upload | User Delegation SAS | Frontend gets token from backend, uploads directly to storage |
| Mobile app downloads | User Delegation SAS | Backend generates token after auth check |
| Third-party integration (temporary) | User Delegation SAS + IP restriction | Time-boxed access, revocable |
| Table Storage access (external) | Service SAS + Stored Access Policy | User delegation SAS not available for tables |
| Cross-service operations | Account SAS (last resort) | Use only if service SAS insufficient |
| CDN origin | Service SAS (long-lived) + Stored Access Policy | CDN requires persistent access |

## User Delegation SAS (Recommended)

### How User Delegation SAS Works

1. Application authenticates to Azure AD (managed identity or service principal)
2. Application requests **user delegation key** from Azure Storage (valid up to 7 days)
3. Application generates SAS token using delegation key (not account key)
4. Client uses SAS token to access storage
5. Storage validates SAS signature against delegation key

**Security benefits**:
- Account keys never used (zero account key exposure)
- Revocable by revoking delegation key
- Azure AD audit trail (who generated SAS)
- Least privilege (requires Storage Blob Delegator role)

### Prerequisites: RBAC Setup

**Backend service needs TWO roles**:

1. **Storage Blob Delegator** - To obtain user delegation key
2. **Storage Blob Data Contributor** - To generate SAS for read/write operations

```bash
# Variables
PRINCIPAL_ID="..."  # Managed identity or service principal
STORAGE_ACCOUNT="mystorageacct"
RG_STORAGE="rg-storage"

# 1. Assign Storage Blob Delegator role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Delegator" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT

# 2. Assign Storage Blob Data Contributor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/$RG_STORAGE/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT
```

**Role GUIDs** (for Bicep):
- Storage Blob Delegator: `db58b8e5-c6ad-4a2a-8342-4190687cbf4a`
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`

### Generate User Delegation SAS: Azure CLI

```bash
# 1. Login with Azure AD
az login

# 2. Generate user delegation SAS for blob
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name uploads \
  --name document.pdf \
  --permissions r \
  --expiry "2025-10-07T00:00:00Z" \
  --auth-mode login \
  --as-user \
  --https-only

# Output: SAS token (query string)
# sv=2023-01-03&sr=b&sig=...&se=2025-10-07T00:00:00Z&sp=r

# 3. Use with full URL
# https://mystorageacct.blob.core.windows.net/uploads/document.pdf?sv=2023-01-03&sr=b&sig=...
```

**Parameters**:
- `--permissions`: `r` (read), `w` (write), `d` (delete), `l` (list), `a` (add), `c` (create)
- `--expiry`: ISO 8601 timestamp (UTC)
- `--auth-mode login`: Use Azure AD (not account key)
- `--as-user`: Generate user delegation SAS
- `--https-only`: Enforce HTTPS (default, recommended)

**Container-level SAS** (list and read all blobs):

```bash
az storage container generate-sas \
  --account-name mystorageacct \
  --name uploads \
  --permissions rl \
  --expiry "2025-10-07T00:00:00Z" \
  --auth-mode login \
  --as-user \
  --https-only
```

### Generate User Delegation SAS: C# Backend API

**Complete backend API implementation**:

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Sas;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class SasController : ControllerBase
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<SasController> _logger;
    private readonly IConfiguration _configuration;

    public SasController(
        BlobServiceClient blobServiceClient,
        ILogger<SasController> logger,
        IConfiguration configuration)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
        _configuration = configuration;
    }

    /// <summary>
    /// Generate user delegation SAS for blob upload
    /// </summary>
    [HttpPost("upload-url")]
    public async Task<IActionResult> GetUploadUrl([FromBody] UploadRequest request)
    {
        try
        {
            // 1. Validate request (authentication, authorization, input validation)
            if (string.IsNullOrWhiteSpace(request.FileName))
            {
                return BadRequest("FileName is required");
            }

            // 2. Sanitize filename (prevent path traversal)
            var sanitizedFileName = Path.GetFileName(request.FileName);

            // 3. Get user delegation key (valid for 1 hour)
            var delegationKey = await _blobServiceClient.GetUserDelegationKeyAsync(
                startsOn: DateTimeOffset.UtcNow,
                expiresOn: DateTimeOffset.UtcNow.AddHours(1));

            // 4. Configure SAS token
            var blobClient = _blobServiceClient
                .GetBlobContainerClient("uploads")
                .GetBlobClient(sanitizedFileName);

            var sasBuilder = new BlobSasBuilder
            {
                BlobContainerName = "uploads",
                BlobName = sanitizedFileName,
                Resource = "b", // b = blob, c = container
                StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5), // Clock skew buffer
                ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15), // Short-lived token
                Protocol = SasProtocol.Https // HTTPS only
            };

            // Set permissions
            sasBuilder.SetPermissions(BlobSasPermissions.Write | BlobSasPermissions.Create);

            // Optional: IP restriction (if known)
            if (!string.IsNullOrEmpty(request.ClientIp))
            {
                sasBuilder.IPRange = new SasIPRange(IPAddress.Parse(request.ClientIp));
            }

            // 5. Generate SAS URI
            var sasUri = blobClient.GenerateSasUri(sasBuilder, delegationKey.Value);

            _logger.LogInformation(
                "Generated upload SAS for user {UserId}, file {FileName}, expires {Expiry}",
                User.Identity?.Name,
                sanitizedFileName,
                sasBuilder.ExpiresOn);

            return Ok(new
            {
                uploadUrl = sasUri.ToString(),
                expiresAt = sasBuilder.ExpiresOn
            });
        }
        catch (Azure.RequestFailedException ex)
        {
            _logger.LogError(ex,
                "Failed to generate SAS token. Status: {Status}, ErrorCode: {ErrorCode}",
                ex.Status,
                ex.ErrorCode);
            return StatusCode(500, "Failed to generate upload URL");
        }
    }

    /// <summary>
    /// Generate user delegation SAS for blob download
    /// </summary>
    [HttpGet("download-url/{containerName}/{blobName}")]
    public async Task<IActionResult> GetDownloadUrl(string containerName, string blobName)
    {
        try
        {
            // 1. Validate user has permission to access this blob
            // (Add your authorization logic here)

            // 2. Get user delegation key
            var delegationKey = await _blobServiceClient.GetUserDelegationKeyAsync(
                startsOn: DateTimeOffset.UtcNow,
                expiresOn: DateTimeOffset.UtcNow.AddHours(1));

            // 3. Configure SAS token
            var blobClient = _blobServiceClient
                .GetBlobContainerClient(containerName)
                .GetBlobClient(blobName);

            var sasBuilder = new BlobSasBuilder
            {
                BlobContainerName = containerName,
                BlobName = blobName,
                Resource = "b",
                StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
                ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15), // Short-lived
                Protocol = SasProtocol.Https
            };

            // Read-only permission
            sasBuilder.SetPermissions(BlobSasPermissions.Read);

            // Optional: Set content disposition (force download)
            sasBuilder.ContentDisposition = $"attachment; filename=\"{blobName}\"";

            // 4. Generate SAS URI
            var sasUri = blobClient.GenerateSasUri(sasBuilder, delegationKey.Value);

            _logger.LogInformation(
                "Generated download SAS for user {UserId}, blob {BlobName}",
                User.Identity?.Name,
                blobName);

            return Ok(new
            {
                downloadUrl = sasUri.ToString(),
                expiresAt = sasBuilder.ExpiresOn
            });
        }
        catch (Azure.RequestFailedException ex) when (ex.Status == 404)
        {
            return NotFound("Blob not found");
        }
        catch (Azure.RequestFailedException ex)
        {
            _logger.LogError(ex, "Failed to generate download SAS");
            return StatusCode(500, "Failed to generate download URL");
        }
    }

    /// <summary>
    /// Generate container-level SAS for listing blobs
    /// </summary>
    [HttpGet("list-url/{containerName}")]
    public async Task<IActionResult> GetListUrl(string containerName)
    {
        try
        {
            // Get user delegation key
            var delegationKey = await _blobServiceClient.GetUserDelegationKeyAsync(
                startsOn: DateTimeOffset.UtcNow,
                expiresOn: DateTimeOffset.UtcNow.AddHours(1));

            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);

            var sasBuilder = new BlobSasBuilder
            {
                BlobContainerName = containerName,
                Resource = "c", // Container-level SAS
                StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
                ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(30),
                Protocol = SasProtocol.Https
            };

            // List and Read permissions
            sasBuilder.SetPermissions(BlobContainerSasPermissions.List | BlobContainerSasPermissions.Read);

            var sasUri = containerClient.GenerateSasUri(sasBuilder, delegationKey.Value);

            return Ok(new
            {
                listUrl = sasUri.ToString(),
                expiresAt = sasBuilder.ExpiresOn
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to generate list SAS");
            return StatusCode(500, "Failed to generate list URL");
        }
    }
}

public record UploadRequest(string FileName, string? ClientIp);
```

**Dependency injection setup** (Program.cs):

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

var builder = WebApplication.CreateBuilder(args);

// Register BlobServiceClient with DefaultAzureCredential
builder.Services.AddSingleton(sp =>
{
    var credential = new DefaultAzureCredential();
    var storageAccountName = builder.Configuration["StorageAccountName"];

    return new BlobServiceClient(
        new Uri($"https://{storageAccountName}.blob.core.windows.net"),
        credential);
});

builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```

### Frontend Implementation: JavaScript Upload

**React example** (fetch SAS from backend, upload to storage):

```javascript
import React, { useState } from 'react';

function FileUploader() {
  const [file, setFile] = useState(null);
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);

  const handleUpload = async () => {
    if (!file) return;

    try {
      setUploading(true);

      // 1. Get SAS token from backend
      const response = await fetch('/api/sas/upload-url', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${accessToken}` // Your auth token
        },
        body: JSON.stringify({
          fileName: file.name,
          clientIp: null // Optional
        })
      });

      const { uploadUrl, expiresAt } = await response.json();

      // 2. Upload directly to Azure Storage using SAS
      const uploadResponse = await fetch(uploadUrl, {
        method: 'PUT',
        headers: {
          'x-ms-blob-type': 'BlockBlob',
          'Content-Type': file.type
        },
        body: file
      });

      if (uploadResponse.ok) {
        console.log('Upload successful');
      } else {
        const error = await uploadResponse.text();
        console.error('Upload failed:', error);
      }
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div>
      <input
        type="file"
        onChange={(e) => setFile(e.target.files[0])}
      />
      <button onClick={handleUpload} disabled={!file || uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
    </div>
  );
}
```

**Azure Storage JavaScript SDK** (alternative to fetch):

```javascript
import { BlobServiceClient } from '@azure/storage-blob';

async function uploadWithSdk(file) {
  // 1. Get SAS from backend
  const response = await fetch('/api/sas/upload-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ fileName: file.name })
  });

  const { uploadUrl } = await response.json();

  // 2. Upload using Azure SDK
  const blobClient = new BlobClient(uploadUrl);
  const blockBlobClient = blobClient.getBlockBlobClient();

  await blockBlobClient.uploadData(file, {
    blobHTTPHeaders: {
      blobContentType: file.type
    },
    onProgress: (ev) => {
      console.log(`Upload progress: ${ev.loadedBytes} / ${file.size}`);
    }
  });

  console.log('Upload complete');
}
```

### Security Best Practices for User Delegation SAS

**1. Shortest expiry possible**:

```csharp
// ❌ BAD: 7 days
ExpiresOn = DateTimeOffset.UtcNow.AddDays(7)

// ✅ GOOD: 15 minutes
ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)
```

**2. Clock skew buffer**:

```csharp
// ✅ Always set StartsOn 5 minutes in past
StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)
```

**3. Minimal permissions**:

```csharp
// ❌ BAD: Full permissions
sasBuilder.SetPermissions(BlobSasPermissions.All);

// ✅ GOOD: Read-only
sasBuilder.SetPermissions(BlobSasPermissions.Read);

// ✅ GOOD: Write-only for uploads
sasBuilder.SetPermissions(BlobSasPermissions.Write | BlobSasPermissions.Create);
```

**4. IP restrictions** (if known):

```csharp
// Restrict to specific IP
sasBuilder.IPRange = new SasIPRange(IPAddress.Parse("203.0.113.42"));

// Restrict to IP range
sasBuilder.IPRange = new SasIPRange(
    IPAddress.Parse("203.0.113.0"),
    IPAddress.Parse("203.0.113.255"));
```

**5. HTTPS only** (always):

```csharp
sasBuilder.Protocol = SasProtocol.Https; // Enforce TLS
```

**6. Content disposition** (force download):

```csharp
// Prevent browser from rendering (XSS protection)
sasBuilder.ContentDisposition = "attachment; filename=\"document.pdf\"";
```

**7. Logging and monitoring**:

```csharp
_logger.LogInformation(
    "SAS generated: User={User}, Blob={Blob}, Permissions={Permissions}, Expiry={Expiry}",
    User.Identity?.Name,
    blobName,
    permissions,
    sasBuilder.ExpiresOn);
```

## Service SAS (Legacy Pattern)

### When to Use Service SAS

**Only use if**:
- User delegation SAS not available (Table Storage, Queue Storage, File Storage)
- Legacy compatibility required
- Using stored access policies for revocability

**Avoid if possible** - prefer user delegation SAS for blob storage.

### Generate Service SAS: C# Backend

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Sas;
using Azure.Storage;

public class ServiceSasGenerator
{
    private readonly string _accountName;
    private readonly string _accountKey;
    private readonly ILogger<ServiceSasGenerator> _logger;

    public ServiceSasGenerator(
        IConfiguration configuration,
        ILogger<ServiceSasGenerator> logger)
    {
        // ⚠️ Account key should be stored in Key Vault
        _accountName = configuration["StorageAccountName"];
        _accountKey = configuration["StorageAccountKey"]; // From Key Vault
        _logger = logger;
    }

    public Uri GenerateServiceSas(
        string containerName,
        string blobName,
        BlobSasPermissions permissions,
        TimeSpan validity)
    {
        // Create BlobClient with account key
        var credential = new StorageSharedKeyCredential(_accountName, _accountKey);
        var blobUri = new Uri($"https://{_accountName}.blob.core.windows.net/{containerName}/{blobName}");
        var blobClient = new BlobClient(blobUri, credential);

        // Configure SAS
        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerName,
            BlobName = blobName,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
            ExpiresOn = DateTimeOffset.UtcNow.Add(validity),
            Protocol = SasProtocol.Https
        };

        sasBuilder.SetPermissions(permissions);

        // Optional: IP restriction
        // sasBuilder.IPRange = new SasIPRange(IPAddress.Parse("203.0.113.42"));

        // Generate SAS
        var sasUri = blobClient.GenerateSasUri(sasBuilder);

        _logger.LogInformation(
            "Generated service SAS for blob {Blob}, expires {Expiry}",
            blobName,
            sasBuilder.ExpiresOn);

        return sasUri;
    }
}
```

**Key Vault integration** (secure account key):

```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Identity;

// Program.cs
builder.Services.AddSingleton<ServiceSasGenerator>(sp =>
{
    var configuration = sp.GetRequiredService<IConfiguration>();
    var keyVaultUrl = configuration["KeyVaultUrl"];

    // Get account key from Key Vault
    var secretClient = new SecretClient(
        new Uri(keyVaultUrl),
        new DefaultAzureCredential());

    var secret = secretClient.GetSecret("StorageAccountKey");

    // Use in-memory configuration
    var inMemoryConfig = new Dictionary<string, string>
    {
        ["StorageAccountName"] = configuration["StorageAccountName"],
        ["StorageAccountKey"] = secret.Value.Value
    };

    var config = new ConfigurationBuilder()
        .AddInMemoryCollection(inMemoryConfig)
        .Build();

    return new ServiceSasGenerator(config, sp.GetRequiredService<ILogger<ServiceSasGenerator>>());
});
```

### Service SAS for Table Storage

```csharp
using Azure.Data.Tables;
using Azure.Data.Tables.Sas;
using Azure.Storage;

public Uri GenerateTableServiceSas(
    string tableName,
    TableSasPermissions permissions,
    TimeSpan validity)
{
    var credential = new TableSharedKeyCredential(_accountName, _accountKey);
    var tableUri = new Uri($"https://{_accountName}.table.core.windows.net/{tableName}");
    var tableClient = new TableClient(tableUri, credential);

    var sasBuilder = new TableSasBuilder(
        tableName,
        permissions,
        expiresOn: DateTimeOffset.UtcNow.Add(validity))
    {
        StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
        Protocol = SasProtocol.Https
    };

    var sasUri = tableClient.GenerateSasUri(sasBuilder);
    return sasUri;
}

// Usage
var sasUri = GenerateTableServiceSas(
    "Users",
    TableSasPermissions.Read | TableSasPermissions.Add,
    TimeSpan.FromMinutes(30));
```

## Stored Access Policies

### What Are Stored Access Policies?

**Stored access policies** define SAS parameters (permissions, start/expiry) at the container level. SAS tokens reference the policy by ID. **Revoking the policy invalidates all SAS tokens using it**.

**Benefits**:
- ✅ Revoke SAS without regenerating account keys
- ✅ Change expiry/permissions without reissuing tokens
- ✅ Centralized SAS management

**Limitations**:
- Max 5 stored access policies per container
- Only works with Service SAS (not User Delegation SAS)

### Create Stored Access Policy: Azure CLI

```bash
# 1. Create stored access policy on container
az storage container policy create \
  --account-name mystorageacct \
  --container-name uploads \
  --name upload-policy \
  --permissions rw \
  --start "2025-10-06T00:00:00Z" \
  --expiry "2025-10-07T00:00:00Z" \
  --account-key $ACCOUNT_KEY

# 2. Generate SAS using stored policy
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name uploads \
  --name document.pdf \
  --policy-name upload-policy \
  --account-key $ACCOUNT_KEY

# 3. Revoke all SAS tokens using this policy
az storage container policy delete \
  --account-name mystorageacct \
  --container-name uploads \
  --name upload-policy \
  --account-key $ACCOUNT_KEY
```

### Create Stored Access Policy: C#

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public async Task CreateStoredAccessPolicyAsync(
    string containerName,
    string policyName,
    BlobSasPermissions permissions,
    TimeSpan validity)
{
    var credential = new StorageSharedKeyCredential(_accountName, _accountKey);
    var containerClient = new BlobContainerClient(
        new Uri($"https://{_accountName}.blob.core.windows.net/{containerName}"),
        credential);

    // Get existing policies
    var accessPolicy = await containerClient.GetAccessPolicyAsync();
    var policies = accessPolicy.Value.SignedIdentifiers.ToList();

    // Add new policy
    var newPolicy = new BlobSignedIdentifier
    {
        Id = policyName,
        AccessPolicy = new BlobAccessPolicy
        {
            PolicyStartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
            PolicyExpiresOn = DateTimeOffset.UtcNow.Add(validity),
            Permissions = permissions.ToString()
        }
    };

    policies.Add(newPolicy);

    // Update container with new policies
    await containerClient.SetAccessPolicyAsync(permissions: policies);

    _logger.LogInformation(
        "Created stored access policy {PolicyName} on container {Container}",
        policyName,
        containerName);
}

// Generate SAS using stored policy
public Uri GenerateSasWithPolicy(
    string containerName,
    string blobName,
    string policyName)
{
    var credential = new StorageSharedKeyCredential(_accountName, _accountKey);
    var blobClient = new BlobClient(
        new Uri($"https://{_accountName}.blob.core.windows.net/{containerName}/{blobName}"),
        credential);

    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = containerName,
        BlobName = blobName,
        Resource = "b",
        Identifier = policyName // Reference stored policy
    };

    var sasUri = blobClient.GenerateSasUri(sasBuilder);
    return sasUri;
}

// Revoke stored access policy
public async Task RevokeStoredAccessPolicyAsync(
    string containerName,
    string policyName)
{
    var credential = new StorageSharedKeyCredential(_accountName, _accountKey);
    var containerClient = new BlobContainerClient(
        new Uri($"https://{_accountName}.blob.core.windows.net/{containerName}"),
        credential);

    // Get existing policies
    var accessPolicy = await containerClient.GetAccessPolicyAsync();
    var policies = accessPolicy.Value.SignedIdentifiers
        .Where(p => p.Id != policyName) // Remove policy
        .ToList();

    // Update container
    await containerClient.SetAccessPolicyAsync(permissions: policies);

    _logger.LogWarning(
        "Revoked stored access policy {PolicyName} on container {Container}. All SAS tokens using this policy are now invalid.",
        policyName,
        containerName);
}
```

### Stored Access Policy Best Practices

**1. Policy naming convention**:

```csharp
// Use descriptive names with versioning
var policyName = "upload-v1-2025-10";
```

**2. Rotation strategy**:

```csharp
// Create new policy before old one expires
await CreateStoredAccessPolicyAsync("uploads", "upload-v2", permissions, TimeSpan.FromDays(30));

// Wait for clients to migrate to v2

// Delete old policy
await RevokeStoredAccessPolicyAsync("uploads", "upload-v1");
```

**3. Monitor policy usage**:

```csharp
// Query storage logs for policy usage
StorageBlobLogs
| where TimeGenerated > ago(7d)
| where Properties.sasPolicy == "upload-v1"
| summarize count() by bin(TimeGenerated, 1h)
```

## Backend API Patterns for SAS Generation

### Pattern 1: API Endpoint per Operation

```csharp
[ApiController]
[Route("api/files")]
public class FileController : ControllerBase
{
    // Upload endpoint
    [HttpPost("upload-url")]
    [Authorize] // Require authentication
    public async Task<IActionResult> GetUploadUrl([FromBody] UploadRequest request)
    {
        // 1. Authorize user (check permissions)
        if (!await _authService.CanUploadAsync(User))
        {
            return Forbid();
        }

        // 2. Validate input
        if (!IsValidFileName(request.FileName))
        {
            return BadRequest("Invalid file name");
        }

        // 3. Generate SAS
        var sasUri = await _sasService.GenerateUploadSasAsync(
            containerName: "uploads",
            blobName: request.FileName,
            validity: TimeSpan.FromMinutes(15));

        return Ok(new { uploadUrl = sasUri });
    }

    // Download endpoint
    [HttpGet("{blobId}/download-url")]
    [Authorize]
    public async Task<IActionResult> GetDownloadUrl(string blobId)
    {
        // 1. Get blob metadata from database
        var blob = await _blobRepository.GetByIdAsync(blobId);
        if (blob == null)
        {
            return NotFound();
        }

        // 2. Authorize user
        if (!await _authService.CanDownloadAsync(User, blob))
        {
            return Forbid();
        }

        // 3. Generate SAS
        var sasUri = await _sasService.GenerateDownloadSasAsync(
            containerName: blob.ContainerName,
            blobName: blob.BlobName,
            validity: TimeSpan.FromMinutes(15));

        return Ok(new { downloadUrl = sasUri });
    }
}
```

### Pattern 2: Pre-signed Upload with Metadata

```csharp
[HttpPost("files/initiate-upload")]
[Authorize]
public async Task<IActionResult> InitiateUpload([FromBody] InitiateUploadRequest request)
{
    // 1. Validate request
    if (!IsValidContentType(request.ContentType))
    {
        return BadRequest("Invalid content type");
    }

    if (request.FileSizeBytes > 100 * 1024 * 1024) // 100 MB limit
    {
        return BadRequest("File too large");
    }

    // 2. Create upload record in database
    var uploadId = Guid.NewGuid().ToString();
    var blobName = $"{uploadId}/{request.FileName}";

    await _uploadRepository.CreateAsync(new Upload
    {
        Id = uploadId,
        UserId = User.Identity.Name,
        FileName = request.FileName,
        BlobName = blobName,
        Status = UploadStatus.Pending,
        CreatedAt = DateTimeOffset.UtcNow
    });

    // 3. Generate SAS for upload
    var sasUri = await _sasService.GenerateUploadSasAsync(
        containerName: "uploads",
        blobName: blobName,
        validity: TimeSpan.FromMinutes(30));

    return Ok(new
    {
        uploadId,
        uploadUrl = sasUri,
        expiresAt = DateTimeOffset.UtcNow.AddMinutes(30)
    });
}

[HttpPost("files/{uploadId}/complete")]
[Authorize]
public async Task<IActionResult> CompleteUpload(string uploadId)
{
    // 1. Verify upload record exists
    var upload = await _uploadRepository.GetByIdAsync(uploadId);
    if (upload == null || upload.UserId != User.Identity.Name)
    {
        return NotFound();
    }

    // 2. Verify blob exists in storage
    var blobClient = _blobServiceClient
        .GetBlobContainerClient("uploads")
        .GetBlobClient(upload.BlobName);

    if (!await blobClient.ExistsAsync())
    {
        return BadRequest("Upload not found in storage");
    }

    // 3. Update upload status
    upload.Status = UploadStatus.Completed;
    upload.CompletedAt = DateTimeOffset.UtcNow;
    await _uploadRepository.UpdateAsync(upload);

    return Ok(new { uploadId, status = "completed" });
}
```

### Pattern 3: Time-Limited Access with Usage Tracking

```csharp
public class SasService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ISasUsageRepository _sasUsageRepository;

    public async Task<Uri> GenerateTrackedSasAsync(
        string userId,
        string containerName,
        string blobName,
        string purpose,
        TimeSpan validity)
    {
        // 1. Generate SAS
        var delegationKey = await _blobServiceClient.GetUserDelegationKeyAsync(
            DateTimeOffset.UtcNow,
            DateTimeOffset.UtcNow.AddHours(1));

        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerName,
            BlobName = blobName,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5),
            ExpiresOn = DateTimeOffset.UtcNow.Add(validity),
            Protocol = SasProtocol.Https
        };

        sasBuilder.SetPermissions(BlobSasPermissions.Read);

        var sasUri = blobClient.GenerateSasUri(sasBuilder, delegationKey.Value);

        // 2. Track SAS generation
        await _sasUsageRepository.CreateAsync(new SasUsage
        {
            UserId = userId,
            BlobName = blobName,
            Purpose = purpose,
            GeneratedAt = DateTimeOffset.UtcNow,
            ExpiresAt = sasBuilder.ExpiresOn
        });

        return sasUri;
    }
}

// Query SAS usage
public async Task<IActionResult> GetSasUsageReport(DateTimeOffset since)
{
    var usage = await _sasUsageRepository.GetUsageSinceAsync(since);

    return Ok(usage.GroupBy(u => u.UserId)
        .Select(g => new
        {
            UserId = g.Key,
            TotalRequests = g.Count(),
            UniqueBlobs = g.Select(u => u.BlobName).Distinct().Count()
        }));
}
```

## SAS vs RBAC Decision Matrix

| Scenario | Recommended Approach | Rationale |
|----------|---------------------|-----------|
| Azure App Service → Storage | RBAC (Managed Identity) | No credential management, automatic rotation |
| Azure Function → Storage | RBAC (Managed Identity) | Same as App Service |
| AKS Pod → Storage | RBAC (Workload Identity) | Kubernetes-native identity federation |
| Browser → Storage (upload) | User Delegation SAS (backend-generated) | Browser cannot use Azure AD directly |
| Mobile App → Storage | User Delegation SAS (after backend auth) | Backend validates user, generates SAS |
| Third-party SaaS → Storage | User Delegation SAS + IP restriction | Time-limited, revocable access |
| On-premises app → Storage | RBAC (Service Principal) | Azure AD integration, audit trail |
| CDN → Storage (origin) | Service SAS (long-lived) + Stored Access Policy | CDN requires persistent URL |
| Data migration tool → Storage | RBAC (Azure CLI) or User Delegation SAS | Prefer Azure AD when possible |
| Public read access | Container-level SAS (anonymous) or Public Access | Use SAS for auditing; Public Access for CDN |

## Security Best Practices Checklist

### SAS Generation

- [ ] Use User Delegation SAS for blob storage (not Service/Account SAS)
- [ ] Generate SAS server-side via backend API (never in client code)
- [ ] Use shortest expiry possible (minutes to hours, < 1 day)
- [ ] Set `StartsOn` with clock skew buffer (-5 minutes)
- [ ] Enforce HTTPS only (`Protocol = SasProtocol.Https`)
- [ ] Scope to specific blob (not container/account) when possible
- [ ] Use minimal permissions (read-only for downloads, write-only for uploads)
- [ ] Add IP restrictions if client IP known
- [ ] Set content disposition for downloads (prevent XSS)

### SAS Usage Tracking

- [ ] Log SAS generation events (user, blob, expiry, permissions)
- [ ] Never log SAS tokens (contains signature)
- [ ] Monitor SAS usage via storage diagnostic logs
- [ ] Alert on excessive SAS generation (potential abuse)
- [ ] Track SAS expiry and revoke unused policies

### Account Key Management (if using Service SAS)

- [ ] Store account keys in Key Vault (never hardcode)
- [ ] Use managed identity to access Key Vault
- [ ] Rotate account keys every 90 days
- [ ] Use stored access policies for revocability
- [ ] Disable account keys if possible (`--allow-shared-key-access false`)

### Authorization

- [ ] Authenticate users before generating SAS
- [ ] Authorize user access to specific resources
- [ ] Validate and sanitize file names (prevent path traversal)
- [ ] Enforce file size limits
- [ ] Validate content types (prevent malicious uploads)
- [ ] Rate limit SAS generation endpoints

### Monitoring

- [ ] Enable diagnostic logs for blob/table storage
- [ ] Query logs for SAS usage patterns
- [ ] Alert on 403 errors (authorization failures)
- [ ] Review SAS expiry times (detect long-lived tokens)
- [ ] Audit stored access policies quarterly

## Navigation

- **Previous**: `identity-authentication.md`
- **Next**: `encryption.md`
- **Up**: `00-overview.md`

## See Also

- `identity-authentication.md` - Managed Identity and RBAC setup
- `02-core-concepts/security-model.md` - Authentication methods overview
- `network-security.md` - IP restrictions and firewall rules
- `03-blob-storage/blob-implementation.md` - Blob upload/download with SAS

## References

[1] "Create a user delegation SAS" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/blobs/storage-blob-user-delegation-sas-create-dotnet
[2] "Create a service SAS for a blob" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/storage/blobs/sas-service-create-dotnet
[3] "Create an account SAS" - Microsoft Learn - 2024-08 - https://learn.microsoft.com/azure/storage/common/storage-account-sas-create-dotnet
[4] "Define stored access policy" - Microsoft Learn - 2024-07 - https://learn.microsoft.com/rest/api/storageservices/define-stored-access-policy
[5] "Best practices for SAS" - Microsoft Learn - 2024-09 - https://learn.microsoft.com/azure/storage/common/storage-sas-overview#best-practices-when-using-sas
[6] "Prevent authorization with Shared Key" - Microsoft Learn - 2024-07 - https://learn.microsoft.com/azure/storage/common/shared-key-authorization-prevent
