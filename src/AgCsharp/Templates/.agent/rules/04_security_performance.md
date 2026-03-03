# Security & Performance Rules (WinForms Desktop)

## Security

### Input Validation

#### Always Validate User Input at the UI and Application Layers
```csharp
// UI Layer Example (WinForms ErrorProvider)
private void btnSave_Click(object sender, EventArgs e)
{
    errorProvider.Clear();
    if (string.IsNullOrWhiteSpace(txtName.Text))
    {
        errorProvider.SetError(txtName, "Name is required.");
        return;
    }
    // Proceed to Presenter/Command...
}

// Application Layer (FluentValidation)
public class CreateCustomerValidator : AbstractValidator<CreateCustomerCommand>
{
    public CreateCustomerValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).EmailAddress();
    }
}
```

### Data Protection & Secrets storage

#### Securing Local Secrets uses Windows DPAPI
Unlike Web APIs, desktop applications run on the user's machine and cannot safely embed secrets (like database passwords or API keys) in `appsettings.json` in plaintext.

```csharp
// Example using Microsoft.AspNetCore.DataProtection
using Microsoft.AspNetCore.DataProtection;

var destFolder = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "MyApp\\Keys");
var services = new ServiceCollection();
services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(destFolder))
    .SetApplicationName("MyApp");
var provider = services.BuildServiceProvider();
var protector = provider.GetDataProtector("MyApp.SecretStore");

// Encrypt
string protectedData = protector.Protect("SensitiveConnectionString");

// Decrypt
string plainData = protector.Unprotect(protectedData);
```

### Authentication & Authorization

#### Windows Authentication
Desktop apps within a corporate domain should leverage Windows Integrated Authentication.

```csharp
// Retrieve current Windows Identity
using System.Security.Principal;

WindowsIdentity currentIdentity = WindowsIdentity.GetCurrent();
WindowsPrincipal principal = new WindowsPrincipal(currentIdentity);

if (principal.IsInRole(WindowsBuiltInRole.Administrator))
{
    // Enable admin features in UI
    adminPanel.Visible = true;
}
```

---

## Performance

### Responsive UI and Threading

#### Never Block the Main UI Thread
WinForms uses a single UI thread to process the message loop. Heavy operations (DB queries, file I/O, network requests) must run asynchronously to prevent the application from freezing.

```csharp
// ✅ Proper async usage in Presenter/Form
public async Task LoadDataAsync()
{
    try 
    {
        // UI thread is freed while awaiting
        var data = await _repository.GetDataAsync();
        
        // Continuation happens on UI context automatically if awaited from a UI event
        dataGridView.DataSource = data;
    }
    catch (Exception ex)
    {
        MessageBox.Show("Failed to load data.");
    }
}

// ❌ DON'T DO THIS - Freezes the app
public void LoadData()
{
    var data = _repository.GetDataAsync().Result; // Deadlock potential and freezes UI
    dataGridView.DataSource = data;
}
```

#### Safe Cross-Thread UI Updates
If a background thread (e.g., `Task.Run()`, or `System.Timers.Timer`) needs to update UI controls, you must marshall the call back to the UI thread using `Invoke` or `BeginInvoke`.

```csharp
public void UpdateProgress(int percent)
{
    if (progressBar.InvokeRequired)
    {
        progressBar.BeginInvoke(new Action(() => UpdateProgress(percent)));
        return;
    }
    progressBar.Value = percent;
}
```

### Database Performance (Local SQLite / EF Core)

#### Connection Management
For desktop caching or local DBs (SQLite/LocalDB), avoid keeping `DbContext` open indefinitely. Instantiate it per unit of work (Scope).

```csharp
// In a bounded operation
using (var scope = _serviceProvider.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    // Query local DB
    var uiData = await context.Customers.AsNoTracking().ToListAsync();
    return uiData;
}
```

#### Avoid N+1 Queries
```csharp
// ✅ Use eager loading to prevent multiple round trips
var orders = await _context.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

### Memory Management and GDI+ Leaks

WinForms heavily relies on GDI+ for rendering. Failing to dispose of GDI objects (Fonts, Brushes, Pens, Bitmaps) will cause a memory leak and eventually exhaust handles, crashing the app.

```csharp
// ✅ Always dispose of GDI objects created manually
private void pbImage_Paint(object sender, PaintEventArgs e)
{
    using (Pen myPen = new Pen(Color.Red, 2))
    using (SolidBrush myBrush = new SolidBrush(Color.Blue))
    {
        e.Graphics.DrawRectangle(myPen, 10, 10, 50, 50);
        e.Graphics.FillEllipse(myBrush, 10, 10, 50, 50);
    } // Disposed properly
}
```
