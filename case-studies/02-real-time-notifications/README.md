# Case Study 2: Real-Time Notification System

> **Interview Prompt**: "Design a notification system for a platform with 50M users that delivers push notifications, in-app notifications, emails, and SMS — with real-time delivery for in-app and push, and guaranteed delivery for transactional messages."

---

## 📑 Table of Contents

| # | Section | Focus |
|---|---------|-------|
| 00 | [Overview & Requirements](00-overview.md) | Problem statement, functional/non-functional requirements, capacity assumptions |
| 01 | [Capacity Estimation](01-capacity-estimation.md) | QPS, storage, bandwidth, connection count math |
| 02 | [High-Level Architecture](02-high-level-architecture.md) | System diagram, API design, service boundaries |
| 03 | [Data Model & Storage](03-data-model.md) | Schema design, notification store, read/unread tracking |
| 04 | [Connection Management](04-connection-management.md) | WebSocket vs SSE vs polling, connection lifecycle, heartbeats |
| 05 | [Fan-Out Strategy](05-fan-out-strategy.md) | Fan-out-on-write vs fan-out-on-read, pub/sub, Kafka topology |
| 06 | [Delivery Guarantees](06-delivery-guarantees.md) | At-least-once, deduplication, priority routing, rate limiting |
| 07 | [Failure & Recovery](07-failure-recovery.md) | Connection drops, broker failure, thundering herd, recovery playbooks |
| 08 | [Scaling Strategy](08-scaling-strategy.md) | Day 1 → 50M → 200M users, cost progression |

---

## 🎯 Interview Prompt (Full)

> Your company runs a social/e-commerce platform with 50 million registered users.
> You need to build a unified notification system that:
>
> 1. Delivers **in-app notifications** in real-time (< 1 second)
> 2. Sends **push notifications** to iOS/Android devices
> 3. Sends **transactional emails** (order confirmation, password reset)
> 4. Sends **SMS** for critical alerts (2FA, payment)
> 5. Supports **notification preferences** (users control what they receive)
> 6. Handles **fan-out** (one event → millions of notifications, e.g., "new feature announcement")
> 7. Guarantees **delivery** for transactional messages (payment, 2FA)
> 8. Scales to **50M users**, 5M concurrent connections, 100K notifications/second during peak
>
> Design the system. Focus on real-time delivery, fan-out strategy, and scaling the connection layer.

---

## 🔑 Key Challenges

| Challenge | Why It's Hard |
|-----------|---------------|
| **5M concurrent WebSocket connections** | Each connection holds a TCP socket open — that's 5M open file descriptors |
| **100K notifications/second** | Fan-out to targeted users within 1 second |
| **Broadcast to 50M users** | "New feature" notification → 50M deliveries in minutes, not hours |
| **Multi-channel coordination** | Same event triggers in-app + push + email — deduplication and preference routing |
| **Guaranteed delivery** | 2FA SMS and payment emails MUST arrive — no "best effort" |
| **Offline users** | User opens app after 3 days — see all missed notifications, in order |
| **Connection affinity** | User connected to Server A, notification routed to Server B — how to find them? |

---

## 📖 Reading Guide

**Quick pass (15 min)**: Read 00-overview → 02-architecture → 05-fan-out  
**Full study (45 min)**: Read all sections in order  
**Deep dive**: Focus on 04-connection-management + 05-fan-out-strategy — these are the core interview differentiators

---

## 🏗 Technology Decisions Summary

| Component | Choice | Alternative Considered | Rationale |
|-----------|--------|----------------------|----------|
| **Real-time transport** | WebSocket (SSE fallback) | SSE only, Long polling | Bidirectional needed for acks + read receipts |
| **Message broker** | Kafka (3 partitions/topic) | RabbitMQ | 200M msgs/day + replay capability for catch-up |
| **Notification store** | Cassandra (TTL 90d) | MySQL, DynamoDB | Write-heavy, partition-by-user, built-in TTL expiry |
| **User/config store** | MySQL | PostgreSQL | Relational for preferences, templates, users |
| **Connection routing** | Redis hash map | In-memory per-server | Cross-server routing; survives WS server restarts |
| **Load balancer** | L4 (TCP) for WebSocket | L7 (HTTP) | Lower overhead for persistent connections |
| **WS server language** | Go | Node.js, Java | Goroutines handle 500K concurrent connections per server |
| **Fan-out strategy** | Hybrid (write < 1M, read > 1M) | Pure write fan-out | Can't write 50M rows for broadcast; read fan-out for large audiences |
| **Multi-region** | Delay until Stage 5 (50M+) | From Day 1 | Complexity not justified until global user base |

> Full trade-off analysis with reasoning: [08-scaling-strategy.md → Key Trade-offs Summary](08-scaling-strategy.md)

---

## ⬅️ [← Back to All Case Studies](../../README.md)
