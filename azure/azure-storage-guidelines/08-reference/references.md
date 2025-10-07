# References and Documentation

> **File Purpose**: Comprehensive collection of official Microsoft documentation, SDKs, tools, and community resources
> **Prerequisites**: None (reference document)
> **Agent Use Case**: Quick access to authoritative sources for Azure Storage

## Quick Context

This reference organizes all essential Azure Storage documentation by category: official Microsoft Learn docs, SDK references, CLI/PowerShell tools, security best practices, compliance resources, and community samples. All links are verified as of 2024-2025.

**Key principle**: Bookmark official Microsoft documentation for the latest API changes and security advisories.

---

## Microsoft Learn Documentation

### Core Storage Concepts

**Azure Storage Overview**
- [Azure Storage documentation home](https://learn.microsoft.com/azure/storage/)
  - Last verified: January 2025
  - Comprehensive landing page for all storage services

- [Introduction to Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-introduction)
  - Updated: December 2024
  - Overview of Blob, Table, Queue, File services

- [Storage account overview](https://learn.microsoft.com/azure/storage/common/storage-account-overview)
  - Updated: December 2024
  - Account types, redundancy options, performance tiers

- [Redundancy options](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
  - Updated: November 2024
  - LRS, ZRS, GRS, GZRS, RA-GZRS explained

### Blob Storage

**Fundamentals**
- [Blob Storage overview](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction)
  - Updated: December 2024
  - Block blobs, append blobs, page blobs

- [Access tiers](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview)
  - Updated: November 2024
  - Hot, Cool, Cold, Archive tiers and pricing

- [Lifecycle management](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
  - Updated: December 2024
  - Automated tiering and deletion policies

- [Blob versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
  - Updated: October 2024
  - Automatic version creation on overwrites

- [Soft delete for blobs](https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview)
  - Updated: November 2024
  - Recovery from accidental deletes

**Performance and Scalability**
- [Scalability and performance targets](https://learn.microsoft.com/azure/storage/blobs/scalability-targets)
  - Updated: December 2024
  - Request rates, bandwidth limits, partition targets

- [Performance optimization](https://learn.microsoft.com/azure/storage/blobs/storage-performance-checklist)
  - Updated: November 2024
  - Checklist for high-throughput scenarios

- [Data Lake Storage Gen2](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
  - Updated: December 2024
  - Hierarchical namespace for analytics

**Advanced Features**
- [Blob index tags](https://learn.microsoft.com/azure/storage/blobs/storage-blob-index-how-to)
  - Updated: September 2024
  - Key-value metadata for queries

- [Change feed](https://learn.microsoft.com/azure/storage/blobs/storage-blob-change-feed)
  - Updated: October 2024
  - Ordered log of blob changes

- [Point-in-time restore](https://learn.microsoft.com/azure/storage/blobs/point-in-time-restore-overview)
  - Updated: November 2024
  - Restore block blobs to earlier state

### Table Storage

**Fundamentals**
- [Table Storage overview](https://learn.microsoft.com/azure/storage/tables/table-storage-overview)
  - Updated: November 2024
  - NoSQL key-value store for semi-structured data

- [Table Storage design guide](https://learn.microsoft.com/azure/storage/tables/table-storage-design)
  - Updated: December 2024
  - Schema design, partition key strategies

- [Query design](https://learn.microsoft.com/azure/storage/tables/table-storage-design-for-query)
  - Updated: October 2024
  - Optimizing queries with PartitionKey and RowKey

- [Data modeling](https://learn.microsoft.com/azure/storage/tables/table-storage-design-modeling)
  - Updated: November 2024
  - Denormalization patterns, entity group transactions

**Performance**
- [Scalability targets](https://learn.microsoft.com/azure/storage/tables/scalability-targets)
  - Updated: December 2024
  - Partition throughput limits

### Security

**Identity and Access**
- [Authorize access to data](https://learn.microsoft.com/azure/storage/common/authorize-data-access)
  - Updated: December 2024
  - Azure AD, Shared Key, SAS comparison

- [Assign Azure RBAC roles](https://learn.microsoft.com/azure/storage/blobs/assign-azure-role-data-access)
  - Updated: November 2024
  - Built-in roles for blob and table access

- [Managed identities with storage](https://learn.microsoft.com/azure/storage/common/storage-auth-aad-msi)
  - Updated: December 2024
  - System-assigned and user-assigned identities

- [Shared Access Signatures (SAS)](https://learn.microsoft.com/azure/storage/common/storage-sas-overview)
  - Updated: December 2024
  - User delegation, account, and service SAS

- [Best practices for SAS](https://learn.microsoft.com/azure/storage/common/storage-sas-best-practices)
  - Updated: November 2024
  - Expiry, permissions, IP restrictions

**Encryption**
- [Encryption at rest](https://learn.microsoft.com/azure/storage/common/storage-service-encryption)
  - Updated: December 2024
  - Microsoft-managed and customer-managed keys

- [Customer-managed keys (CMK)](https://learn.microsoft.com/azure/storage/common/customer-managed-keys-overview)
  - Updated: November 2024
  - Key Vault integration for CMK

- [Infrastructure encryption](https://learn.microsoft.com/azure/storage/common/infrastructure-encryption-enable)
  - Updated: October 2024
  - Double encryption for high-security workloads

- [TLS encryption](https://learn.microsoft.com/azure/storage/common/transport-layer-security-configure-minimum-version)
  - Updated: November 2024
  - Enforce TLS 1.2 or higher

**Network Security**
- [Configure firewalls and virtual networks](https://learn.microsoft.com/azure/storage/common/storage-network-security)
  - Updated: December 2024
  - Network ACLs, service endpoints, private endpoints

- [Private endpoints](https://learn.microsoft.com/azure/storage/common/storage-private-endpoints)
  - Updated: December 2024
  - Secure connectivity via Azure Private Link

- [Disable public network access](https://learn.microsoft.com/azure/storage/common/storage-network-security#change-the-default-network-access-rule)
  - Updated: November 2024
  - Enforce private-only access

**Compliance**
- [Security baseline](https://learn.microsoft.com/security/benchmark/azure/baselines/storage-security-baseline)
  - Updated: December 2024
  - Azure Security Benchmark for storage

- [Compliance certifications](https://learn.microsoft.com/azure/compliance/)
  - Updated: Ongoing
  - HIPAA, PCI DSS, SOC 2, FedRAMP, GDPR

### Monitoring and Operations

**Observability**
- [Monitoring Azure Storage](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage)
  - Updated: December 2024
  - Metrics, logs, alerts

- [Storage Analytics logs](https://learn.microsoft.com/azure/storage/common/storage-analytics-logging)
  - Updated: November 2024
  - Legacy logging (use Azure Monitor instead)

- [Azure Monitor for Storage](https://learn.microsoft.com/azure/azure-monitor/insights/storage-insights-overview)
  - Updated: December 2024
  - Unified monitoring experience

- [Diagnostic settings](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage#create-diagnostic-settings)
  - Updated: November 2024
  - Send logs to Log Analytics, Event Hubs, Storage

**Cost Optimization**
- [Plan and manage costs](https://learn.microsoft.com/azure/storage/common/storage-plan-manage-costs)
  - Updated: December 2024
  - Pricing calculator, cost analysis

- [Optimize costs for Blob Storage](https://learn.microsoft.com/azure/storage/blobs/storage-blob-storage-tiers)
  - Updated: November 2024
  - Tiering strategies, lifecycle policies

**Troubleshooting**
- [Troubleshoot availability](https://learn.microsoft.com/azure/storage/common/troubleshoot-storage-availability)
  - Updated: October 2024
  - 429, 503, timeout errors

- [Troubleshoot performance](https://learn.microsoft.com/azure/storage/common/troubleshoot-storage-performance)
  - Updated: November 2024
  - High latency, low throughput

- [Troubleshoot client application errors](https://learn.microsoft.com/azure/storage/common/storage-client-side-errors)
  - Updated: October 2024
  - 403, 404, authentication failures

### Disaster Recovery

- [Storage account failover](https://learn.microsoft.com/azure/storage/common/storage-disaster-recovery-guidance)
  - Updated: November 2024
  - Customer-managed and Microsoft-managed failover

- [Backup and recovery](https://learn.microsoft.com/azure/storage/blobs/backup-restore-overview)
  - Updated: October 2024
  - Soft delete, versioning, snapshots

---

## Azure Architecture Center

### Reference Architectures

- [Store unstructured data with Azure Blob Storage](https://learn.microsoft.com/azure/architecture/example-scenario/data/unstructured-data-store)
  - Updated: December 2024
  - CDN integration, lifecycle policies

- [NoSQL data stores with Azure Table Storage](https://learn.microsoft.com/azure/architecture/data-guide/big-data/non-relational-data)
  - Updated: November 2024
  - When to use Table Storage vs Cosmos DB

- [Multi-region storage with RA-GZRS](https://learn.microsoft.com/azure/architecture/reference-architectures/app-service-web-app/multi-region)
  - Updated: December 2024
  - Global read access patterns

### Design Patterns

- [Competing Consumers pattern](https://learn.microsoft.com/azure/architecture/patterns/competing-consumers)
  - Queue Storage for work distribution

- [Valet Key pattern](https://learn.microsoft.com/azure/architecture/patterns/valet-key)
  - SAS tokens for direct client access

- [Static Content Hosting](https://learn.microsoft.com/azure/architecture/patterns/static-content-hosting)
  - Blob Storage with CDN

### Best Practices

- [Storage account design](https://learn.microsoft.com/azure/architecture/best-practices/storage-accounts)
  - Updated: November 2024
  - Naming, partitioning, security

- [Data partitioning strategies](https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning)
  - Updated: October 2024
  - Table Storage partition key design

---

## Azure SDK Documentation

### .NET (C#)

**SDK Package Versions**
- Azure.Storage.Blobs 12.22.1 (Latest: January 2025)
- Azure.Data.Tables 12.9.1 (Latest: December 2024)
- Azure.Identity 1.13.0 (Latest: January 2025)

**Official Documentation**
- [Azure Storage Blobs client library for .NET](https://learn.microsoft.com/dotnet/api/overview/azure/storage.blobs-readme)
  - Updated: January 2025
  - BlobServiceClient, BlobContainerClient, BlobClient

- [Azure Data Tables client library for .NET](https://learn.microsoft.com/dotnet/api/overview/azure/data.tables-readme)
  - Updated: December 2024
  - TableServiceClient, TableClient

- [Azure Identity client library for .NET](https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme)
  - Updated: January 2025
  - DefaultAzureCredential, ManagedIdentityCredential

**API Reference**
- [Azure.Storage.Blobs namespace](https://learn.microsoft.com/dotnet/api/azure.storage.blobs)
  - All blob classes and methods

- [Azure.Data.Tables namespace](https://learn.microsoft.com/dotnet/api/azure.data.tables)
  - All table classes and methods

**GitHub**
- [Azure SDK for .NET repository](https://github.com/Azure/azure-sdk-for-net)
  - Source code, samples, issues

- [Blob Storage samples](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/storage/Azure.Storage.Blobs/samples)
  - Authentication, upload, download, metadata

- [Table Storage samples](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/tables/Azure.Data.Tables/samples)
  - CRUD operations, queries, transactions

### Python

**SDK Package Versions**
- azure-storage-blob 12.23.1 (Latest: January 2025)
- azure-data-tables 12.6.0 (Latest: December 2024)
- azure-identity 1.19.0 (Latest: January 2025)

**Official Documentation**
- [Azure Storage Blobs client library for Python](https://learn.microsoft.com/python/api/overview/azure/storage-blob-readme)
  - Updated: January 2025
  - BlobServiceClient, ContainerClient, BlobClient

- [Azure Data Tables client library for Python](https://learn.microsoft.com/python/api/overview/azure/data-tables-readme)
  - Updated: December 2024
  - TableServiceClient, TableClient

- [Azure Identity client library for Python](https://learn.microsoft.com/python/api/overview/azure/identity-readme)
  - Updated: January 2025
  - DefaultAzureCredential, ManagedIdentityCredential

**API Reference**
- [azure.storage.blob package](https://learn.microsoft.com/python/api/azure-storage-blob/)

- [azure.data.tables package](https://learn.microsoft.com/python/api/azure-data-tables/)

**GitHub**
- [Azure SDK for Python repository](https://github.com/Azure/azure-sdk-for-python)

- [Blob samples](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/storage/azure-storage-blob/samples)

- [Table samples](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/tables/azure-data-tables/samples)

### Node.js (JavaScript/TypeScript)

**SDK Package Versions**
- @azure/storage-blob 12.25.0 (Latest: January 2025)
- @azure/data-tables 13.3.0 (Latest: December 2024)
- @azure/identity 4.5.0 (Latest: January 2025)

**Official Documentation**
- [Azure Storage Blob client library for JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/storage-blob-readme)
  - Updated: January 2025

- [Azure Data Tables client library for JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/data-tables-readme)
  - Updated: December 2024

- [Azure Identity client library for JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/identity-readme)
  - Updated: January 2025

**API Reference**
- [@azure/storage-blob package](https://learn.microsoft.com/javascript/api/@azure/storage-blob/)

- [@azure/data-tables package](https://learn.microsoft.com/javascript/api/@azure/data-tables/)

**GitHub**
- [Azure SDK for JavaScript repository](https://github.com/Azure/azure-sdk-for-js)

- [Blob samples](https://github.com/Azure/azure-sdk-for-js/tree/main/sdk/storage/storage-blob/samples)

- [Table samples](https://github.com/Azure/azure-sdk-for-js/tree/main/sdk/tables/data-tables/samples)

### Java

**SDK Package Versions**
- azure-storage-blob 12.28.1 (Latest: January 2025)
- azure-data-tables 12.5.0 (Latest: December 2024)
- azure-identity 1.14.2 (Latest: January 2025)

**Official Documentation**
- [Azure Storage Blobs client library for Java](https://learn.microsoft.com/java/api/overview/azure/storage-blob-readme)
  - Updated: January 2025

- [Azure Data Tables client library for Java](https://learn.microsoft.com/java/api/overview/azure/data-tables-readme)
  - Updated: December 2024

**GitHub**
- [Azure SDK for Java repository](https://github.com/Azure/azure-sdk-for-java)

- [Blob samples](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/storage/azure-storage-blob/src/samples)

- [Table samples](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/tables/azure-data-tables/src/samples)

---

## Azure CLI and PowerShell

### Azure CLI

**Installation and Updates**
- [Install Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
  - Updated: January 2025
  - Windows, macOS, Linux

- [Update Azure CLI](https://learn.microsoft.com/cli/azure/update-azure-cli)
  - Updated: December 2024

**Storage Commands**
- [az storage account](https://learn.microsoft.com/cli/azure/storage/account)
  - Create, update, delete storage accounts

- [az storage blob](https://learn.microsoft.com/cli/azure/storage/blob)
  - Blob operations (upload, download, list, delete)

- [az storage container](https://learn.microsoft.com/cli/azure/storage/container)
  - Container management

- [az storage table](https://learn.microsoft.com/cli/azure/storage/table)
  - Table operations

- [az storage account management-policy](https://learn.microsoft.com/cli/azure/storage/account/management-policy)
  - Lifecycle policies

- [az storage account private-endpoint-connection](https://learn.microsoft.com/cli/azure/storage/account/private-endpoint-connection)
  - Private endpoint management

**Authentication**
- [az login](https://learn.microsoft.com/cli/azure/reference-index#az-login)
  - Azure AD authentication

- [az account set](https://learn.microsoft.com/cli/azure/account#az-account-set)
  - Set active subscription

### Azure PowerShell

**Installation**
- [Install Azure PowerShell](https://learn.microsoft.com/powershell/azure/install-azps-windows)
  - Updated: January 2025
  - Windows, macOS, Linux

**Storage Cmdlets**
- [Az.Storage module](https://learn.microsoft.com/powershell/module/az.storage/)
  - All storage cmdlets

- [New-AzStorageAccount](https://learn.microsoft.com/powershell/module/az.storage/new-azstorageaccount)
  - Create storage accounts

- [Set-AzStorageBlobContent](https://learn.microsoft.com/powershell/module/az.storage/set-azstorageblobcontent)
  - Upload blobs

- [Get-AzStorageBlob](https://learn.microsoft.com/powershell/module/az.storage/get-azstorageblob)
  - List and retrieve blobs

- [New-AzStorageTable](https://learn.microsoft.com/powershell/module/az.storage/new-azstoragetable)
  - Create tables

---

## Bicep and ARM Templates

### Bicep

**Official Documentation**
- [Bicep documentation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/)
  - Updated: January 2025
  - Language reference, best practices

- [Bicep file structure](https://learn.microsoft.com/azure/azure-resource-manager/bicep/file)
  - Updated: December 2024
  - Parameters, variables, resources, outputs

- [Bicep modules](https://learn.microsoft.com/azure/azure-resource-manager/bicep/modules)
  - Updated: December 2024
  - Reusable module patterns

**Resource Definitions**
- [Microsoft.Storage/storageAccounts](https://learn.microsoft.com/azure/templates/microsoft.storage/storageaccounts)
  - Updated: December 2024
  - All properties and sub-resources

- [Microsoft.Storage/storageAccounts/blobServices](https://learn.microsoft.com/azure/templates/microsoft.storage/storageaccounts/blobservices)
  - Blob service properties

- [Microsoft.Storage/storageAccounts/tableServices](https://learn.microsoft.com/azure/templates/microsoft.storage/storageaccounts/tableservices)
  - Table service properties

**Tools**
- [Bicep Playground](https://aka.ms/bicepdemo)
  - Interactive Bicep editor

- [Bicep VS Code extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep)
  - IntelliSense, validation, snippets

**GitHub**
- [Bicep repository](https://github.com/Azure/bicep)
  - Source code, issues, discussions

- [Azure Quickstart Templates (Bicep)](https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.storage)
  - Community-contributed templates

### ARM Templates

**Documentation**
- [ARM template documentation](https://learn.microsoft.com/azure/azure-resource-manager/templates/)
  - Updated: December 2024

- [ARM template best practices](https://learn.microsoft.com/azure/azure-resource-manager/templates/best-practices)
  - Updated: November 2024

- [Template reference](https://learn.microsoft.com/azure/templates/)
  - All resource types

**Tools**
- [ARM Template Visualizer](https://armviz.io/)
  - Visualize template dependencies

---

## Security Best Practices

### Microsoft Security Documentation

- [Azure Security Baseline for Storage](https://learn.microsoft.com/security/benchmark/azure/baselines/storage-security-baseline)
  - Updated: December 2024
  - Comprehensive security recommendations

- [Security recommendations for Blob Storage](https://learn.microsoft.com/azure/storage/blobs/security-recommendations)
  - Updated: December 2024
  - Identity, network, data protection

- [Azure Storage security guide](https://learn.microsoft.com/azure/storage/blobs/security-recommendations)
  - Updated: November 2024

- [Prevent Anonymous Read Access](https://learn.microsoft.com/azure/storage/blobs/anonymous-read-access-prevent)
  - Updated: November 2024

- [Require secure transfer (HTTPS)](https://learn.microsoft.com/azure/storage/common/storage-require-secure-transfer)
  - Updated: October 2024

### Zero Trust and Defender for Cloud

- [Apply Zero Trust principles to storage](https://learn.microsoft.com/security/zero-trust/azure-infrastructure-storage)
  - Updated: December 2024
  - Verify explicitly, least privilege, assume breach

- [Microsoft Defender for Storage](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-storage-introduction)
  - Updated: January 2025
  - Threat detection for storage accounts

- [Enable Defender for Storage](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-storage-deploy)
  - Updated: December 2024

### Azure Policy

- [Azure Policy built-in definitions for Storage](https://learn.microsoft.com/azure/storage/common/policy-reference)
  - Updated: December 2024
  - Deny insecure configurations

- [Enforce TLS 1.2](https://learn.microsoft.com/azure/governance/policy/samples/built-in-policies#storage)
  - Policy: Storage accounts should have minimum TLS version of 1.2

- [Disable public access](https://learn.microsoft.com/azure/governance/policy/samples/built-in-policies#storage)
  - Policy: Storage accounts should prevent public blob access

---

## Performance Optimization

### Performance Guides

- [Performance and scalability checklist](https://learn.microsoft.com/azure/storage/blobs/storage-performance-checklist)
  - Updated: December 2024
  - Upload/download optimization

- [Optimize performance for data operations](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-tune-upload-download-dotnet)
  - Updated: November 2024
  - .NET SDK tuning

- [Partitioning in Table Storage](https://learn.microsoft.com/azure/storage/tables/table-storage-design-for-query)
  - Updated: November 2024
  - Avoid hot partitions

### CDN Integration

- [Azure CDN with Blob Storage](https://learn.microsoft.com/azure/cdn/cdn-create-a-storage-account-with-cdn)
  - Updated: December 2024
  - Cache static content globally

- [CDN caching rules](https://learn.microsoft.com/azure/cdn/cdn-caching-rules)
  - Updated: November 2024

---

## Compliance Resources

### Certifications and Attestations

- [Azure compliance documentation](https://learn.microsoft.com/azure/compliance/)
  - Updated: Ongoing
  - HIPAA, PCI DSS, SOC 2, ISO 27001, FedRAMP

- [Azure compliance offerings](https://learn.microsoft.com/azure/compliance/offerings/)
  - Detailed certification info

### HIPAA

- [HIPAA and HITECH Act compliance](https://learn.microsoft.com/azure/compliance/offerings/offering-hipaa-us)
  - Updated: December 2024
  - Business Associate Agreement (BAA)

- [Encrypt PHI at rest and in transit](https://learn.microsoft.com/azure/storage/common/storage-service-encryption)
  - Customer-managed keys for additional control

### PCI DSS

- [PCI DSS compliance](https://learn.microsoft.com/azure/compliance/offerings/offering-pci-dss)
  - Updated: December 2024
  - Storage requirements for cardholder data

- [Network isolation for PCI DSS](https://learn.microsoft.com/azure/storage/common/storage-network-security)
  - Private endpoints, network ACLs

### SOC 2

- [SOC 2 Type II attestation](https://learn.microsoft.com/azure/compliance/offerings/offering-soc-2)
  - Updated: November 2024

- [Audit logs for SOC 2](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage)
  - Log Analytics for control evidence

### GDPR

- [GDPR compliance](https://learn.microsoft.com/compliance/regulatory/gdpr)
  - Updated: December 2024

- [Data residency](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
  - Choose regions for data sovereignty

---

## Community Resources

### GitHub Repositories

**Official Microsoft Samples**
- [Azure Storage Samples (.NET)](https://github.com/Azure-Samples/storage-dotnet-getting-started)
  - Comprehensive .NET examples

- [Azure Storage Samples (Python)](https://github.com/Azure-Samples/storage-python-getting-started)
  - Python SDK examples

- [Azure Storage Samples (Node.js)](https://github.com/Azure-Samples/storage-node-getting-started)
  - JavaScript examples

- [Azure Bicep Samples](https://github.com/Azure/bicep/tree/main/docs/examples)
  - IaC patterns

**Community Projects**
- [Azure Storage Explorer (open source)](https://github.com/microsoft/AzureStorageExplorer)
  - Cross-platform GUI for storage

- [Azurite (local emulator)](https://github.com/Azure/Azurite)
  - Open-source storage emulator

### Blogs and Articles

**Official Microsoft Blogs**
- [Azure Storage Blog](https://techcommunity.microsoft.com/t5/azure-storage-blog/bg-p/AzureStorageBlog)
  - Product announcements, best practices

- [Azure Architecture Blog](https://techcommunity.microsoft.com/t5/azure-architecture-blog/bg-p/AzureArchitectureBlog)
  - Design patterns, case studies

**Community Blogs (2024-2025)**
- [John Savill's Technical Training](https://www.youtube.com/@NTFAQGuy)
  - Azure deep-dives, storage tutorials

- [Azure Tips and Tricks](https://microsoft.github.io/AzureTipsAndTricks/)
  - Quick how-tos

### Learning Paths

**Microsoft Learn**
- [Store data in Azure](https://learn.microsoft.com/training/paths/store-data-in-azure/)
  - Interactive learning modules

- [Secure your cloud data](https://learn.microsoft.com/training/paths/secure-your-cloud-data/)
  - Security best practices

- [Architect storage infrastructure in Azure](https://learn.microsoft.com/training/paths/architect-storage-infrastructure/)
  - Design patterns for architects

**Certifications**
- [AZ-104: Microsoft Azure Administrator](https://learn.microsoft.com/certifications/azure-administrator/)
  - Includes storage management

- [AZ-305: Designing Microsoft Azure Infrastructure Solutions](https://learn.microsoft.com/certifications/azure-solutions-architect/)
  - Storage architecture patterns

### Tools and Utilities

**Azure Storage Explorer**
- [Download](https://azure.microsoft.com/features/storage-explorer/)
  - Windows, macOS, Linux GUI

**Azurite (Local Emulator)**
- [Azurite documentation](https://learn.microsoft.com/azure/storage/common/storage-use-azurite)
  - Updated: December 2024
  - Local development and testing

- [GitHub repository](https://github.com/Azure/Azurite)

**AzCopy**
- [AzCopy documentation](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-v10)
  - Updated: December 2024
  - High-performance bulk transfer

- [Download AzCopy](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-v10#download-azcopy)

**Azure Storage REST API**
- [REST API reference](https://learn.microsoft.com/rest/api/storageservices/)
  - Updated: December 2024
  - Low-level API documentation

---

## Release Notes and What's New

**Azure Updates (Storage)**
- [Azure Updates - Storage](https://azure.microsoft.com/updates/?category=storage)
  - Latest features, deprecations

**SDK Release Notes**
- [Azure SDK for .NET releases](https://github.com/Azure/azure-sdk-for-net/releases)
- [Azure SDK for Python releases](https://github.com/Azure/azure-sdk-for-python/releases)
- [Azure SDK for JavaScript releases](https://github.com/Azure/azure-sdk-for-js/releases)

**Breaking Changes**
- [Azure Breaking Changes](https://azure.microsoft.com/updates/?updateType=breakingChanges&category=storage)
  - Monitor for API deprecations

---

## Support and Feedback

**Official Support**
- [Azure Support](https://azure.microsoft.com/support/options/)
  - Create support ticket (paid plans)

**Community Forums**
- [Microsoft Q&A - Azure Storage](https://learn.microsoft.com/answers/tags/170/azure-storage)
  - Community-driven Q&A

- [Stack Overflow - azure-storage](https://stackoverflow.com/questions/tagged/azure-storage)
  - Community troubleshooting

**Report Issues**
- [Azure SDK for .NET issues](https://github.com/Azure/azure-sdk-for-net/issues)
- [Azure SDK for Python issues](https://github.com/Azure/azure-sdk-for-python/issues)
- [Bicep issues](https://github.com/Azure/bicep/issues)

---

## Quick Reference Links

### Most Used Documentation (Bookmarks)

1. [Storage account overview](https://learn.microsoft.com/azure/storage/common/storage-account-overview)
2. [Assign RBAC roles](https://learn.microsoft.com/azure/storage/blobs/assign-azure-role-data-access)
3. [Managed identities](https://learn.microsoft.com/azure/storage/common/storage-auth-aad-msi)
4. [SAS best practices](https://learn.microsoft.com/azure/storage/common/storage-sas-best-practices)
5. [Lifecycle management](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-overview)
6. [Monitoring storage](https://learn.microsoft.com/azure/storage/blobs/monitor-blob-storage)
7. [Bicep storage templates](https://learn.microsoft.com/azure/templates/microsoft.storage/storageaccounts)
8. [.NET Blob SDK readme](https://learn.microsoft.com/dotnet/api/overview/azure/storage.blobs-readme)
9. [Table Storage design guide](https://learn.microsoft.com/azure/storage/tables/table-storage-design)
10. [Security baseline](https://learn.microsoft.com/security/benchmark/azure/baselines/storage-security-baseline)

### CLI Quick Reference

```bash
# Login
az login

# Create storage account
az storage account create --name <name> --resource-group <rg> --location <location>

# Upload blob
az storage blob upload --account-name <name> --container-name <container> --name <blob> --file <path> --auth-mode login

# List blobs
az storage blob list --account-name <name> --container-name <container> --auth-mode login

# Deploy Bicep
az deployment group create --resource-group <rg> --template-file main.bicep --parameters parameters.json
```

---

## Navigation

- **Previous**: `bicep-templates.md`
- **Next**: None (end of reference section)
- **Up**: `00-overview.md`

## See Also

- `08-reference/bicep-templates.md` - Deployable infrastructure templates
- `08-reference/code-examples.md` - Production-ready code samples
- `08-reference/checklists.md` - Security and operations checklists
- `00-overview.md` - Documentation overview
