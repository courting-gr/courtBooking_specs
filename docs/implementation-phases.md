# Court Booking Platform — Implementation Phases

A phased approach to design and implementation, where each phase delivers a working, deployable increment. Each phase gets its own spec (design + tasks) created when ready to start.

Dependencies flow downward — later phases build on earlier ones.

---

## Phase 1a — Foundation Infrastructure

**Target repos:** `court-booking-infrastructure`

**Spec:** `.kiro/specs/phase-1-foundation-infrastructure/`

**Requirements covered:** Req 20 (local development environment — Docker Compose, IDE integration), Req 22 (multi-environment deployment — Kubernetes namespaces, Terraform environments)

**Scope:**
- Terraform modules for DigitalOcean (DOKS, Managed PostgreSQL + PostGIS, Managed Redis, Spaces, Container Registry, VPC, DNS, Project)
- Environment compositions (shared + production) with Terraform state management
- Kubernetes manifests, NGINX Ingress Controller, cert-manager, Sealed Secrets, namespace setup (dev/test/staging/production)
- Kustomize base/overlays structure with Helm values for observability stack
- CI/CD pipelines in GitHub Actions for Terraform (plan on PR, deploy on merge with approval gates)
- Docker Compose for local development (PostgreSQL + PostGIS, Redis, Kafka in KRaft mode)
- Operational scripts (backup verification, secret rotation, cluster maintenance)
- Validation scripts (module structure, naming conventions, resource tagging)
- External services setup documentation (Redpanda, Stripe, SendGrid, FCM, OpenWeatherMap, OAuth providers)

**Deliverable:** All infrastructure provisioned and validated. DOKS clusters, managed databases, container registry, and CI/CD pipelines ready for service deployment.

---

## Phase 1b — Service Scaffolding

**Target repos:** `court-booking-platform-service`, `court-booking-transaction-service`, `court-booking-common`

**Spec:** `.kiro/specs/phase-1b-service-scaffolding/`

**Requirements covered:** Req 18 (database management and migrations — Flyway setup, schema creation), Req 20 (local development environment — Spring profiles, hot reload), Req 29 (API contracts — initial OpenAPI specification scaffolding)

**Scope:**
- Shared common library setup (`court-booking-common`) with Maven/GitHub Packages — DTOs, Kafka event classes, exceptions, utilities
- Spring Boot project scaffolding for both services (hexagonal architecture, profiles, actuator health endpoints)
- Database schema migrations (Flyway) — initial tables for both platform and transaction schemas, cross-schema views
- CI/CD pipelines in GitHub Actions for both services (build, test, scan with Trivy, push to DOCR, deploy with approval gates)
- Kubernetes deployment manifests for both services (Deployments, Services, Ingress, ConfigMaps)
- Dockerfiles for both services (multi-stage builds)
- Spring profiles configuration (local, dev, staging, prod)
- Cross-repo QA trigger integration

**Deliverable:** Both services running on DOKS (empty but healthy), CI/CD green, local dev environment working, Flyway migrations applied.

---

## Phase 2 — Auth & User Management (Platform Service)

**Target repos:** `court-booking-platform-service`

**Requirements covered:** Req 1 (full)

**Scope:**
- OAuth registration and login (Google, Facebook, Apple)
- JWT access token issuance (RS256, 15-min lifetime) with role-based claims
- Refresh token rotation (30-day lifetime, replay detection, device tracking)
- Biometric authentication flow (secure enclave refresh token storage)
- Role-based access control: CUSTOMER, COURT_OWNER, SUPPORT_AGENT, PLATFORM_ADMIN
- User profile CRUD (name, email, phone, language)
- Account deletion with GDPR compliance (anonymization, future booking cancellation)
- Concurrent session management (max 5 devices)
- Rate limiting on auth endpoints
- OAuth provider linking/unlinking

**Deliverable:** Users can register, login, manage profiles, and receive proper JWT tokens. Auth is the foundation for everything else.

---

## Phase 3 — Court Management (Platform Service)

**Target repos:** `court-booking-platform-service`

**Requirements covered:** Req 2, Req 3, Req 4, Req 5, Req 6, Req 7 (availability caching), Req 37 (holiday calendar)

**Scope:**
- Court CRUD (create, update, delete with booking conflict checks)
- Image uploads to DigitalOcean Spaces (format validation, EXIF stripping, CDN URLs)
- Court types: Tennis, Padel, Basketball, Football 5x5
- Indoor/Outdoor classification
- Availability windows (recurring weekly schedule) and overrides (maintenance, holidays)
- Holiday calendar management (Req 37):
  - Pre-defined Greek national holidays with Orthodox Easter calculation
  - Custom holidays with optional annual recurrence
  - Bulk holiday application across multiple courts
  - Holiday calendar view with conflict detection
- Geospatial search with PostGIS (radius search, map bounds, filtering by type)
- Aggregated map endpoint (`GET /api/courts/map`)
- Court owner verification workflow (submit documents, admin review, approve/reject)
- Pricing rules and cancellation tiers per court
- Availability caching in Redis (5-min TTL)
- Weather forecast integration (OpenWeatherMap, 7-day window, Redis cache)
- Favorites and user preferences
- Court owner default settings
- Kafka publishing: `court-update-events` topic (COURT_UPDATED, PRICING_UPDATED, AVAILABILITY_UPDATED, etc.)

**Deliverable:** Courts can be created, discovered on a map, and have availability/pricing configured. Weather integration working. Holiday calendar operational.

---

## Phase 4 — Booking & Payments (Transaction Service)


**Target repos:** `court-booking-transaction-service`

**Requirements covered:** Req 8, Req 9, Req 10, Req 11, Req 11a (Stripe Connect onboarding), Req 12, Req 14

**Scope:**
- Customer booking flow: slot hold (5-min Redis lock) → payment → confirm
- Stripe Connect onboarding for court owners (account creation, identity verification, bank details)
- Payment processing: PaymentIntent creation, authorize, capture, refund
- Platform fee calculation and Stripe Connect transfers
- Manual bookings for court owners (no payment required)
- Pending confirmation workflow (MANUAL mode): court owner confirm/reject, auto-cancel timeout
- Recurring bookings (weekly pattern, advance scheduling via Quartz)
- Booking modifications and cancellations with tiered refund policy
- No-show flagging by court owners
- External payment tracking (cash, bank transfer)
- Kafka publishing: `booking-events` topic (BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, etc.)
- Kafka consuming: `court-update-events` (pricing cache, constraint validation)
- Stripe webhook handling (payment_intent.succeeded, charge.dispute.created, etc.)
- Cross-schema views for court/user lookups
- Idempotency key support for all state-changing operations

**Deliverable:** End-to-end booking and payment flow working. Court owners receive payouts via Stripe Connect.

---

## Phase 5 — Real-time & Notifications

**Target repos:** `court-booking-transaction-service`, `court-booking-platform-service`

**Requirements covered:** Req 13, Req 19 (WebSocket/real-time parts)

**Scope:**
- WebSocket setup in Transaction Service (STOMP over WebSocket)
- Redis Pub/Sub for horizontal WebSocket scaling across pods
- Real-time availability updates (booking → cache invalidation → WebSocket broadcast)
- Real-time booking status updates (confirmation, cancellation)
- Push notifications via Firebase Cloud Messaging (Android + iOS)
- Email notifications via SendGrid (booking confirmations, receipts, reminders)
- In-app notifications via WebSocket
- Web Push for admin portal
- Notification preferences (per-event-type channel selection, do-not-disturb windows)
- Notification urgency levels (CRITICAL bypasses DND, PROMOTIONAL respects opt-out)
- Kafka consuming: `notification-events` topic
- Device token management (FCM registration, Web Push subscriptions)
- Booking reminders via Quartz scheduled jobs
- Pending confirmation reminders for court owners

**Deliverable:** Users receive real-time updates and notifications across all channels. Availability updates propagate within 2 seconds.

---

## Phase 6 — Admin, Analytics & Support

**Target repos:** `court-booking-platform-service`, `court-booking-admin-web`

**Requirements covered:** Req 15, Req 15a, Req 15b, Req 15c, Req 15d (admin UI security), Req 15e (global search, bulk operations), Req 16 (admin dashboard metrics), Req 17 (centralized logging — admin dashboard integration), Req 18 (database management — analytics migrations), Req 19 (admin/analytics performance), Req 28 (multi-language support), Req 29 (API contracts — OpenAPI maintenance), Req 30 (customer support system)

**Scope:**
- Court owner dashboard: today's bookings, revenue summary, occupancy rates, pending actions
- Analytics API: revenue reports, booking trends, occupancy heatmaps (using read replica)
- CSV/PDF export (rate-limited)
- Court owner audit logs (immutable trail of all actions)
- Smart reminder rules (configurable alerts for low occupancy, pending confirmations, etc.)
- Court owner notification preferences (per-event-type channel config)
- Admin web portal (React): user management, verification review queue, feature flags, platform analytics
- Support ticket system: creation, assignment, responses, attachments, metrics
- Global search across bookings, courts, promo codes
- Bulk operations (bulk court visibility toggle, bulk booking confirm/reject)
- Platform admin: dispute escalation, user suspend/unsuspend
- Feature flag management
- Kafka consuming: `analytics-events` topic
- Multi-language support (Req 28): translations API, content management for Greek/English, Accept-Language header routing across all endpoints, translation workflow for court descriptions, notification templates, and error messages
- OpenAPI specification maintenance and validation (Req 29)

**Deliverable:** Court owners have a full admin dashboard. Platform admins can manage the system. Support workflow operational. Full i18n support for Greek and English.

---

## Phase 7 — Security Hardening

**Target repos:** `court-booking-platform-service`, `court-booking-transaction-service`, `court-booking-admin-web`, `court-booking-infrastructure`

**Requirements covered:** Req 32, Req 33, Req 34, Req 35, Req 36

**Scope:**
- Security monitoring and abuse detection (brute force, booking abuse, payment fraud, scraping)
- Security alert system: `security-events` Kafka topic, `security_alerts` table, admin dashboard
- Automated responses (account restriction, IP auto-blocking, rate limit escalation)
- IP blocklist management (Redis + admin API)
- Failed auth attempt tracking and account lockout
- Input validation and sanitization (OWASP Java HTML Sanitizer, parameterized queries)
- Security headers (HSTS, X-Content-Type-Options, X-Frame-Options, CSP, Referrer-Policy)
- CORS configuration (allowlisted origins only)
- CSRF protection for admin web (Synchronizer Token Pattern)
- Webhook security (signature verification, timestamp tolerance, IP allowlisting)
- WebSocket authentication (JWT on upgrade) and channel authorization
- Data encryption at rest (managed services) and in transit (TLS 1.2+, mTLS via Istio)
- Data classification enforcement (log sanitization, PII exclusion from Kafka events)
- Retention policies and anonymization scheduled jobs
- Secret rotation procedures (JWT keys every 90 days, DB credentials every 90 days)
- Stripe Connect security (account status monitoring, payout fraud detection, self-booking detection)

**Deliverable:** Platform hardened against OWASP Top 10, abuse patterns detected and auto-mitigated, data lifecycle managed.

---

## Phase 8 — Mobile App (Flutter)

**Target repos:** `court-booking-mobile-app`

**Requirements covered:** Req 31 (mobile app client resilience, offline behavior, error handling), plus mobile UX requirements from the Customer User Journey and Court Owner Admin Journey sections of the master requirements document

**Scope:**
- Auth screens: OAuth login, biometric setup, terms acceptance
- Map view: court markers, open match markers, type tabs, indoor/outdoor filter, clustering
- Court discovery: search, filter, favorites, weather integration
- Court detail screen: images, ratings, amenities, availability slots, weather
- Booking flow: slot selection → confirmation → payment (Stripe SDK) → receipt
- My bookings: upcoming, past, cancelled, pending confirmation
- Notifications: push notification handling, in-app notification center
- Court owner screens: court management, booking calendar, manual bookings, analytics
- Offline handling: cached data, graceful degradation, retry mechanisms
- Deep linking from notifications
- Background app behavior (token refresh, data sync on foreground)
- Client-side validation (with server-side as authoritative)
- Partial failure handling (per-section loading, error states with retry)

**Note:** Can start in parallel with Phase 5-6 once APIs from Phases 2-4 are stable. Use mock/staging APIs initially.

**Deliverable:** Fully functional mobile app for customers and court owners on iOS and Android.

---

## Phase 9 — Observability & Production Readiness

**Target repos:** `court-booking-infrastructure`, `court-booking-qa`

**Requirements covered:** Req 16 (system observability, monitoring, traceability), Req 17 (centralized logging and alerting — infrastructure), Req 20 (local development environment — validation and tuning), Req 21 (QA testing framework and automation), Req 22 (multi-environment deployment with service mesh)

**Scope:**
- Prometheus metrics collection (custom business metrics + JVM/Spring metrics)
- Grafana dashboards (service health, booking throughput, payment success rates, error rates)
- Grafana alerting (PagerDuty/Slack integration for critical alerts)
- Loki log aggregation (structured JSON logging, log sanitization verification)
- Jaeger distributed tracing (trace ID propagation across services and Kafka)
- Kiali service mesh visualization (with Istio)
- Istio service mesh setup for staging/production (mTLS, traffic management, canary deployments)
- Health check tuning (liveness, readiness, startup probes)
- Graceful shutdown (WebSocket drain, in-flight request completion)
- Payment reconciliation job: Quartz job (every 15 minutes) in Transaction Service to detect orphaned payments (payment captured but booking not created due to server error) and auto-create bookings or initiate refunds. Publishes push notification when resolved. (From Customer Journey Step 8a)
- QA test suite: pytest functional tests, Locust stress tests, contract tests
- Load testing and performance benchmarking
- Backup restoration testing
- Runbook documentation for common operational scenarios

**Deliverable:** Full observability stack, production-grade resilience, load-tested and validated.

---

## Phase 10 (⏳) — Phase 2 Features

**Target repos:** All application repos

**Requirements covered:** Req 23 (advertisement system), Req 24, Req 25, Req 26, Req 27, Req 9a (subscription billing), court ratings (Req 2.19–2.20)

**Scope:**
- Advertisement system: feature-flagged ad placements, ad provider integration (Google AdMob, Facebook Audience Network), ad-free premium option
- Open matches: creation, join requests (auto-accept/manual), player coordination, map markers
- Waitlists: FIFO queue, auto-notify on cancellation, slot hold with timeout
- Split payments: invitation, per-player payment tracking, deadline enforcement
- Promo codes: court-level and platform-wide, validation, redemption tracking
- Dynamic pricing: peak/off-peak multipliers, special date pricing
- Court ratings and reviews (post-booking, one per booking)
- Court owner subscription billing (Stripe Billing, trial management, tier upgrades)
- Email digest option: daily summary email instead of individual notification emails (from Court Owner Journey Step 9)
- FAQ / Help Center: searchable FAQ content for in-app and admin portal support hub (from Customer Journey Step 9 and Court Owner Journey Step 9)
- Scheduled reports: configure automatic weekly/monthly email reports (from Court Owner Journey Step 8)
- Court ownership transfer (Req 2.29–2.32)
- Waitlist/match abuse detection
- Promo code abuse detection

**Deliverable:** Full feature set as designed in the requirements document.

---

## Parallel Work Streams

Some phases can overlap:

```
Phase 1a ─────►
Phase 1b ──────────►
Phase 2   ────────────►
Phase 3     ────────────────►
Phase 4          ────────────────►
Phase 5               ──────────────►
Phase 6                    ──────────────►
Phase 7                         ──────────────►
Phase 8          ════════════════════════════════► (parallel, starts after Phase 2-4 APIs stable)
Phase 9                              ──────────────►
Phase 10                                      ──────────────►
```

## Spec Creation Strategy

For each phase:
1. Create a dedicated spec directory: `.kiro/specs/{phase-name}/`
2. Generate `requirements.md` (scoped to that phase's requirements from the master requirements doc)
3. Generate `design.md` (detailed technical design for that phase)
4. Generate `tasks.md` (implementation task list with property-based tests)
5. Execute tasks
6. Review and validate before moving to next phase
