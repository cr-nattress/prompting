# Docker - .NET 8

> **File Purpose**: Production-ready Docker multi-stage builds, optimization, and security
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Project scaffolded
> **Related Files**: `kubernetes.md`, `ci-cd.md`, `../07-observability/health-checks.md`
> **Agent Use Case**: Reference when containerizing .NET 8 APIs for production deployment

## Quick Context

Docker containers provide consistent deployment environments across development, testing, and production. .NET 8 offers optimized base images and runtime features specifically designed for containerization. This guide covers multi-stage builds, security hardening, size optimization, and production best practices.

## Multi-Stage Dockerfile (.NET 8)

### Production-Optimized Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj files and restore (layer caching)
COPY ["src/MyApi.Api/MyApi.Api.csproj", "src/MyApi.Api/"]
COPY ["src/MyApi.Application/MyApi.Application.csproj", "src/MyApi.Application/"]
COPY ["src/MyApi.Domain/MyApi.Domain.csproj", "src/MyApi.Domain/"]
COPY ["src/MyApi.Infrastructure/MyApi.Infrastructure.csproj", "src/MyApi.Infrastructure/"]
RUN dotnet restore "src/MyApi.Api/MyApi.Api.csproj"

# Copy everything else and build
COPY src/ .
WORKDIR "/src/src/MyApi.Api"
RUN dotnet build "MyApi.Api.csproj" -c Release -o /app/build --no-restore

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApi.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    --no-build \
    /p:UseAppHost=false \
    /p:PublishTrimmed=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS final
WORKDIR /app

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -u 1000 -G appuser -s /bin/sh -D appuser && \
    chown -R appuser:appuser /app

# Install security updates (Alpine)
RUN apk upgrade --no-cache

# Copy published app
COPY --from=publish --chown=appuser:appuser /app/publish .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Expose port (non-root port)
EXPOSE 8080

# Set environment
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1

ENTRYPOINT ["dotnet", "MyApi.Api.dll"]
```

**Source**: [Microsoft Docs - Docker](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container) (December 2023)

### .dockerignore

```
# .dockerignore
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.vs
**/.vscode
**/*.*proj.user
**/bin
**/obj
**/Dockerfile*
**/docker-compose*
**/*.md
**/node_modules
**/npm-debug.log
**/appsettings.Development.json
**/secrets.json
**/.secret
**/.secrets
```

## Image Optimization

### Alpine-based Image (Smaller)

```dockerfile
# Runtime: 108MB (Alpine) vs 216MB (Debian)
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS final
```

### Chiseled Ubuntu Images (Even Smaller)

```dockerfile
# Chiseled: ultra-small, no shell, non-root by default
FROM mcr.microsoft.com/dotnet/nightly/aspnet:8.0-jammy-chiseled AS final
```

## Docker Compose for Local Development

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: src/MyApi.Api/Dockerfile
      target: final
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=myapi;Username=myapi;Password=dev_password
      - ConnectionStrings__Redis=redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapi
      POSTGRES_USER: myapi
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapi"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  postgres_data:
  redis_data:
```

## Security Hardening

### Non-Root User (Required)

```dockerfile
# Create and switch to non-root user
RUN addgroup -g 1000 appuser && \
    adduser -u 1000 -G appuser -s /bin/sh -D appuser && \
    chown -R appuser:appuser /app

USER appuser
```

### Read-Only Root Filesystem

```yaml
# Kubernetes deployment
spec:
  template:
    spec:
      containers:
      - name: api
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

### Scanning for Vulnerabilities

```bash
# Trivy scan
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapi:latest

# Snyk scan
docker scan myapi:latest
```

## Production Configuration

### appsettings.Production.json (Not in Image)

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### Environment Variables (Runtime)

```bash
docker run -d \
  -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="Host=prod-db;Database=myapi;..." \
  -e ConnectionStrings__Redis="prod-redis:6379" \
  -e JWT__Secret="$(cat /run/secrets/jwt-secret)" \
  myapi:latest
```

## Checklist

- [ ] Multi-stage build separates build and runtime
- [ ] Alpine or chiseled base image for smaller size
- [ ] Non-root user (appuser) used in runtime
- [ ] .dockerignore excludes unnecessary files
- [ ] Health check endpoint configured
- [ ] Security updates installed in image
- [ ] Secrets passed via environment variables (not in image)
- [ ] Image scanned for vulnerabilities (Trivy/Snyk)
- [ ] Docker Compose for local development
- [ ] Port 8080 used (non-privileged)

## References

- [Microsoft Docs - Docker](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container) (December 2023)
- [Microsoft Docs - Container images](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/container-docker-introduction/docker-containers-images-registries) (November 2023)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) (2024)
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) (2024)

---

**Next Steps**: Review `kubernetes.md` for orchestration, or `ci-cd.md` for automated builds.
