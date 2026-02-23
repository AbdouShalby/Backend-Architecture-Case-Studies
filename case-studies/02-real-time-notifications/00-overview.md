# 0. Overview & Requirements

> A notification system isn't a "send message" API. It's a **real-time delivery infrastructure** with multi-channel routing, preference management, fan-out at scale, and guaranteed delivery for critical paths.

---

## 📋 Problem Statement

Build a unified notification platform for a social/e-commerce app with 50M users that:

1. Delivers **real-time in-app notifications** (visible within 1 second)
2. Sends **push notifications** to mobile devices (iOS/Android)
3. Sends **transactional emails** (order confirmation, password reset, receipts)
4. Sends **SMS** for critical/security events (2FA codes, payment alerts)
5. Respects **user preferences** (opt-in/opt-out per channel per notification type)
6. Supports **mass fan-out** (product launch → notify all 50M users)
7. Provides **notification history** (inbox with read/unread state)

---

## ✅ Functional Requirements

### Notification Types

| Type | Channel(s) | Latency Target | Delivery Guarantee | Example |
|------|-----------|---------------|-------------------|---------|
| **Transactional** | Email + SMS + In-App | < 30s (email/SMS), < 1s (in-app) | Must deliver | Order confirmation, 2FA code, password reset |
| **Real-Time** | In-App + Push | < 1s | Best effort | New message, new follower, item shipped |
| **Marketing** | Email + Push | < 5 min | Best effort | Weekly digest, promotional offer |
| **System** | In-App + Push | < 5s | Should deliver | Maintenance notice, policy update |
| **Broadcast** | All channels | < 30 min (full fan-out) | Best effort | New feature announcement, platform-wide alert |

### Core Features

| Feature | Priority | Description |
|---------|----------|-------------|
| Multi-channel dispatch | P0 | Route to in-app, push, email, SMS based on type |
| User preferences | P0 | Per-user, per-channel, per-notification-type settings |
| Real-time delivery | P0 | In-app notifications via persistent connection |
| Notification inbox | P0 | List of all in-app notifications with read/unread |
| Delivery tracking | P1 | Track sent/delivered/read status per notification |
| Template system | P1 | Reusable templates with variable substitution |
| Batch/broadcast | P1 | Send to millions of users efficiently |
| Rate limiting | P1 | Max notifications per user per hour (anti-spam) |
| Scheduling | P2 | Send at scheduled time or optimal engagement time |
| A/B testing | P2 | Test different notification content |

---

## 📊 Non-Functional Requirements

| Requirement | Target | Justification |
|-------------|--------|---------------|
| **In-app latency** | < 1s (p95) | Real-time feel — user sees notification instantly |
| **Push latency** | < 5s (p95) | Acceptable for mobile push (APNs/FCM add ~1-3s) |
| **Email latency** | < 30s (p95) | Transactional emails should arrive quickly |
| **SMS latency** | < 10s (p95) | 2FA codes are time-sensitive |
| **Availability** | 99.95% | Critical path for 2FA and payment notifications |
| **Throughput** | 100K notifications/sec (peak) | Flash sale or broadcast event |
| **Concurrent connections** | 5M WebSocket connections | ~10% of MAU online at peak |
| **Notification inbox** | < 100ms query time | Fast inbox loading |
| **Data retention** | 90 days in-app, forever for transactional | Inbox visible 90 days, audit trail permanent |

---

## 👤 User Preferences Model

```
User Preference Matrix:
  User × Notification Type × Channel = enabled/disabled

Example (user_42):
  ┌────────────────────┬─────────┬──────┬───────┬─────┐
  │ Notification Type  │ In-App  │ Push │ Email │ SMS │
  ├────────────────────┼─────────┼──────┼───────┼─────┤
  │ Order updates      │ ✅      │ ✅   │ ✅    │ ❌  │
  │ New messages       │ ✅      │ ✅   │ ❌    │ ❌  │
  │ Promotions         │ ✅      │ ❌   │ ✅    │ ❌  │
  │ Security alerts    │ ✅      │ ✅   │ ✅    │ ✅  │ ← cannot disable
  │ System updates     │ ✅      │ ❌   │ ❌    │ ❌  │
  └────────────────────┴─────────┴──────┴───────┴─────┘

Rules:
  - Security/2FA: always ON for all channels (non-negotiable)
  - In-App: always ON (it's passive, no interruption)
  - Default: everything ON except SMS (user opts-in to SMS)
```

---

## 🔢 Key Assumptions

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| Total registered users | 50M | Given in prompt |
| Monthly Active Users (MAU) | 30M (60%) | Industry standard for active platforms |
| Daily Active Users (DAU) | 10M (20%) | Typical DAU/MAU ratio |
| Peak concurrent users | 5M (10%) | Soccer match or flash sale peak |
| Notifications per DAU per day | 20 avg | Social + transactional + system |
| Total notifications/day | 200M | 10M DAU × 20 notifications |
| Peak QPS | 100,000 notif/sec | 10x average during flash sale |
| Average QPS | ~2,300 notif/sec | 200M / 86,400 |
| Notification channels per event | 1.5 avg | Most events → in-app + one other |
| Push notification opt-in rate | 60% | Industry average |
| Email open rate | 20-25% | Industry average |
| SMS usage | < 5% of notifications | Only security/payment |

---

## 🔲 Core Entities

```
Notification
  ├── id (unique)
  ├── type (transactional, real_time, marketing, system, broadcast)
  ├── priority (critical, high, normal, low)
  ├── template_id → Template
  ├── data (JSON payload for template variables)
  ├── channels[] (in_app, push, email, sms)
  └── target → User or UserSegment

User Preference
  ├── user_id → User
  ├── notification_type
  ├── channel
  └── enabled (boolean)

Device Token
  ├── user_id → User
  ├── platform (ios, android, web)
  ├── token
  └── last_active_at

Notification Delivery
  ├── notification_id → Notification
  ├── user_id → User
  ├── channel
  ├── status (pending, sent, delivered, read, failed)
  └── timestamps (sent_at, delivered_at, read_at)
```

---

## ⬅️ [← Case Study Index](README.md) · [Capacity Estimation →](01-capacity-estimation.md)
