# Skill: Generate Presenter

## When to Use
User requests to create a WinForms MVP Presenter, or needs logic to back a user interface.

## Template

```csharp
using MediatR;
using System;
using System.ComponentModel;
using System.Threading.Tasks;

namespace {Namespace}.Presentation.Presenters;

public class {Entity}Presenter
{
    private readonly I{Entity}View _view;
    private readonly ISender _sender;

    public {Entity}Presenter(I{Entity}View view, ISender sender)
    {
        _view = view;
        _sender = sender;

        // Subscribe to view events here
        _view.LoadDataRequested += OnLoadDataRequested;
        _view.SaveRequested += OnSaveRequested;
    }

    private async void OnLoadDataRequested(object sender, EventArgs e)
    {
        _view.SetLoading(true);
        try
        {
            var result = await _sender.Send(new Get{Entity}Query());
            if (result.IsSuccess)
            {
                _view.BindData(new BindingList<{Entity}Dto>(result.Value));
            }
            else
            {
                _view.ShowError(result.Error);
            }
        }
        finally
        {
            _view.SetLoading(false);
        }
    }

    private async void OnSaveRequested(object sender, {Entity}SaveEventArgs e)
    {
        var command = new Save{Entity}Command(e.Data);
        var result = await _sender.Send(command);

        if (result.IsSuccess)
        {
            _view.ShowSuccess("Saved successfully.");
            OnLoadDataRequested(this, EventArgs.Empty);
        }
        else
        {
            _view.ShowError(result.Error);
        }
    }
}
```

## Guidelines

1. **Inject dependencies**: Always pass the `IView` specific to this presenter via constructor, along with `ISender` (MediatR) or required service interfaces.
2. **Handle Async Void**: Map WinForm events to `async void` methods that contain `try/finally` blocks for UI states (like Loaders) and handle errors gracefully using `_view.ShowError()`.
3. **Keep it Pure**: The Presenter should not contain references to `System.Windows.Forms`. Do not accept `TextBox` or `DataGridView` references. Data flows entirely through primitive types or DTOs described by the `IView` interface.
