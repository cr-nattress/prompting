# OpenTelemetry - .NET 8

> **File Purpose**: Implement OpenTelemetry for distributed tracing, metrics, and OTLP exporters
> **Prerequisites**: `structured-logging.md` - Logging configured
> **Related Files**: `health-checks.md`, `../08-performance/resilience.md`
> **Agent Use Case**: Reference when implementing distributed tracing and metrics collection

## Quick Context

OpenTelemetry (OTel) provides vendor-neutral instrumentation for traces, metrics, and logs. .NET 8 has native OpenTelemetry support, enabling distributed tracing across microservices and integration with observability platforms (Jaeger, Zipkin, Prometheus, Application Insights).

**Microsoft References**:
- [.NET Observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [OpenTelemetry .NET](https://github.com/open-telemetry/opentelemetry-dotnet)

## Install Packages

```bash
# Core packages
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http

# Exporters
dotnet add package OpenTelemetry.Exporter.Console
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol # OTLP
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

# Entity Framework instrumentation
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
```

## Basic Configuration

```csharp
// Program.cs
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

var builder = WebApplication.CreateBuilder(args);

// Configure OpenTelemetry
var serviceName = "MyApi";
var serviceVersion = "1.0.0";

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: Environment.MachineName
        )
    )
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddConsoleExporter()
    )
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddConsoleExporter()
    );

var app = builder.Build();
app.Run();
```

## Distributed Tracing

### Automatic Instrumentation

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        // ASP.NET Core - traces HTTP requests automatically
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.EnrichWithHttpRequest = (activity, httpRequest) =>
            {
                activity.SetTag("http.client_ip", httpRequest.HttpContext.Connection.RemoteIpAddress);
                activity.SetTag("http.user_agent", httpRequest.Headers["User-Agent"].ToString());
            };
            options.EnrichWithHttpResponse = (activity, httpResponse) =>
            {
                activity.SetTag("http.response_length", httpResponse.ContentLength);
            };
        })
        // HttpClient - traces outbound HTTP calls
        .AddHttpClientInstrumentation(options =>
        {
            options.RecordException = true;
            options.EnrichWithHttpRequestMessage = (activity, httpRequestMessage) =>
            {
                activity.SetTag("http.request.method", httpRequestMessage.Method.ToString());
            };
        })
        // Entity Framework Core - traces database queries
        .AddEntityFrameworkCoreInstrumentation(options =>
        {
            options.SetDbStatementForText = true;
            options.SetDbStatementForStoredProcedure = true;
            options.EnrichWithIDbCommand = (activity, command) =>
            {
                activity.SetTag("db.query_type", command.CommandType.ToString());
            };
        })
    );
```

### Manual Instrumentation

```csharp
using System.Diagnostics;

// Create ActivitySource at class level
public class OrderService : IOrderService
{
    private static readonly ActivitySource ActivitySource = new("MyApi.OrderService");
    private readonly IOrderRepository _repository;

    public async Task<Order> ProcessOrderAsync(ProcessOrderRequest request, CancellationToken ct)
    {
        using var activity = ActivitySource.StartActivity("ProcessOrder");

        activity?.SetTag("order.id", request.OrderId);
        activity?.SetTag("order.amount", request.Amount);

        try
        {
            // Validation span
            using (var validationActivity = ActivitySource.StartActivity("ValidateOrder"))
            {
                validationActivity?.SetTag("order.item_count", request.Items.Count);
                await ValidateOrderAsync(request, ct);
            }

            // Payment span
            using (var paymentActivity = ActivitySource.StartActivity("ProcessPayment"))
            {
                paymentActivity?.SetTag("payment.method", request.PaymentMethod);
                await ProcessPaymentAsync(request, ct);
            }

            // Save span
            var order = await _repository.SaveOrderAsync(request, ct);

            activity?.SetTag("order.status", "completed");
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}

// Register ActivitySource
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("MyApi.OrderService")
        .AddSource("MyApi.*") // All custom sources
    );
```

## Metrics

### Built-in Metrics

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        // ASP.NET Core metrics
        .AddAspNetCoreInstrumentation()
        // Runtime metrics (GC, thread pool, etc.)
        .AddRuntimeInstrumentation()
        // Process metrics (CPU, memory)
        .AddProcessInstrumentation()
        // HttpClient metrics
        .AddHttpClientInstrumentation()
    );
```

### Custom Metrics

```csharp
using System.Diagnostics.Metrics;

// Create Meter at class level
public class OrderMetrics
{
    private static readonly Meter Meter = new("MyApi.Orders", "1.0.0");

    private static readonly Counter<long> OrdersCreated = Meter.CreateCounter<long>(
        "orders.created",
        description: "Number of orders created"
    );

    private static readonly Histogram<double> OrderAmount = Meter.CreateHistogram<double>(
        "orders.amount",
        unit: "USD",
        description: "Order amount distribution"
    );

    private static readonly ObservableGauge<int> PendingOrders = Meter.CreateObservableGauge<int>(
        "orders.pending",
        () => GetPendingOrderCount(),
        description: "Current number of pending orders"
    );

    public void RecordOrderCreated(Order order)
    {
        OrdersCreated.Add(1, new KeyValuePair<string, object?>("order.type", order.Type));
        OrderAmount.Record(order.TotalAmount, new KeyValuePair<string, object?>("currency", "USD"));
    }

    private static int GetPendingOrderCount()
    {
        // Query database or cache
        return 0;
    }
}

// Register
builder.Services.AddSingleton<OrderMetrics>();

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("MyApi.Orders")
    );
```

## Exporters

### OTLP Exporter (Jaeger, Tempo, etc.)

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:4317"); // gRPC endpoint
            options.Protocol = OtlpExportProtocol.Grpc;
        })
    )
    .WithMetrics(metrics => metrics
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:4317");
        })
    );
```

### Prometheus Metrics

```csharp
using OpenTelemetry.Exporter;

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddPrometheusExporter()
    );

var app = builder.Build();

// Expose /metrics endpoint for Prometheus scraping
app.MapPrometheusScrapingEndpoint();
```

### Application Insights

```bash
dotnet add package Azure.Monitor.OpenTelemetry.Exporter
```

```csharp
using Azure.Monitor.OpenTelemetry.Exporter;

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAzureMonitorTraceExporter(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        })
    )
    .WithMetrics(metrics => metrics
        .AddAzureMonitorMetricExporter(options =>
        {
            options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
        })
    );
```

## Complete Production Example

```csharp
// Program.cs
using OpenTelemetry.Exporter;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using System.Diagnostics;

var builder = WebApplication.CreateBuilder(args);

var serviceName = builder.Configuration["ServiceName"] ?? "MyApi";
var serviceVersion = builder.Configuration["ServiceVersion"] ?? "1.0.0";
var otlpEndpoint = builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317";

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: serviceName,
            serviceVersion: serviceVersion,
            serviceInstanceId: Environment.MachineName
        )
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = builder.Environment.EnvironmentName,
            ["host.name"] = Environment.MachineName
        })
    )
    .WithTracing(tracing => tracing
        .SetSampler(new TraceIdRatioBasedSampler(1.0)) // 100% sampling in dev
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            options.Filter = httpContext =>
            {
                // Don't trace health checks
                return !httpContext.Request.Path.StartsWithSegments("/health");
            };
            options.EnrichWithHttpRequest = (activity, httpRequest) =>
            {
                activity.SetTag("http.client_ip", httpRequest.HttpContext.Connection.RemoteIpAddress);
                activity.SetTag("correlation_id", httpRequest.HttpContext.Items["CorrelationId"]);
            };
        })
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation(options =>
        {
            options.SetDbStatementForText = true;
        })
        .AddSource("MyApi.*") // Custom sources
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(otlpEndpoint);
            options.Protocol = OtlpExportProtocol.Grpc;
        })
        .AddConsoleExporter() // Development only
    )
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddMeter("MyApi.*") // Custom meters
        .AddPrometheusExporter()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(otlpEndpoint);
        })
    );

var app = builder.Build();

// Expose Prometheus metrics
app.MapPrometheusScrapingEndpoint();

app.Run();
```

## Correlation with Logging

```csharp
// Enrich Serilog with trace context
using Serilog;
using OpenTelemetry;

builder.Host.UseSerilog((context, services, configuration) => configuration
    .Enrich.WithProperty("ServiceName", serviceName)
    .Enrich.WithProperty("ServiceVersion", serviceVersion)
    .Enrich.FromLogContext()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} " +
        "{Properties} " +
        "TraceId={TraceId} SpanId={SpanId} " +
        "{NewLine}{Exception}"
    )
);

// In middleware, add trace context to LogContext
app.Use(async (context, next) =>
{
    var activity = Activity.Current;
    if (activity is not null)
    {
        using (LogContext.PushProperty("TraceId", activity.TraceId.ToString()))
        using (LogContext.PushProperty("SpanId", activity.SpanId.ToString()))
        {
            await next(context);
        }
    }
    else
    {
        await next(context);
    }
});
```

## Testing with Jaeger

### Docker Compose

```yaml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```bash
docker-compose up -d
# Access Jaeger UI at http://localhost:16686
```

## OpenTelemetry Best Practices Checklist

- [ ] Service name and version configured in resource
- [ ] Automatic instrumentation enabled for ASP.NET Core, HttpClient, EF Core
- [ ] Custom ActivitySource created for business logic tracing
- [ ] Sensitive data not included in span tags
- [ ] Sampling configured appropriately (100% dev, <10% prod)
- [ ] Health check endpoints excluded from tracing
- [ ] Custom metrics created for business KPIs
- [ ] Trace context propagated through distributed calls
- [ ] Correlation with structured logging implemented
- [ ] OTLP exporter configured for production observability platform

## Navigation

- **Previous**: `structured-logging.md` - Logging configuration
- **Next**: `health-checks.md` - Health monitoring
- **Related**: `../08-performance/resilience.md` - Resilience patterns

## References

- [.NET Observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [OpenTelemetry .NET](https://github.com/open-telemetry/opentelemetry-dotnet)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [Jaeger](https://www.jaegertracing.io/)
- [Prometheus](https://prometheus.io/)
