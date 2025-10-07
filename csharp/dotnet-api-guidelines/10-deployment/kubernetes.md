# Kubernetes - .NET 8

> **File Purpose**: Kubernetes deployment manifests, ConfigMaps, Secrets, probes, and resource limits
> **Prerequisites**: `docker.md` - Container image ready
> **Related Files**: `ci-cd.md`, `../07-observability/health-checks.md`
> **Agent Use Case**: Reference when deploying .NET 8 APIs to Kubernetes clusters

## Quick Context

Kubernetes orchestrates containerized applications with automatic scaling, self-healing, and rolling updates. This guide provides production-ready manifests for deploying .NET 8 APIs with proper health checks, resource limits, secrets management, and zero-downtime deployments.

## Deployment Manifest

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  namespace: production
  labels:
    app: myapi
    version: v1
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapi
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: api
        image: myregistry.azurecr.io/myapi:latest
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ASPNETCORE_URLS
          value: "http://+:8080"
        envFrom:
        - configMapRef:
            name: myapi-config
        - secretRef:
            name: myapi-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      imagePullSecrets:
      - name: acr-secret
```

**Source**: [Microsoft Docs - Kubernetes](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/deploy-application-kubernetes) (October 2023)

## Service

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapi
  namespace: production
  labels:
    app: myapi
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: myapi
```

## ConfigMap and Secrets

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
  namespace: production
data:
  ConnectionStrings__DefaultConnection: "Host=postgres.production.svc.cluster.local;Database=myapi;Port=5432;Username=myapi"
  ConnectionStrings__Redis: "redis.production.svc.cluster.local:6379"
  Jwt__Issuer: "https://api.myapp.com"
  Jwt__Audience: "https://myapp.com"
  Serilog__MinimumLevel__Default: "Information"
```

### secret.yaml (Base64 encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
  namespace: production
type: Opaque
data:
  ConnectionStrings__DefaultConnection--Password: cGFzc3dvcmQxMjM=  # password123
  ConnectionStrings__Redis--Password: cmVkaXNwYXNz  # redispass
  Jwt__Secret: c3VwZXJzZWNyZXRrZXk=  # supersecretkey
```

```bash
# Create secret from file
kubectl create secret generic myapi-secrets \
  --from-file=jwt-secret=./secrets/jwt.key \
  --namespace=production
```

## Ingress

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapi-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapi
            port:
              number: 80
```

## Horizontal Pod Autoscaler

### hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

## Health Checks (.NET 8)

### Program.cs Health Endpoints

```csharp
// Map health check endpoints for Kubernetes probes
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Liveness: always healthy if process is running
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/startup", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("startup")
});

// Register health checks
builder.Services.AddHealthChecks()
    .AddNpgSql(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready", "startup" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: new[] { "ready" });
```

**Source**: [Microsoft Docs - Health checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) (November 2023)

## Resource Quotas

### resourcequota.yaml

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
```

## Network Policy

### networkpolicy.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapi
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapi
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

## Zero-Downtime Deployment

### Rolling Update Strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Max 1 extra pod during update
    maxUnavailable: 0    # Always keep 100% available
```

### Graceful Shutdown (Program.cs)

```csharp
builder.Host.ConfigureHostOptions(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// Handle SIGTERM gracefully
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
lifetime.ApplicationStopping.Register(() =>
{
    // Give Kubernetes time to remove pod from service endpoints
    Thread.Sleep(TimeSpan.FromSeconds(5));
});
```

## Checklist

- [ ] Deployment with 3+ replicas for high availability
- [ ] Rolling update strategy for zero-downtime
- [ ] Liveness, readiness, and startup probes configured
- [ ] Resource requests and limits set
- [ ] ConfigMap for non-sensitive configuration
- [ ] Secrets for sensitive data (base64 encoded)
- [ ] Horizontal Pod Autoscaler for scaling
- [ ] Ingress with TLS termination
- [ ] Network policies restrict traffic
- [ ] Service account with RBAC permissions
- [ ] Read-only root filesystem enabled
- [ ] Non-root user (runAsUser: 1000)

## References

- [Microsoft Docs - Kubernetes](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/deploy-application-kubernetes) (October 2023)
- [Microsoft Docs - Health checks](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks) (November 2023)
- [Kubernetes Documentation](https://kubernetes.io/docs/concepts/) (2024)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/) (2024)

---

**Next Steps**: Review `ci-cd.md` for automated deployment pipelines, or `docker.md` for container optimization.
