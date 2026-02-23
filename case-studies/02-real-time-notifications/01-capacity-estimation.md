# 1. Capacity Estimation

> Before designing anything, know the numbers. The difference between 2K QPS and 100K QPS is the difference between "one server" and "distributed infrastructure."

---

## 📊 Traffic Estimation

### Daily Volume

```
Total registered users:    50,000,000
Daily Active Users (DAU):  10,000,000
Notifications per DAU/day: 20 (average across all types)

Total notifications/day:   10M × 20 = 200,000,000 (200M)

Breakdown by type:
  In-app:        200M × 0.70 = 140M/day  (every notification has in-app)
  Push:          200M × 0.40 =  80M/day  (60% opt-in × some types)
  Email:         200M × 0.15 =  30M/day  (mostly transactional)
  SMS:           200M × 0.02 =   4M/day  (2FA + payment only)
```

### QPS Calculation

```
Average QPS:
  200M notifications / 86,400 seconds = ~2,315 notifications/sec

Peak QPS (assume 5x average during flash sale):
  2,315 × 5 = ~11,575 notifications/sec

Extreme peak (broadcast to all users in 30 minutes):
  50M / 1,800 seconds = ~27,800 notifications/sec

Design target: 100,000 notifications/sec
  (headroom for concurrent flash sale + broadcast + organic traffic)
```

### QPS by Channel (Peak)

| Channel | Peak QPS | Processing Time | Workers Needed |
|---------|----------|----------------|---------------|
| **In-App (WebSocket)** | 70,000/sec | < 1ms per push | 10-20 WS servers |
| **Push (APNs/FCM)** | 40,000/sec | ~50ms per batch of 1000 | 5-10 workers |
| **Email (SMTP/SES)** | 15,000/sec | ~100ms per email | 10-15 workers |
| **SMS (Twilio)** | 2,000/sec | ~200ms per SMS | 3-5 workers |

---

## 💾 Storage Estimation

### Notification Records

```
Single notification record:
  id:              16 bytes (UUID)
  user_id:         16 bytes
  type:             1 byte (enum)
  priority:         1 byte (enum)
  title:           50 bytes avg
  body:           200 bytes avg
  data (JSON):    100 bytes avg
  channels:         2 bytes (bitmask)
  status:           1 byte
  created_at:       8 bytes
  read_at:          8 bytes
  metadata:        50 bytes avg
  ─────────────────────────
  Total:          ~450 bytes per notification per user

Daily storage:
  200M notifications × 450 bytes = ~90 GB/day

90-day retention (in-app):
  90 × 90 GB = ~8.1 TB

Annual (for audit trail of transactional):
  365 × 90 GB = ~32.8 TB
```

### Delivery Log Records

```
Each notification generates 1.5 delivery records (multi-channel):
  200M × 1.5 = 300M delivery records/day
  Each record: ~200 bytes

  300M × 200B = ~60 GB/day
  90-day: ~5.4 TB
```

### Device Token Storage

```
Users with push enabled: 50M × 60% = 30M users
Average devices per user: 1.5
Total tokens: 45M

Token record:
  user_id:      16 bytes
  platform:      1 byte
  token:       200 bytes (FCM/APNs tokens)
  last_active:   8 bytes
  created_at:    8 bytes
  ─────────────────────
  Total: ~250 bytes

Storage: 45M × 250B = ~11.25 GB (fits in memory easily)
```

### User Preferences

```
Users: 50M
Notification types: ~10
Channels: 4

Preference matrix:
  50M users × 10 types × 4 channels = 2B preference entries
  But only store NON-DEFAULT preferences (most users keep defaults):
  Estimated: 5% customize = 100M entries × 20 bytes = ~2 GB
```

### Storage Summary

| Data | Daily Growth | 90-Day Total | Storage Type |
|------|-------------|-------------|-------------|
| Notification records | 90 GB | 8.1 TB | Cassandra / DynamoDB |
| Delivery logs | 60 GB | 5.4 TB | Cassandra / DynamoDB |
| Device tokens | ~11 GB (static) | ~11 GB | Redis + MySQL |
| User preferences | ~2 GB (static) | ~2 GB | Redis + MySQL |
| Templates | < 100 MB | < 100 MB | MySQL |
| **Total** | **~150 GB/day** | **~13.5 TB** | |

---

## 🌐 Bandwidth Estimation

### WebSocket Connections

```
Peak concurrent connections: 5,000,000

Per connection:
  TCP overhead: ~10 KB (socket buffers, kernel memory)
  WebSocket frame overhead: minimal per message
  Average message size: ~500 bytes (notification payload)
  Messages per connection per minute: ~2 (heartbeat + notifications)

Memory per connection:
  Kernel: ~10 KB
  Application state: ~2 KB (user_id, channels subscribed, metadata)
  Total: ~12 KB per connection

Total memory for 5M connections:
  5M × 12 KB = 60 GB

Split across servers:
  Assume 500K connections per server
  → 10 WebSocket servers, each holding 6 GB for connections
  → Use servers with 16-32 GB RAM
```

### Outbound Bandwidth

```
In-app notifications (WebSocket):
  70K messages/sec × 500 bytes = 35 MB/sec = 280 Mbps

Push notifications (to APNs/FCM):
  40K/sec × 1 KB = 40 MB/sec = 320 Mbps

Email (via SES/SMTP):
  15K/sec × 10 KB avg = 150 MB/sec = 1.2 Gbps

SMS (via Twilio API):
  2K/sec × 500 bytes = 1 MB/sec = 8 Mbps

Total outbound: ~1.8 Gbps during peak
```

---

## 💰 Infrastructure Cost Estimate

| Component | Spec | Count | Monthly Cost |
|-----------|------|-------|-------------|
| **WebSocket servers** | 8 vCPU, 32 GB RAM | 10-15 | $2,000-$4,500 |
| **Queue workers** | 4 vCPU, 8 GB RAM | 20-30 | $1,500-$3,000 |
| **Kafka cluster** | 8 vCPU, 32 GB, 500 GB SSD | 5 brokers | $2,500 |
| **Cassandra cluster** | 8 vCPU, 32 GB, 2 TB SSD | 6 nodes | $3,600 |
| **Redis cluster** | 16 GB RAM per node | 3 nodes | $1,200 |
| **MySQL (metadata)** | 8 vCPU, 32 GB | 1 primary + 1 replica | $600 |
| **Load balancer** | L4+L7 | 2 | $300 |
| **Email (SES)** | 30M emails/day | — | $3,000 |
| **SMS (Twilio)** | 4M SMS/day | — | $30,000+ |
| **Push (FCM/APNs)** | 80M pushes/day | — | Free (FCM) / minimal |
| **CDN / bandwidth** | ~1.8 Gbps peak | — | $2,000 |
| **Total** | | | **~$47K-50K/mo** |

> 💡 **SMS is the biggest cost by far** ($30K+/mo). This is why every system minimizes SMS usage — only for 2FA and critical payment alerts.

---

## 📋 Capacity Cheat Sheet

```
┌─────────────────────────────────────────┐
│       NOTIFICATION SYSTEM NUMBERS       │
├─────────────────────────────────────────┤
│  Users:           50M registered        │
│  DAU:             10M                   │
│  Concurrent:      5M WebSocket conns    │
│  Notifications:   200M/day              │
│  Peak QPS:        100K notif/sec        │
│  Avg QPS:         2.3K notif/sec        │
│  Storage:         ~150 GB/day           │
│  WS Servers:      10-15 @ 500K conn/ea  │
│  WS Memory:       ~60 GB total          │
│  Bandwidth:       ~1.8 Gbps peak        │
│  Monthly cost:    ~$50K                 │
│  SMS cost:        60% of total bill     │
└─────────────────────────────────────────┘
```

---

## ⬅️ [← Overview](00-overview.md) · [High-Level Architecture →](02-high-level-architecture.md)
