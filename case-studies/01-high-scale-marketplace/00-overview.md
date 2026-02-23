# 0. Overview — Problem Statement & Requirements

## 🎯 Problem Statement

Design a **multi-vendor marketplace** where:
- **Sellers** list products with descriptions, images, pricing, and inventory
- **Buyers** browse, search, filter, add to cart, and purchase
- The system handles **10 million monthly active users** (MAU)
- Must survive **flash sales** (10x peak traffic)
- Must process **payments reliably** (no double charges, no lost orders)

Think: **Amazon Marketplace**, **Jumia**, **Noon**, **Etsy** — not a single-vendor store.

---

## 📌 Functional Requirements

### Core User Flows

```
SELLER FLOW:
  Register → Create Shop → Add Products → Manage Inventory → View Orders → Get Paid

BUYER FLOW:
  Browse/Search → View Product → Add to Cart → Checkout → Pay → Track Order → Review
```

### Detailed Requirements

| Feature | Description | Priority |
|---------|-------------|----------|
| **User Registration** | Email/phone signup, OAuth (Google, Apple) | P0 |
| **Product Catalog** | Create, update, delete listings with images | P0 |
| **Search & Discovery** | Full-text search, category filters, sorting | P0 |
| **Shopping Cart** | Persistent cart, multi-seller items | P0 |
| **Checkout & Payment** | Multiple payment methods, order creation | P0 |
| **Order Management** | Order status tracking, seller fulfillment | P0 |
| **Inventory Management** | Real-time stock tracking, oversell prevention | P0 |
| **Reviews & Ratings** | Buyer reviews on purchased products | P1 |
| **Notifications** | Order updates, promotions, stock alerts | P1 |
| **Seller Dashboard** | Sales analytics, revenue tracking | P1 |
| **Recommendations** | "Similar products", "Frequently bought together" | P2 |
| **Flash Sales** | Time-limited deals with limited stock | P2 |

### Out of Scope (for this design)

- Shipping/logistics tracking (assume third-party integration)
- Customer support ticketing
- Seller verification/KYC
- Admin CMS panel
- Mobile app specifics (API-first design covers both)

---

## 📌 Non-Functional Requirements

| Requirement | Target | Reasoning |
|-------------|--------|-----------|
| **Availability** | 99.95% | Max 4.38h downtime/year — payments can't go down |
| **Latency (p50)** | < 200ms | Product page loads — perceived performance |
| **Latency (p99)** | < 800ms | Worst-case acceptable wait |
| **Checkout latency** | < 2s | End-to-end including payment gateway |
| **Consistency** | Eventual (reads), Strong (payments) | Product catalog can be stale by seconds; money cannot |
| **Read:Write ratio** | 100:1 | Browsing dominates; purchases are rare events |
| **Data durability** | 99.999999999% (11 nines) | No order or payment data loss — ever |
| **Concurrent flash sale** | 50,000 req/s burst | 10x normal peak for flash sales |
| **Search latency** | < 300ms | Including filters and sorting |

### SLA Breakdown

```
99.95% availability = 525,600 min/year × 0.0005 = 262.8 min downtime/year
                    = ~4.38 hours/year
                    = ~21.9 min/month
                    = ~5.04 min/week

This means:
- No more than ~5 minutes of downtime per week
- Requires: health checks, auto-restart, circuit breakers, redundancy
- Payment service needs higher SLA: 99.99% (52.6 min/year max downtime)
```

---

## 📌 Core Entities

```
┌──────────────────────────────────────────────────────────────────┐
│                     MARKETPLACE ENTITIES                          │
├──────────────┬───────────────────────────────────────────────────┤
│ User         │ buyer or seller (role-based)                      │
│ Shop         │ seller's storefront (1 seller → 1 shop)           │
│ Product      │ listing with title, description, images, price    │
│ SKU          │ product variant (size, color) — tracks inventory  │
│ Cart         │ buyer's active cart (items from multiple sellers)  │
│ CartItem     │ single item in cart (links to SKU)                │
│ Order        │ confirmed purchase (snapshot of cart at checkout)  │
│ OrderItem    │ single item in order (snapshot of SKU + price)    │
│ Payment      │ payment attempt (may succeed/fail/retry)          │
│ Review       │ buyer review on a purchased product               │
│ Category     │ product categorization (hierarchical)             │
│ Notification │ async messages to users (order updates, promos)   │
└──────────────┴───────────────────────────────────────────────────┘
```

---

## 📌 Key Assumptions

| Assumption | Value | Source/Reasoning |
|------------|-------|------------------|
| Monthly Active Users (MAU) | 10,000,000 | Given requirement |
| Daily Active Users (DAU) | 3,000,000 | ~30% of MAU (industry average for e-commerce) |
| Active sellers | 100,000 | 1% of users are sellers |
| Products listed | 5,000,000 | ~50 products per seller average |
| Average product images | 4 | 4 images × ~500KB = 2MB per product |
| Orders per day | 150,000 | ~5% of DAU purchase daily |
| Cart additions per day | 1,500,000 | 10x orders (many carts abandoned) |
| Average order value | $35 | Mid-range marketplace |
| Average items per order | 2.5 | Multi-item checkout |
| Peak traffic multiplier | 10x | Flash sales / Black Friday |
| Avg session duration | 8 min | E-commerce industry benchmark |
| Pages per session | 12 | Browse → search → product → cart |

---

## ➡️ Next: [Capacity Estimation →](01-capacity-estimation.md)
