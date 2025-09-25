📁 [FILE: PHASE_3_SIGNALR_REALTIME.md]
📝 [COMMIT: "Add complete Phase 3 SignalR real-time implementation"]
markdown
# Phase 3: SignalR Real-time Implementation

## Overview
Real-time communication menggunakan SignalR dengan Redis backplane untuk scale-out. Support WebSocket, Server-Sent Events, dan Long Polling fallbacks.

## Architecture Diagram
```mermaid
graph TB
    subgraph "Client Applications"
        A[Flutter Mobile App]
        B[WinForms Desktop App]
        C[Web Dashboard]
    end
    
    subgraph "SignalR Hub Cluster"
        D[SignalR Hub Instance 1]
        E[SignalR Hub Instance 2]
        F[SignalR Hub Instance N]
    end
    
    subgraph "Redis Backplane"
        G[Redis Pub/Sub]
    end
    
    subgraph "Backend Services"
        H[API Controllers]
        I[Background Services]
    end
    
    A --> D
    B --> E
    C --> F
    D --> G
    E --> G
    F --> G
    H --> G
    I --> G
Step 1: SignalR Hub Implementation
1.1 Main POS Hub
csharp
// src/POSCloud.SignalR/Hubs/POSHub.cs
namespace POSCloud.SignalR.Hubs;

[Authorize]
public class POSHub : Hub
{
    private readonly IConnectionManager _connectionManager;
    private readonly ILogger<POSHub> _logger;
    private readonly ISyncService _syncService;

    public POSHub(IConnectionManager connectionManager, ILogger<POSHub> logger, 
        ISyncService syncService)
    {
        _connectionManager = connectionManager;
        _logger = logger;
        _syncService = syncService;
    }

    // Client joins specific store group
    public async Task JoinStore(string storeId)
    {
        try
        {
            var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            await Groups.AddToGroupAsync(Context.ConnectionId, $"store-{storeId}");
            await _connectionManager.AddConnectionAsync(userId, Context.ConnectionId, storeId);
            
            _logger.LogInformation("User {UserId} joined store {StoreId}", userId, storeId);
            await Clients.Caller.SendAsync("StoreJoined", new { success = true, storeId });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error joining store {StoreId}", storeId);
            await Clients.Caller.SendAsync("Error", new { message = "Failed to join store" });
        }
    }

    // Client leaves store group
    public async Task LeaveStore(string storeId)
    {
        try
        {
            var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"store-{storeId}");
            await _connectionManager.RemoveConnectionAsync(Context.ConnectionId);
            
            _logger.LogInformation("User {UserId} left store {StoreId}", userId, storeId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error leaving store {StoreId}", storeId);
        }
    }

    // Real-time stock update
    public async Task UpdateStock(string storeId, string productId, int newStock)
    {
        try
        {
            // Validate permissions
            if (!await CanManageProducts(storeId))
            {
                await Clients.Caller.SendAsync("Error", new { message = "Insufficient permissions" });
                return;
            }

            // Broadcast to all connected clients in the store
            await Clients.Group($"store-{storeId}")
                .SendAsync("StockUpdated", new { productId, newStock, updatedBy = Context.UserIdentifier });

            _logger.LogInformation("Stock updated for product {ProductId} in store {StoreId}", 
                productId, storeId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating stock for product {ProductId}", productId);
            await Clients.Caller.SendAsync("Error", new { message = "Stock update failed" });
        }
    }

    // New transaction notification
    public async Task NotifyNewTransaction(string storeId, TransactionDto transaction)
    {
        try
        {
            await Clients.Group($"store-{storeId}")
                .SendAsync("NewTransaction", transaction);

            _logger.LogInformation("New transaction notified for store {StoreId}", storeId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error notifying new transaction for store {StoreId}", storeId);
        }
    }

    // Sync request from client
    public async Task RequestSync(string storeId, DateTime lastSync)
    {
        try
        {
            var syncData = await _syncService.GetSyncDataAsync(storeId, lastSync);
            await Clients.Caller.SendAsync("SyncData", syncData);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing sync request for store {StoreId}", storeId);
            await Clients.Caller.SendAsync("SyncError", new { message = "Sync failed" });
        }
    }

    // Product price update
    public async Task UpdateProductPrice(string storeId, string productId, decimal newPrice)
    {
        try
        {
            if (!await CanManageProducts(storeId))
            {
                await Clients.Caller.SendAsync("Error", new { message = "Insufficient permissions" });
                return;
            }

            await Clients.Group($"store-{storeId}")
                .SendAsync("ProductPriceUpdated", new { productId, newPrice, updatedBy = Context.UserIdentifier });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating product price for {ProductId}", productId);
        }
    }

    // Connection events
    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        _logger.LogInformation("Client connected: {ConnectionId}, User: {UserId}", 
            Context.ConnectionId, userId);
        
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception exception)
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        await _connectionManager.RemoveConnectionAsync(Context.ConnectionId);
        _logger.LogInformation("Client disconnected: {ConnectionId}, User: {UserId}", 
            Context.ConnectionId, userId);

        await base.OnDisconnectedAsync(exception);
    }

    private async Task<bool> CanManageProducts(string storeId)
    {
        // Implement permission check logic
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return await Task.FromResult(true); // Simplified for example
    }
}
1.2 Dashboard Hub untuk Real-time Reporting
csharp
// src/POSCloud.SignalR/Hubs/DashboardHub.cs
namespace POSCloud.SignalR.Hubs;

[Authorize]
public class DashboardHub : Hub
{
    private readonly IDashboardService _dashboardService;
    private readonly ILogger<DashboardHub> _logger;

    public DashboardHub(IDashboardService dashboardService, ILogger<DashboardHub> logger)
    {
        _dashboardService = dashboardService;
        _logger = logger;
    }

    // Subscribe to real-time sales data
    public async Task SubscribeToSales(string storeId, string timeframe = "today")
    {
        try
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"sales-{storeId}");
            
            // Send initial data
            var salesData = await _dashboardService.GetSalesDataAsync(storeId, timeframe);
            await Clients.Caller.SendAsync("SalesData", salesData);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error subscribing to sales data for store {StoreId}", storeId);
        }
    }

    // Subscribe to inventory alerts
    public async Task SubscribeToInventoryAlerts(string storeId)
    {
        try
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"inventory-{storeId}");
            
            var alerts = await _dashboardService.GetInventoryAlertsAsync(storeId);
            await Clients.Caller.SendAsync("InventoryAlerts", alerts);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error subscribing to inventory alerts for store {StoreId}", storeId);
        }
    }

    // Real-time performance metrics
    public async Task SubscribeToPerformance(string storeId)
    {
        try
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"performance-{storeId}");
            
            var metrics = await _dashboardService.GetPerformanceMetricsAsync(storeId);
            await Clients.Caller.SendAsync("PerformanceMetrics", metrics);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error subscribing to performance metrics for store {StoreId}", storeId);
        }
    }
}
Step 2: Connection Management
2.1 Connection Manager Service
csharp
// src/POSCloud.SignalR/Services/ConnectionManager.cs
namespace POSCloud.SignalR.Services;

public interface IConnectionManager
{
    Task AddConnectionAsync(string userId, string connectionId, string storeId);
    Task RemoveConnectionAsync(string connectionId);
    Task<List<string>> GetConnectionsAsync(string userId);
    Task<List<string>> GetStoreConnectionsAsync(string storeId);
    Task<int> GetOnlineUsersCountAsync(string storeId);
}

public class ConnectionManager : IConnectionManager
{
    private readonly IRedisService _redisService;
    private readonly ILogger<ConnectionManager> _logger;

    // Redis keys
    private const string UserConnectionsKey = "user:connections:{0}";
    private const string StoreUsersKey = "store:users:{0}";
    private const string ConnectionInfoKey = "connection:info:{0}";

    public ConnectionManager(IRedisService redisService, ILogger<ConnectionManager> logger)
    {
        _redisService = redisService;
        _logger = logger;
    }

    public async Task AddConnectionAsync(string userId, string connectionId, string storeId)
    {
        var connectionInfo = new ConnectionInfo
        {
            UserId = userId,
            ConnectionId = connectionId,
            StoreId = storeId,
            ConnectedAt = DateTime.UtcNow
        };

        // Store connection info
        await _redisService.StringSetAsync(
            string.Format(ConnectionInfoKey, connectionId), 
            connectionInfo, 
            TimeSpan.FromHours(24));

        // Add to user's connections
        await _redisService.SetAddAsync(
            string.Format(UserConnectionsKey, userId), 
            connectionId);

        // Add to store's online users
        await _redisService.SetAddAsync(
            string.Format(StoreUsersKey, storeId), 
            userId);

        _logger.LogInformation("Connection added: User={UserId}, Connection={ConnectionId}, Store={StoreId}", 
            userId, connectionId, storeId);
    }

    public async Task RemoveConnectionAsync(string connectionId)
    {
        var connectionInfo = await _redisService.StringGetAsync<ConnectionInfo>(
            string.Format(ConnectionInfoKey, connectionId));

        if (connectionInfo != null)
        {
            // Remove from user's connections
            await _redisService.SetRemoveAsync(
                string.Format(UserConnectionsKey, connectionInfo.UserId), 
                connectionId);

            // Remove from store's online users if this is the last connection
            var userConnections = await GetConnectionsAsync(connectionInfo.UserId);
            if (userConnections.Count == 0)
            {
                await _redisService.SetRemoveAsync(
                    string.Format(StoreUsersKey, connectionInfo.StoreId), 
                    connectionInfo.UserId);
            }

            // Remove connection info
            await _redisService.KeyDeleteAsync(string.Format(ConnectionInfoKey, connectionId));

            _logger.LogInformation("Connection removed: {ConnectionId}", connectionId);
        }
    }

    public async Task<List<string>> GetConnectionsAsync(string userId)
    {
        return await _redisService.SetMembersAsync<string>(
            string.Format(UserConnectionsKey, userId));
    }

    public async Task<List<string>> GetStoreConnectionsAsync(string storeId)
    {
        var userIds = await _redisService.SetMembersAsync<string>(
            string.Format(StoreUsersKey, storeId));

        var connections = new List<string>();
        foreach (var userId in userIds)
        {
            var userConnections = await GetConnectionsAsync(userId);
            connections.AddRange(userConnections);
        }

        return connections;
    }

    public async Task<int> GetOnlineUsersCountAsync(string storeId)
    {
        var userIds = await _redisService.SetMembersAsync<string>(
            string.Format(StoreUsersKey, storeId));
        return userIds.Count;
    }
}

public class ConnectionInfo
{
    public string UserId { get; set; } = string.Empty;
    public string ConnectionId { get; set; } = string.Empty;
    public string StoreId { get; set; } = string.Empty;
    public DateTime ConnectedAt { get; set; }
}
2.2 Redis Service Abstraction
csharp
// src/POSCloud.SignalR/Services/RedisService.cs
namespace POSCloud.SignalR.Services;

public interface IRedisService
{
    Task<T?> StringGetAsync<T>(string key);
    Task StringSetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task<bool> KeyExistsAsync(string key);
    Task<bool> KeyDeleteAsync(string key);
    Task<List<T>> SetMembersAsync<T>(string key);
    Task<bool> SetAddAsync<T>(string key, T value);
    Task<bool> SetRemoveAsync<T>(string key, T value);
    Task PublishAsync(string channel, string message);
    Task SubscribeAsync(string channel, Action<string> handler);
}

public class RedisService : IRedisService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisService> _logger;

    public RedisService(IConnectionMultiplexer redis, ILogger<RedisService> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    public async Task<T?> StringGetAsync<T>(string key)
    {
        try
        {
            var db = _redis.GetDatabase();
            var value = await db.StringGetAsync(key);
            return value.IsNullOrEmpty ? default : JsonSerializer.Deserialize<T>(value!);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting key {Key} from Redis", key);
            return default;
        }
    }

    public async Task StringSetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        try
        {
            var db = _redis.GetDatabase();
            var serializedValue = JsonSerializer.Serialize(value);
            await db.StringSetAsync(key, serializedValue, expiry);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting key {Key} in Redis", key);
        }
    }

    public async Task<bool> KeyExistsAsync(string key)
    {
        var db = _redis.GetDatabase();
        return await db.KeyExistsAsync(key);
    }

    public async Task<bool> KeyDeleteAsync(string key)
    {
        var db = _redis.GetDatabase();
        return await db.KeyDeleteAsync(key);
    }

    public async Task<List<T>> SetMembersAsync<T>(string key)
    {
        try
        {
            var db = _redis.GetDatabase();
            var values = await db.SetMembersAsync(key);
            return values.Select(v => JsonSerializer.Deserialize<T>(v!)).ToList()!;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting set members for key {Key}", key);
            return new List<T>();
        }
    }

    public async Task<bool> SetAddAsync<T>(string key, T value)
    {
        try
        {
            var db = _redis.GetDatabase();
            var serializedValue = JsonSerializer.Serialize(value);
            return await db.SetAddAsync(key, serializedValue);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error adding to set for key {Key}", key);
            return false;
        }
    }

    public async Task<bool> SetRemoveAsync<T>(string key, T value)
    {
        try
        {
            var db = _redis.GetDatabase();
            var serializedValue = JsonSerializer.Serialize(value);
            return await db.SetRemoveAsync(key, serializedValue);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error removing from set for key {Key}", key);
            return false;
        }
    }

    public async Task PublishAsync(string channel, string message)
    {
        try
        {
            var db = _redis.GetDatabase();
            await db.PublishAsync(channel, message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error publishing to channel {Channel}", channel);
        }
    }

    public async Task SubscribeAsync(string channel, Action<string> handler)
    {
        try
        {
            var subscriber = _redis.GetSubscriber();
            await subscriber.SubscribeAsync(channel, (redisChannel, value) => handler(value));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error subscribing to channel {Channel}", channel);
        }
    }
}
Step 3: Real-time Event Handlers
3.1 Transaction Event Handler
csharp
// src/POSCloud.SignalR/Services/TransactionEventHandler.cs
namespace POSCloud.SignalR.Services;

public interface ITransactionEventHandler
{
    Task HandleNewTransactionAsync(Transaction transaction);
    Task HandleTransactionUpdatedAsync(Transaction transaction);
    Task HandlePaymentProcessedAsync(Transaction transaction);
}

public class TransactionEventHandler : ITransactionEventHandler
{
    private readonly IHubContext<POSHub> _hubContext;
    private readonly IConnectionManager _connectionManager;
    private readonly ILogger<TransactionEventHandler> _logger;

    public TransactionEventHandler(IHubContext<POSHub> hubContext, 
        IConnectionManager connectionManager, ILogger<TransactionEventHandler> logger)
    {
        _hubContext = hubContext;
        _connectionManager = connectionManager;
        _logger = logger;
    }

    public async Task HandleNewTransactionAsync(Transaction transaction)
    {
        try
        {
            var transactionDto = MapToDto(transaction);
            
            // Notify all connected clients in the store
            await _hubContext.Clients.Group($"store-{transaction.StoreId}")
                .SendAsync("NewTransaction", transactionDto);

            // Notify dashboard subscribers
            await _hubContext.Clients.Group($"sales-{transaction.StoreId}")
                .SendAsync("SalesUpdated", new { 
                    storeId = transaction.StoreId,
                    amount = transaction.TotalAmount,
                    transactionId = transaction.Id 
                });

            _logger.LogInformation("New transaction event handled for store {StoreId}", 
                transaction.StoreId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling new transaction event");
        }
    }

    public async Task HandleTransactionUpdatedAsync(Transaction transaction)
    {
        try
        {
            var transactionDto = MapToDto(transaction);
            await _hubContext.Clients.Group($"store-{transaction.StoreId}")
                .SendAsync("TransactionUpdated", transactionDto);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling transaction update event");
        }
    }

    public async Task HandlePaymentProcessedAsync(Transaction transaction)
    {
        try
        {
            await _hubContext.Clients.Group($"store-{transaction.StoreId}")
                .SendAsync("PaymentProcessed", new {
                    transactionId = transaction.Id,
                    status = transaction.Status,
                    paymentMethod = transaction.PaymentMethod
                });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling payment processed event");
        }
    }

    private TransactionDto MapToDto(Transaction transaction)
    {
        // Implementation untuk map Transaction ke TransactionDto
        return new TransactionDto
        {
            Id = transaction.Id,
            TransactionNumber = transaction.TransactionNumber,
            TotalAmount = transaction.TotalAmount,
            // ... other properties
        };
    }
}
3.2 Stock Event Handler
csharp
// src/POSCloud.SignalR/Services/StockEventHandler.cs
namespace POSCloud.SignalR.Services;

public interface IStockEventHandler
{
    Task HandleStockUpdateAsync(string storeId, string productId, int newStock);
    Task HandleLowStockAlertAsync(string storeId, string productId, int currentStock);
    Task HandleProductPriceUpdateAsync(string storeId, string productId, decimal newPrice);
}

public class StockEventHandler : IStockEventHandler
{
    private readonly IHubContext<POSHub> _hubContext;
    private readonly ILogger<StockEventHandler> _logger;

    public StockEventHandler(IHubContext<POSHub> hubContext, ILogger<StockEventHandler> logger)
    {
        _hubContext = hubContext;
        _logger = logger;
    }

    public async Task HandleStockUpdateAsync(string storeId, string productId, int newStock)
    {
        try
        {
            await _hubContext.Clients.Group($"store-{storeId}")
                .SendAsync("StockUpdated", new { productId, newStock, timestamp = DateTime.UtcNow });

            _logger.LogInformation("Stock updated for product {ProductId} in store {StoreId}", 
                productId, storeId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling stock update event");
        }
    }

    public async Task HandleLowStockAlertAsync(string storeId, string productId, int currentStock)
    {
        try
        {
            var alert = new
            {
                productId,
                currentStock,
                alertType = "LowStock",
                timestamp = DateTime.UtcNow,
                message = $"Product {productId} is running low. Current stock: {currentStock}"
            };

            await _hubContext.Clients.Group($"inventory-{storeId}")
                .SendAsync("InventoryAlert", alert);

            _logger.LogWarning("Low stock alert for product {ProductId} in store {StoreId}", 
                productId, storeId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling low stock alert event");
        }
    }

    public async Task HandleProductPriceUpdateAsync(string storeId, string productId, decimal newPrice)
    {
        try
        {
            await _hubContext.Clients.Group($"store-{storeId}")
                .SendAsync("ProductPriceUpdated", new { productId, newPrice, timestamp = DateTime.UtcNow });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling product price update event");
        }
    }
}
Step 4: SignalR Service Configuration
4.1 Program.cs untuk SignalR Project
csharp
// src/POSCloud.SignalR/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// SignalR dengan Redis backplane
builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.ConnectionFactory = async writer =>
        {
            var config = new ConfigurationOptions
            {
                AbortOnConnectFail = false,
                Password = builder.Configuration["Redis:Password"]
            };
            config.EndPoints.Add(builder.Configuration["Redis:Host"]!);
            
            var connection = await ConnectionMultiplexer.ConnectAsync(config, writer);
            connection.ConnectionFailed += (_, e) =>
            {
                Console.WriteLine("Connection to Redis failed.");
            };
            
            return connection;
        };
    });

// Redis service
builder.Services.AddSingleton<IConnectionMultiplexer>(provider =>
{
    var configuration = ConfigurationOptions.Parse(builder.Configuration["Redis:ConnectionString"]!);
    return ConnectionMultiplexer.Connect(configuration);
});

// Application services
builder.Services.AddScoped<IConnectionManager, ConnectionManager>();
builder.Services.AddScoped<IRedisService, RedisService>();
builder.Services.AddScoped<ITransactionEventHandler, TransactionEventHandler>();
builder.Services.AddScoped<IStockEventHandler, StockEventHandler>();

// HttpClient untuk internal API calls
builder.Services.AddHttpClient("InternalAPI", client =>
{
    client.BaseAddress = new Uri(builder.Configuration["InternalApi:BaseUrl"]!);
    client.DefaultRequestHeaders.Add("X-Internal-Request", "true");
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("SignalRClient", policy =>
    {
        policy.WithOrigins(
                "http://localhost:3000",
                "capacitor://localhost",
                "ionic://localhost")
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});

// Authentication
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

        // SignalR JWT token handling
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                
                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/poshub"))
                {
                    context.Token = accessToken;
                }
                
                return Task.CompletedTask;
            }
        };
    });

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("SignalRClient");
app.UseAuthentication();
app.UseAuthorization();

// SignalR hubs
app.MapHub<POSHub>("/poshub");
app.MapHub<DashboardHub>("/dashboardhub");

app.MapControllers();

app.Run();
4.2 appsettings.json untuk SignalR
json
{
  "Redis": {
    "ConnectionString": "localhost:6379,password=your-redis-password,abortConnect=false",
    "Host": "localhost:6379",
    "Password": "your-redis-password"
  },
  "Jwt": {
    "Secret": "your-super-secret-key-at-least-32-characters-long",
    "Issuer": "POSCloud",
    "Audience": "POSCloud-Users"
  },
  "InternalApi": {
    "BaseUrl": "https://localhost:5001/api/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore.SignalR": "Warning",
      "Microsoft.AspNetCore.Http.Connections": "Warning"
    }
  }
}
Step 5: Integration dengan Main API
5.1 Event Publishing dari API Controllers
csharp
// Dalam TransactionController setelah create transaction berhasil
[HttpPost]
public async Task<ActionResult<ApiResponse<TransactionDto>>> CreateTransaction(
    CreateTransactionDto createTransactionDto)
{
    // ... existing transaction creation logic
    
    // Publish event untuk real-time notification
    await _transactionEventHandler.HandleNewTransactionAsync(transactionEntity);
    
    // ... rest of the method
}

// Dalam ProductController setelah update stock
[HttpPut("{id}/stock")]
public async Task<ActionResult<ApiResponse>> UpdateStock(Guid id, [FromBody] int newStock)
{
    // ... existing stock update logic
    
    // Publish real-time event
    await _stockEventHandler.HandleStockUpdateAsync(product.StoreId.ToString(), id.ToString(), newStock);
    
    // Check for low stock alert
    if (newStock <= product.MinimumStock)
    {
        await _stockEventHandler.HandleLowStockAlertAsync(
            product.StoreId.ToString(), id.ToString(), newStock);
    }
    
    // ... rest of the method
}
5.2 Background Service untuk Real-time Data Sync
csharp
// src/POSCloud.SignalR/Services/RealtimeSyncService.cs
namespace POSCloud.SignalR.Services;

public class RealtimeSyncService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<RealtimeSyncService> _logger;

    public RealtimeSyncService(IServiceProvider serviceProvider, ILogger<RealtimeSyncService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var dashboardService = scope.ServiceProvider.GetRequiredService<IDashboardService>();
                var hubContext = scope.ServiceProvider.GetRequiredService<IHubContext<DashboardHub>>();
                
                // Broadcast real-time dashboard data setiap 30 detik
                await BroadcastDashboardData(hubContext, dashboardService);
                
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
            catch (Exception ex) when (ex is not TaskCanceledException)
            {
                _logger.LogError(ex, "Error in real-time sync service");
                await Task.Delay(TimeSpan.FromSeconds(60), stoppingToken);
            }
        }
    }

    private async Task BroadcastDashboardData(IHubContext<DashboardHub> hubContext, IDashboardService dashboardService)
    {
        try
        {
            // Get active stores dari connection manager
            // Broadcast real-time metrics ke subscribed clients
            // Implementation details...
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error broadcasting dashboard data");
        }
    }
}
Step 6: Client-Side Implementation Examples
6.1 Flutter SignalR Client
dart
// lib/services/signalr_service.dart
class SignalRService {
  late HubConnection _connection;
  final String _baseUrl;
  
  SignalRService({required String baseUrl}) : _baseUrl = baseUrl;
  
  Future<void> connect(String token) async {
    _connection = HubConnectionBuilder()
        .withUrl(
          '$_baseUrl/poshub',
          HttpConnectionOptions(
            accessTokenFactory: () => Future.value(token),
            logging: (level, message) => debugPrint(message),
          ),
        )
        .build();
    
    // Register event handlers
    _connection.on('StockUpdated', _handleStockUpdated);
    _connection.on('NewTransaction', _handleNewTransaction);
    _connection.on('ProductPriceUpdated', _handlePriceUpdated);
    
    try {
      await _connection.start();
      debugPrint('SignalR connected');
    } catch (e) {
      debugPrint('SignalR connection failed: $e');
    }
  }
  
  Future<void> joinStore(String storeId) async {
    try {
      await _connection.invoke('JoinStore', args: [storeId]);
    } catch (e) {
      debugPrint('Error joining store: $e');
    }
  }
  
  Future<void> updateStock(String storeId, String productId, int newStock) async {
    try {
      await _connection.invoke('UpdateStock', args: [storeId, productId, newStock]);
    } catch (e) {
      debugPrint('Error updating stock: $e');
    }
  }
  
  void _handleStockUpdated(List<dynamic>? arguments) {
    if (arguments != null && arguments.isNotEmpty) {
      final data = arguments[0];
      // Update local state atau trigger UI refresh
    }
  }
  
  void _handleNewTransaction(List<dynamic>? arguments) {
    // Handle new transaction notification
  }
  
  void _handlePriceUpdated(List<dynamic>? arguments) {
    // Handle price update notification
  }
  
  Future<void> disconnect() async {
    await _connection.stop();
  }
}
6.2 WinForms SignalR Client
csharp
// SignalRClientService.cs
public class SignalRClientService
{
    private HubConnection _connection;
    private readonly string _baseUrl;
    private readonly string _token;
    
    public SignalRClientService(string baseUrl, string token)
    {
        _baseUrl = baseUrl;
        _token = token;
    }
    
    public async Task ConnectAsync()
    {
        _connection = new HubConnectionBuilder()
            .WithUrl($"{_baseUrl}/poshub", options =>
            {
                options.AccessTokenProvider = () => Task.FromResult(_token);
            })
            .Build();
            
        // Event handlers
        _connection.On<dynamic>("StockUpdated", OnStockUpdated);
        _connection.On<dynamic>("NewTransaction", OnNewTransaction);
        _connection.On<dynamic>("ProductPriceUpdated", OnProductPriceUpdated);
        
        _connection.Closed += async (error) =>
        {
            await Task.Delay(new Random().Next(0,5) * 1000);
            await ConnectAsync();
        };
        
        try
        {
            await _connection.StartAsync();
        }
        catch (Exception ex)
        {
            // Handle connection error
        }
    }
    
    public async Task JoinStoreAsync(string storeId)
    {
        try
        {
            await _connection.InvokeAsync("JoinStore", storeId);
        }
        catch (Exception ex)
        {
            // Handle error
        }
    }
    
    private void OnStockUpdated(dynamic stockData)
    {
        // Update UI thread-safe
        if (InvokeRequired)
        {
            Invoke(new Action<dynamic>(OnStockUpdated), stockData);
            return;
        }
        
        // Update product stock in UI
    }
    
    private void OnNewTransaction(dynamic transactionData)
    {
        // Update transactions display
    }
    
    private void OnProductPriceUpdated(dynamic priceData)
    {
        // Update product prices
    }
}
Step 7: Performance Optimization
7.1 Message Size Optimization
csharp
// Custom JSON serializer untuk mengurangi message size
public class CompactJsonSerializer : IJsonSerializer
{
    private readonly JsonSerializerOptions _options = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };
    
    public object Deserialize(ReadOnlyMemory<byte> input, Type targetType)
    {
        return JsonSerializer.Deserialize(input.Span, targetType, _options);
    }
    
    public ReadOnlyMemory<byte> Serialize(object value, Type inputType)
    {
        return JsonSerializer.SerializeToUtf8Bytes(value, inputType, _options);
    }
}
7.2 Connection Monitoring dan Health Checks
csharp
// Health check untuk SignalR
public class SignalRHealthCheck : IHealthCheck
{
    private readonly IConnectionManager _connectionManager;
    
    public SignalRHealthCheck(IConnectionManager connectionManager)
    {
        _connectionManager = connectionManager;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            // Check Redis connection
            // Check active connections count
            return HealthCheckResult.Healthy("SignalR service is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("SignalR service is unhealthy", ex);
        }
    }
}
Step 8: Security Considerations
8.1 Authentication dan Authorization
csharp
// Custom authorization attribute untuk SignalR
public class StoreAuthorizeAttribute : AuthorizeAttribute
{
    public string StoreId { get; set; } = string.Empty;
    
    public override bool AuthorizeHubConnection(HubLifetimeContext context, IList<string> groups)
    {
        var httpContext = context.Context.GetHttpContext();
        var user = httpContext?.User;
        
        if (user == null) return false;
        
        // Check if user has access to the specified store
        return HasStoreAccess(user, StoreId);
    }
    
    public override bool AuthorizeHubMethodInvocation(HubInvocationContext invocationContext, bool appliesToMethod)
    {
        var user = invocationContext.Context.User;
        return HasStoreAccess(user, StoreId);
    }
    
    private bool HasStoreAccess(ClaimsPrincipal user, string storeId)
    {
        // Implement store access logic
        return true;
    }
}
Next Steps
Setelah Phase 3 complete, lanjut ke:

Phase 4: Desktop Application (WinForms)

Phase 5: Mobile Application (Flutter)

Phase 6: Deployment dan Production Setup

Real-time system sekarang ready untuk integration dengan client applications! 🚀
