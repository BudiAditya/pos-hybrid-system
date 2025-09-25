📁 [FILE: PHASE_4_DESKTOP_APP.md]
📝 [COMMIT: "Add complete Phase 4 Desktop Application documentation"]
markdown
# Phase 4: Desktop Application (WinForms)

## Overview
Windows Desktop application menggunakan WinForms dengan Entity Framework untuk local database (PostgreSQL) dan real-time sync dengan cloud backend.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Desktop Application"
        A[WinForms UI Layer]
        B[Business Logic Layer]
        C[Data Access Layer]
        D[Local PostgreSQL]
    end
    
    subgraph "Cloud Services"
        E[REST API]
        F[SignalR Hub]
        G[Cloud PostgreSQL]
    end
    
    subgraph "Hardware Integration"
        H[Barcode Scanner]
        I[Receipt Printer]
        J[Customer Display]
        K[Cash Drawer]
    end
    
    A --> B
    B --> C
    C --> D
    B --> E
    B --> F
    B --> H
    B --> I
    B --> J
    B --> K
Project Structure
text
POSDesktop/
├── 📁 POSDesktop/                      # Main WinForms project
│   ├── 📁 Forms/                       # Windows Forms
│   ├── 📁 Controls/                    # Custom User Controls
│   ├── 📁 Services/                    # Business Logic Services
│   └── Program.cs
├── 📁 POSDesktop.Core/                 # Domain Models & Interfaces
├── 📁 POSDesktop.Data/                 # Data Access (Entity Framework)
├── 📁 POSDesktop.Hardware/             # Hardware Integration
├── 📁 POSDesktop.Sync/                 # Cloud Sync Services
└── 📁 POSDesktop.Tests/                # Unit Tests
Step 1: Project Setup dan Dependencies
1.1 Create Solution Structure
bash
# Create solution directory
mkdir POSDesktop
cd POSDesktop

# Create solution file
dotnet new sln -n POSDesktop

# Create projects
dotnet new winforms -n POSDesktop
dotnet new classlib -n POSDesktop.Core
dotnet new classlib -n POSDesktop.Data
dotnet new classlib -n POSDesktop.Hardware
dotnet new classlib -n POSDesktop.Sync
dotnet new xunit -n POSDesktop.Tests

# Add projects to solution
dotnet sln add POSDesktop/POSDesktop.csproj
dotnet sln add POSDesktop.Core/POSDesktop.Core.csproj
dotnet sln add POSDesktop.Data/POSDesktop.Data.csproj
dotnet sln add POSDesktop.Hardware/POSDesktop.Hardware.csproj
dotnet sln add POSDesktop.Sync/POSDesktop.Sync.csproj
dotnet sln add POSDesktop.Tests/POSDesktop.Tests.csproj
1.2 Configure Project Dependencies
xml
<!-- POSDesktop.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.SignalR.Client" Version="8.0.0" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="Serilog" Version="3.0.1" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\POSDesktop.Core\POSDesktop.Core.csproj" />
    <ProjectReference Include="..\POSDesktop.Data\POSDesktop.Data.csproj" />
    <ProjectReference Include="..\POSDesktop.Hardware\POSDesktop.Hardware.csproj" />
    <ProjectReference Include="..\POSDesktop.Sync\POSDesktop.Sync.csproj" />
  </ItemGroup>
</Project>
Step 2: Domain Models (Core Project)
2.1 Entity Models
csharp
// POSDesktop.Core/Entities/Product.cs
namespace POSDesktop.Core.Entities;

public class Product : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
    public string Barcode { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public decimal CostPrice { get; set; }
    public int StockQuantity { get; set; }
    public int MinimumStock { get; set; }
    public string Category { get; set; } = string.Empty;
    public string Unit { get; set; } = "pcs";
    public bool IsActive { get; set; } = true;
    public Guid StoreId { get; set; }
    
    // Sync properties
    public DateTime LastSynced { get; set; }
    public bool IsDirty { get; set; }
    public bool IsDeleted { get; set; }
    
    public ICollection<TransactionItem> TransactionItems { get; set; } = new List<TransactionItem>();
}

// POSDesktop.Core/Entities/Transaction.cs
public class Transaction : BaseEntity
{
    public string TransactionNumber { get; set; } = string.Empty;
    public Guid StoreId { get; set; }
    public TransactionStatus Status { get; set; } = TransactionStatus.Completed;
    public decimal Subtotal { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal DiscountAmount { get; set; }
    public decimal TotalAmount { get; set; }
    public decimal AmountPaid { get; set; }
    public decimal ChangeAmount { get; set; }
    public DateTime TransactionDate { get; set; } = DateTime.UtcNow;
    public string? CustomerName { get; set; }
    public string? CustomerPhone { get; set; }
    
    // Payment information
    public PaymentMethod PaymentMethod { get; set; }
    public string? PaymentReference { get; set; }
    
    // Sync properties
    public bool IsSynced { get; set; }
    public DateTime? SyncedAt { get; set; }
    
    public ICollection<TransactionItem> Items { get; set; } = new List<TransactionItem>();
}

// POSDesktop.Core/Entities/Store.cs
public class Store : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
    public string? Address { get; set; }
    public string? Phone { get; set; }
    public string? Email { get; set; }
    public bool IsActive { get; set; } = true;
    
    // Cloud sync information
    public Guid CloudStoreId { get; set; }
    public string ApiKey { get; set; } = string.Empty;
    public DateTime LastSync { get; set; }
    
    public ICollection<Product> Products { get; set; } = new List<Product>();
    public ICollection<Transaction> Transactions { get; set; } = new List<Transaction>();
}

// POSDesktop.Core/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}
2.2 Enums dan DTOs
csharp
// POSDesktop.Core/Enums/PaymentMethod.cs
public enum PaymentMethod
{
    Cash = 1,
    CreditCard = 2,
    DebitCard = 3,
    QRIS = 4,
    Gopay = 5,
    OVO = 6,
    Dana = 7,
    LinkAja = 8,
    BankTransfer = 9
}

// POSDesktop.Core/Enums/TransactionStatus.cs
public enum TransactionStatus
{
    Pending = 1,
    Completed = 2,
    Cancelled = 3,
    Refunded = 4
}

// POSDesktop.Core/DTOs/SyncDtos.cs
public class SyncRequest
{
    public Guid StoreId { get; set; }
    public DateTime LastSync { get; set; }
    public List<Product> Products { get; set; } = new();
    public List<Transaction> Transactions { get; set; } = new();
}

public class SyncResponse
{
    public bool Success { get; set; }
    public string Message { get; set; } = string.Empty;
    public DateTime ServerTime { get; set; }
    public List<Product> Products { get; set; } = new();
    public List<Transaction> Transactions { get; set; } = new();
    public List<Guid> DeletedProductIds { get; set; } = new();
}
Step 3: Data Access Layer (Entity Framework)
3.1 Database Context
csharp
// POSDesktop.Data/ApplicationDbContext.cs
namespace POSDesktop.Data;

public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Transaction> Transactions => Set<Transaction>();
    public DbSet<TransactionItem> TransactionItems => Set<TransactionItem>();
    public DbSet<Store> Stores => Set<Store>();
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseNpgsql("Host=localhost;Database=pos_desktop;Username=postgres;Password=password");
        }
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasIndex(p => new { p.StoreId, p.Code }).IsUnique();
            entity.HasIndex(p => new { p.StoreId, p.Barcode }).IsUnique();
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.Property(p => p.CostPrice).HasPrecision(18, 2);
        });
        
        modelBuilder.Entity<Transaction>(entity =>
        {
            entity.HasIndex(t => t.TransactionNumber).IsUnique();
            entity.Property(t => t.Subtotal).HasPrecision(18, 2);
            entity.Property(t => t.TaxAmount).HasPrecision(18, 2);
            entity.Property(t => t.DiscountAmount).HasPrecision(18, 2);
            entity.Property(t => t.TotalAmount).HasPrecision(18, 2);
            entity.Property(t => t.AmountPaid).HasPrecision(18, 2);
            entity.Property(t => t.ChangeAmount).HasPrecision(18, 2);
        });
    }
    
    public override int SaveChanges()
    {
        UpdateTimestamps();
        return base.SaveChanges();
    }
    
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        UpdateTimestamps();
        return await base.SaveChangesAsync(cancellationToken);
    }
    
    private void UpdateTimestamps()
    {
        var entries = ChangeTracker.Entries<BaseEntity>()
            .Where(e => e.State == EntityState.Modified || e.State == EntityState.Added);
        
        foreach (var entry in entries)
        {
            entry.Entity.UpdatedAt = DateTime.UtcNow;
            
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = DateTime.UtcNow;
            }
        }
    }
}
3.2 Repository Pattern
csharp
// POSDesktop.Core/Interfaces/IProductRepository.cs
namespace POSDesktop.Core.Interfaces;

public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<Product?> GetByCodeAsync(string code);
    Task<Product?> GetByBarcodeAsync(string barcode);
    Task<List<Product>> GetAllAsync(bool includeInactive = false);
    Task<List<Product>> GetByCategoryAsync(string category);
    Task<List<Product>> SearchAsync(string searchTerm);
    Task AddAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(Guid id);
    Task<int> GetCountAsync();
}

// POSDesktop.Data/Repositories/ProductRepository.cs
namespace POSDesktop.Data.Repositories;

public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;
    
    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(Guid id)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public async Task<Product?> GetByCodeAsync(string code)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Code == code && p.IsActive);
    }
    
    public async Task<Product?> GetByBarcodeAsync(string barcode)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Barcode == barcode && p.IsActive);
    }
    
    public async Task<List<Product>> GetAllAsync(bool includeInactive = false)
    {
        var query = _context.Products.AsQueryable();
        
        if (!includeInactive)
        {
            query = query.Where(p => p.IsActive);
        }
        
        return await query
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<List<Product>> GetByCategoryAsync(string category)
    {
        return await _context.Products
            .Where(p => p.Category == category && p.IsActive)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<List<Product>> SearchAsync(string searchTerm)
    {
        if (string.IsNullOrWhiteSpace(searchTerm))
            return await GetAllAsync();
        
        return await _context.Products
            .Where(p => p.IsActive && 
                (p.Name.Contains(searchTerm) || 
                 p.Code.Contains(searchTerm) || 
                 p.Barcode.Contains(searchTerm)))
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task UpdateAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task DeleteAsync(Guid id)
    {
        var product = await GetByIdAsync(id);
        if (product != null)
        {
            product.IsActive = false;
            await UpdateAsync(product);
        }
    }
    
    public async Task<int> GetCountAsync()
    {
        return await _context.Products.CountAsync(p => p.IsActive);
    }
}
Step 4: Main Application Forms
4.1 Main POS Form
csharp
// POSDesktop/Forms/MainForm.cs
namespace POSDesktop.Forms;

public partial class MainForm : Form
{
    private readonly IProductRepository _productRepository;
    private readonly ITransactionService _transactionService;
    private readonly ISyncService _syncService;
    private readonly IHardwareService _hardwareService;
    private readonly ISignalRService _signalRService;
    
    private BindingList<Product> _products;
    private BindingList<TransactionItem> _cartItems;
    private decimal _cartTotal;
    
    public MainForm(IProductRepository productRepository, 
        ITransactionService transactionService, ISyncService syncService,
        IHardwareService hardwareService, ISignalRService signalRService)
    {
        InitializeComponent();
        
        _productRepository = productRepository;
        _transactionService = transactionService;
        _syncService = syncService;
        _hardwareService = hardwareService;
        _signalRService = signalRService;
        
        _products = new BindingList<Product>();
        _cartItems = new BindingList<TransactionItem>();
        
        SetupDataBinding();
        InitializeHardware();
        InitializeRealTime();
    }
    
    private void SetupDataBinding()
    {
        // Products data grid
        productsDataGrid.AutoGenerateColumns = false;
        productsDataGrid.DataSource = _products;
        
        // Cart data grid
        cartDataGrid.AutoGenerateColumns = false;
        cartDataGrid.DataSource = _cartItems;
        
        // Real-time data updates
        _products.ListChanged += (s, e) => UpdateProductDisplay();
        _cartItems.ListChanged += (s, e) => CalculateTotal();
    }
    
    private async void InitializeHardware()
    {
        try
        {
            await _hardwareService.InitializeAsync();
            _hardwareService.BarcodeScanned += OnBarcodeScanned;
            _hardwareService.PaymentProcessed += OnPaymentProcessed;
        }
        catch (Exception ex)
        {
            ShowError($"Hardware initialization failed: {ex.Message}");
        }
    }
    
    private async void InitializeRealTime()
    {
        try
        {
            await _signalRService.ConnectAsync();
            await _signalRService.JoinStoreAsync(GetCurrentStoreId());
            
            _signalRService.StockUpdated += OnStockUpdated;
            _signalRService.NewTransaction += OnNewTransaction;
        }
        catch (Exception ex)
        {
            ShowError($"Real-time connection failed: {ex.Message}");
        }
    }
    
    private async void OnBarcodeScanned(object? sender, BarcodeScannedEventArgs e)
    {
        if (InvokeRequired)
        {
            Invoke(new Action<object?, BarcodeScannedEventArgs>(OnBarcodeScanned), sender, e);
            return;
        }
        
        try
        {
            var product = await _productRepository.GetByBarcodeAsync(e.Barcode);
            if (product != null)
            {
                AddToCart(product);
            }
            else
            {
                ShowError($"Product with barcode {e.Barcode} not found");
            }
        }
        catch (Exception ex)
        {
            ShowError($"Error processing barcode: {ex.Message}");
        }
    }
    
    private void AddToCart(Product product, int quantity = 1)
    {
        var existingItem = _cartItems.FirstOrDefault(i => i.ProductId == product.Id);
        
        if (existingItem != null)
        {
            existingItem.Quantity += quantity;
        }
        else
        {
            _cartItems.Add(new TransactionItem
            {
                ProductId = product.Id,
                Product = product,
                Quantity = quantity,
                UnitPrice = product.Price,
                TotalPrice = product.Price * quantity
            });
        }
        
        CalculateTotal();
    }
    
    private void CalculateTotal()
    {
        _cartTotal = _cartItems.Sum(item => item.TotalPrice);
        totalLabel.Text = _cartTotal.ToString("C2");
        
        // Update change calculation
        if (decimal.TryParse(amountPaidTextBox.Text, out decimal amountPaid))
        {
            changeLabel.Text = (amountPaid - _cartTotal).ToString("C2");
        }
    }
    
    private async void ProcessPayment()
    {
        if (_cartItems.Count == 0)
        {
            ShowError("Cart is empty");
            return;
        }
        
        if (!decimal.TryParse(amountPaidTextBox.Text, out decimal amountPaid) || amountPaid < _cartTotal)
        {
            ShowError("Invalid amount paid");
            return;
        }
        
        try
        {
            var paymentMethod = (PaymentMethod)paymentMethodComboBox.SelectedValue;
            var transaction = await _transactionService.CreateTransactionAsync(
                _cartItems.ToList(), paymentMethod, amountPaid);
            
            // Print receipt
            await _hardwareService.PrintReceiptAsync(transaction);
            
            // Clear cart
            _cartItems.Clear();
            
            ShowSuccess($"Transaction completed: {transaction.TransactionNumber}");
        }
        catch (Exception ex)
        {
            ShowError($"Transaction failed: {ex.Message}");
        }
    }
    
    private async void OnStockUpdated(object? sender, StockUpdatedEventArgs e)
    {
        if (InvokeRequired)
        {
            Invoke(new Action<object?, StockUpdatedEventArgs>(OnStockUpdated), sender, e);
            return;
        }
        
        // Update product stock in UI
        var product = _products.FirstOrDefault(p => p.Id == e.ProductId);
        if (product != null)
        {
            product.StockQuantity = e.NewStock;
            productsDataGrid.Refresh();
        }
    }
    
    private void ShowError(string message)
    {
        MessageBox.Show(message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
    }
    
    private void ShowSuccess(string message)
    {
        MessageBox.Show(message, "Success", MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
    
    // Event handlers untuk UI controls
    private void searchTextBox_TextChanged(object sender, EventArgs e)
    {
        SearchProducts(searchTextBox.Text);
    }
    
    private async void SearchProducts(string searchTerm)
    {
        try
        {
            var products = await _productRepository.SearchAsync(searchTerm);
            _products.Clear();
            foreach (var product in products)
            {
                _products.Add(product);
            }
        }
        catch (Exception ex)
        {
            ShowError($"Search failed: {ex.Message}");
        }
    }
    
    private void addToCartButton_Click(object sender, EventArgs e)
    {
        if (productsDataGrid.CurrentRow?.DataBoundItem is Product product)
        {
            AddToCart(product);
        }
    }
    
    private void processPaymentButton_Click(object sender, EventArgs e)
    {
        ProcessPayment();
    }
    
    protected override void OnFormClosing(FormClosingEventArgs e)
    {
        _hardwareService?.Dispose();
        _signalRService?.DisconnectAsync();
        base.OnFormClosing(e);
    }
}
4.2 Product Management Form
csharp
// POSDesktop/Forms/ProductManagementForm.cs
namespace POSDesktop.Forms;

public partial class ProductManagementForm : Form
{
    private readonly IProductRepository _productRepository;
    private readonly ISyncService _syncService;
    private BindingList<Product> _products;
    
    public ProductManagementForm(IProductRepository productRepository, ISyncService syncService)
    {
        InitializeComponent();
        
        _productRepository = productRepository;
        _syncService = syncService;
        _products = new BindingList<Product>();
        
        productsDataGrid.DataSource = _products;
        LoadProducts();
    }
    
    private async void LoadProducts()
    {
        try
        {
            var products = await _productRepository.GetAllAsync(true);
            _products.Clear();
            foreach (var product in products)
            {
                _products.Add(product);
            }
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Error loading products: {ex.Message}", "Error", 
                MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
    }
    
    private async void addButton_Click(object sender, EventArgs e)
    {
        using var form = new ProductEditForm();
        if (form.ShowDialog() == DialogResult.OK)
        {
            try
            {
                var product = form.Product;
                await _productRepository.AddAsync(product);
                _products.Add(product);
                
                // Mark for sync
                product.IsDirty = true;
                await _productRepository.UpdateAsync(product);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error adding product: {ex.Message}", "Error", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
    
    private async void editButton_Click(object sender, EventArgs e)
    {
        if (productsDataGrid.CurrentRow?.DataBoundItem is not Product product)
            return;
            
        using var form = new ProductEditForm(product);
        if (form.ShowDialog() == DialogResult.OK)
        {
            try
            {
                await _productRepository.UpdateAsync(product);
                productsDataGrid.Refresh();
                
                // Mark for sync
                product.IsDirty = true;
                await _productRepository.UpdateAsync(product);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error updating product: {ex.Message}", "Error", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
    
    private async void deleteButton_Click(object sender, EventArgs e)
    {
        if (productsDataGrid.CurrentRow?.DataBoundItem is not Product product)
            return;
            
        if (MessageBox.Show($"Delete product {product.Name}?", "Confirm", 
            MessageBoxButtons.YesNo, MessageBoxIcon.Question) == DialogResult.Yes)
        {
            try
            {
                await _productRepository.DeleteAsync(product.Id);
                _products.Remove(product);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error deleting product: {ex.Message}", "Error", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
    
    private async void syncButton_Click(object sender, EventArgs e)
    {
        try
        {
            syncButton.Enabled = false;
            syncButton.Text = "Syncing...";
            
            var result = await _syncService.SyncAsync();
            
            if (result.Success)
            {
                MessageBox.Show("Sync completed successfully", "Success", 
                    MessageBoxButtons.OK, MessageBoxIcon.Information);
                LoadProducts(); // Refresh data
            }
            else
            {
                MessageBox.Show($"Sync failed: {result.Message}", "Error", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
        finally
        {
            syncButton.Enabled = true;
            syncButton.Text = "Sync with Cloud";
        }
    }
}
Step 5: Hardware Integration
5.1 Hardware Service Abstraction
csharp
// POSDesktop.Hardware/IHardwareService.cs
namespace POSDesktop.Hardware;

public interface IHardwareService : IDisposable
{
    Task InitializeAsync();
    Task PrintReceiptAsync(Transaction transaction);
    Task OpenCashDrawerAsync();
    Task DisplayCustomerInfoAsync(string line1, string line2 = "");
    
    event EventHandler<BarcodeScannedEventArgs> BarcodeScanned;
    event EventHandler<PaymentProcessedEventArgs> PaymentProcessed;
}

public class BarcodeScannedEventArgs : EventArgs
{
    public string Barcode { get; set; } = string.Empty;
    public DateTime ScannedAt { get; set; } = DateTime.Now;
}

public class PaymentProcessedEventArgs : EventArgs
{
    public decimal Amount { get; set; }
    public PaymentMethod PaymentMethod { get; set; }
    public bool Success { get; set; }
    public string? Message { get; set; }
}

// POSDesktop.Hardware/HardwareService.cs
namespace POSDesktop.Hardware;

public class HardwareService : IHardwareService
{
    private readonly IReceiptPrinter _receiptPrinter;
    private readonly IBarcodeScanner _barcodeScanner;
    private readonly ICustomerDisplay _customerDisplay;
    private readonly ICashDrawer _cashDrawer;
    private readonly ILogger<HardwareService> _logger;
    
    public event EventHandler<BarcodeScannedEventArgs>? BarcodeScanned;
    public event EventHandler<PaymentProcessedEventArgs>? PaymentProcessed;
    
    public HardwareService(IReceiptPrinter receiptPrinter, IBarcodeScanner barcodeScanner,
        ICustomerDisplay customerDisplay, ICashDrawer cashDrawer, ILogger<HardwareService> logger)
    {
        _receiptPrinter = receiptPrinter;
        _barcodeScanner = barcodeScanner;
        _customerDisplay = customerDisplay;
        _cashDrawer = cashDrawer;
        _logger = logger;
    }
    
    public async Task InitializeAsync()
    {
        try
        {
            // Initialize barcode scanner
            _barcodeScanner.BarcodeScanned += (s, e) => 
                BarcodeScanned?.Invoke(this, e);
            await _barcodeScanner.InitializeAsync();
            
            // Initialize other devices
            await _receiptPrinter.InitializeAsync();
            await _customerDisplay.InitializeAsync();
            await _cashDrawer.InitializeAsync();
            
            _logger.LogInformation("All hardware devices initialized successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Hardware initialization failed");
            throw;
        }
    }
    
    public async Task PrintReceiptAsync(Transaction transaction)
    {
        try
        {
            var receipt = new Receipt
            {
                Header = "TOKO KITA",
                TransactionNumber = transaction.TransactionNumber,
                Items = transaction.Items.Select(i => new ReceiptItem
                {
                    Name = i.Product?.Name ?? "Unknown Product",
                    Quantity = i.Quantity,
                    Price = i.UnitPrice,
                    Total = i.TotalPrice
                }).ToList(),
                Subtotal = transaction.Subtotal,
                Tax = transaction.TaxAmount,
                Discount = transaction.DiscountAmount,
                Total = transaction.TotalAmount,
                AmountPaid = transaction.AmountPaid,
                Change = transaction.ChangeAmount,
                PaymentMethod = transaction.PaymentMethod.ToString(),
                DateTime = transaction.TransactionDate
            };
            
            await _receiptPrinter.PrintAsync(receipt);
            _logger.LogInformation("Receipt printed for transaction {TransactionNumber}", 
                transaction.TransactionNumber);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error printing receipt");
            throw;
        }
    }
    
    public async Task OpenCashDrawerAsync()
    {
        try
        {
            await _cashDrawer.OpenAsync();
            _logger.LogInformation("Cash drawer opened");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error opening cash drawer");
            throw;
        }
    }
    
    public async Task DisplayCustomerInfoAsync(string line1, string line2 = "")
    {
        try
        {
            await _customerDisplay.DisplayTextAsync(line1, line2);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error displaying customer info");
        }
    }
    
    public void Dispose()
    {
        _barcodeScanner?.Dispose();
        _receiptPrinter?.Dispose();
        _customerDisplay?.Dispose();
        _cashDrawer?.Dispose();
    }
}
5.2 Barcode Scanner Implementation
csharp
// POSDesktop.Hardware/Devices/BarcodeScanner.cs
namespace POSDesktop.Hardware.Devices;

public interface IBarcodeScanner : IDisposable
{
    Task InitializeAsync();
    event EventHandler<BarcodeScannedEventArgs> BarcodeScanned;
}

public class BarcodeScanner : IBarcodeScanner
{
    private readonly ILogger<BarcodeScanner> _logger;
    private SerialPort? _serialPort;
    
    public event EventHandler<BarcodeScannedEventArgs>? BarcodeScanned;
    
    public BarcodeScanner(ILogger<BarcodeScanner> logger)
    {
        _logger = logger;
    }
    
    public async Task InitializeAsync()
    {
        try
        {
            // Try to auto-detect barcode scanner
            var portName = await DetectScannerPortAsync();
            if (string.IsNullOrEmpty(portName))
            {
                _logger.LogWarning("Barcode scanner not detected");
                return;
            }
            
            _serialPort = new SerialPort(portName)
            {
                BaudRate = 9600,
                Parity = Parity.None,
                DataBits = 8,
                StopBits = StopBits.One,
                Handshake = Handshake.None
            };
            
            _serialPort.DataReceived += OnDataReceived;
            _serialPort.Open();
            
            _logger.LogInformation("Barcode scanner initialized on port {PortName}", portName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error initializing barcode scanner");
            throw;
        }
    }
    
    private void OnDataReceived(object sender, SerialDataReceivedEventArgs e)
    {
        try
        {
            var barcode = _serialPort?.ReadLine()?.Trim();
            if (!string.IsNullOrEmpty(barcode))
            {
                _logger.LogInformation("Barcode scanned: {Barcode}", barcode);
                BarcodeScanned?.Invoke(this, new BarcodeScannedEventArgs { Barcode = barcode });
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading barcode data");
        }
    }
    
    private async Task<string?> DetectScannerPortAsync()
    {
        // Implementation untuk auto-detect barcode scanner port
        // Bisa dari registry, USB device detection, atau manual configuration
        return await Task.Run(() =>
        {
            var ports = SerialPort.GetPortNames();
            return ports.FirstOrDefault(); // Simple detection - enhance as needed
        });
    }
    
    public void Dispose()
    {
        _serialPort?.Close();
        _serialPort?.Dispose();
    }
}
Step 6: Cloud Sync Services
6.1 Sync Service Implementation
csharp
// POSDesktop.Sync/ISyncService.cs
namespace POSDesktop.Sync;

public interface ISyncService
{
    Task<SyncResult> SyncAsync();
    Task<SyncResult> SyncProductsAsync();
    Task<SyncResult> SyncTransactionsAsync();
    Task<bool> CheckConnectionAsync();
}

public class SyncResult
{
    public bool Success { get; set; }
    public string Message { get; set; } = string.Empty;
    public int ProductsSynced { get; set; }
    public int TransactionsSynced { get; set; }
    public DateTime LastSync { get; set; }
}

// POSDesktop.Sync/SyncService.cs
namespace POSDesktop.Sync;

public class SyncService : ISyncService
{
    private readonly IApiClient _apiClient;
    private readonly IProductRepository _productRepository;
    private readonly ITransactionRepository _transactionRepository;
    private readonly IStoreRepository _storeRepository;
    private readonly ILogger<SyncService> _logger;
    
    public SyncService(IApiClient apiClient, IProductRepository productRepository,
        ITransactionRepository transactionRepository, IStoreRepository storeRepository,
        ILogger<SyncService> logger)
    {
        _apiClient = apiClient;
        _productRepository = productRepository;
        _transactionRepository = transactionRepository;
        _storeRepository = storeRepository;
        _logger = logger;
    }
    
    public async Task<SyncResult> SyncAsync()
    {
        var result = new SyncResult();
        
        try
        {
            // Check connection first
            if (!await CheckConnectionAsync())
            {
                result.Message = "No connection to cloud server";
                return result;
            }
            
            var store = await _storeRepository.GetCurrentStoreAsync();
            if (store == null)
            {
                result.Message = "No store configured";
                return result;
            }
            
            // Sync products
            var productResult = await SyncProductsAsync();
            result.ProductsSynced = productResult.ProductsSynced;
            
            // Sync transactions
            var transactionResult = await SyncTransactionsAsync();
            result.TransactionsSynced = transactionResult.TransactionsSynced;
            
            // Update last sync time
            store.LastSync = DateTime.UtcNow;
            await _storeRepository.UpdateAsync(store);
            
            result.Success = true;
            result.Message = "Sync completed successfully";
            result.LastSync = store.LastSync;
            
            _logger.LogInformation("Sync completed: {Products} products, {Transactions} transactions", 
                result.ProductsSynced, result.TransactionsSynced);
        }
        catch (Exception ex)
        {
            result.Message = $"Sync failed: {ex.Message}";
            _logger.LogError(ex, "Sync failed");
        }
        
        return result;
    }
    
    public async Task<SyncResult> SyncProductsAsync()
    {
        var result = new SyncResult();
        var store = await _storeRepository.GetCurrentStoreAsync();
        
        if (store == null) return result;
        
        try
        {
            // Get local changes to push to cloud
            var dirtyProducts = await _productRepository.GetDirtyProductsAsync();
            
            // Get cloud changes to pull to local
            var cloudProducts = await _apiClient.GetProductsAsync(store.LastSync);
            
            // Merge changes
            foreach (var cloudProduct in cloudProducts)
            {
                var localProduct = await _productRepository.GetByIdAsync(cloudProduct.Id);
                
                if (localProduct == null)
                {
                    // New product from cloud
                    await _productRepository.AddAsync(cloudProduct);
                    result.ProductsSynced++;
                }
                else if (cloudProduct.UpdatedAt > localProduct.UpdatedAt)
                {
                    // Cloud version is newer
                    MapProductProperties(localProduct, cloudProduct);
                    await _productRepository.UpdateAsync(localProduct);
                    result.ProductsSynced++;
                }
            }
            
            // Push local changes to cloud
            foreach (var localProduct in dirtyProducts)
            {
                if (localProduct.UpdatedAt > store.LastSync)
                {
                    await _apiClient.UpdateProductAsync(localProduct);
                    localProduct.IsDirty = false;
                    await _productRepository.UpdateAsync(localProduct);
                    result.ProductsSynced++;
                }
            }
            
            result.Success = true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Product sync failed");
            throw;
        }
        
        return result;
    }
    
    public async Task<bool> CheckConnectionAsync()
    {
        try
        {
            return await _apiClient.TestConnectionAsync();
        }
        catch
        {
            return false;
        }
    }
    
    private void MapProductProperties(Product target, Product source)
    {
        target.Name = source.Name;
        target.Price = source.Price;
        target.StockQuantity = source.StockQuantity;
        // ... other properties
    }
}
Step 7: SignalR Client Integration
7.1 SignalR Service untuk Desktop
csharp
// POSDesktop.Sync/SignalRService.cs
namespace POSDesktop.Sync;

public interface ISignalRService
{
    Task ConnectAsync();
    Task DisconnectAsync();
    Task JoinStoreAsync(Guid storeId);
    Task UpdateStockAsync(Guid productId, int newStock);
    
    event EventHandler<StockUpdatedEventArgs> StockUpdated;
    event EventHandler<NewTransactionEventArgs> NewTransaction;
    event EventHandler<ProductPriceUpdatedEventArgs> ProductPriceUpdated;
}

public class SignalRService : ISignalRService
{
    private HubConnection? _connection;
    private readonly string _hubUrl;
    private readonly string _accessToken;
    private readonly ILogger<SignalRService> _logger;
    
    public event EventHandler<StockUpdatedEventArgs>? StockUpdated;
    public event EventHandler<NewTransactionEventArgs>? NewTransaction;
    public event EventHandler<ProductPriceUpdatedEventArgs>? ProductPriceUpdated;
    
    public SignalRService(string hubUrl, string accessToken, ILogger<SignalRService> logger)
    {
        _hubUrl = hubUrl;
        _accessToken = accessToken;
        _logger = logger;
    }
    
    public async Task ConnectAsync()
    {
        try
        {
            _connection = new HubConnectionBuilder()
                .WithUrl(_hubUrl, options =>
                {
                    options.AccessTokenProvider = () => Task.FromResult(_accessToken);
                })
                .WithAutomaticReconnect()
                .Build();
            
            SetupEventHandlers();
            
            await _connection.StartAsync();
            _logger.LogInformation("SignalR connected to {HubUrl}", _hubUrl);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "SignalR connection failed");
            throw;
        }
    }
    
    private void SetupEventHandlers()
    {
        _connection.On<Guid, int>("StockUpdated", (productId, newStock) =>
        {
            StockUpdated?.Invoke(this, new StockUpdatedEventArgs
            {
                ProductId = productId,
                NewStock = newStock
            });
        });
        
        _connection.On<Transaction>("NewTransaction", (transaction) =>
        {
            NewTransaction?.Invoke(this, new NewTransactionEventArgs
            {
                Transaction = transaction
            });
        });
        
        _connection.On<Guid, decimal>("ProductPriceUpdated", (productId, newPrice) =>
        {
            ProductPriceUpdated?.Invoke(this, new ProductPriceUpdatedEventArgs
            {
                ProductId = productId,
                NewPrice = newPrice
            });
        });
        
        _connection.Closed += async (error) =>
        {
            _logger.LogWarning("SignalR connection closed: {Error}", error?.Message);
            await Task.Delay(new Random().Next(0, 5) * 1000);
            await ConnectAsync();
        };
        
        _connection.Reconnecting += (error) =>
        {
            _logger.LogInformation("SignalR reconnecting...");
            return Task.CompletedTask;
        };
        
        _connection.Reconnected += (connectionId) =>
        {
            _logger.LogInformation("SignalR reconnected");
            return Task.CompletedTask;
        };
    }
    
    public async Task JoinStoreAsync(Guid storeId)
    {
        if (_connection?.State == HubConnectionState.Connected)
        {
            await _connection.InvokeAsync("JoinStore", storeId.ToString());
        }
    }
    
    public async Task UpdateStockAsync(Guid productId, int newStock)
    {
        if (_connection?.State == HubConnectionState.Connected)
        {
            await _connection.InvokeAsync("UpdateStock", productId.ToString(), newStock);
        }
    }
    
    public async Task DisconnectAsync()
    {
        if (_connection != null)
        {
            await _connection.StopAsync();
            await _connection.DisposeAsync();
        }
    }
}
Step 8: Dependency Injection dan Configuration
8.1 Program.cs dengan Dependency Injection
csharp
// POSDesktop/Program.cs
namespace POSDesktop;

internal static class Program
{
    private static ServiceProvider? _serviceProvider;
    
    [STAThread]
    static void Main()
    {
        ApplicationConfiguration.Initialize();
        
        // Setup dependency injection
        var services = new ServiceCollection();
        ConfigureServices(services);
        
        _serviceProvider = services.BuildServiceProvider();
        
        // Ensure database exists and is migrated
        EnsureDatabase();
        
        // Run the application
        var mainForm = _serviceProvider.GetRequiredService<MainForm>();
        Application.Run(mainForm);
    }
    
    private static void ConfigureServices(IServiceCollection services)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseNpgsql("Host=localhost;Database=pos_desktop;Username=postgres;Password=password"));
        
        // Repositories
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<ITransactionRepository, TransactionRepository>();
        services.AddScoped<IStoreRepository, StoreRepository>();
        
        // Services
        services.AddScoped<ITransactionService, TransactionService>();
        services.AddScoped<ISyncService, SyncService>();
        services.AddScoped<IApiClient, ApiClient>();
        
        // Hardware
        services.AddScoped<IHardwareService, HardwareService>();
        services.AddScoped<IBarcodeScanner, BarcodeScanner>();
        services.AddScoped<IReceiptPrinter, ReceiptPrinter>();
        services.AddScoped<ICustomerDisplay, CustomerDisplay>();
        services.AddScoped<ICashDrawer, CashDrawer>();
        
        // SignalR
        services.AddScoped<ISignalRService>(provider =>
        {
            var config = provider.GetRequiredService<IConfiguration>();
            return new SignalRService(
                config["SignalR:HubUrl"]!,
                config["SignalR:AccessToken"]!,
                provider.GetRequiredService<ILogger<SignalRService>>());
        });
        
        // Forms
        services.AddTransient<MainForm>();
        services.AddTransient<ProductManagementForm>();
        services.AddTransient<TransactionHistoryForm>();
        
        // Configuration
        var configuration = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false)
            .Build();
        
        services.AddSingleton<IConfiguration>(configuration);
        
        // Logging
        services.AddLogging(builder =>
        {
            builder.AddConsole();
            builder.AddFile("Logs/pos-desktop-{Date}.txt");
        });
    }
    
    private static void EnsureDatabase()
    {
        using var scope = _serviceProvider!.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        dbContext.Database.EnsureCreated();
    }
}
8.2 appsettings.json
json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=pos_desktop;Username=postgres;Password=password"
  },
  "Api": {
    "BaseUrl": "https://your-api-domain.com/api",
    "Timeout": 30
  },
  "SignalR": {
    "HubUrl": "https://your-api-domain.com/poshub",
    "AccessToken": "your-jwt-token"
  },
  "Hardware": {
    "BarcodeScannerPort": "COM3",
    "ReceiptPrinterName": "XP-58",
    "CustomerDisplayPort": "COM4"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  }
}
Step 9: Deployment dan Installation
9.1 Installer Setup
Buat installer menggunakan Inno Setup atau Windows Installer:

ini
; setup.iss
[Setup]
AppName=POS Desktop
AppVersion=1.0.0
DefaultDirName={pf}\POS Desktop
DefaultGroupName=POS Desktop
OutputDir=Output
OutputBaseFilename=POSDesktopSetup

[Files]
Source: "POSDesktop\bin\Release\net8.0-windows\*"; DestDir: "{app}"; Flags: ignoreversion

[Icons]
Name: "{group}\POS Desktop"; Filename: "{app}\POSDesktop.exe"
Name: "{autodesktop}\POS Desktop"; Filename: "{app}\POSDesktop.exe"

[Run]
Filename: "{app}\POSDesktop.exe"; Description: "Launch POS Desktop"; Flags: nowait postinstall skipifsilent
9.2 Prerequisites Installer
Include .NET 8 Runtime dan PostgreSQL dalam installer:

xml
<!-- Bundle configuration -->
<ItemGroup>
  <RuntimePack Include="Microsoft.WindowsDesktop.App.Runtime.win-x64" />
  <PublishReadyToRun>true</PublishReadyToRun>
</ItemGroup>
Next Steps
Setelah Phase 4 complete, lanjut ke:

Phase 5: Mobile Application (Flutter)

Phase 6: Deployment dan Production Setup

Phase 7: Testing dan Quality Assurance

Desktop application sekarang ready untuk development dan testing! 🚀
