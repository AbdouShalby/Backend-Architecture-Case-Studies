# 1. Capacity Estimation

> "A rate limiter that adds 10ms to every request is worse than no rate limiter at all. Know your overhead budget."

---

## 📊 Traffic Volume

### Platform Traffic

| Metric | Value | Notes |
|--------|-------|-------|
| Total requests/sec | 500,000 | Aggregate across all tenants |
| Total requests/day | 43.2 billion | 500K × 86,400 |
| API servers | 20 | Each handles ~25K req/sec |
| Rate limit checks per request | 2.5 avg | (tenant + user + maybe endpoint) |
| **Total rate limit ops/sec** | **1,250,000** | 500K × 2.5 |

### Tenant Distribution

```
10,000 tenants, Pareto distribution:

  Top 1% (100 tenants):   40% of traffic = 200K req/sec
  Top 10% (1,000 tenants): 80% of traffic = 400K req/sec
  Bottom 50% (5,000):      5% of traffic  = 25K req/sec
  Bottom 90% (9,000):      20% of traffic = 100K req/sec

  Largest single tenant: ~20K req/sec
  Median tenant: ~5 req/sec

  This distribution matters for:
    • Hot key handling (top tenants create hot Redis keys)
    • Counter memory (bottom 50% have mostly idle counters)
```

---

## 💾 Memory Estimation (Redis)

### Per-Counter Memory

```
Each rate limit counter in Redis:

  Fixed window (simplest):
    Key: "rl:{tenant_id}:{endpoint}:{minute_bucket}"
    Value: integer counter
    Redis overhead: ~80 bytes per key
    TTL: window duration

  Sliding window counter:
    Key: "rl:{tenant_id}:{endpoint}:sw"
    Value: 2 integers (current + previous window)
    Redis overhead: ~120 bytes per key
    TTL: 2× window duration

  Token bucket:
    Key: "rl:{tenant_id}:{endpoint}:tb"
    Value: tokens_remaining + last_refill_time (2 values)
    Redis overhead: ~120 bytes per key
    TTL: none (persistent until removed)
```

### Active Counters

| Dimension | Active Keys | Memory (per-key ~100B) |
|-----------|-------------|----------------------|
| Per-tenant (10K tenants) | 10,000 | 1 MB |
| Per-tenant-per-endpoint (10K × 10 endpoints) | 100,000 | 10 MB |
| Per-user (unique active users/min) | ~200,000 | 20 MB |
| Per-IP | ~100,000 | 10 MB |
| **Total active counters** | **~410,000** | **~41 MB** |

```
41 MB for all rate limit counters?

Yes. Rate limiting is NOT memory-intensive.
Redis with 1 GB of RAM has 25× headroom for counters.

The bottleneck is OPERATIONS/SEC, not memory.
1.25M Redis ops/sec is the real constraint.
```

---

## ⚡ Redis Operations Budget

### Operations Per Rate Limit Check

| Algorithm | Redis Ops | Latency |
|-----------|----------|---------|
| Fixed window | 1 INCR + 1 EXPIRE | ~0.2ms |
| Sliding window counter | 2 GET + 1 INCR + 1 EXPIRE | ~0.4ms |
| Token bucket | 1 GET + 1 SET (Lua script = 1 round trip) | ~0.3ms |
| Sliding window log | 1 ZADD + 1 ZREMRANGEBYSCORE + 1 ZCARD | ~0.5ms |

### Redis Cluster Sizing

```
Required: 1.25M ops/sec

Single Redis node: ~100K ops/sec (simple commands)
  With pipelining: ~300K ops/sec
  
Cluster needed: 1.25M ÷ 300K = ~5 Redis nodes (with pipelining)
Safety margin (2×): 10 Redis nodes

But: Lua scripts (atomic operations) are ~50K/sec per node
  If using Lua: 1.25M ÷ 50K = 25 nodes → expensive!
  
Optimization: pipeline + batch where possible
  Sliding window counter with Lua: ~100K scripts/sec per node
  1.25M ÷ 100K = 13 nodes → more reasonable
  
Design: 6-node Redis Cluster (3 primary + 3 replica)
  Each primary: ~420K ops/sec capacity (with optimization)
  Each primary: ~210K ops/sec used
  Headroom: 2× (good for burst traffic)
```

---

## 🌐 Network Budget

| Traffic | Rate | Size | Bandwidth |
|---------|------|------|-----------|
| Redis requests (check + update) | 1.25M/sec | ~200B avg | 250 MB/sec |
| Redis responses | 1.25M/sec | ~50B avg | 62 MB/sec |
| Config sync (rule updates) | ~10/sec | ~1 KB | negligible |
| Analytics events | 500K/sec | ~100B (sampled 1%) | 0.5 MB/sec |
| **Total Redis bandwidth** | | | **~312 MB/sec** |

```
312 MB/sec between API servers and Redis cluster.
That's ~2.5 Gbps.

Within a datacenter (10 Gbps links): comfortable.
Cross-AZ: might want to co-locate or use local replicas.
```

---

## 💰 Infrastructure Cost

### Compute & Storage

| Component | Spec | Monthly Cost |
|-----------|------|-------------|
| Redis Cluster (6 nodes) | r6g.xlarge (4 vCPU, 32 GB) × 6 | $2,400 |
| Config DB (PostgreSQL) | db.r6g.large (2 vCPU, 16 GB) | $400 |
| Rate Limiter Library | Embedded in API servers (no extra cost) | $0 |
| Dashboard / Analytics | 1 server + time-series DB | $300 |
| **Total** | | **~$3,100/month** |

### Cost Per Decision

```
$3,100/month ÷ (500K req/sec × 86,400 × 30 days) = $0.0000000024 per decision

Or: $0.0024 per million decisions

That's incredibly cheap. For comparison:
  AWS API Gateway rate limiting: ~$3.50 per million requests
  Cloudflare rate limiting: ~$0.05 per 10K requests = $5 per million
  Our solution: $0.0024 per million

At 1.3 trillion decisions/month, managed services would cost $4.5M/month.
Self-hosted Redis: $3.1K/month.
```

---

## 📊 Capacity Cheat Sheet

```
┌────────────────────────────────────────────────────┐
│         RATE LIMITER — KEY NUMBERS                 │
├────────────────────────────────────────────────────┤
│ Platform traffic:      500K requests/sec           │
│ Rate limit checks:     1.25M ops/sec              │
│ Active counters:       ~410K keys (41 MB)         │
│                                                    │
│ Redis cluster:         6 nodes (3 primary + 3 rep)│
│ Per-node ops:          ~210K ops/sec used          │
│ Per-node capacity:     ~420K ops/sec               │
│                                                    │
│ Latency budget:        < 0.5ms per check (p50)    │
│ Accuracy:              ±5% acceptable              │
│                                                    │
│ Monthly cost:          $3.1K                       │
│ Cost per million:      $0.0024                     │
│                                                    │
│ Bottleneck: Redis ops/sec (not memory, not CPU)    │
└────────────────────────────────────────────────────┘
```

---

## ⬅️ [← Overview](00-overview.md) · [Architecture →](02-high-level-architecture.md)
