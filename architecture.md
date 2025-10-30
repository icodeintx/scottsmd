# Payment Journal - Architecture Documentation

## Overview and Purpose

Payment Journal is a personal finance management application built with ASP.NET Core 9.0 Blazor Server. It replaces an Excel-based workflow for tracking monthly budgets, expenses, income sources, and bill payments. The application allows users to create budgets, manage payment accounts (bank accounts, credit cards, online services), record individual payments with multiple payees, and analyze financial metrics like debt-to-income ratio.

This is a server-side Blazor application where UI components execute on the server and UI updates are sent to the browser over SignalR. Data is persisted in an embedded LiteDB document database.

## Runtime Architecture

### How the Application Runs

Payment Journal uses the **Blazor Server** hosting model. In this model:
- Razor components execute on the server within an ASP.NET Core application
- UI interactions are sent from the browser to the server over a SignalR WebSocket connection
- The server processes events, updates component state, and calculates UI diffs
- Only the UI changes are sent back to the browser for rendering

### Application Bootstrap Flow

1. **Server Startup** (Program.cs:5-62)
   - The application configures logging (Program.cs:8-10), sets up dependency injection (Program.cs:28-37), and registers services
   - Connection string selection happens at startup based on `IsDebug` configuration flag (Program.cs:17-24)
   - Middleware pipeline is configured with HTTPS redirection, static files, and routing (Program.cs:49-58)
   - `app.MapBlazorHub()` establishes the SignalR hub endpoint (Program.cs:59)
   - `app.MapFallbackToPage("/_Host")` routes all requests to the host page (Program.cs:60)

2. **Host Page** (Pages/_Host.cshtml:1-45)
   - The `_Host.cshtml` page is a Razor Page that serves as the server-rendered HTML shell
   - It includes MudBlazor CSS, Bootstrap, and the Blazor Server JavaScript (Pages/_Host.cshtml:14-18, 38)
   - The `<component type="typeof(App)" render-mode="Server" />` tag bootstraps the Blazor app (Pages/_Host.cshtml:25)

3. **Router Configuration** (App.razor:2-13)
   - App.razor defines the Router component that handles client-side routing
   - Default layout is set to `SinglePage` (App.razor:4)
   - Not found routes display an error message within the SinglePage layout (App.razor:7-12)

4. **Layout** (Shared/SinglePage.razor:1-17)
   - SinglePage layout provides MudBlazor theme and service providers (MudThemeProvider, MudDialogProvider, MudSnackbarProvider, MudPopoverProvider)
   - Dark mode is enabled by default (Shared/SinglePage.razor:2)
   - The `@Body` directive renders the matched route component (Shared/SinglePage.razor:8)

### Request Lifecycle Diagram

```
Browser
   ↓ (HTTP GET /)
ASP.NET Core Middleware Pipeline
   ↓
_Host.cshtml (Server-rendered HTML + blazor.server.js)
   ↓ (SignalR connection established)
App.razor Router
   ↓ (Route matching)
Component (e.g., Budgets.razor)
   ↓ (OnParametersSet lifecycle)
Code-behind (e.g., Budgets.razor.cs)
   ↓ (Inject repository)
Repository (e.g., BudgetRepo)
   ↓ (Open LiteDatabase connection)
LiteDB (Data/paymentjournallitedb.db)
   ↓ (Query/Insert/Update/Delete)
Repository returns data
   ↓
Component renders UI
   ↓ (UI diff calculated)
SignalR sends updates to browser
   ↓
Browser updates DOM
```

## Dependency Injection and Configuration

### Service Registration (Program.cs:28-37)

The application registers services in Program.cs:

```csharp
// Repositories registered as Transient with connection string
builder.Services.AddTransient<PaymentRepo>(s => new PaymentRepo(connectionString));
builder.Services.AddTransient<BudgetRepo>(s => new BudgetRepo(connectionString));
builder.Services.AddTransient<CacheRepo>(s => new CacheRepo(connectionString));

// Scoped service for JS interop
builder.Services.AddScoped<IHighlightJS, HighlightJS>();

// Blazor and MudBlazor services
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();
builder.Services.AddMudServices();
```

Repositories are registered as **Transient** because each method opens and closes its own LiteDatabase connection. This means a new repository instance is created for each injection, but since operations are short-lived, this is acceptable.

### Configuration (appsettings.json:1-15)

Connection string selection is based on the `IsDebug` flag (Program.cs:17-24):

```csharp
if (builder.Configuration["IsDebug"]?.ToLower() == "true")
{
    connectionString = builder.Configuration.GetConnectionString("LiteDb_Debug") ?? "";
}
else
{
    connectionString = builder.Configuration.GetConnectionString("LiteDb_Production") ?? "";
}
```

**Important for Local Development:** The default appsettings.json uses Windows paths and sets `IsDebug=false` (appsettings.json:9-13):

```json
"IsDebug": "false",
"ConnectionStrings": {
    "LiteDb_Debug": "C:\\OneDrive\\src\\PaymentJournal\\Data\\paymentjournallitedb.db",
    "LiteDb_Production": "C:\\OneDrive\\Production\\apps\\PaymentJournal\\Data\\litedb.db"
}
```

For Linux/Mac development, you should:
1. Create an `appsettings.Development.json` file (already exists but doesn't override connection strings)
2. Set `IsDebug` to `"true"`
3. Provide a valid local path like `"./Data/paymentjournallitedb.db"` or use environment variables

The project includes database files in the Data directory (Data/paymentjournallitedb.db, Data/litedb.db) and the .csproj copies Data/paymentjournallitedb.db to output (PaymentJournal.Web.csproj:25-28).

## Project Layout and Key Namespaces

### Directory Structure

```
PaymentJournal.Web/
├── Data/                          # LiteDB database files
│   ├── paymentjournallitedb.db   # Main database file
│   └── litedb.db                 # Alternative database file
├── Models/                        # Domain models (POCOs)
│   ├── Budget.cs                 # Core budget aggregate
│   ├── PaymentItem.cs            # Payment records
│   ├── Expense.cs                # Monthly expense line items
│   ├── Income.cs                 # Income sources
│   ├── BankAccount.cs            # Bank account details
│   ├── CreditCard.cs             # Credit card details
│   ├── OnlineService.cs          # Online payment services
│   ├── Payee.cs                  # Individual payee within payment
│   ├── AppState.cs               # UI state persistence
│   └── MonthYear.cs              # Month/year selection state
├── Repositories/                  # Data access layer
│   ├── BaseRepo.cs               # Generic repository base class
│   ├── PaymentRepo.cs            # Payment data access
│   ├── BudgetRepo.cs             # Budget data access
│   └── CacheRepo.cs              # AppState data access
├── Services/                      # Application services
│   ├── IHighlightJS.cs           # JS interop interface
│   └── HighlightJS.cs            # Code highlighting service
├── Pages/                         # Razor pages and components
│   ├── _Host.cshtml              # Server-rendered host page
│   ├── Error.cshtml              # Error page
│   ├── Index.razor               # Placeholder index (route: /abc)
│   └── Components/               # Reusable Blazor components
│       ├── Budgets.razor         # Budget list view
│       ├── Budgets.razor.cs      # Budget list code-behind
│       ├── Budget_View.razor     # Budget detail/edit view
│       ├── Budget_View.razor.cs  # Budget detail code-behind
│       ├── Budget_Printable.razor # Printable budget view
│       ├── Payments.razor        # Payment list view
│       ├── InsertEditPayment.razor # Payment insert/edit form
│       ├── InsertEditPayment.razor.cs # Payment form code-behind
│       ├── MonthYearSelector.razor # Month/year picker component
│       ├── MonthYearSelector.razor.cs # Month/year picker logic
│       ├── AccountLists.razor    # Account list display
│       └── SimpleDialog.razor    # Confirmation dialog
├── Shared/                        # Shared layouts
│   ├── SinglePage.razor          # Main application layout
│   └── NoLayout.razor            # Minimal layout
├── wwwroot/                       # Static web assets
│   ├── css/                      # Stylesheets
│   ├── js/                       # JavaScript files
│   └── webfonts/                 # Font files
├── App.razor                      # Router configuration
├── Program.cs                     # Application entry point
├── _Imports.razor                # Global using directives
├── GlobalUsings.cs               # Global C# usings
├── Static.cs                     # Static utility methods
└── Dockerfile                     # Container configuration
```

### Models (Domain Layer)

Models are pure C# classes (POCOs) representing domain entities. They contain no business logic beyond computed properties and simple helper methods.

**Budget** (Models/Budget.cs:3-95) - The core aggregate containing:
- Collections: `Expenses`, `Incomes`, `BankAccounts`, `CreditCards`, `OnlineServices`
- Computed properties: `TotalMonthlyExpenses`, `TotalMonthlyIncomes`, `Debt_Income_Ratio`, `AccountLists`
- Helper methods: `GetAccountLists()`, `GetExpensePayGroups()`, `CalculateDTI()`

**PaymentItem** (Models/PaymentItem.cs:5-18) - Represents a payment transaction:
- Links to Budget via `BudgetId`
- Contains a list of `Payees` (one payment can have multiple payees)
- Tracks creation date and notes

**Expense** (Models/Expense.cs:3-14) - Monthly recurring expense:
- Properties: `BillName`, `PaidTo`, `PaidBy`, `Amount`, `EstimatedDueDay`
- Used within Budget to calculate totals and group by account

**Income** (Models/Income.cs:3-10) - Income source:
- Properties: `Employer`, `Type`, `Amount`
- Used within Budget to calculate income totals

**BankAccount** (Models/BankAccount.cs:3-11), **CreditCard** (Models/CreditCard.cs:3-13), **OnlineService** (Models/OnlineService.cs:3-9) - Payment account types stored within Budget

**Payee** (Models/Payee.cs:3-11) - Individual payee within a PaymentItem:
- Properties: `Name`, `Amount`, `Date`
- Computed property: `AmountFormatted` returns decimal formatted to 2 places

**AppState** (Models/AppState.cs:3-16) - Persists UI state:
- Contains `MonthYear` for the month/year selector component
- Stored in LiteDB to maintain state across sessions

### Repositories (Data Access Layer)

The repository pattern abstracts LiteDB operations. All repositories inherit from `BaseRepo<T>`.

**BaseRepo<T>** (Repositories/BaseRepo.cs:6-180) - Generic base class providing:
- `GetCollection(collectionName)` - Returns queryable collection (Repositories/BaseRepo.cs:46-63)
- `GetCollectionList(collectionName)` - Returns list of all documents (Repositories/BaseRepo.cs:65-83)
- `GetDocumentById(documentId, collectionName)` - Finds document by Guid (Repositories/BaseRepo.cs:85-103)
- `InsertDocument(document, collectionName)` - Inserts new document (Repositories/BaseRepo.cs:105-128)
- `UpdateDocument(document, collectionName)` - Updates existing document (Repositories/BaseRepo.cs:130-153)
- `DeleteDocument(documentId, collectionName)` - Deletes document (Repositories/BaseRepo.cs:17-44)
- `UpsertDocument(document, collectionName)` - Insert or update (Repositories/BaseRepo.cs:155-178)

**Important:** Each method opens a new `LiteDatabase` connection using `using` statements (Repositories/BaseRepo.cs:21, 50, 69, 89, 109, 134, 159). This means:
- Connections are short-lived and automatically disposed
- No connection pooling is used (LiteDB is embedded, so this is acceptable)
- Operations are not transactional across multiple method calls
- Safe for concurrent access as each operation gets its own database instance

**Known Issue:** `UpsertDocument` has a comment noting that "Upsert always returns false" (Repositories/BaseRepo.cs:164). Some callers like `BudgetRepo.SaveBudget` ignore this and return `Success=true` regardless (Repositories/BudgetRepo.cs:101-105).

**PaymentRepo** (Repositories/PaymentRepo.cs:6-186) - Manages PaymentItem documents:
- Collection name: `"PaymentItems"` (Repositories/PaymentRepo.cs:8)
- Key methods:
  - `GetAllItems(budgetId)` - Returns all payments for a budget (Repositories/PaymentRepo.cs:36-51)
  - `GetItemsByMonthYear(budgetId, month, year)` - Filters by month/year (Repositories/PaymentRepo.cs:106-123)
  - `GetItemsByDate(budgetId, date)` - Filters by specific date (Repositories/PaymentRepo.cs:71-87)
  - `InsertDocument(document)` - Creates new payment, generates Guid if needed (Repositories/PaymentRepo.cs:125-153)
  - `UpdateDocument(document)` - Updates existing payment (Repositories/PaymentRepo.cs:155-176)
  - `DeleteDocument(paymentId)` - Deletes payment (Repositories/PaymentRepo.cs:14-34)
  - `GetDistintYears()` - Returns unique years for year selector (Repositories/PaymentRepo.cs:53-69)
- Formats decimal amounts to 2 decimal places before saving (Repositories/PaymentRepo.cs:178-184)

**BudgetRepo** (Repositories/BudgetRepo.cs:6-117) - Manages Budget documents:
- Collection name: `"Budget"` (Repositories/BudgetRepo.cs:8)
- Key methods:
  - `GetAllBudgets()` - Returns all budgets ordered by create date (Repositories/BudgetRepo.cs:38-52)
  - `GetLatestBudget()` - Returns most recently saved budget (Repositories/BudgetRepo.cs:69-84)
  - `GetBudget(budgetId)` - Returns specific budget by ID (Repositories/BudgetRepo.cs:54-67)
  - `SaveBudget(document)` - Upserts budget, updates LastSavedDate (Repositories/BudgetRepo.cs:86-115)
  - `DeleteBudget(budgetId)` - Deletes budget (Repositories/BudgetRepo.cs:16-36)

**CacheRepo** (Repositories/CacheRepo.cs:5-55) - Manages AppState document:
- Collection name: `"AppState"` (Repositories/CacheRepo.cs:7)
- Key methods:
  - `GetAppState()` - Returns the single AppState document or creates new (Repositories/CacheRepo.cs:13-26)
  - `SaveAppState(appState)` - Upserts AppState (Repositories/CacheRepo.cs:43-53)
  - `ResetAppState()` - Resets MonthYear to default (Repositories/CacheRepo.cs:28-41)

### Services

**IHighlightJS / HighlightJS** (Services/IHighlightJS.cs:5-9, Services/HighlightJS.cs:5-32) - JavaScript interop service for syntax highlighting:
- Lazy-loads the highlight.min.js module (Services/HighlightJS.cs:11-13)
- Provides `HighlightElement(element)` method to highlight code blocks (Services/HighlightJS.cs:26-30)
- Registered as scoped service (Program.cs:32)

### Pages and Components (UI Layer)

Blazor components use the code-behind pattern: `.razor` files contain markup and `.razor.cs` files contain logic.

**Route Map:**

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | Budgets.razor | Budget list (default landing page) |
| `/budgets` | Budgets.razor | Budget list (alternate route) |
| `/budget/view` | Budget_View.razor | Create new budget or view latest |
| `/budget/view/{BudgetId}` | Budget_View.razor | View/edit specific budget |
| `/budget/print/{BudgetId}` | Budget_Printable.razor | Printable budget view |
| `/payments/{BudgetId}` | Payments.razor | Payment list for budget |
| `/payment/insert/{BudgetId}` | InsertEditPayment.razor | Create new payment |
| `/payment/edit/{PaymentItemId}` | InsertEditPayment.razor | Edit existing payment |
| `/abc` | Index.razor | Placeholder page (not used) |

**Note:** Index.razor is routed to `/abc` (Pages/Index.razor:1), so the actual landing page is Budgets at `/`.

**Budgets Component** (Pages/Components/Budgets.razor:1-52, Budgets.razor.cs:9-91):
- Lists all budgets in a table with key metrics (DTI, expenses, income)
- Loads budgets via `BudgetRepo.GetAllBudgets()` (Budgets.razor.cs:61)
- Actions: View budget, Delete budget, Create new budget
- Delete uses SimpleDialog for confirmation (Budgets.razor.cs:28-49)

**Budget_View Component** (Pages/Components/Budget_View.razor:1-299, Budget_View.razor.cs:10-325):
- Displays and edits a complete budget with all sections
- Sections: Financial summary, Expenses by account, Financial stats, Monthly expenses, Monthly income, Bank accounts, Credit cards, Online services
- Edit mode toggle enables/disables form fields (Budget_View.razor.cs:19-26)
- Parameter handling (Budget_View.razor.cs:260-313):
  - No BudgetId: loads latest budget or creates new
  - BudgetId = Guid.Empty: creates new budget with edit mode enabled
  - Valid BudgetId: loads specific budget
- Save via `BudgetRepo.SaveBudget()` (Budget_View.razor.cs:63-67)
- Navigation to Payments, Budgets list, and Print view

**Payments Component** (Pages/Components/Payments.razor:1-102):
- Lists all payments for a budget filtered by month/year
- Uses MonthYearSelector component for date filtering (Payments.razor:8)
- Displays payment note and nested table of payees with amounts
- Actions: Insert payment, Edit payment, Delete payment
- Shows AccountLists component at bottom for reference (Payments.razor:101)

**InsertEditPayment Component** (Pages/Components/InsertEditPayment.razor:1-73, InsertEditPayment.razor.cs:9-141):
- Dual-purpose form for creating and editing payments
- Routes: `/payment/insert/{BudgetId}` for new, `/payment/edit/{PaymentItemId}` for edit
- Parameter handling (InsertEditPayment.razor.cs:112-139):
  - If PaymentItemId provided: loads existing payment via `PaymentRepo.GetItemsById()`
  - If BudgetId provided: creates new PaymentItem linked to budget
- Form fields: Note, Date, and dynamic list of Payees
- Each Payee has: Name, Amount, Date
- Save logic (InsertEditPayment.razor.cs:44-64):
  - If PaymentItemId is empty: calls `PaymentRepo.InsertDocument()`
  - Otherwise: calls `PaymentRepo.UpdateDocument()`
- Shows AccountLists component for reference (InsertEditPayment.razor:71)

**MonthYearSelector Component** (Pages/Components/MonthYearSelector.razor:1-30, MonthYearSelector.razor.cs:7-81):
- Reusable month/year picker with increment/decrement buttons
- Persists selection to AppState via `CacheRepo.SaveAppState()` (MonthYearSelector.razor.cs:28-33)
- Loads available years from payment history via `PaymentRepo.GetDistintYears()` (MonthYearSelector.razor.cs:49)
- Fires `OnSearch` event callback when selection changes (MonthYearSelector.razor.cs:32)
- "Today" button resets to current month/year (MonthYearSelector.razor.cs:35-41)
- Increment/Decrement handle year rollover (MonthYearSelector.razor.cs:53-79)

**SimpleDialog Component** - Reusable confirmation dialog used throughout the app for delete operations

### Shared Layouts

**SinglePage** (Shared/SinglePage.razor:1-17) - Main application layout:
- Provides MudBlazor theme and service providers
- Dark mode enabled by default (Shared/SinglePage.razor:2)
- No navigation bar or header (single-page app design)

**NoLayout** (Shared/NoLayout.razor:1-11) - Minimal layout without dark mode setting

## Data Model and Relationships

### Entity Relationship Overview

```
Budget (1) ──────────────────────────────────────┐
  │                                               │
  ├── Expenses (*)                                │
  ├── Incomes (*)                                 │
  ├── BankAccounts (*)                            │
  ├── CreditCards (*)                             │
  └── OnlineServices (*)                          │
                                                  │
                                                  │ BudgetId (FK)
                                                  │
PaymentItem (*) ──────────────────────────────────┘
  │
  └── Payees (*)

AppState (1)
  └── MonthYear (1)
```

### Budget Aggregate

Budget is the primary aggregate containing all budget-related data as nested collections. This design means:
- A single Budget document contains all expenses, incomes, and accounts
- Updating any expense requires loading and saving the entire Budget
- No separate tables/collections for expenses, incomes, etc.
- Budget documents can become large but remain manageable for personal finance use

**Computed Properties:**
- `TotalMonthlyExpenses` - Sum of all expense amounts (Models/Budget.cs:17)
- `TotalMonthlyIncomes` - Sum of all income amounts (Models/Budget.cs:18)
- `TotalYearlyExpenses` - Monthly expenses × 12 (Models/Budget.cs:19)
- `TotalYearlyIncomes` - Monthly incomes × 12 (Models/Budget.cs:20)
- `Debt_Income_Ratio` - (Monthly expenses / (Annual salary / 12)) (Models/Budget.cs:11, 81-93)
- `YearlyWitholdings` - Annual salary - Total yearly incomes (Models/Budget.cs:21)
- `HalfMonthlyExpenses` - Total monthly expenses / 2 (Models/Budget.cs:22)
- `AccountLists` - Flattened list of all accounts from all sources (Models/Budget.cs:6, 27-66)

**Helper Methods:**
- `GetExpensePayGroups()` - Groups expenses by PaidBy account and sums amounts (Models/Budget.cs:68-79)

### PaymentItem Aggregate

PaymentItem is a separate aggregate linked to Budget via `BudgetId`. This design allows:
- Independent querying and filtering of payments without loading budgets
- Efficient month/year filtering for payment history
- Multiple payees per payment (e.g., one check paying multiple bills)

**Payee Relationship:**
- Each PaymentItem contains a list of Payees
- Payees are not stored separately; they're embedded in PaymentItem
- Each Payee has its own date and amount, allowing split payments on different dates

### AppState

AppState is a singleton document storing UI state:
- Only one AppState document exists in the database
- Contains MonthYear selection for the payment filter
- Persists user's last selected month/year across sessions

## Persistence Layer Details

### LiteDB Overview

LiteDB is an embedded NoSQL document database:
- Single-file database (no server process required)
- ACID transactions
- BSON document format
- Supports LINQ queries
- Thread-safe for concurrent access

### Connection Management

Each repository method opens and closes its own LiteDatabase connection:

```csharp
protected List<T> GetCollectionList(string collectionName)
{
    using (Database = new LiteDatabase(DatabaseName))  // Opens connection
    {
        var col = Database.GetCollection<T>(collectionName);
        var results = col.Query().ToList();
        return results;
    }  // Automatically closes and disposes connection
}
```

This pattern means:
- No connection pooling (not needed for embedded database)
- Each operation is isolated
- No risk of connection leaks
- Safe for concurrent access (LiteDB handles locking)
- Cannot span transactions across multiple method calls

### Collections

LiteDB organizes documents into collections (similar to tables in SQL):

| Collection Name | Entity Type | Repository | Description |
|----------------|-------------|------------|-------------|
| `PaymentItems` | PaymentItem | PaymentRepo | Payment transactions |
| `Budget` | Budget | BudgetRepo | Budget documents |
| `AppState` | AppState | CacheRepo | UI state |

### Indexing and Querying

The application uses LINQ queries over LiteDB collections:

```csharp
// Example: Get payments for a specific month/year and budget
var results = base.GetCollectionList(DBCollection)
    .Where(x => x.CreateDate.Value.Month == month && 
                x.CreateDate.Value.Year == year &&
                x.BudgetId == budgetId)
    .OrderByDescending(x => x.CreateDate)
    .ToList();
```

LiteDB automatically indexes the `_id` field (Guid). Additional indexes could be added for frequently queried fields like `BudgetId` or `CreateDate`, but are not currently implemented.

### Known Issues and Considerations

1. **Upsert Behavior** (Repositories/BaseRepo.cs:164-171):
   - Comment indicates "Upsert always returns false"
   - `BudgetRepo.SaveBudget` ignores the actual result and returns `Success=true` (Repositories/BudgetRepo.cs:101-105)
   - This could mask actual save failures

2. **Nullable Date Handling**:
   - Code uses `.Value` on nullable DateTime properties (e.g., `CreateDate.Value`)
   - Assumes dates are always set; could throw NullReferenceException if not
   - Example: Payments.razor:46, PaymentRepo.cs:59, 76, 111

3. **Decimal Formatting**:
   - PaymentRepo formats decimal amounts to 2 places before saving (Repositories/PaymentRepo.cs:178-184)
   - Prevents floating-point precision issues
   - Payee.AmountFormatted provides formatted display value (Models/Payee.cs:6)

4. **Budget Document Size**:
   - Entire Budget aggregate is loaded/saved as one document
   - Could become slow with hundreds of expenses/incomes
   - Currently acceptable for personal finance use case

## UI Composition and Routing

### Component Lifecycle

Blazor components follow a lifecycle with key methods:

1. **SetParametersAsync** - Parameters are set
2. **OnInitialized / OnInitializedAsync** - Component initialization
3. **OnParametersSet / OnParametersSetAsync** - After parameters are set (called on every render)
4. **OnAfterRender / OnAfterRenderAsync** - After component renders

Most components in this app use `OnParametersSet` to load data based on route parameters.

### Navigation

Navigation uses `NavigationManager` injected into components:

```csharp
[Inject]
private NavigationManager NavigationManager { get; set; } = null!;

protected void NavigateToBudgets()
{
    NavigationManager.NavigateTo($"/budgets");
}
```

The second parameter to `NavigateTo` can force a full page reload:
```csharp
NavigationManager.NavigateTo($"/budgets", true);  // Force reload
```

### MudBlazor Components

The UI uses MudBlazor extensively for:
- **Forms**: MudTextField, MudNumericField, MudDatePicker, MudSelect, MudCheckBox
- **Layout**: MudStack, MudGrid, MudPaper, MudSimpleTable
- **Actions**: MudButton, MudIconButton, MudButtonGroup
- **Dialogs**: MudDialogProvider, DialogService.Show<SimpleDialog>()
- **Theme**: MudThemeProvider with dark mode enabled

### Dialog Pattern

Delete operations use a consistent dialog pattern:

```csharp
var parameters = new DialogParameters();
parameters.Add("ContentText", $"Delete item? This cannot be undone.");
parameters.Add("ButtonText", "Delete");
parameters.Add("Color", Color.Error);

var options = new DialogOptions() { CloseButton = true, MaxWidth = MaxWidth.Small };
var dialog = await DialogService.ShowAsync<SimpleDialog>("Delete", parameters, options);
var result = await dialog.Result;

if (result is { Canceled: true })
{
    return;  // User cancelled
}
else if (result?.Data is true)
{
    // Perform delete
}
```

### Form Handling

Components use EditForm with two-way binding:

```razor
<EditForm Model="Budget" OnValidSubmit="HandleSubmit">
    <MudTextField @bind-Value="Budget.Name" Label="Budget Name" />
    <MudButton ButtonType="ButtonType.Submit">Save</MudButton>
</EditForm>
```

Code-behind handles submission:

```csharp
public void HandleSubmit()
{
    DbResult result = Repo.SaveBudget(Budget);
    NavigationManager.NavigateTo($"/budget/view/{Budget.BudgetId}", true);
}
```

## Request Pipeline and Static Assets

### Middleware Pipeline (Program.cs:41-62)

1. **Exception Handling** - UseExceptionHandler in production (Program.cs:42-46)
2. **HSTS** - HTTP Strict Transport Security in production (Program.cs:46)
3. **HTTPS Redirection** - Redirects HTTP to HTTPS (Program.cs:49)
4. **Static Files** - Serves wwwroot with logging (Program.cs:50-56)
5. **Routing** - UseRouting enables endpoint routing (Program.cs:58)
6. **Blazor Hub** - MapBlazorHub for SignalR (Program.cs:59)
7. **Fallback** - MapFallbackToPage to _Host (Program.cs:60)

### Static Assets (wwwroot/)

- **CSS**: Bootstrap, MudBlazor, custom site.css, Font Awesome
- **JavaScript**: clipboard.min.js, blazor.server.js (injected by framework), MudBlazor.min.js
- **Fonts**: Font Awesome webfonts, Open Iconic fonts

Static files are served with a custom logger (Program.cs:52-55):
```csharp
OnPrepareResponse = ctx =>
{
    Console.WriteLine($"Serving static file: {ctx.File.Name}");
}
```

## Cross-Cutting Concerns

### Logging (Program.cs:8-10)

Console and Debug logging providers are configured:
```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();
```

Default log level is Information for most categories, Warning for Microsoft.AspNetCore (appsettings.json:2-7).

### Styling and Theming

- MudBlazor provides the component library and theming
- Dark mode is enabled by default (Shared/SinglePage.razor:2)
- Custom CSS in wwwroot/css/site.css
- CSS isolation enabled per component (PaymentJournal.Web.csproj:7)

### JavaScript Interop

HighlightJS service demonstrates JS interop pattern:
1. Lazy-load JS module (Services/HighlightJS.cs:11-13)
2. Invoke JS functions from C# (Services/HighlightJS.cs:28-30)
3. Dispose module when done (Services/HighlightJS.cs:15-24)

### Error Handling

- Production: UseExceptionHandler redirects to /Error (Program.cs:44)
- Development: Detailed error UI in browser (Pages/_Host.cshtml:31-33)
- Blazor error UI shows at bottom of page (Pages/_Host.cshtml:27-36)

## Extension Points and Adding Features

### Adding a New Entity Type

1. **Create Model** in Models/ directory
2. **Create Repository** extending BaseRepo<T>:
   ```csharp
   public class MyEntityRepo : BaseRepo<MyEntity>
   {
       private string DBCollection = "MyEntities";
       
       public MyEntityRepo(string connectionString) : base(connectionString) { }
       
       // Add custom query methods
   }
   ```
3. **Register in DI** (Program.cs):
   ```csharp
   builder.Services.AddTransient<MyEntityRepo>(s => new MyEntityRepo(connectionString));
   ```
4. **Create Component** in Pages/Components/
5. **Add Route** with @page directive

### Adding a New Page

1. Create `.razor` file with `@page "/route"` directive
2. Create `.razor.cs` code-behind if needed
3. Inject required repositories/services
4. Implement `OnParametersSet` to load data
5. Add navigation links from other pages

### Adding New Queries

Prefer adding methods to repositories rather than querying directly in components:

```csharp
// In PaymentRepo.cs
public List<PaymentItem> GetPaymentsByPayeeName(string payeeName)
{
    return base.GetCollectionList(DBCollection)
        .Where(x => x.Payees.Any(p => p.Name.Contains(payeeName)))
        .ToList();
}
```

### Extending Budget Calculations

Add computed properties to Budget model:

```csharp
public decimal AverageExpenseAmount => 
    Expenses.Count > 0 ? TotalMonthlyExpenses / Expenses.Count : 0;
```

## Known Caveats and Gotchas

### 1. Upsert Always Returns False

**Location**: Repositories/BaseRepo.cs:164-171

The `UpsertDocument` method has a comment: "TODO there is a bug here. Upsert always returns false"

**Impact**: `BudgetRepo.SaveBudget` ignores the actual result and always returns `Success=true` (Repositories/BudgetRepo.cs:101-105). This could mask actual save failures.

**Workaround**: Consider using explicit Insert/Update logic instead of Upsert, or investigate the LiteDB Upsert behavior.

### 2. Windows Paths in Configuration

**Location**: appsettings.json:9-13

Default configuration uses Windows paths and `IsDebug=false`:
```json
"IsDebug": "false",
"ConnectionStrings": {
    "LiteDb_Debug": "C:\\OneDrive\\src\\PaymentJournal\\Data\\paymentjournallitedb.db",
    "LiteDb_Production": "C:\\OneDrive\\Production\\apps\\PaymentJournal\\Data\\litedb.db"
}
```

**Impact**: Won't work on Linux/Mac without modification.

**Solution**: For local development, create `appsettings.Development.json` or set environment variables:
```json
{
  "IsDebug": "true",
  "ConnectionStrings": {
    "LiteDb_Debug": "./Data/paymentjournallitedb.db"
  }
}
```

Or use environment variable:
```bash
export ConnectionStrings__LiteDb_Debug="./Data/paymentjournallitedb.db"
export IsDebug="true"
```

### 3. Nullable DateTime .Value Usage

**Locations**: Multiple (Payments.razor:46, PaymentRepo.cs:59, 76, 111)

Code frequently uses `.Value` on nullable DateTime properties:
```csharp
x.CreateDate.Value.Month
```

**Impact**: Will throw NullReferenceException if CreateDate is null.

**Mitigation**: Ensure all PaymentItem and Payee objects have dates set. Consider adding null checks or using null-conditional operators.

### 4. Index.razor Route is /abc

**Location**: Pages/Index.razor:1

Index.razor is routed to `/abc` instead of a typical route:
```razor
@page "/abc"
```

**Impact**: The actual landing page is Budgets at `/` and `/budgets`. The Index page is not used.

**Note**: This appears to be intentional to avoid conflicts with the Budgets component.

### 5. Budget Document Size

**Impact**: Entire Budget aggregate (with all expenses, incomes, accounts) is loaded and saved as one document.

**Consideration**: Could become slow with hundreds of line items, but acceptable for personal finance use case.

### 6. No Validation Attributes

**Impact**: Models don't use data annotations for validation (e.g., [Required], [Range]).

**Consideration**: Validation is handled implicitly by UI constraints (required fields, numeric inputs). Consider adding explicit validation for robustness.

### 7. Dockerfile Uses .NET 8.0

**Location**: Dockerfile:2, 8

Dockerfile references `dotnet/aspnet:8.0` and `dotnet/sdk:8.0`, but the project targets `net9.0` (PaymentJournal.Web.csproj:3).

**Impact**: Docker build will fail or use wrong runtime version.

**Solution**: Update Dockerfile to use .NET 9.0 images:
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
```

### 8. Transient Repository Lifetime

**Location**: Program.cs:28-30

Repositories are registered as Transient, meaning a new instance is created for each injection.

**Impact**: Fine for this application since each method opens/closes its own connection. However, if you need to share state across multiple repository calls, consider Scoped lifetime instead.

## First-Run Setup

### Prerequisites

- .NET 9.0 SDK
- A code editor (Visual Studio, VS Code, Rider)

### Running Locally

1. **Clone the repository**
   ```bash
   git clone https://github.com/icodeintx/paymentjournal.git
   cd paymentjournal
   ```

2. **Configure database path** (Linux/Mac)
   
   Create or edit `appsettings.Development.json`:
   ```json
   {
     "IsDebug": "true",
     "ConnectionStrings": {
       "LiteDb_Debug": "./Data/paymentjournallitedb.db"
     }
   }
   ```

3. **Ensure database file exists**
   ```bash
   ls -la Data/paymentjournallitedb.db
   ```
   
   The repository includes a database file. If missing, LiteDB will create one automatically.

4. **Build the application**
   ```bash
   dotnet build
   ```

5. **Run the application**
   ```bash
   dotnet run
   ```

6. **Open browser**
   
   Navigate to `https://localhost:5001` or the URL shown in console output.

### Docker Deployment

**Note**: Update Dockerfile to use .NET 9.0 images first (see Gotcha #7).

```bash
docker build -t paymentjournal .
docker run -p 8080:8080 paymentjournal
```

The application will be available at `http://localhost:8080`.

## Summary

Payment Journal is a well-structured Blazor Server application following common patterns:
- **Repository pattern** for data access abstraction
- **Code-behind pattern** for component logic separation
- **Dependency injection** for loose coupling
- **Embedded database** for simplicity and portability
- **Component-based UI** with MudBlazor for rich interactions

The architecture is appropriate for a personal finance application with a single user. For multi-user scenarios, consider:
- Moving to a client-server database (SQL Server, PostgreSQL)
- Adding authentication and authorization
- Implementing user-specific data isolation
- Adding API layer for mobile apps

For questions or contributions, refer to the repository README and existing code patterns demonstrated in this document.
