📁 [FILE: TROUBLESHOOTING_GUIDE.md]
📝 [COMMIT: "Add comprehensive troubleshooting guide for POS Hybrid System"]
markdown
# POS Hybrid System - Troubleshooting Guide

## 🚨 Emergency Quick Reference

### Critical Issues - Immediate Action Required
| Issue | Symptoms | Immediate Action |
|-------|----------|------------------|
| Database not responding | API errors, sync failures | Restart PostgreSQL, check disk space |
| SignalR connections dropping | Real-time updates not working | Check Redis service, restart SignalR |
| Payment gateway timeout | Transactions failing | Switch to cash payments, check gateway status |
| High server CPU usage | Slow response, timeouts | Scale up resources, identify bottleneck |

### Emergency Contacts & Resources
- **Server Access**: SSH port 2222, user `deployer`
- **Database**: PostgreSQL port 5432, user `pos_user`
- **Monitoring**: Grafana http://your-domain.com:3000
- **Logs Directory**: `/var/log/pos-cloud/`
- **Backup Location**: `/backup/`

## 🔧 Infrastructure Issues

### VPS & Server Problems

#### Issue: Cannot SSH to Server
**Symptoms**: Connection timeout, connection refused
```bash
# Diagnosis Steps:
1. Check if server is running (ping your-domain.com)
2. Verify SSH port (2222 instead of default 22)
3. Check firewall rules: sudo firewall-cmd --list-all
4. Verify SSH service: sudo systemctl status sshd

# Solutions:
# If firewall blocking:
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload

# If SSH service down:
sudo systemctl start sshd
sudo systemctl enable sshd

# If key authentication issues:
ssh-keygen -R your-domain.com  # Clear known hosts
ssh -o StrictHostKeyChecking=no -p 2222 deployer@your-domain.com
Issue: High CPU/Memory Usage
Symptoms: Slow performance, timeouts, system instability

bash
# Diagnosis:
htop                          # Real-time system monitoring
top                          # Process monitoring
free -h                      # Memory usage
df -h                        # Disk space check

# Identify problematic processes:
ps aux --sort=-%cpu | head -10    # Top CPU processes
ps aux --sort=-%mem | head -10    # Top memory processes

# Solutions:
# If .NET applications using high CPU:
sudo systemctl restart pos-api
sudo systemctl restart pos-signalr

# If database issues:
sudo -u postgres psql -c "SELECT pid, query_start, state, query FROM pg_stat_activity WHERE state = 'active';"

# Clear memory cache:
echo 3 > /proc/sys/vm/drop_caches
Issue: Disk Space Full
Symptoms: Write errors, service crashes, backup failures

bash
# Diagnosis:
df -h                        # Check disk usage
du -sh /var/log/*            # Check log directory sizes
du -sh /var/lib/postgresql/* # Check database size

# Solutions:
# Clean up log files:
sudo journalctl --vacuum-time=7d  # Keep only last 7 days
sudo find /var/log -name "*.log" -type f -mtime +7 -delete

# Clear package cache:
sudo dnf clean all

# Check and rotate large logs:
sudo logrotate -f /etc/logrotate.conf
Database Issues
Issue: PostgreSQL Connection Errors
Symptoms: "Connection refused", "too many connections", timeouts

bash
# Diagnosis:
sudo systemctl status postgresql-15
sudo tail -f /var/lib/pgsql/15/data/log/postgresql-*.log
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_activity;"

# Solutions:
# Restart PostgreSQL:
sudo systemctl restart postgresql-15

# Check connection limits:
sudo -u postgres psql -c "SHOW max_connections;"

# Kill idle connections:
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND pid <> pg_backend_pid();"

# If too many connections, use PgBouncer:
sudo systemctl restart pgbouncer
Issue: Slow Database Queries
Symptoms: High API response times, transaction delays

sql
-- Diagnosis: Identify slow queries
SELECT query, calls, total_time, mean_time, rows 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- Check for missing indexes
SELECT schemaname, tablename, indexname, indexdef 
FROM pg_indexes 
WHERE schemaname = 'public';

-- Solutions: Create missing indexes
CREATE INDEX CONCURRENTLY idx_products_store_active 
ON products(store_id, is_active) WHERE is_active = true;

CREATE INDEX CONCURRENTLY idx_transactions_date 
ON transactions(transaction_date DESC);

-- Update statistics
ANALYZE VERBOSE;
Issue: Database Corruption or Crash
Symptoms: PostgreSQL won't start, data inconsistency errors

bash
# Diagnosis:
sudo tail -f /var/lib/pgsql/15/data/log/postgresql-*.log
sudo -u postgres psql -l      # Check if database accessible

# Solutions:
# Try to start in recovery mode:
sudo systemctl stop postgresql-15
sudo -u postgres pg_ctl -D /var/lib/pgsql/15/data start

# If corruption suspected:
sudo -u postgres psql -c "CHECKPOINT;"
sudo -u postgres psql -c "VACUUM FULL ANALYZE;"

# Restore from backup if needed:
sudo systemctl stop postgresql-15
sudo -u postgres psql -c "DROP DATABASE pos_cloud;"
sudo -u postgres psql -c "CREATE DATABASE pos_cloud;"
pg_restore -h localhost -U pos_user -d pos_cloud /backup/latest.dump
Network & Connectivity Issues
Issue: SSL Certificate Problems
Symptoms: "Certificate expired", "SSL handshake failed"

bash
# Diagnosis:
openssl s_client -connect your-domain.com:443
sudo certbot certificates

# Solutions:
# Renew certificate:
sudo certbot renew --dry-run
sudo certbot renew

# Force renewal if expired:
sudo certbot renew --force-renewal

# Check Nginx configuration:
sudo nginx -t
sudo systemctl reload nginx
Issue: DNS Resolution Problems
Symptoms: "Unable to resolve host", API calls failing

bash
# Diagnosis:
nslookup your-domain.com
dig your-domain.com
ping your-domain.com

# Solutions:
# Flush DNS cache (on client):
sudo systemd-resolve --flush-caches  # Linux
ipconfig /flushdns                   # Windows

# Check /etc/hosts for incorrect entries:
sudo nano /etc/hosts
🚀 Application Issues
Backend API Issues
Issue: API Not Responding
Symptoms: HTTP 500 errors, timeout, "Service Unavailable"

bash
# Diagnosis:
sudo systemctl status pos-api
sudo journalctl -u pos-api -f --lines=50
curl -v http://localhost:5000/health

# Check application logs:
tail -f /var/log/pos-cloud/app.log

# Solutions:
# Restart API service:
sudo systemctl restart pos-api

# Check database connection:
sudo -u postgres psql -d pos_cloud -c "SELECT 1;"

# Check environment variables:
sudo cat /var/www/pos-cloud/appsettings.Production.json
Issue: SignalR Connection Failures
Symptoms: Real-time updates not working, WebSocket errors

bash
# Diagnosis:
sudo systemctl status pos-signalr
sudo journalctl -u pos-signalr -f
sudo systemctl status redis

# Test Redis connection:
redis-cli -a your_password ping

# Check SignalR hub:
curl http://localhost:5001/poshub/negotiate

# Solutions:
# Restart services:
sudo systemctl restart redis
sudo systemctl restart pos-signalr

# Check Redis memory:
redis-cli -a your_password info memory
Issue: Payment Gateway Integration Failures
Symptoms: Payment timeouts, "Payment failed" errors

csharp
// Diagnosis: Check payment gateway logs
// Enable detailed logging in appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "POSCloud.Infrastructure.Services.PaymentGatewayService": "Debug"
    }
  }
}

// Solutions:
// Implement retry logic:
public async Task<PaymentResponse> ProcessPaymentWithRetry(PaymentRequest request, int maxRetries = 3)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await _paymentGateway.ProcessPaymentAsync(request);
        }
        catch (Exception ex) when (i < maxRetries - 1)
        {
            _logger.LogWarning($"Payment attempt {i + 1} failed: {ex.Message}");
            await Task.Delay(1000 * (i + 1)); // Exponential backoff
        }
    }
    throw new PaymentException("All payment attempts failed");
}

// Fallback to cash payment:
if (paymentMethod != PaymentMethod.Cash)
{
    try
    {
        return await ProcessPaymentWithRetry(request);
    }
    catch
    {
        // Fallback to cash
        transaction.PaymentMethod = PaymentMethod.Cash;
        await _transactionRepository.UpdateAsync(transaction);
    }
}
Desktop Application Issues
Issue: WinForms App Won't Start
Symptoms: Application crash on startup, missing dependencies

bash
# Diagnosis:
# Check Windows Event Viewer for .NET errors
# Verify .NET 8 runtime is installed:
dotnet --list-runtimes

# Check application logs:
# Located in %APPDATA%/POSDesktop/logs/

# Solutions:
# Reinstall .NET 8 Runtime:
# Download from https://dotnet.microsoft.com/download/dotnet/8.0

# Clear application cache:
rm -rf %APPDATA%/POSDesktop/cache/

# Repair installation:
# Re-run the installer or restore from backup
Issue: Local Database Connection Problems
Symptoms: "Cannot connect to database", sync failures

csharp
// Diagnosis: Check connection string
// In appsettings.json or configuration file
"ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=pos_desktop;Username=postgres;Password=password"
}

// Solutions:
// Verify PostgreSQL service is running:
// Services.msc -> Check "postgresql-x64-15"

// Test connection:
try
{
    using var connection = new NpgsqlConnection(connectionString);
    await connection.OpenAsync();
    // Connection successful
}
catch (Exception ex)
{
    _logger.LogError($"Database connection failed: {ex.Message}");
    // Show user-friendly message
}

// Repair database:
// Run database migration tool included with application
Issue: Hardware Integration Failures
Symptoms: Barcode scanner not working, receipt printer errors

csharp
// Diagnosis: Check device connectivity
// Barcode Scanner:
try
{
    _serialPort = new SerialPort("COM3", 9600, Parity.None, 8, StopBits.One);
    _serialPort.Open();
    _serialPort.DataReceived += OnBarcodeScanned;
}
catch (Exception ex)
{
    _logger.LogError($"Barcode scanner initialization failed: {ex.Message}");
    // Try alternative COM ports
}

// Solutions:
// Implement device auto-detection:
public string DetectBarcodeScannerPort()
{
    var ports = SerialPort.GetPortNames();
    foreach (var port in ports)
    {
        try
        {
            using var testPort = new SerialPort(port, 9600);
            testPort.Open();
            testPort.Close();
            return port; // Port is available
        }
        catch
        {
            continue;
        }
    }
    return null;
}

// Receipt Printer fallback:
public async Task<bool> PrintReceiptWithFallback(Receipt receipt)
{
    try
    {
        await _receiptPrinter.PrintAsync(receipt);
        return true;
    }
    catch (Exception ex)
    {
        _logger.LogWarning($"Primary printer failed: {ex.Message}");
        
        // Try secondary printer or save for later
        await SaveReceiptForLaterPrinting(receipt);
        return false;
    }
}
Mobile Application Issues
Issue: Flutter App Crashes on Startup
Symptoms: App immediately closes, white screen, initialization errors

dart
// Diagnosis: Check Flutter logs
flutter logs
# Or use Android Studio Logcat

// Check for missing permissions (Android):
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.WAKE_LOCK" />

// Solutions:
// Clear app data and cache:
// Android: Settings -> Apps -> POS Mobile -> Storage -> Clear Data
// iOS: Delete and reinstall app

// Rebuild application:
flutter clean
flutter pub get
flutter build apk --release
Issue: API Calls Failing in Mobile App
Symptoms: Network errors, timeouts, "Unable to connect"

dart
// Diagnosis: Check network connectivity
final connectivityResult = await Connectivity().checkConnectivity();
if (connectivityResult == ConnectivityResult.none) {
  showOfflineModeDialog();
  return;
}

// Check API base URL:
const String apiBaseUrl = 'https://your-domain.com/api';

// Solutions:
// Implement retry logic with exponential backoff:
class RetryHttpClient {
  final Dio _dio = Dio();
  
  Future<Response> requestWithRetry(
    String path, {
    int maxRetries = 3,
    Duration initialDelay = const Duration(seconds: 1),
  }) async {
    for (int i = 0; i < maxRetries; i++) {
      try {
        return await _dio.get('$apiBaseUrl/$path');
      } catch (e) {
        if (i == maxRetries - 1) rethrow;
        await Future.delay(initialDelay * (i + 1));
      }
    }
    throw Exception('All retry attempts failed');
  }
}

// Implement offline mode:
class OfflineProductRepository {
  final ProductRepository onlineRepository;
  final ProductLocalDataSource localDataSource;
  
  Future<List<Product>> getProducts() async {
    if (await Connectivity().checkConnectivity() != ConnectivityResult.none) {
      try {
        final products = await onlineRepository.getProducts();
        await localDataSource.cacheProducts(products);
        return products;
      } catch (e) {
        // Fallback to local data
        return await localDataSource.getProducts();
      }
    } else {
      return await localDataSource.getProducts();
    }
  }
}
Issue: SignalR Connection Drops Frequently
Symptoms: Real-time updates stop working, connection errors

dart
// Diagnosis: Monitor connection state
_hubConnection.onclose((error) {
  _logger.warning('SignalR connection closed: $error');
  _reconnect();
});

// Solutions:
// Implement automatic reconnection:
void _reconnect() async {
  int retryCount = 0;
  const maxRetries = 5;
  
  while (retryCount < maxRetries) {
    try {
      await Future.delayed(Duration(seconds: pow(2, retryCount).toInt()));
      await _hubConnection.start();
      _logger.info('SignalR reconnected successfully');
      return;
    } catch (e) {
      retryCount++;
      _logger.warning('Reconnection attempt $retryCount failed: $e');
    }
  }
  _logger.error('Max reconnection attempts reached');
}

// Implement connection state management:
class ConnectionManager {
  final SignalRService _signalRService;
  Timer? _healthCheckTimer;
  
  void startHealthChecks() {
    _healthCheckTimer = Timer.periodic(Duration(seconds: 30), (_) {
      if (_signalRService.connectionState != HubConnectionState.connected) {
        _signalRService.reconnect();
      }
    });
  }
}
🔄 Sync & Data Issues
Issue: Data Sync Conflicts
Symptoms: Data inconsistencies between devices, merge conflicts

csharp
// Diagnosis: Check sync logs and conflict records
// Implement conflict detection:
public class SyncConflictDetector
{
    public List<SyncConflict> DetectConflicts(SyncRequest local, SyncRequest remote)
    {
        var conflicts = new List<SyncConflict>();
        
        // Check for update conflicts
        foreach (var localProduct in local.Products)
        {
            var remoteProduct = remote.Products.FirstOrDefault(p => p.Id == localProduct.Id);
            if (remoteProduct != null && remoteProduct.UpdatedAt > localProduct.UpdatedAt)
            {
                conflicts.Add(new SyncConflict
                {
                    EntityType = nameof(Product),
                    EntityId = localProduct.Id,
                    LocalVersion = localProduct.UpdatedAt,
                    RemoteVersion = remoteProduct.UpdatedAt,
                    Resolution = ConflictResolution.UseRemote
                });
            }
        }
        return conflicts;
    }
}

// Solutions: Implement conflict resolution strategies
public enum ConflictResolution
{
    UseLocal,
    UseRemote,
    Merge,
    Manual
}

public async Task ResolveSyncConflicts(List<SyncConflict> conflicts)
{
    foreach (var conflict in conflicts)
    {
        switch (conflict.Resolution)
        {
            case ConflictResolution.UseRemote:
                await _localRepository.UpdateFromRemote(conflict.EntityId);
                break;
            case ConflictResolution.UseLocal:
                await _remoteRepository.UpdateFromLocal(conflict.EntityId);
                break;
            case ConflictResolution.Merge:
                await MergeEntities(conflict.EntityId);
                break;
            case ConflictResolution.Manual:
                await QueueForManualResolution(conflict);
                break;
        }
    }
}
Issue: Offline Data Loss
Symptoms: Data entered offline not syncing, missing transactions

csharp
// Diagnosis: Check offline queue and sync status
public class OfflineTransactionQueue
{
    public async Task QueueTransactionForSync(Transaction transaction)
    {
        // Mark as pending sync
        transaction.IsSynced = false;
        transaction.SyncedAt = null;
        
        await _localRepository.AddAsync(transaction);
        await _syncService.QueueForSync(transaction);
    }
    
    public async Task ProcessPendingSyncs()
    {
        var pendingTransactions = await _localRepository.GetPendingSyncsAsync();
        
        foreach (var transaction in pendingTransactions)
        {
            try
            {
                await _remoteRepository.SyncTransactionAsync(transaction);
                transaction.IsSynced = true;
                transaction.SyncedAt = DateTime.UtcNow;
                await _localRepository.UpdateAsync(transaction);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Failed to sync transaction {transaction.Id}: {ex.Message}");
                // Retry later
            }
        }
    }
}

// Solutions: Implement robust offline storage
public class RobustLocalStorage
{
    public async Task SaveWithBackup<T>(T entity) where T : BaseEntity
    {
        // Primary storage
        await _primaryRepository.SaveAsync(entity);
        
        // Backup storage
        await _backupRepository.SaveAsync(entity);
        
        // Transaction log
        await _transactionLog.LogOperationAsync("SAVE", entity);
    }
}
📊 Performance Issues
Issue: Slow Database Queries
Symptoms: High response times, timeouts under load

sql
-- Diagnosis: Identify slow queries
SELECT query, calls, total_time, mean_time, rows 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- Solutions: Query optimization
-- Add appropriate indexes:
CREATE INDEX CONCURRENTLY idx_products_search 
ON products USING gin(to_tsvector('english', name || ' ' || code));

CREATE INDEX CONCURRENTLY idx_transactions_date_store 
ON transactions(store_id, transaction_date DESC);

-- Optimize expensive queries:
EXPLAIN ANALYZE 
SELECT * FROM transactions 
WHERE store_id = 'store-uuid' 
AND transaction_date >= NOW() - INTERVAL '30 days'
ORDER BY transaction_date DESC 
LIMIT 100;
Issue: Memory Leaks in Applications
Symptoms: Increasing memory usage, application slowdowns, crashes

csharp
// Diagnosis: Use memory profiling tools
// .NET: dotnet-counters, dotnet-dump
// Flutter: DevTools memory profiler

// Common memory leak patterns:

// 1. Event handlers not unsubscribed:
public class ProductService : IDisposable
{
    private readonly List<IDisposable> _subscriptions = new();
    
    public void Initialize()
    {
        _subscriptions.Add(_eventBus.Subscribe<ProductUpdated>(OnProductUpdated));
    }
    
    public void Dispose()
    {
        foreach (var subscription in _subscriptions)
        {
            subscription.Dispose();
        }
    }
}

// 2. Large object retention:
public class CacheService
{
    private readonly MemoryCache _cache = new();
    private readonly TimeSpan _cacheDuration = TimeSpan.FromHours(1);
    
    public void AddToCache(string key, object value)
    {
        _cache.Set(key, value, _cacheDuration);
    }
    
    public void ClearExpiredCache()
    {
        // Regularly clear expired entries
        _cache.Compact(0.5); // Remove 50% of expired entries
    }
}
🔒 Security Issues
Issue: Authentication/Authorization Failures
Symptoms: "Access denied", "Invalid token", login failures

csharp
// Diagnosis: Check JWT token validation
public class JwtTokenValidator
{
    public bool ValidateToken(string token)
    {
        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = _configuration["Jwt:Issuer"],
                ValidAudience = _configuration["Jwt:Audience"],
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(_configuration["Jwt:Secret"]))
            };
            
            tokenHandler.ValidateToken(token, validationParameters, out _);
            return true;
        }
        catch
        {
            return false;
        }
    }
}

// Solutions: Implement token refresh
public class TokenRefreshService
{
    public async Task<string> RefreshTokenAsync(string expiredToken, string refreshToken)
    {
        // Validate refresh token
        if (!await ValidateRefreshTokenAsync(refreshToken))
        {
            throw new SecurityException("Invalid refresh token");
        }
        
        // Generate new access token
        return GenerateNewAccessToken(expiredToken);
    }
}
Issue: Suspicious Activity Detection
Symptoms: Unusual access patterns, failed login attempts

bash
# Diagnosis: Check security logs
sudo tail -f /var/log/secure
sudo grep "Failed password" /var/log/secure

# Check application security logs:
tail -f /var/log/pos-cloud/security.log

# Solutions: Implement intrusion detection
# Fail2ban configuration for SSH:
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 3600
🆘 Emergency Recovery Procedures
Complete System Failure
bash
# 1. Assess the situation
sudo systemctl status postgresql-15 redis nginx pos-api pos-signalr

# 2. Check resource usage
htop
df -h
free -h

# 3. Restart services in order
sudo systemctl restart postgresql-15
sudo systemctl restart redis
sudo systemctl restart pos-api
sudo systemctl restart pos-signalr
sudo systemctl restart nginx

# 4. Verify recovery
curl -f https://your-domain.com/health
curl -f https://your-domain.com/api/products

# 5. If still failing, restore from backup
/usr/local/bin/disaster-recovery.sh latest
Data Corruption Recovery
bash
# 1. Stop applications
sudo systemctl stop pos-api pos-signalr

# 2. Backup current state (for analysis)
sudo -u postgres pg_dump pos_cloud > /backup/corrupted-state.dump

# 3. Restore from last known good backup
/usr/local/bin/disaster-recovery.sh 2024-01-15

# 4. Start applications
sudo systemctl start pos-api pos-signalr

# 5. Verify data integrity
sudo -u postgres psql -d pos_cloud -c "SELECT count(*) FROM transactions;"
📞 Support Escalation Matrix
Level 1: Basic Troubleshooting (User/Operator)
Application restart

Basic connectivity checks

Cache clearing

Log file examination

Level 2: Technical Support (Admin/Developer)
Database queries and repairs

Service configuration changes

Performance optimization

Backup restoration

Level 3: Engineering Support (System Architect)
Code-level debugging

Infrastructure scaling

Security incident response

Architecture changes

Level 4: Vendor Support (Microsoft, PostgreSQL, etc.)
Platform-specific issues

Critical security patches

License and compliance issues

Last Updated: 2024-01-15
Maintenance Schedule: Monthly review and update
Emergency Contact: infrastructure@your-company.com

"The time to repair the roof is when the sun is shining." - John F. Kennedy

text

**Silakan copy content di atas dan commit:**

```bash
python auto_docs.py TROUBLESHOOTING_GUIDE.md "Add comprehensive troubleshooting guide for POS Hybrid System"
Sekarang kita sudah memiliki emergency response guide yang lengkap! 🚨
