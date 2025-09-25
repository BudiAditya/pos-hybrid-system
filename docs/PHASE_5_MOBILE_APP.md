📁 [FILE: PHASE_5_MOBILE_APP.md]
📝 [COMMIT: "Add complete Phase 5 Mobile Application documentation"]
markdown
# Phase 5: Mobile Application (Flutter)

## Overview
Cross-platform mobile application menggunakan Flutter untuk iOS dan Android dengan real-time sync, offline capability, dan modern UI/UX.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Flutter Application"
        A[UI Layer - Widgets]
        B[Business Logic - Bloc/Cubit]
        C[Data Layer - Repositories]
        D[Local Storage - SQLite]
    end
    
    subgraph "Cloud Services"
        E[REST API]
        F[SignalR Hub]
        G[Cloud PostgreSQL]
    end
    
    subgraph "Device Features"
        H[Camera - Barcode Scan]
        I[Push Notifications]
        J[Local Authentication]
        K[Network Connectivity]
    end
    
    A --> B
    B --> C
    C --> D
    C --> E
    C --> F
    A --> H
    B --> I
    B --> J
    B --> K
Project Structure
text
pos_mobile/
├── 📁 android/                 # Android specific files
├── 📁 ios/                     # iOS specific files
├── 📁 lib/
│   ├── 📁 src/
│   │   ├── 📁 core/            # Domain models, constants, utilities
│   │   ├── 📁 data/            # Data layer - repositories, datasources
│   │   ├── 📁 domain/          # Business logic - use cases, entities
│   │   ├── 📁 presentation/    # UI layer - pages, widgets, blocs
│   │   └── 📁 services/        # External services - API, SignalR, etc.
│   ├── 📁 generated/           # Auto-generated files
│   ├── main.dart               # Application entry point
│   └── injection_container.dart # Dependency injection
├── 📁 test/                    # Unit tests
└── 📁 integration_test/        # Integration tests
Step 1: Project Setup dan Dependencies
1.1 Create Flutter Project
bash
# Create new Flutter project
flutter create pos_mobile
cd pos_mobile

# Clean up default structure
rm -rf lib/main.dart
mkdir -p lib/src/{core,data,domain,presentation,services}
1.2 pubspec.yaml Configuration
yaml
name: pos_mobile
description: POS Mobile Application for iOS and Android
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2

  # State Management
  flutter_bloc: ^8.1.3
  equatable: ^2.0.5

  # HTTP & API
  dio: ^5.3.2
  retrofit: ^4.0.1
  logger: ^1.1.0

  # SignalR Real-time
  signalr_netcore: ^3.0.0

  # Local Database
  sqflite: ^2.3.0
  path: ^1.8.3

  # Object Mapping
  json_annotation: ^4.8.1

  # Dependency Injection
  get_it: ^7.6.4
  injectable: ^2.1.0

  # Forms & Validation
  formz: ^0.5.0

  # Connectivity
  connectivity_plus: ^5.0.1

  # Local Authentication
  local_auth: ^2.1.2

  # Barcode Scanning
  mobile_scanner: ^3.6.0

  # Push Notifications
  firebase_messaging: ^14.6.5

  # File Storage
  shared_preferences: ^2.2.2

  # UI Components
  flutter_staggered_grid_view: ^0.7.0
  pull_to_refresh: ^2.0.0
  badges: ^3.0.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  build_runner: ^2.4.6
  retrofit_generator: ^4.0.1
  json_serializable: ^6.7.1
  injectable_generator: ^2.1.1

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/icons/
  
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
Step 2: Domain Layer (Core Business Logic)
2.1 Entity Models
dart
// lib/src/core/entities/product.dart
class Product {
  final String id;
  final String name;
  final String code;
  final String barcode;
  final double price;
  final double costPrice;
  final int stockQuantity;
  final int minimumStock;
  final String category;
  final String unit;
  final bool isActive;
  final String storeId;
  final DateTime lastSynced;
  final bool isDirty;

  const Product({
    required this.id,
    required this.name,
    required this.code,
    required this.barcode,
    required this.price,
    required this.costPrice,
    required this.stockQuantity,
    required this.minimumStock,
    required this.category,
    required this.unit,
    required this.isActive,
    required this.storeId,
    required this.lastSynced,
    required this.isDirty,
  });

  Product copyWith({
    String? id,
    String? name,
    String? code,
    String? barcode,
    double? price,
    double? costPrice,
    int? stockQuantity,
    int? minimumStock,
    String? category,
    String? unit,
    bool? isActive,
    String? storeId,
    DateTime? lastSynced,
    bool? isDirty,
  }) {
    return Product(
      id: id ?? this.id,
      name: name ?? this.name,
      code: code ?? this.code,
      barcode: barcode ?? this.barcode,
      price: price ?? this.price,
      costPrice: costPrice ?? this.costPrice,
      stockQuantity: stockQuantity ?? this.stockQuantity,
      minimumStock: minimumStock ?? this.minimumStock,
      category: category ?? this.category,
      unit: unit ?? this.unit,
      isActive: isActive ?? this.isActive,
      storeId: storeId ?? this.storeId,
      lastSynced: lastSynced ?? this.lastSynced,
      isDirty: isDirty ?? this.isDirty,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'code': code,
      'barcode': barcode,
      'price': price,
      'costPrice': costPrice,
      'stockQuantity': stockQuantity,
      'minimumStock': minimumStock,
      'category': category,
      'unit': unit,
      'isActive': isActive,
      'storeId': storeId,
      'lastSynced': lastSynced.toIso8601String(),
      'isDirty': isDirty,
    };
  }

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      code: json['code'] as String,
      barcode: json['barcode'] as String,
      price: (json['price'] as num).toDouble(),
      costPrice: (json['costPrice'] as num).toDouble(),
      stockQuantity: json['stockQuantity'] as int,
      minimumStock: json['minimumStock'] as int,
      category: json['category'] as String,
      unit: json['unit'] as String,
      isActive: json['isActive'] as bool,
      storeId: json['storeId'] as String,
      lastSynced: DateTime.parse(json['lastSynced'] as String),
      isDirty: json['isDirty'] as bool,
    );
  }
}

// lib/src/core/entities/transaction.dart
class Transaction {
  final String id;
  final String transactionNumber;
  final String storeId;
  final TransactionStatus status;
  final double subtotal;
  final double taxAmount;
  final double discountAmount;
  final double totalAmount;
  final double amountPaid;
  final double changeAmount;
  final DateTime transactionDate;
  final String? customerName;
  final String? customerPhone;
  final PaymentMethod paymentMethod;
  final String? paymentReference;
  final bool isSynced;
  final DateTime? syncedAt;
  final List<TransactionItem> items;

  const Transaction({
    required this.id,
    required this.transactionNumber,
    required this.storeId,
    required this.status,
    required this.subtotal,
    required this.taxAmount,
    required this.discountAmount,
    required this.totalAmount,
    required this.amountPaid,
    required this.changeAmount,
    required this.transactionDate,
    this.customerName,
    this.customerPhone,
    required this.paymentMethod,
    this.paymentReference,
    required this.isSynced,
    this.syncedAt,
    required this.items,
  });
}

// lib/src/core/entities/transaction_item.dart
class TransactionItem {
  final String id;
  final String transactionId;
  final String productId;
  final Product? product;
  final int quantity;
  final double unitPrice;
  final double discountAmount;
  final double totalPrice;

  const TransactionItem({
    required this.id,
    required this.transactionId,
    required this.productId,
    this.product,
    required this.quantity,
    required this.unitPrice,
    required this.discountAmount,
    required this.totalPrice,
  });
}
2.2 Enums dan Constants
dart
// lib/src/core/enums/payment_method.dart
enum PaymentMethod {
  cash(1, 'Cash', '💰'),
  creditCard(2, 'Credit Card', '💳'),
  debitCard(3, 'Debit Card', '💳'),
  qris(4, 'QRIS', '📱'),
  gopay(5, 'GoPay', '📱'),
  ovo(6, 'OVO', '📱'),
  dana(7, 'DANA', '📱'),
  linkAja(8, 'LinkAja', '📱'),
  bankTransfer(9, 'Bank Transfer', '🏦');

  final int value;
  final String displayName;
  final String emoji;

  const PaymentMethod(this.value, this.displayName, this.emoji);

  static PaymentMethod fromValue(int value) {
    return PaymentMethod.values.firstWhere(
      (e) => e.value == value,
      orElse: () => PaymentMethod.cash,
    );
  }
}

// lib/src/core/enums/transaction_status.dart
enum TransactionStatus {
  pending(1, 'Pending', Colors.orange),
  completed(2, 'Completed', Colors.green),
  cancelled(3, 'Cancelled', Colors.red),
  refunded(4, 'Refunded', Colors.blue);

  final int value;
  final String displayName;
  final Color color;

  const TransactionStatus(this.value, this.displayName, this.color);
}

// lib/src/core/constants/app_constants.dart
class AppConstants {
  static const String appName = 'POS Mobile';
  static const String apiBaseUrl = 'https://your-api-domain.com/api';
  static const String signalRHubUrl = 'https://your-api-domain.com/poshub';
  
  // Database
  static const String databaseName = 'pos_mobile.db';
  static const int databaseVersion = 1;
  
  // Sync intervals
  static const Duration syncInterval = Duration(minutes: 5);
  static const Duration connectionTimeout = Duration(seconds: 30);
  
  // Pagination
  static const int productsPageSize = 20;
  static const int transactionsPageSize = 15;
}
Step 3: Data Layer (Repositories & Data Sources)
3.1 Local Database (SQLite)
dart
// lib/src/data/datasources/local/database_helper.dart
class DatabaseHelper {
  static Database? _database;
  
  static Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  static Future<Database> _initDatabase() async {
    final databasesPath = await getDatabasesPath();
    final path = join(databasesPath, AppConstants.databaseName);
    
    return await openDatabase(
      path,
      version: AppConstants.databaseVersion,
      onCreate: _createDatabase,
      onUpgrade: _upgradeDatabase,
    );
  }
  
  static Future<void> _createDatabase(Database db, int version) async {
    // Products table
    await db.execute('''
      CREATE TABLE products (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        code TEXT NOT NULL,
        barcode TEXT,
        price REAL NOT NULL,
        cost_price REAL NOT NULL,
        stock_quantity INTEGER NOT NULL,
        minimum_stock INTEGER NOT NULL,
        category TEXT NOT NULL,
        unit TEXT NOT NULL,
        is_active INTEGER NOT NULL,
        store_id TEXT NOT NULL,
        last_synced INTEGER NOT NULL,
        is_dirty INTEGER NOT NULL,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL
      )
    ''');
    
    // Transactions table
    await db.execute('''
      CREATE TABLE transactions (
        id TEXT PRIMARY KEY,
        transaction_number TEXT NOT NULL UNIQUE,
        store_id TEXT NOT NULL,
        status INTEGER NOT NULL,
        subtotal REAL NOT NULL,
        tax_amount REAL NOT NULL,
        discount_amount REAL NOT NULL,
        total_amount REAL NOT NULL,
        amount_paid REAL NOT NULL,
        change_amount REAL NOT NULL,
        transaction_date INTEGER NOT NULL,
        customer_name TEXT,
        customer_phone TEXT,
        payment_method INTEGER NOT NULL,
        payment_reference TEXT,
        is_synced INTEGER NOT NULL,
        synced_at INTEGER,
        created_at INTEGER NOT NULL,
        updated_at INTEGER NOT NULL
      )
    ''');
    
    // Transaction items table
    await db.execute('''
      CREATE TABLE transaction_items (
        id TEXT PRIMARY KEY,
        transaction_id TEXT NOT NULL,
        product_id TEXT NOT NULL,
        quantity INTEGER NOT NULL,
        unit_price REAL NOT NULL,
        discount_amount REAL NOT NULL,
        total_price REAL NOT NULL,
        FOREIGN KEY (transaction_id) REFERENCES transactions (id),
        FOREIGN KEY (product_id) REFERENCES products (id)
      )
    ''');
    
    // Indexes for performance
    await db.execute('CREATE INDEX idx_products_store_id ON products(store_id)');
    await db.execute('CREATE INDEX idx_products_code ON products(code)');
    await db.execute('CREATE INDEX idx_products_barcode ON products(barcode)');
    await db.execute('CREATE INDEX idx_transactions_store_id ON transactions(store_id)');
    await db.execute('CREATE INDEX idx_transactions_date ON transactions(transaction_date)');
  }
  
  static Future<void> _upgradeDatabase(Database db, int oldVersion, int newVersion) async {
    // Database migration logic
    if (oldVersion < 2) {
      // Add new columns or tables for version 2
    }
  }
}
3.2 Product Repository Implementation
dart
// lib/src/data/repositories/product_repository_impl.dart
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource remoteDataSource;
  final ProductLocalDataSource localDataSource;
  final NetworkInfo networkInfo;
  
  ProductRepositoryImpl({
    required this.remoteDataSource,
    required this.localDataSource,
    required this.networkInfo,
  });
  
  @override
  Future<Either<Failure, List<Product>>> getProducts({
    bool includeInactive = false,
    int page = 1,
    int limit = AppConstants.productsPageSize,
  }) async {
    try {
      if (await networkInfo.isConnected) {
        // Try to get from remote
        final remoteProducts = await remoteDataSource.getProducts(
          includeInactive: includeInactive,
          page: page,
          limit: limit,
        );
        
        // Cache the remote data
        await localDataSource.cacheProducts(remoteProducts);
        
        return Right(remoteProducts);
      } else {
        // Get from local cache
        final localProducts = await localDataSource.getProducts(
          includeInactive: includeInactive,
          page: page,
          limit: limit,
        );
        
        return Right(localProducts);
      }
    } on ServerException catch (e) {
      // Fallback to local data on server error
      final localProducts = await localDataSource.getProducts(
        includeInactive: includeInactive,
        page: page,
        limit: limit,
      );
      
      return Right(localProducts);
    } catch (e) {
      return Left(DataSourceException(message: e.toString()));
    }
  }
  
  @override
  Future<Either<Failure, Product>> getProductById(String id) async {
    try {
      // Try local first for speed
      final localProduct = await localDataSource.getProductById(id);
      if (localProduct != null) {
        return Right(localProduct);
      }
      
      if (await networkInfo.isConnected) {
        final remoteProduct = await remoteDataSource.getProductById(id);
        await localDataSource.cacheProduct(remoteProduct);
        return Right(remoteProduct);
      } else {
        return Left(CacheException());
      }
    } catch (e) {
      return Left(DataSourceException(message: e.toString()));
    }
  }
  
  @override
  Future<Either<Failure, List<Product>>> searchProducts(String query) async {
    try {
      // Always search locally first for instant results
      final localResults = await localDataSource.searchProducts(query);
      
      if (await networkInfo.isConnected) {
        // Perform remote search for more comprehensive results
        final remoteResults = await remoteDataSource.searchProducts(query);
        
        // Merge results (remove duplicates)
        final allResults = {...localResults, ...remoteResults}.toList();
        return Right(allResults);
      } else {
        return Right(localResults);
      }
    } catch (e) {
      return Left(DataSourceException(message: e.toString()));
    }
  }
  
  @override
  Future<Either<Failure, void>> syncProducts() async {
    try {
      if (!await networkInfo.isConnected) {
        return Left(NetworkException());
      }
      
      // Get local changes that need to be synced
      final dirtyProducts = await localDataSource.getDirtyProducts();
      
      // Sync each dirty product
      for (final product in dirtyProducts) {
        await remoteDataSource.updateProduct(product);
        await localDataSource.markProductAsClean(product.id);
      }
      
      // Get latest products from server
      final remoteProducts = await remoteDataSource.getProducts();
      await localDataSource.cacheProducts(remoteProducts);
      
      return const Right(null);
    } catch (e) {
      return Left(DataSourceException(message: e.toString()));
    }
  }
}
3.3 API Client dengan Retrofit
dart
// lib/src/data/datasources/remote/api_service.dart
part 'api_service.g.dart';

@RestApi(baseUrl: AppConstants.apiBaseUrl)
abstract class ApiService {
  factory ApiService(Dio dio, {String baseUrl}) = _ApiService;
  
  @GET('/products')
  Future<List<ProductDto>> getProducts({
    @Query('includeInactive') bool includeInactive = false,
    @Query('page') int page = 1,
    @Query('limit') int limit = 20,
    @Query('lastSync') int? lastSync,
  });
  
  @GET('/products/{id}')
  Future<ProductDto> getProductById(@Path('id') String id);
  
  @GET('/products/search')
  Future<List<ProductDto>> searchProducts(@Query('q') String query);
  
  @POST('/products')
  Future<ProductDto> createProduct(@Body() CreateProductDto product);
  
  @PUT('/products/{id}')
  Future<ProductDto> updateProduct(@Path('id') String id, @Body() UpdateProductDto product);
  
  @POST('/transactions')
  Future<TransactionDto> createTransaction(@Body() CreateTransactionDto transaction);
  
  @GET('/transactions')
  Future<List<TransactionDto>> getTransactions({
    @Query('storeId') required String storeId,
    @Query('startDate') int? startDate,
    @Query('endDate') int? endDate,
    @Query('page') int page = 1,
    @Query('limit') int limit = 15,
  });
  
  @POST('/sync/data')
  Future<SyncResponseDto> syncData(@Body() SyncRequestDto request);
  
  @GET('/sync/products/{storeId}')
  Future<List<ProductDto>> getProductsForSync(
    @Path('storeId') String storeId,
    @Query('lastSync') int lastSync,
  );
}

// DTO classes
class ProductDto {
  final String id;
  final String name;
  final String code;
  final String barcode;
  final double price;
  final double costPrice;
  final int stockQuantity;
  final int minimumStock;
  final String category;
  final String unit;
  final bool isActive;
  final String storeId;
  final DateTime lastSynced;
  final bool isDirty;
  
  ProductDto({
    required this.id,
    required this.name,
    required this.code,
    required this.barcode,
    required this.price,
    required this.costPrice,
    required this.stockQuantity,
    required this.minimumStock,
    required this.category,
    required this.unit,
    required this.isActive,
    required this.storeId,
    required this.lastSynced,
    required this.isDirty,
  });
  
  factory ProductDto.fromJson(Map<String, dynamic> json) => _$ProductDtoFromJson(json);
  Map<String, dynamic> toJson() => _$ProductDtoToJson(this);
}
Step 4: Business Logic Layer (Bloc/Cubit)
4.1 Product Bloc
dart
// lib/src/presentation/blocs/product/product_bloc.dart
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  final GetProducts getProducts;
  final SearchProducts searchProducts;
  final GetProductById getProductById;
  final SyncProducts syncProducts;
  
  ProductBloc({
    required this.getProducts,
    required this.searchProducts,
    required this.getProductById,
    required this.syncProducts,
  }) : super(ProductInitial()) {
    on<LoadProducts>(_onLoadProducts);
    on<SearchProduct>(_onSearchProduct);
    on<LoadProductDetail>(_onLoadProductDetail);
    on<SyncProductsEvent>(_onSyncProducts);
  }
  
  Future<void> _onLoadProducts(
    LoadProducts event,
    Emitter<ProductState> emit,
  ) async {
    emit(ProductLoading());
    
    final result = await getProducts.call(ProductsParams(
      includeInactive: event.includeInactive,
      page: event.page,
      limit: event.limit,
    ));
    
    result.fold(
      (failure) => emit(ProductError(failure.message)),
      (products) => emit(ProductLoaded(products: products)),
    );
  }
  
  Future<void> _onSearchProduct(
    SearchProduct event,
    Emitter<ProductState> emit,
  ) async {
    if (event.query.isEmpty) {
      add(LoadProducts());
      return;
    }
    
    emit(ProductLoading());
    
    final result = await searchProducts.call(SearchProductsParams(query: event.query));
    
    result.fold(
      (failure) => emit(ProductError(failure.message)),
      (products) => emit(ProductLoaded(products: products)),
    );
  }
  
  Future<void> _onLoadProductDetail(
    LoadProductDetail event,
    Emitter<ProductState> emit,
  ) async {
    emit(ProductDetailLoading());
    
    final result = await getProductById.call(ProductDetailParams(id: event.productId));
    
    result.fold(
      (failure) => emit(ProductDetailError(failure.message)),
      (product) => emit(ProductDetailLoaded(product: product)),
    );
  }
  
  Future<void> _onSyncProducts(
    SyncProductsEvent event,
    Emitter<ProductState> emit,
  ) async {
    emit(ProductSyncing());
    
    final result = await syncProducts.call(NoParams());
    
    result.fold(
      (failure) => emit(ProductSyncError(failure.message)),
      (_) {
        emit(ProductSyncSuccess());
        add(LoadProducts()); // Reload products after sync
      },
    );
  }
}

// Product States
abstract class ProductState extends Equatable {
  const ProductState();
  
  @override
  List<Object> get props => [];
}

class ProductInitial extends ProductState {}
class ProductLoading extends ProductState {}
class ProductLoaded extends ProductState {
  final List<Product> products;
  
  const ProductLoaded({required this.products});
  
  @override
  List<Object> get props => [products];
}
class ProductError extends ProductState {
  final String message;
  
  const ProductError(this.message);
  
  @override
  List<Object> get props => [message];
}
class ProductSyncing extends ProductState {}
class ProductSyncSuccess extends ProductState {}
class ProductSyncError extends ProductState {
  final String message;
  
  const ProductSyncError(this.message);
  
  @override
  List<Object> get props => [message];
}

// Product Events
abstract class ProductEvent extends Equatable {
  const ProductEvent();
  
  @override
  List<Object> get props => [];
}

class LoadProducts extends ProductEvent {
  final bool includeInactive;
  final int page;
  final int limit;
  
  const LoadProducts({
    this.includeInactive = false,
    this.page = 1,
    this.limit = AppConstants.productsPageSize,
  });
  
  @override
  List<Object> get props => [includeInactive, page, limit];
}

class SearchProduct extends ProductEvent {
  final String query;
  
  const SearchProduct(this.query);
  
  @override
  List<Object> get props => [query];
}
4.2 Cart Cubit untuk Transaction Management
dart
// lib/src/presentation/cubits/cart/cart_cubit.dart
class CartCubit extends Cubit<CartState> {
  CartCubit() : super(CartInitial());
  
  final List<CartItem> _items = [];
  
  void addItem(Product product, [int quantity = 1]) {
    final existingIndex = _items.indexWhere((item) => item.product.id == product.id);
    
    if (existingIndex >= 0) {
      // Update existing item
      _items[existingIndex] = _items[existingIndex].copyWith(
        quantity: _items[existingIndex].quantity + quantity,
      );
    } else {
      // Add new item
      _items.add(CartItem(
        product: product,
        quantity: quantity,
        unitPrice: product.price,
      ));
    }
    
    emit(CartUpdated(items: List.from(_items)));
    _calculateTotal();
  }
  
  void removeItem(String productId) {
    _items.removeWhere((item) => item.product.id == productId);
    emit(CartUpdated(items: List.from(_items)));
    _calculateTotal();
  }
  
  void updateQuantity(String productId, int quantity) {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }
    
    final index = _items.indexWhere((item) => item.product.id == productId);
    if (index >= 0) {
      _items[index] = _items[index].copyWith(quantity: quantity);
      emit(CartUpdated(items: List.from(_items)));
      _calculateTotal();
    }
  }
  
  void clearCart() {
    _items.clear();
    emit(CartUpdated(items: List.from(_items)));
    _calculateTotal();
  }
  
  void _calculateTotal() {
    final subtotal = _items.fold<double>(
      0, 
      (sum, item) => sum + (item.unitPrice * item.quantity) - item.discountAmount
    );
    
    final state = this.state;
    if (state is CartUpdated) {
      emit(state.copyWith(
        subtotal: subtotal,
        taxAmount: subtotal * 0.1, // 10% tax example
        discountAmount: 0, // Could be calculated based on promotions
        totalAmount: subtotal + (subtotal * 0.1),
      ));
    }
  }
  
  double getTotalAmount() {
    final state = this.state;
    if (state is CartUpdated) {
      return state.totalAmount;
    }
    return 0.0;
  }
  
  List<CartItem> getItems() => List.from(_items);
}

// Cart State
class CartState extends Equatable {
  const CartState();
  
  @override
  List<Object> get props => [];
}

class CartInitial extends CartState {}
class CartUpdated extends CartState {
  final List<CartItem> items;
  final double subtotal;
  final double taxAmount;
  final double discountAmount;
  final double totalAmount;
  
  const CartUpdated({
    required this.items,
    this.subtotal = 0,
    this.taxAmount = 0,
    this.discountAmount = 0,
    this.totalAmount = 0,
  });
  
  CartUpdated copyWith({
    List<CartItem>? items,
    double? subtotal,
    double? taxAmount,
    double? discountAmount,
    double? totalAmount,
  }) {
    return CartUpdated(
      items: items ?? this.items,
      subtotal: subtotal ?? this.subtotal,
      taxAmount: taxAmount ?? this.taxAmount,
      discountAmount: discountAmount ?? this.discountAmount,
      totalAmount: totalAmount ?? this.totalAmount,
    );
  }
  
  @override
  List<Object> get props => [items, subtotal, taxAmount, discountAmount, totalAmount];
}

// Cart Item Model
class CartItem {
  final Product product;
  final int quantity;
  final double unitPrice;
  final double discountAmount;
  
  const CartItem({
    required this.product,
    required this.quantity,
    required this.unitPrice,
    this.discountAmount = 0,
  });
  
  double get totalPrice => (unitPrice * quantity) - discountAmount;
  
  CartItem copyWith({
    Product? product,
    int? quantity,
    double? unitPrice,
    double? discountAmount,
  }) {
    return CartItem(
      product: product ?? this.product,
      quantity: quantity ?? this.quantity,
      unitPrice: unitPrice ?? this.unitPrice,
      discountAmount: discountAmount ?? this.discountAmount,
    );
  }
}
Step 5: Presentation Layer (UI/UX)
5.1 Main Dashboard Screen
dart
// lib/src/presentation/pages/dashboard/dashboard_page.dart
class DashboardPage extends StatelessWidget {
  const DashboardPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('POS Mobile'),
        actions: [
          IconButton(
            icon: const Icon(Icons.sync),
            onPressed: () => context.read<ProductBloc>().add(SyncProductsEvent()),
          ),
          IconButton(
            icon: const Icon(Icons.qr_code_scanner),
            onPressed: () => _scanBarcode(context),
          ),
        ],
      ),
      body: Column(
        children: [
          // Quick stats
          _buildStatsSection(context),
          
          // Products grid
          Expanded(
            child: BlocBuilder<ProductBloc, ProductState>(
              builder: (context, state) {
                if (state is ProductLoading) {
                  return const Center(child: CircularProgressIndicator());
                } else if (state is ProductError) {
                  return Center(child: Text('Error: ${state.message}'));
                } else if (state is ProductLoaded) {
                  return _buildProductsGrid(state.products, context);
                } else {
                  return const Center(child: Text('No products available'));
                }
              },
            ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showCart(context),
        child: Badge(
          badgeContent: BlocBuilder<CartCubit, CartState>(
            builder: (context, state) {
              final itemCount = state is CartUpdated ? state.items.length : 0;
              return Text(
                itemCount.toString(),
                style: const TextStyle(color: Colors.white, fontSize: 12),
              );
            },
          ),
          child: const Icon(Icons.shopping_cart),
        ),
      ),
      bottomNavigationBar: _buildBottomNavBar(),
    );
  }
  
  Widget _buildStatsSection(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.blue[50],
        borderRadius: BorderRadius.circular(12),
      ),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceAround,
        children: [
          _buildStatItem('Products', '150', Icons.inventory_2),
          _buildStatItem('Sales Today', '\$1,250', Icons.attach_money),
          _buildStatItem('Transactions', '25', Icons.receipt),
        ],
      ),
    );
  }
  
  Widget _buildStatItem(String title, String value, IconData icon) {
    return Column(
      children: [
        Icon(icon, size: 24, color: Colors.blue),
        const SizedBox(height: 4),
        Text(value, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
        Text(title, style: const TextStyle(fontSize: 12, color: Colors.grey)),
      ],
    );
  }
  
  Widget _buildProductsGrid(List<Product> products, BuildContext context) {
    return GridView.builder(
      padding: const EdgeInsets.all(8),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 8,
        mainAxisSpacing: 8,
        childAspectRatio: 0.8,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) {
        final product = products[index];
        return ProductCard(
          product: product,
          onTap: () => _addToCart(product, context),
        );
      },
    );
  }
  
  Widget _buildBottomNavBar() {
    return BottomNavigationBar(
      items: const [
        BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
        BottomNavigationBarItem(icon: Icon(Icons.point_of_sale), label: 'POS'),
        BottomNavigationBarItem(icon: Icon(Icons.analytics), label: 'Reports'),
        BottomNavigationBarItem(icon: Icon(Icons.settings), label: 'Settings'),
      ],
    );
  }
  
  void _addToCart(Product product, BuildContext context) {
    context.read<CartCubit>().addItem(product);
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('Added ${product.name} to cart'),
        duration: const Duration(seconds: 2),
      ),
    );
  }
  
  void _scanBarcode(BuildContext context) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => const BarcodeScannerPage(),
      ),
    );
  }
  
  void _showCart(BuildContext context) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => const CartPage(),
      ),
    );
  }
}
5.2 Product Card Widget
dart
// lib/src/presentation/widgets/product_card.dart
class ProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback onTap;
  
  const ProductCard({
    super.key,
    required this.product,
    required this.onTap,
  });
  
  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      child: InkWell(
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // Product image placeholder
              Container(
                height: 100,
                width: double.infinity,
                decoration: BoxDecoration(
                  color: Colors.grey[200],
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Icon(Icons.inventory_2, size: 40, color: Colors.grey[400]),
              ),
              
              const SizedBox(height: 8),
              
              // Product name
              Text(
                product.name,
                style: const TextStyle(
                  fontSize: 14,
                  fontWeight: FontWeight.bold,
                ),
                maxLines: 2,
                overflow: TextOverflow.ellipsis,
              ),
              
              // Product code
              Text(
                product.code,
                style: TextStyle(
                  fontSize: 12,
                  color: Colors.grey[600],
                ),
              ),
              
              const Spacer(),
              
              // Price and stock
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    '\$${product.price.toStringAsFixed(2)}',
                    style: const TextStyle(
                      fontSize: 16,
                      fontWeight: FontWeight.bold,
                      color: Colors.green,
                    ),
                  ),
                  
                  Container(
                    padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                    decoration: BoxDecoration(
                      color: product.stockQuantity > 10 
                          ? Colors.green[100] 
                          : Colors.orange[100],
                      borderRadius: BorderRadius.circular(4),
                    ),
                    child: Text(
                      '${product.stockQuantity} pcs',
                      style: TextStyle(
                        fontSize: 10,
                        color: product.stockQuantity > 10 
                            ? Colors.green[800] 
                            : Colors.orange[800],
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
5.3 Cart Screen
dart
// lib/src/presentation/pages/cart/cart_page.dart
class CartPage extends StatelessWidget {
  const CartPage({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shopping Cart'),
        actions: [
          IconButton(
            icon: const Icon(Icons.clear_all),
            onPressed: () => _clearCart(context),
          ),
        ],
      ),
      body: BlocBuilder<CartCubit, CartState>(
        builder: (context, state) {
          if (state is CartUpdated && state.items.isNotEmpty) {
            return Column(
              children: [
                // Cart items list
                Expanded(
                  child: ListView.builder(
                    itemCount: state.items.length,
                    itemBuilder: (context, index) {
                      final item = state.items[index];
                      return CartItemTile(item: item);
                    },
                  ),
                ),
                
                // Total section
                _buildTotalSection(state, context),
                
                // Checkout button
                _buildCheckoutButton(context),
              ],
            );
          } else {
            return const Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.shopping_cart_outlined, size: 64, color: Colors.grey),
                  SizedBox(height: 16),
                  Text('Your cart is empty', style: TextStyle(fontSize: 18)),
                ],
              ),
            );
          }
        },
      ),
    );
  }
  
  Widget _buildTotalSection(CartUpdated state, BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        border: Border(top: BorderSide(color: Colors.grey[300]!)),
      ),
      child: Column(
        children: [
          _buildTotalRow('Subtotal', state.subtotal),
          _buildTotalRow('Tax (10%)', state.taxAmount),
          _buildTotalRow('Discount', -state.discountAmount),
          const Divider(),
          _buildTotalRow('TOTAL', state.totalAmount, isTotal: true),
        ],
      ),
    );
  }
  
  Widget _buildTotalRow(String label, double amount, {bool isTotal = false}) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(
            label,
            style: TextStyle(
              fontSize: isTotal ? 16 : 14,
              fontWeight: isTotal ? FontWeight.bold : FontWeight.normal,
            ),
          ),
          Text(
            '\$${amount.toStringAsFixed(2)}',
            style: TextStyle(
              fontSize: isTotal ? 18 : 14,
              fontWeight: isTotal ? FontWeight.bold : FontWeight.normal,
              color: isTotal ? Colors.green : Colors.black,
            ),
          ),
        ],
      ),
    );
  }
  
  Widget _buildCheckoutButton(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: SizedBox(
        width: double.infinity,
        height: 50,
        child: ElevatedButton(
          onPressed: () => _checkout(context),
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.green,
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(8),
            ),
          ),
          child: const Text(
            'PROCEED TO CHECKOUT',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
        ),
      ),
    );
  }
  
  void _clearCart(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Clear Cart'),
        content: const Text('Are you sure you want to clear all items from the cart?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
          TextButton(
            onPressed: () {
              context.read<CartCubit>().clearCart();
              Navigator.pop(context);
            },
            child: const Text('Clear', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
  
  void _checkout(BuildContext context) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => const CheckoutPage(),
      ),
    );
  }
}
Step 6: SignalR Real-time Integration
6.1 SignalR Service
dart
// lib/src/services/signalr_service.dart
class SignalRService {
  late HubConnection _connection;
  final String _hubUrl;
  final String _accessToken;
  
  final _stockUpdatedController = StreamController<StockUpdatedEvent>.broadcast();
  final _newTransactionController = StreamController<NewTransactionEvent>.broadcast();
  final _priceUpdatedController = StreamController<PriceUpdatedEvent>.broadcast();
  
  SignalRService({required String hubUrl, required String accessToken})
      : _hubUrl = hubUrl,
        _accessToken = accessToken;
  
  Stream<StockUpdatedEvent> get stockUpdated => _stockUpdatedController.stream;
  Stream<NewTransactionEvent> get newTransaction => _newTransactionController.stream;
  Stream<PriceUpdatedEvent> get priceUpdated => _priceUpdatedController.stream;
  
  Future<void> connect() async {
    _connection = HubConnectionBuilder()
        .withUrl(
          _hubUrl,
          HttpConnectionOptions(
            accessTokenFactory: () => Future.value(_accessToken),
            logging: (level, message) => debugPrint(message),
          ),
        )
        .build();
    
    // Setup event handlers
    _connection.on('StockUpdated', _handleStockUpdated);
    _connection.on('NewTransaction', _handleNewTransaction);
    _connection.on('ProductPriceUpdated', _handlePriceUpdated);
    
    _connection.onclose((error) {
      debugPrint('SignalR connection closed: $error');
      // Implement reconnection logic
      _reconnect();
    });
    
    try {
      await _connection.start();
      debugPrint('SignalR connected successfully');
    } catch (e) {
      debugPrint('SignalR connection failed: $e');
      throw Exception('Failed to connect to real-time service');
    }
  }
  
  Future<void> joinStore(String storeId) async {
    try {
      await _connection.invoke('JoinStore', args: [storeId]);
      debugPrint('Joined store: $storeId');
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
      final event = StockUpdatedEvent(
        productId: data['productId'],
        newStock: data['newStock'],
        timestamp: DateTime.now(),
      );
      _stockUpdatedController.add(event);
    }
  }
  
  void _handleNewTransaction(List<dynamic>? arguments) {
    if (arguments != null && arguments.isNotEmpty) {
      final data = arguments[0];
      // Parse transaction data and add to stream
      final event = NewTransactionEvent.fromJson(data);
      _newTransactionController.add(event);
    }
  }
  
  void _handlePriceUpdated(List<dynamic>? arguments) {
    if (arguments != null && arguments.isNotEmpty) {
      final data = arguments[0];
      final event = PriceUpdatedEvent(
        productId: data['productId'],
        newPrice: data['newPrice'],
        timestamp: DateTime.now(),
      );
      _priceUpdatedController.add(event);
    }
  }
  
  Future<void> _reconnect() async {
    await Future.delayed(const Duration(seconds: 5));
    try {
      await connect();
    } catch (e) {
      debugPrint('Reconnection failed: $e');
    }
  }
  
  Future<void> disconnect() async {
    await _connection.stop();
    await _stockUpdatedController.close();
    await _newTransactionController.close();
    await _priceUpdatedController.close();
  }
}

// Event classes
class StockUpdatedEvent {
  final String productId;
  final int newStock;
  final DateTime timestamp;
  
  StockUpdatedEvent({
    required this.productId,
    required this.newStock,
    required this.timestamp,
  });
}

class NewTransactionEvent {
  final String transactionId;
  final double amount;
  final DateTime timestamp;
  
  NewTransactionEvent({
    required this.transactionId,
    required this.amount,
    required this.timestamp,
  });
  
  factory NewTransactionEvent.fromJson(Map<String, dynamic> json) {
    return NewTransactionEvent(
      transactionId: json['id'],
      amount: (json['totalAmount'] as num).toDouble(),
      timestamp: DateTime.parse(json['transactionDate']),
    );
  }
}
Step 7: Dependency Injection Setup
7.1 Service Locator dengan GetIt
dart
// lib/injection_container.dart
final getIt = GetIt.instance;

@InjectableInit()
void configureDependencies() => getIt.init();

@module
abstract class InjectionModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(
        baseUrl: AppConstants.apiBaseUrl,
        connectTimeout: AppConstants.connectionTimeout,
        receiveTimeout: AppConstants.connectionTimeout,
      ))
        ..interceptors.add(LogInterceptor(
          requestBody: true,
          responseBody: true,
        ))
        ..interceptors.add(InterceptorsWrapper(
          onRequest: (options, handler) {
            // Add auth token to requests
            final token = getIt<AuthService>().token;
            if (token != null) {
              options.headers['Authorization'] = 'Bearer $token';
            }
            return handler.next(options);
          },
          onError: (error, handler) {
            // Handle auth errors
            if (error.response?.statusCode == 401) {
              getIt<AuthService>().logout();
            }
            return handler.next(error);
          },
        ));
  
  @lazySingleton
  ApiService get apiService => ApiService(getIt<Dio>());
  
  @lazySingleton
  SignalRService get signalRService => SignalRService(
        hubUrl: AppConstants.signalRHubUrl,
        accessToken: getIt<AuthService>().token ?? '',
      );
  
  @lazySingleton
  ProductRepository get productRepository => ProductRepositoryImpl(
        remoteDataSource: ProductRemoteDataSourceImpl(getIt<ApiService>()),
        localDataSource: ProductLocalDataSourceImpl(),
        networkInfo: NetworkInfoImpl(),
      );
}

// main.dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Configure dependencies
  configureDependencies();
  
  // Initialize services
  getIt<AuthService>().initialize();
  
  runApp(const MyApp());
}
Step 8: Platform-Specific Configuration
8.1 Android Configuration
xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.VIBRATE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    
    <application
        android:name=".Application"
        android:label="POS Mobile"
        android:icon="@mipmap/ic_launcher"
        android:usesCleartextTraffic="true">
        
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">
            
            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme" />
                
            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>
</manifest>
8.2 iOS Configuration
xml
<!-- ios/Runner/Info.plist -->
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>$(DEVELOPMENT_LANGUAGE)</string>
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>POS Mobile</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleShortVersionString</key>
    <string>$(FLUTTER_BUILD_NAME)</string>
    <key>CFBundleVersion</key>
    <string>$(FLUTTER_BUILD_NUMBER)</string>
    <key>LSRequiresIPhoneOS</key>
    <true/>
    
    <!-- Permissions -->
    <key>NSCameraUsageDescription</key>
    <string>This app needs camera access to scan barcodes</string>
    
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <true/>
    </dict>
    
    <key>UIApplicationSceneManifest</key>
    <dict>
        <key>UIApplicationSupportsMultipleScenes</key>
        <false/>
        <key>UISceneConfigurations</key>
        <dict>
            <key>UIWindowScene</key>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>FlutterSceneDelegate</string>
            </dict>
        </dict>
    </dict>
</dict>
Step 9: Build dan Deployment
9.1 Build Commands
bash
# Build Android APK
flutter build apk --release

# Build Android App Bundle
flutter build appbundle --release

# Build iOS
flutter build ios --release

# Build for specific flavors
flutter build apk --release --flavor production
flutter build apk --release --flavor development
9.2 Continuous Integration
yaml
# .github/workflows/flutter.yml
name: Flutter CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.x'
        
    - name: Install dependencies
      run: flutter pub get
      
    - name: Run tests
      run: flutter test
      
    - name: Build APK
      run: flutter build apk --release
      
    - name: Upload APK
      uses: actions/upload-artifact@v3
      with:
        name: pos-mobile-apk
        path: build/app/outputs/flutter-apk/app-release.apk
Next Steps
Setelah Phase 5 complete, lanjut ke:

Phase 6: Deployment dan Production Setup

Phase 7: Testing dan Quality Assurance

Phase 8: Monitoring dan Maintenance

Mobile application sekarang ready untuk development dan testing! 🚀
