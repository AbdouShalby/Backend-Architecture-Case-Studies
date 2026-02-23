# 1. Capacity Estimation — Back-of-the-Envelope Math

> In a system design interview, capacity estimation is your **credibility check**.  
> It proves you think in numbers, not hand-waves.

---

## 🔢 Traffic Estimation

### Daily Active Users → Requests Per Second

```
DAU = 3,000,000 users
Average session = 8 min
Pages per session = 12 page views
Avg API calls per page = 3 (product data + images + recommendations)

Total daily API calls = 3,000,000 × 12 × 3 = 108,000,000 requests/day
```

### QPS (Queries Per Second)

```
Average QPS = 108,000,000 / 86,400 = ~1,250 QPS

Peak QPS (assume traffic concentrated in 8 peak hours):
  Peak QPS = 108,000,000 / (8 × 3,600) = ~3,750 QPS

Flash sale burst (10x peak):
  Burst QPS = 3,750 × 10 = ~37,500 QPS
```

### QPS Breakdown by Endpoint Type

| Endpoint Category | % of Traffic | Normal QPS | Peak QPS | Flash Sale QPS |
|-------------------|-------------|-----------|---------|---------------|
| **Product reads** (browse, search, detail) | 60% | 750 | 2,250 | 22,500 |
| **Cart operations** (add, update, view) | 15% | 188 | 563 | 5,625 |
| **User/auth** (login, profile) | 10% | 125 | 375 | 3,750 |
| **Search queries** | 10% | 125 | 375 | 3,750 |
| **Orders/payments** | 3% | 38 | 113 | 1,125 |
| **Seller operations** | 2% | 25 | 75 | 750 |
| **Total** | 100% | **1,250** | **3,750** | **37,500** |

> 💡 **Key insight**: Product reads dominate (60%). This is why caching is the #1 optimization — not database tuning.

---

## 💾 Storage Estimation

### Product Data

```
5,000,000 products × average row size:
  - Product metadata: ~2 KB (title, description, price, category, status)
  - 4 images × 500 KB = 2 MB per product (stored in S3/CDN, not DB)
  - SKU variants: ~200 bytes × 3 variants = 600 bytes

Database storage (metadata only):
  5,000,000 × 2.6 KB = ~13 GB

Image storage (object storage):
  5,000,000 × 2 MB = ~10 TB
```

### Order Data

```
150,000 orders/day
Average order row: ~500 bytes
Average 2.5 order items × 300 bytes = 750 bytes per order

Daily order storage: 150,000 × 1.25 KB = ~188 MB/day
Annual order storage: 188 MB × 365 = ~68.6 GB/year
5-year retention: ~343 GB
```

### User Data

```
10,000,000 users × ~1 KB per user = ~10 GB
100,000 sellers × ~2 KB shop data = ~200 MB
```

### Search Index

```
5,000,000 products
Elasticsearch index (with analyzed fields):
  ~5x raw data = 5,000,000 × 2 KB × 5 = ~50 GB
  With replicas (1 primary + 1 replica): ~100 GB
```

### Total Storage Summary

| Data Type | Storage | Location | Growth Rate |
|-----------|---------|----------|-------------|
| Product metadata | 13 GB | MySQL/PostgreSQL | +2 GB/year |
| Product images | 10 TB | S3 + CDN | +2 TB/year |
| Orders (5 years) | 343 GB | MySQL/PostgreSQL | +69 GB/year |
| Users + Sellers | 10.2 GB | MySQL/PostgreSQL | +2 GB/year |
| Search index | 100 GB | Elasticsearch | +20 GB/year |
| Cart data | ~5 GB | Redis | Transient |
| Cache | ~20 GB | Redis | Transient |
| **Total persistent** | **~477 GB** | — | **+93 GB/year** |
| **Total with images** | **~10.5 TB** | — | **+2.1 TB/year** |

> 💡 **Key insight**: The database is small (~500 GB). The real storage cost is **images** (10 TB). This is why S3 + CDN is non-negotiable.

---

## 📡 Bandwidth Estimation

### Ingress (Client → Server)

```
Write operations:
  - Product creation: ~100/day by sellers (small payloads, ~5 KB + images uploaded separately)
  - Cart operations: 1,500,000/day × ~500 bytes = ~750 MB/day
  - Orders: 150,000/day × ~2 KB = ~300 MB/day
  - Image uploads: 100 new products × 4 images × 500 KB = ~200 MB/day

Total ingress: ~1.25 GB/day = ~115 Kbps average
Peak ingress: ~1.15 Mbps
```

### Egress (Server → Client)

```
Read operations:
  - Product pages: 108,000,000 API calls × ~3 KB avg response = ~324 GB/day
  - Image serving (via CDN): 108,000,000 × 0.3 (30% img requests) × 200KB = ~6.5 TB/day
  
API egress: 324 GB/day = ~30 Mbps average
Peak API egress: ~90 Mbps
CDN egress: 6.5 TB/day (handled by CDN, not our servers)
```

### Bandwidth Summary

| Direction | Average | Peak | Flash Sale |
|-----------|---------|------|-----------|
| **API Ingress** | 115 Kbps | 1.15 Mbps | 11.5 Mbps |
| **API Egress** | 30 Mbps | 90 Mbps | 900 Mbps |
| **CDN Egress** | 600 Mbps | 1.8 Gbps | 18 Gbps |

> ⚠️ **CDN is not optional**. Without it, our servers would need to handle 18 Gbps during flash sales. That's ~$50K/month in bandwidth alone.

---

## 🖥 Infrastructure Estimation

### Application Servers

```
Assumption: Each app server handles ~500 QPS (well-tuned Laravel/Node)
  Normal: 1,250 / 500 = 3 servers
  Peak: 3,750 / 500 = 8 servers
  Flash sale: 37,500 / 500 = 75 servers

With headroom (2x): 
  Normal: 6 servers
  Peak: 16 servers
  Flash sale: 150 servers (auto-scaled)
```

> **⚠️ Known Risk: Auto-Scaling Cold Start**
>
> The flash sale target of 150 servers assumes auto-scaling has time to provision them. In reality, VM provisioning takes **2-5 minutes**, during which the first wave of traffic hits only the existing 6-16 servers. For scheduled flash sales, we pre-scale 30 minutes before. For unscheduled viral traffic spikes, a warm pool of standby instances (costly but necessary) or container-based scaling (30-60 second startup) reduces this gap.

### Database Servers

```
Read QPS (with 95% cache hit):
  Total read QPS: ~3,000 (peak)
  After cache: 3,000 × 0.05 = 150 QPS hitting DB
  Single MySQL can handle: ~3,000 QPS
  → 1 primary + 2 read replicas is sufficient until 10M+ users

Write QPS:
  ~113 QPS peak (orders, cart updates, products)
  Single MySQL primary can handle this easily
```

### Cache Servers (Redis)

```
Cache QPS: 3,000 × 0.95 = 2,850 QPS (cache hits)
Single Redis: ~100,000 QPS
→ 1 Redis instance is sufficient for cache

Cart storage: ~5 GB
Session storage: ~2 GB
→ 1 Redis instance with 16 GB RAM

Recommendation: Redis Cluster with 3 nodes for HA
```

### Search (Elasticsearch)

```
Search QPS: ~375 peak
Single ES node handles: ~1,000 QPS for simple queries
→ 3-node ES cluster (1 primary + 2 replicas) for HA

Index size: ~100 GB
→ Each node needs ~64 GB RAM (50% for OS cache)
```

### Infrastructure Summary

| Component | Normal | Peak | Flash Sale | Monthly Cost (est.) |
|-----------|--------|------|-----------|-------------------|
| **App Servers** (4 vCPU, 8GB) | 6 | 16 | 150 | $2,000–$15,000 |
| **DB Primary** (8 vCPU, 32GB) | 1 | 1 | 1 | $800 |
| **DB Read Replicas** (8 vCPU, 32GB) | 2 | 2 | 4 | $1,600–$3,200 |
| **Redis Cluster** (16GB RAM) | 3 nodes | 3 | 6 | $900–$1,800 |
| **Elasticsearch** (64GB RAM) | 3 nodes | 3 | 3 | $2,400 |
| **S3 + CDN** | — | — | — | $3,000–$8,000 |
| **Queue (SQS/RabbitMQ)** | 1 | 3 | 10 | $200–$1,000 |
| **Load Balancer** | 1 | 1 | 1 | $200 |
| **Total estimate** | — | — | — | **$11,100–$33,200/mo** |

> 💡 **Day 1 reality**: You don't need all this on day 1. Start with 2 app servers + 1 DB + 1 Redis + CloudFlare CDN ≈ **$500/month**. Scale as you grow.

---

## 📊 Summary: Key Numbers to Remember

```
┌────────────────────────────────────────────────────────┐
│              CAPACITY CHEAT SHEET                       │
├────────────────────────────────────────────────────────┤
│  DAU:              3,000,000                           │
│  Normal QPS:       1,250                               │
│  Peak QPS:         3,750                               │
│  Flash sale QPS:   37,500                              │
│                                                        │
│  DB size:          ~500 GB (5 years)                   │
│  Image storage:    ~10 TB                              │
│  Cache size:       ~20 GB                              │
│                                                        │
│  Read:Write ratio: 100:1                               │
│  Cache hit target: 95%+                                │
│  DB read QPS:      150 (after cache)                   │
│                                                        │
│  App servers:      6–150 (auto-scaled)                 │
│  DB servers:       1 primary + 2–4 replicas            │
│  Monthly cost:     $500 (Day 1) → $33K (full scale)   │
└────────────────────────────────────────────────────────┘
```

---

## ⬅️ [← Overview](00-overview.md) · [High-Level Architecture →](02-high-level-architecture.md)
