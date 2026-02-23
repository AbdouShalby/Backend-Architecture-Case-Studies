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

| # | Case Study | Key Topics | Status |
|---|-----------|------------|--------|
| 1 | [**High-Scale Marketplace**](case-studies/01-high-scale-marketplace/) | Read/write separation, caching, sharding, payment flow, event-driven | ✅ Complete |
| 2 | [**Real-Time Notification System**](case-studies/02-real-time-notifications/) | WebSocket vs SSE, fan-out, pub/sub vs Kafka, horizontal scaling | 🔜 Coming |
| 3 | [**Payment Processing System**](case-studies/03-payment-processing/) | Idempotency, fraud detection, ledger design, exactly-once myth | 🔜 Coming |
| 4 | [**Rate Limiting System**](case-studies/04-rate-limiting/) | Sliding window, token bucket, Redis implementation, DDoS mitigation | 🔜 Coming |

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
│   │   └── diagrams/                     # Mermaid architecture diagrams
│   │
│   ├── 02-real-time-notifications/
│   │   └── (coming soon)
│   │
│   ├── 03-payment-processing/
│   │   └── (coming soon)
│   │
│   └── 04-rate-limiting/
│       └── (coming soon)
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

## License

MIT
