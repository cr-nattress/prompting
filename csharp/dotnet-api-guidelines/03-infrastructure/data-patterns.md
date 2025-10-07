# Data Access Patterns

## Overview

This guide covers data access patterns for .NET 8 applications using Entity Framework Core 8, including Repository pattern, Specification pattern, Compiled Queries, and Unit of Work. Learn when to use each pattern and understand their trade-offs.

## Related Documentation
- [EF Core Setup](./ef-core-setup.md) - DbContext configuration and migrations
- [Connection Management](./connection-management.md) - Connection pooling and resiliency
- [Performance Optimization](../08-performance/database-optimization.md) - Query optimization techniques
- [Testing Strategies](../09-testing/integration-testing.md) - Testing data access layer

## Table of Contents
1. [Repository Pattern](#repository-pattern)
2. [Specification Pattern](#specification-pattern)
3. [Compiled Queries](#compiled-queries)
4. [Unit of Work Pattern](#unit-of-work-pattern)
5. [Decision Matrix](#decision-matrix)
6. [Generic Repository Anti-Pattern](#generic-repository-anti-pattern)

---

## Repository Pattern

### Direct DbContext vs Repository Pattern

#### Pros and Cons Comparison

**Direct DbContext Access:**

**Pros:**
- Simple and straightforward
- Full EF Core feature access
- Less code to maintain
- Better performance (no abstraction overhead)
- Easier to debug and understand
- Natural fit for CQRS patterns

**Cons:**
- Domain logic coupled to EF Core
- Harder to test without real database
- Data access logic scattered across application
- Difficult to enforce consistent query patterns

**Repository Pattern:**

**Pros:**
- Clean separation of concerns
- Easier to mock for unit testing
- Centralized data access logic
- Can enforce business rules at data layer
- Easier to switch ORM (theoretical benefit)
- Better for domain-driven design

**Cons:**
- Additional abstraction layer
- More code to write and maintain
- May hide EF Core capabilities
- Can lead to repository explosion
- Performance overhead from abstraction

### Direct DbContext Usage (Recommended for most scenarios)

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.API.Services;

// Simple service using DbContext directly
public class ProductService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        ApplicationDbContext context,
        ILogger<ProductService> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<List<Product>> GetAllProductsAsync(CancellationToken cancellationToken = default)
    {
        return await _context.Products
            .AsNoTracking()
            .Where(p => !p.IsDeleted)
            .OrderBy(p => p.Name)
            .ToListAsync(cancellationToken);
    }

    public async Task<Product?> GetProductByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _context.Products
            .Include(p => p.Category)
            .Include(p => p.Reviews)
            .FirstOrDefaultAsync(p => p.Id == id && !p.IsDeleted, cancellationToken);
    }

    public async Task<List<Product>> GetProductsByCategoryAsync(
        int categoryId,
        CancellationToken cancellationToken = default)
    {
        return await _context.Products
            .AsNoTracking()
            .Where(p => p.CategoryId == categoryId && !p.IsDeleted)
            .OrderBy(p => p.Name)
            .ToListAsync(cancellationToken);
    }

    public async Task<Product> CreateProductAsync(Product product, CancellationToken cancellationToken = default)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("Created product {ProductId}: {ProductName}", product.Id, product.Name);

        return product;
    }

    public async Task UpdateProductAsync(Product product, CancellationToken cancellationToken = default)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("Updated product {ProductId}", product.Id);
    }

    public async Task DeleteProductAsync(int id, CancellationToken cancellationToken = default)
    {
        var product = await _context.Products.FindAsync(new object[] { id }, cancellationToken);

        if (product != null)
        {
            // Soft delete
            product.IsDeleted = true;
            product.DeletedAt = DateTime.UtcNow;

            await _context.SaveChangesAsync(cancellationToken);

            _logger.LogInformation("Deleted product {ProductId}", id);
        }
    }
}
```

### Repository Pattern Implementation

```csharp
using Microsoft.EntityFrameworkCore;
using System.Linq.Expressions;

namespace MyApp.Infrastructure.Repositories;

// Base repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<List<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(int id, CancellationToken cancellationToken = default);
    Task<int> CountAsync(CancellationToken cancellationToken = default);
}

// Base repository implementation
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _dbSet.FindAsync(new object[] { id }, cancellationToken);
    }

    public virtual async Task<List<T>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        return await _dbSet.AsNoTracking().ToListAsync(cancellationToken);
    }

    public virtual async Task<List<T>> FindAsync(
        Expression<Func<T, bool>> predicate,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(predicate)
            .ToListAsync(cancellationToken);
    }

    public virtual async Task<T> AddAsync(T entity, CancellationToken cancellationToken = default)
    {
        await _dbSet.AddAsync(entity, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
        return entity;
    }

    public virtual async Task UpdateAsync(T entity, CancellationToken cancellationToken = default)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public virtual async Task DeleteAsync(T entity, CancellationToken cancellationToken = default)
    {
        _dbSet.Remove(entity);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public virtual async Task<bool> ExistsAsync(int id, CancellationToken cancellationToken = default)
    {
        var entity = await _dbSet.FindAsync(new object[] { id }, cancellationToken);
        return entity != null;
    }

    public virtual async Task<int> CountAsync(CancellationToken cancellationToken = default)
    {
        return await _dbSet.CountAsync(cancellationToken);
    }
}
```

### Specific Repository Implementation

```csharp
namespace MyApp.Infrastructure.Repositories;

public interface IProductRepository : IRepository<Product>
{
    Task<List<Product>> GetProductsByCategoryAsync(int categoryId, CancellationToken cancellationToken = default);
    Task<List<Product>> GetFeaturedProductsAsync(int count, CancellationToken cancellationToken = default);
    Task<List<Product>> SearchProductsAsync(string searchTerm, CancellationToken cancellationToken = default);
    Task<Product?> GetProductWithDetailsAsync(int id, CancellationToken cancellationToken = default);
    Task<bool> IsSkuUniqueAsync(string sku, int? excludeProductId = null, CancellationToken cancellationToken = default);
}

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<List<Product>> GetProductsByCategoryAsync(
        int categoryId,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.CategoryId == categoryId && !p.IsDeleted)
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync(cancellationToken);
    }

    public async Task<List<Product>> GetFeaturedProductsAsync(
        int count,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.IsFeatured && !p.IsDeleted)
            .OrderByDescending(p => p.CreatedAt)
            .Take(count)
            .ToListAsync(cancellationToken);
    }

    public async Task<List<Product>> SearchProductsAsync(
        string searchTerm,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => !p.IsDeleted &&
                (p.Name.Contains(searchTerm) ||
                 p.Description.Contains(searchTerm) ||
                 p.Sku.Contains(searchTerm)))
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync(cancellationToken);
    }

    public async Task<Product?> GetProductWithDetailsAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .Include(p => p.Category)
            .Include(p => p.Reviews)
            .Include(p => p.Images)
            .FirstOrDefaultAsync(p => p.Id == id && !p.IsDeleted, cancellationToken);
    }

    public async Task<bool> IsSkuUniqueAsync(
        string sku,
        int? excludeProductId = null,
        CancellationToken cancellationToken = default)
    {
        var query = _dbSet.Where(p => p.Sku == sku);

        if (excludeProductId.HasValue)
        {
            query = query.Where(p => p.Id != excludeProductId.Value);
        }

        return !await query.AnyAsync(cancellationToken);
    }

    // Override to include soft delete filter
    public override async Task<List<Product>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => !p.IsDeleted)
            .ToListAsync(cancellationToken);
    }
}
```

### Repository Registration

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddRepositories(this IServiceCollection services)
    {
        // Register generic repository
        services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

        // Register specific repositories
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();

        return services;
    }
}
```

---

## Specification Pattern

### Specification Interface and Base Class

```csharp
using System.Linq.Expressions;

namespace MyApp.Core.Specifications;

// Specification interface
public interface ISpecification<T>
{
    Expression<Func<T, bool>>? Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    List<string> IncludeStrings { get; }
    Expression<Func<T, object>>? OrderBy { get; }
    Expression<Func<T, object>>? OrderByDescending { get; }
    Expression<Func<T, object>>? GroupBy { get; }
    int Take { get; }
    int Skip { get; }
    bool IsPagingEnabled { get; }
}

// Base specification implementation
public abstract class BaseSpecification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>>? Criteria { get; private set; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public List<string> IncludeStrings { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }
    public Expression<Func<T, object>>? OrderByDescending { get; private set; }
    public Expression<Func<T, object>>? GroupBy { get; private set; }
    public int Take { get; private set; }
    public int Skip { get; private set; }
    public bool IsPagingEnabled { get; private set; }

    protected BaseSpecification()
    {
    }

    protected BaseSpecification(Expression<Func<T, bool>> criteria)
    {
        Criteria = criteria;
    }

    protected virtual void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }

    protected virtual void AddInclude(string includeString)
    {
        IncludeStrings.Add(includeString);
    }

    protected virtual void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }

    protected virtual void ApplyOrderByDescending(Expression<Func<T, object>> orderByDescExpression)
    {
        OrderByDescending = orderByDescExpression;
    }

    protected virtual void ApplyPaging(int skip, int take)
    {
        Skip = skip;
        Take = take;
        IsPagingEnabled = true;
    }

    protected virtual void ApplyGroupBy(Expression<Func<T, object>> groupByExpression)
    {
        GroupBy = groupByExpression;
    }
}
```

### Concrete Specifications

```csharp
namespace MyApp.Core.Specifications;

// Product specifications
public class ProductsWithCategorySpecification : BaseSpecification<Product>
{
    public ProductsWithCategorySpecification()
        : base(p => !p.IsDeleted)
    {
        AddInclude(p => p.Category);
        ApplyOrderBy(p => p.Name);
    }

    public ProductsWithCategorySpecification(int categoryId)
        : base(p => p.CategoryId == categoryId && !p.IsDeleted)
    {
        AddInclude(p => p.Category);
        ApplyOrderBy(p => p.Name);
    }
}

public class ProductByIdWithDetailsSpecification : BaseSpecification<Product>
{
    public ProductByIdWithDetailsSpecification(int productId)
        : base(p => p.Id == productId && !p.IsDeleted)
    {
        AddInclude(p => p.Category);
        AddInclude(p => p.Reviews);
        AddInclude(p => p.Images);
    }
}

public class FeaturedProductsSpecification : BaseSpecification<Product>
{
    public FeaturedProductsSpecification(int count)
        : base(p => p.IsFeatured && !p.IsDeleted)
    {
        ApplyOrderByDescending(p => p.CreatedAt);
        ApplyPaging(0, count);
    }
}

public class ProductSearchSpecification : BaseSpecification<Product>
{
    public ProductSearchSpecification(string searchTerm)
        : base(p => !p.IsDeleted &&
            (p.Name.Contains(searchTerm) ||
             p.Description.Contains(searchTerm) ||
             p.Sku.Contains(searchTerm)))
    {
        AddInclude(p => p.Category);
        ApplyOrderBy(p => p.Name);
    }
}

public class PaginatedProductsSpecification : BaseSpecification<Product>
{
    public PaginatedProductsSpecification(int pageIndex, int pageSize, int? categoryId = null)
        : base(p => !p.IsDeleted && (!categoryId.HasValue || p.CategoryId == categoryId.Value))
    {
        AddInclude(p => p.Category);
        ApplyOrderBy(p => p.Name);
        ApplyPaging(pageIndex * pageSize, pageSize);
    }
}
```

### Specification Evaluator

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Specifications;

public static class SpecificationEvaluator<T> where T : class
{
    public static IQueryable<T> GetQuery(IQueryable<T> inputQuery, ISpecification<T> specification)
    {
        var query = inputQuery;

        // Apply criteria
        if (specification.Criteria != null)
        {
            query = query.Where(specification.Criteria);
        }

        // Apply includes
        query = specification.Includes
            .Aggregate(query, (current, include) => current.Include(include));

        // Apply string-based includes
        query = specification.IncludeStrings
            .Aggregate(query, (current, include) => current.Include(include));

        // Apply ordering
        if (specification.OrderBy != null)
        {
            query = query.OrderBy(specification.OrderBy);
        }
        else if (specification.OrderByDescending != null)
        {
            query = query.OrderByDescending(specification.OrderByDescending);
        }

        // Apply grouping
        if (specification.GroupBy != null)
        {
            query = query.GroupBy(specification.GroupBy).SelectMany(x => x);
        }

        // Apply paging
        if (specification.IsPagingEnabled)
        {
            query = query.Skip(specification.Skip).Take(specification.Take);
        }

        return query;
    }
}
```

### Repository with Specification Support

```csharp
namespace MyApp.Infrastructure.Repositories;

public interface IRepositoryWithSpec<T> : IRepository<T> where T : class
{
    Task<T?> GetBySpecAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
    Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
    Task<int> CountAsync(ISpecification<T> spec, CancellationToken cancellationToken = default);
}

public class RepositoryWithSpec<T> : Repository<T>, IRepositoryWithSpec<T> where T : class
{
    public RepositoryWithSpec(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<T?> GetBySpecAsync(
        ISpecification<T> spec,
        CancellationToken cancellationToken = default)
    {
        var query = ApplySpecification(spec);
        return await query.FirstOrDefaultAsync(cancellationToken);
    }

    public async Task<List<T>> ListAsync(
        ISpecification<T> spec,
        CancellationToken cancellationToken = default)
    {
        var query = ApplySpecification(spec);
        return await query.ToListAsync(cancellationToken);
    }

    public async Task<int> CountAsync(
        ISpecification<T> spec,
        CancellationToken cancellationToken = default)
    {
        var query = ApplySpecification(spec);
        return await query.CountAsync(cancellationToken);
    }

    private IQueryable<T> ApplySpecification(ISpecification<T> spec)
    {
        return SpecificationEvaluator<T>.GetQuery(_dbSet.AsQueryable(), spec);
    }
}
```

### Using Specifications

```csharp
namespace MyApp.API.Services;

public class ProductService
{
    private readonly IRepositoryWithSpec<Product> _productRepository;

    public ProductService(IRepositoryWithSpec<Product> productRepository)
    {
        _productRepository = productRepository;
    }

    public async Task<List<Product>> GetProductsAsync(CancellationToken cancellationToken = default)
    {
        var spec = new ProductsWithCategorySpecification();
        return await _productRepository.ListAsync(spec, cancellationToken);
    }

    public async Task<List<Product>> GetProductsByCategoryAsync(
        int categoryId,
        CancellationToken cancellationToken = default)
    {
        var spec = new ProductsWithCategorySpecification(categoryId);
        return await _productRepository.ListAsync(spec, cancellationToken);
    }

    public async Task<Product?> GetProductWithDetailsAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        var spec = new ProductByIdWithDetailsSpecification(id);
        return await _productRepository.GetBySpecAsync(spec, cancellationToken);
    }

    public async Task<List<Product>> GetFeaturedProductsAsync(
        int count,
        CancellationToken cancellationToken = default)
    {
        var spec = new FeaturedProductsSpecification(count);
        return await _productRepository.ListAsync(spec, cancellationToken);
    }

    public async Task<List<Product>> SearchProductsAsync(
        string searchTerm,
        CancellationToken cancellationToken = default)
    {
        var spec = new ProductSearchSpecification(searchTerm);
        return await _productRepository.ListAsync(spec, cancellationToken);
    }

    public async Task<(List<Product> Items, int TotalCount)> GetPaginatedProductsAsync(
        int pageIndex,
        int pageSize,
        int? categoryId = null,
        CancellationToken cancellationToken = default)
    {
        var spec = new PaginatedProductsSpecification(pageIndex, pageSize, categoryId);
        var countSpec = new ProductsWithCategorySpecification(categoryId ?? 0);

        var items = await _productRepository.ListAsync(spec, cancellationToken);
        var totalCount = await _productRepository.CountAsync(countSpec, cancellationToken);

        return (items, totalCount);
    }
}
```

---

## Compiled Queries

### Basic Compiled Queries

```csharp
using Microsoft.EntityFrameworkCore;

namespace MyApp.Infrastructure.Queries;

public static class ProductQueries
{
    // Compiled query for getting product by ID
    private static readonly Func<ApplicationDbContext, int, Task<Product?>> _getProductById =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Products
                .FirstOrDefault(p => p.Id == id && !p.IsDeleted));

    // Compiled query for getting products by category
    private static readonly Func<ApplicationDbContext, int, IAsyncEnumerable<Product>> _getProductsByCategory =
        EF.CompileAsyncQuery((ApplicationDbContext context, int categoryId) =>
            context.Products
                .Where(p => p.CategoryId == categoryId && !p.IsDeleted)
                .OrderBy(p => p.Name));

    // Compiled query for product search
    private static readonly Func<ApplicationDbContext, string, IAsyncEnumerable<Product>> _searchProducts =
        EF.CompileAsyncQuery((ApplicationDbContext context, string searchTerm) =>
            context.Products
                .Where(p => !p.IsDeleted &&
                    (p.Name.Contains(searchTerm) ||
                     p.Description.Contains(searchTerm)))
                .OrderBy(p => p.Name));

    // Compiled query for count
    private static readonly Func<ApplicationDbContext, int, Task<int>> _countProductsByCategory =
        EF.CompileAsyncQuery((ApplicationDbContext context, int categoryId) =>
            context.Products.Count(p => p.CategoryId == categoryId && !p.IsDeleted));

    // Public methods to use compiled queries
    public static Task<Product?> GetProductByIdAsync(ApplicationDbContext context, int id)
    {
        return _getProductById(context, id);
    }

    public static IAsyncEnumerable<Product> GetProductsByCategoryAsync(
        ApplicationDbContext context,
        int categoryId)
    {
        return _getProductsByCategory(context, categoryId);
    }

    public static IAsyncEnumerable<Product> SearchProductsAsync(
        ApplicationDbContext context,
        string searchTerm)
    {
        return _searchProducts(context, searchTerm);
    }

    public static Task<int> CountProductsByCategoryAsync(
        ApplicationDbContext context,
        int categoryId)
    {
        return _countProductsByCategory(context, categoryId);
    }
}
```

### Using Compiled Queries in Service

```csharp
namespace MyApp.API.Services;

public class ProductQueryService
{
    private readonly ApplicationDbContext _context;

    public ProductQueryService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product?> GetProductByIdAsync(int id)
    {
        return await ProductQueries.GetProductByIdAsync(_context, id);
    }

    public async Task<List<Product>> GetProductsByCategoryAsync(int categoryId)
    {
        var products = new List<Product>();

        await foreach (var product in ProductQueries.GetProductsByCategoryAsync(_context, categoryId))
        {
            products.Add(product);
        }

        return products;
    }

    public async Task<List<Product>> SearchProductsAsync(string searchTerm)
    {
        var products = new List<Product>();

        await foreach (var product in ProductQueries.SearchProductsAsync(_context, searchTerm))
        {
            products.Add(product);
        }

        return products;
    }

    public async Task<int> CountProductsByCategoryAsync(int categoryId)
    {
        return await ProductQueries.CountProductsByCategoryAsync(_context, categoryId);
    }
}
```

### Advanced Compiled Queries with Projections

```csharp
namespace MyApp.Infrastructure.Queries;

public record ProductSummary(int Id, string Name, decimal Price, string CategoryName);

public static class AdvancedProductQueries
{
    // Compiled query with projection
    private static readonly Func<ApplicationDbContext, IAsyncEnumerable<ProductSummary>> _getProductSummaries =
        EF.CompileAsyncQuery((ApplicationDbContext context) =>
            context.Products
                .Where(p => !p.IsDeleted)
                .Select(p => new ProductSummary(
                    p.Id,
                    p.Name,
                    p.Price,
                    p.Category.Name))
                .OrderBy(p => p.Name));

    // Compiled query with parameters and projection
    private static readonly Func<ApplicationDbContext, decimal, decimal, IAsyncEnumerable<ProductSummary>>
        _getProductsInPriceRange = EF.CompileAsyncQuery(
            (ApplicationDbContext context, decimal minPrice, decimal maxPrice) =>
                context.Products
                    .Where(p => !p.IsDeleted && p.Price >= minPrice && p.Price <= maxPrice)
                    .Select(p => new ProductSummary(
                        p.Id,
                        p.Name,
                        p.Price,
                        p.Category.Name))
                    .OrderBy(p => p.Price));

    public static IAsyncEnumerable<ProductSummary> GetProductSummariesAsync(ApplicationDbContext context)
    {
        return _getProductSummaries(context);
    }

    public static IAsyncEnumerable<ProductSummary> GetProductsInPriceRangeAsync(
        ApplicationDbContext context,
        decimal minPrice,
        decimal maxPrice)
    {
        return _getProductsInPriceRange(context, minPrice, maxPrice);
    }
}
```

---

## Unit of Work Pattern

### Unit of Work Interface and Implementation

```csharp
namespace MyApp.Core.Interfaces;

public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }

    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync(CancellationToken cancellationToken = default);
    Task CommitTransactionAsync(CancellationToken cancellationToken = default);
    Task RollbackTransactionAsync(CancellationToken cancellationToken = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;

    private IProductRepository? _productRepository;
    private IOrderRepository? _orderRepository;
    private ICustomerRepository? _customerRepository;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public IProductRepository Products =>
        _productRepository ??= new ProductRepository(_context);

    public IOrderRepository Orders =>
        _orderRepository ??= new OrderRepository(_context);

    public ICustomerRepository Customers =>
        _customerRepository ??= new CustomerRepository(_context);

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task BeginTransactionAsync(CancellationToken cancellationToken = default)
    {
        _transaction = await _context.Database.BeginTransactionAsync(cancellationToken);
    }

    public async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            await SaveChangesAsync(cancellationToken);
            await _transaction!.CommitAsync(cancellationToken);
        }
        catch
        {
            await RollbackTransactionAsync(cancellationToken);
            throw;
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }

    public async Task RollbackTransactionAsync(CancellationToken cancellationToken = default)
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync(cancellationToken);
            _transaction.Dispose();
            _transaction = null;
        }
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}
```

### Using Unit of Work

```csharp
namespace MyApp.API.Services;

public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IUnitOfWork unitOfWork, ILogger<OrderService> logger)
    {
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(
        CreateOrderDto orderDto,
        CancellationToken cancellationToken = default)
    {
        await _unitOfWork.BeginTransactionAsync(cancellationToken);

        try
        {
            // Verify customer exists
            var customer = await _unitOfWork.Customers.GetByIdAsync(
                orderDto.CustomerId,
                cancellationToken);

            if (customer == null)
            {
                throw new InvalidOperationException("Customer not found");
            }

            // Verify products exist and have sufficient stock
            var order = new Order
            {
                CustomerId = orderDto.CustomerId,
                OrderDate = DateTime.UtcNow,
                Status = OrderStatus.Pending,
                OrderItems = new List<OrderItem>()
            };

            foreach (var item in orderDto.Items)
            {
                var product = await _unitOfWork.Products.GetByIdAsync(
                    item.ProductId,
                    cancellationToken);

                if (product == null)
                {
                    throw new InvalidOperationException($"Product {item.ProductId} not found");
                }

                if (product.StockQuantity < item.Quantity)
                {
                    throw new InvalidOperationException(
                        $"Insufficient stock for product {product.Name}");
                }

                // Reduce stock
                product.StockQuantity -= item.Quantity;
                await _unitOfWork.Products.UpdateAsync(product, cancellationToken);

                // Add order item
                order.OrderItems.Add(new OrderItem
                {
                    ProductId = product.Id,
                    Quantity = item.Quantity,
                    UnitPrice = product.Price
                });
            }

            // Create order
            await _unitOfWork.Orders.AddAsync(order, cancellationToken);

            // Commit transaction
            await _unitOfWork.CommitTransactionAsync(cancellationToken);

            _logger.LogInformation("Created order {OrderId} for customer {CustomerId}",
                order.Id, customer.Id);

            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating order");
            await _unitOfWork.RollbackTransactionAsync(cancellationToken);
            throw;
        }
    }
}
```

### Unit of Work Registration

```csharp
namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddUnitOfWork(this IServiceCollection services)
    {
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        return services;
    }
}
```

---

## Decision Matrix

### When to Use Each Pattern

| Pattern | Use When | Avoid When | Performance Impact |
|---------|----------|------------|-------------------|
| **Direct DbContext** | - Simple CRUD operations<br>- CQRS architecture<br>- Small to medium projects<br>- Team familiar with EF Core | - Complex domain logic<br>- Need strict abstraction<br>- Switching ORMs frequently | Lowest (no overhead) |
| **Repository** | - Domain-driven design<br>- Need testability<br>- Enforce data access patterns<br>- Large teams needing structure | - Simple applications<br>- Performance-critical apps<br>- CQRS read models | Low to Medium |
| **Specification** | - Complex, reusable queries<br>- Dynamic query building<br>- Rich domain model<br>- Query composition needed | - Simple queries<br>- Performance-critical paths<br>- Small applications | Medium |
| **Compiled Queries** | - Frequently executed queries<br>- Performance-critical operations<br>- High-traffic endpoints<br>- Stable query patterns | - Rarely used queries<br>- Dynamic queries<br>- Queries that change often | Best (cached compilation) |
| **Unit of Work** | - Multi-repository transactions<br>- Complex business operations<br>- Need explicit transaction control | - Single entity operations<br>- Using DbContext directly<br>- Simple CRUD | Low |

### Recommendation by Project Type

**Small API (< 10 endpoints):**
- Use Direct DbContext in services
- Skip Repository and UoW patterns
- Consider Compiled Queries for hot paths

**Medium API (10-50 endpoints):**
- Use Repository pattern for complex entities
- Use Direct DbContext for simple queries
- Add Specifications for complex filtering
- Use Compiled Queries for high-traffic endpoints

**Large Enterprise Application:**
- Full Repository pattern
- Specification pattern for complex queries
- Unit of Work for transaction management
- Compiled Queries for performance-critical paths
- Consider CQRS for read/write separation

---

## Generic Repository Anti-Pattern

### Why Generic Repository is Often an Anti-Pattern

```csharp
// ANTI-PATTERN: Generic repository that tries to do everything
public interface IGenericRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
    Task<bool> ExistsAsync(int id);
    Task<int> CountAsync();

    // And many more generic methods that don't fit all entities...
    Task<IEnumerable<T>> GetPagedAsync(int page, int pageSize);
    Task<T> GetByIdWithIncludesAsync(int id, params Expression<Func<T, object>>[] includes);
    Task<IEnumerable<T>> FindWithIncludesAsync(
        Expression<Func<T, bool>> predicate,
        params Expression<Func<T, object>>[] includes);
}
```

### Problems with Generic Repository

1. **Leaky Abstraction**: Exposes EF Core concepts (Expression, Include) but doesn't provide full EF Core power
2. **One Size Fits None**: Different entities have different access patterns
3. **Unnecessary Complexity**: Adds abstraction without real benefit
4. **Performance Issues**: Generic methods can't be optimized for specific scenarios
5. **Testing Illusion**: Doesn't actually improve testability over DbContext

### Better Alternative: Specific Repositories

```csharp
// BETTER: Specific repository with domain-relevant methods
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<Product?> GetBySkuAsync(string sku, CancellationToken cancellationToken = default);
    Task<List<Product>> GetByCategoryAsync(int categoryId, CancellationToken cancellationToken = default);
    Task<List<Product>> GetFeaturedAsync(int count, CancellationToken cancellationToken = default);
    Task<List<Product>> SearchAsync(string searchTerm, CancellationToken cancellationToken = default);
    Task<bool> IsSkuUniqueAsync(string sku, int? excludeId = null, CancellationToken cancellationToken = default);
    Task<Product> CreateAsync(Product product, CancellationToken cancellationToken = default);
    Task UpdateAsync(Product product, CancellationToken cancellationToken = default);
    Task DeleteAsync(int id, CancellationToken cancellationToken = default);
}
```

### Or Better Still: Direct DbContext with Service Layer

```csharp
// BEST for most scenarios: Direct DbContext with well-designed service
public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    // Domain-specific methods with full EF Core power
    public async Task<Product?> GetProductBySkuAsync(string sku)
    {
        return await _context.Products
            .Include(p => p.Category)
            .Include(p => p.Reviews.OrderByDescending(r => r.CreatedAt).Take(5))
            .FirstOrDefaultAsync(p => p.Sku == sku && !p.IsDeleted);
    }

    // Performance-optimized queries
    public async Task<List<ProductSummaryDto>> GetProductSummariesAsync(int categoryId)
    {
        return await _context.Products
            .AsNoTracking()
            .Where(p => p.CategoryId == categoryId && !p.IsDeleted)
            .Select(p => new ProductSummaryDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category.Name,
                AverageRating = p.Reviews.Average(r => (double?)r.Rating) ?? 0
            })
            .ToListAsync();
    }
}
```

---

## Best Practices

### 1. Pattern Selection
- Start simple with Direct DbContext
- Add Repository only when needed for domain isolation
- Use Specifications for complex, reusable queries
- Apply Compiled Queries to proven bottlenecks

### 2. Performance
- Always use `AsNoTracking()` for read-only queries
- Compile frequently executed queries
- Project to DTOs in the database query
- Avoid N+1 queries with proper includes

### 3. Testing
- DbContext is mockable in EF Core 8
- Use in-memory database for integration tests
- Use SQLite for faster test database
- Don't over-abstract for "testability"

### 4. Maintainability
- Keep data access logic close to business logic
- Use meaningful method names
- Document complex queries
- Prefer explicit over generic

---

## References

- [EF Core Performance](https://docs.microsoft.com/en-us/ef/core/performance/)
- [Compiled Queries](https://docs.microsoft.com/en-us/ef/core/performance/advanced-performance-topics#compiled-queries)
- [Repository Pattern Debate](https://www.reddit.com/r/dotnet/comments/pzc0qr/ef_core_repository_pattern_is_it_worth_it/)
- [Specification Pattern](https://deviq.com/design-patterns/specification-pattern)
- [Unit of Work Pattern](https://docs.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application)

---

**Last Updated**: 2025-10-06
**EF Core Version**: 8.0
**.NET Version**: 8.0
