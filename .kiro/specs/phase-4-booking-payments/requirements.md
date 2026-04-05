# Requirements Document — Phase 4: Booking & Payments

## Introduction

Phase 4 implements the complete booking and payment subsystem for the Court Booking Platform, targeting the `court-booking-transaction-service`. This phase delivers the customer booking flow (slot hold → payment → confirm), Stripe Connect onboarding for court owners, payment processing with PaymentIntent authorize/capture/refund, platform fee calculation with Stripe Connect transfers, manual booking creation for court owners, pending confirmation workflow (MANUAL mode) with auto-cancel timeout, recurring bookings (weekly pattern with Quartz scheduling), booking modifications and cancellations with tiered refund policy, no-show flagging, external payment tracking, bulk booking operations (confirm/reject), iCal export, Kafka publishing to the `booking-events` topic, Kafka consumption from the `court-update-events` topic, Stripe webhook handling, cross-schema views for court/user lookups, and idempotency key support for all state-changing operations.

This phase builds on Phase 3 (Court Management), which established court CRUD, availability windows/overrides, pricing rules, cancellation tiers, court owner verification, Stripe Connect status tracking, holiday calendar management, and Kafka `court-update-events` publishing. Phase 2 (Auth & User Management) established OAuth authentication, JWT token issuance with role-based claims (including `verified`, `stripeConnected`, `subscriptionStatus` for COURT_OWNER), and role-based access control. Phase 1b established the database schema (including `bookings`, `payments`, `audit_logs`, `notifications`, `device_tokens`, Quartz tables), Flyway migrations, cross-schema views (`v_court_summary`, `v_user_basic`, `v_court_cancellation_tiers`), and CI/CD pipelines.

**Master requirements coverage:** Req 8 (Atomic Booking Creation with Conflict Prevention — including recurring bookings), Req 9 (Manual Booking Creation by Court Owners — excluding subscription enforcement until Phase 10), Req 10 (Pending Booking Confirmation Workflow), Req 11 (Integrated Payment Processing — including payment method management, receipts, VAT display), Req 11a (Court Owner Payment Onboarding — Stripe Connect), Req 12 (Booking Modification and Cancellation — including tiered refund policy, default policy), Req 14 (Booking Management and History — including no-show tracking, external payment tracking).

**Subscription stub strategy:** Until Phase 10, all court owners default to `ACTIVE` subscription status. The `subscriptionStatus` JWT claim is included but subscription enforcement is deferred. Court owners can create manual bookings and accept customer bookings regardless of subscription status. When Phase 10 is implemented, the following enforcement will be added:
- WHEN a court owner's subscription lapses, THE Platform_Service SHALL restrict access to manual booking creation while preserving existing booking data and customer-facing court visibility
- WHEN the trial period expires, THE Platform_Service SHALL require the court owner to subscribe to a paid plan to continue using the admin interface and manual booking features

**Scope boundaries:**
- Court owner subscription billing (Req 9a) is ⏳ Phase 10. Until Phase 10, all court owners are treated as having an active subscription.
- Notification delivery (email, push, in-app, WebSocket) is Phase 5. Phase 4 publishes `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic for forward compatibility but they are not consumed until Phase 5.
- WebSocket real-time availability broadcasts are Phase 5. Phase 4 publishes booking events to the `booking-events` Kafka topic.
- Split payments (Req 25) are ⏳ Phase 10. The `splitPayment` field on `CreateBookingRequest` is accepted but ignored; `isSplitPayment` is always `false`.
- Open matches (Req 24) are ⏳ Phase 10. The `openMatch` field on `CreateBookingRequest` is accepted but ignored; `isOpenMatch` is always `false`.
- Waitlist (Req 26) is ⏳ Phase 10. The `waitlistEnabled` field is read from `v_court_summary` but no waitlist processing occurs.
- Promo codes (Req 27) are ⏳ Phase 10. The `promoCode` field on `CreateBookingRequest` is accepted but ignored; `discount_cents` is always `0`.
- Court ratings and reviews are ⏳ Phase 10.
- Security hardening (abuse detection, booking fraud detection, self-booking detection) is Phase 7.
- Analytics event publishing to `analytics-events` topic is Phase 6.

## Glossary

- **Transaction_Service**: The Spring Boot microservice responsible for bookings, payments, notifications, and scheduled jobs. Deployed to `court-booking-transaction-service`. Validates JWT tokens independently using the shared RS256 public key.
- **Platform_Service**: The Spring Boot microservice responsible for authentication, authorization, user management, courts, availability, and weather. Provides internal APIs and cross-schema views consumed by Transaction Service.
- **Booking**: A reservation of a court for a specific date and time slot. Stored in the `transaction.bookings` table. Has a lifecycle: PENDING_CONFIRMATION → CONFIRMED → COMPLETED, or CANCELLED/REJECTED at any stage.
- **Manual_Booking**: A booking created by a court owner through the admin interface without payment processing (walk-in, phone reservation). `booking_type = 'MANUAL'`, `payment_status = 'NOT_REQUIRED'` or `'PAID_EXTERNALLY'`.
- **Customer_Booking**: A booking created by a customer through the mobile app with payment processing. `booking_type = 'CUSTOMER'`.
- **Slot_Hold**: A temporary 5-minute Redis lock on a court time slot during the booking flow to prevent double-booking. Key format: `slot-hold:{courtId}:{date}:{startTime}`. Published as `SLOT_HELD` event.
- **Confirmation_Mode**: Court-level setting (`INSTANT` or `MANUAL`) that determines whether bookings are confirmed immediately after payment or require court owner approval. Snapshot from court at booking time into `bookings.confirmation_mode`.
- **Confirmation_Timeout**: Configurable time period (default 24 hours, from `courts.confirmation_timeout_hours`) for court owner to confirm pending bookings before auto-cancellation with full refund.
- **Recurring_Booking**: A set of weekly booking instances sharing a `recurring_group_id`. Created for up to 12 weeks. Each instance is an independent booking with its own payment.
- **Cancellation_Tier**: A time-based refund percentage rule from `v_court_cancellation_tiers`. Tiers are ordered by `threshold_hours` descending. The first tier where `hours_until_booking >= threshold_hours` determines the refund percentage.
- **Platform_Fee**: Percentage-based commission (default 10%) deducted from each customer booking payment. Stored as `platform_fee_cents` on both `bookings` and `payments` tables. Non-refundable on cancellation.
- **Stripe_Connect**: Stripe's marketplace payment platform enabling court owners to receive payouts directly to their bank accounts. Uses Express account type with hosted onboarding.
- **Stripe_Connect_Express**: A Stripe Connect account type where Stripe hosts the identity verification and bank account linking flow. Court owners are redirected to Stripe's hosted onboarding page.
- **PaymentIntent**: Stripe API object representing a payment. Created with `capture_method: 'manual'` for MANUAL confirmation mode (authorize-then-capture) or `capture_method: 'automatic'` for INSTANT mode.
- **Authorization_Hold**: Temporary hold on a customer's payment method for the booking amount, used in MANUAL confirmation mode. Expires after 7 days per Stripe's policy. Captured when court owner confirms, voided when rejected or timed out.
- **Destination_Charge**: Stripe Connect payment pattern where the platform creates a PaymentIntent with `transfer_data.destination` set to the court owner's connected account, automatically splitting funds at capture time.
- **Application_Fee_Amount**: The platform commission amount specified on a Stripe PaymentIntent via `application_fee_amount`, automatically retained by the platform when funds are transferred to the court owner's connected account.
- **Idempotency_Key**: Client-generated UUID included in the `X-Idempotency-Key` header for state-changing operations. Stored in `bookings.idempotency_key` (UNIQUE constraint). The server returns the same response for duplicate requests within 24 hours.
- **Euro_Cents**: Integer representation of EUR amounts (e.g., €25.50 = 2550) used throughout the API, database, and Stripe calls to avoid floating-point rounding issues.
- **No_Show**: A completed booking where the customer did not appear, manually flagged by the court owner within 24 hours after the booking's scheduled end time. Stored as `bookings.no_show = true`.
- **Paid_Externally**: Status flag on manual bookings indicating the court owner received offline payment (cash, bank transfer). Sets `bookings.payment_status = 'PAID_EXTERNALLY'` and `bookings.paid_externally = true`.
- **Booking_Window**: Configurable maximum number of days in advance a booking can be made. Default: 30 days. Stored in platform configuration.
- **Minimum_Notice**: Configurable minimum time before court start time that a booking can be created. Default: 1 hour. Stored in platform configuration.
- **Default_Cancellation_Policy**: Applied when a court owner has not configured a cancellation policy. Default tiers: 100% refund if cancelled 24+ hours before, 50% if 12-24 hours, 0% if less than 12 hours.
- **Audit_Log**: Immutable record of booking lifecycle events in the `transaction.audit_logs` table. Every status change, modification, or cancellation is recorded with actor, role, and details. Append-only.
- **Quartz_Scheduler**: Clustered job scheduler using JDBC-based job store (`org.quartz.jobStore.isClustered=true`) for reliable execution of timed operations across multiple Transaction Service pods.
- **Cross_Schema_View**: Read-only database view granting Transaction Service SELECT access to Platform Service schema data. Includes `v_court_summary`, `v_user_basic`, `v_court_cancellation_tiers`.
- **Kafka_Booking_Events**: The `booking-events` Kafka topic where Transaction Service publishes booking lifecycle events (BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, BOOKING_MODIFIED, BOOKING_COMPLETED, SLOT_HELD, SLOT_RELEASED). Partitioned by `courtId`.
- **Kafka_Notification_Events**: The `notification-events` Kafka topic where Transaction Service publishes `NOTIFICATION_REQUESTED` events for booking confirmations, cancellations, rejections, pending confirmation alerts, and payment events. Partitioned by `userId`. Not consumed until Phase 5.
- **Kafka_Court_Update_Events**: The `court-update-events` Kafka topic consumed by Transaction Service for pricing cache updates, availability constraint validation, cancellation policy sync, and Stripe Connect status changes.
- **Stripe_Webhook**: Server-to-server HTTP POST from Stripe to `POST /api/webhooks/stripe` for asynchronous payment event notifications. Secured via Stripe webhook signature verification with timestamp tolerance.
- **iCal_Export**: Calendar feed in iCalendar format (RFC 5545) for court owner booking integration with Google Calendar, Outlook, etc.
- **Bulk_Action**: Admin operation applied to multiple pending bookings simultaneously (confirm or reject). Returns `207 Multi-Status` with per-booking results.

## Requirements

### Requirement 1: Stripe Connect Onboarding for Court Owners

**User Story:** As a court owner, I want to complete Stripe Connect onboarding with identity verification and bank account linking, so that I can receive payouts from customer bookings.

#### Acceptance Criteria

1. WHEN a court owner initiates Stripe Connect onboarding via `POST /api/payments/stripe-connect/onboard`, THE Transaction_Service SHALL create a Stripe Connect Express account (if one does not already exist) and return a Stripe-hosted onboarding URL
   - **Input:** `{ returnUrl: string, refreshUrl: string }`
   - **Output (success):** `200 OK` with `{ onboardingUrl: string, stripeAccountId: string }`
   - **Output (already active):** `200 OK` with `{ status: "ACTIVE", stripeAccountId: string, message: "Stripe Connect already active" }`
   - **Output (not court owner):** `403 Forbidden` with `{ error: "AUTHZ_INSUFFICIENT_ROLE" }`
2. WHEN creating a Stripe Connect Express account, THE Transaction_Service SHALL use the court owner's email and country from `v_user_basic` and set `business_type` based on the court owner's verification data
3. WHEN the court owner completes Stripe's hosted onboarding flow and is redirected back to `returnUrl`, THE Transaction_Service SHALL provide a `GET /api/payments/stripe-connect/status` endpoint that returns the current onboarding status
   - **Output:** `{ status: enum(NOT_STARTED|PENDING|ACTIVE|RESTRICTED|DISABLED), stripeAccountId?: string, requiresAction: boolean, actionUrl?: string, payoutsEnabled: boolean, chargesEnabled: boolean }`
4. WHEN Stripe sends an `account.updated` webhook event, THE Transaction_Service SHALL update the court owner's Stripe Connect status by calling Platform Service's internal API `PUT /internal/users/{userId}/stripe-connect-status` with the new status
5. WHEN a court owner's Stripe Connect status transitions to `ACTIVE`, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic notifying the court owner that payment setup is complete
6. WHEN a court owner's Stripe Connect status transitions to `RESTRICTED`, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event with urgency `CRITICAL` notifying the court owner of required actions
7. IF a court owner attempts to create a customer booking endpoint and the court's owner does not have `stripe_connect_status = 'ACTIVE'` (checked via `v_user_basic`), THEN THE Transaction_Service SHALL return `422 Unprocessable Entity` with `{ error: "STRIPE_CONNECT_NOT_ACTIVE", message: "Court owner has not completed payment setup" }`
8. WHEN a court owner abandons the Stripe onboarding flow mid-way and returns later, THE Transaction_Service SHALL generate a new onboarding link for the existing Stripe Connect account (not create a duplicate account)
9. THE Transaction_Service SHALL store the Stripe Connect account ID mapping in the `v_user_basic` cross-schema view (Platform Service owns `users.stripe_connect_account_id`)
10. THE Transaction_Service SHALL provide a `GET /api/payments/stripe-connect/payouts` endpoint where court owners can view their payout schedule, payout history, balance, and Stripe account status — data fetched via Stripe Connect API on demand
    - **Output:** `{ payoutSchedule: { interval, weeklyAnchor?, monthlyAnchor? }, payouts: [{ id, amount, currency, status, arrivalDate, createdAt }], balance: { available: number, pending: number, currency: string }, accountStatus: string }`
11. THE Transaction_Service SHALL support configurable payout schedules (daily, weekly, monthly) through Stripe Connect settings. Court owners can update their payout schedule via `PUT /api/payments/stripe-connect/payout-schedule`
    - **Input:** `{ interval: enum(DAILY|WEEKLY|MONTHLY), weeklyAnchor?: enum(MONDAY|TUESDAY|...|SUNDAY), monthlyAnchor?: number(1-31) }`
    - **Output (success):** `200 OK` with updated payout schedule
    - **Output (invalid anchor):** `400 Bad Request` with `{ error: "INVALID_PAYOUT_ANCHOR" }`

### Requirement 2: Customer Booking Creation with Atomic Conflict Prevention

**User Story:** As a customer, I want to book a court for a specific date and time with guaranteed conflict prevention, so that my booking is confirmed without double-booking.

#### Acceptance Criteria

1. WHEN a customer submits a booking via `POST /api/bookings`, THE Transaction_Service SHALL atomically validate slot availability, acquire a 5-minute Redis slot hold, process payment, and create the booking
   - **Input:** `{ courtId: UUID, date: ISO8601-date, startTime: HH:mm, paymentMethodId: string, durationMinutes?: number, numberOfPeople?: number, promoCode?: string, openMatch?: boolean, splitPayment?: boolean }` with `X-Idempotency-Key: UUID` header
   - **Output (success, INSTANT mode):** `201 Created` with Booking object (`status: "CONFIRMED"`, `paymentStatus: "CAPTURED"`)
   - **Output (success, MANUAL mode):** `201 Created` with Booking object (`status: "PENDING_CONFIRMATION"`, `paymentStatus: "AUTHORIZED"`)
   - **Output (slot conflict):** `409 Conflict` with `{ error: "TIME_SLOT_UNAVAILABLE", conflictingSlot: { date, startTime }, alternativeSlots: [{ startTime, endTime, priceCents }] }` — up to 3 alternative available slots on the same date
   - **Output (payment failed):** `402 Payment Required` with `{ error: "PAYMENT_FAILED", message: string }`
   - **Output (validation error):** `400 Bad Request` with `{ errors: [{ field, message }] }`
   - **Output (booking window exceeded):** `422 Unprocessable Entity` with `{ error: "BOOKING_WINDOW_EXCEEDED", maxAdvanceDays: number, requestedDate: string }`
   - **Output (minimum notice not met):** `422 Unprocessable Entity` with `{ error: "MINIMUM_NOTICE_NOT_MET", minimumNoticeHours: number }`
   - **Output (rate limited):** `429 Too Many Requests` with `Retry-After` header
2. WHEN a booking request is received, THE Transaction_Service SHALL first attempt to acquire a Redis slot hold with key `slot-hold:{courtId}:{date}:{startTime}` and TTL of 5 minutes. IF the key already exists, THEN THE Transaction_Service SHALL return `409 Conflict`
3. WHEN a slot hold is acquired, THE Transaction_Service SHALL publish a `SLOT_HELD` event to the `booking-events` Kafka topic with `courtId`, `date`, `startTime`, `endTime`, and `holdExpiresAt`
4. WHEN a slot hold expires (5-minute TTL) without a booking being created, THE Transaction_Service SHALL publish a `SLOT_RELEASED` event to the `booking-events` Kafka topic with `releaseReason: "HOLD_EXPIRED"`
5. THE Transaction_Service SHALL validate slot availability against existing confirmed bookings in the `bookings` table using a `SELECT ... FOR UPDATE` row-level lock on the `(court_id, date)` index to prevent race conditions between concurrent booking attempts
6. THE Transaction_Service SHALL validate the booking against the court's availability windows and overrides by calling Platform Service's internal API `GET /internal/courts/{courtId}/validate-slot?date=&startTime=&endTime=`
7. THE Transaction_Service SHALL calculate the booking price by calling Platform Service's internal API `GET /internal/courts/{courtId}/calculate-price?date=&startTime=&endTime=` and store the result in `bookings.total_amount_cents`
8. THE Transaction_Service SHALL calculate the platform fee as 10% of `total_amount_cents` (rounded to nearest cent) and store it in `bookings.platform_fee_cents`. The court owner net amount (`total_amount_cents - platform_fee_cents`) SHALL be stored in `bookings.court_owner_net_cents`
9. WHEN the court's `confirmation_mode` is `INSTANT`, THE Transaction_Service SHALL create a Stripe PaymentIntent with `capture_method: 'automatic'`, `transfer_data.destination` set to the court owner's Stripe Connect account, and `application_fee_amount` set to the platform fee. The booking status SHALL be set to `CONFIRMED`
10. WHEN the court's `confirmation_mode` is `MANUAL`, THE Transaction_Service SHALL create a Stripe PaymentIntent with `capture_method: 'manual'` (authorize only), `transfer_data.destination` set to the court owner's Stripe Connect account, and `application_fee_amount` set to the platform fee. The booking status SHALL be set to `PENDING_CONFIRMATION`
11. THE Transaction_Service SHALL store the `idempotency_key` from the `X-Idempotency-Key` header in `bookings.idempotency_key`. IF a booking with the same idempotency key already exists, THEN THE Transaction_Service SHALL return the existing booking response with the original status code
12. WHEN a booking is created, THE Transaction_Service SHALL snapshot the court's `confirmation_mode` into `bookings.confirmation_mode` so that subsequent confirmation mode changes do not affect existing bookings
13. WHEN a booking is created, THE Transaction_Service SHALL record a `CREATED` entry in the `audit_logs` table with `performed_by` set to the customer's user ID and `performed_by_role` set to `CUSTOMER`
14. WHEN a booking is created, THE Transaction_Service SHALL publish a `BOOKING_CREATED` event to the `booking-events` Kafka topic with all required fields per the event contract
15. WHEN a booking is created, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic to notify the court owner of the new booking (with urgency `STANDARD`)
16. WHEN a booking is created with `confirmation_mode = 'MANUAL'`, THE Transaction_Service SHALL additionally publish a `NOTIFICATION_REQUESTED` event to notify the customer that the booking is pending confirmation with the estimated confirmation timeout
17. IF the `durationMinutes` parameter is not provided, THEN THE Transaction_Service SHALL use the court's default `duration_minutes` from `v_court_summary`
18. THE Transaction_Service SHALL validate that `numberOfPeople` does not exceed the court's `max_capacity` from `v_court_summary`. IF exceeded, THEN THE Transaction_Service SHALL return `400 Bad Request` with `{ error: "CAPACITY_EXCEEDED", maxCapacity: number }`
19. THE Transaction_Service SHALL compute `endTime` as `startTime + durationMinutes` and validate that the resulting time slot falls within the court's availability windows
20. THE Transaction_Service SHALL use the court's timezone (from `v_court_summary.timezone`) for all booking time calculations and date comparisons
21. WHEN a customer and court are in different timezones, THE Transaction_Service SHALL display times in the court's local timezone with the customer's timezone shown for reference in booking responses and notification events
22. IF payment processing fails after the slot hold is acquired, THEN THE Transaction_Service SHALL release the Redis slot hold and publish a `SLOT_RELEASED` event with `releaseReason: "PAYMENT_FAILED"`
23. THE Transaction_Service SHALL store the Stripe PaymentIntent ID in `bookings.stripe_payment_intent_id` and create a corresponding record in the `payments` table with all payment details
24. THE Transaction_Service SHALL enforce a configurable maximum advance booking window (default: 30 days). IF the requested booking date exceeds this window, THEN THE Transaction_Service SHALL return `422 Unprocessable Entity` with `{ error: "BOOKING_WINDOW_EXCEEDED", maxAdvanceDays: 30, requestedDate: ISO8601-date }`
25. THE Transaction_Service SHALL enforce a configurable minimum booking notice period (default: 1 hour before start time). IF the booking start time is less than the minimum notice period from the current time, THEN THE Transaction_Service SHALL return `422 Unprocessable Entity` with `{ error: "MINIMUM_NOTICE_NOT_MET", minimumNoticeHours: 1, bookingStartTime: ISO8601 }`


### Requirement 3: Manual Booking Creation by Court Owners

**User Story:** As a court owner, I want to create bookings manually for walk-in customers or phone reservations, so that I can manage all court usage in one system regardless of how the booking was made.

#### Acceptance Criteria

1. WHEN a court owner submits a manual booking via `POST /api/bookings/manual`, THE Transaction_Service SHALL create a booking without payment processing using the same conflict prevention as customer bookings
   - **Input:** `{ courtId: UUID, date: ISO8601-date, startTime: HH:mm, durationMinutes?: number, customerName?: string, customerPhone?: string, customerEmail?: string, notes?: string }`
   - **Output (success):** `201 Created` with Booking object (`status: "CONFIRMED"`, `bookingType: "MANUAL"`, `paymentStatus: "NOT_REQUIRED"`)
   - **Output (slot conflict):** `409 Conflict` with `{ error: "TIME_SLOT_UNAVAILABLE" }`
   - **Output (not court owner):** `403 Forbidden` with `{ error: "AUTHZ_INSUFFICIENT_ROLE" }`
   - **Output (not own court):** `403 Forbidden` with `{ error: "NOT_COURT_OWNER" }`
2. WHEN a manual booking is created, THE Transaction_Service SHALL validate that the authenticated court owner owns the specified court by checking `v_court_summary.owner_id`
3. WHEN a manual booking is created, THE Transaction_Service SHALL set `booking_type = 'MANUAL'`, `payment_status = 'NOT_REQUIRED'`, `total_amount_cents = NULL`, `platform_fee_cents = NULL`, and `court_owner_net_cents = NULL`
4. WHEN a manual booking is created, THE Transaction_Service SHALL apply the same slot conflict prevention (Redis hold + database check) as customer bookings to prevent double-booking
5. WHEN a manual booking is created, THE Transaction_Service SHALL always set `status = 'CONFIRMED'` regardless of the court's `confirmation_mode` (manual bookings skip the pending confirmation workflow)
6. WHEN a manual booking is created, THE Transaction_Service SHALL record a `CREATED` entry in the `audit_logs` table with `performed_by` set to the court owner's user ID and `performed_by_role` set to `COURT_OWNER`
7. WHEN a manual booking is created, THE Transaction_Service SHALL publish a `BOOKING_CREATED` event to the `booking-events` Kafka topic with `bookingType: "MANUAL"` and `totalAmountCents: null`
8. IF `durationMinutes` is not provided, THEN THE Transaction_Service SHALL use the court's default `duration_minutes` from `v_court_summary`
9. THE Transaction_Service SHALL store `customerName`, `customerPhone`, and `notes` in the corresponding `bookings` table columns for court owner reference
10. WHEN a court owner submits a recurring manual booking, THE Transaction_Service SHALL accept an optional `recurring` field in the `POST /api/bookings/manual` input: `{ recurring?: { frequencyWeeks: 1, durationWeeks: number(1-12) } }`. THE Transaction_Service SHALL create weekly booking instances for the specified duration (1-12 weeks), each as an independent manual booking sharing a `recurring_group_id`
11. WHEN creating recurring manual bookings, THE Transaction_Service SHALL allow partial creation — if some dates have conflicts (existing bookings or availability overrides), the non-conflicting dates SHALL be booked and the response SHALL be `207 Multi-Status` with per-instance results including `conflictingDates`
12. WHEN a recurring manual booking instance is cancelled, THE Transaction_Service SHALL only cancel that specific instance, not the entire group (each instance is independently cancellable)

### Requirement 4: Pending Booking Confirmation Workflow

**User Story:** As a court owner using manual confirmation mode, I want to review and confirm or reject pending bookings within a configurable timeout, so that I can control which bookings are accepted for my courts.

#### Acceptance Criteria

1. WHEN a booking is created for a court with `confirmation_mode = 'MANUAL'`, THE Transaction_Service SHALL set the booking status to `PENDING_CONFIRMATION` and the payment status to `AUTHORIZED` (payment held but not captured)
2. WHEN a court owner confirms a pending booking via `POST /api/bookings/{bookingId}/confirm`, THE Transaction_Service SHALL capture the authorized Stripe PaymentIntent, update the booking status to `CONFIRMED`, and set `confirmed_at` to the current timestamp
   - **Input:** `{ message?: string }` — optional message sent to the customer in the confirmation notification
   - **Output (success):** `200 OK` with updated Booking object (`status: "CONFIRMED"`, `paymentStatus: "CAPTURED"`)
   - **Output (not pending):** `400 Bad Request` with `{ error: "BOOKING_NOT_PENDING", currentStatus: string }`
   - **Output (not court owner):** `403 Forbidden`
   - **Output (Stripe Connect not active):** `422 Unprocessable Entity` with `{ error: "STRIPE_CONNECT_NOT_ACTIVE" }`
3. WHEN a court owner confirms a booking, THE Transaction_Service SHALL verify the court owner's Stripe Connect status is `ACTIVE` by calling Platform Service's internal API `GET /internal/users/{userId}/stripe-connect-status` before capturing the payment
4. WHEN a court owner rejects a pending booking via `POST /api/bookings/{bookingId}/reject`, THE Transaction_Service SHALL void the authorized Stripe PaymentIntent (full refund of the authorization hold), update the booking status to `REJECTED`, and record the rejection reason
   - **Input:** `{ reason?: string }`
   - **Output (success):** `200 OK` with CancellationResult (`refundAmount: full amount`, `refundPercentage: 100`)
   - **Output (not pending):** `400 Bad Request` with `{ error: "BOOKING_NOT_PENDING" }`
5. WHEN a pending booking is not confirmed or rejected within the court's `confirmation_timeout_hours`, THE Transaction_Service SHALL auto-cancel the booking via a Quartz scheduled job, void the Stripe authorization, set `cancelled_by = 'SYSTEM'`, and set `cancellation_reason = 'CONFIRMATION_TIMEOUT'`
6. WHEN a pending booking is auto-cancelled due to timeout, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer of the cancellation with urgency `CRITICAL` and a `NOTIFICATION_REQUESTED` event to notify the court owner that the booking was auto-cancelled
7. WHEN a booking is confirmed, THE Transaction_Service SHALL record a `CONFIRMED` entry in the `audit_logs` table with `performed_by` set to the court owner's user ID
8. WHEN a booking is rejected, THE Transaction_Service SHALL record a `REJECTED` entry in the `audit_logs` table with the rejection reason in the `details` JSONB field
9. WHEN a booking is confirmed, THE Transaction_Service SHALL publish a `BOOKING_CONFIRMED` event to the `booking-events` Kafka topic with `capturedAmountCents`, `platformFeeCents`, and `courtOwnerNetCents`
10. WHEN a booking is rejected or auto-cancelled, THE Transaction_Service SHALL publish a `BOOKING_CANCELLED` event to the `booking-events` Kafka topic with `cancelledBy` set to `COURT_OWNER` or `SYSTEM` respectively
11. WHEN a court owner retrieves pending bookings via `GET /api/bookings/pending`, THE Transaction_Service SHALL return all bookings with `status = 'PENDING_CONFIRMATION'` for courts owned by the authenticated court owner, sorted by creation date ascending
    - **Query params:** `courtId?: UUID, page: number (default 0), size: number (default 20)`
    - **Output:** BookingListResponse with pagination
12. WHEN a pending booking is confirmed, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer that the booking is confirmed with urgency `STANDARD`
13. WHEN a pending booking is rejected, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer that the booking was rejected with the rejection reason and urgency `CRITICAL`
14. THE Transaction_Service SHALL schedule the confirmation timeout job via Quartz at booking creation time with a fire time of `booking.created_at + court.confirmation_timeout_hours`. The job SHALL be idempotent — if the booking is already confirmed or cancelled when the job fires, it SHALL be a no-op
15. WHEN a Stripe authorization hold is approaching expiration (24 hours before the 7-day Stripe hold window expires), THE Transaction_Service SHALL auto-cancel the pending booking, void the authorization hold, set `cancelled_by = 'SYSTEM'` and `cancellation_reason = 'AUTHORIZATION_HOLD_EXPIRING'`, and publish `NOTIFICATION_REQUESTED` events to notify both the customer and court owner with urgency `CRITICAL`
16. WHEN a court owner configures `confirmation_timeout_hours`, THE Transaction_Service SHALL validate that the timeout value is at most `(7 * 24) - 24 = 144 hours` (Stripe's 7-day authorization hold window minus a 24-hour safety buffer). IF the configured timeout exceeds this limit, THEN THE Transaction_Service SHALL return `400 Bad Request` with `{ error: "CONFIRMATION_TIMEOUT_EXCEEDS_STRIPE_HOLD", maxTimeoutHours: 144 }`
17. THE Transaction_Service SHALL implement a Quartz scheduled job for pending confirmation reminders that sends `NOTIFICATION_REQUESTED` events to the court owner at configured intervals (default: 1 hour, 4 hours, and 12 hours after booking creation) for bookings still in `PENDING_CONFIRMATION` status
18. WHEN a court owner changes confirmation mode from `INSTANT` to `MANUAL` via Platform Service, THE Transaction_Service SHALL apply the change only to future bookings — existing confirmed bookings are not affected, and the snapshotted `confirmation_mode` in existing bookings remains unchanged


### Requirement 5: Payment Processing with Stripe

**User Story:** As a customer, I want to pay for court bookings using my saved payment methods (card, Apple Pay, Google Pay) with secure processing, so that my payment is handled reliably and the court owner receives their share.

#### Acceptance Criteria

1. WHEN a customer booking is created, THE Transaction_Service SHALL create a Stripe PaymentIntent with `currency: 'eur'`, `amount` set to `total_amount_cents`, `transfer_data.destination` set to the court owner's `stripe_connect_account_id`, and `application_fee_amount` set to `platform_fee_cents`
2. WHEN the court's `confirmation_mode` is `INSTANT`, THE Transaction_Service SHALL create the PaymentIntent with `capture_method: 'automatic'` so that payment is captured immediately upon successful authorization
3. WHEN the court's `confirmation_mode` is `MANUAL`, THE Transaction_Service SHALL create the PaymentIntent with `capture_method: 'manual'` so that payment is authorized but not captured until the court owner confirms
4. THE Transaction_Service SHALL create a record in the `payments` table with `booking_id`, `user_id`, `amount_cents`, `platform_fee_cents`, `court_owner_net_cents`, `currency`, `status`, `stripe_payment_intent_id`, and `payment_method_type`
5. WHEN a customer requests saved payment methods via `GET /api/payments/methods`, THE Transaction_Service SHALL retrieve the customer's payment methods from Stripe using the customer's `stripe_customer_id` from `v_user_basic`
   - **Output:** `{ methods: [{ id, type: enum(CARD|APPLE_PAY|GOOGLE_PAY), last4?, brand?, expiryMonth?, expiryYear?, isDefault }] }`
6. WHEN a customer adds a payment method via `POST /api/payments/methods`, THE Transaction_Service SHALL attach the payment method to the Stripe Customer object
   - **Input:** `{ setupIntentId: string }`
   - **Output:** `201 Created` with PaymentMethod object
7. WHEN a customer requests a Stripe SetupIntent via `POST /api/payments/setup-intent`, THE Transaction_Service SHALL create a Stripe SetupIntent for the customer and return the `clientSecret` for frontend SDK use
   - **Output:** `{ clientSecret: string }`
8. WHEN Stripe sends a `payment_intent.succeeded` webhook event, THE Transaction_Service SHALL update the `payments.status` to `CAPTURED` and the `bookings.payment_status` to `CAPTURED`
9. WHEN Stripe sends a `payment_intent.payment_failed` webhook event, THE Transaction_Service SHALL update the `payments.status` to `FAILED`, the `bookings.payment_status` to `FAILED`, cancel the booking, release the slot hold, and publish a `NOTIFICATION_REQUESTED` event to notify the customer with urgency `CRITICAL`
10. WHEN Stripe sends a `charge.dispute.created` webhook event, THE Transaction_Service SHALL update the `payments.status` to `DISPUTED` and publish a `NOTIFICATION_REQUESTED` event to notify the court owner with urgency `CRITICAL`
11. THE Transaction_Service SHALL verify Stripe webhook signatures using the webhook signing secret and reject requests with invalid signatures or timestamps older than 5 minutes
12. THE Transaction_Service SHALL process webhook events idempotently by checking the Stripe event ID against a processed events cache (Redis with 24-hour TTL) to prevent duplicate processing
13. WHEN a customer requests a booking receipt via `GET /api/bookings/{bookingId}/receipt`, THE Transaction_Service SHALL return a digital receipt with booking details, payment amount, platform fee, transaction ID, and court information
    - **Output (JSON):** Receipt object with `bookingId`, `transactionId`, `courtName`, `date`, `startTime`, `endTime`, `amount`, `platformFee`, `discount`, `totalCharged`, `currency`, `paymentMethod`, `issuedAt`
    - **Output (PDF):** `application/pdf` binary when `Accept: application/pdf` header is present
14. WHEN a refund is initiated, THE Transaction_Service SHALL create a Stripe Refund on the PaymentIntent, update `payments.refund_amount_cents`, `payments.refund_reason`, `payments.refunded_at`, and set `payments.status` to `REFUNDED` or `PARTIALLY_REFUNDED`
15. WHEN a customer requests refund status via `GET /api/payments/{paymentId}/refund`, THE Transaction_Service SHALL return the refund status with `initiatedAt` and `completedAt` timestamps
16. WHEN a customer requests dispute details via `GET /api/payments/{paymentId}/dispute`, THE Transaction_Service SHALL return dispute information including `reason`, `status` (OPEN, UNDER_REVIEW, WON, LOST), and `createdAt`
17. WHEN a Stripe Customer object does not exist for a user, THE Transaction_Service SHALL create one on the user's first payment and store the `stripeCustomerId` in the platform schema `users` table via Platform Service's internal API `PUT /internal/users/{userId}/stripe-customer-id`
18. WHEN a customer's saved payment method expires or is removed between booking creation and pending confirmation capture, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer to update their payment method within 24 hours. IF the payment method is not updated within 24 hours, THEN THE Transaction_Service SHALL auto-cancel the pending booking, void the authorization hold, and publish cancellation notifications to both the customer and court owner
19. IF the court owner is VAT-registered (`vat_registered = true` from `v_user_basic`), THE receipt generated by `GET /api/bookings/{bookingId}/receipt` SHALL additionally display the court owner's VAT number and a "VAT included" line calculated as `gross_amount - (gross_amount / 1.24)` (Greek standard 24% VAT rate)
20. THE Transaction_Service SHALL provide an API for the court owner to submit dispute evidence via `POST /api/payments/{paymentId}/dispute/evidence`
    - **Input:** `{ bookingConfirmation?: string, courtUsageLogs?: string, additionalNotes?: string }`
    - **Output (success):** `200 OK` with `{ paymentId, evidenceSubmittedAt: ISO8601 }`
    - **Output (no active dispute):** `422 Unprocessable Entity` with `{ error: "NO_ACTIVE_DISPUTE" }`
21. THE Transaction_Service SHALL support Apple Pay and Google Pay as payment methods through Stripe's Payment Request Button API. The mobile app SHALL use Stripe's native SDK to present Apple Pay / Google Pay options when available on the device
22. WHEN a customer saves a payment method during checkout, THE Transaction_Service SHALL create a Stripe SetupIntent to securely store the card on the Stripe Customer object for future use. The saved method SHALL appear in `GET /api/payments/methods` responses

### Requirement 6: Platform Fee Calculation and Stripe Connect Transfers

**User Story:** As the platform operator, I want to automatically calculate and retain a commission on each booking payment, so that the platform generates revenue while court owners receive their net share.

#### Acceptance Criteria

1. THE Transaction_Service SHALL calculate the platform fee as 10% of `total_amount_cents` for every customer booking, rounded to the nearest euro cent using half-up rounding
2. THE Transaction_Service SHALL store the platform fee in `bookings.platform_fee_cents` and `payments.platform_fee_cents`
3. THE Transaction_Service SHALL calculate the court owner net amount as `total_amount_cents - platform_fee_cents` and store it in `bookings.court_owner_net_cents` and `payments.court_owner_net_cents`
4. THE Transaction_Service SHALL use Stripe's `application_fee_amount` parameter on the PaymentIntent to automatically retain the platform fee when funds are transferred to the court owner's connected account
5. WHEN a full refund is issued (court owner cancellation, rejection, or timeout), THE Transaction_Service SHALL refund the full `total_amount_cents` to the customer. The platform fee is also refunded (Stripe reverses the application fee automatically)
6. WHEN a partial refund is issued (customer cancellation with tiered policy), THE Transaction_Service SHALL calculate the refund as `(total_amount_cents - platform_fee_cents) * refund_percentage / 100`. The platform fee is non-refundable on customer-initiated cancellations
7. FOR ALL bookings, THE Transaction_Service SHALL ensure that `platform_fee_cents + court_owner_net_cents = total_amount_cents` (invariant)


### Requirement 7: Booking Modification

**User Story:** As a customer, I want to change the date or time of my booking with automatic price adjustment, so that I can reschedule without cancelling and rebooking.

#### Acceptance Criteria

1. WHEN a customer modifies a booking via `PUT /api/bookings/{bookingId}/modify`, THE Transaction_Service SHALL atomically validate the new slot, adjust payment, and update the booking
   - **Input:** `{ newDate: ISO8601-date, newStartTime: HH:mm }`
   - **Output (success, no price change):** `200 OK` with BookingModificationResult (`priceDifferenceCents: 0`, `additionalPaymentRequired: false`)
   - **Output (success, price increase):** `200 OK` with BookingModificationResult (`priceDifferenceCents: positive`, `additionalPaymentRequired: true`, `paymentIntentClientSecret: string`)
   - **Output (success, price decrease):** `200 OK` with BookingModificationResult (`priceDifferenceCents: negative`, `refundAmountCents: number`, `refundStatus: "PROCESSING"`)
   - **Output (new slot conflict):** `409 Conflict` with BookingConflictResponse including alternative slots
   - **Output (modification not allowed):** `422 Unprocessable Entity` with `{ error: "MODIFICATION_NOT_ALLOWED", reason: string }`
2. THE Transaction_Service SHALL only allow modification of bookings with `status = 'CONFIRMED'` and `booking_type = 'CUSTOMER'`. Bookings with `status = 'PENDING_CONFIRMATION'`, `'CANCELLED'`, `'COMPLETED'`, or `'REJECTED'` SHALL NOT be modifiable
3. THE Transaction_Service SHALL validate the new time slot against availability windows, overrides, and existing bookings using the same validation as booking creation
4. WHEN the new slot has a different price than the original, THE Transaction_Service SHALL calculate the price difference by calling Platform Service's internal API for the new slot price
5. WHEN the new slot is more expensive, THE Transaction_Service SHALL create a new Stripe PaymentIntent for the difference amount and return the `paymentIntentClientSecret` for the customer to complete the additional payment
6. WHEN the new slot is cheaper, THE Transaction_Service SHALL initiate a partial refund for the difference amount via Stripe
7. WHEN a booking is modified, THE Transaction_Service SHALL update `bookings.date`, `bookings.start_time`, `bookings.end_time`, `bookings.total_amount_cents` (if changed), and `bookings.updated_at`
8. WHEN a booking is modified, THE Transaction_Service SHALL record a `MODIFIED` entry in the `audit_logs` table with before/after date, time, and price in the `details` JSONB field
9. WHEN a booking is modified, THE Transaction_Service SHALL publish a `BOOKING_MODIFIED` event to the `booking-events` Kafka topic with previous and new date/time and `priceDifferenceCents`
10. WHEN a booking is modified, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the court owner of the modification with the old and new time slot details

### Requirement 8: Booking Cancellation with Tiered Refund Policy

**User Story:** As a customer or court owner, I want to cancel a booking with a refund amount determined by the court's cancellation policy, so that cancellations are handled fairly based on how far in advance they occur.

#### Acceptance Criteria

1. WHEN a customer cancels a booking via `POST /api/bookings/{bookingId}/cancel`, THE Transaction_Service SHALL calculate the refund amount based on the court's cancellation tiers from `v_court_cancellation_tiers` and the time remaining until the booking start
   - **Input:** `{ reason?: string }` with `X-Idempotency-Key: UUID` header
   - **Output:** `200 OK` with CancellationResult (`refundAmount`, `refundPercentage`, `policyTierApplied`, `platformFeeRetained`, `refundStatus`)
2. THE Transaction_Service SHALL determine the applicable cancellation tier by calculating the hours between the current time (in the court's timezone) and the booking start time, then selecting the first tier where `hours_until_booking >= threshold_hours` (tiers ordered by `threshold_hours` descending)
3. WHEN a customer cancels and a cancellation tier applies, THE Transaction_Service SHALL calculate the refund as `court_owner_net_cents * refund_percentage / 100`. The platform fee (`platform_fee_cents`) is non-refundable on customer-initiated cancellations
4. WHEN a court owner cancels a booking via `POST /api/bookings/{bookingId}/cancel`, THE Transaction_Service SHALL issue a full refund of `total_amount_cents` (including platform fee) regardless of cancellation tiers. **Note:** Stripe automatically reverses the `application_fee_amount` when a full refund is issued on a destination charge, so the customer receives the full amount back. This behavior differs from the master requirement (Req 12.8) which states platform commission is always non-refundable — the system design specifies full refund including platform fee for court-owner-initiated cancellations
5. WHEN a booking is cancelled, THE Transaction_Service SHALL update `bookings.status` to `CANCELLED`, set `cancelled_by` to `CUSTOMER`, `COURT_OWNER`, or `SYSTEM`, set `cancellation_reason`, `cancelled_at`, and `refund_amount_cents`
6. WHEN a booking is cancelled, THE Transaction_Service SHALL initiate the Stripe refund (full or partial based on the calculated amount) and update the `payments` table accordingly
7. WHEN a booking is cancelled, THE Transaction_Service SHALL record a `CANCELLED` entry in the `audit_logs` table with the cancellation reason, refund amount, and applied tier in the `details` JSONB field
8. WHEN a booking is cancelled, THE Transaction_Service SHALL publish a `BOOKING_CANCELLED` event to the `booking-events` Kafka topic with `cancelledBy`, `cancellationReason`, `refundAmountCents`, `refundPercentage`, and `waitlistEnabled`
9. WHEN a booking is cancelled, THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events to notify both the customer (with refund details) and the court owner (with cancellation details)
10. THE Transaction_Service SHALL only allow cancellation of bookings with `status` in (`CONFIRMED`, `PENDING_CONFIRMATION`). Bookings with `status = 'CANCELLED'`, `'COMPLETED'`, or `'REJECTED'` SHALL NOT be cancellable
11. WHEN a `PENDING_CONFIRMATION` booking is cancelled by the customer, THE Transaction_Service SHALL void the Stripe authorization hold (full refund) regardless of cancellation tiers
12. IF no cancellation tiers are configured for the court, THEN THE Transaction_Service SHALL apply a default policy: 100% refund if cancelled 24+ hours before booking start, 50% refund if cancelled 12-24 hours before, 0% refund if cancelled less than 12 hours before. The platform fee remains non-refundable in all cases for customer-initiated cancellations
13. THE Transaction_Service SHALL process cancellation requests idempotently using the `X-Idempotency-Key` header. Duplicate cancellation requests SHALL return the original cancellation result


### Requirement 9: Recurring Booking Creation

**User Story:** As a customer, I want to create weekly recurring bookings for up to 12 weeks, so that I can secure a regular playing schedule without rebooking each week.

#### Acceptance Criteria

1. WHEN a customer submits a recurring booking via `POST /api/bookings/recurring`, THE Transaction_Service SHALL create individual booking instances for each week, validating availability for all dates
   - **Input:** `{ courtId: UUID, startDate: ISO8601-date, startTime: HH:mm, paymentMethodId: string, weeks: number (2-12), numberOfPeople?: number, promoCode?: string }`
   - **Output (success):** `201 Created` with RecurringBookingResult (`recurringGroupId`, `createdBookings`, `conflictingDates`, `totalCreated`, `totalConflicts`)
   - **Output (all dates conflict):** `409 Conflict` with `{ error: "ALL_DATES_UNAVAILABLE" }`
2. THE Transaction_Service SHALL generate a `recurring_group_id` (UUID) and assign it to all booking instances in the group
3. THE Transaction_Service SHALL allow partial creation — if some dates have conflicts (existing bookings or availability overrides), the non-conflicting dates SHALL be booked and the conflicting dates SHALL be returned in `conflictingDates`
4. THE Transaction_Service SHALL process each recurring booking instance with its own Stripe PaymentIntent (each week is charged separately)
5. THE Transaction_Service SHALL calculate the price for each instance independently by calling Platform Service's pricing API, as pricing rules may vary by day of week
6. THE Transaction_Service SHALL store the recurring pattern in `bookings.recurring_pattern` as JSONB: `{ dayOfWeek, startTime, endTime, startDate, endDate, weeksAhead }`
7. WHEN a customer retrieves a recurring booking group via `GET /api/bookings/recurring/{recurringGroupId}`, THE Transaction_Service SHALL return all instances in the group with their individual statuses
   - **Output:** RecurringBookingGroup (`recurringGroupId`, `instances: [Booking]`, `totalInstances`)
8. THE Transaction_Service SHALL validate that `weeks` is between 2 and 12 inclusive
9. THE Transaction_Service SHALL validate that `startDate` falls on the correct day of week for the recurring pattern
10. WHEN a recurring booking instance is cancelled, THE Transaction_Service SHALL only cancel that specific instance, not the entire group
11. THE Transaction_Service SHALL publish a `BOOKING_CREATED` event for each successfully created recurring instance with `recurringGroupId` set

### Requirement 10: Recurring Booking Advance Scheduling

**User Story:** As a platform operator, I want recurring bookings to be automatically extended into future weeks via scheduled jobs, so that customers maintain their regular booking slots without manual intervention.

#### Acceptance Criteria

1. THE Transaction_Service SHALL implement a Quartz scheduled job that runs weekly to create new booking instances for active recurring booking groups
2. WHEN the advance scheduling job runs, THE Transaction_Service SHALL identify recurring groups where the latest instance date is within the configured advance window (default: 4 weeks ahead) and create new instances to maintain the window
3. WHEN a new recurring instance is created by the scheduled job, THE Transaction_Service SHALL validate slot availability and skip dates with conflicts (logging the skip reason)
4. WHEN a new recurring instance is created by the scheduled job and the court's pricing has changed since the original booking, THE Transaction_Service SHALL use the current pricing and publish a `NOTIFICATION_REQUESTED` event to notify the customer of the price change
5. THE Transaction_Service SHALL use Quartz's clustered mode to ensure only one pod executes the advance scheduling job at a time
6. WHEN the advance scheduling job encounters a payment failure for a recurring instance, THE Transaction_Service SHALL skip that instance, publish a `NOTIFICATION_REQUESTED` event to notify the customer, and continue processing remaining instances


### Requirement 11: No-Show Flagging

**User Story:** As a court owner, I want to flag bookings where the customer did not show up, so that I can track no-show patterns for my courts.

#### Acceptance Criteria

1. WHEN a court owner flags a booking as no-show via `POST /api/bookings/{bookingId}/no-show`, THE Transaction_Service SHALL update `bookings.no_show = true` and `bookings.no_show_flagged_at` to the current timestamp
   - **Input:** `{ notes?: string }`
   - **Output (success):** `200 OK` with `{ bookingId, noShow: true, flaggedAt: ISO8601 }`
   - **Output (window expired):** `422 Unprocessable Entity` with `{ error: "NO_SHOW_WINDOW_EXPIRED", message: "No-show can only be flagged within 24 hours after booking end time" }`
   - **Output (wrong status):** `422 Unprocessable Entity` with `{ error: "BOOKING_NOT_COMPLETED", currentStatus: string }`
2. THE Transaction_Service SHALL only allow no-show flagging for bookings with `status = 'COMPLETED'`
3. THE Transaction_Service SHALL only allow no-show flagging within 24 hours after the booking's scheduled end time (calculated in the court's timezone)
4. THE Transaction_Service SHALL validate that the authenticated court owner owns the court associated with the booking
5. WHEN a booking is flagged as no-show, THE Transaction_Service SHALL record a `NO_SHOW_FLAGGED` entry in the `audit_logs` table with the court owner's notes
6. WHEN a booking is flagged as no-show, THE Transaction_Service SHALL publish a `BOOKING_COMPLETED` event to the `booking-events` Kafka topic with `noShow: true` (for analytics tracking)
7. THE Transaction_Service SHALL not apply any automatic financial penalty for no-shows (no-show tracking is informational only in Phase 4)

### Requirement 12: External Payment Tracking

**User Story:** As a court owner, I want to mark manual bookings as paid via offline methods (cash, bank transfer), so that I can track all payments in one system.

#### Acceptance Criteria

1. WHEN a court owner marks a manual booking as paid via `POST /api/bookings/{bookingId}/mark-paid`, THE Transaction_Service SHALL update `bookings.payment_status` to `PAID_EXTERNALLY`, set `bookings.paid_externally = true`, and store the payment method and notes
   - **Input:** `{ paymentMethod: string, notes?: string }` — `paymentMethod` values: `"CASH"`, `"BANK_TRANSFER"`, or free-text
   - **Output (success):** `200 OK` with `{ bookingId, paymentStatus: "PAID_EXTERNALLY", paidAt: ISO8601 }`
   - **Output (not manual booking):** `422 Unprocessable Entity` with `{ error: "NOT_MANUAL_BOOKING", message: "Only manual bookings can be marked as paid externally" }`
2. THE Transaction_Service SHALL only allow marking as paid for bookings with `booking_type = 'MANUAL'`
3. THE Transaction_Service SHALL validate that the authenticated court owner owns the court associated with the booking
4. WHEN a booking is marked as paid externally, THE Transaction_Service SHALL store `paymentMethod` in `bookings.external_payment_method` and `notes` in `bookings.external_payment_notes`
5. WHEN a booking is marked as paid externally, THE Transaction_Service SHALL record a `PAYMENT_MARKED_EXTERNAL` entry in the `audit_logs` table

### Requirement 13: Booking Listing and History

**User Story:** As a customer or court owner, I want to view my bookings with filtering and pagination, so that I can manage my upcoming and past reservations.

#### Acceptance Criteria

1. WHEN a customer retrieves bookings via `GET /api/bookings`, THE Transaction_Service SHALL return the customer's own bookings with pagination
   - **Query params:** `status?: enum(CONFIRMED|PENDING_CONFIRMATION|CANCELLED|COMPLETED), fromDate?: ISO8601-date, toDate?: ISO8601-date, courtId?: UUID, courtType?: enum, page: number (default 0), size: number (default 20)`
   - **Output:** BookingListResponse with `bookings`, `totalElements`, `page`, `size`
2. WHEN a court owner retrieves bookings via `GET /api/bookings`, THE Transaction_Service SHALL return bookings for all courts owned by the court owner (both customer and manual bookings in a unified view)
3. THE Transaction_Service SHALL enrich each booking in the list with court name, court type, and timezone from `v_court_summary`
4. WHEN a user retrieves booking details via `GET /api/bookings/{bookingId}`, THE Transaction_Service SHALL return the full booking detail including court info, payment status, cancellation policy (applicable tiers from `v_court_cancellation_tiers`), and audit trail
   - **Output:** BookingDetail with `courtAddress`, `courtLocationType`, `ownerName`, `paymentId`, `cancellationPolicy` (array of tiers with `thresholdHours` and `refundPercent`), `auditTrail`
   - **Output (not found or not authorized):** `404 Not Found`
5. THE Transaction_Service SHALL only return bookings that belong to the authenticated user (customer's own bookings) or to courts owned by the authenticated court owner. Requests for other users' bookings SHALL return `404 Not Found`
6. THE Transaction_Service SHALL support filtering by `status`, `fromDate`, `toDate`, `courtId`, and `courtType`
7. THE Transaction_Service SHALL sort bookings by `date` and `start_time` descending by default (most recent first)


### Requirement 14: Bulk Booking Operations

**User Story:** As a court owner, I want to confirm or reject multiple pending bookings in a single action, so that I can efficiently manage my booking queue.

#### Acceptance Criteria

1. WHEN a court owner submits a bulk action via `POST /api/bookings/bulk-action`, THE Transaction_Service SHALL process each booking independently and return per-booking results
   - **Input:** `{ bookingIds: [UUID], action: enum(CONFIRM|REJECT), message?: string, reason?: string }`
   - **Output:** `207 Multi-Status` with BulkActionResult (`results: [{ bookingId, status: enum(SUCCESS|FAILED), error? }]`, `successCount`, `failureCount`)
2. THE Transaction_Service SHALL validate that all specified bookings belong to courts owned by the authenticated court owner
3. THE Transaction_Service SHALL process each booking independently — a failure on one booking SHALL NOT prevent processing of other bookings in the batch
4. WHEN `action = 'CONFIRM'`, THE Transaction_Service SHALL apply the same confirmation logic as `POST /api/bookings/{bookingId}/confirm` for each booking (capture payment, update status, publish events)
5. WHEN `action = 'REJECT'`, THE Transaction_Service SHALL apply the same rejection logic as `POST /api/bookings/{bookingId}/reject` for each booking (void authorization, update status, publish events)
6. THE Transaction_Service SHALL publish individual `BOOKING_CONFIRMED` or `BOOKING_CANCELLED` events for each successfully processed booking
7. THE Transaction_Service SHALL publish individual `NOTIFICATION_REQUESTED` events for each successfully processed booking to notify the respective customers
8. THE Transaction_Service SHALL limit the batch size to a maximum of 50 bookings per request. Requests exceeding this limit SHALL return `400 Bad Request` with `{ error: "BATCH_SIZE_EXCEEDED", maxBatchSize: 50 }`

### Requirement 15: iCal Calendar Export

**User Story:** As a court owner, I want to export my booking calendar in iCal format, so that I can integrate bookings with Google Calendar, Outlook, or other calendar applications.

#### Acceptance Criteria

1. WHEN a court owner requests a calendar export via `GET /api/bookings/calendar/ical`, THE Transaction_Service SHALL generate an iCalendar (RFC 5545) feed of bookings for the court owner's courts
   - **Query params:** `courtId?: UUID, fromDate?: ISO8601-date, toDate?: ISO8601-date`
   - **Output:** `200 OK` with `Content-Type: text/calendar` containing VCALENDAR with VEVENT entries
2. THE Transaction_Service SHALL include the following fields in each VEVENT: `UID` (booking ID), `DTSTART` and `DTEND` (in the court's timezone), `SUMMARY` (court name + booking type), `DESCRIPTION` (customer name if available, booking status, payment status), `LOCATION` (court address from `v_court_summary`)
3. WHEN `courtId` is not provided, THE Transaction_Service SHALL include bookings for all courts owned by the authenticated court owner
4. THE Transaction_Service SHALL default `fromDate` to today and `toDate` to 30 days from today when not provided
5. THE Transaction_Service SHALL only include bookings with `status` in (`CONFIRMED`, `PENDING_CONFIRMATION`) in the calendar export (exclude cancelled and completed bookings)
6. THE Transaction_Service SHALL set the `VTIMEZONE` component to the court's timezone for correct calendar display

### Requirement 16: Booking Completion Lifecycle

**User Story:** As a platform operator, I want bookings to automatically transition to COMPLETED status after their scheduled end time, so that the booking lifecycle is properly tracked.

#### Acceptance Criteria

1. THE Transaction_Service SHALL implement a Quartz scheduled job that runs periodically (every 15 minutes) to transition `CONFIRMED` bookings past their scheduled end time to `COMPLETED` status
2. WHEN a booking transitions to `COMPLETED`, THE Transaction_Service SHALL record a `COMPLETED` entry in the `audit_logs` table with `performed_by_role = 'SYSTEM'`
3. WHEN a booking transitions to `COMPLETED`, THE Transaction_Service SHALL publish a `BOOKING_COMPLETED` event to the `booking-events` Kafka topic with `noShow: false`
4. THE Transaction_Service SHALL use the court's timezone (from `v_court_summary.timezone`) to determine whether a booking's end time has passed
5. THE Transaction_Service SHALL use Quartz's clustered mode to ensure only one pod executes the completion job at a time


### Requirement 17: Kafka Event Publishing (Booking Events)

**User Story:** As a platform operator, I want all booking lifecycle events published to Kafka, so that other services can react to booking changes (availability cache invalidation, real-time updates, analytics).

#### Acceptance Criteria

1. THE Transaction_Service SHALL publish events to the `booking-events` Kafka topic using `courtId` as the partition key to guarantee ordering per court
2. THE Transaction_Service SHALL wrap all events in the standard EventEnvelope with `eventId` (UUID), `eventType`, `source: "transaction-service"`, `timestamp` (ISO 8601 UTC), `traceId`, `spanId`, and `payload`
3. THE Transaction_Service SHALL publish `BOOKING_CREATED` events when a booking is created (customer or manual) with all required payload fields per the Kafka event contract
4. THE Transaction_Service SHALL publish `BOOKING_CONFIRMED` events when a pending booking is confirmed by the court owner with `capturedAmountCents`, `platformFeeCents`, and `courtOwnerNetCents`
5. THE Transaction_Service SHALL publish `BOOKING_CANCELLED` events when a booking is cancelled (by customer, court owner, or system) with `cancelledBy`, `cancellationReason`, `refundAmountCents`, `refundPercentage`, and `waitlistEnabled`
6. THE Transaction_Service SHALL publish `BOOKING_MODIFIED` events when a booking's time slot is changed with previous and new date/time and `priceDifferenceCents`
7. THE Transaction_Service SHALL publish `BOOKING_COMPLETED` events when a booking transitions to COMPLETED status with `noShow` flag and financial details
8. THE Transaction_Service SHALL publish `SLOT_HELD` events when a slot hold is acquired with `holdExpiresAt`
9. THE Transaction_Service SHALL publish `SLOT_RELEASED` events when a slot hold is released with `releaseReason` (HOLD_EXPIRED, PAYMENT_FAILED, USER_ABANDONED)
10. THE Transaction_Service SHALL propagate the W3C Trace Context `traceId` and OpenTelemetry `spanId` in all published events for end-to-end distributed tracing

### Requirement 18: Kafka Event Consumption (Court Update Events)

**User Story:** As a transaction service operator, I want to consume court configuration changes from Kafka, so that booking validation uses up-to-date court data.

#### Acceptance Criteria

1. THE Transaction_Service SHALL consume events from the `court-update-events` Kafka topic published by Platform Service
2. WHEN a `COURT_UPDATED` event is received, THE Transaction_Service SHALL update its local cache of court configuration (confirmation mode, duration, capacity, waitlist enabled)
3. WHEN a `PRICING_UPDATED` event is received, THE Transaction_Service SHALL update its local pricing cache for the affected court and recalculate prices for future unconfirmed recurring booking instances on or after the `effectiveFrom` date
4. WHEN a `PRICING_UPDATED` event triggers a price change for recurring booking instances, THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events to notify affected customers of the price change with old and new amounts
5. WHEN a `CANCELLATION_POLICY_UPDATED` event is received, THE Transaction_Service SHALL update its local cache of cancellation tiers for the affected court (used for future cancellation calculations)
6. WHEN a `COURT_DELETED` event is received, THE Transaction_Service SHALL remove the court from its local cache and prevent new bookings for that court
7. WHEN a `STRIPE_CONNECT_STATUS_CHANGED` event is received, THE Transaction_Service SHALL update its local cache of the court owner's Stripe Connect status. IF the new status is `RESTRICTED` or `DISABLED`, THEN THE Transaction_Service SHALL prevent new payment captures for that court owner's courts
8. THE Transaction_Service SHALL process consumed events idempotently using the `eventId` field to prevent duplicate processing
9. WHEN an `AVAILABILITY_UPDATED` event is received with `changeType: "OVERRIDE_CREATED"` for a date that has existing confirmed bookings, THE Transaction_Service SHALL auto-cancel the affected individual bookings that conflict with the new availability override, process a refund per the cancellation policy (court-owner-initiated = full refund of `total_amount_cents`), set `cancelled_by = 'SYSTEM'` and `cancellation_reason = 'AVAILABILITY_OVERRIDE_CONFLICT'`, and publish `NOTIFICATION_REQUESTED` events to notify the affected customers with urgency `CRITICAL`


### Requirement 19: Stripe Webhook Handling

**User Story:** As a transaction service operator, I want to handle Stripe webhook events reliably, so that payment state is always synchronized between the platform and Stripe.

#### Acceptance Criteria

1. WHEN Stripe sends a webhook event to `POST /api/webhooks/stripe`, THE Transaction_Service SHALL verify the webhook signature using the Stripe webhook signing secret before processing
2. THE Transaction_Service SHALL reject webhook requests with invalid signatures by returning `400 Bad Request`
3. THE Transaction_Service SHALL reject webhook requests with timestamps older than 5 minutes (clock tolerance) to prevent replay attacks
4. THE Transaction_Service SHALL process the following Stripe webhook event types:
   - `payment_intent.succeeded` — update payment and booking status to CAPTURED/CONFIRMED
   - `payment_intent.payment_failed` — update payment status to FAILED, cancel booking, notify customer
   - `payment_intent.canceled` — update payment status to CANCELLED, update booking status accordingly
   - `charge.dispute.created` — update payment status to DISPUTED, notify court owner
   - `charge.dispute.closed` — update dispute status (WON or LOST)
   - `charge.refunded` — update payment status to REFUNDED or PARTIALLY_REFUNDED, reconcile with local refund state
   - `account.updated` — update court owner's Stripe Connect status
5. THE Transaction_Service SHALL process webhook events idempotently by storing processed Stripe event IDs in Redis with a 24-hour TTL. Duplicate events SHALL be acknowledged with `200 OK` without reprocessing
6. THE Transaction_Service SHALL always return `200 OK` to Stripe after successful processing to prevent Stripe from retrying. Processing failures SHALL be logged and handled asynchronously
7. THE Transaction_Service SHALL implement a payment reconciliation Quartz job that runs every 15 minutes to detect and resolve orphaned payments (PaymentIntent succeeded but booking not updated) by querying Stripe's API
8. THE Transaction_Service SHALL expose the webhook endpoint without JWT authentication (Stripe cannot send JWTs). Security is provided by webhook signature verification only

### Requirement 20: Cross-Schema Data Access

**User Story:** As a transaction service developer, I want read-only access to court and user data from the platform schema, so that booking operations can enrich responses without HTTP round-trips.

#### Acceptance Criteria

1. THE Transaction_Service SHALL use the `v_court_summary` cross-schema view to read court data including `id`, `owner_id`, `name_el`, `name_en`, `court_type`, `location_type`, `timezone`, `base_price_cents`, `duration_minutes`, `max_capacity`, `confirmation_mode`, `confirmation_timeout_hours`, `waitlist_enabled`, and `version`
2. THE Transaction_Service SHALL use the `v_user_basic` cross-schema view to read user data including `id`, `email`, `name`, `phone`, `role`, `language`, `stripe_connect_account_id`, `stripe_connect_status`, `stripe_customer_id`, and `status`
3. THE Transaction_Service SHALL use the `v_court_cancellation_tiers` cross-schema view to read cancellation policy tiers including `court_id`, `threshold_hours`, `refund_percent`, and `sort_order`
4. THE Transaction_Service SHALL use the `v_user_skill_level` cross-schema view to read user skill level data including `user_id`, `court_type`, and `skill_level` (for future Phase 10 open match functionality)
5. THE Transaction_Service SHALL NOT write to any platform schema tables. All writes to platform data SHALL go through Platform Service's internal APIs
6. THE Transaction_Service SHALL use cross-schema views for display purposes and non-critical reads. For critical operations (Stripe Connect status before payment capture), THE Transaction_Service SHALL call Platform Service's internal API for a fresh value

### Requirement 21: Idempotency Support

**User Story:** As a mobile app developer, I want all state-changing booking operations to support idempotency keys, so that network retries do not create duplicate bookings or payments.

#### Acceptance Criteria

1. THE Transaction_Service SHALL accept an `X-Idempotency-Key` header (UUID v4) on all state-changing endpoints: `POST /api/bookings`, `POST /api/bookings/manual`, `POST /api/bookings/recurring`, `POST /api/bookings/{bookingId}/cancel`, `POST /api/bookings/{bookingId}/confirm`, `POST /api/bookings/{bookingId}/reject`
2. WHEN a request is received with an idempotency key that matches an existing `bookings.idempotency_key`, THE Transaction_Service SHALL return the original response (same status code and body) without re-executing the operation
3. THE Transaction_Service SHALL store idempotency keys in the `bookings.idempotency_key` column with a UNIQUE constraint. For non-booking operations (confirm, reject, cancel), THE Transaction_Service SHALL use a Redis-based idempotency cache with a 24-hour TTL keyed by `{operation}:{bookingId}:{idempotencyKey}`
4. THE Transaction_Service SHALL pass the idempotency key to Stripe when creating PaymentIntents to ensure Stripe-level deduplication as well
5. WHEN a request is received without an `X-Idempotency-Key` header on a required endpoint, THE Transaction_Service SHALL generate one internally (UUID v4) to ensure database uniqueness

### Requirement 22: Notification Event Publishing

**User Story:** As a platform operator, I want booking-related notification events published to Kafka for forward compatibility with Phase 5 (Notifications), so that notification delivery can be enabled without modifying booking logic.

#### Acceptance Criteria

1. THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic using `userId` as the partition key
2. THE Transaction_Service SHALL include bilingual `title` and `body` (Greek and English) in all notification events
3. THE Transaction_Service SHALL set the appropriate `urgency` level for each notification type:
   - `CRITICAL`: payment failures, booking cancellations, booking rejections, confirmation timeouts, Stripe Connect restrictions, disputes
   - `STANDARD`: booking confirmations, booking modifications, pending confirmation alerts, no-show flags, external payment marks
   - `PROMOTIONAL`: recurring booking price changes
4. THE Transaction_Service SHALL include `data` payload with `bookingId`, `courtId`, `paymentId` (where applicable), and `deepLink` for client-side navigation
5. THE Transaction_Service SHALL set the `channels` array based on notification type:
   - Booking confirmations/cancellations: `["PUSH", "EMAIL", "IN_APP"]`
   - Payment failures/disputes: `["PUSH", "EMAIL", "IN_APP"]`
   - Pending confirmation reminders: `["PUSH", "IN_APP"]`
   - Price change notifications: `["EMAIL", "IN_APP"]`
6. THE Transaction_Service SHALL publish the following notification types from Phase 4:
   - `BOOKING_CONFIRMED` — to customer when booking is confirmed
   - `BOOKING_CANCELLED` — to customer/court owner when booking is cancelled
   - `BOOKING_REJECTED` — to customer when booking is rejected by court owner
   - `BOOKING_PENDING_CONFIRMATION` — to customer when booking requires court owner approval, and to court owner when a new pending booking arrives
   - `PAYMENT_RECEIVED` — to court owner when payment is captured
   - `PAYMENT_FAILED` — to customer when payment fails
   - `PAYMENT_DISPUTE` — to court owner when a dispute is opened
   - `REFUND_COMPLETED` — to customer when refund is processed
   - `RECURRING_BOOKING_PRICE_CHANGED` — to customer when recurring booking price changes due to pricing update
   - `PENDING_CONFIRMATION_REMINDER` — to court owner as a reminder for unacted pending bookings (via Quartz job)


### Requirement 23: Booking Audit Trail

**User Story:** As a court owner or platform admin, I want a complete audit trail of all booking lifecycle events, so that I can review the history of any booking for dispute resolution or operational review.

#### Acceptance Criteria

1. THE Transaction_Service SHALL record an entry in the `audit_logs` table for every booking state change with `booking_id`, `action`, `performed_by`, `performed_by_role`, `details` (JSONB), and `created_at`
2. THE Transaction_Service SHALL record the following actions: `CREATED`, `CONFIRMED`, `REJECTED`, `CANCELLED`, `MODIFIED`, `COMPLETED`, `NO_SHOW_FLAGGED`, `PAYMENT_CAPTURED`, `PAYMENT_FAILED`, `PAYMENT_REFUNDED`, `PAYMENT_DISPUTED`, `PAYMENT_MARKED_EXTERNAL`
3. THE Transaction_Service SHALL set `performed_by_role` to `CUSTOMER`, `COURT_OWNER`, or `SYSTEM` based on who triggered the action
4. THE Transaction_Service SHALL include before/after snapshots in the `details` JSONB field for modification actions (date, time, price changes)
5. THE Transaction_Service SHALL include refund details (amount, percentage, applied tier) in the `details` JSONB field for cancellation actions
6. THE `audit_logs` table SHALL be append-only. THE Transaction_Service SHALL NOT update or delete audit log entries
7. WHEN a booking detail is retrieved via `GET /api/bookings/{bookingId}`, THE Transaction_Service SHALL include the complete audit trail sorted by `created_at` ascending

### Requirement 24: Booking Status Transition Integrity

**User Story:** As a platform operator, I want booking status transitions to follow a strict state machine, so that bookings cannot reach invalid states.

#### Acceptance Criteria

1. THE Transaction_Service SHALL enforce the following valid booking status transitions:
   - `PENDING_CONFIRMATION` → `CONFIRMED` (court owner confirms)
   - `PENDING_CONFIRMATION` → `REJECTED` (court owner rejects)
   - `PENDING_CONFIRMATION` → `CANCELLED` (customer cancels, or system timeout)
   - `CONFIRMED` → `CANCELLED` (customer or court owner cancels)
   - `CONFIRMED` → `COMPLETED` (scheduled end time passes)
   - `CONFIRMED` → `MODIFIED` (time slot changed — then back to `CONFIRMED`)
2. IF an operation attempts an invalid status transition, THEN THE Transaction_Service SHALL return `400 Bad Request` with `{ error: "INVALID_STATUS_TRANSITION", currentStatus: string, attemptedAction: string }`
3. THE Transaction_Service SHALL enforce status transitions at the application layer before any database update or Stripe API call
4. THE Transaction_Service SHALL use optimistic locking on the `bookings` table (via `updated_at` comparison or a version column) to prevent concurrent status transitions on the same booking

### Requirement 25: Payment Status Transition Integrity

**User Story:** As a platform operator, I want payment status transitions to follow a strict state machine synchronized with Stripe, so that payment state is always consistent.

#### Acceptance Criteria

1. THE Transaction_Service SHALL enforce the following valid payment status transitions:
   - `PENDING` → `AUTHORIZED` (Stripe authorization succeeded)
   - `PENDING` → `FAILED` (Stripe authorization failed)
   - `AUTHORIZED` → `CAPTURED` (payment captured on confirmation)
   - `AUTHORIZED` → `REFUNDED` (authorization voided on rejection/timeout)
   - `CAPTURED` → `REFUNDED` (full refund issued)
   - `CAPTURED` → `PARTIALLY_REFUNDED` (partial refund issued)
   - `CAPTURED` → `DISPUTED` (chargeback opened)
   - `NOT_REQUIRED` → `PAID_EXTERNALLY` (manual booking marked as paid)
2. IF an operation attempts an invalid payment status transition, THEN THE Transaction_Service SHALL log an error and reject the operation
3. THE Transaction_Service SHALL synchronize payment status with Stripe webhook events. WHEN a discrepancy is detected between local state and Stripe state, THE Transaction_Service SHALL log a reconciliation warning and update local state to match Stripe (Stripe is the source of truth for payment state)

