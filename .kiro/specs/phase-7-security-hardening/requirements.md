# Requirements Document — Phase 7: Security Hardening

## Introduction

Phase 7 hardens the Court Booking Platform against OWASP Top 10 vulnerabilities, implements abuse detection and automated mitigation, enforces data lifecycle policies, and secures real-time communication channels. This phase **extends** existing security infrastructure delivered in earlier phases — `RateLimitFilter`, `JwtAuthenticationFilter`, `SuspendedUserFilter`, and the `JwksController` from Phase 2 (Auth & User Management); `CsrfValidationFilter` and the `CsrfTokenController` from Phase 6a (Admin, Analytics & Support); the `failed_auth_attempts`, `security_alerts`, and `ip_blocklist` tables created by the Phase 1b Flyway migration `V1__create_platform_schema.sql`; the `security-events` Kafka topic and `SECURITY_ALERT` event schema defined in `docs/api/kafka-event-contracts.json`; the WebSocket connection authentication and STOMP destinations defined in Phase 5 (Real-time & Notifications) — and closes the remaining gaps in the Transaction Service, NGINX Ingress, and admin web application.

**Phase boundaries (what is NEW in Phase 7 vs. what is EXTENDED from earlier phases):**

| Concern | Earlier-phase deliverable | Phase 7 addition |
|---------|---------------------------|------------------|
| JWT validation in Transaction Service | Phase 2 ships `JwtAuthenticationFilter` in `court-booking-common`; Transaction Service uses no-op security in Phase 4 | Wire the shared filter into Transaction Service, fetch keys from JWKS, enforce role-based authorization on every endpoint (Req 1) |
| WebSocket authentication | Phase 5 ships JWT validation on `/ws?token=...` upgrade and user/role association | Token-refresh flow, channel authorization with self-ownership checks, message validation/rate limiting (Req 12–14) |
| CSRF protection | Phase 6a ships `CsrfValidationFilter`, `CsrfTokenController`, in-memory token in admin web | Codify the existing implementation as a Phase 7 contract; no net-new behavior unless gaps are found (Req 10) |
| IP-based auth lockout | Phase 2 ships IP-keyed lockout (20 failed attempts/15-min → 30-min lock) backed by `failed_auth_attempts` | Add account-keyed lockout (5 failed/15-min per email), distributed-attack detection (cross-account same-IP), suspicious-login challenge, unlock email link (Req 5) |
| Rate limiting | Phase 2 ships `RateLimitFilter` for auth endpoints; an `ApiRateLimitFilter` covers application endpoints | Progressive escalation of lockout durations on repeated violations (Req 6) |
| `security_alerts`, `ip_blocklist`, `failed_auth_attempts` tables | Phase 1b migration creates the schema | Phase 7 implements the producer/consumer logic, REST APIs, dashboards, and auto-response actions on top of these tables (Req 2, Req 7, Req 23, Req 24) |

**Target repositories:** `court-booking-platform-service`, `court-booking-transaction-service`, `court-booking-admin-web`, `court-booking-infrastructure`

**Master requirements covered:** Req 32, Req 33, Req 34, Req 35, Req 36

## Glossary

- **Platform_Service**: The `court-booking-platform-service` Spring Boot application handling user management, court management, authentication, and admin operations
- **Transaction_Service**: The `court-booking-transaction-service` Spring Boot application handling bookings, payments, Stripe Connect, and real-time notifications
- **Admin_Web**: The `court-booking-admin-web` React application used by PLATFORM_ADMIN users
- **NGINX_Ingress**: The NGINX Ingress Controller deployed in Kubernetes handling TLS termination and request routing
- **Security_Alert**: A record in the `platform.security_alerts` table representing a detected security event
- **IP_Blocklist**: A Redis-backed set of blocked IP addresses/CIDR ranges with optional TTL, also enforced at NGINX level
- **Kafka_Security_Topic**: The `security-events` Kafka topic used for publishing SECURITY_ALERT events between services
- **Progressive_Rate_Limiter**: An extension of the existing rate limiting system that escalates lockout durations on repeated violations
- **Data_Classification**: A four-tier classification system (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED) governing access and retention
- **Anonymization_Job**: A scheduled job that replaces PII with placeholder values while retaining aggregate data
- **WebSocket_Channel**: A logical subscription path within the STOMP-over-WebSocket connection for receiving targeted real-time updates
- **Self_Booking_Fraud**: A pattern where a court owner creates a customer account to book their own courts and extract platform payouts

## Requirements

### Requirement 1: Transaction Service JWT Authentication and Role-Based Enforcement

**User Story:** As a platform operator, I want the Transaction Service to enforce JWT authentication and role-based access control on all API endpoints, so that only authorized users can perform booking and payment operations.

#### Acceptance Criteria

1. WHEN a request arrives at a non-public Transaction_Service endpoint, THE Transaction_Service SHALL validate the JWT access token from the `Authorization: Bearer` header by fetching the signing key from the Platform_Service JWKS endpoint at `/api/auth/.well-known/jwks.json` and verifying the RS256 signature, expiration (`exp`), and issuer claims
2. IF the JWT token is missing, expired, or has an invalid signature, THEN THE Transaction_Service SHALL reject the request with `401 Unauthorized` and a JSON response body containing an error message indicating the authentication failure reason
3. IF the JWT token is valid but the authenticated user's `role` claim does not match the required role for the requested endpoint, THEN THE Transaction_Service SHALL reject the request with `403 Forbidden`
4. THE Transaction_Service SHALL enforce role-based authorization: CUSTOMER role for booking creation (`POST /api/bookings`), modification (`PUT /api/bookings/{id}/modify`), cancellation (`POST /api/bookings/{id}/cancel`), and listing own bookings (`GET /api/bookings` filtered to bookings where the `sub` claim matches the booking's customer ID)
5. THE Transaction_Service SHALL enforce role-based authorization: COURT_OWNER role for manual booking creation (`POST /api/bookings/manual`), booking confirmation (`POST /api/bookings/{id}/confirm`), rejection (`POST /api/bookings/{id}/reject`), no-show flagging (`POST /api/bookings/{id}/no-show`), mark-paid (`POST /api/bookings/{id}/mark-paid`), bulk actions (`POST /api/bookings/bulk-action`), iCal export (`GET /api/bookings/calendar/ical`), and listing pending bookings (`GET /api/bookings/pending`)
6. THE Transaction_Service SHALL enforce role-based authorization: PLATFORM_ADMIN role for cross-user booking queries and any future administrative endpoints under `/api/admin/**`
7. THE Transaction_Service SHALL permit unauthenticated access only to: `/actuator/health/**`, `/api/webhooks/stripe`, `/ws/**`, `/v3/api-docs/**`, `/swagger-ui/**`
8. THE Transaction_Service SHALL cache the JWKS response in memory with a 24-hour TTL and SHALL refresh the cache automatically when a JWT presents an unknown `kid` (key ID) value, retaining each retired key for at least 15 minutes after it is removed from the upstream JWKS to bridge the maximum access-token lifetime. IF the Platform_Service JWKS endpoint is unreachable AND no usable cached key exists for the presented `kid`, THEN THE Transaction_Service SHALL reject the request with `503 Service Unavailable` and emit a `JWKS_UNAVAILABLE` metric. IF the cached key is still usable, THEN THE Transaction_Service SHALL serve requests from the cache and continue retrying the JWKS endpoint in the background with exponential backoff (1s, 2s, 4s, max 60s)
9. WHEN a CUSTOMER role user requests a booking resource (`GET /api/bookings/{id}`, `GET /api/bookings/{id}/receipt`), THE Transaction_Service SHALL verify that the `sub` claim in the JWT matches the booking's customer ID and reject with `403 Forbidden` if it does not match

### Requirement 2: Security Event Publishing and Consumption

**User Story:** As a platform administrator, I want all security events published to a central Kafka topic and stored for dashboard display, so that I can monitor threats and respond to incidents in real time.

#### Acceptance Criteria

1. WHEN any abuse detection rule is triggered in the Platform_Service or Transaction_Service, THE triggering service SHALL publish a `SECURITY_ALERT` event to the `security-events` Kafka topic within 1 second of rule evaluation completing
2. THE `SECURITY_ALERT` event SHALL include: alertType (enum: BOOKING_ABUSE, PAYMENT_FRAUD, SCRAPING, BRUTE_FORCE, SUSPICIOUS_LOGIN, RATE_LIMIT_EXCEEDED, WEBHOOK_REPLAY, ACCOUNT_TAKEOVER), severity (enum: LOW, MEDIUM, HIGH, CRITICAL), userId (nullable UUID), ipAddress (nullable String), description (String, max 1000 characters), metadata (JSON object with rule-specific details, max 10 KB), and timestamp (ISO-8601)
3. THE Platform_Service SHALL consume events from the `security-events` Kafka topic using consumer group `platform-service-security-consumer` with `auto.offset.reset=earliest`, and persist them to the existing `platform.security_alerts` table with status `OPEN` (matching the database CHECK constraint values: OPEN, ACKNOWLEDGED, INVESTIGATING, RESOLVED, FALSE_POSITIVE)
4. WHEN a CRITICAL severity alert is persisted, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic targeting all users with the PLATFORM_ADMIN role, with `urgency = 'CRITICAL'` and `channels` including both `EMAIL` and `PUSH`, within 5 seconds of event consumption
5. THE Platform_Service SHALL provide a paginated REST API for PLATFORM_ADMIN to query security alerts with filtering by severity, alertType, date range (max 90 days), and status (OPEN, ACKNOWLEDGED, INVESTIGATING, RESOLVED, FALSE_POSITIVE). Pagination defaults: page=0, size=20 (max 100), sort=created_at DESC
6. THE Platform_Service SHALL provide a REST API for PLATFORM_ADMIN to update alert status with the following valid transitions: OPEN → ACKNOWLEDGED (no additional fields required), OPEN → INVESTIGATING (no additional fields required), ACKNOWLEDGED → INVESTIGATING (no additional fields required), ACKNOWLEDGED → RESOLVED (requires resolution_notes, max 2000 characters), INVESTIGATING → RESOLVED (requires resolution_notes), any non-terminal status → FALSE_POSITIVE (requires resolution_notes). Transitions out of RESOLVED or FALSE_POSITIVE SHALL be rejected with `409 Conflict`. THE Platform_Service SHALL record the authenticated admin's user ID in `acknowledged_by` (on first transition out of OPEN) or `resolved_by` (on transition to RESOLVED or FALSE_POSITIVE) and set the corresponding timestamp (`acknowledged_at` or `resolved_at`)
7. IF a malformed or unrecognized event is consumed from the `security-events` Kafka topic, THEN THE Platform_Service SHALL log the error at WARN level with the raw event payload and skip the event without blocking the consumer
8. IF the `security-events` Kafka topic is unavailable when an abuse detection rule is triggered, THEN THE triggering service SHALL retry publishing with exponential backoff (1s, 2s, 4s) for up to 3 attempts, and log the failure at ERROR level if all attempts fail

### Requirement 3: Booking Abuse Detection

**User Story:** As a platform operator, I want automated detection of booking abuse patterns, so that malicious users cannot grief court owners or manipulate availability.

#### Acceptance Criteria

1. THE Transaction_Service SHALL enforce booking velocity limits by counting only bookings with status CONFIRMED or PENDING_CONFIRMATION: maximum 5 bookings per user per rolling 60-minute window and maximum 10 bookings per user per rolling 24-hour window, rejecting excess attempts with `429 Too Many Requests`
2. WHEN a user creates and cancels more than 3 bookings for the same court within a rolling 24-hour window, THE Transaction_Service SHALL publish a MEDIUM severity SECURITY_ALERT with alertType BOOKING_ABUSE and block further bookings for that court by that user for 48 hours
3. IF a user who is currently blocked for a specific court attempts to create a booking for that court, THEN THE Transaction_Service SHALL reject the request with `403 Forbidden` and include an error message indicating the block reason and remaining block duration
4. WHEN a user cancels more than 5 peak-time bookings (as defined by the court's pricing rules) within a rolling 7-day window, THE Transaction_Service SHALL publish a HIGH severity SECURITY_ALERT with alertType BOOKING_ABUSE and restrict the user to instant-pay-only bookings for 14 days
5. WHEN the same user attempts to book the same court, date, and start time within a 5-second window of a prior attempt, THE Transaction_Service SHALL reject the duplicate with `409 Conflict`

### Requirement 4: Payment Fraud Detection

**User Story:** As a platform operator, I want automated detection of payment fraud patterns, so that the platform is protected from chargebacks and financial abuse.

#### Acceptance Criteria

1. THE Transaction_Service SHALL evaluate chargeback rates per court owner every 15 minutes using the formula: (disputes in rolling 90 days) / (total successful payments in rolling 90 days). WHEN the rate exceeds 0.75% of total transactions, THE Transaction_Service SHALL publish a HIGH severity SECURITY_ALERT with type PAYMENT_FRAUD to the `security-events` Kafka topic
2. WHEN a court owner chargeback rate exceeds 1.0% of total transactions (rolling 90-day window), THE Transaction_Service SHALL publish a CRITICAL severity SECURITY_ALERT with type PAYMENT_FRAUD and set the court owner account status to UNDER_REVIEW, preventing new bookings for that owner courts until a PLATFORM_ADMIN resolves the alert
3. WHEN a user accumulates 3 or more **failed** payment attempts using different Stripe PaymentMethod card fingerprints within a 1-hour sliding window, THE Transaction_Service SHALL publish a HIGH severity SECURITY_ALERT with alertType PAYMENT_FRAUD including the distinct card count, the user ID, and the Stripe decline codes encountered in the alert metadata, and SHALL block further payment attempts from that user for 1 hour with HTTP `429 Too Many Requests`. This rule targets card-testing attacks (Master Req 32.11)
4. WHEN a user completes 3 or more **successful** payments with different Stripe PaymentMethod card fingerprints within a 1-hour sliding window, THE Transaction_Service SHALL publish a MEDIUM severity SECURITY_ALERT with alertType PAYMENT_FRAUD including the distinct card count and user ID in the alert metadata
5. WHEN a booking payment is completed and the Stripe PaymentMethod card.country differs from the country derived from the user billing address, THE Transaction_Service SHALL record the country mismatch (`fraud_card_country`, `fraud_user_country`) in the booking's metadata and flag the booking for manual review by setting `bookings.fraud_review_required = true`
6. IF the card country cannot be determined from the Stripe PaymentMethod (e.g., digital wallet payments where card.country is null), THEN THE Transaction_Service SHALL skip the country mismatch check for that booking and log the event at INFO level

### Requirement 5: Credential Stuffing and Brute Force Detection

**User Story:** As a platform operator, I want detection of credential stuffing attacks beyond simple per-IP lockout, so that distributed attacks are identified and blocked.

This requirement extends the existing failed-auth tracking introduced in Phase 2 (Req 10) by adding dual-key tracking (IP + email), distributed-attack detection, suspicious-login challenges, and an account-level unlock email. THE existing `failed_auth_attempts` table schema (Phase 1b migration; columns: ip_address, email, attempt_count, window_start, locked_until) and Phase 2's IP-based lockout logic remain in place; Phase 7 layers account-keyed lockouts and cross-IP correlation on top.

#### Acceptance Criteria

1. WHEN 20 or more failed authentication attempts occur across 3 or more distinct user accounts from the same IP address within a 5-minute sliding window, THE Platform_Service SHALL publish a CRITICAL severity SECURITY_ALERT with alertType BRUTE_FORCE and auto-block the IP for 24 hours via the IP blocklist
2. WHEN a successful authentication occurs from a new device (previously unseen User-Agent string for that account) AND a new IP address AND a country different from all countries seen in the user's login history within the last 90 days, THE Platform_Service SHALL withhold access, send an email confirmation link to the user's registered email, and publish a LOW severity SECURITY_ALERT with alertType SUSPICIOUS_LOGIN
3. IF the email confirmation link from a suspicious login challenge is not confirmed within 15 minutes, THEN THE Platform_Service SHALL deny the login session and require the user to re-authenticate
4. IF the GeoIP lookup service is unavailable during login, THEN THE Platform_Service SHALL treat the country as unknown, skip the country-based suspicious login check, and proceed with device and IP checks only
5. THE Platform_Service SHALL track failed authentication attempts using **dual-key tracking** in Redis with separate sliding-window counters keyed by (a) IP address and (b) user email, persisted to `failed_auth_attempts` for audit. THE Phase 2 IP-based lockout (20 failed attempts per IP within 15 minutes → 30-minute lock) remains in effect; in addition, WHEN 5 failed attempts occur for the same user email within a 15-minute window (regardless of source IP), THE Platform_Service SHALL set an account-level lock for 30 minutes by setting `failed_auth_attempts.locked_until` keyed by email and rejecting further auth attempts for that account with HTTP `423 Locked`. THE Platform_Service SHALL send an email notification to the account owner containing a one-time unlock link valid for 30 minutes that allows the user to unlock the account and reset both the IP-keyed and email-keyed failed attempt counters

### Requirement 6: Scraping Detection and Progressive Rate Limiting

**User Story:** As a platform operator, I want detection of automated scraping and progressive escalation of rate limits, so that API abuse is deterred without impacting legitimate users.

#### Acceptance Criteria

1. THE Platform_Service SHALL detect scraping patterns based on any of the following indicators from a single IP address: (a) a request with a missing or empty User-Agent header, (b) a sustained request rate exceeding 10 requests per second measured over a rolling 5-second window, or (c) 10 or more requests within a 30-second window to the same path template (e.g., `/api/courts/{id}`) where the response status indicates resource enumeration (a mix of `200 OK` and `404 Not Found` responses, with at least 3 `404 Not Found` responses among them). For UUID-keyed resources, "enumeration" is defined as repeated requests to the same path template with distinct path parameters and a non-trivial 404 rate, not strict integer-sequential incrementing
2. WHEN a scraping pattern is detected, THE Platform_Service SHALL publish a MEDIUM severity SECURITY_ALERT with type SCRAPING including the triggering indicator (missing User-Agent, high frequency, or sequential enumeration), the source IP address, and the request count that triggered detection
3. THE Platform_Service SHALL implement progressive rate limiting that extends the existing ApiRateLimitFilter: after the first rate limit violation, subsequent violations within the same 1-hour sliding window result in escalating lockout durations of 1 minute for the 2nd violation, 5 minutes for the 3rd, 15 minutes for the 4th, and 1 hour for the 5th and all subsequent violations within that window
4. WHILE a progressive lockout is active for a user ID or IP address, THE Platform_Service SHALL reject all API requests from that identifier with HTTP 429 and a Retry-After header indicating the remaining lockout duration in seconds
5. THE Platform_Service SHALL track progressive rate limit state in Redis keyed by user ID for authenticated requests or IP address for unauthenticated requests, with a 1-hour sliding window that resets the escalation level to zero when no violations occur within the window

### Requirement 7: IP Blocklist Management

**User Story:** As a platform administrator, I want to manage an IP blocklist that is enforced at both the application and NGINX levels, so that known malicious IPs are rejected as early as possible.

#### Acceptance Criteria

1. THE Platform_Service SHALL maintain an IP blocklist in Redis supporting individual IPv4 addresses and CIDR ranges with prefix lengths between /8 and /32, each with an optional TTL (expiry timestamp), up to a maximum of 100,000 entries
2. THE Platform_Service SHALL provide a REST API for PLATFORM_ADMIN to add, remove, and list blocked IPs with a reason (maximum 500 characters) and optional expiry, returning paginated results of up to 100 entries per page on the list endpoint
3. IF a PLATFORM_ADMIN submits an add request with an invalid IP address format or an out-of-range CIDR prefix, THEN THE Platform_Service SHALL reject the request with a 400 response and an error message indicating the validation failure
4. THE Platform_Service SHALL check incoming requests against the IP blocklist in the existing RateLimitFilter, rejecting blocked IPs with `403 Forbidden` before any further processing
5. IF Redis is unavailable during the IP blocklist lookup in the RateLimitFilter, THEN THE Platform_Service SHALL serve the lookup from a process-local LRU cache (last 1000 blocklist entries, 60-second TTL refreshed on every successful Redis read) and increment a `blocklist_redis_unavailable` metric. IF both Redis and the local cache are unavailable for more than 30 seconds, THEN THE Platform_Service SHALL fail closed by rejecting all requests with `503 Service Unavailable` and emitting a CRITICAL log entry. The same fail-closed behavior applies if the local LRU cache has never been populated (cold start with Redis already down)
6. THE NGINX_Ingress SHALL sync the IP blocklist from Redis every 60 seconds and reject blocked IPs at the ingress level with `403 Forbidden` for early rejection
7. WHEN an IP is auto-blocked by a security rule, THE Platform_Service SHALL record the block in the `platform.ip_blocklist` table with a reference to the triggering security alert

### Requirement 8: Security Headers Configuration

**User Story:** As a platform operator, I want all HTTP responses to include security headers, so that the platform is protected against clickjacking, MIME sniffing, and protocol downgrade attacks.

#### Acceptance Criteria

1. THE NGINX_Ingress SHALL set `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` on all HTTPS responses, and SHALL NOT set this header on plain HTTP responses
2. THE NGINX_Ingress SHALL set `X-Content-Type-Options: nosniff` on all responses exactly once (no duplicate header values)
3. THE NGINX_Ingress SHALL set `X-Frame-Options: DENY` on all responses exactly once (no duplicate header values)
4. THE NGINX_Ingress SHALL set `Referrer-Policy: strict-origin-when-cross-origin` on all responses
5. THE NGINX_Ingress SHALL set `Permissions-Policy: camera=(), microphone=(), geolocation=(self)` on all responses
6. THE Admin_Web SHALL set a Content-Security-Policy header parameterized by environment, with the API and WebSocket origins injected at build time via `VITE_API_ORIGIN` and `VITE_WS_ORIGIN`. The production policy SHALL resolve to: `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' https://*.digitaloceanspaces.com; connect-src 'self' https://api.courtbooking.gr wss://api.courtbooking.gr https://api.stripe.com; frame-src https://js.stripe.com; font-src 'self'`. Staging SHALL substitute `https://staging-api.courtbooking.gr` and `wss://staging-api.courtbooking.gr`. Local development SHALL additionally permit `http://localhost:8080` and `ws://localhost:8080`. THE policy SHALL NOT contain wildcards in `connect-src` or `script-src` in any environment
7. WHEN a client sends an HTTP request to any host served by NGINX_Ingress, THE NGINX_Ingress SHALL respond with a permanent redirect (301) to the equivalent HTTPS URL

### Requirement 9: CORS Configuration

**User Story:** As a platform operator, I want CORS restricted to known origins only, so that unauthorized domains cannot make cross-origin requests to the API.

#### Acceptance Criteria

1. THE NGINX_Ingress SHALL configure CORS to allow requests only from allowlisted origins: `https://admin.courtbooking.gr`, the mobile app web build domain, and `http://localhost:3000` for the `local` environment only
2. THE NGINX_Ingress SHALL restrict allowed CORS methods to `GET, POST, PUT, DELETE, OPTIONS`
3. THE NGINX_Ingress SHALL restrict allowed CORS headers to `Authorization, Content-Type, Accept, Accept-Language, X-Idempotency-Key, X-Request-ID, X-CSRF-Token`
4. THE NGINX_Ingress SHALL NOT use wildcard (`*`) for `Access-Control-Allow-Origin` in staging or production environments
5. IF a request's `Origin` header does not match any allowlisted origin, THEN THE NGINX_Ingress SHALL omit the `Access-Control-Allow-Origin` response header, causing the browser to block the response
6. WHEN a preflight `OPTIONS` request is received from an allowlisted origin, THE NGINX_Ingress SHALL respond with HTTP `204 No Content` and include `Access-Control-Max-Age: 86400` to cache preflight results for 24 hours
7. THE NGINX_Ingress SHALL include `Access-Control-Allow-Credentials: true` in CORS responses to allowlisted origins to permit cookie-based session tokens from the admin web portal

### Requirement 10: CSRF Protection for Admin Web

**User Story:** As a platform operator, I want CSRF protection on the admin web application, so that state-changing operations cannot be triggered by malicious cross-site requests.

This requirement codifies the CSRF protection implementation already delivered by Phase 6a (`CsrfValidationFilter` and `CsrfTokenController` in Platform Service; `csrfToken` in the Admin Web Zustand store with the X-CSRF-Token request header on state-changing requests). Phase 7 adds no net-new behavior; it formalizes the contract and adds explicit error-code expectations and 30-minute session-expiry handling so the implementation can be validated against the security test suite.

#### Acceptance Criteria

1. THE Admin_Web SHALL include the CSRF token in the `X-CSRF-Token` request header on all state-changing requests (POST, PUT, DELETE, PATCH) sent to the Platform_Service (existing Phase 6a behavior)
2. THE Admin_Web SHALL obtain the CSRF token by calling `GET /api/auth/csrf-token` after successful authentication and store it in memory in the auth store (not in localStorage or a cookie accessible to JavaScript). Existing Phase 6a behavior
3. THE Admin_Web SHALL set secure cookie attributes on all session-related cookies: `HttpOnly`, `Secure`, `SameSite=Strict`. Existing Phase 6a behavior; verified against the security test suite
4. THE Platform_Service SHALL validate CSRF tokens on state-changing requests (POST, PUT, DELETE, PATCH) originating from browser clients (detected by presence of `Origin` header) using the existing `CsrfValidationFilter`, and return HTTP `403` with error code `CSRF_VALIDATION_FAILED` when the token is missing or does not match the session token. THE filter SHALL exempt the prefixes already configured in Phase 6a (`/internal/`, `/api/auth/oauth/`, `/api/auth/refresh`, `/api/auth/csrf-token`, `/actuator/`)
5. IF the CSRF token has expired (HTTP session inactive for more than 30 minutes; aligned with the Phase 6a session timeout), THEN THE Platform_Service SHALL reject the request with HTTP `403` and error code `CSRF_VALIDATION_FAILED`, and THE Admin_Web SHALL detect this error code, clear its in-memory token, and redirect the user to the login page

### Requirement 11: Webhook Security Hardening

**User Story:** As a platform operator, I want webhook endpoints protected against replay attacks and unauthorized access, so that only legitimate Stripe events are processed.

#### Acceptance Criteria

1. THE Transaction_Service SHALL verify Stripe webhook signatures using the `Stripe-Signature` header and the endpoint-specific signing secret, and reject requests with invalid or missing signatures with HTTP `400 Bad Request`
2. WHEN the `Stripe-Signature` timestamp is older than 5 minutes relative to the server's current time, THE Transaction_Service SHALL reject the event with HTTP `400 Bad Request` to prevent replay attacks
3. THE NGINX_Ingress SHALL restrict the webhook endpoint (`/api/webhooks/stripe`) to Stripe's published IP ranges (sourced from <https://stripe.com/docs/ips>) using an IP allowlist annotation, refreshed by an operational runbook on a quarterly schedule and on Stripe security advisories, returning HTTP `403 Forbidden` for requests from non-allowlisted IPs. IP allowlisting SHALL be treated as defense-in-depth only — primary protection against unauthorized webhook delivery is signature verification (criterion 1) and timestamp tolerance (criterion 2). IF Stripe stops publishing IP ranges or the operational team determines maintenance overhead outweighs the benefit, THEN the IP allowlist SHALL be removed without weakening other webhook controls
4. THE Platform_Service SHALL verify Stripe Connect webhook signatures separately using the Connect-specific signing secret, distinct from the direct webhook signing secret
5. IF Stripe webhook signature verification fails, THEN THE Transaction_Service SHALL log the failure at WARN level including the source IP address and SHALL NOT process the event payload

### Requirement 12: WebSocket Authentication

**User Story:** As a user receiving real-time updates, I want my WebSocket connection authenticated via JWT, so that only authorized users can establish connections and receive updates.

This requirement extends Phase 5 (Real-time & Notifications), which ships JWT validation on the `/ws?token=...` upgrade handshake and user/role association. Phase 7 adds: in-connection token-refresh handling, sub-claim consistency enforcement on refresh, and explicit close-code semantics.

#### Acceptance Criteria

1. WHEN a client initiates a WebSocket connection, THE Transaction_Service SHALL require a valid JWT access token passed as a query parameter (`?token=<jwt>`) during the WebSocket upgrade handshake (extends Phase 5 Req 6 AC 2)
2. IF the JWT token is missing, expired, or has an invalid RS256 signature, THEN THE Transaction_Service SHALL reject the WebSocket upgrade with HTTP `401 Unauthorized` and SHALL NOT establish the STOMP session (extends Phase 5 Req 6 AC 2)
3. WHEN a WebSocket connection is established, THE Transaction_Service SHALL associate the connection with the authenticated user ID (`sub` claim) and role (`role` claim) extracted from the JWT claims (extends Phase 5 Req 6 AC 3)
4. WHEN the JWT access token is within 60 seconds of expiration (based on the `exp` claim), THE Transaction_Service SHALL send a `TOKEN_EXPIRING` message to the client on the `/user/queue/system` destination (matches `websocket-message-contracts.json` `TOKEN_EXPIRING` schema)
5. WHEN the client sends a message to `/app/token-refresh` with a new valid JWT token, THE Transaction_Service SHALL validate the new token's RS256 signature and `sub` claim (must match the existing connection's user ID), and update the connection's expiration context (matches `websocket-message-contracts.json` `TOKEN_REFRESH` schema)
6. IF no valid token refresh is received within 60 seconds of the `TOKEN_EXPIRING` message being sent, THEN THE Transaction_Service SHALL close the connection with WebSocket close code `4001`
7. IF a token refresh message contains a token with a different `sub` claim than the established connection, THEN THE Transaction_Service SHALL reject the refresh and send an error frame on `/user/queue/system` without closing the connection

### Requirement 13: WebSocket Channel Authorization

**User Story:** As a platform operator, I want channel-level authorization on WebSocket subscriptions, so that users only receive updates they are authorized to see.

#### Acceptance Criteria

1. THE Transaction_Service SHALL enforce that CUSTOMER users can only subscribe to `/user/queue/bookings` for their own bookings, `/user/queue/notifications` for their own notifications, and `/topic/courts/{courtId}/availability` for any court
2. THE Transaction_Service SHALL enforce that COURT_OWNER users can only subscribe to `/user/queue/bookings` for bookings on their own courts, `/user/queue/notifications` for their own notifications, and `/topic/courts/{courtId}/availability` for any court
3. IF a client attempts to subscribe to a STOMP destination for which they lack authorization, THEN THE Transaction_Service SHALL reject the subscription with a STOMP ERROR frame and close the connection with WebSocket close code `4002`
4. WHEN a client subscribes to `/topic/courts/{courtId}/availability`, THE Transaction_Service SHALL allow the subscription for any authenticated user but SHALL NOT include customer names, user IDs, or contact details in availability messages
5. WHEN a client subscribes to `/user/queue/bookings`, THE Transaction_Service SHALL verify the user is either the booking customer (for CUSTOMER role) or the court owner (for COURT_OWNER role) before delivering booking status updates to that connection

### Requirement 14: WebSocket Message Validation and Rate Limiting

**User Story:** As a platform operator, I want WebSocket messages validated and rate-limited, so that malicious clients cannot abuse the real-time connection.

#### Acceptance Criteria

1. THE Transaction_Service SHALL validate all incoming WebSocket messages against expected STOMP frame structure and message schemas, and increment an invalid message counter per connection
2. WHEN a connection accumulates 3 invalid messages, THE Transaction_Service SHALL disconnect the client with WebSocket close code `4004` and a STOMP ERROR frame indicating the reason
3. THE Transaction_Service SHALL enforce a maximum WebSocket message size of 64KB (65,536 bytes) and reject frames exceeding this limit by disconnecting the client with close code `4004`
4. THE Transaction_Service SHALL rate-limit incoming WebSocket messages to 10 messages per second per connection. WHEN the limit is exceeded, THE Transaction_Service SHALL drop the excess messages and emit at most one warning frame per 1-second throttle window on `/user/queue/system` indicating that messages were dropped (no per-message client notification)
5. IF a connection exceeds the rate limit for 3 consecutive seconds, THEN THE Transaction_Service SHALL disconnect the client with WebSocket close code `4003`


### Requirement 15: Data Encryption in Transit

**User Story:** As a platform operator, I want all communication encrypted with modern TLS, so that data cannot be intercepted in transit.

#### Acceptance Criteria

1. THE NGINX_Ingress SHALL enforce TLS 1.2 as the minimum protocol version, disabling TLS 1.0 and TLS 1.1, and SHALL restrict cipher suites to ECDHE-based forward-secrecy ciphers only
2. ALL service-to-service communication SHALL use mTLS via Istio service mesh in staging and production environments, with Istio PeerAuthentication set to STRICT mode
3. ALL connections to managed PostgreSQL SHALL use TLS/SSL with certificate verification enabled (`sslmode=verify-full`) in staging and production environments
4. ALL connections to managed Redis SHALL use TLS/SSL with certificate verification enabled in staging and production environments
5. THE Kafka (Redpanda Serverless) connection SHALL use SASL/SCRAM authentication over TLS with TLS certificate verification enabled
6. IF a service fails to establish a TLS connection to PostgreSQL, Redis, or Kafka, THEN THE service SHALL refuse to start and log an error message indicating the TLS handshake failure and target host

### Requirement 16: Log Sanitization and PII Protection

**User Story:** As a platform operator, I want sensitive data excluded from logs and event payloads, so that RESTRICTED and CONFIDENTIAL data is not exposed through observability systems.

#### Acceptance Criteria

1. BOTH Platform_Service and Transaction_Service SHALL implement a Logback log sanitization filter that strips JWT tokens, refresh tokens, Stripe API keys, and payment card details from all log output before writing to the log sink
2. BOTH services SHALL ensure that Kafka event payloads reference users by UUID only and do not include CONFIDENTIAL data (emails, phone numbers, names)
3. BOTH services SHALL implement a Logback pattern filter that detects and replaces patterns matching credit card numbers (sequences of 13-19 digits), JWT tokens (strings starting with `eyJ`), and API key formats (strings starting with `sk_` or `rk_` followed by alphanumeric characters) with `[REDACTED]`
4. IF a log statement inadvertently contains a RESTRICTED value matching any defined pattern, THEN THE log sanitizer SHALL replace the matched value with `[REDACTED]` before writing to the log sink
5. BOTH services SHALL sanitize exception stack traces and HTTP request/response logs through the same Logback filter, ensuring sensitive values in headers (Authorization, Cookie) and request bodies are masked

### Requirement 17: Data Retention and Anonymization

**User Story:** As a platform operator, I want automated data retention enforcement, so that personal data is anonymized or archived according to GDPR requirements and legal retention periods.

#### Acceptance Criteria

1. THE Platform_Service SHALL implement a weekly scheduled job that anonymizes user data for accounts deleted more than 30 days ago, replacing names and emails with "Anonymized User", clearing phone numbers, and removing profile images, processing a maximum of 10000 records per execution (per-user anonymization cascades to dependent rows in `oauth_providers`, `refresh_tokens`, and `support_tickets` via SQL UPDATE; cascading cost is the rationale for keeping the cap conservative even though it is the same as other retention jobs)
2. THE Platform_Service SHALL implement a monthly scheduled job that archives audit log entries older than 2 years (both `court_owner_audit_logs` and `support_messages` audit content) to compressed cold storage in DigitalOcean Spaces (gzip-compressed JSON Lines, one file per month) and removes them from the active tables, processing a maximum of 10000 records per execution
3. THE Transaction_Service SHALL implement a monthly scheduled job that anonymizes booking records older than 2 years, replacing customer names and emails referenced in `bookings.metadata` JSON with "Anonymized User" while retaining booking amounts, dates, and court references for analytics, processing a maximum of 10000 records per execution
4. THE Platform_Service SHALL implement a monthly scheduled job that archives security alert records older than 1 year (any `security_alerts` row in status RESOLVED or FALSE_POSITIVE) to compressed cold storage and removes them from the active `security_alerts` table, processing a maximum of 10000 records per execution
5. WHEN an anonymization or archival job runs, THE job SHALL log the count of records processed and publish a summary event to the security audit trail without including any PII
6. IF an anonymization or archival job fails mid-execution, THEN THE job SHALL roll back the current batch, log the error with the number of records successfully processed before failure, and retry on the next scheduled run

### Requirement 18: Secret Rotation Procedures

**User Story:** As a platform operator, I want automated secret rotation with zero-downtime procedures, so that compromised credentials have limited exposure windows.

#### Acceptance Criteria

1. THE Platform_Service SHALL support JWT signing key rotation every 90 days: generate new key pair, publish new public key to the JWKS endpoint at `/api/auth/.well-known/jwks.json`, start signing with new private key, retain old public key for 15 minutes (max access token lifetime), then retire old key pair
2. THE database credentials SHALL be rotated every 90 days via External Secrets Operator with zero-downtime: new credentials provisioned, services pick up new credentials via secret refresh, old credentials revoked after a 5-minute grace period to allow in-flight connections to complete
3. THE internal API key (service-to-service authentication in dev/test) SHALL be rotated every 30 days, with both old and new keys accepted during a 5-minute overlap window
4. THE Stripe webhook signing secrets SHALL be rotated annually or immediately if a compromise is suspected, with both old and new secrets validated during a 60-minute overlap window to handle in-flight webhook deliveries
5. ALL secret rotation events SHALL be logged in the security audit trail with rotation timestamp, secret type (JWT_SIGNING_KEY, DATABASE_CREDENTIAL, INTERNAL_API_KEY, STRIPE_WEBHOOK_SECRET), and initiator (AUTOMATED or admin user ID)
6. IF a secret rotation fails at any step, THEN THE system SHALL retain the current active secret, log the failure in the security audit trail, and publish a HIGH severity SECURITY_ALERT to notify PLATFORM_ADMIN users

### Requirement 19: Stripe Connect Security Monitoring

**User Story:** As a platform operator, I want Stripe Connect account status monitored and enforced, so that bookings are not processed for court owners with restricted or disabled Stripe accounts.

#### Acceptance Criteria

1. THE Platform_Service SHALL verify Stripe Connect account status (`charges_enabled` and `payouts_enabled` both equal `true`) before allowing customer bookings for a court owner's courts, rejecting booking attempts with an error message indicating the court is temporarily unavailable for online booking
2. WHEN a Stripe `account.updated` webhook indicates `charges_enabled` is `false` or `payouts_enabled` is `false`, THE Platform_Service SHALL immediately set court visibility to hidden for all courts owned by that account, update `stripe_connect_status` to RESTRICTED, and publish a HIGH severity SECURITY_ALERT with alertType PAYMENT_FRAUD
3. WHEN a Stripe `account.application.deauthorized` webhook is received, THE Platform_Service SHALL set `stripe_connect_status` to DISABLED, set court visibility to hidden for all courts owned by that account, and notify the court owner via email within 5 minutes of webhook receipt
4. THE Platform_Service SHALL require re-verification of Stripe Connect identity when a court owner changes bank account details, setting `stripe_connect_status` to PENDING until Stripe confirms the updated account details via a subsequent `account.updated` webhook with `charges_enabled: true`
5. WHEN a court owner re-onboards Stripe Connect within 7 days of a prior `account.application.deauthorized` event AND the new Stripe Connect account is linked to a bank account whose IBAN differs from the IBAN previously associated with that owner, THE Platform_Service SHALL publish a HIGH severity SECURITY_ALERT with alertType ACCOUNT_TAKEOVER, hold all payouts to the new account until a PLATFORM_ADMIN resolves the alert, and send an email to the court owner's primary email asking them to confirm the bank-account change before payouts resume. This protects against the deauthorize-then-reauthorize-with-different-bank attack vector (Master Req 36 deauthorization redirect prevention)
6. IF the Stripe API is unreachable when verifying account status for a booking attempt, THEN THE Platform_Service SHALL reject the booking with an error message indicating a temporary service issue and retry verification on the next booking attempt

### Requirement 20: Payout Fraud Detection

**User Story:** As a platform operator, I want payout amounts monitored for unusual patterns, so that fraudulent payouts are caught before funds leave the platform.

#### Acceptance Criteria

1. WHEN a single payout exceeds €5,000, THE Transaction_Service SHALL publish a SECURITY_ALERT event with alertType PAYMENT_FRAUD, severity MEDIUM, and metadata containing the payout amount in cents and the court owner's user ID
2. WHEN the total weekly payouts (Monday 00:00 UTC to Sunday 23:59 UTC) for a single court owner exceed €10,000, THE Transaction_Service SHALL publish a SECURITY_ALERT event with alertType PAYMENT_FRAUD, severity HIGH, and metadata containing the weekly total in cents and the court owner's user ID
3. WHEN a court owner's weekly payout total exceeds 300% of their average weekly payout total over the preceding 4 weeks, THE Transaction_Service SHALL publish a SECURITY_ALERT event with alertType PAYMENT_FRAUD, severity HIGH, and metadata containing the current week total, the 4-week average, and the percentage increase
4. WHEN a payout is flagged by any of criteria 1–3, THE Transaction_Service SHALL set the payout status to HELD and prevent the Stripe transfer until a PLATFORM_ADMIN updates the associated security alert status to RESOLVED or FALSE_POSITIVE
5. IF a court owner has fewer than 4 weeks of payout history, THEN THE Transaction_Service SHALL skip the percentage-increase check (criterion 3) for that court owner

### Requirement 21: Self-Booking Fraud Detection

**User Story:** As a platform operator, I want detection of court owners booking their own courts through fake customer accounts, so that fraudulent payouts are prevented.

#### Acceptance Criteria

1. WHEN a booking is created, THE Transaction_Service SHALL evaluate self-booking fraud indicators by checking whether: the customer's IP address matches the court owner's last known IP address, the customer's device ID matches the court owner's last known device ID, or the customer account was created within 24 hours before the booking time
2. IF at least one self-booking fraud indicator from criterion 1 matches, THEN THE Transaction_Service SHALL publish a SECURITY_ALERT event with alertType PAYMENT_FRAUD, severity CRITICAL, and metadata containing the list of matched indicators (matched IP, matched device ID, account age in hours at booking time)
3. WHEN self-booking fraud is suspected per criterion 2, THE Transaction_Service SHALL set the associated payout status to HELD and prevent the Stripe transfer until a PLATFORM_ADMIN resolves the alert
4. THE Transaction_Service SHALL record the court owner's last known IP address and device ID in Redis (key pattern `user:last-seen:{userId}` with a 30-day TTL) on each authenticated request, with batched persistence to the `users` table no more frequently than once every 5 minutes per court owner to avoid per-request DB writes. The fraud-indicator check (criterion 1) SHALL read from Redis with a fall-through to the persisted DB columns when Redis is unavailable

### Requirement 22: Input Validation and Sanitization

**User Story:** As a developer, I want comprehensive input validation across all endpoints, so that injection attacks and malformed data are rejected at the boundary.

#### Acceptance Criteria

1. BOTH Platform_Service and Transaction_Service SHALL validate all input parameters against strict schemas: string fields limited to a maximum of 5,000 characters unless a field-specific limit is defined, numeric values within their declared min/max range, enum values matching declared membership, dates in ISO-8601 format, and UUIDs in standard 8-4-4-4-12 hex format, rejecting non-conforming requests with `400 Bad Request` and a response body indicating which field failed validation
2. THE Platform_Service SHALL sanitize all user-generated text content (court descriptions, support messages) using OWASP Java HTML Sanitizer with an allowlist-based policy permitting only `<p>`, `<br>`, `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>` tags and stripping all HTML attributes, JavaScript, and unrecognized tags
3. THE Platform_Service SHALL re-encode uploaded images (JPEG, PNG, WebP formats only, maximum 10MB per file) to strip EXIF metadata and potential embedded payloads, rejecting files that fail re-encoding with `422 Unprocessable Entity`
4. THE Platform_Service SHALL scan PDF uploads (verification documents, support attachments) for embedded JavaScript and reject PDFs containing executable content with `422 Unprocessable Entity`
5. BOTH services SHALL validate that the `Content-Type` header matches the actual request body format and reject mismatches with `415 Unsupported Media Type`
6. THE NGINX_Ingress SHALL enforce maximum request body sizes: 10MB for image upload paths (`/api/courts/*/images`), 1MB for all other paths, rejecting oversized requests with `413 Payload Too Large`

### Requirement 23: Automated Security Responses

**User Story:** As a platform operator, I want automated responses to high-severity security alerts, so that threats are mitigated immediately without waiting for manual intervention.

#### Acceptance Criteria

1. WHEN a SECURITY_ALERT event with severity HIGH or CRITICAL is persisted and the auto-response rule for that alertType is set to AUTOMATIC, THE Platform_Service SHALL restrict the associated user account to read-only mode (existing sessions remain active but the user cannot create bookings, initiate payments, or modify data) within 5 seconds of alert creation, and the restriction SHALL remain until a PLATFORM_ADMIN updates the alert status to RESOLVED or FALSE_POSITIVE
2. WHEN an IP address triggers a BRUTE_FORCE alert (credential stuffing detection), THE Platform_Service SHALL add the IP address to the ip_blocklist table with a TTL of 24 hours and record the action in the security audit trail including the related alert ID
3. THE Platform_Service SHALL enforce auto-response rules per alertType: each alertType (BOOKING_ABUSE, PAYMENT_FRAUD, SCRAPING, BRUTE_FORCE, SUSPICIOUS_LOGIN, RATE_LIMIT_EXCEEDED, WEBHOOK_REPLAY, ACCOUNT_TAKEOVER) is independently configurable as AUTOMATIC, MANUAL_REVIEW, or DISABLED
4. WHEN an automated response action (account restriction or IP block) is executed, THE Platform_Service SHALL publish a follow-up SECURITY_ALERT event with severity LOW, alertType matching the original, and metadata documenting the automated action taken (action type, target user ID or IP, original alert ID)

### Requirement 24: Security Alerts Dashboard API

**User Story:** As a platform administrator, I want a dashboard showing security alerts with filtering and actions, so that I can monitor and respond to security incidents efficiently.

#### Acceptance Criteria

1. THE Platform_Service SHALL provide a paginated REST API endpoint (`GET /api/admin/security/alerts`) returning security alerts sorted by created_at descending, with optional query parameters for filtering by: severity (LOW, MEDIUM, HIGH, CRITICAL), alertType, status (OPEN, ACKNOWLEDGED, INVESTIGATING, RESOLVED, FALSE_POSITIVE), date range (`fromDate`, `toDate` ISO-8601 timestamps), userId (UUID), and ipAddress (string, max 45 characters), with a default page size of 20 and a maximum page size of 100. THE endpoint matches the existing OpenAPI definition in `docs/api/openapi-platform-service.yaml` (operationId `listSecurityAlerts`) and SHALL be accessible to PLATFORM_ADMIN (full access) and SUPPORT_AGENT (read-only)
2. THE Platform_Service SHALL provide endpoints (`POST /api/admin/security/alerts/{alertId}/acknowledge` and `POST /api/admin/security/alerts/{alertId}/resolve`) for PLATFORM_ADMIN to update alert status. The acknowledge endpoint transitions OPEN → ACKNOWLEDGED with no body. The resolve endpoint accepts `{ status: 'RESOLVED' | 'FALSE_POSITIVE', resolutionNotes?: string }` (resolutionNotes max 2000 characters; required when status is RESOLVED or FALSE_POSITIVE per Req 2 AC 6) and transitions to a terminal status. Both endpoints record the acting admin's user ID and timestamp on the alert record. THE endpoints match the existing OpenAPI definition (operationIds `acknowledgeSecurityAlert`, `resolveSecurityAlert`)
3. THE Platform_Service SHALL provide an endpoint (`GET /api/admin/security/alerts/summary`) returning aggregate counts of alerts grouped by severity and by alertType, counting only alerts with status OPEN, ACKNOWLEDGED, or INVESTIGATING. SUPPORT_AGENT has read-only access; PLATFORM_ADMIN has full access
4. THE Admin_Web SHALL display the security alerts dashboard and subscribe to a new STOMP destination `/user/queue/security-alerts` to receive real-time push updates when new alerts with severity CRITICAL or HIGH are created. THE Phase 7 implementation SHALL extend `docs/api/websocket-message-contracts.json` to register `/user/queue/security-alerts` (allowedRoles: PLATFORM_ADMIN, SUPPORT_AGENT) and define a `SECURITY_ALERT_PUSH` server-to-client message schema. The Transaction Service publishes to this destination via Redis Pub/Sub when consuming a CRITICAL or HIGH severity event from the `security-events` topic. The admin dashboard appends pushed alerts to the displayed list without requiring a page refresh

### Requirement 25: Automated Responses Configuration API

**User Story:** As a platform administrator, I want to configure which security alert types trigger automated responses, so that I can tune the system behavior without code changes.

#### Acceptance Criteria

1. THE Platform_Service SHALL provide a REST API (`GET /api/admin/security/auto-responses`) for PLATFORM_ADMIN to retrieve the current auto-response configuration, returning one entry per alertType (BOOKING_ABUSE, PAYMENT_FRAUD, SCRAPING, BRUTE_FORCE, SUSPICIOUS_LOGIN, RATE_LIMIT_EXCEEDED, WEBHOOK_REPLAY, ACCOUNT_TAKEOVER) with its current response mode (AUTOMATIC, MANUAL_REVIEW, or DISABLED), the timestamp of the last modification, and the user ID of the admin who last modified it. The endpoint coexists with the existing `/api/admin/security/config` (`SecurityConfig` schema covers detection thresholds; `auto-responses` covers alert→action mapping)
2. THE Platform_Service SHALL provide a REST API (`PUT /api/admin/security/auto-responses`) for PLATFORM_ADMIN to update auto-response rules, accepting a list of alertType-to-responseMode mappings, validating that each alertType is a recognized value and each responseMode is one of AUTOMATIC, MANUAL_REVIEW, or DISABLED, rejecting invalid values with `400 Bad Request`
3. WHEN a PLATFORM_ADMIN updates auto-response configuration, THE Platform_Service SHALL persist the change to the `security_auto_response_config` table (created by the Phase 7 Flyway migration enumerated in Requirement 30) and record an audit trail entry containing the admin's user ID, the previous configuration values, the new configuration values, and the timestamp of the change
4. THE default auto-response configuration seeded by the Phase 7 Flyway migration SHALL be: BRUTE_FORCE = AUTOMATIC, ACCOUNT_TAKEOVER = AUTOMATIC, PAYMENT_FRAUD = MANUAL_REVIEW, BOOKING_ABUSE = AUTOMATIC, SCRAPING = AUTOMATIC, SUSPICIOUS_LOGIN = MANUAL_REVIEW, RATE_LIMIT_EXCEEDED = AUTOMATIC, WEBHOOK_REPLAY = MANUAL_REVIEW. The defaults reflect that automated lockout is preferred for clearly malicious traffic and deferred to admin review for cases where false positives could lock out legitimate users or court owners

### Requirement 26: Encryption at Rest Verification

**User Story:** As a platform operator, I want all data encrypted at rest using managed service encryption, so that data is protected even if storage media is compromised.

#### Acceptance Criteria

1. THE DigitalOcean Managed PostgreSQL instance SHALL have encryption at rest enabled (AES-256, managed by DigitalOcean) — verified via Terraform configuration and DigitalOcean API
2. THE DigitalOcean Managed Redis instance SHALL have encryption at rest enabled — verified via Terraform configuration
3. THE DigitalOcean Spaces (S3-compatible storage) SHALL have server-side encryption enabled for all stored objects (court images, verification documents, support attachments) — verified via bucket policy configuration
4. ALL database backups SHALL be encrypted at rest using DigitalOcean's managed encryption — verified via backup configuration settings
5. THE infrastructure Terraform modules SHALL include explicit encryption configuration parameters for all managed services, and `terraform plan` SHALL fail if encryption settings are missing or disabled

### Requirement 27: Output Encoding and XSS Prevention

**User Story:** As a platform operator, I want all API responses properly encoded and client applications protected against XSS, so that injected content cannot execute in user browsers.

#### Acceptance Criteria

1. BOTH Platform_Service and Transaction_Service SHALL set `Content-Type: application/json; charset=utf-8` on all JSON responses to prevent content-type sniffing
2. THE Admin_Web (React) SHALL use React's built-in JSX escaping for all dynamic content rendering and SHALL NOT use `dangerouslySetInnerHTML` without explicit sanitization via DOMPurify
3. THE Admin_Web SHALL set `X-XSS-Protection: 0` (disabled in favor of CSP, per modern best practice) — this is set alongside the CSP header from Requirement 8
4. THE mobile application (Flutter) SHALL sanitize any HTML content received from the API (court descriptions, support messages) before rendering, using a safe HTML rendering widget that strips script tags and event handlers. **⚠ Cross-phase note:** the Flutter app is built in Phase 8 (`court-booking-mobile-app`); Phase 7 captures this requirement only as the contract that Phase 8 must satisfy. Verification (a Flutter widget test that asserts script tags are stripped from rendered HTML) is part of Phase 8's deliverables, not Phase 7
5. THE mobile application (Flutter) is exempt from CSRF protection as it uses Bearer token authentication without cookies. **⚠ Cross-phase note:** specification only; verified in Phase 8

### Requirement 28: Data Classification and Access Control Enforcement

**User Story:** As a platform operator, I want a formal data classification system enforced across all services, so that sensitive data is handled according to its classification level.

#### Acceptance Criteria

1. THE platform SHALL enforce the following four-tier data classification:

| Classification | Examples | Access Controls | Retention |
|---------------|----------|-----------------|-----------|
| **PUBLIC** | Court names, addresses, types, amenities, pricing, availability | No auth required for read | Indefinite |
| **INTERNAL** | Booking counts, aggregated analytics, feature flags | Authenticated users with appropriate role | Indefinite |
| **CONFIDENTIAL** | User emails, phone numbers, names, booking details, payment amounts | Owner + authorized roles per matrix | Active: indefinite; Deleted accounts: anonymized |
| **RESTRICTED** | Payment card tokens (Stripe-managed), refresh token hashes, OAuth tokens, Stripe API keys, JWT signing keys | Service-level only, never exposed in API responses or logs | Per Stripe/provider policy; tokens: 30 days |

2. BOTH services SHALL ensure RESTRICTED data is never included in API response bodies — refresh token hashes, OAuth tokens, Stripe API keys, and JWT signing keys SHALL never appear in any REST API response
3. BOTH services SHALL ensure CONFIDENTIAL data (user emails, phone numbers) is not returned in list endpoints to unauthorized roles — only the data owner or authorized roles per the authorization matrix SHALL receive CONFIDENTIAL fields
4. THE Platform_Service SHALL track failed authentication attempts in Redis with a sliding window counter keyed by BOTH IP address AND user email (dual-key tracking) to enable detection of distributed attacks targeting a single account across multiple IPs
5. THE SUPPORT_AGENT role SHALL have read-only access to the security alerts dashboard (`GET /api/admin/security/alerts` and `GET /api/admin/security/alerts/summary`) but SHALL NOT be able to update alert status, manage IP blocklist, or configure auto-response rules

### Requirement 29: Phase 7 Database Schema Migrations

**User Story:** As a developer implementing Phase 7, I want every new column, table, and constraint enumerated in one place, so that the Flyway migration plan is unambiguous and database review can be batched.

This requirement enumerates the schema additions Phase 7 requires beyond what already exists in the Phase 1b base migration. All columns referenced elsewhere in this document (Req 4 `bookings.fraud_review_required`, Req 19 `users.stripe_connect_status` extensions, Req 21 court-owner last-seen tracking, Req 23 user restriction model, Req 25 auto-response config) trace back to migration entries here.

#### Acceptance Criteria

1. THE Phase 7 platform-schema migration `V7__phase7_security_schema.sql` SHALL add the following columns and tables to the `platform` schema:
   - `users.last_known_ip_address VARCHAR(45) NULL` and `users.last_known_device_id VARCHAR(255) NULL` and `users.last_seen_at TIMESTAMPTZ NULL` for self-booking-fraud baselines (Req 21). These columns are populated from Redis on a 5-minute persistence cadence
   - `users.restriction_status VARCHAR(20) NOT NULL DEFAULT 'NONE'` with `CHECK (restriction_status IN ('NONE','READ_ONLY'))` and `users.restriction_reason VARCHAR(255) NULL` and `users.restriction_alert_id UUID NULL REFERENCES security_alerts(id)` and `users.restricted_at TIMESTAMPTZ NULL` for Req 23 read-only mode enforcement
   - Extend `users.stripe_connect_status` CHECK constraint to include all values referenced by Req 19: `('NONE','PENDING','ACTIVE','RESTRICTED','DISABLED','UNDER_REVIEW')`. UNDER_REVIEW is added by Phase 7 (per Req 4 AC 2); the prior values are preserved
   - `users.previous_stripe_iban_hash VARCHAR(64) NULL` to support the deauthorize-then-reauthorize-with-different-bank detection in Req 19 AC 5 (SHA-256 hash of the IBAN; raw IBAN is never stored)
   - New table `security_auto_response_config` with columns `(alert_type VARCHAR(50) PRIMARY KEY CHECK alert_type IN (...), response_mode VARCHAR(20) NOT NULL CHECK response_mode IN ('AUTOMATIC','MANUAL_REVIEW','DISABLED'), updated_by UUID NULL REFERENCES users(id), updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW())`, seeded with the defaults from Req 25 AC 4
   - New table `security_audit_trail` with columns `(id UUID PK, actor_user_id UUID NULL REFERENCES users(id), action VARCHAR(50) NOT NULL, target_user_id UUID NULL REFERENCES users(id), target_ip VARCHAR(45) NULL, related_alert_id UUID NULL REFERENCES security_alerts(id), metadata JSONB NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT NOW())` for Req 18 AC 5 (secret rotation events) and Req 23 AC 4 (automated action records). Append-only; no UPDATE or DELETE permitted
2. THE Phase 7 transaction-schema migration `V<n>__phase7_fraud_columns.sql` SHALL add the following to the `transaction` schema:
   - `bookings.fraud_review_required BOOLEAN NOT NULL DEFAULT FALSE` and `bookings.fraud_metadata JSONB NULL` for Req 4 AC 5 country-mismatch flagging
   - `payments.payout_status VARCHAR(20) NOT NULL DEFAULT 'NORMAL'` with `CHECK (payout_status IN ('NORMAL','HELD','RELEASED','CANCELLED'))` and `payments.payout_hold_reason VARCHAR(255) NULL` and `payments.payout_held_alert_id UUID NULL` for Req 20 AC 4 and Req 21 AC 3 payout holds. Cross-schema FK to `platform.security_alerts` is intentionally omitted (PostgreSQL prohibits cross-schema FKs across schemas with separate write owners); the linkage is application-enforced
   - Index `idx_bookings_fraud_review` on `bookings(fraud_review_required) WHERE fraud_review_required = TRUE`
   - Index `idx_payments_held` on `payments(payout_status) WHERE payout_status = 'HELD'`
3. THE migrations SHALL be idempotent and reversible; each ALTER COLUMN operation SHALL be paired with a documented rollback in the Phase 7 design document. The migrations SHALL pass `flyway validate` against a copy of the staging database before merge, per the existing Phase 1b CI gate
4. WHEN the migrations run on an environment that already has data, THE backfill SHALL: set `restriction_status = 'NONE'` for all existing users (already the default), set `fraud_review_required = FALSE` for all existing bookings (already the default), and set `payout_status = 'NORMAL'` for all existing payments (already the default); no data backfill beyond defaults is required
5. THE Phase 7 OpenAPI updates SHALL extend `docs/api/openapi-platform-service.yaml` to include the auto-responses endpoints (Req 25) and `docs/api/websocket-message-contracts.json` to register the `/user/queue/security-alerts` STOMP destination and `SECURITY_ALERT_PUSH` message schema (Req 24 AC 4); these are documentation deliverables that gate the Phase 7 PR merge per the standing OpenAPI maintenance requirement (Master Req 29)

---

## Phase 2 Exclusions

The following requirements from the master document are explicitly excluded from Phase 7 and deferred to Phase 2:

- ⏳ **PHASE 2** — Waitlist manipulation detection (Master Req 32.15): detecting users joining and leaving waitlists repeatedly to block slots
- ⏳ **PHASE 2** — Open match abuse detection (Master Req 32.16): detecting creators who cancel matches after players have joined
- ⏳ **PHASE 2** — Promo code abuse detection (Master Req 32.17, 32.18): multi-account promo code abuse and global rate limits on promo code redemptions
