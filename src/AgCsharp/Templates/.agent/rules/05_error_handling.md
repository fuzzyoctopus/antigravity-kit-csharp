# Error Handling (WinForms Desktop)

## Global Exception Handling

Unlike Web APIs where middleware catches exceptions, WinForms requires subscribing to specific application events to catch unhandled errors globally. **Failure to do this will cause the application to crash immediately** upon an unhandled UI exception.

### Registering Global Handlers

```csharp
// Program.cs
internal static class Program
{
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();

        // 1. Handle UI Thread Exceptions
        Application.ThreadException += Application_ThreadException;
        
        // 2. Override default WinForms exception dialog
        Application.SetUnhandledExceptionMode(UnhandledExceptionMode.CatchException);
        
        // 3. Handle non-UI Thread Exceptions (e.g., Task.Run)
        AppDomain.CurrentDomain.UnhandledException += CurrentDomain_UnhandledException;

        var host = CreateHostBuilder().Build();
        var mainForm = host.Services.GetRequiredService<MainForm>();
        
        Application.Run(mainForm);
    }

    private static void Application_ThreadException(object sender, ThreadExceptionEventArgs e)
    {
        HandleException(e.Exception);
    }

    private static void CurrentDomain_UnhandledException(object sender, UnhandledExceptionEventArgs e)
    {
        if (e.ExceptionObject is Exception ex)
        {
            HandleException(ex);
        }
    }

    private static void HandleException(Exception ex)
    {
        // Consider injecting ILogger globally or using a static logger instance
        // GenericLogger.LogError(ex, "An unhandled exception occurred.");

        if (ex is DomainException domainEx)
        {
            MessageBox.Show(domainEx.Message, "Warning", MessageBoxButtons.OK, MessageBoxIcon.Warning);
        }
        else
        {
            MessageBox.Show(
                "An unexpected error occurred. Please restart the application.\n\n" + ex.Message,
                "Critical Error",
                MessageBoxButtons.OK,
                MessageBoxIcon.Error);
        }
    }
}
```

## Custom Exceptions

To provide specific feedback to users, throw custom domain exceptions that Presenters or Global handlers can intercept and report cleanly.

```csharp
// Domain/Exceptions/DomainException.cs
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}

public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object key)
        : base($"{entityName} with key '{key}' was not found.") { }
}

public class ConflictException : DomainException
{
    public ConflictException(string message) : base(message) { }
}

public class BusinessRuleException : DomainException
{
    public BusinessRuleException(string message) : base(message) { }
}

// Usage in Application Service or Repository
public async Task<Order> GetOrderAsync(int id)
{
    var order = await _repository.GetByIdAsync(id);
    return order ?? throw new NotFoundException(nameof(Order), id);
}
```

## Result Pattern (For Application Layer)

Rather than using exceptions for control flow, Application logic and Use Cases should return a `Result` object. The Presenter inspects the `Result` to determine how to update the View.

### Result Class Implementation

```csharp
// Common/Models/Result.cs
public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string Error { get; }

    protected Result(bool isSuccess, string error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new(true, string.Empty);
    public static Result Failure(string error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, string.Empty);
    public static Result<T> Failure<T>(string error) => new(default!, false, error);
}

public class Result<T> : Result
{
    public T Value { get; }

    protected internal Result(T value, bool isSuccess, string error)
        : base(isSuccess, error)
    {
        Value = value;
    }
}
```

### Using Result Pattern in WinForms MVP

```csharp
// In Application Layer (Command Handler)
public async Task<Result<int>> Handle(CreateOrderCommand request, CancellationToken ct)
{
    var customer = await repository.GetByIdAsync(request.CustomerId, ct);
    if (customer is null)
        return Result.Failure<int>("Customer not found");

    var order = Order.Create(customer.Id, request.Items);
    await repository.AddAsync(order, ct);
    await unitOfWork.SaveChangesAsync(ct);

    return Result.Success(order.Id);
}

// In Presenter
public async Task OnCreateOrderClicked(OrderDto data)
{
    var result = await _sender.Send(new CreateOrderCommand(data));
    
    if (result.IsSuccess)
    {
        _view.ShowSuccess("Order Created!");
        _view.ClearForm();
    }
    else
    {
        // Don't throw an alert for every rule break, let the form display the string visually mapping it
        _view.ShowError(result.Error);
    }
}
```

## Logging Best Practices

Desktop applications often write to a local log file, SQLite database, or Event Logger.

```csharp
// Using Microsoft.Extensions.Logging.EventLog or Serilog File sink
using var scope = _logger.BeginScope(new Dictionary<string, object>
{
    ["OrderId"] = orderId,
    ["Operation"] = "ProcessOrder",
    ["User"] = WindowsIdentity.GetCurrent().Name
});

_logger.LogInformation("Starting order processing in WinForms Client");

try
{
    // Process...
    _logger.LogInformation("Order processing completed successfully");
}
catch (Exception ex)
{
    _logger.LogError(ex, "Order processing failed for order {OrderId}", orderId);
    // Let Global Exception Handler catch UI thread bubble up or handle gracefully if awaited
    throw;
}
```
