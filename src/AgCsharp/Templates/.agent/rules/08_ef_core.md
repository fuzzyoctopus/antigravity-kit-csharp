# Entity Framework Core (WinForms Desktop)

## DbContext Configuration

In Desktop Applications using local databases (like SQLite or LocalDB), EF Core requires specific threading and connection lifecycle management considerations that differ heavily from the web context.

### Basic Setup
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

### Registration in Program.cs
Instead of relying purely on scoped lifetimes (which are effectively singletons if bounded to the main form), we register `IDbContextFactory` to easily generate short-lived contexts on demand, allowing thread-safe UI interactions.

```csharp
// Program.cs
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
{
    // SQLite used for local desktop DB
    options.UseSqlite(builder.Configuration.GetConnectionString("LocalDatabase"));

    if (builder.Environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});
```

## DbContext Lifetime & Service Scope
**Warning:** WinForms does not have a "per-request" DI scope by default. If you resolve a `DbContext` into a Presenter, it will live as long as the Presenter (often the whole app lifespan). This causes the `ChangeTracker` to bloat, consume memory, and prevents multiple UI components from writing concurrently.

### Best Practice: DbContextFactory for Operations
```csharp
public class CustomerRepository : ICustomerRepository
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public CustomerRepository(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task<List<Customer>> GetAllAsync()
    {
        // 1. Create a short-lived context
        using var context = await _contextFactory.CreateDbContextAsync();
        
        // 2. Disable tracking for Read-Only operations speeds up rendering DataGridViews
        return await context.Customers.AsNoTracking().ToListAsync();
    }

    public async Task AddCustomerAsync(Customer customer)
    {
        using var context = await _contextFactory.CreateDbContextAsync();
        context.Customers.Add(customer);
        await context.SaveChangesAsync();
    }
}
```

### Alternative Practice: Creating Service Scopes per Operation
If using MediatR (`ISender`), it creates its handlers inside a transient scope already if properly configured, but you must ensure you wrap long-running operations in their own Scope if invoking directly from a Form.

```csharp
// Helper in Presenter
public async Task RefreshGridAsync()
{
    // Create scope explicitly
    using var scope = _serviceProvider.CreateScope();
    var sender = scope.ServiceProvider.GetRequiredService<ISender>();
    
    // DbContext will be disposed when scope ends
    var data = await sender.Send(new GetCustomersQuery());
    _view.UpdateGrid(data);
}
```

## Entity Configuration
(Same principles as Web. Use Fluent API)

```csharp
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");
        builder.HasKey(c => c.Id);
        builder.Property(c => c.Name).IsRequired().HasMaxLength(100);
        builder.HasIndex(c => c.Email).IsUnique();
    }
}
```

## Query Optimization for UI Responsiveness

### Use Projections
Do not pull whole entities into a `BindingList` if you only need 3 columns in the DataGridView.
```csharp
// ✅ Good - Maps directly to a DTO for DataGridView
public async Task<List<CustomerDto>> GetCustomersAsync()
{
    using var context = await _contextFactory.CreateDbContextAsync();
    return await context.Customers
        .Select(c => new CustomerDto(c.Id, c.Name, c.Email))
        .ToListAsync();
}
```

### Use BindingSource / BindingList for the UI
Even if the DB query is fast, returning raw lists isn't ideal for Two-Way data binding. Instead, map the result.

```csharp
// Presenter receiving a list, mapping to a BindingList for the View
var customers = await _repository.GetAllAsync();
var bindingList = new BindingList<CustomerDto>(customers);
_view.BindCustomers(bindingList);
```

### Avoid N+1 Queries
```csharp
// ✅ Eager Loading
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToListAsync();
```

## Migrations in Desktop Apps

Desktop applications usually do not run `dotnet ef database update` on deployment. You must run migrations automatically at startup the first time the app is launched on a new client machine or after an update.

```csharp
// Program.cs - Apply Migrations Automatically
var host = builder.Build();

using (var scope = host.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    // Apply pending migrations directly to local SQLite DB
    context.Database.Migrate(); 
}

Application.Run(host.Services.GetRequiredService<MainForm>());
```

### Creating Migrations locally
```bash
# Standard creation
dotnet ef migrations add InitialCreate -p src/MyApp.Infrastructure -s src/MyApp.Presentation
```
