# Clean Architecture (N-Tier for WinForms)

## Overview
Clean Architecture organizes code into concentric layers with dependencies pointing inward. The core business logic has no dependencies on external frameworks or infrastructure. For WinForms desktop applications, this allows for high testability and separation of UI concerns from business logic.

```
┌─────────────────────────────────────────────────┐
│                 Presentation                     │
│    (Forms, Presenters, Views, UserControls)      │
├─────────────────────────────────────────────────┤
│                Infrastructure                    │
│      (Database, External Services, Files)        │
├─────────────────────────────────────────────────┤
│                  Application                     │
│         (Use Cases, DTOs, Interfaces)            │
├─────────────────────────────────────────────────┤
│                    Domain                        │
│         (Entities, Value Objects, Events)        │
└─────────────────────────────────────────────────┘
```

## Project Structure

```
src/
├── MyApp.Domain/                    # Core business logic
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Events/
│   ├── Exceptions/
│   └── Interfaces/                  # Repository interfaces
│
├── MyApp.Application/               # Use cases & orchestration
│   ├── Common/
│   │   ├── Behaviors/               # MediatR pipeline behaviors
│   │   ├── Interfaces/              # Application service interfaces
│   │   └── Models/                  # DTOs, Results
│   ├── Features/
│   │   ├── Customers/
│   │   │   ├── Commands/
│   │   │   ├── Queries/
│   │   │   └── EventHandlers/
│   └── DependencyInjection.cs
│
├── MyApp.Infrastructure/            # External concerns
│   ├── Persistence/
│   │   ├── DbContext/
│   │   ├── Configurations/
│   │   ├── Repositories/
│   │   └── Migrations/
│   ├── Services/
│   │   ├── LocalStorageService.cs
│   │   └── WindowsIdentityProvider.cs
│   └── DependencyInjection.cs
│
└── MyApp.Presentation/              # Windows Forms UI Project
    ├── Views/                       # Forms and UserControls implementing IView interfaces
    │   ├── ICustomerListView.cs
    │   └── CustomerListForm.cs
    ├── Presenters/                  # MVP Presenters bridging Views to Application logic
    │   └── CustomerListPresenter.cs
    ├── Program.cs                   # Application Bootstrapper (Generic Host)
    └── DependencyInjection.cs       # DI Registration for UI components
```

## Layer Rules

### Domain Layer (Innermost)
- **No dependencies** on other layers or external packages.
- Contains: Entities, Value Objects, Domain Events, Domain Services.
- Pure business logic, no I/O operations.
- Repository interfaces defined here (not implementations).

```csharp
// Domain/Entities/Customer.cs
namespace MyApp.Domain.Entities;

public class Customer : BaseEntity
{
    public string Name { get; private set; }
    public string Email { get; private set; }
    public CustomerStatus Status { get; private set; }

    private Customer() { } // EF Core
}
```

### Application Layer
- Depends only on Domain layer.
- Contains: Use Cases (Commands/Queries), DTOs, Validators.
- Orchestrates domain objects to perform tasks.
- Defines interfaces for infrastructure services.

```csharp
// Application/Features/Customers/Queries/GetCustomersQuery.cs
namespace MyApp.Application.Features.Customers.Queries;

public record GetCustomersQuery() : IRequest<Result<List<CustomerDto>>>;

public class GetCustomersQueryHandler(ICustomerRepository repository)
    : IRequestHandler<GetCustomersQuery, Result<List<CustomerDto>>>
{
    public async Task<Result<List<CustomerDto>>> Handle(GetCustomersQuery request, CancellationToken ct)
    {
        var customers = await repository.GetAllAsync(ct);
        return Result<List<CustomerDto>>.Success(customers.Select(c => new CustomerDto(c.Id, c.Name)).ToList());
    }
}
```

### Infrastructure Layer
- Depends on Domain and Application layers.
- Implements interfaces defined in inner layers.
- Contains: Database access (EF Core/SQLite/LocalDB), External APIs, Windows APIs.

```csharp
// Infrastructure/Persistence/Repositories/CustomerRepository.cs
namespace MyApp.Infrastructure.Persistence.Repositories;

public class CustomerRepository(ApplicationDbContext context) : ICustomerRepository
{
    public async Task<List<Customer>> GetAllAsync(CancellationToken ct = default)
    {
        return await context.Customers.ToListAsync(ct);
    }
}
```

### Presentation Layer (Outermost - WinForms)
- **Model-View-Presenter (MVP)** is the preferred UI pattern to decouple UI logic from WinForms controls.
- Depends on Application layer (and Infrastructure for DI registration).
- Contains: Forms, UserControls, Presenters, View Interfaces.
- **Rule of Thumb**: Forms should contain ZERO business logic. All logic lives in Presenters or Application layers.

```csharp
// Presentation/Views/ICustomerListView.cs
public interface ICustomerListView
{
    void ShowCustomers(List<CustomerDto> customers);
    void ShowError(string message);
    event EventHandler LoadCustomers;
}

// Presentation/Views/CustomerListForm.cs
public partial class CustomerListForm : Form, ICustomerListView
{
    public event EventHandler LoadCustomers;

    public CustomerListForm()
    {
        InitializeComponent();
        btnLoad.Click += (s, e) => LoadCustomers?.Invoke(this, EventArgs.Empty);
    }

    public void ShowCustomers(List<CustomerDto> customers)
    {
        if (InvokeRequired)
        {
            Invoke(new Action(() => ShowCustomers(customers)));
            return;
        }
        dataGridView1.DataSource = customers;
    }

    public void ShowError(string message) => MessageBox.Show(message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
}

// Presentation/Presenters/CustomerListPresenter.cs
public class CustomerListPresenter
{
    private readonly ICustomerListView _view;
    private readonly ISender _sender; // MediatR

    public CustomerListPresenter(ICustomerListView view, ISender sender)
    {
        _view = view;
        _sender = sender;
        _view.LoadCustomers += OnLoadCustomers;
    }

    private async void OnLoadCustomers(object sender, EventArgs e)
    {
        var result = await _sender.Send(new GetCustomersQuery());
        if (result.IsSuccess)
            _view.ShowCustomers(result.Value);
        else
            _view.ShowError(result.Error);
    }
}
```

## Dependency Injection Setup (Generic Host)
Use `Microsoft.Extensions.Hosting` in `Program.cs` to bootstrap the application.

```csharp
// Program.cs
internal static class Program
{
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();

        var host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                services.AddApplication();
                services.AddInfrastructure(context.Configuration);
                services.AddPresentation(); // Registers Forms and Presenters
            })
            .Build();

        var mainForm = host.Services.GetRequiredService<MainForm>();
        Application.Run(mainForm);
    }
}
```

## Key Principles
1. **Responsive UI**: Never block the Main UI Thread. Await long-running background tasks. Update UI controls safely via `Control.Invoke`.
2. **Dependency Rule**: Dependencies point inward only.
3. **Abstraction**: Inner layers define interfaces, outer layers implement.
4. **Testability**: Presenters and Business logic are isolated and testable without launching a GUI (`Mock<IView>`).
5. **No Logic in Code-Behind**: Do not place business rules in `Form.cs` event handlers.
