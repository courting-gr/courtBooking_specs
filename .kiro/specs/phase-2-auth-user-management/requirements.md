# Requirements Document — Phase 2: Auth & User Management

## Introduction

Phase 2 implements the complete authentication, authorization, and user management subsystem for the Court Booking Platform, targeting the `court-booking-platform-service`. This phase delivers OAuth-based registration and login, JWT token issuance with role-based claims, refresh token rotation with replay detection, biometric authentication support, role-based access control enforcement, user profile management, GDPR-compliant account deletion, concurrent session management, rate limiting on auth endpoints, and OAuth provider linking/unlinking.

This phase builds on Phase 1a (infrastructure) and Phase 1b (service scaffolding), which established the database schema (including `users`, `oauth_providers`, `refresh_tokens`, `failed_auth_attempts` tables), Flyway migrations, Spring Boot project structure with hexagonal architecture, and CI/CD pipelines.

**Subscription stub strategy:** Until Phase 10 (Subscription Billing), all court owners default to `ACTIVE` subscription status. The `subscriptionStatus` JWT claim is included but subscription enforcement is deferred. The `subscription_status` column defaults to `NONE` in the database; the JWT claim maps `NONE` → `ACTIVE` for court owners until Phase 10 activates real billing.

**Scope boundaries:**
- Court owner verification workflow (submit documents, admin review) is Phase 3 (Court Management).
- Stripe Connect onboarding is Phase 4 (Booking & Payments).
- Subscription billing enforcement is Phase 10.
- Security hardening (abuse detection, IP blocklist management, security alerts) is Phase 7.
- This phase focuses on the Platform Service auth domain. Transaction Service JWT validation is included as a cross-service concern.

## Glossary

- **Platform_Service**: The Spring Boot microservice responsible for authentication, authorization, user management, courts, availability, analytics, and support. Deployed to `court-booking-platform-service`.
- **Transaction_Service**: The Spring Boot microservice responsible for bookings, payments, and notifications. Validates JWT tokens independently using the shared RS256 public key.
- **OAuth_Provider**: An external identity provider (Google, Facebook, or Apple ID) used for user authentication.
- **JWT_Access_Token**: A short-lived (15-minute) RS256-signed JSON Web Token containing role-based claims, validated independently by both services.
- **Refresh_Token**: A long-lived (30-day) opaque token stored server-side (SHA-256 hashed in `refresh_tokens` table), used to obtain new access tokens without re-authentication.
- **Refresh_Token_Rotation**: The security mechanism where each refresh request issues a new refresh token and invalidates the previous one.
- **Replay_Detection**: The mechanism that detects reuse of an already-invalidated refresh token, triggering revocation of all refresh tokens for that user.
- **Biometric_Authentication**: A mobile authentication flow where biometric unlock (fingerprint/face ID) gates access to a refresh token stored in the device secure enclave (iOS Keychain / Android Keystore).
- **Device_Secure_Enclave**: The hardware-backed secure storage on mobile devices (iOS Keychain / Android Keystore) where biometric-gated refresh tokens are stored.
- **Session**: A logical user session represented by a refresh token entry in the `refresh_tokens` table, associated with a specific device.
- **Authorization_Matrix**: The role-to-operation permission mapping that governs access control across the platform (defined in Requirement 1 of the master requirements document).
- **CUSTOMER**: A platform role for users who book courts, join matches, and manage personal bookings.
- **COURT_OWNER**: A platform role for users who manage courts, availability, pricing, and view analytics. Has sub-states defined below.
- **COURT_OWNER_Sub_States**: The operational states within the COURT_OWNER role that affect available operations:
  - **Unverified**: Can set up courts (hidden from customers), access admin dashboard. Cannot have courts publicly visible.
  - **Verified**: Courts become publicly visible. Full admin features available.
  - **Stripe_Not_Connected**: Manual bookings work. Customer bookings blocked (courts not bookable by customers).
  - **Stripe_Connected**: Full payment flow enabled.
  - **Trial_Active**: Full feature access with trial banner. (Stub: until Phase 10, all court owners default to ACTIVE — see subscription stub strategy.)
  - **Subscribed**: Full feature access, active subscription.
  - **Expired**: Read-only access to existing data. No new manual bookings. (Stub: not enforced until Phase 10.)
- **SUPPORT_AGENT**: A platform role for users who view and respond to support tickets and view security alerts (read-only). Cannot manage feature flags, users, system config, or security actions. Assigned by PLATFORM_ADMIN.
- **PLATFORM_ADMIN**: A platform role for users with full platform-wide operations: user management, dispute escalation, platform promo codes, feature flags, system configuration, security management. Not self-registerable; seeded or created by another PLATFORM_ADMIN.
- **Rate_Limiter**: A mechanism that restricts the number of requests to authentication endpoints within a time window to prevent brute force attacks.
- **Failed_Auth_Attempts_Table**: Database table (`failed_auth_attempts`) tracking failed authentication attempts per IP address with columns: `ip_address`, `attempt_count`, `window_start`, `locked_until`. Used for IP-based lockout mechanism.
- **GDPR_Anonymization**: The process of replacing personal data with anonymized values in historical records while preserving the records for analytics and audit trail integrity.
- **JWKS_Endpoint**: A well-known endpoint (`GET /api/auth/.well-known/jwks.json`) that exposes the RS256 public key in JSON Web Key Set format for JWT validation by external services.
- **Service_To_Service_Auth**: Authentication between Platform Service and Transaction Service — mTLS via Istio in staging/production, shared API key via `X-Internal-Api-Key` header in dev/test.
- **Shared_JWT_Filter**: A Spring Security filter packaged in `court-booking-common` library that provides consistent JWT validation logic for both Platform Service and Transaction Service.
- **Admin_User_Management_API**: The set of administrative endpoints (`/api/admin/users`) that allow PLATFORM_ADMIN users to create, suspend, unsuspend, and list user accounts.
- **Notification_Preferences**: User-configurable settings controlling which notification types (booking events, favorite alerts, promotional) are received and through which channels (email, push, in-app), stored in the `preferences` table.
- **Do_Not_Disturb**: A user-configurable time window (start time, end time) during which non-critical notifications are suppressed.
- **Suspended_Users_Cache**: A Redis-backed cache of suspended user IDs used by the JWT filter to reject requests from suspended accounts without a database lookup on every request.

## Requirements

### Requirement 1: OAuth Registration

**User Story:** As a new user, I want to register using my existing Google, Facebook, or Apple account, so that I can quickly create a platform account without managing a separate password.

#### Acceptance Criteria

1. WHEN a user initiates registration, THE Platform_Service SHALL present OAuth registration options for Google, Facebook, and Apple ID
2. WHEN a user selects an OAuth_Provider for registration, THE Platform_Service SHALL redirect to the selected provider's authorization endpoint with the appropriate scopes (email, profile)
3. WHEN the OAuth_Provider returns an authorization code, THE Platform_Service SHALL exchange the code for provider tokens and extract the user's email and profile information
4. WHEN a user registers via OAuth, THE Platform_Service SHALL create a new user record in the `users` table with the provider information stored in the `oauth_providers` table and the selected role (CUSTOMER or COURT_OWNER)
5. WHEN registration is initiated, THE Platform_Service SHALL present terms and conditions that the user must accept before account creation, recording the acceptance timestamp in `users.terms_accepted_at`
6. WHEN terms and conditions are not accepted, THE Platform_Service SHALL prevent account creation and return an error indicating terms acceptance is required
7. WHEN a user attempts to register with an email that already exists in the `users` table, THE Platform_Service SHALL return an error indicating the account already exists and suggest login instead
8. WHEN a user registers as COURT_OWNER, THE Platform_Service SHALL set `users.verified` to FALSE, `users.stripe_connect_status` to `NOT_STARTED`, and `users.subscription_status` to `NONE` (mapped to ACTIVE in JWT until Phase 10), and record `users.terms_accepted_at` with the current timestamp
9. THE Platform_Service SHALL validate that the role selection is either CUSTOMER or COURT_OWNER during self-registration; SUPPORT_AGENT and PLATFORM_ADMIN roles are not self-registerable

### Requirement 2: OAuth Login

**User Story:** As a returning user, I want to log in using my linked OAuth provider, so that I can access my account without managing a separate password.

#### Acceptance Criteria

1. WHEN a user selects an OAuth_Provider for login, THE Platform_Service SHALL redirect to the selected provider's authorization endpoint
2. WHEN the OAuth_Provider returns a valid authorization code, THE Platform_Service SHALL exchange the code for provider tokens and look up the user by `oauth_providers.provider` and `oauth_providers.provider_user_id`
3. WHEN a matching user is found, THE Platform_Service SHALL issue a JWT_Access_Token and a Refresh_Token as defined in Requirement 3
4. WHEN no matching user is found for the OAuth provider credentials, THE Platform_Service SHALL return an error indicating the account does not exist and suggest registration
5. WHEN the user's account status is SUSPENDED, THE Platform_Service SHALL reject the login attempt and return an error indicating the account is suspended
6. WHEN the user's account status is DELETED, THE Platform_Service SHALL reject the login attempt and return an error indicating the account no longer exists
7. WHEN an OAuth_Provider becomes unavailable during login, THE Platform_Service SHALL return an error with the provider name and suggest trying an alternative linked provider

### Requirement 3: JWT Access Token Issuance

**User Story:** As an authenticated user, I want to receive a JWT access token with my role and permissions encoded, so that both Platform Service and Transaction Service can authorize my requests without additional lookups.

#### Acceptance Criteria

1. WHEN authentication is complete (via OAuth login, token refresh, or biometric flow), THE Platform_Service SHALL issue a JWT_Access_Token signed with RS256 using the platform's private key
2. THE JWT_Access_Token SHALL contain the following claims: `sub` (user UUID), `role` (CUSTOMER, COURT_OWNER, SUPPORT_AGENT, or PLATFORM_ADMIN), `email` (user email), `iat` (issued-at timestamp), `exp` (expiration timestamp set to 15 minutes from issuance)
3. WHEN the authenticated user has the COURT_OWNER role, THE JWT_Access_Token SHALL additionally contain: `verified` (boolean from `users.verified`), `stripeConnected` (boolean derived from `users.stripe_connect_status = 'ACTIVE'`), `subscriptionStatus` (mapped from `users.subscription_status`, with `NONE` mapped to `ACTIVE` until Phase 10)
4. THE Platform_Service and Transaction_Service SHALL validate JWT_Access_Token signatures independently using the shared RS256 public key, rejecting tokens with invalid signatures, expired timestamps, or missing required claims
5. THE JWT_Access_Token SHALL have a fixed lifetime of 15 minutes; the Platform_Service SHALL NOT issue tokens with longer lifetimes

### Requirement 4: Refresh Token Lifecycle

**User Story:** As an authenticated user, I want my session to persist beyond the 15-minute access token lifetime without re-authenticating, so that I have a seamless experience while maintaining security through token rotation.

#### Acceptance Criteria

1. WHEN a JWT_Access_Token is issued (login or refresh), THE Platform_Service SHALL also issue a Refresh_Token with a 30-day lifetime, storing its SHA-256 hash in the `refresh_tokens` table along with `device_id`, `device_info`, and `ip_address`
2. WHEN a client presents a valid Refresh_Token to `POST /api/auth/refresh`, THE Platform_Service SHALL issue a new JWT_Access_Token and a new Refresh_Token, and invalidate the previous Refresh_Token by setting `refresh_tokens.invalidated` to TRUE
3. WHEN a client presents a Refresh_Token that has already been invalidated (Replay_Detection), THE Platform_Service SHALL revoke ALL Refresh_Tokens for that user by setting `invalidated` to TRUE on all entries, forcing full re-authentication on all devices
4. WHEN a client presents a Refresh_Token that has expired (past `refresh_tokens.expires_at`), THE Platform_Service SHALL reject the request and return an error requiring full re-authentication
5. WHEN a user logs out, THE Platform_Service SHALL invalidate all Refresh_Tokens associated with the current device (matched by `device_id`)
6. THE Platform_Service SHALL update `refresh_tokens.last_used_at` on each successful token refresh to track session activity

### Requirement 5: Biometric Authentication

**User Story:** As a mobile user, I want to use fingerprint or face recognition to quickly access my account, so that I can log in without going through the OAuth flow each time.

#### Acceptance Criteria

1. WHEN a mobile user enables biometric authentication, THE Platform_Service SHALL issue a Refresh_Token designated for storage in the Device_Secure_Enclave (iOS Keychain / Android Keystore), with the same 30-day lifetime and rotation rules as standard Refresh_Tokens
2. WHEN a returning mobile user opens the app and biometric authentication is enabled, THE Platform_Service SHALL offer biometric authentication (fingerprint or face recognition) as a login option
3. WHEN biometric authentication succeeds on the device, the client SHALL retrieve the stored Refresh_Token from the Device_Secure_Enclave and present it to `POST /api/auth/refresh` to obtain a new JWT_Access_Token
4. WHEN biometric unlock fails three consecutive times, the client SHALL fall back to the OAuth login screen
5. WHEN the biometric-stored Refresh_Token has expired or been revoked (e.g., due to Replay_Detection or session limit enforcement), THE Platform_Service SHALL reject the refresh request, and the client SHALL redirect to the OAuth login screen with a "Session expired, please log in again" message
6. THE Platform_Service SHALL differentiate biometric-designated refresh tokens from standard refresh tokens by storing a `biometric` boolean flag in the `refresh_tokens` table. A Flyway migration (`V2__add_biometric_flag.sql`) SHALL add the `biometric` column (BOOLEAN, default FALSE) to the `refresh_tokens` table, since the Phase 1b schema does not include this column

### Requirement 6: Role-Based Access Control

**User Story:** As a platform operator, I want the system to enforce role-based permissions on every API request, so that users can only perform actions authorized for their role.

#### Authorization Matrix

| Operation | CUSTOMER | COURT_OWNER | SUPPORT_AGENT | PLATFORM_ADMIN |
|-----------|----------|-------------|---------------|----------------|
| Register / Login | ✓ | ✓ | ✓ | ✓ |
| Book a court | ✓ | ✗ | ✗ | ✗ |
| Join open match | ✓ | ✗ | ✗ | ✗ |
| Create open match | ✓ | ✗ | ✗ | ✗ |
| Join waitlist | ✓ | ✗ | ✗ | ✗ |
| View/manage own bookings | ✓ | ✗ | ✗ | ✗ |
| Rate/review courts | ✓ | ✗ | ✗ | ✗ |
| Manage favorites/preferences | ✓ | ✗ | ✗ | ✗ |
| Create/edit/remove courts | ✗ | ✓ (own courts) | ✗ | ✗ |
| Configure availability/pricing | ✗ | ✓ (own courts) | ✗ | ✗ |
| Bulk court operations | ✗ | ✓ (own courts) | ✗ | ✗ |
| Create manual bookings | ✗ | ✓ (own courts, active subscription) | ✗ | ✗ |
| Confirm/reject pending bookings | ✗ | ✓ (own courts) | ✗ | ✗ |
| Bulk confirm/reject bookings | ✗ | ✓ (own courts) | ✗ | ✗ |
| View analytics/revenue | ✗ | ✓ (own courts) | ✗ | ✓ (platform-wide) |
| Export data (CSV/PDF) | ✗ | ✓ (own courts, rate-limited) | ✗ | ✓ (platform-wide) |
| Create court-level promo codes | ✗ | ✓ (own courts) | ✗ | ✗ |
| Create platform-wide promo codes | ✗ | ✗ | ✗ | ✓ |
| Manage personal settings | ✓ (own) | ✓ (own) | ✓ (own) | ✓ (own) |
| Configure reminder rules | ✗ | ✓ (own rules) | ✗ | ✗ |
| View own audit log | ✗ | ✓ (own actions) | ✗ | ✗ |
| View any audit log | ✗ | ✗ | ✗ | ✓ |
| Manage feature flags | ✗ | ✗ | ✗ | ✓ |
| Manage users (suspend/unsuspend) | ✗ | ✗ | ✗ | ✓ |
| Review verification requests | ✗ | ✗ | ✗ | ✓ |
| Handle escalated disputes | ✗ | ✗ | ✗ | ✓ |
| View system health/observability | ✗ | ✗ | ✗ | ✓ |
| Configure platform settings | ✗ | ✗ | ✗ | ✓ |
| Submit support tickets | ✓ | ✓ | ✗ | ✗ |
| View own support tickets | ✓ | ✓ | ✗ | ✗ |
| View/respond to all support tickets | ✗ | ✗ | ✓ | ✓ |
| View security alerts dashboard | ✗ | ✗ | ✓ (read-only) | ✓ |
| Manage security settings (IP blocks, thresholds) | ✗ | ✗ | ✗ | ✓ |

#### Acceptance Criteria

1. WHEN any API request is received, THE Platform_Service SHALL extract the `role` claim from the JWT_Access_Token and enforce the Authorization_Matrix before processing the request
2. WHEN any API request is received, THE Transaction_Service SHALL extract the `role` claim from the JWT_Access_Token and enforce the Authorization_Matrix before processing the request
3. WHEN a request is made with an insufficient role for the requested operation, THE Platform_Service or Transaction_Service SHALL return HTTP 403 Forbidden with error code `AUTHZ_INSUFFICIENT_ROLE`
4. WHEN a COURT_OWNER attempts an operation requiring active subscription (e.g., manual booking creation), THE Platform_Service SHALL check the `subscriptionStatus` claim and reject the request with error code `AUTHZ_SUBSCRIPTION_EXPIRED` if the status is `EXPIRED` (note: until Phase 10, `subscriptionStatus` is always `ACTIVE` for court owners)
5. WHEN a COURT_OWNER's courts are accessed for customer booking, THE Transaction_Service SHALL verify the `stripeConnected` status from the JWT or via internal API and reject customer bookings with error code `PAYMENT_STRIPE_NOT_CONNECTED` if Stripe Connect is not active. _Cross-service concern — Transaction Service enforcement deferred to Phase 4 (Booking & Payments). Phase 2 implements the JWT claim; Phase 4 implements the check._
6. THE PLATFORM_ADMIN role SHALL NOT be self-registerable; platform admin accounts SHALL be seeded in the database or created by an existing PLATFORM_ADMIN via the admin user management API
7. THE SUPPORT_AGENT role SHALL NOT be self-registerable; support agent accounts SHALL be created by a PLATFORM_ADMIN via the admin user management API

### Requirement 7: User Profile Management

**User Story:** As a registered user, I want to view and update my profile information, so that I can keep my account details current.

#### Acceptance Criteria

1. THE Platform_Service SHALL support user profile retrieval via `GET /api/users/me`, returning the authenticated user's name, email, phone, language, role, and role-specific fields (verified, stripeConnected, subscriptionStatus for COURT_OWNER)
2. THE Platform_Service SHALL support user profile updates via `PATCH /api/users/me` for the following fields: name, phone, and language (el or en)
3. WHEN a user updates their email address, THE Platform_Service SHALL verify the new email is not already associated with another account before applying the change. _Note: Email change confirmation (sending verification email to new address) requires notification infrastructure from Phase 5. Until Phase 5, email changes are applied immediately after uniqueness validation._
4. WHEN a user updates their profile, THE Platform_Service SHALL update the `users.updated_at` timestamp
5. THE Platform_Service SHALL validate that the language field is one of the supported values: `el` (Greek) or `en` (English)
6. WHEN a COURT_OWNER updates their profile, THE Platform_Service SHALL additionally support updates to business-related fields: `business_name`, `tax_id`, `business_type`, `business_address`, `vat_registered`, `vat_number`, `contact_phone`, `profile_image_url`
7. THE Platform_Service SHALL support notification preference updates via `PATCH /api/users/me/preferences`, allowing users to configure preferences for the following notification types: booking events, favorite alerts, and promotional notifications
8. THE Platform_Service SHALL allow users to configure notification channel preferences (email, push, in-app) independently for each notification type via `PATCH /api/users/me/preferences`
9. THE Platform_Service SHALL support do-not-disturb hours configuration (start time, end time) as part of notification preferences, during which non-critical notifications are suppressed
10. THE Platform_Service SHALL persist notification preferences in the `preferences` table (established in Phase 1b schema), keyed by user ID

### Requirement 8: OAuth Provider Linking and Unlinking

**User Story:** As a registered user, I want to link additional OAuth providers to my account or unlink existing ones, so that I have flexible login options and account recovery paths.

#### Acceptance Criteria

1. WHEN a user requests to link a new OAuth_Provider, THE Platform_Service SHALL initiate the OAuth flow with the selected provider and, upon success, create a new entry in the `oauth_providers` table associated with the user's account
2. WHEN a user attempts to link an OAuth_Provider that is already linked to a different account, THE Platform_Service SHALL reject the request and return an error indicating the provider account is already associated with another user
3. WHEN a user requests to unlink an OAuth_Provider, THE Platform_Service SHALL remove the corresponding entry from the `oauth_providers` table
4. WHEN a user attempts to unlink their last remaining OAuth_Provider, THE Platform_Service SHALL reject the request and return an error indicating at least one provider must remain linked for account access
5. WHERE a user has multiple OAuth_Providers linked, THE Platform_Service SHALL allow login through any linked provider

### Requirement 9: Concurrent Session Management

**User Story:** As a platform operator, I want to limit the number of concurrent active sessions per user, so that account sharing is discouraged and security is maintained.

#### Acceptance Criteria

1. THE Platform_Service SHALL limit concurrent active sessions to a configurable maximum (default: 5 devices), tracked via non-invalidated, non-expired entries in the `refresh_tokens` table grouped by `user_id`
2. WHEN a new login or token refresh would exceed the concurrent session limit, THE Platform_Service SHALL invalidate the oldest session's Refresh_Token (determined by the earliest `created_at` among active entries for that user)
3. THE Platform_Service SHALL expose a session list endpoint (`GET /api/auth/sessions`) returning the user's active sessions with `device_info`, `ip_address`, `last_used_at`, and `created_at` for each
4. WHEN a user requests to revoke a specific session via `DELETE /api/auth/sessions/{tokenId}`, THE Platform_Service SHALL invalidate the corresponding Refresh_Token

### Requirement 10: Rate Limiting on Auth Endpoints

**User Story:** As a platform operator, I want to rate-limit authentication endpoints, so that brute force attacks and credential stuffing are mitigated.

#### Failed Auth Attempts Table Schema

The `failed_auth_attempts` table tracks failed authentication attempts for IP-based lockout:

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `ip_address` | VARCHAR(45) | IPv4 or IPv6 address |
| `attempt_count` | INTEGER | Number of failed attempts in current window |
| `window_start` | TIMESTAMP | Start of the rolling time window |
| `locked_until` | TIMESTAMP | If set, IP is locked until this time |
| `created_at` | TIMESTAMP | Record creation timestamp |
| `updated_at` | TIMESTAMP | Last update timestamp |

**Rolling Window Behavior:**
- Each failed auth attempt increments `attempt_count` if within the current 15-minute window
- If `window_start` is older than 15 minutes, reset `attempt_count` to 1 and update `window_start` to current time
- When `attempt_count` exceeds threshold (20), set `locked_until` to 30 minutes from now

#### Acceptance Criteria

1. THE Platform_Service SHALL implement rate limiting on all authentication endpoints (`/api/auth/**`) using a sliding window algorithm backed by Redis
2. WHEN the rate limit is exceeded for an IP address, THE Platform_Service SHALL return HTTP 429 Too Many Requests with error code `RATE_LIMIT_EXCEEDED` and a `Retry-After` header indicating when the client may retry
3. THE Platform_Service SHALL apply the following default rate limits: 10 login attempts per IP per minute, 5 token refresh attempts per IP per minute, 3 registration attempts per IP per minute
4. THE Platform_Service SHALL track failed authentication attempts in the `failed_auth_attempts` table, incrementing `attempt_count` within a rolling time window
5. WHEN an IP address accumulates more than 20 failed authentication attempts within a 15-minute window, THE Platform_Service SHALL temporarily lock that IP by setting `failed_auth_attempts.locked_until` to 30 minutes from the current time, rejecting all auth attempts from that IP until the lock expires
6. WHEN Redis is unavailable, THE Platform_Service SHALL fall back to in-memory rate limiting per application instance (non-distributed) and log a warning. THE Platform_Service SHALL NOT fail-open (skip rate limiting entirely) when Redis is down

### Requirement 11: GDPR-Compliant Account Deletion

**User Story:** As a registered user, I want to delete my account and have my personal data anonymized, so that I can exercise my GDPR right to be forgotten while the platform retains anonymized records for analytics.

#### Acceptance Criteria

1. WHEN a user requests account deletion via `DELETE /api/users/me`, THE Platform_Service SHALL initiate the deletion process
2. WHEN account deletion is requested for a CUSTOMER, THE Platform_Service SHALL cancel all future bookings for that user (coordinating with Transaction_Service via Kafka `booking-events` or internal API) and process refunds according to the applicable cancellation policy. _Note: Booking cancellation coordination activates when Phase 4 (Booking & Payments) is deployed. Until then, the Platform_Service SHALL skip booking-related checks._
3. WHEN account deletion is requested for a COURT_OWNER, THE Platform_Service SHALL validate that all future bookings on the owner's courts are resolved (completed, cancelled, or refunded) before allowing deletion; IF unresolved bookings exist, THE Platform_Service SHALL reject the deletion request and return the list of unresolved booking IDs. _Note: Booking cancellation coordination activates when Phase 4 (Booking & Payments) is deployed. Until then, the Platform_Service SHALL skip booking-related checks._
4. WHEN account deletion is processed, THE Platform_Service SHALL anonymize the user's personal data: replace `users.name` with `Deleted User`, set `users.email` to a unique anonymized value (e.g., `deleted-{uuid}@anonymized.local`), clear `users.phone`, and set `users.status` to `DELETED`
5. WHEN account deletion is processed, THE Platform_Service SHALL delete all entries in `oauth_providers` and `refresh_tokens` for that user (cascaded via ON DELETE CASCADE)
6. WHEN a COURT_OWNER account is deleted, THE Platform_Service SHALL retain anonymized court and revenue data for platform analytics, setting court `visible` to FALSE and updating the `owner_id` reference to point to the anonymized user record
7. WHEN account deletion is processed, THE Platform_Service SHALL preserve anonymized historical records (bookings, reviews, payments) by retaining the anonymized user ID reference, ensuring analytics and audit trail integrity
8. WHEN account deletion is processed, THE Platform_Service SHALL publish a `USER_DELETED` event to the `notification-events` Kafka topic so that Transaction_Service can anonymize its cross-schema references and cancel pending operations. _Note: The `USER_DELETED` event is published for forward compatibility. Transaction Service consumer is implemented in Phase 5 (Real-time & Notifications). Until then, the event is published but not consumed._
9. WHEN account deletion is requested and no bookings exist for the user (pre-Phase 4), THE Platform_Service SHALL proceed directly to data anonymization without booking cancellation coordination

### Requirement 12: JWT Token Validation (Cross-Service)

**User Story:** As a platform developer, I want both services to validate JWT tokens consistently using the shared RS256 public key, so that authorization is enforced uniformly across the platform.

#### JWKS Endpoint Specification

The Platform Service SHALL expose the RS256 public key via a JWKS (JSON Web Key Set) endpoint:

- **Endpoint:** `GET /api/auth/.well-known/jwks.json`
- **Response Format:**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "<key-id>",
      "n": "<modulus-base64url>",
      "e": "<exponent-base64url>"
    }
  ]
}
```
- **Cache Headers:** `Cache-Control: public, max-age=86400` (24 hours)
- **Purpose:** Enables external services and Transaction Service to fetch the public key for JWT validation without direct configuration sharing

#### Shared JWT Validation Library

The JWT validation logic SHALL be implemented as a shared Spring Security filter in the `court-booking-common` library:

- **Package:** `gr.courtbooking.common.security`
- **Class:** `JwtAuthenticationFilter extends OncePerRequestFilter`
- **Responsibilities:**
  - Extract Bearer token from `Authorization` header
  - Validate RS256 signature using configured public key
  - Verify `exp` claim (reject expired tokens)
  - Verify required claims (`sub`, `role`, `email`, `iat`, `exp`)
  - Populate Spring Security context with authenticated principal
- **Configuration:** Public key loaded from environment variable `JWT_PUBLIC_KEY` or mounted Kubernetes secret
- **Error Responses:** Consistent error codes (`AUTH_INVALID_TOKEN`, `AUTH_TOKEN_EXPIRED`, `AUTH_MALFORMED_TOKEN`, `AUTH_TOKEN_MISSING`)

#### Service-to-Service Authentication

Internal HTTP calls between Platform Service and Transaction Service (within the Kubernetes cluster) SHALL use one of the following mechanisms:

- **With Istio (staging/production):** Mutual TLS (mTLS) via Istio sidecar proxies provides automatic service identity verification. No additional auth headers needed.
- **Without Istio (dev/test):** A shared internal API key passed via `X-Internal-Api-Key` header, rotated via the secrets management pipeline. Internal endpoints are not exposed through NGINX Ingress.

#### Acceptance Criteria

1. THE Platform_Service and Transaction_Service SHALL validate JWT_Access_Token signatures using the shared RS256 public key, loaded from configuration (environment variable or mounted secret)
2. WHEN a JWT_Access_Token has an invalid signature, THE service SHALL return HTTP 401 Unauthorized with error code `AUTH_INVALID_TOKEN`
3. WHEN a JWT_Access_Token has expired (current time > `exp` claim), THE service SHALL return HTTP 401 Unauthorized with error code `AUTH_TOKEN_EXPIRED`
4. WHEN a JWT_Access_Token is missing required claims (`sub`, `role`, `email`, `iat`, `exp`), THE service SHALL return HTTP 401 Unauthorized with error code `AUTH_MALFORMED_TOKEN`
5. WHEN a request is received without a JWT_Access_Token on a protected endpoint, THE service SHALL return HTTP 401 Unauthorized with error code `AUTH_TOKEN_MISSING`
6. THE Platform_Service SHALL expose the RS256 public key via a JWKS endpoint (`GET /api/auth/.well-known/jwks.json`) for key distribution, with appropriate cache headers
7. THE JWT validation logic SHALL be implemented as a shared Spring Security filter applicable to both services, packaged in the `court-booking-common` library

### Requirement 13: Subscription Stub Strategy (Pre-Phase 10)

**User Story:** As a platform developer, I want a clear subscription stub strategy for Phases 1-9, so that court owners can use the platform without subscription enforcement until Phase 10 implements real billing.

#### Stub Strategy Overview

Until Phase 10 (Subscription Billing) is implemented, the platform SHALL treat all court owners as having an active subscription. This allows full platform functionality during early adoption without the complexity of billing integration.

#### Database Behavior

- The `users.subscription_status` column SHALL default to `NONE` for all court owners at registration
- The `NONE` value indicates "no subscription decision made yet" (not expired, not active)
- No `trial_ends_at` timestamp is set or enforced until Phase 10

#### JWT Claim Mapping

- WHEN issuing a JWT for a COURT_OWNER, THE Platform_Service SHALL map `subscription_status = NONE` to `subscriptionStatus = ACTIVE` in the JWT claims
- This mapping ensures all court owners have full feature access without code changes when Phase 10 lands
- The JWT claim `subscriptionStatus` will contain one of: `TRIAL`, `ACTIVE`, `EXPIRED`, `NONE` — but until Phase 10, only `ACTIVE` is ever issued

#### Enforcement Behavior

- No subscription checks SHALL be enforced until Phase 10
- Manual bookings, court visibility, and all admin features SHALL work without restriction regardless of the underlying `subscription_status` value
- The authorization matrix check for "active subscription" on manual booking creation SHALL always pass (since JWT always contains `ACTIVE`)

#### UI Behavior

- Trial banners, trial countdown UI, subscription tier selection, and Stripe Billing integration SHALL NOT be implemented until Phase 10
- The "Trial Active" and "Trial Expired / No Subscription" court owner sub-states described in the glossary are effectively bypassed — all court owners behave as "Subscribed"

#### Phase 10 Migration

WHEN Phase 10 lands, the following changes will occur:
1. The hardcoded `NONE` → `ACTIVE` JWT mapping is replaced with real status mapping
2. New registrations set `subscription_status = TRIAL` with `trial_ends_at` = 30 days from registration
3. Stripe Billing webhooks update `subscription_status` based on payment events
4. Expired-state restrictions are enforced (read-only access, no new manual bookings)
5. Existing court owners with `NONE` status are migrated to `TRIAL` with a 30-day grace period

#### Acceptance Criteria

1. WHEN a COURT_OWNER registers, THE Platform_Service SHALL set `users.subscription_status` to `NONE` in the database
2. WHEN issuing a JWT for a COURT_OWNER with `subscription_status = NONE`, THE Platform_Service SHALL set the `subscriptionStatus` claim to `ACTIVE`
3. THE Platform_Service SHALL NOT enforce subscription status checks on any COURT_OWNER operations until Phase 10 is implemented
4. THE Platform_Service SHALL NOT display trial banners, subscription prompts, or billing UI until Phase 10 is implemented
5. WHEN a COURT_OWNER attempts to create a manual booking, THE Platform_Service SHALL allow the operation regardless of the underlying `subscription_status` value (since JWT contains `ACTIVE`)

### Requirement 14: Admin User Management

**User Story:** As a platform administrator, I want to create and manage SUPPORT_AGENT and PLATFORM_ADMIN accounts, suspend or unsuspend users, and view the user list, so that I can control platform access and respond to security or policy violations.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN sends a `POST /api/admin/users` request with role SUPPORT_AGENT, THE Platform_Service SHALL create a new SUPPORT_AGENT account linked to the specified OAuth provider
2. WHEN a PLATFORM_ADMIN sends a `POST /api/admin/users` request with role PLATFORM_ADMIN, THE Platform_Service SHALL create a new PLATFORM_ADMIN account linked to the specified OAuth provider
3. WHEN a non-PLATFORM_ADMIN user attempts to access any `/api/admin/users` endpoint, THE Platform_Service SHALL return HTTP 403 Forbidden with error code `AUTHZ_INSUFFICIENT_ROLE`
4. WHEN a PLATFORM_ADMIN sends a `PATCH /api/admin/users/{userId}/suspend` request, THE Platform_Service SHALL set the target user's status to SUSPENDED and record the suspension timestamp
5. WHEN a PLATFORM_ADMIN suspends a user account, THE Platform_Service SHALL immediately revoke ALL refresh tokens for that user by setting `refresh_tokens.invalidated` to TRUE for all entries, forcing logout on all devices
6. WHEN a suspended user's JWT_Access_Token is presented (still valid but account suspended), THE Platform_Service SHALL reject the request with HTTP 403 Forbidden and error code `ACCOUNT_SUSPENDED`. The JWT filter SHALL check account suspension status on each request using the Suspended_Users_Cache in Redis
7. WHEN a PLATFORM_ADMIN sends a `PATCH /api/admin/users/{userId}/unsuspend` request, THE Platform_Service SHALL set the target user's status to ACTIVE and remove the user from the Suspended_Users_Cache
8. WHEN a PLATFORM_ADMIN sends a `GET /api/admin/users` request, THE Platform_Service SHALL return a paginated user list supporting filters by role, status (ACTIVE, SUSPENDED, DELETED), and search by name or email
9. THE Platform_Service SHALL provide a database seed migration (`V2__seed_platform_admin.sql`) that creates an initial PLATFORM_ADMIN account with a pre-configured OAuth provider link, enabling first-time platform setup
10. THE seed migration SHALL be environment-specific, using different admin configurations for local, dev, staging, and production environments
11. THE seed migration SHALL be idempotent, safe to re-run without creating duplicate accounts or failing on subsequent executions
