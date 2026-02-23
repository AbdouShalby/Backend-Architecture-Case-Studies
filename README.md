# 📘 Backend Architecture Case Studies

[![System Design](https://img.shields.io/badge/System_Design-Interview_Ready-blue?style=for-the-badge)](.)
[![Diagrams](https://img.shields.io/badge/Diagrams-Mermaid-ff69b4?style=for-the-badge)](.)
[![Case Studies](https://img.shields.io/badge/Case_Studies-4-green?style=for-the-badge)](.)

Production-grade **system design case studies** written like senior engineering interviews.  
Not generic blog posts — these are **whiteboard-ready designs** with real numbers, QPS calculations, data sizing, bottleneck analysis, and trade-off decisions.

---

## ⚡ What Makes This Different

| Typical "System Design" | This Repository |
|-------------------------|-----------------|
| "Use a load balancer" | **Why** an L7 LB, how many instances, what's the failover strategy |
| "Add caching" | Cache **what**, TTL strategy, invalidation pattern, hit ratio target |
| "Use a queue" | Queue **sizing**, consumer throughput, DLQ strategy, backpressure handling |
| "Scale horizontally" | Scale **what** — read replicas? shards? workers? At what QPS threshold? |
| "Use microservices" | Service **boundaries**, communication patterns, failure domains |

Every design includes:
- 📊 **Back-of-the-envelope calculations** (QPS, storage, bandwidth)
- 🧮 **Data model with index strategy**
- 📈 **Scaling plan** (Day 1 → 1M → 10M → 100M users)
- 💥 **Failure scenarios** and recovery strategies
- 💰 **Cost awareness** (infrastructure estimates)
- ⚖️ **Trade-off analysis** for every major decision

---

## 📚 Case Studies

| # | Case Study | Scale | Key Topics | Status |
|---|-----------|-------|------------|--------|
| 1 | [**High-Scale Marketplace**](case-studies/01-high-scale-marketplace/) | 10M MAU, 36K QPS | CQRS, caching, sharding, saga pattern | ✅ Complete |
| 2 | [**Real-Time Notification System**](case-studies/02-real-time-notifications/) | 50M users, 100K QPS | Kafka, WebSocket, fan-out, Cassandra | ✅ Complete |
| 3 | [**Payment Processing System**](case-studies/03-payment-processing/) | $500M/yr, 2K TPS | Idempotency, double-entry ledger, PCI DSS | ✅ Complete |
| 4 | [**Rate Limiting System**](case-studies/04-rate-limiting/) | 500K req/sec, 10K tenants | Sliding window, token bucket, Redis Lua | ✅ Complete |

---

## 🔍 Case Study Highlights

### CS1: High-Scale Marketplace (12 files, ~3,900 lines)
> **"Design a marketplace like Amazon handling 10M active users with flash sales."**

- 📊 **36K QPS** peak (10× multiplier on flash sales) → 3-tier caching (CDN → Redis → MySQL)
- 🗄️ **MySQL → shard at 10M+** with CQRS read replicas for 100:1 read-to-write ratio
- 💰 **Cost progression**: $50/mo (launch) → $200K+/mo (100M users) across 6 stages
- ⚡ Key insight: **Start monolith, extract services by team boundaries** — not by technical layers

### CS2: Real-Time Notification System (9 files, ~2,700 lines)
> **"Deliver 200M notifications/day to 50M users across 4 channels, with 5M concurrent WebSocket connections."**

- 📊 **100K notifications/sec** peak → Kafka (3 partitions/topic) + Cassandra (TTL 90d)
- 🔌 **500K WebSocket connections per Go server** — L4 load balancing, Redis routing map
- 🔀 **Hybrid fan-out**: write fan-out for < 1M recipients, read fan-out for broadcasts
- ⚡ Key insight: **SMS = dominant cost** at every scale (10M SMS/day = $75K/mo)

### CS3: Payment Processing System (9 files, ~2,500 lines)
> **"Process $500M/year with exactly-once money movement and PCI DSS compliance."**

- 📊 **2,000 TPS** design target (200 payment TPS × 5 internal ops × safety margin)
- 💳 **Double-entry ledger** in PostgreSQL — debits always equal credits, amounts as integer cents
- 🔒 **PCI SAQ A-EP** via client-side tokenization: $15K/yr vs $300K/yr for full compliance
- ⚡ Key insight: **PSP fees = 99.3% of cost** ($1.26M/mo) — infrastructure ($9.5K/mo) is a rounding error

### CS4: Rate Limiting System (9 files, ~2,600 lines)
> **"Protect a 10K-tenant API platform at 500K requests/second with sub-millisecond decisions."**

- 📊 **1.25M Redis ops/sec** across 6-node cluster — Lua scripts for atomic counters
- 🧮 **5 algorithms compared**: Sliding Window Counter wins (0.003% error, O(1) memory)
- 🛡️ **Tiered failure policy**: fail-closed for security limits, fail-open for quotas
- ⚡ Key insight: **±5% accuracy is acceptable** — local aggregation with 100ms sync = 10× fewer Redis ops

---

## 📊 Cross-Study Architecture Comparison

| Decision | Marketplace | Notifications | Payments | Rate Limiting |
|----------|------------|--------------|----------|--------------|
| **Primary DB** | MySQL | Cassandra + MySQL | PostgreSQL | Redis Cluster |
| **Queue/Broker** | RabbitMQ → Kafka | Kafka | RabbitMQ | Redis Pub/Sub |
| **Cache** | Redis (multi-layer) | Redis (routing) | Redis (idempotency) | Redis (counters) |
| **Consistency** | Eventual (most data) | Eventual | Strong (ledger) | Approximate (±5%) |
| **Failure strategy** | Saga compensation | Priority-based retry | Circuit breaker + fallback PSP | Tiered fail-closed/open |
| **Scaling approach** | Shard at 10M+ | Add WS servers | Partition by month | Redis Cluster sharding |
| **Peak design** | 36K QPS | 100K QPS | 2,000 TPS | 500K req/sec |
| **Monthly cost (at scale)** | $5K-$15K (10M MAU) | $30K-$50K (50M users) | $9.5K + $1.26M PSP | $3.1K |

---

## 🧠 How to Read These

Each case study follows a consistent structure that mirrors a real system design interview:

```
1. Requirements & Constraints
   ├── Functional requirements (what the system does)
   ├── Non-functional requirements (latency, availability, consistency)
   └── Capacity estimation (QPS, storage, bandwidth)

2. High-Level Architecture
   ├── Component diagram
   ├── Data flow
   └── API design

3. Deep Dives
   ├── Data model & database choice
   ├── Caching strategy
   ├── Queue & async processing
   ├── Scaling plan
   └── Domain-specific concerns

4. Failure Scenarios
   ├── What breaks
   ├── Blast radius
   └── Recovery strategy

5. Trade-off Analysis
   ├── Decisions made
   ├── Alternatives considered
   └── Why this approach wins
```

---

## 🎯 Who Is This For

- **Senior Backend Engineers** preparing for system design interviews
- **Tech Leads** evaluating architecture patterns
- **Engineers transitioning** from mid-level to senior roles
- **Anyone** who wants to think beyond CRUD

---

## 🏗 Project Structure

```
backend-architecture-case-studies/
├── case-studies/
│   ├── 01-high-scale-marketplace/
│   │   ├── 00-overview.md                # Problem statement & requirements
│   │   ├── 01-capacity-estimation.md     # QPS, storage, bandwidth calculations
│   │   ├── 02-high-level-architecture.md # Component diagram & API design
│   │   ├── 03-data-model.md              # Schema, indexes, database choice
│   │   ├── 04-caching-strategy.md        # Cache layers, invalidation, hit ratios
│   │   ├── 05-read-write-separation.md   # CQRS, replication lag, consistency
│   │   ├── 06-sharding-strategy.md       # Partition key, rebalancing, hot spots
│   │   ├── 07-queue-design.md            # Async flows, DLQ, backpressure
│   │   ├── 08-payment-flow.md            # Payment lifecycle, idempotency
│   │   ├── 09-event-driven-model.md      # Event sourcing, saga pattern
│   │   ├── 10-failure-recovery.md        # Failure modes, blast radius, recovery
│   │   ├── 11-scaling-strategy.md        # Day 1 → 10M → 100M growth plan
│   │
│   ├── 02-real-time-notifications/
│   │   ├── 00-overview.md                # Problem statement & notification types
│   │   ├── 01-capacity-estimation.md     # QPS, storage, bandwidth, cost
│   │   ├── 02-high-level-architecture.md # Kafka pipeline, API design, service boundaries
│   │   ├── 03-data-model.md              # Cassandra + MySQL + Redis schemas
│   │   ├── 04-connection-management.md   # WebSocket at 5M connections, kernel tuning
│   │   ├── 05-fan-out-strategy.md        # Hybrid fan-out, broadcast store
│   │   ├── 06-delivery-guarantees.md     # Priority routing, dedup, rate limiting
│   │   ├── 07-failure-recovery.md        # Failure modes, runbooks, recovery SLAs
│   │   └── 08-scaling-strategy.md        # Day 1 → 200M users growth plan
│   │
│   ├── 03-payment-processing/
│   │   ├── 00-overview.md                # Problem statement & transaction types
│   │   ├── 01-capacity-estimation.md     # TPS, storage, cost ($9.5K vs $1.26M PSP)
│   │   ├── 02-high-level-architecture.md # Service boundaries, PCI scope, API design
│   │   ├── 03-data-model.md              # Double-entry ledger, PostgreSQL schema
│   │   ├── 04-payment-flow.md            # Authorize → capture → settle lifecycle
│   │   ├── 05-idempotency.md             # Exactly-once myth, dedup, edge cases
│   │   ├── 06-fraud-detection.md         # Rule engine, ML scoring, 3DS strategy
│   │   ├── 07-failure-recovery.md        # Double charge, orphaned auth, reconciliation
│   │   └── 08-scaling-compliance.md      # PCI DSS, multi-PSP, data security
│   │
│   └── 04-rate-limiting/
│       ├── 00-overview.md                # Rate limit types, algorithms, requirements
│       ├── 01-capacity-estimation.md     # 500K req/sec, Redis sizing, cost ($3.1K/mo)
│       ├── 02-high-level-architecture.md # Hybrid gateway + library, config sync
│       ├── 03-algorithms.md              # 5 algorithms deep dive with Lua scripts
│       ├── 04-data-model-redis.md        # Redis keys, Lua scripts, cluster sharding
│       ├── 05-distributed-challenges.md  # Race conditions, partitions, clock skew
│       ├── 06-multi-tenant.md            # Plan tiers, weighted quotas, burst control
│       ├── 07-failure-recovery.md        # Redis failure modes, tiered fallback
│       └── 08-ddos-advanced.md           # Adaptive limits, DDoS detection, edge defense
│
└── README.md
```

---

## 📐 Design Methodology

Every decision follows this framework:

```
┌─────────────────────────────────────────┐
│          DECISION FRAMEWORK             │
├─────────────────────────────────────────┤
│                                         │
│  1. What PROBLEM are we solving?        │
│  2. What are the CONSTRAINTS?           │
│  3. What are the OPTIONS?               │
│  4. What TRADE-OFFS does each have?     │
│  5. Which option FITS our constraints?  │
│  6. What's the MIGRATION PATH?          │
│     (Day 1 → scale-up → re-architect)  │
│                                         │
└─────────────────────────────────────────┘
```

---

## 📈 Repository Stats

| Metric | Value |
|--------|-------|
| **Total case studies** | 4 |
| **Total files** | 39 Markdown files + 4 READMEs |
| **Total content** | ~11,700 lines |
| **Mermaid diagrams** | 40+ (inline, rendered on GitHub) |
| **Technology decisions documented** | 30+ with full rationale |
| **Failure scenarios covered** | 25+ with recovery strategies |

---

## License

MIT
