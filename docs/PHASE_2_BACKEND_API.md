📁 [FILE: PHASE_2_BACKEND_API.md]
📝 [COMMIT: "Add complete Phase 2 Backend API documentation"]
markdown
# Phase 2: Backend API Development

## Overview
Backend API menggunakan .NET 8 dengan Entity Framework Core untuk PostgreSQL. Support real-time features dengan SignalR dan payment gateway integration.

## Project Structure
POSCloud.Backend/
├── 📁 src/
│ ├── POSCloud.API/ # Main API project
│ ├── POSCloud.SignalR/ # Real-time hub
│ ├── POSCloud.Core/ # Domain models & interfaces
│ ├── POSCloud.Infrastructure/ # Data access & external services
│ └── POSCloud.Shared/ # Common utilities
├── 📁 tests/
│ ├── POSCloud.API.Tests/
│ └── POSCloud.Infrastructure.Tests/
└── 📁 deployment/
├── docker-compose.yml
└── nginx/

text

## Step 1: Create .NET 8 Solution Structure

### 1.1 Initialize Solution dan Projects
```bash
# Create solution directory
mkdir POSCloud.Backend
cd POSCloud.Backend

# Create solution file
dotnet new sln -n POSCloud

# Create projects
dotnet new webapi -n src/POSCloud.API
dotnet new webapi -n src/POSCloud.SignalR  
dotnet new classlib -n src/POSCloud.Core
dotnet new classlib -n src/POSCloud.Infrastructure
dotnet new classlib -n src/POSCloud.Shared

# Add projects to solution
dotnet sln add src/POSCloud.API/POSCloud.API.csproj
dotnet sln add src/POSCloud.SignalR/POSCloud.SignalR.csproj
dotnet sln add src/POSCloud.Core/POSCloud.Core.csproj
dotnet sln add src/POSCloud.Infrastructure/POSCloud.Infrastructure.csproj
dotnet sln add src/POSCloud.Shared/POSCloud.Shared.csproj

# Create test projects
dotnet new xunit -n tests/POSCloud.API.Tests
dotnet new xunit -n tests/POSCloud.Infrastructure.Tests
dotnet sln add tests/POSCloud.API.Tests/POSCloud.API.Tests.csproj
dotnet sln add tests/POSCloud.Infrastructure.Tests/POSCloud.Infrastructure.Tests.csproj
1.2 Configure Project Dependencies
bash
# API depends on Core dan Infrastructure
dotnet add src/POSCloud.API reference src/POSCloud.Core/ src/POSCloud.Infrastructure/

# SignalR depends on Core dan Shared
dotnet add src/POSCloud.SignalR reference src/POSCloud.Core/ src/POSCloud.Shared/

# Infrastructure depends on Core
dotnet add src/POSCloud.Infrastructure reference src/POSCloud.Core/

# Tests depend on corresponding projects
dotnet add tests/POSCloud.API.Tests reference src/POSCloud.API/
dotnet add tests/POSCloud.Infrastructure.Tests reference src/POSCloud.Infrastructure/
Step 2: Domain Models (Core Project)
2.1 Entity Models
csharp
// src/POSCloud.Core/Entities/Product.cs
namespace POSCloud.Core.Entities;

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
    public Store? Store { get; set; }
    
    // Sync properties
    public DateTime LastSynced { get; set; }
    public bool IsDirty { get; set; }
    
    public ICollection<TransactionItem> TransactionItems { get; set; } = new List<TransactionItem>();
}

// src/POSCloud.Core/Entities/Transaction.cs
public class Transaction : BaseEntity
{
    public string TransactionNumber { get; set; } = string.Empty;
    public Guid StoreId { get; set; }
    public Store? Store { get; set; }
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
    public string? PaymentGatewayResponse { get; set; }
    
    public ICollection<TransactionItem> Items { get; set; } = new List<TransactionItem>();
}

// src/POSCloud.Core/Entities/TransactionItem.cs
public class TransactionItem : BaseEntity
{
    public Guid TransactionId { get; set; }
    public Transaction? Transaction { get; set; }
    public Guid ProductId { get; set; }
    public Product? Product { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal DiscountAmount { get; set; }
    public decimal TotalPrice { get; set; }
}

// src/POSCloud.Core/Entities/Store.cs
public class Store : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
    public string? Address { get; set; }
    public string? Phone { get; set; }
    public string? Email { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime SubscriptionExpiry { get; set; }
    
    // Hardware configuration
    public string? ReceiptPrinterConfig { get; set; }
    public string? BarcodeScannerConfig { get; set; }
    public string? CustomerDisplayConfig { get; set; }
    
    public ICollection<Product> Products { get; set; } = new List<Product>();
    public ICollection<Transaction> Transactions { get; set; } = new List<Transaction>();
    public ICollection<StoreUser> StoreUsers { get; set; } = new List<StoreUser>();
}
2.2 Enums dan Common Types
csharp
// src/POSCloud.Core/Enums/PaymentMethod.cs
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

// src/POSCloud.Core/Enums/TransactionStatus.cs
public enum TransactionStatus
{
    Pending = 1,
    Completed = 2,
    Cancelled = 3,
    Refunded = 4
}

// src/POSCloud.Core/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
}
2.3 DTOs (Data Transfer Objects)
csharp
// src/POSCloud.Core/DTOs/ProductDtos.cs
namespace POSCloud.Core.DTOs;

public class ProductDto
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
    public string Barcode { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public string Category { get; set; } = string.Empty;
    public string Unit { get; set; } = "pcs";
    public bool IsActive { get; set; }
}

public class CreateProductDto
{
    [Required] public string Name { get; set; } = string.Empty;
    [Required] public string Code { get; set; } = string.Empty;
    public string? Barcode { get; set; }
    [Range(0.01, double.MaxValue)] public decimal Price { get; set; }
    public decimal CostPrice { get; set; }
    public int StockQuantity { get; set; } = 0;
    public string Category { get; set; } = "General";
    public string Unit { get; set; } = "pcs";
}

public class UpdateProductDto
{
    [Required] public string Name { get; set; } = string.Empty;
    [Range(0.01, double.MaxValue)] public decimal Price { get; set; }
    public int StockQuantity { get; set; }
}

// src/POSCloud.Core/DTOs/TransactionDtos.cs
public class TransactionDto
{
    public Guid Id { get; set; }
    public string TransactionNumber { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public decimal AmountPaid { get; set; }
    public decimal ChangeAmount { get; set; }
    public PaymentMethod PaymentMethod { get; set; }
    public DateTime TransactionDate { get; set; }
    public List<TransactionItemDto> Items { get; set; } = new();
}

public class CreateTransactionDto
{
    [Required] public Guid StoreId { get; set; }
    public string? CustomerName { get; set; }
    public string? CustomerPhone { get; set; }
    
    [Required, MinLength(1)] 
    public List<TransactionItemDto> Items { get; set; } = new();
    
    [Required] public PaymentMethod PaymentMethod { get; set; }
    public decimal DiscountAmount { get; set; }
    public decimal TaxAmount { get; set; }
    public decimal AmountPaid { get; set; }
}

public class TransactionItemDto
{
    [Required] public Guid ProductId { get; set; }
    [Range(1, int.MaxValue)] public int Quantity { get; set; }
    [Range(0.01, double.MaxValue)] public decimal UnitPrice { get; set; }
    public decimal DiscountAmount { get; set; }
}

// src/POSCloud.Core/DTOs/SyncDtos.cs
public class SyncRequestDto
{
    [Required] public Guid StoreId { get; set; }
    public DateTime LastSync { get; set; } = DateTime.MinValue;
    public List<ProductDto> Products { get; set; } = new();
    public List<TransactionDto> Transactions { get; set; } = new();
}

public class SyncResponseDto
{
    public bool Success { get; set; }
    public string Message { get; set; } = string.Empty;
    public DateTime ServerTime { get; set; } = DateTime.UtcNow;
    public List<ProductDto> Products { get; set; } = new();
    public List<TransactionDto> Transactions { get; set; } = new();
    public List<Guid> DeletedProducts { get; set; } = new();
}
Step 3: Infrastructure Layer (Data Access)
3.1 Database Context
csharp
// src/POSCloud.Infrastructure/Data/ApplicationDbContext.cs
namespace POSCloud.Infrastructure.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }
    
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Transaction> Transactions => Set<Transaction>();
    public DbSet<TransactionItem> TransactionItems => Set<TransactionItem>();
    public DbSet<Store> Stores => Set<Store>();
    public DbSet<StoreUser> StoreUsers => Set<StoreUser>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Product configuration
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasIndex(p => new { p.StoreId, p.Code }).IsUnique();
            entity.HasIndex(p => new { p.StoreId, p.Barcode }).IsUnique();
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.Property(p => p.CostPrice).HasPrecision(18, 2);
            
            entity.HasOne(p => p.Store)
                  .WithMany(s => s.Products)
                  .HasForeignKey(p => p.StoreId);
        });
        
        // Transaction configuration
        modelBuilder.Entity<Transaction>(entity =>
        {
            entity.HasIndex(t => t.TransactionNumber).IsUnique();
            entity.Property(t => t.Subtotal).HasPrecision(18, 2);
            entity.Property(t => t.TaxAmount).HasPrecision(18, 2);
            entity.Property(t => t.DiscountAmount).HasPrecision(18, 2);
            entity.Property(t => t.TotalAmount).HasPrecision(18, 2);
            entity.Property(t => t.AmountPaid).HasPrecision(18, 2);
            entity.Property(t => t.ChangeAmount).HasPrecision(18, 2);
            
            entity.HasOne(t => t.Store)
                  .WithMany(s => s.Transactions)
                  .HasForeignKey(t => t.StoreId);
        });
        
        // TransactionItem configuration
        modelBuilder.Entity<TransactionItem>(entity =>
        {
            entity.Property(ti => ti.UnitPrice).HasPrecision(18, 2);
            entity.Property(ti => ti.DiscountAmount).HasPrecision(18, 2);
            entity.Property(ti => ti.TotalPrice).HasPrecision(18, 2);
        });
        
        // Store configuration
        modelBuilder.Entity<Store>(entity =>
        {
            entity.HasIndex(s => s.Code).IsUnique();
            entity.HasIndex(s => s.Name);
        });
    }
    
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Auto-update UpdatedAt timestamp
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
        
        return await base.SaveChangesAsync(cancellationToken);
    }
}
3.2 Repository Pattern Implementation
csharp
// src/POSCloud.Core/Interfaces/IProductRepository.cs
namespace POSCloud.Core.Interfaces;

public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<Product?> GetByCodeAsync(Guid storeId, string code);
    Task<List<Product>> GetByStoreAsync(Guid storeId, bool includeInactive = false);
    Task<List<Product>> GetProductsForSyncAsync(Guid storeId, DateTime lastSync);
    Task AddAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(Product product);
    Task<bool> CodeExistsAsync(Guid storeId, string code);
}

// src/POSCloud.Infrastructure/Repositories/ProductRepository.cs
namespace POSCloud.Infrastructure.Repositories;

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
            .Include(p => p.Store)
            .FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public async Task<Product?> GetByCodeAsync(Guid storeId, string code)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.StoreId == storeId && p.Code == code);
    }
    
    public async Task<List<Product>> GetByStoreAsync(Guid storeId, bool includeInactive = false)
    {
        var query = _context.Products
            .Where(p => p.StoreId == storeId);
            
        if (!includeInactive)
        {
            query = query.Where(p => p.IsActive);
        }
        
        return await query
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<List<Product>> GetProductsForSyncAsync(Guid storeId, DateTime lastSync)
    {
        return await _context.Products
            .Where(p => p.StoreId == storeId && p.UpdatedAt > lastSync)
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
    
    public async Task DeleteAsync(Product product)
    {
        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task<bool> CodeExistsAsync(Guid storeId, string code)
    {
        return await _context.Products
            .AnyAsync(p => p.StoreId == storeId && p.Code == code);
    }
}
3.3 Payment Gateway Integration
csharp
// src/POSCloud.Core/Interfaces/IPaymentGateway.cs
namespace POSCloud.Core.Interfaces;

public interface IPaymentGateway
{
    Task<PaymentResponse> ProcessPaymentAsync(PaymentRequest request);
    Task<PaymentStatusResponse> CheckPaymentStatusAsync(string transactionId);
    Task<RefundResponse> ProcessRefundAsync(RefundRequest request);
}

public class PaymentRequest
{
    public Guid TransactionId { get; set; }
    public decimal Amount { get; set; }
    public PaymentMethod PaymentMethod { get; set; }
    public string? CustomerEmail { get; set; }
    public string? CustomerPhone { get; set; }
    public string? Description { get; set; }
}

public class PaymentResponse
{
    public bool Success { get; set; }
    public string? TransactionId { get; set; }
    public string? PaymentUrl { get; set; } // Untuk QRIS/e-wallet
    public string? Message { get; set; }
    public DateTime ExpiryTime { get; set; }
}

// src/POSCloud.Infrastructure/Services/PaymentGatewayService.cs
namespace POSCloud.Infrastructure.Services;

public class PaymentGatewayService : IPaymentGateway
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<PaymentGatewayService> _logger;
    
    public PaymentGatewayService(HttpClient httpClient, IConfiguration configuration, 
        ILogger<PaymentGatewayService> logger)
    {
        _httpClient = httpClient;
        _configuration = configuration;
        _logger = logger;
    }
    
    public async Task<PaymentResponse> ProcessPaymentAsync(PaymentRequest request)
    {
        try
        {
            // Implementation untuk berbagai payment method
            return request.PaymentMethod switch
            {
                PaymentMethod.QRIS => await ProcessQRISPaymentAsync(request),
                PaymentMethod.CreditCard => await ProcessCreditCardPaymentAsync(request),
                PaymentMethod.Gopay => await ProcessGopayPaymentAsync(request),
                PaymentMethod.OVO => await ProcessOvoPaymentAsync(request),
                PaymentMethod.Dana => await ProcessDanaPaymentAsync(request),
                _ => new PaymentResponse { Success = true } // Cash handling
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing payment for transaction {TransactionId}", request.TransactionId);
            return new PaymentResponse { Success = false, Message = ex.Message };
        }
    }
    
    private async Task<PaymentResponse> ProcessQRISPaymentAsync(PaymentRequest request)
    {
        // Integrasi dengan Midtrans/Xendit untuk QRIS
        var qrisRequest = new
        {
            transaction_id = request.TransactionId.ToString(),
            amount = request.Amount,
            payment_method = "qris",
            customer = new { email = request.CustomerEmail, phone = request.CustomerPhone }
        };
        
        var response = await _httpClient.PostAsJsonAsync("qris/payments", qrisRequest);
        if (response.IsSuccessStatusCode)
        {
            var content = await response.Content.ReadFromJsonAsync<dynamic>();
            return new PaymentResponse 
            { 
                Success = true,
                TransactionId = content?.transaction_id,
                PaymentUrl = content?.payment_url,
                ExpiryTime = DateTime.UtcNow.AddMinutes(30)
            };
        }
        
        return new PaymentResponse { Success = false, Message = "QRIS payment failed" };
    }
    
    // Implementations untuk payment methods lainnya...
}
Step 4: API Controllers
4.1 Products Controller
csharp
// src/POSCloud.API/Controllers/ProductsController.cs
namespace POSCloud.API.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _productRepository;
    private readonly ILogger<ProductsController> _logger;
    private readonly IMapper _mapper;
    
    public ProductsController(IProductRepository productRepository, 
        ILogger<ProductsController> logger, IMapper mapper)
    {
        _productRepository = productRepository;
        _logger = logger;
        _mapper = mapper;
    }
    
    [HttpGet]
    public async Task<ActionResult<ApiResponse<List<ProductDto>>>> GetProducts(
        [FromQuery] Guid storeId, [FromQuery] bool includeInactive = false)
    {
        try
        {
            var products = await _productRepository.GetByStoreAsync(storeId, includeInactive);
            var productDtos = _mapper.Map<List<ProductDto>>(products);
            
            return Ok(ApiResponse<List<ProductDto>>.Success(productDtos));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting products for store {StoreId}", storeId);
            return StatusCode(500, ApiResponse<List<ProductDto>>.Error("Internal server error"));
        }
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ApiResponse<ProductDto>>> GetProduct(Guid id)
    {
        try
        {
            var product = await _productRepository.GetByIdAsync(id);
            if (product == null)
            {
                return NotFound(ApiResponse<ProductDto>.Error("Product not found"));
            }
            
            var productDto = _mapper.Map<ProductDto>(product);
            return Ok(ApiResponse<ProductDto>.Success(productDto));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting product {ProductId}", id);
            return StatusCode(500, ApiResponse<ProductDto>.Error("Internal server error"));
        }
    }
    
    [HttpPost]
    public async Task<ActionResult<ApiResponse<ProductDto>>> CreateProduct(CreateProductDto createProductDto)
    {
        try
        {
            // Validate product code uniqueness
            if (await _productRepository.CodeExistsAsync(createProductDto.StoreId, createProductDto.Code))
            {
                return BadRequest(ApiResponse<ProductDto>.Error("Product code already exists"));
            }
            
            var product = _mapper.Map<Product>(createProductDto);
            await _productRepository.AddAsync(product);
            
            var productDto = _mapper.Map<ProductDto>(product);
            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, 
                ApiResponse<ProductDto>.Success(productDto));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating product");
            return StatusCode(500, ApiResponse<ProductDto>.Error("Internal server error"));
        }
    }
    
    [HttpPut("{id}")]
    public async Task<ActionResult<ApiResponse<ProductDto>>> UpdateProduct(
        Guid id, UpdateProductDto updateProductDto)
    {
        try
        {
            var product = await _productRepository.GetByIdAsync(id);
            if (product == null)
            {
                return NotFound(ApiResponse<ProductDto>.Error("Product not found"));
            }
            
            _mapper.Map(updateProductDto, product);
            await _productRepository.UpdateAsync(product);
            
            var productDto = _mapper.Map<ProductDto>(product);
            return Ok(ApiResponse<ProductDto>.Success(productDto));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating product {ProductId}", id);
            return StatusCode(500, ApiResponse<ProductDto>.Error("Internal server error"));
        }
    }
    
    [HttpDelete("{id}")]
    public async Task<ActionResult<ApiResponse>> DeleteProduct(Guid id)
    {
        try
        {
            var product = await _productRepository.GetByIdAsync(id);
            if (product == null)
            {
                return NotFound(ApiResponse.Error("Product not found"));
            }
            
            await _productRepository.DeleteAsync(product);
            return Ok(ApiResponse.Success("Product deleted successfully"));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting product {ProductId}", id);
            return StatusCode(500, ApiResponse.Error("Internal server error"));
        }
    }
}
4.2 Transactions Controller
csharp
// src/POSCloud.API/Controllers/TransactionsController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class TransactionsController : ControllerBase
{
    private readonly ITransactionRepository _transactionRepository;
    private readonly IProductRepository _productRepository;
    private readonly IPaymentGateway _paymentGateway;
    private readonly ILogger<TransactionsController> _logger;
    private readonly IMapper _mapper;
    
    public TransactionsController(ITransactionRepository transactionRepository,
        IProductRepository productRepository, IPaymentGateway paymentGateway,
        ILogger<TransactionsController> logger, IMapper mapper)
    {
        _transactionRepository = transactionRepository;
        _productRepository = productRepository;
        _paymentGateway = paymentGateway;
        _logger = logger;
        _mapper = mapper;
    }
    
    [HttpPost]
    public async Task<ActionResult<ApiResponse<TransactionDto>>> CreateTransaction(
        CreateTransactionDto createTransactionDto)
    {
        using var transaction = await _transactionRepository.BeginTransactionAsync();
        
        try
        {
            // Validate products and stock
            foreach (var itemDto in createTransactionDto.Items)
            {
                var product = await _productRepository.GetByIdAsync(itemDto.ProductId);
                if (product == null)
                {
                    return BadRequest(ApiResponse<TransactionDto>.Error($"Product {itemDto.ProductId} not found"));
                }
                
                if (product.StockQuantity < itemDto.Quantity)
                {
                    return BadRequest(ApiResponse<TransactionDto>.Error(
                        $"Insufficient stock for product {product.Name}"));
                }
            }
            
            // Create transaction
            var transactionEntity = _mapper.Map<Transaction>(createTransactionDto);
            transactionEntity.TransactionNumber = GenerateTransactionNumber();
            
            // Calculate totals
            CalculateTransactionTotals(transactionEntity);
            
            // Process payment untuk non-cash methods
            if (createTransactionDto.PaymentMethod != PaymentMethod.Cash)
            {
                var paymentRequest = new PaymentRequest
                {
                    TransactionId = transactionEntity.Id,
                    Amount = transactionEntity.TotalAmount,
                    PaymentMethod = createTransactionDto.PaymentMethod,
                    CustomerEmail = createTransactionDto.CustomerEmail,
                    CustomerPhone = createTransactionDto.CustomerPhone
                };
                
                var paymentResponse = await _paymentGateway.ProcessPaymentAsync(paymentRequest);
                if (!paymentResponse.Success)
                {
                    return BadRequest(ApiResponse<TransactionDto>.Error(
                        $"Payment failed: {paymentResponse.Message}"));
                }
                
                transactionEntity.PaymentReference = paymentResponse.TransactionId;
            }
            
            // Update product stock
            foreach (var item in transactionEntity.Items)
            {
                var product = await _productRepository.GetByIdAsync(item.ProductId);
                product!.StockQuantity -= item.Quantity;
                await _productRepository.UpdateAsync(product);
            }
            
            await _transactionRepository.AddAsync(transactionEntity);
            await transaction.CommitAsync();
            
            var transactionDto = _mapper.Map<TransactionDto>(transactionEntity);
            return CreatedAtAction(nameof(GetTransaction), new { id = transactionEntity.Id },
                ApiResponse<TransactionDto>.Success(transactionDto));
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "Error creating transaction");
            return StatusCode(500, ApiResponse<TransactionDto>.Error("Internal server error"));
        }
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ApiResponse<TransactionDto>>> GetTransaction(Guid id)
    {
        try
        {
            var transaction = await _transactionRepository.GetByIdAsync(id);
            if (transaction == null)
            {
                return NotFound(ApiResponse<TransactionDto>.Error("Transaction not found"));
            }
            
            var transactionDto = _mapper.Map<TransactionDto>(transaction);
            return Ok(ApiResponse<TransactionDto>.Success(transactionDto));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting transaction {TransactionId}", id);
            return StatusCode(500, ApiResponse<TransactionDto>.Error("Internal server error"));
        }
    }
    
    [HttpGet("store/{storeId}")]
    public async Task<ActionResult<ApiResponse<List<TransactionDto>>>> GetStoreTransactions(
        Guid storeId, [FromQuery] DateTime? startDate, [FromQuery] DateTime? endDate)
    {
        try
        {
            var transactions = await _transactionRepository.GetByStoreAsync(storeId, startDate, endDate);
            var transactionDtos = _mapper.Map<List<TransactionDto>>(transactions);
            
            return Ok(ApiResponse<List<TransactionDto>>.Success(transactionDtos));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting transactions for store {StoreId}", storeId);
            return StatusCode(500, ApiResponse<List<TransactionDto>>.Error("Internal server error"));
        }
    }
    
    private string GenerateTransactionNumber()
    {
        return $"TXN{DateTime.UtcNow:yyyyMMddHHmmss}{Random.Shared.Next(1000, 9999)}";
    }
    
    private void CalculateTransactionTotals(Transaction transaction)
    {
        transaction.Subtotal = transaction.Items.Sum(i => i.UnitPrice * i.Quantity);
        transaction.TotalAmount = transaction.Subtotal + transaction.TaxAmount - transaction.DiscountAmount;
        
        if (transaction.PaymentMethod == PaymentMethod.Cash)
        {
            transaction.ChangeAmount = transaction.AmountPaid - transaction.TotalAmount;
        }
    }
}
4.3 Sync Controller
csharp
// src/POSCloud.API/Controllers/SyncController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class SyncController : ControllerBase
{
    private readonly ISyncService _syncService;
    private readonly ILogger<SyncController> _logger;
    
    public SyncController(ISyncService syncService, ILogger<SyncController> logger)
    {
        _syncService = syncService;
        _logger = logger;
    }
    
    [HttpPost("data")]
    public async Task<ActionResult<ApiResponse<SyncResponseDto>>> SyncData(SyncRequestDto syncRequest)
    {
        try
        {
            var syncResult = await _syncService.SyncDataAsync(syncRequest);
            
            if (syncResult.Success)
            {
                return Ok(ApiResponse<SyncResponseDto>.Success(syncResult));
            }
            else
            {
                return BadRequest(ApiResponse<SyncResponseDto>.Error(syncResult.Message));
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during sync for store {StoreId}", syncRequest.StoreId);
            return StatusCode(500, ApiResponse<SyncResponseDto>.Error("Sync failed"));
        }
    }
    
    [HttpGet("products/{storeId}")]
    public async Task<ActionResult<ApiResponse<List<ProductDto>>>> GetProductsForSync(
        Guid storeId, [FromQuery] DateTime lastSync)
    {
        try
        {
            var products = await _syncService.GetProductsForSyncAsync(storeId, lastSync);
            return Ok(ApiResponse<List<ProductDto>>.Success(products));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting products for sync");
            return StatusCode(500, ApiResponse<List<ProductDto>>.Error("Internal server error"));
        }
    }
}
Step 5: Dependency Injection dan Configuration
5.1 Program.cs Configuration
csharp
// src/POSCloud.API/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Database Configuration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// AutoMapper
builder.Services.AddAutoMapper(typeof(Program));

// Repositories
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<ITransactionRepository, TransactionRepository>();
builder.Services.AddScoped<IStoreRepository, StoreRepository>();

// Services
builder.Services.AddScoped<ISyncService, SyncService>();
builder.Services.AddScoped<IPaymentGateway, PaymentGatewayService>();

// HttpClient untuk payment gateways
builder.Services.AddHttpClient<IPaymentGateway, PaymentGatewayService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["PaymentGateway:BaseUrl"]);
    client.DefaultRequestHeaders.Add("Authorization", 
        $"Basic {Convert.ToBase64String(Encoding.UTF8.GetBytes(builder.Configuration["PaymentGateway:ApiKey"]))}");
});

// JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]))
        };
    });

// CORS untuk mobile apps
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowMobileApps", policy =>
    {
        policy.WithOrigins(
                "http://localhost:3000", // Flutter web
                "capacitor://localhost", // Capacitor
                "ionic://localhost")     // Ionic
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowMobileApps");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

// Database migration
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    dbContext.Database.Migrate();
}

app.Run();
5.2 appsettings.json
json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=pos_cloud;Username=pos_user;Password=your_secure_password"
  },
  "Jwt": {
    "Secret": "your-super-secret-key-at-least-32-characters-long",
    "Issuer": "POSCloud",
    "Audience": "POSCloud-Users",
    "ExpiryMinutes": 1440
  },
  "PaymentGateway": {
    "BaseUrl": "https://api.midtrans.com/v2",
    "ApiKey": "your-midtrans-server-key",
    "MerchantId": "your-merchant-id"
  },
  "Redis": {
    "ConnectionString": "localhost:6379,password=your-redis-password"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
Step 6: Testing
6.1 Unit Tests
csharp
// tests/POSCloud.API.Tests/ProductsControllerTests.cs
namespace POSCloud.API.Tests;

public class ProductsControllerTests
{
    private readonly ProductsController _controller;
    private readonly Mock<IProductRepository> _mockRepository;
    private readonly Mock<ILogger<ProductsController>> _mockLogger;
    private readonly IMapper _mapper;
    
    public ProductsControllerTests()
    {
        _mockRepository = new Mock<IProductRepository>();
        _mockLogger = new Mock<ILogger<ProductsController>>();
        
        var config = new MapperConfiguration(cfg =>
        {
            cfg.AddProfile<AutoMapperProfile>();
        });
        _mapper = config.CreateMapper();
        
        _controller = new ProductsController(_mockRepository.Object, _mockLogger.Object, _mapper);
    }
    
    [Fact]
    public async Task GetProduct_WithValidId_ReturnsProduct()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var product = new Product { Id = productId, Name = "Test Product", Price = 100 };
        _mockRepository.Setup(repo => repo.GetByIdAsync(productId))
            .ReturnsAsync(product);
        
        // Act
        var result = await _controller.GetProduct(productId);
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result.Result);
        var response = Assert.IsType<ApiResponse<ProductDto>>(okResult.Value);
        Assert.True(response.Success);
        Assert.Equal("Test Product", response.Data.Name);
    }
    
    [Fact]
    public async Task CreateProduct_WithValidData_ReturnsCreatedProduct()
    {
        // Arrange
        var createDto = new CreateProductDto 
        { 
            Name = "New Product", 
            Code = "TEST001",
            Price = 100,
            StoreId = Guid.NewGuid()
        };
        
        _mockRepository.Setup(repo => repo.CodeExistsAsync(It.IsAny<Guid>(), It.IsAny<string>()))
            .ReturnsAsync(false);
        
        // Act
        var result = await _controller.CreateProduct(createDto);
        
        // Assert
        var createdResult = Assert.IsType<CreatedAtActionResult>(result.Result);
        var response = Assert.IsType<ApiResponse<ProductDto>>(createdResult.Value);
        Assert.True(response.Success);
        Assert.Equal("New Product", response.Data.Name);
    }
}
Step 7: Deployment Preparation
7.1 Dockerfile
dockerfile
# src/POSCloud.API/Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["POSCloud.API/POSCloud.API.csproj", "POSCloud.API/"]
COPY ["POSCloud.Core/POSCloud.Core.csproj", "POSCloud.Core/"]
COPY ["POSCloud.Infrastructure/POSCloud.Infrastructure.csproj", "POSCloud.Infrastructure/"]
RUN dotnet restore "POSCloud.API/POSCloud.API.csproj"
COPY . .
WORKDIR "/src/POSCloud.API"
RUN dotnet build "POSCloud.API.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "POSCloud.API.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "POSCloud.API.dll"]
7.2 docker-compose.yml
yaml
# deployment/docker-compose.yml
version: '3.8'

services:
  pos-api:
    build:
      context: .
      dockerfile: src/POSCloud.API/Dockerfile
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=pos_cloud;Username=pos_user;Password=pos_password
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: pos_cloud
      POSTGRES_USER: pos_user
      POSTGRES_PASSWORD: pos_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass redis_password
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
Next Steps
Setelah Phase 2 complete, lanjut ke:

Phase 3: SignalR Real-time Implementation

Phase 4: Desktop Application (WinForms)

Phase 5: Mobile Application (Flutter)

Backend API sekarang ready untuk development lebih lanjut! 🚀
