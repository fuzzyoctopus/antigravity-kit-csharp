---
description: A step-by-step workflow for implementing new features from database to API using Clean Architecture patterns. Covers defining domain entities with business logic and domain events, creating repository interfaces and implementations, configuring Entity Framework Core with migrations, creating request/response DTOs, implementing commands and queries with MediatR handlers and FluentValidation validators, building API controllers, registering dependencies, and writing unit tests. Includes complete code examples following C# best practices and a comprehensive implementation checklist.
---

# Workflow: Feature Implementation

## Overview
Step-by-step process for implementing a new feature from database to API.

## Steps

### Step 1: Define Domain Entity

1. Create the entity in `Domain/Entities/`
2. Include business logic methods
3. Add domain events for significant changes
4. Use value objects where appropriate

```csharp
// Domain/Entities/Product.cs
public class Product : BaseEntity
{
    public string Name { get; private set; }
    public Money Price { get; private set; }
    public int StockQuantity { get; private set; }
    public ProductStatus Status { get; private set; }

    private Product() { }

    public static Product Create(string name, Money price, int initialStock)
    {
        var product = new Product
        {
            Name = name,
            Price = price,
            StockQuantity = initialStock,
            Status = ProductStatus.Active
        };
        
        product.AddDomainEvent(new ProductCreatedEvent(product.Id));
        return product;
    }

    public void UpdatePrice(Money newPrice)
    {
        if (newPrice.Amount <= 0)
            throw new DomainException("Price must be positive");
        
        var oldPrice = Price;
        Price = newPrice;
        AddDomainEvent(new ProductPriceChangedEvent(Id, oldPrice, newPrice));
    }
}
```

### Step 2: Create Repository Interface

1. Define in `Domain/Repositories/`
2. Include only domain-specific queries

```csharp
// Domain/Repositories/IProductRepository.cs
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<Product>> GetActiveProductsAsync(CancellationToken ct = default);
    Task<bool> NameExistsAsync(string name, CancellationToken ct = default);
    Task AddAsync(Product product, CancellationToken ct = default);
    void Update(Product product);
    void Remove(Product product);
}
```

### Step 3: Implement Repository

1. Create in `Infrastructure/Persistence/Repositories/`
2. Implement the interface

```csharp
// Infrastructure/Persistence/Repositories/ProductRepository.cs
public class ProductRepository(ApplicationDbContext context) : IProductRepository
{
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return await context.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
    }

    public async Task<IReadOnlyList<Product>> GetActiveProductsAsync(CancellationToken ct = default)
    {
        return await context.Products
            .AsNoTracking()
            .Where(p => p.Status == ProductStatus.Active)
            .OrderBy(p => p.Name)
            .ToListAsync(ct);
    }

    // ... other methods
}
```

### Step 4: Configure EF Core

1. Create configuration in `Infrastructure/Persistence/Configurations/`
2. Run migration

```csharp
// Infrastructure/Persistence/Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);
        
        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);
        
        builder.OwnsOne(p => p.Price, price =>
        {
            price.Property(m => m.Amount).HasColumnName("Price").HasPrecision(18, 2);
            price.Property(m => m.Currency).HasColumnName("Currency").HasMaxLength(3);
        });
        
        builder.HasIndex(p => p.Name).IsUnique();
    }
}
```

```bash
# Create migration
dotnet ef migrations add AddProductTable -p src/MyApp.Infrastructure -s src/MyApp.Api

# Apply migration
dotnet ef database update -p src/MyApp.Infrastructure -s src/MyApp.Api
```

### Step 5: Create DTOs

1. Request/Response DTOs in `Application/Features/{Feature}/`

```csharp
// Application/Features/Products/ProductDtos.cs
public record CreateProductRequest(
    [Required] [StringLength(200)] string Name,
    [Required] [Range(0.01, 1000000)] decimal Price,
    [Required] [Range(0, 10000)] int InitialStock);

public record UpdateProductRequest(
    [Required] [StringLength(200)] string Name);

public record ProductResponse(
    int Id,
    string Name,
    decimal Price,
    string Currency,
    int StockQuantity,
    string Status,
    DateTime CreatedAt);

public record ProductListItemResponse(
    int Id,
    string Name,
    decimal Price,
    int StockQuantity);
```

### Step 6: Create Commands/Queries with Handlers

```csharp
// Application/Features/Products/Commands/CreateProduct.cs
public record CreateProductCommand(
    string Name,
    decimal Price,
    int InitialStock) : IRequest<Result<int>>;

public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    private readonly IProductRepository _repository;
    
    public CreateProductCommandValidator(IProductRepository repository)
    {
        _repository = repository;
        
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200)
            .MustAsync(BeUniqueName)
            .WithMessage("Product name already exists");
        
        RuleFor(x => x.Price).GreaterThan(0);
        RuleFor(x => x.InitialStock).GreaterThanOrEqualTo(0);
    }
    
    private async Task<bool> BeUniqueName(string name, CancellationToken ct)
    {
        return !await _repository.NameExistsAsync(name, ct);
    }
}

public class CreateProductCommandHandler(
    IProductRepository repository,
    IUnitOfWork unitOfWork,
    ILogger<CreateProductCommandHandler> logger)
    : IRequestHandler<CreateProductCommand, Result<int>>
{
    public async Task<Result<int>> Handle(
        CreateProductCommand request,
        CancellationToken cancellationToken)
    {
        var price = Money.Create(request.Price);
        var product = Product.Create(request.Name, price, request.InitialStock);
        
        await repository.AddAsync(product, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);
        
        logger.LogInformation("Created product {ProductId}: {Name}", product.Id, product.Name);
        
        return Result.Success(product.Id);
    }
}
```

### Step 7: Create Presenter and Form

```csharp
// Presentation/Views/IProductView.cs
public interface IProductView
{
    void ShowSuccess(string message);
    void ShowError(string message);
    event EventHandler<ProductSaveEventArgs> SaveRequested;
}

// Presentation/Presenters/ProductPresenter.cs
public class ProductPresenter
{
    private readonly IProductView _view;
    private readonly ISender _sender;

    public ProductPresenter(IProductView view, ISender sender)
    {
        _view = view;
        _sender = sender;
        _view.SaveRequested += OnSaveRequested;
    }

    private async void OnSaveRequested(object sender, ProductSaveEventArgs e)
    {
        var command = new CreateProductCommand(e.Name, e.Price, e.InitialStock);
        var result = await _sender.Send(command);
        
        if (result.IsSuccess)
            _view.ShowSuccess($"Product Created (ID: {result.Value})");
        else
            _view.ShowError(result.Error);
    }
}
```

### Step 8: Register Dependencies

```csharp
// Infrastructure/DependencyInjection.cs
services.AddScoped<IProductRepository, ProductRepository>();

// Application/DependencyInjection.cs
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly));
services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);

// Presentation/DependencyInjection.cs (Added in Program.cs Host builder)
services.AddTransient<IProductView, ProductForm>();
services.AddTransient<ProductPresenter>();
```

### Step 9: Write Tests

```csharp
// Tests/Unit/ProductTests.cs
public class ProductTests
{
    [Fact]
    public void Create_ShouldCreateProduct_WithValidData()
    {
        var product = Product.Create("Test Product", Money.Create(99.99m), 10);
        
        product.Name.Should().Be("Test Product");
        product.Price.Amount.Should().Be(99.99m);
        product.StockQuantity.Should().Be(10);
        product.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<ProductCreatedEvent>();
    }
}
```

## Checklist

- [ ] Entity created with business logic
- [ ] Repository interface defined
- [ ] Repository implementation created
- [ ] EF Core configuration added
- [ ] Migration created and applied
- [ ] DTOs created (Request/Response)
- [ ] Command/Query handlers implemented
- [ ] Validators created
- [ ] Controller/Endpoints added -> **Replaced with**: IView, Form, and Presenter created
- [ ] Dependencies registered in Generic Host (UI Layer included)
- [ ] Unit tests written (Testing the Presenter and Handlers via Mock Views)
