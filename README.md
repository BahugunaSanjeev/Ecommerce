# Global E-Commerce Order Management Platform

A cloud-agnostic, microservices-based order management system designed to handle **10M+ orders/day** with **<1s latency** and support **100K+ concurrent users** during peak events like Black Friday.

---

## Architecture Overview


<img width="4221" height="3010" alt="Ecommerce_architecture" src="https://github.com/user-attachments/assets/1e4d4ea9-e986-4e78-94a2-d9f5806ff1c1" />

Cloud Architecture using Microsoft Azure
<img width="5268" height="2636" alt="image" src="https://github.com/user-attachments/assets/e9923c57-af46-486c-bedc-e89879d61d56" />


[Global_ECommerce_Order_Management_Platform_Architecture.docx](https://github.com/user-attachments/files/26178741/Global_ECommerce_Order_Management_Platform_Architecture.docx)

The platform follows a **database-per-service** microservices pattern deployed on **Kubernetes**, communicating asynchronously via **RabbitMQ** with resilience patterns powered by **Polly**.

---

## Microservices

| Service | Responsibility | Database/ Message Broker | Key Pattern |
|---------|---------------|----------|-------------|
| **Product Catalog** | Browse products, categories, full-text search | No-SQL DB MongoDB/Cosmos | Optimistic concurrency (ETag) |
| **Order Service** | Cart management, order creation, lifecycle |  No-SQL DB MongoDB/Cosmos + Redis | Saga pattern, event-driven |
| **Payment Service** | Process payments via Stripe/PayPal | PostgreSQL/MS Sql/RRabbitMQ/Azure Service Bus | Polly retry + circuit breaker |
| **Shipping Service** | Track shipments via FedEx/UPS APIs |  No-SQL DB MongoDB/Cosmos | Async event consumer |
| **Analytics Service** | dashboards | Snowflake | CQRS read projections |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | Angular|
| **Backend** | .NET 8, ASP.NET Core Web API |
| **Messaging** | RabbitMQ (queues + DLQ) |
| **Databases** | PostgreSQL,  No-SQL DB MongoDB/CosmosMongoDB |
| **Caching** | Redis (cart sessions, hot data) |
| **Search** | Elasticsearch |
| **Orchestration** | Kubernetes, Helm, Istio |
| **CI/CD** | GitHub Actions / Jenkins / GitLab CI |
| **Monitoring** | Splunk DDynatrac, Grafana |
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
├── ProductCatalogService/
│   ├── Controllers/          # Products & Categories APIs
│   ├── Models/               # Product, Category entities
│   ├── DTOs/                 # Request/Response DTOs
│   ├── Repositories/         # MongoDB/Cosmos DB repository
│   ├── Services/             # Business logic + search
│   └── Program.cs            # Service entry point
├── OrderService/
│   ├── Controllers/          # Orders & Cart APIs
│   ├── Models/               # Order, CartItem entities
│   ├── DTOs/                 # Request/Response DTOs
│   ├── Data/                 # EF Core DbContext
│   ├── Services/             # Order logic + Saga orchestration
│   └── Program.cs
├── PaymentService/
│   ├── Controllers/          # Payments & Webhooks APIs
│   ├── Models/               # Payment entity
│   ├── DTOs/                 # Request/Response DTOs
│   ├── Adapters/             # Stripe & PayPal adapters
│   ├── Resilience/           # Polly policies (retry, circuit breaker, timeout)
│   ├── Services/             # Payment processing + event consumer
│   ├── Data/                 # EF Core DbContext
│   └── Program.cs
└── ECommercePlatform.sln     # Visual Studio solution file
---

## Sample Microservice Deep-Dive: Product Catalog Service


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

## CI/CD Pipeline

```
Developer Push → CI Server → Build & Unit Tests → Docker Build & Push
→ Container Registry → Helm Charts & kubectl → K8s Cluster → Health Checks & Rollback
```

The pipeline supports **GitHub Actions**, **Jenkins**, and **GitLab CI** — configure your preferred CI server in the `.github/workflows/` or `Jenkinsfile`.

---


> **Note:** Never commit secrets. Use environment variables, Kubernetes Secrets, or HashiCorp Vault in production.
---

