# CI/CD - .NET 8

> **File Purpose**: GitHub Actions workflow for build, test, Docker build/push, and deployment
> **Prerequisites**: `docker.md`, `kubernetes.md` - Deployment artifacts ready
> **Related Files**: `../09-testing/integration-testing.md`
> **Agent Use Case**: Reference when setting up automated CI/CD pipelines for .NET 8 APIs

## Quick Context

Continuous Integration and Continuous Deployment automate building, testing, and deploying .NET 8 applications. This guide provides production-ready GitHub Actions workflows for building Docker images, running tests, scanning for vulnerabilities, and deploying to Kubernetes.

## Complete GitHub Actions Workflow

### .github/workflows/dotnet-ci-cd.yml

```yaml
name: .NET CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

env:
  DOTNET_VERSION: '8.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore --configuration Release

    - name: Run unit tests
      run: dotnet test --no-build --configuration Release --verbosity normal --filter "Category=Unit" --collect:"XPlat Code Coverage"

    - name: Run integration tests
      run: dotnet test --no-build --configuration Release --verbosity normal --filter "Category=Integration"

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage.cobertura.xml
        flags: unittests

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
    - uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build-docker:
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    permissions:
      contents: read
      packages: write
    steps:
    - uses: actions/checkout@v4

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./src/MyApi.Api/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-docker
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://api-staging.myapp.com
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Set AKS context
      uses: azure/aks-set-context@v3
      with:
        cluster-name: myapp-aks-staging
        resource-group: myapp-rg-staging

    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v4
      with:
        manifests: |
          k8s/deployment.yaml
          k8s/service.yaml
          k8s/ingress.yaml
        images: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        imagepullsecrets: |
          acr-secret

  deploy-production:
    runs-on: ubuntu-latest
    needs: build-docker
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.myapp.com
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS_PROD }}

    - name: Set AKS context
      uses: azure/aks-set-context@v3
      with:
        cluster-name: myapp-aks-prod
        resource-group: myapp-rg-prod

    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v4
      with:
        manifests: |
          k8s/deployment.yaml
          k8s/service.yaml
          k8s/ingress.yaml
        images: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        imagepullsecrets: |
          acr-secret
```

**Source**: [GitHub Actions Documentation](https://docs.github.com/en/actions) (2024)

## Secrets Configuration

### Required GitHub Secrets

```bash
# Azure credentials (Service Principal)
AZURE_CREDENTIALS='{"clientId":"xxx","clientSecret":"xxx","subscriptionId":"xxx","tenantId":"xxx"}'
AZURE_CREDENTIALS_PROD='{"clientId":"xxx","clientSecret":"xxx","subscriptionId":"xxx","tenantId":"xxx"}'

# Container registry
GITHUB_TOKEN  # Automatically provided by GitHub

# Database and services (for integration tests)
DATABASE_CONNECTION_STRING='Host=localhost;Database=test;...'
REDIS_CONNECTION_STRING='localhost:6379'

# JWT signing key
JWT_SECRET='your-secret-key-here'
```

## Alternative: Azure DevOps Pipeline

### azure-pipelines.yml

```yaml
trigger:
  branches:
    include:
    - main
    - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - task: UseDotNet@2
      inputs:
        version: $(dotnetVersion)

    - task: DotNetCoreCLI@2
      displayName: 'Restore packages'
      inputs:
        command: 'restore'

    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        arguments: '--configuration $(buildConfiguration) --no-restore'

    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

    - task: PublishCodeCoverageResults@1
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(Agent.TempDirectory)/**/*.cobertura.xml'

- stage: Docker
  dependsOn: Build
  jobs:
  - job: BuildAndPush
    steps:
    - task: Docker@2
      displayName: 'Build and Push'
      inputs:
        command: 'buildAndPush'
        repository: 'myapi'
        dockerfile: 'src/MyApi.Api/Dockerfile'
        containerRegistry: 'AzureContainerRegistry'
        tags: |
          $(Build.BuildId)
          latest

- stage: DeployStaging
  dependsOn: Docker
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  jobs:
  - deployment: DeployToStaging
    environment: 'staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'AKS-Staging'
              namespace: 'staging'
              manifests: |
                k8s/deployment.yaml
                k8s/service.yaml
```

## Environment-Specific Configuration

### GitHub Environments

1. Go to **Settings** ’ **Environments**
2. Create `staging` and `production` environments
3. Add protection rules:
   - Required reviewers for production
   - Wait timer before deployment
   - Deployment branches (only main for production)

## Rollback Strategy

### Kubernetes Rollback

```yaml
# Add to workflow for automatic rollback on health check failure
- name: Check deployment health
  run: |
    kubectl rollout status deployment/myapi -n production --timeout=5m

    # Check health endpoint
    HEALTH_URL=$(kubectl get ingress myapi -n production -o jsonpath='{.spec.rules[0].host}')
    if ! curl -f https://$HEALTH_URL/health/ready; then
      echo "Health check failed - rolling back"
      kubectl rollout undo deployment/myapi -n production
      exit 1
    fi
```

## Checklist

- [ ] CI workflow runs on every push and PR
- [ ] Unit and integration tests run automatically
- [ ] Code coverage tracked and reported
- [ ] Security scanning (Trivy/Snyk) integrated
- [ ] Docker images built and pushed to registry
- [ ] Automated deployment to staging on develop branch
- [ ] Manual approval required for production
- [ ] Health checks after deployment
- [ ] Automatic rollback on deployment failure
- [ ] Secrets stored in GitHub/Azure secrets

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions) (2024)
- [Azure DevOps Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/) (December 2023)
- [Docker Build Push Action](https://github.com/docker/build-push-action) (2024)
- [Kubernetes Deploy Action](https://github.com/Azure/k8s-deploy) (2024)

---

**Next Steps**: Review `../09-testing/integration-testing.md` for test setup, or `docker.md` for container optimization.
