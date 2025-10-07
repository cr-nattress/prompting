# Benchmarking - .NET 8

> **File Purpose**: Measure and optimize performance using BenchmarkDotNet
> **Prerequisites**: `../01-quick-start/minimal-program-setup.md` - Basic API setup
> **Related Files**: `async-patterns.md`, `caching.md`, `resilience.md`
> **Agent Use Case**: Reference when profiling hot paths, comparing implementations, and establishing performance baselines

## Quick Context

**BenchmarkDotNet** is the industry-standard library for .NET performance benchmarking. It provides accurate, statistically rigorous measurements of code performance, including CPU time, memory allocations, and garbage collection impact. Use it to validate optimizations, compare algorithms, and establish performance budgets.

## BenchmarkDotNet Setup

### Installation

```bash
# Create separate benchmark project
dotnet new classlib -n MyApi.Benchmarks
cd MyApi.Benchmarks

# Install BenchmarkDotNet
dotnet add package BenchmarkDotNet
dotnet add package BenchmarkDotNet.Diagnostics.Windows # For memory profiling
```

### Project Configuration

```xml
<!-- MyApi.Benchmarks.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <PlatformTarget>AnyCPU</PlatformTarget>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="0.13.12" />
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="0.13.12" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApi.Application\MyApi.Application.csproj" />
    <ProjectReference Include="..\MyApi.Domain\MyApi.Domain.csproj" />
  </ItemGroup>
</Project>
```

**Source**: [BenchmarkDotNet Documentation](https://benchmarkdotnet.org/articles/overview.html) (2024)

## Basic Benchmark

### Simple Benchmark Example

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(warmupCount: 3, iterationCount: 10)]
public class StringConcatenationBenchmark
{
    private const int N = 1000;

    [Benchmark(Baseline = true)]
    public string UsingStringConcat()
    {
        var result = "";
        for (int i = 0; i < N; i++)
        {
            result += i.ToString();
        }
        return result;
    }

    [Benchmark]
    public string UsingStringBuilder()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < N; i++)
        {
            sb.Append(i);
        }
        return sb.ToString();
    }

    [Benchmark]
    public string UsingStringCreate()
    {
        return string.Create(N * 4, N, (span, n) =>
        {
            for (int i = 0; i < n; i++)
            {
                i.TryFormat(span, out int written);
                span = span.Slice(written);
            }
        });
    }
}

// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        BenchmarkRunner.Run<StringConcatenationBenchmark>();
    }
}
```

### Running Benchmarks

```bash
# Release mode only
dotnet run -c Release

# Specific benchmark
dotnet run -c Release --filter *StringBuilder*

# Export results
dotnet run -c Release --exporters json html
```

## Real-World API Benchmarks

### JSON Serialization Comparison

```csharp
using System.Text.Json;
using Newtonsoft.Json;

[MemoryDiagnoser]
[RankColumn]
public class JsonSerializationBenchmark
{
    private Product _product = null!;
    private string _json = null!;

    [GlobalSetup]
    public void Setup()
    {
        _product = new Product
        {
            Id = 1,
            Name = "Test Product",
            Price = 99.99m,
            Categories = new[] { "Electronics", "Gadgets" }
        };
        _json = JsonSerializer.Serialize(_product);
    }

    [Benchmark(Baseline = true)]
    public string SystemTextJson_Serialize()
    {
        return JsonSerializer.Serialize(_product);
    }

    [Benchmark]
    public string NewtonsoftJson_Serialize()
    {
        return JsonConvert.SerializeObject(_product);
    }

    [Benchmark]
    public Product SystemTextJson_Deserialize()
    {
        return JsonSerializer.Deserialize<Product>(_json)!;
    }

    [Benchmark]
    public Product NewtonsoftJson_Deserialize()
    {
        return JsonConvert.DeserializeObject<Product>(_json)!;
    }
}
```

### Database Query Performance

```csharp
[MemoryDiagnoser]
public class DatabaseQueryBenchmark
{
    private AppDbContext _context = null!;

    [GlobalSetup]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase("BenchmarkDb")
            .Options;
        _context = new AppDbContext(options);

        // Seed data
        for (int i = 0; i < 1000; i++)
        {
            _context.Products.Add(new Product { Id = i, Name = $"Product {i}" });
        }
        _context.SaveChanges();
    }

    [GlobalCleanup]
    public void Cleanup()
    {
        _context.Dispose();
    }

    [Benchmark(Baseline = true)]
    public List<Product> ToList()
    {
        return _context.Products.ToList();
    }

    [Benchmark]
    public List<Product> AsNoTracking_ToList()
    {
        return _context.Products.AsNoTracking().ToList();
    }

    [Benchmark]
    public async Task<List<Product>> ToListAsync()
    {
        return await _context.Products.ToListAsync();
    }

    [Benchmark]
    public async Task<List<Product>> AsNoTracking_ToListAsync()
    {
        return await _context.Products.AsNoTracking().ToListAsync();
    }
}
```

### LINQ Performance

```csharp
[MemoryDiagnoser]
public class LinqBenchmark
{
    private List<int> _numbers = null!;

    [Params(100, 1000, 10000)]
    public int N;

    [GlobalSetup]
    public void Setup()
    {
        _numbers = Enumerable.Range(1, N).ToList();
    }

    [Benchmark(Baseline = true)]
    public int Sum_Linq()
    {
        return _numbers.Where(x => x % 2 == 0).Sum();
    }

    [Benchmark]
    public int Sum_For()
    {
        int sum = 0;
        for (int i = 0; i < _numbers.Count; i++)
        {
            if (_numbers[i] % 2 == 0)
                sum += _numbers[i];
        }
        return sum;
    }

    [Benchmark]
    public int Sum_Span()
    {
        int sum = 0;
        var span = CollectionsMarshal.AsSpan(_numbers);
        for (int i = 0; i < span.Length; i++)
        {
            if (span[i] % 2 == 0)
                sum += span[i];
        }
        return sum;
    }
}
```

**Source**: [BenchmarkDotNet Best Practices](https://benchmarkdotnet.org/articles/guides/good-practices.html) (2024)

## Memory Profiling

### Allocation Analysis

```csharp
[MemoryDiagnoser]
[AllocationQuantum]
public class AllocationBenchmark
{
    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        return "Hello" + " " + "World"; // 3 allocations
    }

    [Benchmark]
    public string StringInterpolation()
    {
        return $"Hello World"; // 1 allocation
    }

    [Benchmark]
    public string StringFormat()
    {
        return string.Format("{0} {1}", "Hello", "World"); // 2 allocations
    }

    [Benchmark]
    public string StringCreate()
    {
        return string.Create(11, ("Hello", "World"), (span, state) =>
        {
            state.Item1.AsSpan().CopyTo(span);
            span[5] = ' ';
            state.Item2.AsSpan().CopyTo(span.Slice(6));
        }); // 1 allocation
    }
}
```

### ArrayPool Benchmark

```csharp
[MemoryDiagnoser]
public class ArrayPoolBenchmark
{
    private const int Size = 1024;

    [Benchmark(Baseline = true)]
    public byte[] NewArray()
    {
        var array = new byte[Size];
        // Use array
        return array;
    }

    [Benchmark]
    public byte[] ArrayPoolRent()
    {
        var array = ArrayPool<byte>.Shared.Rent(Size);
        try
        {
            // Use array
            return array;
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(array);
        }
    }
}
```

## Advanced Benchmarking

### Parameterized Benchmarks

```csharp
[MemoryDiagnoser]
public class ParameterizedBenchmark
{
    [Params(10, 100, 1000)]
    public int Size;

    [Params("System.Text.Json", "Newtonsoft.Json")]
    public string Serializer = null!;

    private List<Product> _products = null!;

    [GlobalSetup]
    public void Setup()
    {
        _products = Enumerable.Range(1, Size)
            .Select(i => new Product { Id = i, Name = $"Product {i}" })
            .ToList();
    }

    [Benchmark]
    public string Serialize()
    {
        return Serializer switch
        {
            "System.Text.Json" => JsonSerializer.Serialize(_products),
            "Newtonsoft.Json" => JsonConvert.SerializeObject(_products),
            _ => throw new ArgumentException()
        };
    }
}
```

### Multiple Jobs (Multi-Runtime)

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net70)]
[SimpleJob(RuntimeMoniker.Net80)]
public class MultiRuntimeBenchmark
{
    [Benchmark]
    public int SpanVsList()
    {
        Span<int> span = stackalloc int[100];
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = i;
        }
        return span[50];
    }
}
```

## Analyzing Results

### Reading Benchmark Output

```
| Method                | Mean      | Error    | StdDev   | Median    | Ratio | Allocated |
|---------------------- |----------:|---------:|---------:|----------:|------:|----------:|
| UsingStringConcat     | 45.23 廣  | 0.89 廣  | 0.83 廣  | 45.10 廣  | 1.00  | 2000 KB   |
| UsingStringBuilder    | 12.34 廣  | 0.24 廣  | 0.22 廣  | 12.30 廣  | 0.27  | 16 KB     |
| UsingStringCreate     | 8.91 廣   | 0.17 廣  | 0.16 廣  | 8.88 廣   | 0.20  | 8 KB      |
```

**Key Metrics**:
- **Mean**: Average execution time
- **Ratio**: Relative to baseline (1.00)
- **Allocated**: Heap allocations (lower is better)
- **StdDev**: Standard deviation (lower = more consistent)

## Performance Baselines

### Establishing Budgets

```csharp
// Document expected performance
[MemoryDiagnoser]
public class PerformanceBudgetBenchmark
{
    // Budget: < 100廣, < 1KB allocated
    [Benchmark]
    public async Task<User> GetUserAsync()
    {
        // Implementation
    }

    // Budget: < 500廣, < 10KB allocated
    [Benchmark]
    public async Task<List<Product>> GetProductsAsync()
    {
        // Implementation
    }
}
```

## Checklist

- [ ] Benchmark project created in Release configuration
- [ ] Hot paths identified and benchmarked
- [ ] Baseline established for critical operations
- [ ] Memory allocations measured with [MemoryDiagnoser]
- [ ] Parameterized benchmarks for different scenarios
- [ ] Results analyzed for Mean, Ratio, and Allocated
- [ ] Performance budgets documented
- [ ] Optimizations validated with before/after benchmarks
- [ ] Benchmarks run on representative hardware
- [ ] CI/CD includes benchmark regression checks

## References

- [BenchmarkDotNet Documentation](https://benchmarkdotnet.org/) (2024)
- [BenchmarkDotNet Best Practices](https://benchmarkdotnet.org/articles/guides/good-practices.html) (2024)
- [Microsoft Docs - Performance profiling](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) (November 2023)
- [.NET Blog - Writing high-performance code](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/) (November 2023)

---

**Next Steps**: Review `async-patterns.md` for async optimization, or `caching.md` to reduce hot path execution.
