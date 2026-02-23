# 2. High-Level Architecture

## 🏗 System Architecture Diagram

```mermaid
graph TB
    subgraph "Clients"
        WEB["🌐 Web App<br/>(React/Next.js)"]
        MOB["📱 Mobile App<br/>(iOS/Android)"]
        SEL["🏪 Seller Dashboard"]
    end

    subgraph "Edge Layer"
        CDN["☁️ CDN<br/>(CloudFlare/CloudFront)<br/>Static assets + images"]
        LB["⚖️ Load Balancer<br/>(L7 — ALB/Nginx)<br/>SSL termination<br/>Rate limiting"]
    end

    subgraph "API Gateway"
        GW["🚪 API Gateway<br/>Auth, routing, throttling<br/>Request validation"]
    end

    subgraph "Application Services"
        PS["📦 Product Service<br/>Catalog CRUD<br/>Search proxy"]
        CS["🛒 Cart Service<br/>Cart management<br/>Price validation"]
        OS["📋 Order Service<br/>Order lifecycle<br/>Saga orchestrator"]
        PAY["💳 Payment Service<br/>Payment processing<br/>Idempotency"]
        US["👤 User Service<br/>Auth, profiles<br/>Sessions"]
        NS["🔔 Notification Service<br/>Email, push, SMS"]
        SS["🔍 Search Service<br/>Full-text search<br/>Filters & facets"]
    end

    subgraph "Async Layer"
        MQ["📨 Message Queue<br/>(RabbitMQ/SQS)<br/>Order events<br/>Email jobs"]
        WK["⚙️ Workers<br/>Order processing<br/>Email sending<br/>Analytics"]
    end

    subgraph "Data Layer"
        DB_P[("🔵 MySQL Primary<br/>Writes")]
        DB_R1[("🔵 MySQL Replica 1<br/>Reads")]
        DB_R2[("🔵 MySQL Replica 2<br/>Reads")]
        REDIS[("🔴 Redis Cluster<br/>Cache + Sessions<br/>+ Cart + Rate Limits")]
        ES[("🟡 Elasticsearch<br/>Product search index")]
        S3[("🟢 S3 / Object Storage<br/>Product images")]
    end

    WEB & MOB & SEL --> CDN
    CDN --> LB
    LB --> GW

    GW --> PS & CS & OS & PAY & US & SS

    PS --> DB_P & DB_R1 & REDIS & ES
    CS --> REDIS
    OS --> DB_P & MQ
    PAY --> DB_P
    US --> DB_P & DB_R2 & REDIS
    NS --> MQ
    SS --> ES

    MQ --> WK
    WK --> DB_P & NS

    PS -.->|image upload| S3
    CDN -.->|serve images| S3

    DB_P -->|replication| DB_R1 & DB_R2

    style DB_P fill:#1976d2,color:#fff
    style DB_R1 fill:#42a5f5,color:#fff
    style DB_R2 fill:#42a5f5,color:#fff
    style REDIS fill:#dc382d,color:#fff
    style ES fill:#f9a825,color:#000
    style S3 fill:#4caf50,color:#fff
    style MQ fill:#ff9800,color:#fff
    style CDN fill:#9c27b0,color:#fff
```

---

## 🚪 API Design

### Core API Endpoints

#### Product Service

```
GET    /api/v1/products                    # List products (paginated, filtered)
GET    /api/v1/products/:id                # Get product detail
POST   /api/v1/products                    # Create product (seller)
PUT    /api/v1/products/:id                # Update product (seller)
DELETE /api/v1/products/:id                # Soft-delete product (seller)
GET    /api/v1/products/:id/reviews        # Get product reviews
POST   /api/v1/products/:id/reviews        # Add review (buyer, post-purchase)
GET    /api/v1/categories                  # Get category tree
GET    /api/v1/search?q=...&category=...   # Full-text search with filters
```

#### Cart Service

```
GET    /api/v1/cart                         # Get current cart
POST   /api/v1/cart/items                   # Add item to cart
PUT    /api/v1/cart/items/:sku_id           # Update quantity
DELETE /api/v1/cart/items/:sku_id           # Remove item
DELETE /api/v1/cart                         # Clear cart
```

#### Order Service

```
POST   /api/v1/orders                       # Create order (checkout)
GET    /api/v1/orders                       # List user's orders
GET    /api/v1/orders/:id                   # Get order detail
PUT    /api/v1/orders/:id/cancel            # Cancel order (if eligible)
GET    /api/v1/seller/orders                # Seller: list orders to fulfill
PUT    /api/v1/seller/orders/:id/ship       # Seller: mark as shipped
```

#### Payment Service

```
POST   /api/v1/payments                     # Initiate payment
GET    /api/v1/payments/:id                 # Get payment status
POST   /api/v1/payments/:id/webhook         # Payment gateway callback
```

#### User Service

```
POST   /api/v1/auth/register                # Register
POST   /api/v1/auth/login                   # Login (returns JWT)
POST   /api/v1/auth/refresh                 # Refresh token
GET    /api/v1/users/me                     # Get profile
PUT    /api/v1/users/me                     # Update profile
```

### API Response Format

```json
{
  "status": "success",
  "data": {
    "id": "prod_abc123",
    "title": "Wireless Bluetooth Headphones",
    "price": 2499,
    "currency": "EGP",
    "stock": 42,
    "seller": {
      "id": "seller_xyz",
      "name": "TechStore Egypt",
      "rating": 4.7
    },
    "images": [
      "https://cdn.marketplace.com/products/prod_abc123/1.webp",
      "https://cdn.marketplace.com/products/prod_abc123/2.webp"
    ]
  },
  "meta": {
    "request_id": "req_7f8a9b2c",
    "timestamp": "2026-02-23T10:30:00Z"
  }
}
```

> 💡 **Design decisions**:
> - Prices in **smallest currency unit** (cents/piasters) — avoids floating-point issues
> - All responses include **request_id** for tracing
> - Image URLs point to **CDN**, not the API server
> - Seller info **embedded** in product response to avoid N+1 frontend calls

---

## 🔀 Service Boundaries — Why These Services?

| Service | Owns | Why Separate? |
|---------|------|---------------|
| **Product Service** | Products, SKUs, Categories, Reviews | Highest traffic — needs independent scaling |
| **Cart Service** | Cart, CartItems | Redis-backed, stateful per user, very different access pattern |
| **Order Service** | Orders, OrderItems | Transactional, needs saga orchestration |
| **Payment Service** | Payments, Refunds | Highest security requirements, PCI compliance scope |
| **User Service** | Users, Auth, Sessions | Shared dependency — must be highly available |
| **Search Service** | ES index | Different tech stack, own scaling characteristics |
| **Notification Service** | Notification logs | Purely async, fire-and-forget, own rate limits |

### Service Communication

```mermaid
graph LR
    subgraph "Synchronous (REST/gRPC)"
        A["Product Service"] -->|GET product| B["Cart Service"]
        B -->|validate prices| A
        C["Order Service"] -->|create payment| D["Payment Service"]
        C -->|lock inventory| A
    end

    subgraph "Asynchronous (Message Queue)"
        C -->|order.created| E["Notification Service"]
        D -->|payment.completed| C
        C -->|order.confirmed| A
        A -->|product.updated| F["Search Service"]
    end

    style A fill:#2196f3,color:#fff
    style B fill:#4caf50,color:#fff
    style C fill:#ff9800,color:#fff
    style D fill:#f44336,color:#fff
    style E fill:#9c27b0,color:#fff
    style F fill:#ffc107,color:#000
```

### ✅ Decision: Synchronous for Reads, Async for Side Effects

| Communication | Pattern | Why |
|--------------|---------|-----|
| Cart → Product (validate price) | **Sync** (REST) | User is waiting, needs immediate response |
| Order → Payment (charge) | **Sync** (REST) | Checkout flow — user must know result |
| Order → Notification (send email) | **Async** (Queue) | User doesn't need to wait for email |
| Product update → Search index | **Async** (Queue) | Eventual consistency is fine (seconds-delay OK) |
| Payment webhook → Order status | **Async** (Queue) | Decouples payment gateway from order logic |

> ⚖️ **Trade-off**: Async means eventual consistency. Product search results may be stale by 1-5 seconds after a product update. This is acceptable for a marketplace.

> **⚠️ Mitigation: Synchronous Coupling on Checkout Path**
>
> The checkout path's synchronous dependency on Product Service (for price verification) and Cart Service (for item retrieval) is managed with: (1) **timeout budgets** — each sync call has a 500ms budget; if exceeded, the call fails fast rather than blocking checkout, (2) **circuit breakers** — after 5 consecutive failures to Product Service, the circuit opens and checkout uses the last-known cached price (with a "price may have changed" flag), (3) **bulkhead pattern** — checkout's connection pool to Product Service is isolated from browse/search traffic, preventing a search traffic spike from exhausting checkout's connections.

---

## 🚦 Request Flow: Product Page Load

```mermaid
sequenceDiagram
    participant C as Client
    participant CDN as CDN
    participant LB as Load Balancer
    participant GW as API Gateway
    participant PS as Product Service
    participant R as Redis Cache
    participant DB as MySQL (Replica)
    participant ES as Elasticsearch

    C->>CDN: GET /products/abc123
    CDN-->>C: (cache HIT → return images, static)

    C->>LB: GET /api/v1/products/abc123
    LB->>GW: Route request
    GW->>GW: Validate JWT, rate limit
    GW->>PS: Forward to Product Service

    PS->>R: GET cache:product:abc123
    alt Cache HIT (95% of the time)
        R-->>PS: Product JSON
        PS-->>GW: 200 OK + product data
    else Cache MISS (5%)
        R-->>PS: nil
        PS->>DB: SELECT * FROM products WHERE id = 'abc123'
        DB-->>PS: Product row
        PS->>R: SET cache:product:abc123 (TTL: 5 min)
        PS-->>GW: 200 OK + product data
    end

    GW-->>LB: Response
    LB-->>C: Product data (< 200ms)

    Note over C,ES: Search queries go directly to Elasticsearch
    C->>LB: GET /api/v1/search?q=headphones
    LB->>GW: Route
    GW->>PS: Forward
    PS->>ES: Search query
    ES-->>PS: Results
    PS-->>C: Search results (< 300ms)
```

---

## 🚦 Request Flow: Checkout

```mermaid
sequenceDiagram
    participant C as Client
    participant OS as Order Service
    participant CS as Cart Service
    participant PS as Product Service
    participant PAY as Payment Service
    participant MQ as Message Queue
    participant NS as Notification Service

    C->>OS: POST /api/v1/orders
    OS->>CS: Get cart items
    CS-->>OS: Cart items + quantities

    OS->>PS: Lock inventory (reserve stock)
    PS-->>OS: Stock reserved ✅

    OS->>OS: Create order (status: pending_payment)
    OS->>PAY: POST /payments (order_id, amount)
    PAY->>PAY: Idempotency check (order_id)
    PAY-->>OS: Payment initiated (redirect URL)
    OS-->>C: 200 OK {redirect: payment_url}

    Note over C,PAY: Customer completes payment on gateway

    PAY->>MQ: payment.completed event
    MQ->>OS: Consume payment event
    OS->>OS: Update order → confirmed
    OS->>PS: Confirm inventory deduction
    OS->>MQ: order.confirmed event
    MQ->>NS: Send confirmation email
    MQ->>NS: Send seller notification

    NS-->>C: 📧 Order confirmation email
```

---

## 🏛 Why Not Microservices on Day 1?

> ⚠️ **Common interview mistake**: Immediately jumping to microservices.

### Recommended Evolution Path

```
Day 1 (0–100K users):
  → Modular Monolith
  → Single Laravel/Django app with clear module boundaries
  → 1 MySQL + 1 Redis + 1 Elasticsearch
  → Deploy: 2 app servers behind LB
  → Cost: ~$500/month

Growth (100K–1M users):
  → Extract Payment Service (PCI compliance)
  → Add read replicas
  → Add worker queue (async processing)
  → Cost: ~$3,000/month

Scale (1M–10M users):
  → Extract Cart Service (Redis-backed)
  → Extract Search Service (Elasticsearch-specific)
  → Add CDN for images
  → Cost: ~$15,000/month

Full Scale (10M+ users):
  → Full service decomposition
  → Sharding for orders/products
  → Multi-region deployment
  → Cost: ~$33,000+/month
```

### ✅ Decision: Start Modular, Extract When Needed

| Approach | Pros | Cons |
|----------|------|------|
| **Microservices Day 1** | Clean boundaries | Operational complexity, network latency, debugging nightmares |
| **Monolith forever** | Simple | Coupling grows, can't scale independently |
| **Modular Monolith → Extract** ✅ | Best of both — clear modules, extract when pain is real | Need discipline to keep modules separate |

> ⚖️ **Trade-off**: We accept short-term coupling in exchange for faster development and simpler operations. We extract services when the **pain of coupling exceeds the pain of distribution**.

---

## ⬅️ [← Capacity Estimation](01-capacity-estimation.md) · [Data Model →](03-data-model.md)
