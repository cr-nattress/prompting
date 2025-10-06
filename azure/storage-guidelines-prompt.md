Prompt: Deep-Research Implementation Guidelines & Best Practices (Azure Blob Storage & Table Storage)

Role: You are a principal cloud architect and standards author.
Mission: Produce a single, publication-ready Markdown document that defines implementation guidelines and industry best practices for Azure Blob Storage and Azure Table Storage, including runnable code (C#/.NET 8), infra snippets (CLI/Bicep), operations guidance, and references.

Scope (cover each item thoroughly)

Accounts & Redundancy: LRS/ZRS/GRS/GZRS, RA-* variants; hierarchical namespace (ADLS Gen2) trade-offs.

Security: RBAC vs. SAS vs. Account Keys; Managed Identity; CMK encryption; double encryption; immutability (time-based/legal hold); soft delete; private endpoints; network rules; firewall & VNET service endpoints.

Authentication & Authorization: DefaultAzureCredential chain; per-resource RBAC; SAS best practices (scopes, IP ranges, expiry); stored access policies; user delegation SAS.

Blob Types & Patterns: Block blobs (streaming, chunking), Append blobs (logs), Page blobs (VM disks); tiering (Hot/Cool/Archive); rehydration; versioning; snapshots; lifecycle management.

Table Storage Design: Partition/RowKey strategy; hot partition avoidance; batch semantics; query shapes; secondary indexes pattern; compare with Cosmos DB Table API.

Data Access Layer: SDKs (Azure.Storage.Blobs, Azure.Data.Tables, Azure.Identity), retries, idempotency, concurrency (ETags, If-Match/If-None-Match), async streaming, upload/download resilience, client pooling.

Performance: Throughput targets, parallelism, chunk sizes, TCP reuse, gzip/deflate, CDN in front of blobs, ADLS Gen2 for analytics.

Observability: Diagnostics settings → Log Analytics; Storage metrics; request IDs; end-to-end correlation; Azure Monitor alerts; access logs.

Governance & Operations: Naming standards; tagging; lifecycle policies; cost controls (tombstoning vs. hard delete, archive tier), data retention; backup/restore patterns.

Networking: Private Endpoints, DNS zones, restricted egress; SAS through private endpoints; CORS for browser access.

Testing & Local Dev: Azurite emulator; local SAS; contract tests; perf tests.

CI/CD: GitHub Actions; environment promotion; keyless auth (OIDC to Azure); drift detection.

Compliance: Encryption at rest, TLS 1.2+, audit trails; immutability for WORM requirements.

Output Contract

Deliverable: One cohesive Markdown guide.

Audience: Senior engineers & tech leads.

Tone: Clear, pragmatic, opinionated where trade-offs matter; justify choices.

Citations: Prefer primary sources (Microsoft Learn/Docs, Azure Architecture Center). Use inline footnotes like [1] and include a References section with Title — Publisher — Date — URL.

Code: Must run on .NET 8. Tag blocks with csharp, bash, json, bicep, yaml. Provide brief Node/Python hints where relevant.

Required Structure (use this exact order)

Title, Author, Date (ISO), Abstract

Table of Contents (with anchors)

Principles & Non-Goals (security by default, least privilege, immutability options, cost awareness, testability, observability)

Reference Architecture

Mermaid diagram: Client → (Private Endpoint) → Storage Account (Blob/Table) → Log Analytics/Monitor; optional CDN; VNET layout.

Decision matrix: Blob tiering, redundancy, ADLS Gen2 yes/no, Table vs. Cosmos Table API.

Provisioning

Azure CLI commands to create a Storage Account with recommended flags (HNS on/off, redundancy, TLS min version, soft delete).

Bicep (or Terraform outline) for idempotent infra (account, containers, tables, private endpoints, diagnostic settings).

Security & Identity

RBAC roles to assign (Storage Blob Data Contributor/Reader, Table Data roles).

Managed Identity flow vs. SAS; user delegation SAS; stored access policy examples.

Network: private endpoints, deny public network access, firewall rules, CORS sample.

Encryption: SSE with Microsoft-managed keys vs. CMK (Key Vault), immutability policies.

Blob Storage — Implementation

Containers, naming, folder conventions (if HNS).

Upload/download patterns: streaming, chunked, parallel; content type & metadata.

Concurrency: ETags, conditional headers; optimistic concurrency example.

Versioning, snapshots, soft delete; lifecycle rules (JSON).

Large objects: block sizes, parallel degree, resume uploads.

Table Storage — Implementation

Schema-less modeling; PartitionKey/RowKey design; monotonic keys; composite keys.

CRUD, batch operations (same partition), pagination (continuation tokens).

Queries: filters, projection; secondary index strategies (maintain your own).

Idempotency and concurrency with ETags.

Data Access Layer (C#)

Reusable clients with DefaultAzureCredential; DI setup; retry policies; timeouts.

Error handling taxonomy (429/5xx backoff, network timeouts, auth failures).

Abstractions for testability; Azurite switch for local.

Observability & Ops

Diagnostic settings to Log Analytics; sample KQL queries; request ID propagation.

Alerts (latency, throttling, auth failures, capacity).

Performance Playbook

Guidance tables for parallelism, chunk sizes, typical throughput.

CDN/offload patterns; index partition strategies; hot partition mitigation.

Testing & Local Dev

Azurite docker/local; seeded data; contract tests; perf smoke tests.

CI/CD

GitHub Actions with OIDC to Azure; build, test, security scan, deploy infra + app; promote by environment; source map & symbols (if applicable).

Checklists

Security, Performance, Resilience, Operations.

References (primary sources, dates, URLs)

Must-Include Artifacts (runnable / copy-paste)
Azure CLI (provisioning)
# Resource group & storage account (with best-practice flags)
az group create -n rg-storage-demo -l westus3
az storage account create -n storacctdemo$RANDOM -g rg-storage-demo -l westus3 \
  --sku Standard_GZRS --kind StorageV2 --https-only true \
  --min-tls-version TLS1_2 --allow-blob-public-access false \
  --default-action Deny --bypass AzureServices \
  --enable-delete-retention true --delete-retention-days 30

# Enable hierarchical namespace (ADLS Gen2) if needed:
# az storage account update -n <name> -g rg-storage-demo --enable-hierarchical-namespace true

# Containers & table
az storage container create --account-name <name> --name app-data
az storage table create --account-name <name> --name Users

Bicep (sketch)
param location string = resourceGroup().location
param accountName string
param sku string = 'Standard_GZRS'

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: accountName
  location: location
  sku: { name: sku }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
    supportsHttpsTrafficOnly: true
    encryption: {
      services: {
        blob: { enabled: true }
        table: { enabled: true }
      }
      keySource: 'Microsoft.Storage'
    }
  }
}

resource appData 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  name: '${sa.name}/default/app-data'
  properties: { publicAccess: 'None' }
}

C# — Auth & Clients (Managed Identity preferred)
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Data.Tables;

var credential = new DefaultAzureCredential();
var blobService = new BlobServiceClient(new Uri("https://<account>.blob.core.windows.net"), credential);
var tableService = new TableServiceClient(new Uri("https://<account>.table.core.windows.net"), credential);

C# — Blob upload/download (streaming, metadata, ETag concurrency)
var container = blobService.GetBlobContainerClient("app-data");
await container.CreateIfNotExistsAsync();

var blob = container.GetBlobClient("images/2025/10/photo.jpg");

// Upload with parallel transfer & content type
var upOpts = new Azure.Storage.Blobs.Models.BlobUploadOptions {
  HttpHeaders = new Azure.Storage.Blobs.Models.BlobHttpHeaders { ContentType = "image/jpeg" },
  TransferOptions = new Azure.Storage.StorageTransferOptions { InitialTransferSize = 8*1024*1024, MaximumTransferSize = 8*1024*1024, MaximumConcurrency = 4 },
  Metadata = new Dictionary<string,string> { ["source"] = "uploader", ["tenant"] = "acme" }
};
using var fs = File.OpenRead("photo.jpg");
await blob.UploadAsync(fs, upOpts);

// Conditional overwrite using ETag
var props = await blob.GetPropertiesAsync();
var accessCond = new Azure.RequestConditions { IfMatch = props.Value.ETag };
await blob.SetMetadataAsync(new Dictionary<string,string>{{"reviewed","true"}}, accessCond);

C# — User Delegation SAS (prefer over account SAS when possible)
var key = await blobService.GetUserDelegationKeyAsync(DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddHours(1));
var builder = new Azure.Storage.Sas.BlobSasBuilder {
  BlobContainerName = "app-data",
  BlobName = "images/2025/10/photo.jpg",
  ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15),
  Resource = "b"
};
builder.SetPermissions(Azure.Storage.Sas.BlobAccountSasPermissions.Read);
var sasUri = blob.GenerateUserDelegationSasUri(builder, key.Value);
// Share sasUri cautiously; set IP ranges if applicable

C# — Table Storage CRUD, Batch, Query
var table = tableService.GetTableClient("Users");
await table.CreateIfNotExistsAsync();

public record UserEntity(string PartitionKey, string RowKey) : ITableEntity {
  public string? Email { get; init; }
  public ETag ETag { get; set; }
  public DateTimeOffset? Timestamp { get; set; }
}

var u1 = new UserEntity("tenant-acme", "user-0001") { Email = "a@example.com" };
await table.AddEntityAsync(u1);

// Batch (same partition)
var actions = new List<TableTransactionAction> {
  new TableTransactionAction(TableTransactionActionType.Add, new UserEntity("tenant-acme","user-0002"){Email="b@example.com"}),
  new TableTransactionAction(TableTransactionActionType.UpsertMerge, new UserEntity("tenant-acme","user-0003"){Email="c@example.com"})
};
await table.SubmitTransactionAsync(actions);

// Query with filter and projection
var filter = TableClient.CreateQueryFilter($"PartitionKey eq { "tenant-acme" } and RowKey ge { "user-0002" }");
await foreach (var e in table.QueryAsync<UserEntity>(filter: filter, maxPerPage: 100)) {
  Console.WriteLine($"{e.RowKey} -> {e.Email}");
}

Lifecycle Management (cold/archive, delete, version)
{
  "rules": [
    {
      "name": "tiering-and-retention",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/","images/"] },
        "actions": {
          "baseBlob": { "tierToCool": { "daysAfterModificationGreaterThan": 30 }, "tierToArchive": { "daysAfterModificationGreaterThan": 180 }, "delete": { "daysAfterModificationGreaterThan": 730 } },
          "snapshot": { "delete": { "daysAfterCreationGreaterThan": 90 } },
          "version": { "delete": { "daysAfterCreationGreaterThan": 365 } }
        }
      }
    }
  ]
}

Private Endpoints & CORS (outline)

Private Endpoint: create for blob and table subresources; configure privatelink DNS; deny public network access.

CORS (for browser apps): restrict origins/methods/headers; short max-age; no wildcards on credentials.

Azurite (local dev)
npm i -g azurite
azurite --blobPort 10000 --queuePort 10001 --tablePort 10002

# .NET connection string for Azurite
UseDevelopmentStorage=true

GitHub Actions (OIDC to Azure, minimal)
name: ci
on: [push, pull_request]
jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - run: dotnet build --configuration Release
      - run: dotnet test --configuration Release --no-build
      - uses: azure/login@v2
        with: { client-id: ${{ secrets.AZURE_CLIENT_ID }}, tenant-id: ${{ secrets.AZURE_TENANT_ID }}, subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }} }
      - name: Deploy Bicep
        run: az deployment group create -g rg-storage-demo -f infra/main.bicep -p accountName=storacct...

Research & Citations

For each major section (Security, Identity, Networking, Lifecycle, Performance, Table design), consult ≥3 primary sources; note any disagreements; choose a default and justify it.

Include publish/update dates; avoid obsolete API versions.

Checklists (must include in the guide)

Security: Managed Identity only; no account keys in code; deny public network; private endpoints; CMK if compliance; soft delete + versioning; short-lived SAS w/ IP scope; TLS 1.2+; immutability when required.

Performance: Parallel/chunked uploads; gzip/deflate; CDN; partition keys avoid hotspots; pagination with continuation tokens.

Resilience: Exponential backoff; idempotent writes (ETags); retries for 5xx/429; server-side timeouts.

Operations: Diagnostics to Log Analytics; KQL saved queries; alert rules; lifecycle policies; cost review.

Final Checks

All code targets .NET 8; uses Azure.Storage.Blobs, Azure.Data.Tables, Azure.Identity.

No account keys in examples unless explicitly teaching legacy fallback.

Every guideline has a code sample, a checklist, or both.

Document is self-contained and immediately actionable.