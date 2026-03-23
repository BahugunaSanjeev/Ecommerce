# Global E-Commerce Order Management Platform

A cloud-agnostic, microservices-based order management system designed to handle **10M+ orders/day** with **<1s latency** and support **100K+ concurrent users** during peak events like Black Friday.

---

## Architecture Overview


<img width="4221" height="3010" alt="Ecommerce_architecture" src="https://github.com/user-attachments/assets/1e4d4ea9-e986-4e78-94a2-d9f5806ff1c1" />

[Global_ECommerce_Order_Management_Platform_Architecture.docx](https://github.com/user-attachments/files/26178741/Global_ECommerce_Order_Management_Platform_Architecture.docx)

The platform follows a **database-per-service** microservices pattern deployed on **Kubernetes**, communicating asynchronously via **RabbitMQ** with resilience patterns powered by **Polly**.

---

## Microservices

| Service | Responsibility | Database | Key Pattern |
|---------|---------------|----------|-------------|
| **Product Catalog** | Browse products, categories, full-text search | MongoDB | Optimistic concurrency (ETag) |
| **Order Service** | Cart management, order creation, lifecycle | PostgreSQL + Redis | Saga pattern, event-driven |
| **Payment Service** | Process payments via Stripe/PayPal | PostgreSQL | Polly retry + circuit breaker |
| **Shipping Service** | Track shipments via FedEx/UPS APIs | MongoDB | Async event consumer |
| **Analytics Service** | Clickstreams, conversion rates, dashboards | ClickHouse | CQRS read projections |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Angular, Ionic (mobile) |
| **Backend** | .NET 8, ASP.NET Core Web API |
| **Messaging** | RabbitMQ (queues + DLQ) |
| **Databases** | PostgreSQL, MongoDB, ClickHouse |
| **Caching** | Redis (cart sessions, hot data) |
| **Search** | Elasticsearch |
| **Orchestration** | Kubernetes, Helm, Istio |
| **CI/CD** | GitHub Actions / Jenkins / GitLab CI |
| **Monitoring** | Prometheus, Grafana, Jaeger, ELK Stack |
| **IaC** | Terraform, Helm Charts |
| **Resilience** | Polly (retry, circuit breaker, timeout) |

---

## Resilience — Polly Patterns (Payment Service)

The Payment Service uses **Polly** to handle transient failures when communicating with Stripe/PayPal:

1. **Retry Policy** — 3 attempts with exponential backoff (1s, 2s, 4s) + jitter
2. **Circuit Breaker** — Opens after 5 consecutive failures, stays open for 30s, then half-open test
3. **Timeout** — 10s per attempt to prevent hanging calls
4. **Fallback** — Failed payments routed to Dead-Letter Queue (DLQ) for manual review

```
Retry (3x) → Circuit Breaker (5-fail) → Timeout (10s) → Fallback (DLQ)
```

---

## Security & Compliance (PCI-DSS)

| Control | Implementation |
|---------|---------------|
| **Data at Rest** | AES-256 encryption via DB-level TDE |
| **Data in Transit** | TLS 1.3 on all endpoints and inter-service communication |
| **Secrets Management** | HashiCorp Vault / Sealed Secrets |
| **Tokenization** | Card numbers never stored — delegated to Stripe/PayPal |
| **Authentication** | OAuth2 / OIDC with JWT tokens |
| **Authorization** | RBAC + Kubernetes Pod Security Standards |
| **Network** | Network Policies, service mesh (Istio) mTLS |
| **Compliance** | PCI-DSS Level 1, GDPR |

---

## Scalability — How We Meet NFRs

### 10M+ Orders/Day with <1s Latency

- **Horizontal Pod Autoscaling (HPA)** — Scales pods 2x–50x based on CPU and request rate; each pod handles ~2K TPS
- **Async Event-Driven (RabbitMQ)** — Orders dequeued in parallel; decouples write path from payment/shipping processing
- **Database Read Replicas + Sharding** — PostgreSQL read replicas for queries; MongoDB partitioned by category
- **Redis Caching** — Sub-millisecond reads; reduces DB load by ~80%; 24h TTL for cart sessions
- **CQRS Pattern** — Separate read/write models; reads served from optimized projections (<50ms p99)

### 100K+ Concurrent Users (Black Friday Spikes)

- **CDN + WAF** — Static assets served at edge; DDoS protection and bot mitigation
- **Kubernetes Cluster Autoscaler** — Automatically adds/removes nodes based on pending pod demand
- **RabbitMQ Backpressure** — Queues buffer 500K+ messages during burst traffic, consumers scale independently
- **Rate Limiting** — Per-user throttling enforced at the API Gateway layer
- **Circuit Breakers** — Isolate failing downstream dependencies to prevent cascade failures

---

## Project Structure

```
ECommercePlatform/
│
├── ECommercePlatform.sln                        # Visual Studio solution file
│
├── ProductCatalogService/                       # Microservice 1: Product & Category Management
│   ├── ProductCatalogService.csproj             #   Project file (.NET 8, NuGet packages)
│   ├── Program.cs                               #   Service entry point & DI configuration
│   ├── appsettings.json                         #   Configuration (Cosmos DB, Search, App Insights)
│   ├── Configuration/
│   │   └── CosmosDbSettings.cs                  #   Strongly-typed config: CosmosDbSettings, SearchSettings
│   ├── Models/
│   │   ├── Product.cs                           #   Product entity (17 properties, partition key)
│   │   └── Category.cs                          #   Category entity (hierarchical, sort order)
│   ├── DTOs/
│   │   └── ProductDtos.cs                       #   Records: CreateProductRequest, UpdateProductRequest,
│   │                                            #   ProductSearchRequest, ProductResponse, ProductListResponse,
│   │                                            #   CategoryResponse, InventoryUpdateRequest
│   ├── Repositories/
│   │   ├── IProductRepository.cs                #   Interfaces: IProductRepository, ICategoryRepository
│   │   └── CosmosProductRepository.cs           #   Cosmos DB implementation: CRUD, pagination,
│   │                                            #   ETag-based optimistic concurrency,
│   │                                            #   CosmosCategoryRepository, FeedIteratorExtensions
│   ├── Services/
│   │   ├── IProductService.cs                   #   Interfaces: IProductService, ICategoryService
│   │   ├── ProductService.cs                    #   Business logic: CRUD, search delegation, mapping
│   │   │                                        #   Also contains CategoryService implementation
│   │   ├── IProductSearchService.cs             #   Interface: SearchAsync, IndexProductAsync, RemoveProductAsync
│   │   └── AzureSearchService.cs                #   Full-text search: filters, sorting, pagination, indexing
│   └── Controllers/
│       ├── ProductsController.cs                #   REST API: GET/POST/PUT/DELETE/PATCH /api/v1/products
│       │                                        #   8 endpoints: GetById, GetAll, GetByCategory, Search,
│       │                                        #   Create, Update, Delete, UpdateInventory
│       └── CategoriesController.cs              #   REST API: GET/POST/DELETE /api/v1/categories
│                                                #   4 endpoints: GetAll, GetById, Create, Delete
│
├── OrderService/                                # Microservice 2: Cart & Order Management
│   ├── OrderService.csproj                      #   Project file (.NET 8, EF Core, Redis, Service Bus)
│   ├── Program.cs                               #   Service entry point & DI configuration
│   ├── appsettings.json                         #   Configuration (SQL, Redis, Service Bus)
│   ├── Configuration/
│   │   └── OrderSettings.cs                     #   Strongly-typed config: OrderSettings, ServiceBusSettings
│   ├── Models/
│   │   ├── Order.cs                             #   Order entity (OrderStatus enum, line items, totals)
│   │   └── Cart.cs                              #   Cart & CartItem models (Redis-backed, 24h TTL)
│   ├── DTOs/
│   │   └── OrderDtos.cs                         #   Records: CreateOrderRequest, OrderResponse,
│   │                                            #   CartItemRequest, CartResponse
│   ├── Data/
│   │   └── OrderDbContext.cs                    #   EF Core DbContext with SQL Server / PostgreSQL
│   ├── Events/
│   │   └── OrderEvents.cs                       #   Integration events: OrderCreated, OrderConfirmed,
│   │                                            #   OrderCancelled (published to RabbitMQ)
│   ├── Services/
│   │   ├── IOrderService.cs                     #   Interface: CreateOrder, GetOrder, UpdateStatus
│   │   ├── OrderServiceImpl.cs                  #   Saga orchestration, event publishing, status mgmt
│   │   ├── ICartService.cs                      #   Interface: AddItem, RemoveItem, GetCart, ClearCart
│   │   ├── CartService.cs                       #   Redis-backed cart with 24h TTL
│   │   ├── IEventPublisher.cs                   #   Interface: PublishAsync<T>
│   │   └── ServiceBusEventPublisher.cs          #   RabbitMQ / Service Bus event publisher
│   └── Controllers/
│       ├── OrdersController.cs                  #   REST API: GET/POST/PUT /api/v1/orders
│       └── CartController.cs                    #   REST API: GET/POST/DELETE /api/v1/cart
│
├── PaymentService/                              # Microservice 3: Payment Processing (PCI-DSS)
│   ├── PaymentService.csproj                    #   Project file (.NET 8, EF Core, Polly, Stripe, PayPal)
│   ├── Program.cs                               #   Service entry point, DI, Polly policy registration
│   ├── appsettings.json                         #   Configuration (SQL, Stripe, PayPal, Service Bus)
│   ├── Configuration/
│   │   └── PaymentSettings.cs                   #   Strongly-typed config: StripeSettings, PayPalSettings
│   ├── Models/
│   │   └── Payment.cs                           #   Payment entity (PaymentStatus enum, provider, amounts)
│   ├── DTOs/
│   │   └── PaymentDtos.cs                       #   Records: ProcessPaymentRequest, PaymentResponse,
│   │                                            #   RefundRequest, WebhookPayload
│   ├── Adapters/
│   │   ├── IPaymentProviderAdapter.cs           #   Interface: ProcessAsync, RefundAsync, ValidateWebhook
│   │   ├── StripePaymentAdapter.cs              #   Stripe API: PaymentIntent, 3D Secure, refunds
│   │   └── PayPalPaymentAdapter.cs              #   PayPal REST v2: OAuth2 token, capture flow, refunds
│   ├── Resilience/
│   │   └── ResiliencePolicies.cs                #   Polly: Retry (3x exponential backoff + jitter),
│   │                                            #   Circuit Breaker (5-fail, 30s open),
│   │                                            #   Timeout (10s), composed PolicyWrap
│   ├── Data/
│   │   └── PaymentDbContext.cs                  #   EF Core DbContext with SQL Server / PostgreSQL
│   ├── Services/
│   │   ├── IPaymentService.cs                   #   Interface: ProcessPayment, GetPayment, Refund
│   │   ├── PaymentServiceImpl.cs                #   Core payment logic with Polly-wrapped provider calls
│   │   ├── IPaymentEventPublisher.cs            #   Interface: PublishPaymentCompleted/Failed
│   │   ├── ServiceBusPaymentEventPublisher.cs   #   Publishes payment events to RabbitMQ
│   │   └── OrderCreatedEventConsumer.cs         #   BackgroundService: listens to OrderCreated events,
│   │                                            #   triggers payment, routes to DLQ on failure
│   └── Controllers/
│       └── PaymentsController.cs                #   REST API: POST /api/v1/payments
│                                                #   5 endpoints: Process, GetById, Refund,
│                                                #   StripeWebhook, PayPalWebhook
│
└── docs/                                        # Documentation (optional)
    ├── architecture-diagram.png                 #   Cloud-agnostic architecture diagram
    └── architecture.drawio                      #   Editable draw.io file
```

### File Count Summary

| Service | Files | Lines of Code (approx.) |
|---------|-------|------------------------|
| **ProductCatalogService** | 12 files | ~600 LOC |
| **OrderService** | 14 files | ~700 LOC |
| **PaymentService** | 15 files | ~800 LOC |
| **Solution** | 1 file | — |
| **Total** | **42 files** | **~2,100 LOC** |

---

## Sample Microservice Deep-Dive: Product Catalog Service

### Overview

The Product Catalog Service manages the product inventory and categories for the e-commerce platform. It provides full-text search, pagination, and optimistic concurrency control for inventory updates.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      Product Catalog Service                             │
│                                                                          │
│  ┌───────────────┐    ┌───────────────┐    ┌──────────────────────┐     │
│  │  Controllers   │───>│   Services    │───>│    Repositories      │     │
│  │               │    │               │    │                      │     │
│  │ Products      │    │ ProductSvc    │    │ CosmosProductRepo    │────────> MongoDB /
│  │ Categories    │    │ CategorySvc   │    │ CosmosCategoryRepo   │         Cosmos DB
│  └───────────────┘    │ SearchSvc     │    └──────────────────────┘     │
│                       └───────┬───────┘                                  │
│                               │                                          │
│                       ┌───────▼────────┐                                 │
│                       │  Elasticsearch  │                                 │
│                       │  (Full-text)    │                                 │
│                       └────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

### Architecture Layers

| Layer | Responsibility | Files |
|-------|---------------|-------|
| **Controllers** | HTTP request handling, validation, routing | `ProductsController.cs`, `CategoriesController.cs` |
| **DTOs** | Request/response contracts (immutable records) | `ProductDtos.cs` |
| **Services** | Business logic, orchestration, mapping | `ProductService.cs`, `AzureSearchService.cs` |
| **Repositories** | Data access abstraction (Cosmos DB / MongoDB) | `CosmosProductRepository.cs` |
| **Models** | Domain entities | `Product.cs`, `Category.cs` |
| **Configuration** | Strongly-typed settings classes | `CosmosDbSettings.cs` |

### API Endpoints

#### Products API (`/api/v1/products`)

| Method | Endpoint | Description | Response |
|--------|----------|-------------|----------|
| `GET` | `/api/v1/products` | List all products (paginated) | `ProductListResponse` |
| `GET` | `/api/v1/products/{id}?categoryId=` | Get product by ID | `ProductResponse` |
| `GET` | `/api/v1/products/category/{categoryId}` | Get products by category | `ProductListResponse` |
| `GET` | `/api/v1/products/search?query=&minPrice=&maxPrice=` | Full-text search with filters | `ProductListResponse` |
| `POST` | `/api/v1/products` | Create new product | `201 Created` |
| `PUT` | `/api/v1/products/{id}?categoryId=` | Update product | `ProductResponse` |
| `DELETE` | `/api/v1/products/{id}?categoryId=` | Delete product | `204 No Content` |
| `PATCH` | `/api/v1/products/{id}/inventory?categoryId=` | Update inventory (atomic) | `200 OK` |

#### Categories API (`/api/v1/categories`)

| Method | Endpoint | Description | Response |
|--------|----------|-------------|----------|
| `GET` | `/api/v1/categories` | List all categories | `List<CategoryResponse>` |
| `GET` | `/api/v1/categories/{id}` | Get category by ID | `CategoryResponse` |
| `POST` | `/api/v1/categories` | Create new category | `201 Created` |
| `DELETE` | `/api/v1/categories/{id}` | Delete category | `204 No Content` |

### Data Models

#### Product Entity

```csharp
public class Product
{
    public string Id { get; set; }              // Auto-generated GUID
    public string Name { get; set; }            // Product name
    public string Description { get; set; }     // Product description
    public string Sku { get; set; }             // Stock Keeping Unit
    public string CategoryId { get; set; }      // Foreign key to Category
    public string CategoryName { get; set; }    // Denormalized for read performance
    public decimal Price { get; set; }          // Price in specified currency
    public string Currency { get; set; }        // Default: "USD"
    public List<string> ImageUrls { get; set; } // Product image URLs
    public Dictionary<string, string> Attributes { get; set; }  // Custom key-value pairs
    public int InventoryCount { get; set; }     // Available stock quantity
    public bool IsActive { get; set; }          // Soft delete flag
    public DateTime CreatedAt { get; set; }     // UTC creation timestamp
    public DateTime UpdatedAt { get; set; }     // UTC last-modified timestamp
    public List<string> Tags { get; set; }      // Searchable tags
    public string PartitionKey => CategoryId;   // Cosmos DB partition key
}
```

#### Category Entity

```csharp
public class Category
{
    public string Id { get; set; }                  // Auto-generated GUID
    public string Name { get; set; }                // Category name
    public string Description { get; set; }         // Category description
    public string? ParentCategoryId { get; set; }   // Hierarchical parent (nullable for root)
    public string? ImageUrl { get; set; }           // Category image
    public bool IsActive { get; set; }              // Soft delete flag
    public int SortOrder { get; set; }              // Display ordering
    public string PartitionKey => "category";       // Fixed partition key
}
```

### Key Design Patterns

#### 1. Repository Pattern

The data access layer is abstracted behind interfaces, making it easy to swap MongoDB for any other database:

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(string id, string categoryId);
    Task<(List<Product>, int TotalCount)> GetByCategoryAsync(string categoryId, int page, int pageSize);
    Task<(List<Product>, int TotalCount)> GetAllAsync(int page, int pageSize, bool activeOnly = true);
    Task<Product> CreateAsync(Product product);
    Task<Product> UpdateAsync(Product product);
    Task DeleteAsync(string id, string categoryId);
    Task<bool> UpdateInventoryAsync(string id, string categoryId, int quantityChange);
}
```

#### 2. Optimistic Concurrency (ETag)

Inventory updates use ETag-based optimistic concurrency to prevent race conditions when multiple requests try to update the same product's stock simultaneously:

```csharp
public async Task<bool> UpdateInventoryAsync(string id, string categoryId, int quantityChange)
{
    const int maxRetries = 5;
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        // 1. Read current product + ETag
        var response = await _container.ReadItemAsync<Product>(id, new PartitionKey(categoryId));
        var product = response.Resource;
        var etag = response.ETag;

        // 2. Validate inventory
        var newCount = product.InventoryCount + quantityChange;
        if (newCount < 0) return false;  // Insufficient stock

        // 3. Update with ETag condition (fails if document changed since read)
        product.InventoryCount = newCount;
        var options = new ItemRequestOptions { IfMatchEtag = etag };
        await _container.ReplaceItemAsync(product, id, new PartitionKey(categoryId), options);
        return true;
    }
    return false;  // All retries exhausted
}
```

**How it works:**
- Read the product and capture its ETag (version token)
- Modify the inventory count
- Write back with `IfMatchEtag` — if another request modified the document between read and write, Cosmos DB returns `412 Precondition Failed`
- Retry with incremental backoff (up to 5 attempts, 50ms * attempt delay)

#### 3. Full-Text Search (Elasticsearch / Cognitive Search)

Product creation and updates trigger async indexing in the search engine:

```csharp
// Fire-and-forget indexing after product creation
var created = await _productRepo.CreateAsync(product);
_ = Task.Run(() => _searchService.IndexProductAsync(created));
```

Search supports:
- **Full-text query** across name, description, tags
- **Filters**: category, price range (`minPrice`, `maxPrice`), in-stock status
- **Sorting**: by any field (asc/desc)
- **Pagination**: `page` + `pageSize` with total count

#### 4. Dependency Injection (Program.cs)

```csharp
// Cosmos DB Client (singleton — thread-safe, connection-pooled)
builder.Services.AddSingleton(sp => {
    var settings = builder.Configuration.GetSection("CosmosDb").Get<CosmosDbSettings>()!;
    return new CosmosClient(settings.ConnectionString, new CosmosClientOptions {
        ConnectionMode = ConnectionMode.Direct,
        SerializerOptions = new CosmosSerializationOptions {
            PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
        }
    });
});

// Repository layer (scoped — one per HTTP request)
builder.Services.AddScoped<IProductRepository, CosmosProductRepository>();
builder.Services.AddScoped<ICategoryRepository, CosmosCategoryRepository>();

// Service layer (scoped)
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<ICategoryService, CategoryService>();
builder.Services.AddScoped<IProductSearchService, AzureSearchService>();
```

### Configuration (`appsettings.json`)

```json
{
  "CosmosDb": {
    "ConnectionString": "AccountEndpoint=https://...;AccountKey=...;",
    "DatabaseName": "ProductCatalog",
    "ProductContainerName": "Products",
    "CategoryContainerName": "Categories"
  },
  "Search": {
    "Endpoint": "https://your-search-endpoint",
    "ApiKey": "your-search-api-key",
    "IndexName": "products-index"
  }
}
```

> **Production:** Replace with environment variables or HashiCorp Vault references. Never commit secrets.

### NuGet Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `Microsoft.Azure.Cosmos` | 3.37.1 | Cosmos DB / MongoDB API client |
| `Azure.Search.Documents` | 11.5.1 | Elasticsearch / Cognitive Search client |
| `Swashbuckle.AspNetCore` | 6.5.0 | Swagger / OpenAPI documentation |
| `Serilog.AspNetCore` | 8.0.2 | Structured logging |
| `Microsoft.ApplicationInsights.AspNetCore` | 2.22.0 | Application monitoring (Prometheus alternative) |

### Health Check

```
GET /health → 200 OK (healthy) | 503 Service Unavailable (unhealthy)
```

Used by Kubernetes liveness/readiness probes:

```yaml
# Kubernetes deployment excerpt
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 15
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Database Initialization

On startup, the service auto-creates the database and containers if they don't exist:

```csharp
var database = await cosmosClient.CreateDatabaseIfNotExistsAsync(settings.DatabaseName);
await database.Database.CreateContainerIfNotExistsAsync(settings.ProductContainerName, "/partitionKey");
await database.Database.CreateContainerIfNotExistsAsync(settings.CategoryContainerName, "/partitionKey");
```

**Partition Strategy:**
- **Products**: Partitioned by `categoryId` — queries within a category hit a single partition (fast, low RU cost)
- **Categories**: Fixed partition key `"category"` — small dataset, all in one partition

---

## Getting Started

### Prerequisites

- .NET 8 SDK
- Docker & Docker Compose
- Kubernetes cluster (minikube for local, or any cloud K8s)
- Helm 3.x

### Run Locally

```bash
# Clone the repository
git clone https://github.com/<your-org>/ECommercePlatform.git
cd ECommercePlatform

# Restore and build all services
dotnet restore
dotnet build

# Run individual services
dotnet run --project ProductCatalogService
dotnet run --project OrderService
dotnet run --project PaymentService
```

### Deploy to Kubernetes

```bash
# Build Docker images
docker build -t ecommerce/catalog-service ./ProductCatalogService
docker build -t ecommerce/order-service ./OrderService
docker build -t ecommerce/payment-service ./PaymentService

# Push to container registry
docker push <registry>/ecommerce/catalog-service
docker push <registry>/ecommerce/order-service
docker push <registry>/ecommerce/payment-service

# Deploy with Helm
helm upgrade --install ecommerce ./helm-charts \
  --namespace ecommerce \
  --create-namespace
```

---

## CI/CD Pipeline

```
Developer Push → CI Server → Build & Unit Tests → Docker Build & Push
→ Container Registry → Helm Charts & kubectl → K8s Cluster → Health Checks & Rollback
```

The pipeline supports **GitHub Actions**, **Jenkins**, and **GitLab CI** — configure your preferred CI server in the `.github/workflows/` or `Jenkinsfile`.

---

## Configuration

Each service uses `appsettings.json` for configuration. Key settings:

| Setting | Description |
|---------|-------------|
| `ConnectionStrings:*` | Database connection strings |
| `RabbitMQ:Host` | RabbitMQ broker hostname |
| `Redis:ConnectionString` | Redis cache connection |
| `Stripe:SecretKey` | Stripe API secret key |
| `PayPal:ClientId` | PayPal API client ID |

> **Note:** Never commit secrets. Use environment variables, Kubernetes Secrets, or HashiCorp Vault in production.

---

