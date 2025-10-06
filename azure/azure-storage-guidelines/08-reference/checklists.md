# Implementation Checklists

> **File Purpose**: Comprehensive checklists for security, performance, resilience, and operations
> **Prerequisites**: None (reference document)
> **Agent Use Case**: Verify implementation completeness before production deployment

## Quick Context

Use these checklists to validate Azure Storage implementations against best practices. Each checklist is organized by priority: Critical (must-have for production), High (strongly recommended), and Optional (nice-to-have or compliance-driven).

**How to use**: Review checklists at each stage: provisioning, development, pre-production, and ongoing operations.

---

## Security Checklist

### Critical (Must-Have for Production)

- [ ] **Managed Identity enabled** for all Azure-hosted applications (no account keys in code)
- [ ] **DefaultAzureCredential** used in application code (works across local/staging/production)
- [ ] **RBAC roles assigned** with least privilege (`Storage Blob/Table Data Contributor`, not `Contributor`)
- [ ] **Account keys disabled** or access audited (`allowSharedKeyAccess: false` if Managed Identity covers all use cases)
- [ ] **Public network access denied** (`networkAcls.defaultAction: Deny`)
- [ ] **TLS 1.2+ enforced** (`minimumTlsVersion: TLS1_2`)
- [ ] **Public blob access disabled** (`allowBlobPublicAccess: false` at account level)
- [ ] **Container public access set to None** (no anonymous access)
- [ ] **Soft delete enabled** for blobs (30-day retention minimum)
- [ ] **Diagnostic settings configured** (logs to Log Analytics)
- [ ] **Firewall rules or private endpoints** configured (no open internet access)

### High Priority (Strongly Recommended)

- [ ] **Private endpoints** deployed for production workloads (eliminates internet exposure)
- [ ] **User delegation SAS** used instead of account SAS (when SAS is required)
- [ ] **SAS tokens** have expiry (≤1 hour for high-security, ≤24 hours for general use)
- [ ] **SAS tokens** scoped to minimum permissions (read vs read/write/delete)
- [ ] **SAS tokens** include IP restrictions when possible
- [ ] **Blob versioning enabled** (recover from accidental overwrites)
- [ ] **Container soft delete enabled** (7-day retention)
- [ ] **Immutability policies** applied for compliance workloads (WORM requirements)
- [ ] **Customer-managed keys (CMK)** configured if required by compliance
- [ ] **Network rules** restrict access to known VNETs or IP ranges
- [ ] **CORS rules** restrictive (no wildcards; specific origins only)

### Optional (Compliance or Advanced)

- [ ] **Double encryption** enabled (infrastructure + service encryption)
- [ ] **Legal hold** policies for litigation/audit requirements
- [ ] **Time-based retention** policies (immutability for audit trails)
- [ ] **Azure Policy** enforces storage account standards (deny non-compliant configurations)
- [ ] **Key rotation** automated for CMK (Key Vault versioning)
- [ ] **Stored access policies** used for SAS revocation capability
- [ ] **Cross-tenant access** blocked unless explicitly required

### Security Verification Commands

```bash
# Verify network access is denied
az storage account show -n <account-name> -g <rg> \
  --query "networkRuleSet.defaultAction" -o tsv
# Expected: Deny

# Verify TLS version
az storage account show -n <account-name> -g <rg> \
  --query "minimumTlsVersion" -o tsv
# Expected: TLS1_2

# Verify public blob access is disabled
az storage account show -n <account-name> -g <rg> \
  --query "allowBlobPublicAccess" -o tsv
# Expected: false

# Check if account keys are disabled
az storage account show -n <account-name> -g <rg> \
  --query "allowSharedKeyAccess" -o tsv
# Expected: false (if using Managed Identity only)
```

---

## Performance Checklist

### Blob Storage Performance

#### Critical

- [ ] **Parallel uploads** configured for files >4MB (`MaximumConcurrency: 4-8`)
- [ ] **Chunk sizes** optimized (8MB for most workloads, 16MB for >1GB files)
- [ ] **Streaming APIs** used for large files (avoid loading into memory)
- [ ] **Content-Type headers** set correctly (enables browser rendering, CDN caching)
- [ ] **Retry policies** configured (exponential backoff, 5 retries minimum)
- [ ] **Client pooling** implemented (reuse BlobServiceClient instances)

#### High Priority

- [ ] **CDN** configured for frequently accessed blobs (images, videos, static assets)
- [ ] **Cache-Control headers** set for cacheable blobs (`max-age` directive)
- [ ] **Hot tier** used for frequently accessed data (default tier)
- [ ] **Cool tier** used for infrequently accessed data (>30 days without access)
- [ ] **Archive tier** used for cold storage (>180 days without access)
- [ ] **Lifecycle policies** automate tiering and deletion
- [ ] **ETag-based caching** implemented (If-None-Match for conditional downloads)
- [ ] **Progress handlers** implemented for long-running uploads (UX feedback)

#### Optional

- [ ] **ADLS Gen2** enabled for analytics workloads (hierarchical namespace)
- [ ] **Gzip compression** applied to text-based blobs (reduce bandwidth)
- [ ] **Read-access geo-redundancy (RA-GZRS)** used for global read performance
- [ ] **Block blob upload parallelism** tuned based on network capacity
- [ ] **Blob prefetch** implemented for predictable access patterns

### Table Storage Performance

#### Critical

- [ ] **PartitionKey** distributes load evenly (no hot partitions)
- [ ] **Queries filter by PartitionKey** (avoid full table scans)
- [ ] **Batch operations** used for multi-entity writes (same partition)
- [ ] **Pagination** implemented with continuation tokens (maxPerPage: 100-1000)

#### High Priority

- [ ] **RowKey** designed for range queries (sort order matters)
- [ ] **Projection (select)** used to reduce bandwidth (fetch only needed properties)
- [ ] **Point queries** used when PartitionKey + RowKey are known (fastest)
- [ ] **Secondary indexes** maintained for common non-PK/RK queries
- [ ] **ETags** used for optimistic concurrency control

#### Optional

- [ ] **Cosmos DB Table API** evaluated for global distribution needs
- [ ] **Denormalization** applied to reduce query complexity
- [ ] **Time-based RowKeys** used for time-series data (reverse timestamp for newest-first)

### Performance Verification

```csharp
// Verify BlobServiceClient retry policy
var options = new BlobClientOptions();
Console.WriteLine($"Max retries: {options.Retry.MaxRetries}");
Console.WriteLine($"Retry mode: {options.Retry.Mode}");

// Verify parallel transfer settings
var uploadOptions = new BlobUploadOptions();
Console.WriteLine($"Max concurrency: {uploadOptions.TransferOptions.MaximumConcurrency}");
Console.WriteLine($"Chunk size: {uploadOptions.TransferOptions.MaximumTransferSize}");
```

---

## Resilience Checklist

### Critical

- [ ] **Exponential backoff** implemented for retries (SDK default: 5 retries)
- [ ] **Transient error handling** for 429 (throttling) and 5xx (server errors)
- [ ] **Network timeout** configured (default: 100 seconds; increase for large files)
- [ ] **Idempotent operations** designed (safe to retry)
- [ ] **Circuit breaker** pattern for cascading failure prevention (optional but recommended)

### High Priority

- [ ] **ETag-based concurrency** for updates (prevent lost updates)
- [ ] **Conditional operations** used (If-Match, If-None-Match)
- [ ] **Soft delete** enables recovery from accidental deletes (30-day retention)
- [ ] **Versioning** enables recovery from overwrites
- [ ] **Health checks** probe storage account accessibility
- [ ] **Fallback strategies** for read operations (RA-GZRS secondary endpoint)

### Optional

- [ ] **Chaos engineering** tests storage failures (Polly for simulated faults)
- [ ] **Rate limiting** implemented client-side (prevent self-throttling)
- [ ] **Request ID propagation** for end-to-end tracing
- [ ] **Retry-After header** respected for 429 responses

### Resilience Verification

```csharp
// Test transient error handling
try
{
    await blobClient.UploadAsync(stream);
}
catch (RequestFailedException ex) when (ex.Status == 429)
{
    // Check Retry-After header
    if (ex.Headers.TryGetValue("Retry-After", out var retryAfter))
    {
        Console.WriteLine($"Throttled, retry after: {retryAfter}");
    }
}
```

---

## Operations Checklist

### Monitoring & Observability

#### Critical

- [ ] **Diagnostic settings** send logs to Log Analytics
- [ ] **Storage metrics** monitored (transactions, latency, availability)
- [ ] **Azure Monitor alerts** configured for:
  - [ ] Throttling (429 responses)
  - [ ] High latency (P95 >1 second)
  - [ ] Authentication failures (403)
  - [ ] Capacity thresholds (>80% of quota)
- [ ] **Request IDs** logged in application for correlation
- [ ] **Dashboards** visualize storage health (Azure Monitor or Grafana)

#### High Priority

- [ ] **KQL queries** saved for common investigations (see `06-operations/observability.md`)
- [ ] **Alerts** sent to on-call team (PagerDuty, Teams, Slack)
- [ ] **Log retention** configured (90+ days for compliance)
- [ ] **Cost monitoring** tracks storage spend by tier and operation type
- [ ] **Application Insights** integrated for end-to-end tracing

#### Optional

- [ ] **Custom metrics** track business KPIs (uploads per user, query latency by tenant)
- [ ] **Anomaly detection** enabled on storage metrics
- [ ] **Runbooks** documented for common alerts

### Governance

#### Critical

- [ ] **Naming conventions** enforced (e.g., `storacct<env><app><region>`)
- [ ] **Resource tags** applied (Environment, Project, Owner, CostCenter)
- [ ] **Lifecycle policies** automate tiering and deletion (cost optimization)
- [ ] **Backup strategy** documented (soft delete + versioning vs snapshot-based)
- [ ] **Access reviews** conducted quarterly (RBAC assignments)

#### High Priority

- [ ] **Cost budgets** set with alerts (Azure Cost Management)
- [ ] **Deletion protection** via Azure Policy or resource locks
- [ ] **Data retention policies** documented and enforced
- [ ] **Disaster recovery plan** includes RPO/RTO for storage
- [ ] **Compliance certifications** verified (HIPAA, PCI-DSS, SOC 2)

#### Optional

- [ ] **Azure Blueprints** standardize storage account deployments
- [ ] **Policy-as-code** (deny non-GZRS accounts, enforce CMK)
- [ ] **Tagging automation** via Azure Policy (inherit tags from resource group)

### Governance Verification

```bash
# Verify tags
az storage account show -n <account-name> -g <rg> --query "tags"

# Check lifecycle policies
az storage account management-policy show -n <account-name> -g <rg>

# Review RBAC assignments
az role assignment list --scope <storage-account-id> -o table
```

---

## Testing Checklist

### Local Development

- [ ] **Azurite** running locally (blob + table services)
- [ ] **Connection strings** switched via appsettings.Development.json
- [ ] **Seed scripts** create test containers and tables
- [ ] **Unit tests** run against Azurite (no cloud dependencies)

### Integration Testing

- [ ] **Contract tests** verify API compatibility
- [ ] **End-to-end tests** use dedicated test storage account
- [ ] **Performance tests** measure throughput and latency
- [ ] **Chaos tests** simulate transient failures (timeout, 429, 503)
- [ ] **Security tests** verify RBAC enforcement (unauthorized access fails)

### Pre-Production

- [ ] **Load tests** simulate production traffic (Azure Load Testing)
- [ ] **Failover tests** verify RA-GZRS secondary endpoint access
- [ ] **Disaster recovery drill** tests restore from soft delete/versioning

---

## CI/CD Checklist

### Critical

- [ ] **OIDC authentication** configured (GitHub Actions → Azure)
- [ ] **No secrets** in code or config files (use Managed Identity)
- [ ] **Infrastructure as Code** (Bicep/Terraform) deployed idempotently
- [ ] **Environment promotion** (dev → staging → prod)
- [ ] **Deployment validation** tests after infra deployment

### High Priority

- [ ] **Pre-commit hooks** scan for secrets (detect-secrets, trufflehog)
- [ ] **Security scans** run on dependencies (Dependabot, Snyk)
- [ ] **Smoke tests** run after deployment
- [ ] **Rollback plan** documented and tested
- [ ] **Deployment notifications** sent to team (Teams, Slack)

### Optional

- [ ] **Blue-green deployments** for zero-downtime updates
- [ ] **Canary deployments** roll out changes gradually
- [ ] **Drift detection** alerts on manual Azure Portal changes

---

## Pre-Production Deployment Gate

**All Critical items from Security, Performance, Resilience, Operations, and Testing checklists must be complete before production deployment.**

### Final Verification Steps

1. Run security scan: `az security assessment list --scope <storage-account-id>`
2. Review diagnostic logs: Check Log Analytics for errors/warnings
3. Test failover: Access RA-GZRS secondary endpoint
4. Verify monitoring: Trigger test alert (upload large file, check latency alert)
5. Document runbooks: Ensure on-call team can respond to alerts
6. Conduct access review: Verify only necessary principals have RBAC roles

---

## Navigation

- **Previous**: `code-examples.md`
- **Next**: `references.md`
- **Up**: `00-overview.md`

## See Also

- `05-security/` - Detailed security implementation
- `06-operations/observability.md` - Monitoring setup
- `07-deployment/cicd-patterns.md` - CI/CD configuration
