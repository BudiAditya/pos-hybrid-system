📁 [FILE: API_DOCUMENTATION.md]
📝 [COMMIT: "Add comprehensive API documentation with examples"]
markdown
# POS Hybrid System - API Documentation

## 🌐 API Overview

**Base URL**: `https://api.your-domain.com/api`  
**API Version**: `v1`  
**Authentication**: JWT Bearer Token  
**Rate Limiting**: 1000 requests/hour per API key  
**Response Format**: JSON

## 🔐 Authentication

### JWT Token Authentication
All API requests require a valid JWT token in the Authorization header.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Obtaining Access Token
http
POST /api/auth/login
Content-Type: application/json

{
  "username": "store_admin",
  "password": "your_password",
  "storeId": "store-uuid-123"
}
Response:

json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 1440,
    "user": {
      "id": "user-uuid-123",
      "username": "store_admin",
      "email": "admin@store.com",
      "role": "StoreAdmin",
      "storeId": "store-uuid-123"
    }
  }
}
Token Refresh
http
POST /api/auth/refresh
Authorization: Bearer {current_token}
Content-Type: application/json

{
  "refreshToken": "refresh-token-here"
}
📦 Products API
Get Products List
Retrieve paginated list of products with filtering and sorting.

http
GET /api/products?storeId={storeId}&page=1&limit=20&search=keyword&category=Electronics
Authorization: Bearer {token}
Query Parameters:

Parameter	Type	Required	Description
storeId	string	Yes	Store identifier
page	integer	No	Page number (default: 1)
limit	integer	No	Items per page (default: 20, max: 100)
search	string	No	Search in name, code, or barcode
category	string	No	Filter by category
includeInactive	boolean	No	Include inactive products (default: false)
sortBy	string	No	Sort field (name, price, stock)
sortOrder	string	No	Sort order (asc, desc)
Response:

json
{
  "success": true,
  "data": {
    "products": [
      {
        "id": "product-uuid-123",
        "name": "iPhone 14 Pro",
        "code": "IP14P-256",
        "barcode": "194253774410",
        "price": 999.99,
        "costPrice": 850.00,
        "stockQuantity": 25,
        "minimumStock": 5,
        "category": "Electronics",
        "unit": "pcs",
        "isActive": true,
        "storeId": "store-uuid-123",
        "lastSynced": "2024-01-15T10:30:00Z",
        "createdAt": "2024-01-01T08:00:00Z",
        "updatedAt": "2024-01-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "totalItems": 150,
      "totalPages": 8,
      "hasNext": true,
      "hasPrev": false
    }
  }
}
Get Single Product
http
GET /api/products/{productId}
Authorization: Bearer {token}
Response:

json
{
  "success": true,
  "data": {
    "product": {
      "id": "product-uuid-123",
      "name": "iPhone 14 Pro",
      "code": "IP14P-256",
      "barcode": "194253774410",
      "price": 999.99,
      "costPrice": 850.00,
      "stockQuantity": 25,
      "minimumStock": 5,
      "category": "Electronics",
      "unit": "pcs",
      "isActive": true,
      "storeId": "store-uuid-123",
      "lastSynced": "2024-01-15T10:30:00Z"
    }
  }
}
Create Product
http
POST /api/products
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Samsung Galaxy S24",
  "code": "SGS24-512",
  "barcode": "887276625311",
  "price": 899.99,
  "costPrice": 750.00,
  "stockQuantity": 15,
  "minimumStock": 3,
  "category": "Electronics",
  "unit": "pcs",
  "storeId": "store-uuid-123"
}
Validation Rules:

name: Required, max 100 characters

code: Required, unique per store, max 20 characters

barcode: Optional, unique per store

price: Required, minimum 0.01

stockQuantity: Required, minimum 0

Update Product
http
PUT /api/products/{productId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Samsung Galaxy S24 Ultra",
  "price": 1099.99,
  "stockQuantity": 20,
  "category": "Premium Electronics"
}
Delete Product (Soft Delete)
http
DELETE /api/products/{productId}
Authorization: Bearer {token}
Bulk Product Operations
http
POST /api/products/bulk
Authorization: Bearer {token}
Content-Type: application/json

{
  "operations": [
    {
      "type": "update",
      "productId": "product-uuid-123",
      "data": {
        "price": 949.99,
        "stockQuantity": 30
      }
    },
    {
      "type": "create",
      "data": {
        "name": "New Product",
        "code": "NEW001",
        "price": 49.99,
        "stockQuantity": 100
      }
    }
  ]
}
💳 Transactions API
Create Transaction
Process a new sales transaction with multiple payment methods support.

http
POST /api/transactions
Authorization: Bearer {token}
Content-Type: application/json

{
  "storeId": "store-uuid-123",
  "customerName": "John Doe",
  "customerPhone": "+62123456789",
  "items": [
    {
      "productId": "product-uuid-123",
      "quantity": 1,
      "unitPrice": 999.99,
      "discountAmount": 0
    },
    {
      "productId": "product-uuid-456",
      "quantity": 2,
      "unitPrice": 49.99,
      "discountAmount": 5.00
    }
  ],
  "paymentMethod": "qris",
  "discountAmount": 10.00,
  "taxAmount": 89.99,
  "amountPaid": 1129.97,
  "notes": "Customer requested gift wrapping"
}
Payment Method Values:

cash - Cash payment

creditCard - Credit card

debitCard - Debit card

qris - QRIS payment

gopay - GoPay

ovo - OVO

dana - DANA

linkAja - LinkAja

bankTransfer - Bank transfer

Response:

json
{
  "success": true,
  "data": {
    "transaction": {
      "id": "transaction-uuid-123",
      "transactionNumber": "TXN202401150001",
      "storeId": "store-uuid-123",
      "status": "completed",
      "subtotal": 1099.97,
      "taxAmount": 89.99,
      "discountAmount": 10.00,
      "totalAmount": 1179.96,
      "amountPaid": 1200.00,
      "changeAmount": 20.04,
      "transactionDate": "2024-01-15T14:30:00Z",
      "customerName": "John Doe",
      "customerPhone": "+62123456789",
      "paymentMethod": "qris",
      "paymentReference": "QRIS123456789",
      "isSynced": true,
      "syncedAt": "2024-01-15T14:30:05Z",
      "items": [
        {
          "id": "item-uuid-123",
          "productId": "product-uuid-123",
          "productName": "iPhone 14 Pro",
          "quantity": 1,
          "unitPrice": 999.99,
          "discountAmount": 0,
          "totalPrice": 999.99
        }
      ]
    },
    "paymentInfo": {
      "paymentUrl": "https://qr-code.url/generate?amount=1179.96",
      "expiryTime": "2024-01-15T14:45:00Z",
      "instructions": "Scan QR code to complete payment"
    }
  }
}
Get Transaction Details
http
GET /api/transactions/{transactionId}
Authorization: Bearer {token}
Get Transactions List
http
GET /api/transactions?storeId={storeId}&startDate=2024-01-01&endDate=2024-01-15&page=1&limit=20
Authorization: Bearer {token}
Query Parameters:

Parameter	Type	Required	Description
storeId	string	Yes	Store identifier
startDate	string	No	Start date (YYYY-MM-DD)
endDate	string	No	End date (YYYY-MM-DD)
status	string	No	Filter by status (pending, completed, cancelled)
paymentMethod	string	No	Filter by payment method
page	integer	No	Page number
limit	integer	No	Items per page
Update Transaction Status
http
PATCH /api/transactions/{transactionId}/status
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "cancelled",
  "reason": "Customer changed mind"
}
Process Refund
http
POST /api/transactions/{transactionId}/refund
Authorization: Bearer {token}
Content-Type: application/json

{
  "refundAmount": 999.99,
  "reason": "Product defect",
  "items": [
    {
      "itemId": "item-uuid-123",
      "quantity": 1,
      "refundAmount": 999.99
    }
  ]
}
🔄 Sync API
Sync Data (Bidirectional)
Main endpoint for synchronizing data between cloud and clients.

http
POST /api/sync/data
Authorization: Bearer {token}
Content-Type: application/json

{
  "storeId": "store-uuid-123",
  "lastSync": "2024-01-14T10:00:00Z",
  "products": [
    {
      "id": "product-uuid-123",
      "name": "Updated Product Name",
      "price": 899.99,
      "stockQuantity": 10,
      "isDirty": true,
      "lastSynced": "2024-01-15T09:00:00Z"
    }
  ],
  "transactions": [
    {
      "id": "transaction-uuid-123",
      "transactionNumber": "TXN202401150001",
      "totalAmount": 1179.96,
      "status": "completed",
      "isSynced": false
    }
  ],
  "deletedProducts": ["product-uuid-456"],
  "deviceId": "desktop-001",
  "syncType": "full"
}
Response:

json
{
  "success": true,
  "data": {
    "serverTime": "2024-01-15T14:35:00Z",
    "conflicts": [],
    "products": [
      {
        "id": "product-uuid-789",
        "name": "New Product from Cloud",
        "price": 299.99,
        "stockQuantity": 50,
        "lastSynced": "2024-01-15T14:30:00Z",
        "action": "created"
      }
    ],
    "transactions": [
      {
        "id": "transaction-uuid-456",
        "transactionNumber": "TXN202401150002",
        "totalAmount": 249.99,
        "status": "completed",
        "action": "created"
      }
    ],
    "deletedProducts": ["product-uuid-999"],
    "summary": {
      "productsReceived": 1,
      "productsSent": 1,
      "transactionsReceived": 1,
      "transactionsSent": 1,
      "conflictsResolved": 0
    }
  }
}
Get Products for Sync
http
GET /api/sync/products/{storeId}?lastSync=2024-01-14T10:00:00Z
Authorization: Bearer {token}
Get Transactions for Sync
http
GET /api/sync/transactions/{storeId}?lastSync=2024-01-14T10:00:00Z
Authorization: Bearer {token}
Conflict Resolution
http
POST /api/sync/conflicts/resolve
Authorization: Bearer {token}
Content-Type: application/json

{
  "conflicts": [
    {
      "entityType": "Product",
      "entityId": "product-uuid-123",
      "localVersion": "2024-01-15T09:00:00Z",
      "remoteVersion": "2024-01-15T10:00:00Z",
      "resolution": "useRemote"
    }
  ]
}
🏪 Store Management API
Get Store Information
http
GET /api/stores/{storeId}
Authorization: Bearer {token}
Response:

json
{
  "success": true,
  "data": {
    "store": {
      "id": "store-uuid-123",
      "name": "Main Store",
      "code": "STORE001",
      "address": "123 Main Street, Jakarta",
      "phone": "+62123456789",
      "email": "store@company.com",
      "isActive": true,
      "subscriptionExpiry": "2024-12-31T23:59:59Z",
      "hardwareConfig": {
        "receiptPrinter": "EPSON TM-T88V",
        "barcodeScanner": "USB HID",
        "customerDisplay": "LCD 20x4"
      },
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-15T10:00:00Z"
    }
  }
}
Update Store Configuration
http
PUT /api/stores/{storeId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Store Name",
  "address": "456 New Street, Jakarta",
  "phone": "+62987654321",
  "hardwareConfig": {
    "receiptPrinter": "Star TSP143",
    "barcodeScanner": "Bluetooth"
  }
}
Get Store Statistics
http
GET /api/stores/{storeId}/statistics?startDate=2024-01-01&endDate=2024-01-15
Authorization: Bearer {token}
Response:

json
{
  "success": true,
  "data": {
    "statistics": {
      "totalSales": 12500.00,
      "totalTransactions": 45,
      "averageTransactionValue": 277.78,
      "topProducts": [
        {
          "productId": "product-uuid-123",
          "productName": "iPhone 14 Pro",
          "quantitySold": 15,
          "revenue": 14999.85
        }
      ],
      "paymentMethodBreakdown": {
        "cash": 12000.00,
        "qris": 3500.00,
        "creditCard": 2000.00
      },
      "dailySales": [
        {
          "date": "2024-01-15",
          "sales": 1500.00,
          "transactions": 5
        }
      ]
    }
  }
}
📊 Reports API
Sales Report
http
GET /api/reports/sales?storeId={storeId}&startDate=2024-01-01&endDate=2024-01-15&groupBy=day
Authorization: Bearer {token}
Query Parameters:

Parameter	Type	Required	Description
storeId	string	Yes	Store identifier
startDate	string	Yes	Start date (YYYY-MM-DD)
endDate	string	Yes	End date (YYYY-MM-DD)
groupBy	string	No	Grouping (day, week, month, product)
format	string	No	Output format (json, csv, pdf)
Inventory Report
http
GET /api/reports/inventory?storeId={storeId}&lowStockOnly=true
Authorization: Bearer {token}
Response:

json
{
  "success": true,
  "data": {
    "report": {
      "totalProducts": 150,
      "totalValue": 75000.00,
      "lowStockItems": 12,
      "outOfStockItems": 3,
      "inventoryByCategory": [
        {
          "category": "Electronics",
          "count": 45,
          "value": 45000.00
        }
      ],
      "lowStockAlerts": [
        {
          "productId": "product-uuid-123",
          "productName": "iPhone 14 Pro",
          "currentStock": 2,
          "minimumStock": 5,
          "lastSold": "2024-01-15T14:30:00Z"
        }
      ]
    }
  }
}
Transaction Report
http
GET /api/reports/transactions?storeId={storeId}&startDate=2024-01-01&endDate=2024-01-15
Authorization: Bearer {token}
🔔 Real-time Events (WebSocket/SignalR)
Connection Setup
Connect to SignalR hub for real-time updates.

javascript
// JavaScript client example
const connection = new signalR.HubConnectionBuilder()
    .withUrl("https://api.your-domain.com/poshub", {
        accessTokenFactory: () => getAccessToken()
    })
    .configureLogging(signalR.LogLevel.Information)
    .build();
Available Events
Stock Updates
Emitted when product stock changes.

Event: StockUpdated
Data:

json
{
  "productId": "product-uuid-123",
  "newStock": 15,
  "previousStock": 20,
  "reason": "sale",
  "transactionId": "transaction-uuid-123",
  "timestamp": "2024-01-15T14:30:00Z",
  "updatedBy": "user-uuid-123"
}
New Transactions
Emitted when new transaction is created.

Event: NewTransaction
Data:

json
{
  "transactionId": "transaction-uuid-123",
  "transactionNumber": "TXN202401150001",
  "totalAmount": 1179.96,
  "paymentMethod": "qris",
  "customerName": "John Doe",
  "timestamp": "2024-01-15T14:30:00Z",
  "items": [
    {
      "productId": "product-uuid-123",
      "productName": "iPhone 14 Pro",
      "quantity": 1,
      "unitPrice": 999.99
    }
  ]
}
Price Updates
Emitted when product price changes.

Event: ProductPriceUpdated
Data:

json
{
  "productId": "product-uuid-123",
  "newPrice": 899.99,
  "previousPrice": 999.99,
  "reason": "promotion",
  "timestamp": "2024-01-15T14:30:00Z",
  "updatedBy": "user-uuid-123"
}
Client Methods
Join Store Group
Subscribe to store-specific events.

javascript
// Join store group
await connection.invoke("JoinStore", "store-uuid-123");

// Leave store group
await connection.invoke("LeaveStore", "store-uuid-123");
Request Sync
Request immediate data synchronization.

javascript
await connection.invoke("RequestSync", "store-uuid-123", "2024-01-14T10:00:00Z");
🛡️ Error Handling
Standard Error Response
All API errors follow a consistent format:

json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "price",
        "message": "Price must be greater than 0"
      }
    ],
    "timestamp": "2024-01-15T14:30:00Z",
    "requestId": "req-123456"
  }
}
Common Error Codes
HTTP Status	Error Code	Description
400	VALIDATION_ERROR	Input validation failed
401	UNAUTHORIZED	Authentication required
403	FORBIDDEN	Insufficient permissions
404	NOT_FOUND	Resource not found
409	CONFLICT	Data conflict detected
429	RATE_LIMITED	Rate limit exceeded
500	INTERNAL_ERROR	Server error
503	SERVICE_UNAVAILABLE	Service temporarily unavailable
Retry Logic
Clients should implement exponential backoff for retries:

javascript
async function apiCallWithRetry(url, options, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(url, options);
            if (response.ok) return response;
            
            if (response.status >= 500) {
                // Retry on server errors
                await new Promise(resolve => 
                    setTimeout(resolve, Math.pow(2, attempt) * 1000)
                );
                continue;
            }
            // Don't retry on client errors
            throw new Error(`HTTP ${response.status}`);
        } catch (error) {
            if (attempt === maxRetries) throw error;
        }
    }
}
⚡ Performance Tips
Caching Strategies
http
# Enable client-side caching
GET /api/products
Cache-Control: max-age=300  # Cache for 5 minutes
ETag: "abc123"
Pagination Best Practices
http
# Use keyset pagination for large datasets
GET /api/transactions?storeId=123&limit=20&after=2024-01-15T10:00:00Z
Batch Operations
http
# Use batch endpoints for multiple operations
POST /api/products/bulk
🔐 Security Headers
Required Headers
http
Authorization: Bearer {token}
Content-Type: application/json
User-Agent: POS-Client/1.0.0
Recommended Client Headers
http
X-Client-Version: 1.0.0
X-Device-Id: desktop-001
X-Store-Id: store-uuid-123
📋 Rate Limiting
Limits by Endpoint Category
Category	Limit	Window	Description
Authentication	10	1 hour	Login attempts
Products	1000	1 hour	Product operations
Transactions	500	1 hour	Transaction processing
Reports	100	1 hour	Report generation
Sync	200	1 hour	Data synchronization
Rate Limit Headers
http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1642252800
🧪 Testing Endpoints
Health Check
http
GET /health
Response:

json
{
  "status": "healthy",
  "timestamp": "2024-01-15T14:30:00Z",
  "version": "1.0.0",
  "services": {
    "database": "connected",
    "redis": "connected",
    "signalr": "connected"
  }
}
Echo Test
http
POST /api/test/echo
Content-Type: application/json

{
  "message": "Hello, World!",
  "timestamp": "2024-01-15T14:30:00Z"
}
Response:

json
{
  "success": true,
  "data": {
    "message": "Hello, World!",
    "timestamp": "2024-01-15T14:30:00Z",
    "receivedAt": "2024-01-15T14:30:01Z"
  }
}
📚 SDKs and Client Libraries
.NET Client Example
csharp
public class PosApiClient
{
    private readonly HttpClient _httpClient;
    
    public PosApiClient(string baseUrl, string token)
    {
        _httpClient = new HttpClient
        {
            BaseAddress = new Uri(baseUrl)
        };
        _httpClient.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
    }
    
    public async Task<List<Product>> GetProductsAsync(string storeId, int page = 1, int limit = 20)
    {
        var response = await _httpClient.GetAsync($"/api/products?storeId={storeId}&page={page}&limit={limit}");
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadFromJsonAsync<ApiResponse<List<Product>>>();
        return content.Data;
    }
}
JavaScript/TypeScript Client
typescript
class PosApiClient {
    private baseUrl: string;
    private token: string;
    
    constructor(baseUrl: string, token: string) {
        this.baseUrl = baseUrl;
        this.token = token;
    }
    
    async getProducts(storeId: string, options?: { page?: number; limit?: number }) {
        const params = new URLSearchParams({
            storeId,
            page: options?.page?.toString() || '1',
            limit: options?.limit?.toString() || '20'
        });
        
        const response = await fetch(`${this.baseUrl}/api/products?${params}`, {
            headers: {
                'Authorization': `Bearer ${this.token}`,
                'Content-Type': 'application/json'
            }
        });
        
        if (!response.ok) {
            throw new Error(`API error: ${response.status}`);
        }
        
        return await response.json();
    }
}
🔄 Webhook Support (Optional)
Transaction Webhook
Receive notifications when transactions are completed.

Endpoint: POST /webhooks/transactions
Content-Type: application/json

Payload:

json
{
  "event": "transaction.completed",
  "data": {
    "transactionId": "transaction-uuid-123",
    "transactionNumber": "TXN202401150001",
    "totalAmount": 1179.96,
    "paymentMethod": "qris",
    "customerPhone": "+62123456789",
    "timestamp": "2024-01-15T14:30:00Z"
  },
  "signature": "hmac-signature"
}
API Version: 1.0.0
Last Updated: 2024-01-15
Contact: api-support@your-company.com
Support Hours: 24/7 for critical issues

For immediate assistance with API integration, contact our developer support team.
