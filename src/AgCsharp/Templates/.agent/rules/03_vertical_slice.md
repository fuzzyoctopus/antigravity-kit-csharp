# Vertical Slice Architecture

## Overview
Vertical Slice Architecture organizes code by feature rather than by technical layer. Each feature is self-contained with all its components (handler, validator, models) in one place.

```
Traditional (Horizontal):          Vertical Slice:
├── Forms/                         ├── Features/
│   ├── CustomerListForm.cs        │   ├── Customers/
│   └── OrderForm.cs               │   │   ├── GetCustomer.cs
├── Services/                      │   │   ├── CreateCustomer.cs
│   ├── CustomerService.cs         │   │   └── DeleteCustomer.cs
│   └── OrderService.cs            │   └── Orders/
├── Repositories/                  │       ├── CreateOrder.cs
│   ├── CustomerRepository.cs      │       └── GetOrder.cs
│   └── OrderRepository.cs         └── Common/
└── Models/                            ├── Behaviors/
    ├── CustomerDto.cs                 └── Extensions/
    └── OrderDto.cs
```

## Project Structure

```
src/
├── MyApp.Desktop/
│   ├── Features/
│   │   ├── Customers/
│   │   │   ├── GetCustomer.cs         # Query + Handler + Response
│   │   │   ├── GetCustomers.cs        # List query
│   │   │   ├── CreateCustomer.cs      # Command + Handler + Request
│   │   │   ├── CustomerListForm.cs    # Presenter + View
│   │   │   └── CustomerDetailForm.cs
│   │   ├── Orders/
│   │   │   ├── CreateOrder.cs
│   │   │   ├── GetOrder.cs
│   │   │   └── OrderForm.cs
│   │   └── Products/
│   │       └── ...
│   ├── Common/
│   │   ├── Behaviors/
│   │   │   ├── ValidationBehavior.cs
│   │   │   └── LoggingBehavior.cs
│   │   ├── Exceptions/
│   │   ├── Extensions/
│   │   └── Models/
│   ├── Configuration/
│   │   └── FeatureExtensions.cs       # DI Mapping
│   └── Program.cs
│
├── MyApp.Domain/                       # Shared domain entities
│   └── Entities/
│
└── MyApp.Infrastructure/               # Shared infrastructure
    └── Persistence/
```

## Feature File Pattern

### Query Example (Get Single Item)

```csharp
// Features/Customers/GetCustomer.cs
namespace MyApp.Features.Customers;

public static class GetCustomer
{
    // Request/Query
    public record Query(int Id) : IRequest<Result<Response>>;

    // Response DTO
    public record Response(
        int Id,
        string Name,
        string Email,
        DateTime CreatedAt);

    // Handler
    public class Handler(ApplicationDbContext db) 
        : IRequestHandler<Query, Result<Response>>
    {
        public async Task<Result<Response>> Handle(
            Query request, 
            CancellationToken cancellationToken)
        {
            var customer = await db.Customers
                .Where(c => c.Id == request.Id)
                .Select(c => new Response(
                    c.Id,
                    c.Name,
                    c.Email,
                    c.CreatedAt))
                .FirstOrDefaultAsync(cancellationToken);

            return customer is null
                ? Result<Response>.Failure("Customer not found")
                : Result<Response>.Success(customer);
        }
    }

    // Presenter Integration
    public class Presenter
    {
        private readonly IView _view;
        private readonly ISender _sender;

        public Presenter(IView view, ISender sender)
        {
            _view = view;
            _sender = sender;
            _view.LoadRequested += OnLoadRequested;
        }

        private async void OnLoadRequested(object sender, int id)
        {
            var result = await _sender.Send(new Query(id));
            if (result.IsSuccess)
                _view.ShowCustomer(result.Value);
        }
    }
}
```

### Command Example (Create Item)

```csharp
// Features/Customers/CreateCustomer.cs
namespace MyApp.Features.Customers;

public static class CreateCustomer
{
    // Request DTO
    public record Request(string Name, string Email);

    // Command
    public record Command(string Name, string Email) : IRequest<Result<int>>;

    // Validator
    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.Name)
                .NotEmpty()
                .MaximumLength(100);

            RuleFor(x => x.Email)
                .NotEmpty()
                .EmailAddress();
        }
    }

    // Handler
    public class Handler(ApplicationDbContext db) 
        : IRequestHandler<Command, Result<int>>
    {
        public async Task<Result<int>> Handle(
            Command request, 
            CancellationToken cancellationToken)
        {
            var customer = new Customer
            {
                Name = request.Name,
                Email = request.Email,
                CreatedAt = DateTime.UtcNow
            };

            db.Customers.Add(customer);
            await db.SaveChangesAsync(cancellationToken);

            return Result<int>.Success(customer.Id);
        }
    }

    // Presenter Integration
    public class Presenter
    {
        private readonly IView _view;
        private readonly ISender _sender;

        public Presenter(IView view, ISender sender)
        {
            _view = view;
            _sender = sender;
            _view.SaveRequested += OnSaveRequested;
        }

        private async void OnSaveRequested(object sender, Request request)
        {
            var command = new Command(request.Name, request.Email);
            var result = await _sender.Send(command);
            
            if (result.IsSuccess)
                _view.ShowSuccess($"Customer {result.Value} created.");
            else
                _view.ShowError(result.Error);
        }
    }
}
```

## DI Registration

```csharp
// Configuration/FeatureExtensions.cs
namespace MyApp.Configuration;

public static class FeatureExtensions
{
    public static IServiceCollection AddFeatureForms(this IServiceCollection services)
    {
        // Customers
        services.AddTransient<Features.Customers.CustomerListForm>();
        services.AddTransient<Features.Customers.CreateCustomerForm>();
        
        // Orders
        services.AddTransient<Features.Orders.OrderForm>();

        return services;
    }
}

// Program.cs
var host = Host.CreateDefaultBuilder()
    .ConfigureServices((context, services) =>
    {
        services.AddFeatureForms();
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
    })
    .Build();
```

## With MediatR Pipeline Behaviors

```csharp
// Common/Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        
        var failures = validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

## Key Principles

1. **Feature Cohesion**: All code for a feature lives together
2. **Minimal Dependencies**: Features are independent of each other
3. **Direct Database Access**: Skip repository abstraction when appropriate
4. **CQRS-lite**: Separate read (Query) and write (Command) operations
5. **Discoverability**: Easy to find all code related to a feature

## When to Use

### Use Vertical Slice When:
- Building Desktop Modules with distinct screens/features
- Team works on features independently
- CRUD operations dominate
- Rapid development is priority

### Consider Clean Architecture When:
- Complex domain logic
- Shared business rules across features
- Multiple presentation layers (Desktop + Web API)
- Long-term maintainability is critical
