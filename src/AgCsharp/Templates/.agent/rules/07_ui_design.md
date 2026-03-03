# UI Design (WinForms & MVP)

## Overview

Unlike Web APIs where the presentation layer is comprised of controllers responding to HTTP requests, a WinForms application centers around stateful Forms, UserControls, and event-driven interactions. 

We use the **Model-View-Presenter (MVP)** pattern to ensure that the code-behind of Forms (`Form.cs`) contains as little logic as possible, making the UI testable and maintainable.

## The Model-View-Presenter (MVP) Pattern

### 1. View Interface (`IView`)

Every Form or distinct UserControl should implement an `IView` interface. This interface defines the contract between the Presenter and the UI. It exposes:
- Methods to display data (`ShowCustomers`, `DisplayError`)
- Events to notify the Presenter of user actions (`SaveClicked`, `SearchRequested`)
- Properties to retrieve simple user input (`SearchTerm`)

```csharp
public interface ICustomerListView
{
    // Outbound Data (Presenter -> View)
    void LoadCustomers(BindingList<CustomerDto> customers);
    void ShowError(string message);
    void SetLoadingState(bool isLoading);

    // Inbound Actions (View -> Presenter)
    event EventHandler LoadRequested;
    event EventHandler<CustomerEventArgs> DeleteRequested;
}
```

### 2. The View Implementation (`Form.cs`)

The Form solely implements the `IView` interface and maps UI control events to the interface events.

```csharp
public partial class CustomerListForm : Form, ICustomerListView
{
    public event EventHandler LoadRequested;
    public event EventHandler<CustomerEventArgs> DeleteRequested;

    public CustomerListForm()
    {
        InitializeComponent();
        
        // Map UI events to Interface events
        btnRefresh.Click += (s, e) => LoadRequested?.Invoke(this, EventArgs.Empty);
        
        btnDelete.Click += (s, e) => 
        {
            if (dataGridView.CurrentRow?.DataBoundItem is CustomerDto selected)
                DeleteRequested?.Invoke(this, new CustomerEventArgs(selected.Id));
        };
    }

    public void LoadCustomers(BindingList<CustomerDto> customers)
    {
        // Ensure Thread-Safe UI Updates
        if (InvokeRequired)
        {
            Invoke(new Action(() => LoadCustomers(customers)));
            return;
        }
        dataGridView.DataSource = customers;
    }

    public void ShowError(string message)
    {
        MessageBox.Show(message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
    }
    
    public void SetLoadingState(bool isLoading)
    {
        if (InvokeRequired) { Invoke(new Action(() => SetLoadingState(isLoading))); return; }
        
        btnRefresh.Enabled = !isLoading;
        lblStatus.Text = isLoading ? "Loading..." : "Ready";
    }
}
```

### 3. The Presenter

The Presenter subscribes to the View's events, executes business logic (often by sending a Command/Query via MediatR to the Application Layer), and updates the View via the `IView` interface.

```csharp
public class CustomerListPresenter
{
    private readonly ICustomerListView _view;
    private readonly ISender _sender; // MediatR

    public CustomerListPresenter(ICustomerListView view, ISender sender)
    {
        _view = view;
        _sender = sender;
        
        // Subscribe to view events
        _view.LoadRequested += OnLoadRequested;
        _view.DeleteRequested += OnDeleteRequested;
    }

    private async void OnLoadRequested(object sender, EventArgs e)
    {
        _view.SetLoadingState(true);
        try
        {
            // Execute business logic asynchronously
            var result = await _sender.Send(new GetCustomersQuery());
            
            if (result.IsSuccess)
            {
                var bindingList = new BindingList<CustomerDto>(result.Value);
                _view.LoadCustomers(bindingList);
            }
            else
            {
                _view.ShowError(result.Error);
            }
        }
        finally
        {
            _view.SetLoadingState(false);
        }
    }

    private async void OnDeleteRequested(object sender, CustomerEventArgs e)
    {
        var result = await _sender.Send(new DeleteCustomerCommand(e.CustomerId));
        if (result.IsSuccess)
        {
            // Reload the grid
            OnLoadRequested(this, EventArgs.Empty);
        }
        else
        {
            _view.ShowError(result.Error);
        }
    }
}
```

## Data Binding Best Practices

WinForms databinding is powerful but can be tricky if not managed correctly.

1. **Use `BindingList<T>`**: Instead of binding directly to a `List<T>`, bind generic grids or lists to a `BindingList<T>`. This allows the UI to automatically react when items are added or removed.
2. **Implement `INotifyPropertyChanged`**: If you want single-property updates to reflect on the UI instantly (e.g., editing a text box that updates a grid row), the DTO or ViewModel needs to implement `INotifyPropertyChanged`.
3. **Use `BindingSource`**: A `BindingSource` acts as an intermediary between your `BindingList` and the controls. It handles currency (current item selection) gracefully.

```csharp
// Example ViewModel implementing INotifyPropertyChanged
public class CustomerViewModel : INotifyPropertyChanged
{
    private string _name;

    public string Name 
    { 
        get => _name; 
        set 
        { 
            if (_name != value)
            {
                _name = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
            }
        } 
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```

## Creating UserControls for Reusability

Do not build monolithic Forms holding hundreds of controls. Break down complex UIs into reusable `UserControl`s.
- Each `UserControl` should have its own `IView` and `Presenter`.
- Forms act as hosts orchestrating multiple child UserControls if necessary.

## Threading & Responsiveness

- **Always `await` I/O**: Network calls, database queries, and heavy computations must use `async/await`.
- **`InvokeRequired` / `Invoke`**: If a background task completes and needs to write to a control property (`TextBox.Text`, `Grid.DataSource`), you **must** check `InvokeRequired` and marshal the call to the UI thread using `Invoke` or `BeginInvoke`.
- **`async void` in Event Handlers**: The only place `async void` is acceptable in C# is in UI Event Handlers (like `btnSave_Click`). Still, ensure any exceptions thrown inside them are caught, as unhandled exceptions in `async void` will crash the application even if you have a global thread exception handler for non-UI tasks. Wrap the inside of the event handler with a `try/catch` or let the MVP Presenter handle the robust `try/catch` logic and simply bubble up an Error string to the `_view.ShowError()`.
