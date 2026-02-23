# 0. Overview & Requirements

> Rate limiting appears simple: "count requests, reject if too many." In practice, it's one of the most deceptively complex distributed systems problems — touching consensus, time synchronization, fairness, and real-time decision making at sub-millisecond latency.

---

## 📋 Functional Requirements

### Core Features (P0)

| Feature | Description |
|---------|-------------|
| **Request rate limiting** | Enforce max requests per time window (e.g., 1000 req/min) |
| **Multi-dimensional keys** | Limit by: API key, user ID, IP address, endpoint, or combination |
| **Multiple algorithms** | Support fixed window, sliding window log, sliding window counter, token bucket |
| **Per-tenant configuration** | Different limits for free, pro, enterprise plans |
| **Rate limit headers** | Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |
| **429 response** | Return `429 Too Many Requests` with `Retry-After` header |

### Important Features (P1)

| Feature | Description |
|---------|-------------|
| **Burst allowance** | Allow temporary spikes above sustained rate (token bucket) |
| **Quota management API** | CRUD for rate limit rules, real-time updates without restart |
| **Dashboard** | Real-time visibility: who's being limited, how often, top consumers |
| **Graceful degradation** | If rate limiter backend is down, define fallback behavior |
| **Distributed consistency** | Accurate counting across 20+ API servers |

### Nice to Have (P2)

| Feature | Description |
|---------|-------------|
| **Adaptive rate limiting** | Automatically tighten limits under system stress |
| **DDoS detection** | Identify and block attack patterns (not just rate limit) |
| **Cost-based limiting** | Expensive endpoints (search, export) cost more quota |
| **Geographic limits** | Different limits by region (local vs international) |

---

## 📋 Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Latency overhead** | p50 < 0.5ms, p99 < 2ms | In hot path of every request |
| **Throughput** | 500K decisions/sec | Aggregate platform traffic |
| **Availability** | 99.99% | Rate limiter down = either outage or unprotected |
| **Accuracy** | ±5% of configured limit | Small window acceptable for performance |
| **Update latency** | < 5 seconds | Config changes propagate within 5s |
| **Memory** | < 1 GB per rate limit dimension | For 10K tenants × multiple keys |

---

## 🎯 Rate Limit Types

### By Scope

| Scope | Example | Key | Typical Limit |
|-------|---------|-----|---------------|
| **Global** | Platform-wide protection | `global` | 500K req/sec |
| **Per-tenant** | API key quota | `tenant:{api_key}` | 1K-100K req/min |
| **Per-user** | Individual user actions | `user:{user_id}` | 100 req/min |
| **Per-IP** | Anonymous/unauthenticated | `ip:{ip_address}` | 50 req/min |
| **Per-endpoint** | Expensive operations | `endpoint:{path}` | varies |
| **Compound** | User + endpoint combo | `user:{id}:endpoint:{path}` | varies |

### By Time Window

| Window | Use Case | Trade-off |
|--------|----------|-----------|
| **Per-second** | DDoS protection, burst control | High Redis ops, tight |
| **Per-minute** | API quota enforcement | Most common, balanced |
| **Per-hour** | Email sending, exports | Allows natural usage patterns |
| **Per-day** | Free tier daily limits | Most relaxed |

### By Algorithm

| Algorithm | Best For | Trade-off |
|-----------|----------|-----------|
| **Fixed window** | Simple, low overhead | Boundary spike (2× at window edge) |
| **Sliding window log** | Exact counting | Memory-heavy (stores all timestamps) |
| **Sliding window counter** | Good accuracy, low memory | Approximation (~0.003% error) |
| **Token bucket** | Burst allowance | Slightly more complex |
| **Leaky bucket** | Smooth output rate | Doesn't allow any burst |

---

## 🏢 Key Entities

| Entity | Description | Scale |
|--------|-------------|-------|
| **Tenant** | API consumer (company/developer) | 10,000 active |
| **Rate Limit Rule** | Configuration (key pattern + limit + window) | ~50K rules |
| **Rate Limit Counter** | Current count for a specific key + window | ~500K active |
| **Rate Limit Event** | Individual decision (allow/deny) — for analytics | 500K/sec |
| **Plan** | Tenant's subscription tier | 4 tiers |

---

## 🏗 Key Assumptions

| Assumption | Value | Basis |
|------------|-------|-------|
| Aggregate traffic | 500K requests/sec | Multi-tenant API platform |
| Active tenants | 10,000 | With diverse usage patterns |
| API servers | 20 | Behind load balancer |
| Rate limit checks per request | 2-3 | (tenant + user + endpoint) |
| Redis ops/sec for rate limiting | 1-1.5M | 500K × 2-3 checks |
| Acceptable accuracy | ±5% | Slight over/under is OK |
| Primary storage | Redis | Sub-ms latency required |
| Configuration storage | PostgreSQL | Durable rule storage |

---

## ⬅️ [← Case Study Index](README.md) · [Capacity Estimation →](01-capacity-estimation.md)
