# 1. Capacity Estimation

> "If you don't know your numbers, you don't know your system."

---

## 📊 Transaction Volume

### Annual Breakdown

| Metric | Calculation | Result |
|--------|------------|--------|
| Annual GMV | Given | **$500,000,000** |
| Avg transaction value | Weighted average | **$50** |
| Annual transactions | $500M ÷ $50 | **10,000,000** |
| Monthly transactions | 10M ÷ 12 | **833,333** |
| Daily transactions | 10M ÷ 365 | **~27,400** |
| Average TPS | 27,400 ÷ 86,400 | **~0.3 TPS** |

```
Wait — 0.3 TPS? That's tiny!

The number is misleading. Payment traffic is extremely bursty:
  • 80% of daily volume happens in 8 hours (10 AM - 6 PM)
  • Flash sales can concentrate 1 hour of traffic into 5 minutes
  • Black Friday: 5-10× normal daily volume

Design for peak, not average.
```

### Peak Traffic Analysis

| Scenario | Multiplier | TPS | Duration |
|----------|-----------|-----|----------|
| Normal business hours | 3× avg | ~1 TPS | 8 hours |
| Lunch rush (12-1 PM) | 10× avg | ~3 TPS | 1 hour |
| Marketing campaign | 30× avg | ~10 TPS | 2-4 hours |
| Flash sale start | 200× avg | ~60 TPS | 15 minutes |
| Black Friday peak | 600× avg | ~200 TPS | 2-4 hours |
| **Design target** | **6,000× avg** | **2,000 TPS** | **Burst** |

```
Why design for 2,000 TPS when peak is ~200?

1. Headroom: 10× peak for unexpected viral moments
2. Each "payment" generates 3-5 internal operations:
   - Fraud check
   - Authorization
   - Status updates
   - Webhook delivery
   - Ledger writes
3. 200 payment TPS × 5 ops = 1,000 internal TPS
4. With safety margin → 2,000 TPS design target
```

---

## 💾 Storage Estimation

### Per-Transaction Storage

| Data | Size | Retention |
|------|------|-----------|
| Payment record | 1 KB | 7 years |
| Transaction log (auth + capture) | 500 B × 2 = 1 KB | 7 years |
| Ledger entries (debit + credit) | 200 B × 2 = 400 B | 7 years (legal) |
| Fraud check result | 500 B | 2 years |
| Audit log | 2 KB | 7 years |
| **Total per payment** | **~5 KB** | |

### Annual Storage Growth

| Data Type | Annual Size | 7-Year Size |
|-----------|-------------|-------------|
| Payment records | 10M × 1 KB = 10 GB | 70 GB |
| Transactions | 20M × 500 B = 10 GB | 70 GB |
| Ledger entries | 40M × 200 B = 8 GB | 56 GB |
| Fraud data | 10M × 500 B = 5 GB | 10 GB (2yr retention) |
| Audit logs | 10M × 2 KB = 20 GB | 140 GB |
| **Total** | **~53 GB/year** | **~350 GB** |

```
350 GB total for 7 years of data? That's tiny.

Payment systems are NOT storage-heavy. They're:
  ✅ Consistency-heavy (every cent must balance)
  ✅ Durability-heavy (zero loss tolerance)
  ✅ Latency-sensitive (users waiting at checkout)
  ✅ Compliance-heavy (encrypted, audited, logged)

A single PostgreSQL instance can hold all of this.
The challenge is NOT storage — it's correctness.
```

---

## 🌐 Network & API Traffic

### Inbound Traffic

| Source | Requests/day | Avg Size | Daily Bandwidth |
|--------|-------------|----------|-----------------|
| Payment creation | 27K | 2 KB | 54 MB |
| Status checks (polling) | 270K (10× payments) | 500 B | 135 MB |
| Webhook confirmations | 54K (2× payments) | 1 KB | 54 MB |
| Dashboard / reporting | 10K | 5 KB | 50 MB |
| **Total** | **~360K** | | **~300 MB/day** |

### Outbound Traffic (to PSPs)

| Call | Volume/day | Avg Latency | Notes |
|------|-----------|-------------|-------|
| Authorize | 27K | 500ms | Real-time, synchronous |
| Capture | 25K | 300ms | Can be batched |
| Void | 2K | 300ms | For cancelled orders |
| Refund | 1K | 300ms | On-demand |
| **Total PSP calls** | **~55K/day** | | **~0.6/sec avg** |

---

## 💰 Infrastructure Cost Estimation

### Compute

| Service | Instances | Spec | Monthly Cost |
|---------|-----------|------|-------------|
| Payment API | 3 (HA) | 4 vCPU, 16 GB | $600 |
| Fraud engine | 2 | 8 vCPU, 32 GB | $800 |
| Worker (async) | 2 | 2 vCPU, 8 GB | $200 |
| Webhook delivery | 2 | 2 vCPU, 4 GB | $150 |

### Storage & Data

| Service | Spec | Monthly Cost |
|---------|------|-------------|
| PostgreSQL (primary) | 8 vCPU, 32 GB, 500 GB SSD | $800 |
| PostgreSQL (replica) | 8 vCPU, 32 GB, 500 GB SSD | $800 |
| Redis (cache + idempotency) | 4 GB cluster | $200 |
| Message queue (RabbitMQ/SQS) | Managed | $100 |
| Object storage (receipts, evidence) | 100 GB | $5 |

### Third-Party Services

| Service | Volume | Monthly Cost |
|---------|--------|-------------|
| **PSP fees** (2.9% + $0.30 avg) | $42M/month | **$1,260,000** |
| PCI DSS compliance (scans, audits) | Annual ÷ 12 | $5,000 |
| Fraud scoring (ML service) | 27K/day | $500 |
| SMS (2FA for high-risk) | 5K/day | $300 |

### Total Monthly Infrastructure

| Category | Cost |
|----------|------|
| Compute | $1,750 |
| Storage & data | $1,905 |
| Third-party (excl. PSP) | $5,800 |
| **Infrastructure total** | **~$9,500/month** |
| PSP processing fees | $1,260,000/month |

```
The elephant in the room: PSP fees are 99.3% of total cost.

Your infrastructure costs $9.5K/month.
Payment processing fees cost $1.26M/month.

This is why payment companies optimize for:
  1. Authorization rate (every declined transaction = lost revenue)
  2. Smart PSP routing (Stripe 2.9% vs Adyen 2.5% for some cards)
  3. Fraud prevention (chargebacks cost $15-25 each in fees PLUS the sale)
  4. NOT infrastructure (it's negligible comparatively)
```

---

## 📊 Capacity Cheat Sheet

```
┌────────────────────────────────────────────────────┐
│           PAYMENT SYSTEM — KEY NUMBERS             │
├────────────────────────────────────────────────────┤
│ Annual GMV:           $500M                        │
│ Transactions/year:    10M                          │
│ Transactions/day:     ~27,400                      │
│ Average TPS:          0.3 (misleadingly low)       │
│ Design TPS:           2,000 (peak + headroom)      │
│                                                    │
│ Storage/year:         ~53 GB                       │
│ Storage/7 years:      ~350 GB (fits one disk)      │
│                                                    │
│ PSP fee/month:        $1.26M                       │
│ Infra cost/month:     $9.5K                        │
│                                                    │
│ Key insight: This is a CORRECTNESS problem,        │
│ not a SCALE problem. PostgreSQL handles it easily. │
│ The hard part is money integrity, not throughput.   │
└────────────────────────────────────────────────────┘
```

---

## ⬅️ [← Overview](00-overview.md) · [Architecture →](02-high-level-architecture.md)
