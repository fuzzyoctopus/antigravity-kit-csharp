# Skill: Generate Form

## When to Use
User requests to create a new WinForms UI screen, window, or UserControl utilizing the MVP pattern.

## Output Structure
Generating a fully functional Form requires an Interface and the Form code-behind.

## Template

### 1. The Interface (`I{Entity}View.cs`)
```csharp
using System;
using System.ComponentModel;

namespace {Namespace}.Presentation.Views;

public interface I{Entity}View
{
    // Outbound from Presenter
    void SetLoading(bool isLoading);
    void BindData(BindingList<{Entity}Dto> items);
    void ShowError(string message);
    void ShowSuccess(string message);

    // Inbound to Presenter
    event EventHandler LoadDataRequested;
    event EventHandler<{Entity}SaveEventArgs> SaveRequested;
}

public class {Entity}SaveEventArgs : EventArgs
{
    public {Entity}SaveDto Data { get; }
    public {Entity}SaveEventArgs({Entity}SaveDto data) => Data = data;
}
```

### 2. The Form (`{Entity}Form.cs`)
```csharp
using System;
using System.ComponentModel;
using System.Windows.Forms;

namespace {Namespace}.Presentation.Views;

public partial class {Entity}Form : Form, I{Entity}View
{
    public event EventHandler LoadDataRequested;
    public event EventHandler<{Entity}SaveEventArgs> SaveRequested;

    public {Entity}Form()
    {
        InitializeComponent();
        
        // Map controls to events
        btnRefresh.Click += (s, e) => LoadDataRequested?.Invoke(this, EventArgs.Empty);
        btnSave.Click += (s, e) => 
        {
            var data = new {Entity}SaveDto { /* extract from controls */ };
            SaveRequested?.Invoke(this, new {Entity}SaveEventArgs(data));
        };
    }

    public void SetLoading(bool isLoading)
    {
        if (InvokeRequired) { Invoke(new Action(() => SetLoading(isLoading))); return; }
        
        btnRefresh.Enabled = !isLoading;
        btnSave.Enabled = !isLoading;
    }

    public void BindData(BindingList<{Entity}Dto> items)
    {
        if (InvokeRequired) { Invoke(new Action(() => BindData(items))); return; }
        
        dataGridView1.DataSource = items;
    }

    public void ShowError(string message)
    {
        if (InvokeRequired) { Invoke(new Action(() => ShowError(message))); return; }
        MessageBox.Show(message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
    }
    
    public void ShowSuccess(string message)
    {
        if (InvokeRequired) { Invoke(new Action(() => ShowSuccess(message))); return; }
        MessageBox.Show(message, "Success", MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
}
```

## Guidelines

1. **No Logic in Forms**: Keep `Form.cs` strictly for routing events to the interface and dealing with UI thread manipulation.
2. **InvokeRequired**: Use `InvokeRequired` safely on any method called by the Presenter to ensure cross-thread safety.
3. **Separation**: Place Interfaces in the same directory as the Form, or a dedicated `/Interfaces/` folder under Presentation.
