# CI/CD Patterns for Azure Storage

> **File Purpose**: Production-ready CI/CD pipelines for Azure Storage infrastructure and applications
> **Prerequisites**: GitHub repository, Azure subscription, Bicep or Terraform knowledge
> **Agent Use Case**: Reference when setting up automated deployments and environment promotion

## Quick Context

Modern Azure Storage deployments require automated, repeatable pipelines that promote infrastructure and code through environments (dev → staging → prod). This guide provides complete GitHub Actions workflows using OpenID Connect (OIDC) for passwordless authentication, eliminating secrets from repositories.

**Key principle**: Infrastructure as Code (IaC) deployments should be idempotent, versioned, and include drift detection. Always use workload identity federation over service principal secrets.

## OIDC Authentication Setup

### Why OIDC Over Service Principals

| Authentication Method | Security | Rotation | Best For |
|----------------------|----------|----------|----------|
| Service Principal Secret | Secret stored in GitHub | Manual every 90 days | Legacy systems |
| Certificate-based SP | Cert stored in GitHub | Manual every year | Regulated environments |
| **OIDC (Federated Identity)** | No secrets in GitHub | Automatic token exchange | All new projects |

**OIDC wins**: No credential management, automatic token rotation, least privilege access per workflow.

### Configure Azure for OIDC

```bash
#!/bin/bash
# setup-oidc.sh - Configure workload identity federation

SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="rg-storage-prod"
APP_NAME="github-storage-deploy"
GITHUB_ORG="your-org"
GITHUB_REPO="your-repo"

# Login to Azure
az login
az account set --subscription "$SUBSCRIPTION_ID"

# Create Azure AD Application
APP_ID=$(az ad app create --display-name "$APP_NAME" --query appId -o tsv)
echo "Application ID: $APP_ID"

# Create Service Principal
SP_ID=$(az ad sp create --id "$APP_ID" --query id -o tsv)
echo "Service Principal ID: $SP_ID"

# Add federated credentials for each environment/branch
# Production: main branch
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Staging: staging branch
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-staging",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':ref:refs/heads/staging",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Pull Request validation
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-pr",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':pull_request",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Assign RBAC permissions (Contributor at Resource Group scope)
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# Additional roles for Key Vault and Storage
az role assignment create \
  --assignee "$APP_ID" \
  --role "Key Vault Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "OIDC Setup Complete!"
echo "Add these secrets to GitHub repository:"
echo "  AZURE_CLIENT_ID: $APP_ID"
echo "  AZURE_TENANT_ID: $(az account show --query tenantId -o tsv)"
echo "  AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
```

**Why it works**: Subject claim matches GitHub context (branch, PR). Azure validates token issuer and subject before granting access.

### GitHub Repository Secrets

Add these as **repository secrets** (Settings → Secrets and variables → Actions):

```plaintext
AZURE_CLIENT_ID: <app-id-from-above>
AZURE_TENANT_ID: <tenant-id>
AZURE_SUBSCRIPTION_ID: <subscription-id>
```

**Never store**: Service principal secrets, storage account keys, connection strings in GitHub.

## GitHub Actions Workflow - Infrastructure Deployment

### Complete Bicep Deployment Pipeline

**.github/workflows/deploy-infrastructure.yml**:

```yaml
name: Deploy Azure Storage Infrastructure

on:
  push:
    branches: [main, staging]
    paths:
      - 'infrastructure/**'
      - '.github/workflows/deploy-infrastructure.yml'
  pull_request:
    branches: [main]
    paths:
      - 'infrastructure/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

permissions:
  id-token: write   # Required for OIDC
  contents: read    # Required to checkout code
  pull-requests: write  # Required for PR comments

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Bicep Build
        run: |
          az bicep build --file infrastructure/main.bicep

      - name: Bicep Lint
        run: |
          az bicep lint --file infrastructure/main.bicep

      - name: What-If Analysis
        id: whatif
        run: |
          az deployment group what-if \
            --resource-group rg-storage-${{ github.ref_name }} \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/parameters.${{ github.ref_name }}.json \
            --result-format FullResourcePayloads \
            --no-pretty-print > whatif-output.json

          cat whatif-output.json

      - name: Comment What-If on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const whatif = fs.readFileSync('whatif-output.json', 'utf8');

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Bicep What-If Analysis\n\`\`\`json\n${whatif}\n\`\`\``
            });

  deploy-dev:
    needs: validate
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Bicep
        id: deploy
        run: |
          az deployment group create \
            --resource-group rg-storage-dev \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/parameters.dev.json \
            --parameters deploymentTimestamp=$(date +%s) \
            --query 'properties.outputs' -o json > outputs.json

          cat outputs.json

      - name: Upload Deployment Outputs
        uses: actions/upload-artifact@v4
        with:
          name: deployment-outputs-dev
          path: outputs.json

      - name: Verify Deployment
        run: |
          STORAGE_ACCOUNT=$(jq -r '.storageAccountName.value' outputs.json)

          # Check storage account exists
          az storage account show --name $STORAGE_ACCOUNT --resource-group rg-storage-dev

          # Verify private endpoint
          PE_STATE=$(az network private-endpoint show \
            --name pe-${STORAGE_ACCOUNT}-blob \
            --resource-group rg-storage-dev \
            --query 'provisioningState' -o tsv)

          if [ "$PE_STATE" != "Succeeded" ]; then
            echo "Private endpoint provisioning failed!"
            exit 1
          fi

  deploy-staging:
    needs: deploy-dev
    if: github.ref == 'refs/heads/staging' || github.event.inputs.environment == 'staging'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Staging
        run: |
          az deployment group create \
            --resource-group rg-storage-staging \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/parameters.staging.json \
            --parameters deploymentTimestamp=$(date +%s)

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://portal.azure.com/#@/resource/subscriptions/${{ secrets.AZURE_SUBSCRIPTION_ID }}/resourceGroups/rg-storage-prod
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Pre-deployment Backup
        run: |
          # Tag current configuration before deployment
          STORAGE_ACCOUNT=$(az storage account list \
            --resource-group rg-storage-prod \
            --query "[0].name" -o tsv)

          az tag create \
            --resource-id "/subscriptions/${{ secrets.AZURE_SUBSCRIPTION_ID }}/resourceGroups/rg-storage-prod/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT" \
            --tags "LastDeployment=$(date -u +%Y-%m-%dT%H:%M:%SZ)" "DeploymentRun=${{ github.run_number }}"

      - name: Deploy to Production
        id: deploy
        run: |
          az deployment group create \
            --resource-group rg-storage-prod \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/parameters.prod.json \
            --parameters deploymentTimestamp=$(date +%s) \
            --query 'properties.outputs' -o json > outputs-prod.json

      - name: Post-Deployment Validation
        run: |
          STORAGE_ACCOUNT=$(jq -r '.storageAccountName.value' outputs-prod.json)

          # Test blob endpoint connectivity
          az storage blob list \
            --account-name $STORAGE_ACCOUNT \
            --container-name system-health \
            --auth-mode login \
            --only-show-errors

          # Verify geo-replication status
          GRS_STATUS=$(az storage account show \
            --name $STORAGE_ACCOUNT \
            --resource-group rg-storage-prod \
            --query 'statusOfPrimary' -o tsv)

          if [ "$GRS_STATUS" != "available" ]; then
            echo "Storage account not fully available!"
            exit 1
          fi

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ github.run_number }}
          name: Production Deployment ${{ github.run_number }}
          body: |
            Infrastructure deployed to production
            Commit: ${{ github.sha }}
            Deployed by: ${{ github.actor }}
          files: outputs-prod.json
```

**Why it works**:
- OIDC eliminates secrets
- What-If analysis prevents surprises
- Environment protection rules enforce approvals
- Deployment verification catches issues early
- Artifacts enable rollback

## Infrastructure Drift Detection

### Detect Configuration Drift

**.github/workflows/drift-detection.yml**:

```yaml
name: Detect Infrastructure Drift

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  issues: write

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Export Current State
        run: |
          az deployment group export \
            --resource-group rg-storage-${{ matrix.environment }} \
            --name current-state > current-state.json

      - name: What-If Against IaC
        id: whatif
        run: |
          az deployment group what-if \
            --resource-group rg-storage-${{ matrix.environment }} \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/parameters.${{ matrix.environment }}.json \
            --result-format ResourceIdOnly > drift-report.txt

          # Check if drift exists
          if grep -q "Resource changes:" drift-report.txt; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
          fi

      - name: Create Issue for Drift
        if: steps.whatif.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const drift = fs.readFileSync('drift-report.txt', 'utf8');

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Infrastructure Drift Detected - ${{ matrix.environment }}`,
              body: `## Drift Report\n\nEnvironment: **${{ matrix.environment }}**\n\n\`\`\`\n${drift}\n\`\`\`\n\n**Action Required**: Review and update IaC templates or remediate manual changes.`,
              labels: ['infrastructure', 'drift', '${{ matrix.environment }}']
            });
```

**Why it works**: Automated daily checks catch configuration drift before it causes production issues.

## Secrets Management with Key Vault

### Key Vault Integration Pattern

**.github/workflows/deploy-app.yml** (excerpt):

```yaml
jobs:
  deploy-app:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Retrieve Secrets from Key Vault
        id: keyvault
        run: |
          # Get storage account key for deployment (avoid in production)
          STORAGE_KEY=$(az keyvault secret show \
            --vault-name kv-storage-prod \
            --name storage-account-key \
            --query 'value' -o tsv)

          echo "::add-mask::$STORAGE_KEY"
          echo "STORAGE_KEY=$STORAGE_KEY" >> $GITHUB_ENV

      - name: Deploy Application Configuration
        run: |
          # Use Key Vault reference in App Service config
          az webapp config appsettings set \
            --name app-storage-prod \
            --resource-group rg-storage-prod \
            --settings \
              "Azure__Storage__BlobUri=@Microsoft.KeyVault(VaultName=kv-storage-prod;SecretName=blob-uri)" \
              "Azure__Storage__TableUri=@Microsoft.KeyVault(VaultName=kv-storage-prod;SecretName=table-uri)"
```

**Why it works**: Secrets stay in Key Vault. App Service resolves references at runtime using Managed Identity.

### Automated Secret Rotation

```yaml
name: Rotate Storage Keys

on:
  schedule:
    - cron: '0 3 1 * *'  # Monthly on 1st at 3 AM
  workflow_dispatch:

jobs:
  rotate-keys:
    runs-on: ubuntu-latest
    steps:
      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Rotate Key2 and Update Key Vault
        run: |
          STORAGE_ACCOUNT="stprodapp001"

          # Regenerate key2
          NEW_KEY=$(az storage account keys renew \
            --resource-group rg-storage-prod \
            --account-name $STORAGE_ACCOUNT \
            --key secondary \
            --query '[0].value' -o tsv)

          # Update Key Vault
          az keyvault secret set \
            --vault-name kv-storage-prod \
            --name storage-account-key \
            --value "$NEW_KEY"

      - name: Wait for Applications to Pick Up New Key
        run: sleep 300  # 5 minutes

      - name: Rotate Key1
        run: |
          az storage account keys renew \
            --resource-group rg-storage-prod \
            --account-name stprodapp001 \
            --key primary
```

**Why it works**: Dual-key rotation with grace period prevents downtime.

## Terraform Alternative Pattern

### Terraform with OIDC

**main.tf**:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstate001"
    container_name       = "tfstate"
    key                  = "storage.tfstate"
    use_oidc             = true
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}

variable "environment" {
  type = string
}

resource "azurerm_storage_account" "main" {
  name                     = "st${var.environment}app001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GZRS" : "LRS"

  identity {
    type = "SystemAssigned"
  }

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }
}

resource "azurerm_private_endpoint" "blob" {
  name                = "pe-${azurerm_storage_account.main.name}-blob"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  subnet_id           = azurerm_subnet.endpoints.id

  private_service_connection {
    name                           = "psc-blob"
    private_connection_resource_id = azurerm_storage_account.main.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}
```

**.github/workflows/terraform.yml**:

```yaml
name: Terraform Deployment

on:
  push:
    branches: [main]
  pull_request:

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      ARM_USE_OIDC: true

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -var="environment=${{ github.ref_name }}" -out=tfplan
          terraform show -no-color tfplan > plan-output.txt

      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan-output.txt', 'utf8');

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`hcl\n${plan}\n\`\`\``
            });

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

## Azure DevOps Pipelines Alternative

### Azure Pipelines with Workload Identity

**azure-pipelines.yml**:

```yaml
trigger:
  branches:
    include:
      - main
      - staging
  paths:
    include:
      - infrastructure/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: azure-credentials  # Variable group with service connection
  - name: environmentName
    ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
      value: 'prod'
    ${{ else }}:
      value: 'staging'

stages:
  - stage: Validate
    jobs:
      - job: ValidateBicep
        steps:
          - task: AzureCLI@2
            displayName: 'Bicep Lint and What-If'
            inputs:
              azureSubscription: 'azure-storage-connection'  # Service connection
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infrastructure/main.bicep

                az deployment group what-if \
                  --resource-group rg-storage-$(environmentName) \
                  --template-file infrastructure/main.bicep \
                  --parameters infrastructure/parameters.$(environmentName).json

  - stage: Deploy
    dependsOn: Validate
    condition: succeeded()
    jobs:
      - deployment: DeployInfrastructure
        environment: $(environmentName)
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: AzureCLI@2
                  displayName: 'Deploy Bicep Template'
                  inputs:
                    azureSubscription: 'azure-storage-connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group rg-storage-$(environmentName) \
                        --template-file infrastructure/main.bicep \
                        --parameters infrastructure/parameters.$(environmentName).json \
                        --parameters deploymentTimestamp=$(date +%s)

                - task: AzureCLI@2
                  displayName: 'Verify Deployment'
                  inputs:
                    azureSubscription: 'azure-storage-connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      STORAGE_ACCOUNT=$(az storage account list \
                        --resource-group rg-storage-$(environmentName) \
                        --query "[0].name" -o tsv)

                      az storage account show \
                        --name $STORAGE_ACCOUNT \
                        --resource-group rg-storage-$(environmentName)

  - stage: PostDeploy
    dependsOn: Deploy
    jobs:
      - job: Smoke Tests
        steps:
          - task: AzureCLI@2
            displayName: 'Run Smoke Tests'
            inputs:
              azureSubscription: 'azure-storage-connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Test blob upload
                echo "Test data" > test.txt
                az storage blob upload \
                  --account-name stprodapp001 \
                  --container-name smoke-test \
                  --name test-$(Build.BuildId).txt \
                  --file test.txt \
                  --auth-mode login
```

**Create Service Connection with Workload Identity**:

```bash
# Create service connection in Azure DevOps
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id $APP_ID \
  --azure-rm-subscription-id $SUBSCRIPTION_ID \
  --azure-rm-tenant-id $TENANT_ID \
  --name "azure-storage-connection" \
  --organization https://dev.azure.com/your-org \
  --project your-project
```

## Deployment Verification and Rollback

### Health Check After Deployment

```yaml
# .github/workflows/health-check.yml
name: Post-Deployment Health Check

on:
  workflow_run:
    workflows: ["Deploy Azure Storage Infrastructure"]
    types:
      - completed

jobs:
  health-check:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Test Storage Account Connectivity
        id: health
        run: |
          STORAGE_ACCOUNT="stprodapp001"
          HEALTH_STATUS="healthy"

          # Test blob service
          if ! az storage blob list \
            --account-name $STORAGE_ACCOUNT \
            --container-name system-health \
            --auth-mode login \
            --num-results 1 > /dev/null 2>&1; then
            HEALTH_STATUS="unhealthy"
          fi

          # Test table service
          if ! az storage table list \
            --account-name $STORAGE_ACCOUNT \
            --auth-mode login \
            --num-results 1 > /dev/null 2>&1; then
            HEALTH_STATUS="unhealthy"
          fi

          # Check private endpoint status
          PE_STATE=$(az network private-endpoint show \
            --name pe-${STORAGE_ACCOUNT}-blob \
            --resource-group rg-storage-prod \
            --query 'provisioningState' -o tsv)

          if [ "$PE_STATE" != "Succeeded" ]; then
            HEALTH_STATUS="unhealthy"
          fi

          echo "status=$HEALTH_STATUS" >> $GITHUB_OUTPUT

          if [ "$HEALTH_STATUS" = "unhealthy" ]; then
            exit 1
          fi

      - name: Trigger Rollback on Failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'rollback.yml',
              ref: 'main',
              inputs: {
                deployment_id: '${{ github.event.workflow_run.id }}',
                reason: 'Health check failed'
              }
            });
```

### Automated Rollback

```yaml
# .github/workflows/rollback.yml
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      deployment_id:
        description: 'Deployment ID to rollback'
        required: true
      reason:
        description: 'Reason for rollback'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for rollback

      - name: Azure Login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get Previous Deployment
        id: previous
        run: |
          # Get last successful deployment
          PREV_DEPLOYMENT=$(az deployment group list \
            --resource-group rg-storage-prod \
            --query "[?properties.provisioningState=='Succeeded'] | [1].name" -o tsv)

          echo "deployment=$PREV_DEPLOYMENT" >> $GITHUB_OUTPUT

      - name: Rollback to Previous State
        run: |
          # Re-deploy previous template
          az deployment group create \
            --resource-group rg-storage-prod \
            --name rollback-$(date +%s) \
            --template-file infrastructure/main.bicep \
            --parameters @${{ steps.previous.outputs.deployment }}.parameters.json \
            --mode Incremental

      - name: Create Rollback Issue
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Rollback Executed - Deployment ${{ inputs.deployment_id }}`,
              body: `## Rollback Details\n\nReason: ${{ inputs.reason }}\n\nRolled back to: ${{ steps.previous.outputs.deployment }}\n\nTriggered by: ${{ github.actor }}`,
              labels: ['rollback', 'incident']
            });
```

## CI/CD Checklist

### Pre-Deployment
- [ ] OIDC federated credentials configured for all branches/environments
- [ ] GitHub repository secrets added (client ID, tenant ID, subscription ID)
- [ ] Bicep/Terraform templates validated and linted
- [ ] Parameter files exist for each environment (dev, staging, prod)
- [ ] What-If analysis reviewed for breaking changes
- [ ] Environment protection rules configured in GitHub

### Deployment
- [ ] Deployment runs What-If before applying changes
- [ ] Deployment outputs captured as artifacts
- [ ] Private endpoint provisioning verified
- [ ] Storage account redundancy matches environment (LRS dev, GZRS prod)
- [ ] Network access restrictions applied (deny public by default)

### Post-Deployment
- [ ] Health checks validate blob and table connectivity
- [ ] Geo-replication status confirmed (if applicable)
- [ ] Drift detection scheduled (daily runs)
- [ ] Rollback workflow tested in non-prod
- [ ] Deployment tagged with version and run number

### Security
- [ ] No secrets stored in GitHub (use OIDC only)
- [ ] Key Vault used for runtime secrets (connection strings, SAS)
- [ ] Service principal follows least privilege (Contributor at RG scope)
- [ ] Storage account keys rotated monthly (automated)
- [ ] Diagnostic logs flowing to Log Analytics

## Navigation

- **Previous**: `06-operations/governance.md`
- **Next**: `testing-strategy.md`
- **Up**: `00-overview.md`

## See Also

- `01-quick-start/provisioning.md` - Bicep template reference
- `05-security/identity-authentication.md` - Managed Identity patterns
- `testing-strategy.md` - Azurite and integration tests
- `08-reference/bicep-templates.md` - Complete IaC examples

## References

- [Workload Identity Federation for GitHub Actions](https://learn.microsoft.com/azure/developer/github/connect-from-azure?tabs=azure-cli%2Cwindows) - Microsoft Learn
- [Azure Bicep What-If Operations](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if) - Microsoft Learn
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) - GitHub Docs
- [Terraform Azure Provider OIDC](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_oidc) - HashiCorp
