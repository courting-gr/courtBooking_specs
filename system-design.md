# Court Booking Platform — System Design Blueprint

## 1. System Overview

The Court Booking Platform is a multi-tenant sports court reservation system serving three user types through two frontend applications backed by two microservices. The platform enables court owners to manage sports facilities while customers discover, book, and pay for court reservations.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                 │
│                                                                     │
│   ┌─────────────────────┐          ┌─────────────────────────┐     │
│   │   Mobile App         │          │   Admin Web Portal       │     │
│   │   (Flutter)          │          │   (React)                │     │
│   │                      │          │                          │     │
│   │   • Court discovery  │          │   • Court management     │     │
│   │   • Booking flow     │          │   • Booking management   │     │
│   │   • Payments         │          │   • Analytics/revenue    │     │
│   │   • Open matches     │          │   • Pricing/policies     │     │
│   │   • Notifications    │          │   • Support management   │     │
│   │   • Support tickets  │          │                          │     │
│   └────────┬────────────┘          └────────────┬─────────────┘     │
│            │                                     │                   │
└────────────┼─────────────────────────────────────┼───────────────────┘
             │          HTTPS / WSS                │
             ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     INGRESS LAYER                                   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │   DigitalOcean Load Balancer + NGINX Ingress Controller     │   │
│   │   • SSL termination (Let's Encrypt)                         │   │
│   │   • Path-based routing                                      │   │
│   │   • Rate limiting                                           │   │
│   │   • JWT presence check                                      │   │
│   │   • Security headers (HSTS, X-Content-Type-Options,         │   │
│   │     X-Frame-Options, CSP, Referrer-Policy)                  │   │
│   └──────────────┬──────────────────────────┬───────────────────┘   │
│                  │                          │                        │
└──────────────────┼──────────────────────────┼────────────────────────┘
                   │                          │
                   ▼                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER (DOKS)                          │
│                                                                     │
│   ┌─────────────────────┐          ┌─────────────────────────┐     │
│   │  Platform Service    │          │  Transaction Service     │     │
│   │  (Spring Boot)       │◄────────►│  (Spring Boot)           │     │
│   │                      │ internal │                          │     │
│   │  Auth, Users, Courts │  HTTP    │  Bookings, Payments,     │     │
│   │  Weather, Analytics  │          │  Notifications, Waitlist │     │
│   │  Support, Flags      │          │  Matches, Split Pay      │     │
│   └──────────┬───────────┘          └──────────┬──────────────┘     │
│              │                                  │                    │
└──────────────┼──────────────────────────────────┼────────────────────┘
               │                                  │
               ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                     │
│                                                                     │
│   ┌──────────────┐  ┌────────┐  ┌──────────┐  ┌────────────────┐  │
│   │ PostgreSQL    │  │ Redis  │  │ Redpanda │  │ DO Spaces      │  │
│   │ + PostGIS     │  │ 2GB    │  │Serverless│  │ (S3-compat)    │  │
│   │               │  │        │  │          │  │                │  │
│   │ • platform    │  │ Cache  │  │ 7 topics │  │ Court images   │  │
│   │   schema      │  │ Queues │  │ 21 event │  │ Attachments    │  │
│   │ • transaction │  │ PubSub │  │ types    │  │ Assets         │  │
│   │   schema      │  │ Flags  │  │          │  │                │  │
│   │ • encryption  │  │        │  │          │  │                │  │
│   │   at rest     │  │        │  │          │  │                │  │
│   └──────────────┘  └────────┘  └──────────┘  └────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Frontend Applications

### Mobile App (Flutter) — `court-booking-mobile-app`

Target users: Customers (CUSTOMER role)

| Capability | Backend Service | Protocol |
|------------|----------------|----------|
| OAuth login + biometric auth | Platform Service | REST |
| Court search (map, filters, geospatial) | Platform Service | REST |
| Weather forecast display | Platform Service | REST |
| Favorites and preferences | Platform Service | REST |
| Booking creation and management | Transaction Service | REST |
| Payments (Stripe, Apple Pay, Google Pay) | Transaction Service | REST |
| Open match join/create | Transaction Service | REST |
| Waitlist join | Transaction Service | REST |
| Split payments | Transaction Service | REST |
| Real-time availability updates | Transaction Service | WebSocket |
| Push notifications | Transaction Service | FCM |
| In-app notifications | Transaction Service | WebSocket |
| Support tickets + diagnostic logs | Platform Service | REST |

### Admin Web Portal (React) — `court-booking-admin-web`

Target users: Court Owners (COURT_OWNER role), Support Agents (SUPPORT_AGENT role), Platform Admins (PLATFORM_ADMIN role)

| Capability | Backend Service | Protocol |
|------------|----------------|----------|
| OAuth login | Platform Service | REST |
| Court CRUD + availability management | Platform Service | REST |
| Holiday calendar management | Platform Service | REST |
| Court verification (admin) | Platform Service | REST |
| Stripe Connect onboarding | Platform Service | REST (redirect to Stripe) |
| Booking calendar + confirm/reject | Transaction Service | REST |
| Manual booking creation | Transaction Service | REST |
| Analytics and revenue dashboards | Platform Service | REST (read replica) |
| Dynamic pricing configuration | Platform Service | REST |
| Cancellation policy management | Platform Service | REST |
| Promo code management | Platform Service | REST |
| Feature flag management (admin) | Platform Service | REST |
| User management (admin) | Platform Service | REST |
| Support ticket management (admin/agent) | Platform Service | REST |
| Security alerts dashboard (admin/agent) | Platform Service | REST |
| IP blocklist management (admin) | Platform Service | REST |
| Real-time booking updates | Transaction Service | WebSocket |

## 3. API Routing

NGINX Ingress routes requests by URL path prefix:

```
                    ┌─────────────────────────────────┐
                    │       NGINX Ingress              │
                    │       (path-based routing)       │
                    └───────────┬──────────────────────┘
                                │
            ┌───────────────────┼───────────────────────┐
            │                   │                       │
            ▼                   ▼                       ▼
   Platform Service    Transaction Service      Transaction Service
                                                   (WebSocket)
   /api/auth/*          /api/bookings/*
   /api/users/*         /api/payments/*            /ws/*
   /api/courts/*        /api/notifications/*
   /api/weather/*       /api/waitlist/*
   /api/analytics/*     /api/matches/*
   /api/promo-codes/*   /api/split-payments/*
   /api/feature-flags/*
   /api/admin/*
   /api/support/*
   /api/holidays/*
```

Both services validate JWT tokens independently using a shared RS256 public key. NGINX performs initial token presence check; services do full signature verification, expiration, and role-based authorization.

## 4. Service Interaction Patterns

### Synchronous (REST)

```
Mobile App ──REST──► NGINX ──► Platform Service
                           ──► Transaction Service

Admin Web  ──REST──► NGINX ──► Platform Service
                           ──► Transaction Service

Transaction Service ──internal HTTP──► Platform Service
  (court validation, pricing calculation, promo codes, Stripe Connect status)
  Auth: mTLS via Istio (staging/prod), API key (dev/test)
```

#### Internal Service-to-Service API

Transaction Service calls Platform Service for operations that involve business logic. These endpoints use the `/internal/*` path prefix, are **not** exposed through NGINX Ingress, and are only reachable within the Kubernetes cluster network.

| Lookup | Method | Path | Rationale |
|--------|--------|------|-----------|
| Court availability validation | GET | `/internal/courts/{courtId}/validate-slot?date=&startTime=&endTime=` | Involves availability windows, overrides, and existing booking conflict checks |
| Pricing calculation | GET | `/internal/courts/{courtId}/calculate-price?date=&startTime=&endTime=` | Involves base price, pricing rules, special date pricing, and promo validation |
| Promo code validation | POST | `/internal/promo-codes/validate` | Validates promo code eligibility (limits, dates, court type) without incrementing usage |
| Promo code redemption | POST | `/internal/promo-codes/redeem` | Increments usage count when booking is confirmed (idempotent by bookingId) |
| Promo code rollback | POST | `/internal/promo-codes/rollback` | Decrements usage count when booking is cancelled/refunded (idempotent by bookingId) |
| Stripe Connect status | GET | `/internal/users/{userId}/stripe-connect-status` | Fresh status check before payment capture (prevents capturing to disabled accounts) |

**Promo Code Flow:**
1. Customer enters promo code → Transaction Service calls `/internal/promo-codes/validate`
2. Validation returns discount details or error → shown in booking preview
3. Booking confirmed + payment captured → Transaction Service calls `/internal/promo-codes/redeem`
4. Booking cancelled/refunded → Transaction Service calls `/internal/promo-codes/rollback`

**Stripe Connect Status Sync:**
Platform Service owns `users.stripe_connect_status` and receives Stripe webhooks. Transaction Service needs this for payment processing:
1. **Kafka event** (`STRIPE_CONNECT_STATUS_CHANGED`) — Platform Service publishes when status changes via webhook. Transaction Service updates its cache.
2. **Fresh fetch before capture** — Transaction Service calls `/internal/users/{userId}/stripe-connect-status` before capturing payments to ensure we never capture to a RESTRICTED/DISABLED account.
3. **Cross-schema view** (`v_user_basic`) — Used for display purposes and non-critical reads.

Simple read-only lookups use **cross-schema database views** instead of HTTP calls (see [Data Ownership](#5-data-ownership)):

| Lookup | View | Fields |
|--------|------|--------|
| Court summary | `v_court_summary` | id, owner_id, name, type, duration, capacity, base_price, timezone, confirmation_mode, version |
| User basic info | `v_user_basic` | id, email, name, phone, role, language, stripe_connect_account_id, stripe_connect_status, stripe_customer_id, vat_registered, vat_number, status |
| Cancellation policy tiers | `v_court_cancellation_tiers` | court_id, threshold_hours, refund_percent, sort_order |
| Player skill level | `v_user_skill_level` | user_id, court_type, level |

#### Internal API Authentication

| Environment | Mechanism | Details |
|-------------|-----------|---------|
| dev / test | API key | Shared secret via `INTERNAL_API_KEY` env var. Passed as `X-Internal-Api-Key` header. Validated by a Spring Security filter on `/internal/**` paths. |
| staging / prod | mTLS | Istio sidecar handles mutual TLS between pods. No application-level auth needed — Istio `AuthorizationPolicy` restricts `/internal/**` to Transaction Service's service account. |

### Asynchronous (Kafka Events)

Formal event schemas are defined in [`docs/api/kafka-event-contracts.json`](docs/api/kafka-event-contracts.json) — the single source of truth for all async interfaces. Both services share event classes via `court-booking-common`.

```
┌────────────────────┐                    ┌────────────────────┐
│ Transaction Service │                    │ Platform Service    │
│                     │                    │                     │
│  booking-events ────┼──► Kafka ────────►─┤  cache invalidation │
│  notification-events┼──► Kafka ──► FCM   │  availability update│
│  waitlist-events ───┼──► Kafka           │                     │
│  match-events ──────┼──► Kafka ────────►─┤  map display update │
│                     │                    │                     │
│                     │    Kafka ◄─────────┤  court-update-events│
│  pricing/policy ◄───┤                    │  (pricing, avail,   │
│  cache update       │    Kafka ◄─────────┤   policy, deletion) │
│                     │                    │  analytics-events   │
│                     │                    │  notification-events│
│  security-events ───┼──► Kafka ────────►─┤  security alerts    │
│                     │                    │  security-events ───┤
└────────────────────┘                    └────────────────────┘
```

Both services publish to `notification-events`: Transaction Service for booking/payment/match/waitlist/split-payment notifications; Platform Service for verification, subscription, support, weather, and reminder notifications. Transaction Service's notification processor consumes all events regardless of source.

#### Topics and Partitioning

| Topic | Partition Key | Producer | Consumer(s) | Events | Purpose |
|-------|--------------|----------|-------------|--------|---------|
| `booking-events` | `courtId` | Transaction | Platform | BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, BOOKING_MODIFIED, BOOKING_COMPLETED, SLOT_HELD, SLOT_RELEASED | Availability cache invalidation, WebSocket broadcasts |
| `notification-events` | `userId` | Both | Transaction | NOTIFICATION_REQUESTED | Notification dispatch routing (FCM, WebSocket, SendGrid, email) |
| `court-update-events` | `courtId` | Platform | Transaction | COURT_UPDATED, PRICING_UPDATED, AVAILABILITY_UPDATED, CANCELLATION_POLICY_UPDATED, COURT_DELETED, STRIPE_CONNECT_STATUS_CHANGED | Pricing/availability/policy sync, Stripe Connect status for payment eligibility |
| `match-events` | `matchId` | Transaction | Platform | MATCH_CREATED, MATCH_UPDATED, MATCH_CLOSED | Open match map display and search index updates |
| `waitlist-events` | `courtId` | Transaction | Transaction | WAITLIST_SLOT_FREED, WAITLIST_HOLD_EXPIRED | FIFO waitlist processing on cancellations |
| `analytics-events` | `courtId` | Both | Platform | BOOKING_ANALYTICS, REVENUE_ANALYTICS, PROMO_CODE_REDEEMED | Dashboard metrics, revenue tracking, promo code usage |
| `security-events` | `userId` | Both | Platform | SECURITY_ALERT | Security monitoring, abuse detection, automated response triggers |

#### Event Envelope

Every Kafka message value follows a common envelope schema:

```json
{
  "eventId": "uuid — consumer-side idempotency key",
  "eventType": "discriminator — determines payload schema",
  "source": "platform-service | transaction-service",
  "timestamp": "ISO 8601 UTC",
  "traceId": "W3C Trace Context ID (Req 16.7)",
  "spanId": "OpenTelemetry span ID",
  "correlationId": "optional — links related events across topics",
  "payload": { }
}
```

- `eventId` enables consumer-side deduplication (at-least-once delivery with Redpanda Serverless)
- `traceId` / `spanId` enable end-to-end distributed tracing across async boundaries
- `correlationId` links related events (e.g., a booking cancellation → waitlist slot freed → notification)

### Real-Time (WebSocket + Redis Pub/Sub)

```
┌──────────┐    WSS     ┌─────────────────────────────────────┐
│ Mobile   │◄──────────►│  Transaction Service (Pod 1)        │
│ App      │            │       │                              │
└──────────┘            │       ▼                              │
                        │  Redis Pub/Sub ◄──► Pod 2, Pod 3... │
┌──────────┐    WSS     │       │                              │
│ Admin    │◄──────────►│       ▼                              │
│ Web      │            │  Broadcasts: availability updates,   │
└──────────┘            │  booking status, match events,       │
                        │  in-app notifications                │
                        └─────────────────────────────────────┘
```

#### Availability Update Latency Budget (Req 19.20 — 2-second SLA)

The end-to-end availability propagation chain must complete within 2 seconds:

| Step | Target | Notes |
|------|--------|-------|
| Transaction Service → Kafka produce | ≤ 100ms | Redpanda Serverless (native Kafka protocol) |
| Kafka → Platform Service consume | ≤ 150ms | Redpanda Serverless poll interval |
| Redis cache invalidation | ≤ 50ms | Managed Redis, same VPC |
| WebSocket broadcast via Redis Pub/Sub | ≤ 100ms | Pod-to-pod via Redis |
| **Total budget** | **≤ 400ms typical, ≤ 1.2s p95** | 800ms headroom for spikes |

**Monitoring:**
- `availability_update_latency_seconds` histogram metric (measured from Kafka event timestamp to WebSocket broadcast)
- `kafka_produce_latency_seconds` histogram (Redpanda produce time)
- `kafka_consume_lag_seconds` gauge (consumer lag per partition)
- Grafana alert at p95 > 1.2s

**Redpanda Serverless Details:**

Redpanda Serverless is a fully-managed, Kafka-compatible streaming platform with pay-as-you-go pricing. Key characteristics:

| Feature | Specification |
|---------|---------------|
| Protocol | Native Kafka API (no HTTP overhead) |
| Max write throughput | 100 MB/s |
| Max read throughput | 300 MB/s |
| Max partitions | 5,000 |
| SLA | 99.9% |
| Networking | Public or AWS PrivateLink |
| Authentication | SASL/SCRAM, ACLs |
| Free trial | $100 credits for 14 days |

**Strimzi Migration Criteria:**

If Redpanda Serverless proves insufficient, migrate to Strimzi (self-hosted Kafka on DOKS). Trigger migration when ANY of these conditions persist for 7+ consecutive days:

| Metric | Threshold | Rationale |
|--------|-----------|-----------|
| `availability_update_latency_seconds` p95 | > 1.2s | SLA breach risk |
| `availability_update_latency_seconds` p99 | > 2.0s | Hard SLA violation |
| `kafka_produce_latency_seconds` p95 | > 200ms | Latency concern |
| `kafka_consume_lag_seconds` | > 5s sustained | Consumer falling behind |
| Redpanda monthly cost | > €150/month | Cost parity with self-hosted |
| Redpanda availability | < 99.9% monthly | Reliability concern |

**Migration Runbook:** See `infrastructure/docs/kafka-migration-runbook.md` (to be created in Phase 1).

#### Race Condition Mitigation

Stale availability can cause double-bookings. Mitigations:

1. **Optimistic Locking**: Bookings table has `version` column. Booking creation uses `WHERE version = :expected` to detect concurrent modifications.

2. **Slot Hold Mechanism**: Before payment, Transaction Service holds the slot for 5 minutes via `SLOT_HELD` event. Other booking attempts for the same slot return `409 Conflict`.

3. **Database-Level Uniqueness**: Unique constraint on `(court_id, date, start_time, end_time, status)` WHERE `status NOT IN ('CANCELLED', 'REJECTED')` prevents duplicate confirmed bookings.

4. **Idempotency Keys**: All booking creation requests require `X-Idempotency-Key` header. Duplicate requests return the original response.

5. **Cache Invalidation on Conflict**: If a booking fails due to conflict, Platform Service immediately invalidates the availability cache for that court+date.

```sql
-- Unique constraint preventing double-bookings
CREATE UNIQUE INDEX uq_bookings_slot_active 
ON transaction.bookings (court_id, date, start_time, end_time) 
WHERE status NOT IN ('CANCELLED', 'REJECTED');
```

### Scheduled Jobs (Quartz in Transaction Service)

- Pending booking confirmation timeouts
- Split payment deadlines
- Waitlist slot hold expiration
- Recurring booking creation (weekly advance)
- Booking reminders
- Payment reconciliation (daily — see below)
- Recurring booking price updates (on PRICING_UPDATED event)
- Clustered via JDBC job store (`isClustered=true`)

#### Payment Reconciliation Job

A daily Quartz job (`PaymentReconciliationJob`) reconciles payment state between the platform and Stripe:

1. Queries bookings with `payment_status = 'AUTHORIZED'` older than 30 minutes — captures or cancels the PaymentIntent based on booking status
2. Queries Stripe for PaymentIntents in `requires_capture` state with no matching booking — cancels them
3. Checks for bookings marked `CONFIRMED` with `payment_status = 'PENDING'` — flags for manual review
4. Logs all reconciliation actions to `transaction.audit_logs`

This catches edge cases where webhook delivery fails or the application crashes between payment authorization and booking confirmation.

#### Recurring Booking Pricing Updates (Req 8.12)

When a court owner changes pricing (base price or pricing rules), Transaction Service must handle existing recurring bookings appropriately. This is triggered by the `PRICING_UPDATED` Kafka event.

**Scope:**
- Only affects **future unconfirmed** recurring booking instances
- Already confirmed/completed/cancelled instances are NOT affected
- Instances with `payment_status = 'CAPTURED'` are NOT affected (already paid)

**Processing Flow:**

```
Platform Service                    Transaction Service
      │                                    │
      │  PRICING_UPDATED event             │
      │  (courtId, effectiveFrom,          │
      │   basePriceCents, pricingRules)    │
      ├───────────────────────────────────►│
      │                                    │
      │                          1. Query affected bookings:
      │                             - recurring_group_id IS NOT NULL
      │                             - court_id = event.courtId
      │                             - date >= event.effectiveFrom
      │                             - status IN ('PENDING_CONFIRMATION', 'CONFIRMED')
      │                             - payment_status NOT IN ('CAPTURED', 'REFUNDED')
      │                                    │
      │                          2. For each affected booking:
      │                             - Recalculate price using new rules
      │                             - Update total_amount_cents, platform_fee_cents, court_owner_net_cents
      │                             - Record price change in audit_logs
      │                                    │
      │                          3. Group by recurring_group_id + user_id
      │                             - Send ONE notification per recurring series
      │                                    │
      │  NOTIFICATION_REQUESTED            │
      │  (RECURRING_BOOKING_PRICE_CHANGED) │
      │◄───────────────────────────────────┤
      │                                    │
```

**Notification Content:**
- Type: `RECURRING_BOOKING_PRICE_CHANGED` (new notification type)
- Urgency: `STANDARD`
- Title: "Price change for your recurring booking"
- Body: "The price for your {courtName} booking on {dayOfWeek}s has changed from €{oldPrice} to €{newPrice}. This affects {count} upcoming bookings starting {firstAffectedDate}."
- Deep link: `/bookings/recurring/{recurringGroupId}`

**Customer Options (via app):**
1. **Accept** — No action needed, bookings continue with new price
2. **Cancel affected instances** — Customer can cancel individual instances (normal cancellation policy applies)
3. **Cancel entire series** — Customer can cancel all remaining instances at once

**Edge Cases:**

| Scenario | Behavior |
|----------|----------|
| Price increases | Update price, notify customer, customer can cancel with refund per policy |
| Price decreases | Update price, notify customer (good news!), no action needed |
| Booking in PENDING_CONFIRMATION | Update price before court owner confirms |
| Payment already authorized but not captured | Cancel old PaymentIntent, create new one with updated amount |
| effectiveFrom is in the past | Apply to all future instances regardless of effectiveFrom |
| Multiple price changes in short period | Coalesce notifications (max 1 per hour per recurring series) |

**Database Updates:**

```sql
-- Update affected bookings
UPDATE transaction.bookings
SET 
  total_amount_cents = :newAmount,
  platform_fee_cents = :newPlatformFee,
  court_owner_net_cents = :newCourtOwnerNet,
  updated_at = NOW()
WHERE 
  court_id = :courtId
  AND recurring_group_id IS NOT NULL
  AND date >= :effectiveFrom
  AND status IN ('PENDING_CONFIRMATION', 'CONFIRMED')
  AND payment_status NOT IN ('CAPTURED', 'REFUNDED', 'PARTIALLY_REFUNDED');

-- Audit log entry
INSERT INTO transaction.audit_logs (booking_id, action, actor_type, details, created_at)
SELECT 
  id, 
  'PRICE_UPDATED_BY_COURT_OWNER',
  'SYSTEM',
  jsonb_build_object(
    'previousAmount', total_amount_cents,
    'newAmount', :newAmount,
    'reason', 'PRICING_UPDATED_EVENT',
    'eventId', :eventId
  ),
  NOW()
FROM transaction.bookings
WHERE court_id = :courtId AND recurring_group_id IS NOT NULL AND date >= :effectiveFrom;
```

**Metrics:**
- `recurring_booking_price_updates_total` — Counter of price updates processed
- `recurring_booking_price_update_notifications_total` — Counter of notifications sent
- `recurring_booking_cancellations_after_price_change_total` — Counter of cancellations within 24h of price change notification

## 5. Data Ownership

Shared PostgreSQL instance, separate schemas with strict write boundaries:

```
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL + PostGIS                       │
│                                                              │
│  ┌───────────────────────────┐  ┌────────────────────────────┐ │
│  │  platform schema          │  │  transaction schema         │ │
│  │  (owned by Platform)      │  │  (owned by Transaction)     │ │
│  │                           │  │                             │ │
│  │  users                    │  │  bookings                   │ │
│  │  oauth_providers          │  │  payments                   │ │
│  │  refresh_tokens           │  │  audit_logs                 │ │
│  │  verification_requests    │  │  notifications              │ │
│  │  courts                   │  │  device_tokens              │ │
│  │  availability_windows     │  │  waitlists ⏳               │ │
│  │  availability_overrides   │  │  open_matches ⏳            │ │
│  │  favorites                │  │  match_players ⏳           │ │
│  │  preferences              │  │  match_join_requests ⏳     │ │
│  │  skill_levels             │  │  match_messages ⏳          │ │
│  │  court_ratings ⏳         │  │  split_payments ⏳          │ │
│  │  pricing_rules            │  │  split_payment_shares ⏳    │ │
│  │  special_date_pricing ⏳  │  │  scheduled_jobs (Quartz)    │ │
│  │  cancellation_tiers       │  │                             │ │
│  │  promo_codes ⏳           │  │                             │ │
│  │  translations             │  │                             │ │
│  │  feature_flags            │  │                             │ │
│  │  support_tickets          │  │                             │ │
│  │  support_messages         │  │                             │ │
│  │  support_attachments      │  │                             │ │
│  │  court_owner_audit_logs   │  │                             │ │
│  │  reminder_rules           │  │                             │ │
│  │  court_owner_notif_prefs  │  │                             │ │
│  │  court_defaults           │  │                             │ │
│  │  security_alerts          │  │                             │ │
│  │  ip_blocklist             │  │                             │ │
│  │  failed_auth_attempts     │  │                             │ │
│  └───────────────────────────┘  └────────────────────────────┘ │
│                                                              │
│  Cross-schema: Transaction has READ-ONLY access to Platform  │
│  via database views (v_court_summary, v_user_basic)          │
│                                                              │
│  Read replica used by Platform Service for analytics queries │
└─────────────────────────────────────────────────────────────┘
```

## 6. External Service Integration

```
┌──────────────────┐     ┌──────────────────────────────────────────┐
│ Platform Service  │────►│ OAuth Providers (Google, Facebook, Apple) │
│                   │────►│ OpenWeatherMap (forecast data)            │
└──────────────────┘     └──────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────────────────────────────┐
│ Transaction       │────►│ Stripe (payments, Connect, webhooks)     │
│ Service           │────►│ SendGrid (email notifications)           │
│                   │────►│ Firebase Cloud Messaging (push notifs)   │
└──────────────────┘     └──────────────────────────────────────────┘
```

## 7. Authentication and Authorization Flow

```
┌────────┐   1. OAuth login    ┌──────────────┐   2. Validate    ┌──────────┐
│ Client │ ──────────────────► │ Platform     │ ◄──────────────► │ OAuth    │
│        │                     │ Service      │                   │ Provider │
│        │ ◄────────────────── │              │                   └──────────┘
│        │   3. JWT access     │              │
│        │      + refresh      └──────────────┘
│        │
│        │   4. API request    ┌──────────────┐
│        │   (Bearer token)    │ Any Service  │
│        │ ──────────────────► │              │
│        │                     │ 5. Validate  │
│        │                     │    JWT sig   │
│        │                     │    + role    │
│        │ ◄────────────────── │    + claims  │
│        │   6. Response       └──────────────┘
└────────┘
```

- Access tokens: 15-min lifetime, RS256 signed, validated independently by both services
- Refresh tokens: 30-day lifetime, server-side storage, rotation with replay detection
- Biometric auth: refresh token stored in device secure enclave, gated by fingerprint/face ID
- Four roles: CUSTOMER, COURT_OWNER (with sub-states), SUPPORT_AGENT (read-only support/security), PLATFORM_ADMIN

## 8. Infrastructure Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    DigitalOcean (FRA1 Region)                    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Shared DOKS Cluster (dev / test / staging)                 │ │
│  │  2-4 nodes × 4GB RAM, auto-scaling                          │ │
│  │                                                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐  │ │
│  │  │ dev ns   │  │ test ns  │  │ staging ns               │  │ │
│  │  │ 1 replica│  │ 1 replica│  │ 3 replicas + Istio       │  │ │
│  │  └──────────┘  └──────────┘  └──────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Production DOKS Cluster (dedicated)                        │ │
│  │  3-6 nodes × 8GB RAM, auto-scaling + Istio                 │ │
│  │  + dedicated observability node pool                        │ │
│  │                                                              │ │
│  │  ┌──────────────────────────────────────────────────────┐   │ │
│  │  │ Platform Service (3 pods) + Transaction Service (3)  │   │ │
│  │  │ Prometheus + Grafana + Jaeger + Loki + Kiali         │   │ │
│  │  └──────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────┐  ┌────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Managed PG   │  │Managed │  │ Spaces   │  │ Container    │ │
│  │ + PostGIS    │  │ Redis  │  │ (CDN)    │  │ Registry     │ │
│  │ + replica    │  │        │  │          │  │              │ │
│  └──────────────┘  └────────┘  └──────────┘  └──────────────┘ │
│                                                                  │
│  Redpanda Serverless (external, Kafka-compatible)               │
└─────────────────────────────────────────────────────────────────┘
```

## 9. CI/CD and Deployment Pipeline

```
Developer ──► GitHub PR ──► GitHub Actions Pipeline:

  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Lint    │───►│ Test     │───►│ Scan     │───►│ Docker   │
  │ Check   │    │ Unit     │    │ Trivy    │    │ Build    │
  │         │    │ PBT      │    │ OWASP    │    │ Push     │
  └─────────┘    │ Integr.  │    │ Sonar    │    └────┬─────┘
                 │ Flyway   │    └──────────┘         │
                 │ validate │                         ▼
                 └──────────┘                  ┌──────────────┐
                                               │ PR Approved  │
                                               │ Merge to     │
                                               │ develop      │
                                               └──────┬───────┘
                                                      │
  ┌──────────────────────────────────────────────────┐│
  │           Progressive Deployment                  ││
  │                                                   ▼│
  │  ┌──────────┐    ┌──────────┐    ┌─────────────┐ │
  │  │ Deploy   │───►│ Deploy   │───►│ QA Func.    │ │
  │  │ dev      │    │ test     │    │ Regression  │ │
  │  │ (auto)   │    │ (auto)   │    │ (repo_disp.)│ │
  │  └──────────┘    └──────────┘    └──────┬──────┘ │
  │                                         │ pass   │
  │  ┌──────────┐    ┌──────────┐    ┌──────▼──────┐ │
  │  │ Tag      │◄───│ QA Stress│◄───│ Deploy      │ │
  │  │ rc-v1.x  │    │ Suite    │    │ staging     │ │
  │  │          │    │ (Locust) │    │ + Smoke     │ │
  │  └────┬─────┘    └──────────┘    └─────────────┘ │
  │       │                                           │
  └───────┼───────────────────────────────────────────┘
          │
          ▼
  ┌──────────────┐
  │ Deploy prod  │
  │ (manual)     │
  │ Tag v1.x     │
  │ + Smoke      │
  └──────────────┘
```

## 10. Repository Map

| Repository | Tech Stack | Purpose |
|------------|-----------|---------|
| `court-booking-platform-service` | Spring Boot 4.x, Java 25 | Auth, users, courts, weather, analytics, support |
| `court-booking-transaction-service` | Spring Boot 4.x, Java 25 | Bookings, payments, notifications, waitlist, matches |
| `court-booking-mobile-app` | Flutter (Dart) | iOS, Android, Web customer app |
| `court-booking-admin-web` | React, TypeScript | Court owner + platform admin portal |
| `court-booking-qa` | pytest, Locust, Playwright, Patrol | Functional, stress, contract, UI tests |
| `court-booking-infrastructure` | Terraform, Kubernetes, Helm | DigitalOcean provisioning, K8s manifests, Istio |
| `court-booking-common` | Java 25 | Shared DTOs, Kafka event classes, exceptions, utilities. See [`LIBRARY-SPEC.md`](court-booking-common/LIBRARY-SPEC.md) |

## 11. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Cloud provider | DigitalOcean | Cost-effective for MVP, managed K8s/PG/Redis |
| Service split | 2 services (Platform + Transaction) | Clear domain boundaries, independent scaling |
| Event streaming | Redpanda Serverless | Kafka-compatible, native protocol (lower latency than HTTP-based alternatives), pay-as-you-go, 99.9% SLA, migration path to Strimzi |
| Payments | Stripe Connect Express | Marketplace payouts, hosted onboarding |
| Mobile framework | Flutter | Single codebase for iOS, Android, Web |
| Admin framework | React | Rich ecosystem, TypeScript support |
| Service mesh | Istio (staging + prod only) | mTLS, traffic management, observability |
| Database | Single PG instance, separate schemas | Cost-effective, cross-schema views for reads |
| Real-time | WebSocket + Redis Pub/Sub | Horizontal scaling across pods |
| API contracts | OpenAPI 3.1 + Kafka JSON Schema (contract-first) | Code generation for clients, CI validation. See [`docs/api/`](docs/api/README.md) |

## 12. Cross-Cutting Concerns

### Timezone Handling

Courts operate in local timezones (primarily `Europe/Athens`), but all internal storage and communication uses UTC. This prevents timezone bugs and simplifies cross-service communication.

**Storage Strategy:**
- All `TIMESTAMPTZ` columns store UTC (PostgreSQL default behavior)
- `Instant` (Java) / `DateTime` (Dart) for all timestamp fields
- Court's `timezone` column stores the IANA timezone ID (e.g., `Europe/Athens`)
- Booking `date` and `start_time`/`end_time` are stored as `DATE` and `TIME` in the court's local timezone (not UTC) because they represent "10:00 AM at this court" regardless of DST changes

**Conversion Rules:**

| Context | Storage | Display | Conversion |
|---------|---------|---------|------------|
| Event timestamps | `Instant` (UTC) | User's locale | Client converts using device timezone |
| Booking date/time | `DATE` + `TIME` (court-local) | Court's timezone | No conversion needed for display |
| Kafka event timestamps | ISO 8601 UTC | N/A | Used for latency measurement |
| Audit log timestamps | `Instant` (UTC) | Admin's locale | Client converts |
| Scheduled job triggers | `Instant` (UTC) | N/A | Quartz uses UTC internally |

**Booking Time Calculation Example:**
```java
// Court in Europe/Athens, booking at 10:00 local time
LocalDate bookingDate = LocalDate.of(2026, 3, 15);
LocalTime startTime = LocalTime.of(10, 0);
ZoneId courtZone = ZoneId.of("Europe/Athens");

// Convert to Instant for comparison/scheduling
ZonedDateTime courtLocal = ZonedDateTime.of(bookingDate, startTime, courtZone);
Instant bookingStart = courtLocal.toInstant();  // Store this for reminders, etc.

// For display, always use court's timezone
String displayTime = startTime.format(DateTimeFormatter.ofPattern("HH:mm"));  // "10:00"
```

**DST Handling:**
- Bookings are stored as local date+time, so "10:00 AM every Monday" remains 10:00 AM even across DST transitions
- Reminder jobs calculate the `Instant` at runtime using the court's current timezone rules
- If a booking falls in a DST gap (e.g., 02:30 when clocks skip from 02:00 to 03:00), reject with `INVALID_TIME_DST_GAP` error

**Shared Utilities:** See `DateTimeUtils` in `court-booking-common` library.

### Idempotency

All state-changing operations must be idempotent to handle retries, network failures, and duplicate webhook deliveries.

**Client-Generated Idempotency Keys:**

| Operation | Header | Key Format | TTL |
|-----------|--------|------------|-----|
| Booking creation | `X-Idempotency-Key` | UUID v4 | 24 hours |
| Payment capture | `X-Idempotency-Key` | UUID v4 | 24 hours |
| Rating submission | `X-Idempotency-Key` | UUID v4 | 24 hours |
| Support ticket creation | `X-Idempotency-Key` | UUID v4 | 24 hours |
| Promo code redemption | N/A (uses `bookingId`) | UUID | Permanent |

**Implementation:**

```java
// Idempotency key storage (Redis)
public class IdempotencyService {
    private static final Duration TTL = Duration.ofHours(24);
    
    public Optional<Response> checkIdempotency(String key) {
        String cached = redis.get("idempotency:" + key);
        return cached != null ? Optional.of(deserialize(cached)) : Optional.empty();
    }
    
    public void storeResponse(String key, Response response) {
        redis.setex("idempotency:" + key, TTL.toSeconds(), serialize(response));
    }
}
```

**Kafka Event Idempotency:**

Consumers use `eventId` for deduplication:

```java
public class EventConsumer {
    public void processEvent(EventEnvelope<?> envelope) {
        String dedupeKey = "event:" + envelope.getEventId();
        if (redis.setnx(dedupeKey, "1") == 0) {
            log.info("Duplicate event ignored: {}", envelope.getEventId());
            return;  // Already processed
        }
        redis.expire(dedupeKey, Duration.ofDays(7).toSeconds());
        
        // Process event...
    }
}
```

**Stripe Webhook Idempotency:**

Stripe webhooks include `event.id`. Store processed event IDs to prevent replay:

```java
public void handleStripeWebhook(Event event) {
    if (processedWebhooks.contains(event.getId())) {
        return;  // Already processed
    }
    // Process webhook...
    processedWebhooks.add(event.getId(), TTL_7_DAYS);
}
```

**Database-Level Idempotency:**

- `bookings.idempotency_key` — UNIQUE constraint prevents duplicate bookings
- `promo_code_redemptions.(promo_code_id, booking_id)` — UNIQUE constraint prevents double redemption
- `payments.stripe_payment_intent_id` — UNIQUE constraint prevents duplicate payment records

**Shared Utilities:** See `IdempotencyUtils` in `court-booking-common` library.

### Error Handling

Consistent error response format across both services:

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable message",
  "details": {
    "field": "fieldName",
    "reason": "specific reason"
  },
  "traceId": "W3C trace ID for debugging",
  "timestamp": "ISO 8601 UTC"
}
```

**Error Code Categories:**

| Prefix | Category | Example |
|--------|----------|---------|
| `AUTH_` | Authentication | `AUTH_TOKEN_EXPIRED`, `AUTH_INVALID_PROVIDER` |
| `AUTHZ_` | Authorization | `AUTHZ_INSUFFICIENT_ROLE`, `AUTHZ_NOT_OWNER` |
| `BOOKING_` | Booking operations | `BOOKING_SLOT_UNAVAILABLE`, `BOOKING_MINIMUM_NOTICE` |
| `PAYMENT_` | Payment operations | `PAYMENT_CARD_DECLINED`, `PAYMENT_CAPTURE_FAILED` |
| `COURT_` | Court operations | `COURT_NOT_FOUND`, `COURT_NOT_VISIBLE` |
| `PROMO_` | Promo codes | `PROMO_EXPIRED`, `PROMO_USAGE_LIMIT_REACHED` |
| `VALIDATION_` | Input validation | `VALIDATION_INVALID_FORMAT`, `VALIDATION_REQUIRED_FIELD` |
| `RATE_LIMIT_` | Rate limiting | `RATE_LIMIT_EXCEEDED` |
| `INTERNAL_` | Internal errors | `INTERNAL_SERVICE_ERROR`, `INTERNAL_TIMEOUT` |

**Shared Utilities:** See exception classes in `court-booking-common` library.
