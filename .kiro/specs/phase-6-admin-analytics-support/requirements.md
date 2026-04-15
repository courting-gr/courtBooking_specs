# Requirements Document — Phase 6: Admin, Analytics & Support

## Introduction

Phase 6 implements the admin dashboard, analytics, support ticket system, and platform administration subsystem for the Court Booking Platform, targeting `court-booking-platform-service` (analytics API, support tickets, feature flags, admin operations, global search) and `court-booking-admin-web` (React admin portal). This phase delivers the analytics API with revenue reports, booking trends, and occupancy heatmaps using the read replica, CSV/PDF export with rate limiting, the support ticket system (CRUD, assignment, responses, attachments, metrics), feature flag management API, platform admin operations (user suspend/unsuspend, dispute escalation, platform-wide analytics), global search across bookings/courts/promo codes, bulk court visibility toggle, admin web portal scaffolding with core pages, admin UI security (CSRF, session timeout, data masking, concurrent access protection), and Kafka consumption from the `analytics-events` topic.

This phase builds on Phase 5 (Real-time & Notifications), which established WebSocket infrastructure, FCM push, SendGrid email, Web Push, notification preferences consumption, and booking reminder delivery. Phase 4 (Booking & Payments) established the booking lifecycle, payment processing, Stripe Connect, recurring bookings, bulk booking confirm/reject, iCal export, and Quartz-based scheduled jobs. Phase 3 (Court Management) established court CRUD, availability, pricing, cancellation tiers, court owner verification, audit logs, reminder rules, notification preferences, court defaults, bulk court pricing/availability operations, and the dashboard service (basic summary with court counts, verification status, Stripe status, and pending actions).

**Master requirements coverage:** Req 15 (Analytics and Revenue Tracking), Req 15a (Court Owner Personal Settings and Profile Management — settings page integration), Req 15b (Smart Reminder Rules — configurable alerts), Req 15c (Court Owner Audit Logs — immutable trail), Req 15d (Admin UI Security and Data Protection), Req 15e (Global Search, Bulk Operations, Calendar Integration), Req 16 (Platform Observability — admin dashboard metrics), Req 17 (Support Ticket System), Req 18 (Platform Admin Operations), Req 19 (Performance, Scalability, and Caching — admin/analytics parts).

**What already exists from previous phases:**
- Dashboard service (`DashboardService`) with basic court summary, verification status, Stripe status, and pending actions (Phase 3). Phase 6 extends this with today's bookings, revenue summary, occupancy rates, and action-required counts.
- Bulk court pricing and availability operations (`BulkCourtOperationsUseCase`) in Platform Service (Phase 3). Phase 6 adds bulk court visibility toggle.
- Bulk booking confirm/reject (`BulkBookingActionService`) in Transaction Service (Phase 4). Already complete.
- iCal export (`ExportBookingCalendarService`) in Transaction Service (Phase 4). Already complete.
- Court owner audit logs (`court_owner_audit_logs` table, `AuditLogPort`) — append-only logging already implemented in Phase 3. Phase 6 adds the read/query API for court owners and platform admins.
- Reminder rules (`reminder_rules` table, `ReminderRulesController`, `ManageReminderRulesUseCase`) — CRUD already implemented in Phase 3. Phase 6 adds the actual reminder evaluation and delivery integration.
- Notification preferences (`court_owner_notification_preferences` table, `NotificationPreferencesController`) — CRUD already implemented in Phase 3. Phase 6 integrates these into the admin settings page.
- Court defaults (`court_defaults` table, `CourtDefaultsController`) — CRUD already implemented in Phase 3. Phase 6 integrates into the admin settings page.
- Database tables already created: `support_tickets`, `support_messages`, `support_attachments`, `court_owner_audit_logs`, `feature_flags`, `reminder_rules`, `court_owner_notification_preferences`, `court_defaults`.

**Database migrations required (Flyway):**
- **ALTER `reminder_rules.rule_type` CHECK constraint:** Change from `('BOOKING_REMINDER','PENDING_CONFIRMATION','LOW_OCCUPANCY','DAILY_SUMMARY')` to `('UNPAID_BOOKING','PAYMENT_HELD_NOT_CAPTURED','PENDING_CONFIRMATION','NO_CONTACT_MANUAL','LOW_OCCUPANCY')`. The Phase 6 / OpenAPI rule types are the canonical set; the existing DB schema from Phase 3 used placeholder types that are now superseded.
- **ADD `PAYOUT_ISSUES` to `support_tickets.category` CHECK constraint:** Extend from `('BOOKING','PAYMENT','COURT','ACCOUNT','TECHNICAL','OTHER')` to `('BOOKING','PAYMENT','COURT','ACCOUNT','TECHNICAL','PAYOUT_ISSUES','OTHER')`.
- **CREATE `platform.admin_audit_logs` table:** For admin user management actions (suspend, unsuspend). Columns: `id UUID PK`, `admin_id UUID NOT NULL FK → users(id)`, `target_user_id UUID NOT NULL FK → users(id)`, `action VARCHAR(50) NOT NULL`, `reason TEXT`, `metadata JSONB`, `ip_address VARCHAR(45)`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`. This table is append-only (no UPDATE or DELETE).
- **CREATE cross-schema view `v_recent_notifications`:** Platform Service Flyway migration to create a view on `transaction.notifications` for dashboard `recentNotifications` data. Grant SELECT to `platform_service_role`. View returns: `id`, `user_id`, `type`, `message`, `created_at`, `read` filtered to the last 5 notifications per user.
- **ADD `tsvector` columns and GIN indexes on `platform.courts`:** For full-text search (Req 12 AC 6). Add `tsvector` columns for `name_el`, `name_en`, `description_el`, `description_en`, and `address`. Create GIN indexes on each. Add a trigger to keep `tsvector` columns in sync on INSERT/UPDATE.

**New API endpoints (not yet in OpenAPI spec):**
Phase 6 introduces the following endpoints that need to be added to the OpenAPI specification as part of implementation:
- `GET /api/users/me/sessions` and `DELETE /api/users/me/sessions/{sessionId}` (Req 22 — Active Sessions Management)
- `POST /api/users/me/profile-photo` and `DELETE /api/users/me/profile-photo` (Req 23 — Profile Photo Upload)
- `POST /api/users/me/data-export` and `GET /api/users/me/data-exports/{exportId}` (Req 21 — GDPR Data Export)
- `GET /api/auth/csrf-token` (Req 14 — CSRF Protection)
- `GET /api/analytics/heatmap` (Req 3 — Occupancy Heatmap)
- `POST /api/admin/feature-flags` and `DELETE /api/admin/feature-flags/{flagKey}` (Req 8 — Feature Flag Management)
- `GET /api/admin/analytics` (Req 11 — Platform-Wide Analytics)
- `GET /api/admin/audit-logs` (Req 5 — Audit Log Query for admins)
- `GET /api/admin/search` (Req 12 — Global Search for admins)
- `POST /api/admin/disputes/{disputeId}/notes` (Req 10 — Dispute Escalation)
- `GET /api/admin/users/{userId}` (Req 9 — User Detail)
- `GET /api/feature-flags/{flagKey}` (Req 8 — Public feature flag endpoint)

**New internal API endpoints (not yet in OpenAPI spec):**
- `GET /internal/disputes` — list/search disputes for Platform Service proxy (Req 10)
- `GET /internal/disputes/{disputeId}` — dispute detail for Platform Service proxy (Req 10)
- `GET /internal/bookings/search` — booking search for global search (Req 12)

**Scope boundaries:**
- Court ratings and reviews are ⏳ Phase 10.
- Promo code management is ⏳ Phase 10. Global search for promo codes returns empty results until Phase 10.
- Subscription billing management is ⏳ Phase 10.
- Security hardening (abuse detection, IP blocklist, security alerts) is Phase 7. Phase 6 implements basic admin UI security (CSRF, session timeout, data masking) but not the full security monitoring system.
- Scheduled reports (Req 15 AC 6-9) are ⏳ Phase 2 features, deferred.
- Scheduled report configuration endpoints (`/api/settings/scheduled-reports`) are defined in the OpenAPI spec but SHALL NOT be implemented in Phase 6. They are deferred to Phase 10 along with the scheduled report delivery feature.
- Mobile app admin screens are Phase 8.

## Glossary

- **Platform_Service**: The Spring Boot microservice responsible for authentication, authorization, user management, courts, availability, analytics, support tickets, feature flags, and admin operations. Deployed to `court-booking-platform-service`.
- **Transaction_Service**: The Spring Boot microservice responsible for bookings, payments, notifications, and scheduled jobs. Provides booking/payment data consumed by Platform Service analytics via cross-schema views and internal APIs.
- **Admin_Web_Portal**: The React single-page application for court owners and platform admins. Deployed to `court-booking-admin-web`. Communicates with Platform Service and Transaction Service via REST APIs through NGINX Ingress.
- **Analytics_API**: REST endpoints in Platform Service that aggregate booking, revenue, and occupancy data using the PostgreSQL read replica to avoid impacting booking performance.
- **Read_Replica**: PostgreSQL read replica used by Platform Service for analytics queries. Configured as a secondary DataSource with `@Transactional(readOnly = true)` routing.
- **Dashboard_Summary**: Aggregated overview data for court owners including today's bookings, pending confirmations, revenue this month, next payout, and action-required counts. Cached in Redis with 1-minute TTL.
- **Support_Ticket**: An in-app support request created by any user, categorized by type (BOOKING, PAYMENT, COURT, ACCOUNT, TECHNICAL, PAYOUT_ISSUES, OTHER), with status lifecycle OPEN → IN_PROGRESS → WAITING_ON_USER → RESOLVED → CLOSED. Stored in `platform.support_tickets`.
- **Support_Message**: A message within a support ticket conversation thread. Stored in `platform.support_messages`.
- **Support_Attachment**: A file attached to a support message, stored in DigitalOcean Spaces. Metadata stored in `platform.support_attachments`.
- **Feature_Flag**: A platform-wide boolean toggle stored in `platform.feature_flags`. Managed by PLATFORM_ADMIN users. Used to enable/disable features like open matches, waitlists, and advertisements.
- **Court_Owner_Audit_Log**: An immutable record of court owner actions stored in `platform.court_owner_audit_logs`. Append-only — no UPDATE or DELETE permitted.
- **Occupancy_Rate**: The ratio of booked time slots to total available time slots for a court within a given period, expressed as a decimal between 0.0 and 1.0.
- **Revenue_Report**: Aggregated financial data including gross revenue, platform fees, net revenue, and period-over-period comparison.
- **Occupancy_Heatmap**: A time-of-day × day-of-week matrix showing booking density for a court or set of courts, used to identify peak usage patterns.
- **CSV_Export**: Comma-separated values file format for data export. Rate-limited to 10 exports per court owner per day.
- **PDF_Export**: Portable Document Format file for formatted report export. Rate-limited to 10 exports per court owner per day.
- **Global_Search**: Cross-entity search across bookings, courts, and promo codes with unified results. Supports text matching on court names, booking IDs, and customer names.
- **Bulk_Visibility_Toggle**: Admin operation to enable or disable visibility for multiple courts simultaneously.
- **Platform_Admin**: Administrative user with platform-wide privileges including user management, dispute escalation, feature flag management, and platform analytics. Role: `PLATFORM_ADMIN`.
- **Data_Masking**: Partial obfuscation of PII (phone numbers, emails) in list views. Full details visible only in detail views with audit log entry.
- **CSRF_Protection**: Cross-Site Request Forgery prevention using the Synchronizer Token Pattern for the admin web portal.
- **Session_Timeout**: Auto-logout after 30 minutes of inactivity with a warning at 25 minutes.
- **Analytics_Events_Topic**: The `analytics-events` Kafka topic partitioned by `courtId` for court-level aggregation. Both services publish analytics events; Platform Service consumes them for dashboard and reporting.
- **GDPR_Data_Export**: An asynchronous export of all personal data for a court owner in compliance with GDPR Article 20. Generated as a ZIP archive containing JSON and CSV files. Rate-limited to 10 per day per court owner.
- **Active_Session**: A record of an authenticated user session including device type, browser, partially masked IP address, and last activity timestamp. Court owners can view and revoke active sessions.
- **Profile_Photo**: A profile image or business logo uploaded by a court owner. Stored in DigitalOcean Spaces under `profiles/{userId}/{uuid}.{ext}`. Max 2MB, JPEG/PNG/WebP.

## Requirements


### Requirement 1: Court Owner Dashboard Enhancement

**User Story:** As a court owner, I want a comprehensive dashboard showing today's bookings, revenue summary, occupancy rates, and pending actions, so that I can quickly assess my business status and take action on items that need attention.

#### Acceptance Criteria

1. WHEN a court owner requests the dashboard via `GET /api/dashboard`, THE Platform_Service SHALL return an enhanced dashboard summary including today's bookings count, pending confirmations count, revenue this month in euro cents, next payout date and amount, and action-required counts
   - **Output:** `200 OK` with:
     ```
     {
       todayBookings: number,
       pendingConfirmations: number,
       revenueThisMonthCents: number,
       nextPayoutDate: ISO8601-date | null,
       nextPayoutAmountCents: number | null,
       occupancyRateToday: number (0.0-1.0),
       actionRequired: {
         pendingBookings: number,
         unpaidManualBookings: number,
         expiringPromos: number,
         reminderAlerts: number
       },
       recentNotifications: [{ id: UUID, type: string, message: string, createdAt: ISO8601, read: boolean }],
       courtSummary: {
         totalCourts: number,
         visibleCourts: number,
         hiddenCourts: number
       },
       verificationStatus: string,
       stripeConnectStatus: string
     }
     ```
   - **Output (not court owner):** `403 Forbidden`
2. WHEN computing today's bookings and pending confirmations, THE Platform_Service SHALL query the `transaction.bookings` cross-schema view filtered by the court owner's courts and today's date in the court's timezone
3. WHEN computing revenue this month, THE Platform_Service SHALL aggregate `court_owner_net_cents` from confirmed and completed bookings for the current calendar month using the read replica
4. WHEN computing the next payout date and amount, THE Platform_Service SHALL fetch payout schedule data from Stripe Connect API for the court owner's connected account
5. THE Platform_Service SHALL cache the dashboard result in Redis with a 1-minute TTL keyed by `dashboard:{courtOwnerId}`. WHEN the cache is unavailable, THE Platform_Service SHALL compute the dashboard directly without caching
6. WHEN computing action-required counts, THE Platform_Service SHALL count: pending bookings awaiting confirmation, manual bookings with `payment_status = 'NOT_REQUIRED'` within the next 7 days, and active reminder rule alerts. **Note:** `expiringPromos` SHALL return 0 until promo code management is implemented in Phase 10
7. WHEN computing occupancy rate for today, THE Platform_Service SHALL calculate the ratio of booked slots to total available slots across all of the court owner's visible courts for today's date
8. THE Admin_Web_Portal SHALL display a "Needs Attention" widget on the dashboard showing all bookings that currently match active reminder rules, grouped by trigger type
9. THE "Needs Attention" widget SHALL provide one-click actions for each alert: view booking details, contact customer (opens email compose with pre-filled subject and booking context if email is available, or displays phone number with click-to-call link if phone is available), cancel booking, or dismiss the alert
10. WHEN computing `recentNotifications`, THE Platform_Service SHALL fetch the 5 most recent notifications for the court owner via the cross-schema view `v_recent_notifications` (created by a Platform Service Flyway migration on `transaction.notifications`, with SELECT granted to `platform_service_role`). This avoids an internal HTTP call to Transaction Service for dashboard rendering. See "Database migrations required" section for the view definition

### Requirement 2: Analytics API — Revenue Reports and Booking Trends

**User Story:** As a court owner, I want to view detailed analytics on revenue, booking trends, and occupancy patterns, so that I can make data-driven decisions about pricing and court management.

#### Acceptance Criteria

1. THE Platform_Service SHALL expose two analytics endpoints aligned with the OpenAPI spec:
   - `GET /api/analytics/revenue` for revenue data
   - `GET /api/analytics/usage` for usage/booking trend data
   - Both endpoints share the same **input query parameters:** `from` (ISO8601-date, required), `to` (ISO8601-date, required), `courtId` (UUID, optional — filter to specific court), `courtType` (enum, optional — filter by court type)
   - **Output for `GET /api/analytics/revenue`:** `200 OK` with:
     ```
     {
       period: { from: ISO8601-date, to: ISO8601-date },
       totalBookings: number,
       confirmedBookings: number,
       cancelledBookings: number,
       noShows: number,
       revenueGrossCents: number,
       platformFeesCents: number,
       revenueNetCents: number,
       courtBreakdown: [{ courtId: UUID, courtName: string, bookings: number, revenueCents: number, occupancyRate: number }],
       previousPeriodComparison: { bookingsChange: number, revenueChange: number }
     }
     ```
   - **Output for `GET /api/analytics/usage`:** `200 OK` with:
     ```
     {
       period: { from: ISO8601-date, to: ISO8601-date },
       occupancyRate: number (0.0-1.0),
       peakHours: [{ dayOfWeek: string, hour: number, bookingCount: number }],
       courtBreakdown: [{ courtId: UUID, courtName: string, bookings: number, occupancyRate: number }]
     }
     ```
   - **Output (invalid date range):** `400 Bad Request` with `{ error: "INVALID_DATE_RANGE", message: "Date range must not exceed 365 days" }`
   - **Output (not court owner):** `403 Forbidden`
2. THE Platform_Service SHALL validate that the date range does not exceed 365 days
3. WHEN computing `previousPeriodComparison`, THE Platform_Service SHALL calculate the same metrics for the equivalent preceding period (e.g., if the requested period is 30 days, compare with the previous 30 days) and return percentage changes for bookings and revenue
4. WHEN computing `peakHours`, THE Platform_Service SHALL aggregate booking counts by day of week and hour of day across the requested period, returning the top 10 busiest time slots
5. WHEN computing `occupancyRate`, THE Platform_Service SHALL calculate the ratio of booked time (in minutes) to total available time (from availability windows minus overrides) for each court, then average across all courts in the result set
6. THE Platform_Service SHALL execute all analytics queries against the read replica DataSource to avoid impacting booking performance
7. WHEN `courtId` is provided, THE Platform_Service SHALL validate that the court belongs to the authenticated court owner before returning analytics
8. THE Platform_Service SHALL include no-show counts in the analytics response, aggregated from `bookings.no_show = true` within the date range

### Requirement 3: Analytics API — Occupancy Heatmap

**User Story:** As a court owner, I want to see an occupancy heatmap showing booking density by day of week and time of day, so that I can identify peak and off-peak periods for pricing optimization.

#### Acceptance Criteria

1. WHEN a court owner requests an occupancy heatmap via `GET /api/analytics/heatmap`, THE Platform_Service SHALL return a matrix of booking counts by day of week and hour of day. **Note:** This endpoint is new and needs to be added to the OpenAPI spec as part of Phase 6 implementation
   - **Input query parameters:** `from` (ISO8601-date, required), `to` (ISO8601-date, required), `courtId` (UUID, optional), `courtType` (enum, optional)
   - **Output:** `200 OK` with:
     ```
     {
       period: { from: ISO8601-date, to: ISO8601-date },
       heatmap: [
         { dayOfWeek: "MONDAY", hours: [{ hour: 8, bookingCount: number, occupancyRate: number }, ...] },
         { dayOfWeek: "TUESDAY", hours: [...] },
         ...
       ]
     }
     ```
2. THE Platform_Service SHALL compute the heatmap using the read replica, aggregating booking start times by day of week (MONDAY through SUNDAY) and hour of day (0 through 23)
3. WHEN `courtId` is provided, THE Platform_Service SHALL validate ownership and return the heatmap for that specific court only
4. THE Platform_Service SHALL calculate `occupancyRate` per cell as the ratio of bookings in that hour to the number of weeks in the period multiplied by the number of courts with availability windows covering that hour

### Requirement 4: CSV and PDF Export with Rate Limiting

**User Story:** As a court owner, I want to export my analytics data and booking reports in CSV and PDF formats, so that I can use the data in external tools and share reports with stakeholders.

#### Acceptance Criteria

1. WHEN a court owner requests a CSV export via `GET /api/analytics/export?format=csv`, THE Platform_Service SHALL generate a CSV file containing the analytics data for the specified date range and return it with `Content-Type: text/csv` and `Content-Disposition: attachment; filename="analytics-{from}-{to}.csv"`
   - **Input query parameters:** `from`, `to`, `courtId` (optional), `courtType` (optional), `format` (required: `csv` or `pdf`)
   - **Output (success):** `200 OK` with file download
   - **Output (rate limited):** `429 Too Many Requests` with `{ error: "EXPORT_RATE_LIMIT_EXCEEDED", maxExportsPerDay: 10, retryAfterSeconds: number }`
2. WHEN a court owner requests a PDF export via `GET /api/analytics/export?format=pdf`, THE Platform_Service SHALL generate a formatted PDF report containing revenue summary, booking trends chart data, occupancy rates, and court breakdown, and return it with `Content-Type: application/pdf`
3. THE Platform_Service SHALL enforce a rate limit of 10 exports per court owner per day (rolling 24-hour window). THE Platform_Service SHALL track export counts in Redis with key `export-limit:{courtOwnerId}` and a 24-hour TTL
4. WHEN an export is generated, THE Platform_Service SHALL record a `DATA_EXPORTED` entry in the `court_owner_audit_logs` table with the export format, date range, and number of records
5. THE Platform_Service SHALL execute export queries against the read replica
6. THE Platform_Service SHALL limit export date ranges to a maximum of 365 days
7. THE CSV export SHALL include columns: date, courtName, courtType, bookingCount, confirmedCount, cancelledCount, noShows, revenueGrossCents, platformFeesCents, revenueNetCents, occupancyRate

### Requirement 5: Court Owner Audit Log Query API

**User Story:** As a court owner, I want to view an immutable audit trail of all my actions, so that I can review my activity history and maintain accountability.

#### Acceptance Criteria

1. WHEN a court owner requests their audit log via `GET /api/audit-logs`, THE Platform_Service SHALL return paginated audit log entries from the `court_owner_audit_logs` table filtered to the authenticated court owner
   - **Input query parameters:** `page` (default 0), `size` (default 20, max 100), `from` (ISO8601-datetime, optional), `to` (ISO8601-datetime, optional), `action` (string, optional — filter by action type), `courtId` (UUID, optional), `sort` (default `createdAt,desc`)
   - **Output:** `200 OK` with:
     ```
     {
       content: [{
         id: UUID,
         courtId: UUID | null,
         action: string,
         entityType: string,
         entityId: UUID | null,
         changes: object,
         createdAt: ISO8601
       }],
       page: number, size: number, totalElements: number, totalPages: number
     }
     ```
   - Note: `ipAddress` and `userAgent` are NOT returned to court owners. IP addresses are stored hashed internally for platform admin investigation use only
2. WHEN a PLATFORM_ADMIN requests audit logs for a specific court owner via `GET /api/admin/audit-logs?courtOwnerId={uuid}`, THE Platform_Service SHALL return the audit log entries for that court owner, including the hashed `ipAddress` field for investigation purposes
3. THE Platform_Service SHALL validate that court owners can only view their own audit logs. Attempts to access another court owner's logs SHALL return `403 Forbidden`
4. THE Platform_Service SHALL support filtering by action type. Valid action types include: `COURT_CREATED`, `COURT_UPDATED`, `COURT_DELETED`, `PRICING_UPDATED`, `AVAILABILITY_UPDATED`, `CANCELLATION_POLICY_UPDATED`, `SETTINGS_UPDATED`, `DATA_EXPORTED`, `VERIFICATION_SUBMITTED`, `DATA_EXPORT_REQUESTED`, `COURT_VISIBILITY_TOGGLED`, `CUSTOMER_DATA_ACCESSED`, `SESSION_REVOKED`
5. THE audit log table is append-only. THE Platform_Service SHALL NOT expose any UPDATE or DELETE operations on audit log entries
6. THE Platform_Service SHALL retain audit log entries for a minimum of 2 years


### Requirement 6: Support Ticket System

**User Story:** As a platform user, I want to create and manage support tickets with message threads and file attachments, so that I can get help with booking, payment, or account issues.

#### Acceptance Criteria

1. WHEN a user creates a support ticket via `POST /api/support/tickets`, THE Platform_Service SHALL create a ticket in the `support_tickets` table with status `OPEN` and priority `NORMAL`
   - **Input:** `{ category: enum(BOOKING|PAYMENT|COURT|ACCOUNT|TECHNICAL|PAYOUT_ISSUES|OTHER), subject: string (3-255 chars), body: string (10-5000 chars), relatedBookingId?: UUID, relatedCourtId?: UUID, diagnosticData?: object }`
   - **Note:** The category enum is the union of all categories across the DB schema and OpenAPI spec. A Flyway migration is required to add `PAYOUT_ISSUES` to the `support_tickets.category` CHECK constraint (see "Database migrations required" section).
   - **Output (success):** `201 Created` with `{ ticketId: UUID, status: "OPEN", priority: "NORMAL", createdAt: ISO8601 }`
   - **Output (validation error):** `400 Bad Request` with `{ errors: [{ field, message }] }`
2. WHEN a user retrieves their tickets via `GET /api/support/tickets`, THE Platform_Service SHALL return paginated tickets filtered to the authenticated user
   - **Input query parameters:** `page` (default 0), `size` (default 20), `status` (optional — filter by status), `sort` (default `updatedAt,desc`)
   - **Output:** `200 OK` with paginated list of `{ ticketId, category, subject, status, priority, assignedTo: { name } | null, createdAt, updatedAt, messageCount: number }`
3. WHEN a user views a ticket via `GET /api/support/tickets/{ticketId}`, THE Platform_Service SHALL return the ticket details with the full message thread and attachments
   - **Output:** `200 OK` with `{ ticketId, category, subject, status, priority, assignedTo, relatedBookingId, relatedCourtId, diagnosticData, messages: [{ messageId, senderId, senderName, senderRole, body, attachments: [{ attachmentId, fileName, fileSize, mimeType, url }], createdAt }], createdAt, updatedAt, resolvedAt }`
   - **Output (not found):** `404 Not Found`
   - **Output (not owner or admin):** `403 Forbidden`
4. WHEN a user or support agent adds a message to a ticket via `POST /api/support/tickets/{ticketId}/messages`, THE Platform_Service SHALL create a message in the `support_messages` table. This endpoint accepts `multipart/form-data` with an optional `attachments` array (max 5 files, max 10 MB each) for inline attachment upload during message creation, matching the OpenAPI spec
   - **Input (JSON):** `{ body: string (1-5000 chars) }`
   - **Input (multipart/form-data):** `body` (string, 1-5000 chars) + optional `attachments` (file array, max 5 files, max 10 MB each, accepted formats: JPEG, PNG, PDF, TXT, LOG)
   - **Output (success):** `201 Created` with the message details including any inline attachments
5. WHEN a support agent or admin adds a message to a ticket, THE Platform_Service SHALL update the ticket status to `IN_PROGRESS` if it was `OPEN`
6. WHEN a user adds a message to a ticket that is `WAITING_ON_USER`, THE Platform_Service SHALL update the ticket status to `IN_PROGRESS`
7. WHEN a file is attached to a message after creation via `POST /api/support/tickets/{ticketId}/messages/{messageId}/attachments`, THE Platform_Service SHALL upload the file to DigitalOcean Spaces under `support/{ticketId}/{messageId}/{uuid}.{ext}` and create an entry in the `support_attachments` table. This is the separate upload endpoint for adding attachments after message creation (complementing the inline attachment support in AC 4)
   - **Accepted formats:** JPEG, PNG, PDF, TXT, LOG
   - **Maximum file size:** 10 MB per attachment
   - **Maximum attachments per message:** 5
   - **Output (success):** `201 Created` with `{ attachmentId, fileName, fileSize, mimeType, url }`
   - **Output (invalid format):** `400 Bad Request` with `{ error: "INVALID_FILE_FORMAT" }`
   - **Output (file too large):** `400 Bad Request` with `{ error: "FILE_TOO_LARGE", maxSizeBytes: 10485760 }`
8. WHEN a PLATFORM_ADMIN assigns a ticket via `PUT /api/support/tickets/{ticketId}/assign`, THE Platform_Service SHALL update the `assigned_to` field
   - **Input:** `{ assignedTo: UUID }`
   - **Output (success):** `200 OK` with updated ticket
   - **Output (assignee not admin):** `400 Bad Request` with `{ error: "ASSIGNEE_NOT_ADMIN" }`
9. WHEN a PLATFORM_ADMIN or SUPPORT_AGENT updates ticket status via `PUT /api/support/tickets/{ticketId}/status`, THE Platform_Service SHALL validate the status transition and update the ticket. SUPPORT_AGENT users have the same status update permissions as PLATFORM_ADMIN for this endpoint. **Note:** SUPPORT_AGENT can respond to tickets (add messages) and update ticket status, but cannot assign tickets (AC 8 is PLATFORM_ADMIN only)
   - **Input:** `{ status: enum(IN_PROGRESS|WAITING_ON_USER|RESOLVED|CLOSED), resolutionNotes?: string }`
   - **Valid transitions:** OPEN → IN_PROGRESS, IN_PROGRESS → WAITING_ON_USER, IN_PROGRESS → RESOLVED, WAITING_ON_USER → IN_PROGRESS, RESOLVED → CLOSED, any → CLOSED (admin only)
   - **Output (invalid transition):** `422 Unprocessable Entity` with `{ error: "INVALID_STATUS_TRANSITION", currentStatus, requestedStatus }`
10. WHEN a ticket is resolved, THE Platform_Service SHALL set the `resolved_at` timestamp
11. WHEN a ticket status changes, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic notifying the ticket creator

### Requirement 7: Support Ticket Metrics

**User Story:** As a platform admin, I want to view support ticket metrics including response times and resolution rates, so that I can monitor support quality and identify areas for improvement.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN requests support metrics via `GET /api/admin/support/metrics`, THE Platform_Service SHALL return aggregated support ticket statistics
   - **Input query parameters:** `from` (ISO8601-date), `to` (ISO8601-date)
   - **Output:** `200 OK` with:
     ```
     {
       period: { from, to },
       totalTickets: number,
       openTickets: number,
       inProgressTickets: number,
       resolvedTickets: number,
       closedTickets: number,
       averageResponseTimeMinutes: number,
       averageResolutionTimeMinutes: number,
       ticketsByCategory: [{ category: string, count: number }],
       ticketsByPriority: [{ priority: string, count: number }]
     }
     ```
2. WHEN computing average response time, THE Platform_Service SHALL calculate the time between ticket creation and the first non-creator message
3. WHEN computing average resolution time, THE Platform_Service SHALL calculate the time between ticket creation and the `resolved_at` timestamp
4. THE Platform_Service SHALL restrict this endpoint to PLATFORM_ADMIN role only

### Requirement 8: Feature Flag Management

**User Story:** As a platform admin, I want to manage feature flags to enable or disable platform features, so that I can control feature rollouts and quickly disable problematic features.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN retrieves feature flags via `GET /api/admin/feature-flags`, THE Platform_Service SHALL return all feature flags from the `feature_flags` table
   - **Output:** `200 OK` with `{ flags: [{ id: UUID, flagKey: string, enabled: boolean, description: string, updatedBy: UUID, updatedAt: ISO8601 }] }`
2. WHEN a PLATFORM_ADMIN updates a feature flag via `PUT /api/admin/feature-flags/{flagKey}`, THE Platform_Service SHALL update the `enabled` field and record the `updated_by` user
   - **Input:** `{ enabled: boolean }`
   - **Output (success):** `200 OK` with updated flag
   - **Output (not found):** `404 Not Found`
   - **Output (not admin):** `403 Forbidden`
3. WHEN a PLATFORM_ADMIN creates a new feature flag via `POST /api/admin/feature-flags`, THE Platform_Service SHALL create a new entry in the `feature_flags` table
   - **Input:** `{ flagKey: string (3-100 chars, uppercase with underscores), description?: string, enabled: boolean }`
   - **Output (success):** `201 Created` with the flag details
   - **Output (duplicate key):** `409 Conflict` with `{ error: "FLAG_KEY_EXISTS" }`
4. WHEN a PLATFORM_ADMIN deletes a feature flag via `DELETE /api/admin/feature-flags/{flagKey}`, THE Platform_Service SHALL remove the flag from the `feature_flags` table
   - **Output (success):** `204 No Content`
5. WHEN a feature flag is updated, THE Platform_Service SHALL invalidate the feature flag cache in Redis and publish the change to all service instances
6. THE Platform_Service SHALL provide a public endpoint `GET /api/feature-flags/{flagKey}` that returns the flag status for client applications
   - **Output:** `200 OK` with `{ flagKey: string, enabled: boolean }`
   - **Output (not found):** `404 Not Found`
7. THE Platform_Service SHALL cache feature flags in Redis with a 5-minute TTL to reduce database queries

### Requirement 9: Platform Admin — User Management

**User Story:** As a platform admin, I want to manage user accounts including suspending and unsuspending users, so that I can enforce platform policies and handle abuse cases.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN retrieves the user list via `GET /api/admin/users`, THE Platform_Service SHALL return paginated user records
   - **Input query parameters:** `page` (default 0), `size` (default 20), `role` (optional), `status` (optional — ACTIVE, SUSPENDED, DELETED), `search` (optional — search by name or email), `sort` (default `createdAt,desc`)
   - **Output:** `200 OK` with paginated list of `{ userId, name, email, role, status, verified, stripeConnectStatus, createdAt, lastLoginAt }`
2. WHEN a PLATFORM_ADMIN suspends a user via `POST /api/admin/users/{userId}/suspend`, THE Platform_Service SHALL set the user's status to `SUSPENDED` in the `users` table
   - **Input:** `{ reason: string (10-500 chars) }`
   - **Output (success):** `200 OK` with `{ userId, status: "SUSPENDED", suspendedAt: ISO8601 }`
   - **Output (already suspended):** `422 Unprocessable Entity` with `{ error: "USER_ALREADY_SUSPENDED" }`
3. WHEN a user is suspended, THE Platform_Service SHALL invalidate all active refresh tokens for that user, preventing further API access after current access token expiry
4. WHEN a user is suspended, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event notifying the user of the suspension with the reason
5. WHEN a PLATFORM_ADMIN unsuspends a user via `POST /api/admin/users/{userId}/unsuspend`, THE Platform_Service SHALL set the user's status back to `ACTIVE`
   - **Output (success):** `200 OK` with `{ userId, status: "ACTIVE", unsuspendedAt: ISO8601 }`
   - **Output (not suspended):** `422 Unprocessable Entity` with `{ error: "USER_NOT_SUSPENDED" }`
6. WHEN a user is unsuspended, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event notifying the user that their account has been reactivated
7. WHEN a PLATFORM_ADMIN views user details via `GET /api/admin/users/{userId}`, THE Platform_Service SHALL return the full user profile including verification history, Stripe Connect status, booking statistics, and support ticket history
8. THE Platform_Service SHALL record all user management actions (suspend, unsuspend) in the `platform.admin_audit_logs` table (see "Database migrations required" section) with the admin's user ID, the target user ID, the action, the reason, metadata (JSONB), and IP address. This is separate from `court_owner_audit_logs` which tracks court owner actions

### Requirement 10: Platform Admin — Dispute Escalation

**User Story:** As a platform admin, I want to view and manage payment disputes that have been escalated, so that I can mediate between court owners and customers and ensure fair resolution.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN retrieves escalated disputes via `GET /api/admin/disputes`, THE Platform_Service SHALL return paginated dispute records by querying the Transaction Service's internal API
   - **Input query parameters:** `page` (default 0), `size` (default 20), `status` (optional — OPEN, UNDER_REVIEW, RESOLVED), `sort` (default `createdAt,desc`)
   - **Output:** `200 OK` with paginated list of `{ disputeId, bookingId, courtOwnerName, customerName, amountCents, reason, status, stripeDisputeId, createdAt }`
2. WHEN a PLATFORM_ADMIN views dispute details via `GET /api/admin/disputes/{disputeId}`, THE Platform_Service SHALL return the full dispute context including booking details, payment timeline, court owner evidence, and customer communication history
3. WHEN a PLATFORM_ADMIN adds a note to a dispute via `POST /api/admin/disputes/{disputeId}/notes`, THE Platform_Service SHALL record the admin note and notify both the court owner and customer
   - **Input:** `{ note: string (10-2000 chars), visibleToParties: boolean }`
4. THE Platform_Service SHALL restrict all dispute management endpoints to PLATFORM_ADMIN role only
5. WHEN a booking dispute is escalated, THE Platform_Service SHALL include the relevant audit log entries (from `court_owner_audit_logs`) as evidence in the dispute record, covering all actions related to the disputed booking and its court

### Requirement 11: Platform Admin — Platform-Wide Analytics

**User Story:** As a platform admin, I want to view platform-wide analytics including total bookings, revenue, user growth, and system health, so that I can monitor the overall platform performance.

#### Acceptance Criteria

1. WHEN a PLATFORM_ADMIN requests platform analytics via `GET /api/admin/analytics`, THE Platform_Service SHALL return platform-wide aggregated metrics using the read replica
   - **Input query parameters:** `from` (ISO8601-date), `to` (ISO8601-date)
   - **Output:** `200 OK` with:
     ```
     {
       period: { from, to },
       totalUsers: number,
       newUsersInPeriod: number,
       totalCourtOwners: number,
       totalCourts: number,
       totalBookings: number,
       totalRevenueCents: number,
       totalPlatformFeesCents: number,
       activeUsers: number,
       bookingsByCourtType: [{ courtType: string, count: number }],
       topCourts: [{ courtId: UUID, courtName: string, bookings: number, revenueCents: number }],
       userGrowth: [{ date: ISO8601-date, newUsers: number, cumulativeUsers: number }]
     }
     ```
2. THE Platform_Service SHALL compute `activeUsers` as users who have made at least one booking or login within the requested period
3. THE Platform_Service SHALL return the top 10 courts by booking count in the `topCourts` array
4. THE Platform_Service SHALL restrict this endpoint to PLATFORM_ADMIN role only
5. THE Platform_Service SHALL execute all platform analytics queries against the read replica

### Requirement 12: Global Search

**User Story:** As a court owner, I want to search across my bookings, courts, and promo codes from a single search bar, so that I can quickly find specific records without navigating to different sections.

#### Acceptance Criteria

1. WHEN a court owner performs a global search via `GET /api/search?q={query}`, THE Platform_Service SHALL search across courts, bookings, and promo codes owned by the authenticated court owner
   - **Input query parameters:** `q` (string, min 2 chars), `type` (optional — `COURT`, `BOOKING`, `PROMO_CODE` to filter result type), `page` (default 0), `size` (default 20)
   - **Output:** `200 OK` with:
     ```
     {
       results: [{
         type: enum(COURT|BOOKING|PROMO_CODE),
         id: UUID,
         title: string,
         subtitle: string,
         metadata: object
       }],
       page: number, size: number, totalElements: number
     }
     ```
2. WHEN searching courts, THE Platform_Service SHALL match against court name (Greek and English), address, and court type
3. WHEN searching bookings, THE Platform_Service SHALL query the Transaction Service's internal API to match against booking ID prefix, customer name, and customer phone. Customer data SHALL be partially masked in search results (e.g., `K***` for name, `+30 69** *** *45` for phone)
4. WHEN searching promo codes, THE Platform_Service SHALL match against promo code string. WHILE promo codes are not implemented (⏳ Phase 10), THE Platform_Service SHALL return empty results for promo code searches
5. WHEN a PLATFORM_ADMIN performs a global search via `GET /api/admin/search?q={query}`, THE Platform_Service SHALL search across all courts, bookings, and users platform-wide
6. THE Platform_Service SHALL use PostgreSQL full-text search with `tsvector` indexes for efficient text matching on court names and descriptions
7. THE Platform_Service SHALL limit search query length to 100 characters

### Requirement 13: Bulk Court Visibility Toggle

**User Story:** As a court owner, I want to enable or disable visibility for multiple courts at once, so that I can efficiently manage seasonal closures or temporary shutdowns.

#### Acceptance Criteria

1. WHEN a court owner submits a bulk visibility toggle via `POST /api/courts/bulk/visibility`, THE Platform_Service SHALL update the `visible` field for all specified courts
   - **Input:** `{ courtIds: [UUID], visible: boolean }`
   - **Output (success):** `200 OK` with `{ successCount: number, failureCount: number, failures: [{ courtId: UUID, reason: string }] }`
   - **Output (limit exceeded):** `400 Bad Request` with `{ error: "BULK_LIMIT_EXCEEDED", maxCourts: 50 }`
2. THE Platform_Service SHALL validate that all specified courts belong to the authenticated court owner. Courts not owned by the user SHALL be reported as failures
3. WHEN setting `visible = true`, THE Platform_Service SHALL validate that the court owner is verified and has an active Stripe Connect account. Courts that fail these preconditions SHALL be reported as failures with specific reasons
4. THE Platform_Service SHALL limit bulk operations to a maximum of 50 courts per request
5. WHEN visibility is toggled, THE Platform_Service SHALL publish a `COURT_UPDATED` event to the `court-update-events` Kafka topic for each affected court
6. WHEN visibility is toggled, THE Platform_Service SHALL record a `COURT_VISIBILITY_TOGGLED` entry in the `court_owner_audit_logs` table for each affected court
7. WHEN a court owner clones a court's configuration via `POST /api/courts/{sourceCourtId}/clone`, THE Platform_Service SHALL copy pricing rules, cancellation tiers, availability windows, confirmation mode, and amenities to the specified target courts. **Note:** This is a Phase 6 addition not present in the original master requirements scope. It is a natural extension of the "Copy settings from one court to another" bullet in the master requirements' Court Owner Admin Journey Step 5
   - **Input:** `{ targetCourtIds: [UUID] }`
   - **Output (success):** `200 OK` with `{ successCount: number, failureCount: number, failures: [{ courtId: UUID, reason: string }] }`
   - THE Platform_Service SHALL validate that both the source court and all target courts belong to the authenticated court owner


### Requirement 14: Admin UI Security and Data Protection

**User Story:** As a platform operator, I want the admin web portal to enforce security best practices including CSRF protection, session timeout, data masking, and concurrent access protection, so that sensitive court owner and customer data is protected.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL implement CSRF protection using the Synchronizer Token Pattern. THE Platform_Service SHALL issue a CSRF token via a `GET /api/auth/csrf-token` endpoint and validate the `X-CSRF-Token` header on all state-changing requests (POST, PUT, DELETE) from the admin portal
2. THE Admin_Web_Portal SHALL implement automatic session timeout after 30 minutes of inactivity. THE Admin_Web_Portal SHALL display a warning dialog at 25 minutes with options to extend the session or log out
3. THE Admin_Web_Portal SHALL mask customer PII in list views: phone numbers displayed as `+30 69** *** *45`, emails displayed as `k***@gmail.com`. Full details SHALL be visible only in detail views
4. WHEN a court owner views full customer details (unmasked phone, email) in a booking detail view, THE Platform_Service SHALL record a `CUSTOMER_DATA_ACCESSED` entry in the `court_owner_audit_logs` table with the booking ID and customer ID
5. THE Admin_Web_Portal SHALL implement optimistic concurrency control for court updates. WHEN two sessions attempt to update the same court simultaneously, THE Admin_Web_Portal SHALL display a conflict resolution dialog showing the diff between the local changes and the server state
6. THE Platform_Service SHALL enforce role-based access control on all admin endpoints: COURT_OWNER endpoints return only the owner's data, PLATFORM_ADMIN endpoints have platform-wide access, SUPPORT_AGENT endpoints have read-only access to support tickets and user activity
7. THE Admin_Web_Portal SHALL implement Content Security Policy (CSP) headers to prevent XSS attacks
8. THE Admin_Web_Portal SHALL sanitize all user-generated content before rendering to prevent stored XSS
9. THE Platform_Service SHALL log all admin portal authentication events (login, logout, session timeout, session extension) with IP address and user agent
10. THE Platform_Service SHALL rate-limit court owner API endpoints to prevent abuse: max 100 requests per minute for read operations, max 30 requests per minute for write operations
11. WHEN rate limits are exceeded, THE Platform_Service SHALL return `429 Too Many Requests` with a `Retry-After` header indicating the number of seconds until the limit resets
12. WHEN rate limited, THE Admin_Web_Portal SHALL display a clear message with a countdown timer and SHALL NOT silently drop requests
13. THE Admin_Web_Portal SHALL set secure cookie attributes: `HttpOnly`, `Secure`, `SameSite=Strict` on all cookies
14. THE Platform_Service SHALL implement rate limiting using a Redis-based sliding window counter at the Spring Security filter level. Redis keys follow the pattern `rate-limit:{courtOwnerId}:{read|write}` with 60-second TTL. This approach allows per-user rate limiting that works across multiple Platform Service pods

### Requirement 15: Admin Web Portal — Core Application

**User Story:** As a court owner, I want a React-based admin web portal with a responsive layout, navigation, and core pages, so that I can manage my courts, bookings, and settings from a web browser.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL be a React single-page application using TypeScript, React Router for navigation, and Ant Design (antd) as the component library for consistent UI. Rationale: Ant Design provides a comprehensive enterprise-grade component set with built-in table, form, and dashboard components that align well with the admin portal's data-heavy UI needs
2. THE Admin_Web_Portal SHALL implement the following core pages:
   - Dashboard (court owner home with summary metrics and quick actions)
   - Courts (list, create, edit, delete courts)
   - Bookings (list, detail, calendar view, pending queue)
   - Analytics (revenue reports, occupancy heatmap, export)
   - Settings (profile, notification preferences, reminder rules, court defaults)
   - Audit Log (filterable activity history)
   - Support (ticket list, ticket detail with message thread)
3. THE Admin_Web_Portal SHALL implement a responsive sidebar navigation with collapsible menu items and active page highlighting
4. THE Admin_Web_Portal SHALL implement OAuth login flow (Google, Facebook, Apple) with JWT token storage in memory (not localStorage) and automatic token refresh
5. THE Admin_Web_Portal SHALL implement role-based UI rendering: COURT_OWNER users see court management, bookings, analytics, and settings; PLATFORM_ADMIN users additionally see user management, feature flags, platform analytics, and dispute management
6. THE Admin_Web_Portal SHALL implement a global search bar in the top navigation that triggers the `GET /api/search` endpoint
7. THE Admin_Web_Portal SHALL implement toast notifications for success/error feedback on all user actions
8. THE Admin_Web_Portal SHALL implement loading skeletons for all data-fetching pages to provide visual feedback during API calls
9. THE Admin_Web_Portal SHALL implement error boundary components that catch rendering errors and display a fallback UI with retry option
10. THE Admin_Web_Portal SHALL support Greek and English languages, using the court owner's language preference from their profile

### Requirement 16: Admin Web Portal — Platform Admin Pages

**User Story:** As a platform admin, I want dedicated admin pages for user management, verification review, feature flags, and platform analytics, so that I can manage the platform effectively.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL implement a User Management page for PLATFORM_ADMIN users with a searchable, filterable user table, user detail view, and suspend/unsuspend actions
2. THE Admin_Web_Portal SHALL implement a Verification Review Queue page showing pending verification requests with document preview, approve/reject actions, and review notes
3. THE Admin_Web_Portal SHALL implement a Feature Flags page with a toggle interface for enabling/disabling flags, creating new flags, and viewing flag history
4. THE Admin_Web_Portal SHALL implement a Platform Analytics page showing platform-wide metrics, user growth charts, booking volume trends, and revenue summaries
5. THE Admin_Web_Portal SHALL implement a Support Tickets page for PLATFORM_ADMIN and SUPPORT_AGENT users with ticket list, assignment, status management, and message thread
6. THE Admin_Web_Portal SHALL implement a Dispute Management page showing escalated disputes with booking context, payment timeline, and admin note functionality
7. WHEN a PLATFORM_ADMIN navigates to admin-only pages, THE Admin_Web_Portal SHALL verify the user's role from the JWT claims and redirect non-admin users to the dashboard with an "Access Denied" message

### Requirement 17: Analytics Events Kafka Consumer

**User Story:** As a platform operator, I want the analytics system to consume events from the analytics-events Kafka topic, so that dashboard and reporting data stays up to date with real-time booking and payment activity.

#### Acceptance Criteria

1. WHEN an analytics event is published to the `analytics-events` Kafka topic, THE Platform_Service SHALL consume the event and update the relevant analytics aggregation tables or caches
2. THE Platform_Service SHALL join the `platform-service-analytics-consumer` consumer group and consume from all partitions of the `analytics-events` topic with `auto.offset.reset=latest`
3. THE Platform_Service SHALL process the following analytics event types: `BOOKING_ANALYTICS` (booking created/confirmed/cancelled/completed), `PAYMENT_ANALYTICS` (payment captured/refunded), `USER_ANALYTICS` (user registered/login)
4. WHEN an analytics event is consumed, THE Platform_Service SHALL update the dashboard cache in Redis to reflect the latest data. THE Platform_Service SHALL invalidate the dashboard cache for the affected court owner
5. THE Platform_Service SHALL implement consumer-side idempotency using the event's `eventId` field stored in Redis with a 7-day TTL to prevent duplicate processing
6. WHEN a malformed analytics event is consumed, THE Platform_Service SHALL log the error at WARN level and skip the event without blocking the consumer

### Requirement 18: Reminder Rule Evaluation and Delivery

**User Story:** As a court owner, I want my configured reminder rules to be evaluated and trigger notifications, so that I receive timely alerts about unpaid bookings, pending confirmations, and low occupancy.

#### Acceptance Criteria

1. THE Transaction_Service SHALL implement a Quartz scheduled job `ReminderRuleEvaluationJob` that runs every 30 minutes and evaluates all active reminder rules against current booking data
2. WHEN evaluating an `UNPAID_BOOKING` rule, THE Transaction_Service SHALL check for manual bookings (created by the court owner without payment processing) that have not been marked as "paid externally" and whose booking date falls within the `daysBeforeEvent` window, and publish a `NOTIFICATION_REQUESTED` event for each matching booking
3. WHEN evaluating a `PAYMENT_HELD_NOT_CAPTURED` rule, THE Transaction_Service SHALL check for customer bookings on manual-confirmation courts where a Stripe authorization hold exists but the court owner has not yet confirmed (captured) the payment, and the booking date falls within the `daysBeforeEvent` window, and publish a `NOTIFICATION_REQUESTED` event for each
4. WHEN evaluating a `PENDING_CONFIRMATION` rule, THE Transaction_Service SHALL check for pending bookings that have been waiting longer than `hoursBeforeThreshold` hours and publish a `NOTIFICATION_REQUESTED` event for each
5. WHEN evaluating a `NO_CONTACT_MANUAL` rule, THE Transaction_Service SHALL check for manual bookings with no customer contact information (no email and no phone) that have existed for longer than `daysBeforeEvent` days and publish a `NOTIFICATION_REQUESTED` event for each
6. WHEN evaluating a `LOW_OCCUPANCY` rule, THE Transaction_Service SHALL calculate the occupancy rate for the next `daysBeforeEvent` days for the rule's courts. WHEN the occupancy rate falls below the threshold, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event
7. THE Transaction_Service SHALL respect the reminder rule's `channels` array when publishing notification events, mapping to the appropriate delivery channels
8. THE Transaction_Service SHALL NOT send duplicate reminders for the same booking-rule combination — each rule fires at most once per booking. Tracking key pattern in Redis: `reminder-fired:{ruleId}:{bookingId}` with 30-day TTL
9. WHEN a reminder rule is disabled (`active = false`), THE Transaction_Service SHALL skip evaluation for that rule
10. WHEN a court owner dismisses a reminder alert, THE Platform_Service SHALL record the dismissal in Redis with key pattern `reminder-dismissed:{ruleId}:{bookingId}` with 30-day TTL (same pattern as `reminder-fired`). THE Transaction_Service SHALL check both `reminder-fired` and `reminder-dismissed` keys before publishing, and SHALL NOT re-trigger the same rule for that booking

### Requirement 19: Court Owner Settings Page Integration

**User Story:** As a court owner, I want a unified settings page in the admin portal that brings together my profile, notification preferences, reminder rules, and court defaults, so that I can manage all my preferences from one place.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL implement a Settings page with tabs for: Profile & Business Info, Notification Preferences, Reminder Rules, Court Defaults, and Security & Privacy
2. THE Profile & Business Info tab SHALL allow court owners to update display name, business name, tax ID, contact phone, business address, language preference, and profile photo via the existing `PUT /api/users/me/profile` endpoint
3. THE Notification Preferences tab SHALL display per-event-type channel toggles (email, push, in-app) and do-not-disturb configuration via the existing `PUT /api/settings/notification-preferences` endpoint
4. THE Reminder Rules tab SHALL display the court owner's configured reminder rules with enable/disable toggles, edit, and delete actions via the existing reminder rules API (`/api/settings/reminder-rules`)
5. THE Court Defaults tab SHALL display and allow editing of default court settings (duration, confirmation mode, cancellation tiers, amenities) via the existing `PUT /api/settings/court-defaults` endpoint
6. THE Security & Privacy tab SHALL display active sessions with device info and "Revoke" buttons, and provide a data export request button
7. WHEN a court owner updates business information that was part of the original verification (business name, tax ID, business address), THE Admin_Web_Portal SHALL display a warning that re-verification may be required

### Requirement 20: Correctness Properties for Property-Based Testing

**User Story:** As a developer, I want property-based tests that verify the correctness of analytics calculations, search results, and bulk operations, so that edge cases are caught automatically.

#### Acceptance Criteria

1. FOR ALL valid date ranges, computing analytics for a period and then computing analytics for two non-overlapping sub-periods that cover the same range SHALL produce totals where `subPeriod1.totalBookings + subPeriod2.totalBookings == fullPeriod.totalBookings` (additivity property)
2. FOR ALL court owners with N courts, the dashboard `courtSummary.totalCourts` SHALL equal `courtSummary.visibleCourts + courtSummary.hiddenCourts` (partition invariant)
3. FOR ALL analytics responses, `revenueGrossCents` SHALL equal `platformFeesCents + revenueNetCents` (revenue decomposition invariant)
4. FOR ALL occupancy heatmap responses, each cell's `occupancyRate` SHALL be between 0.0 and 1.0 inclusive (bounded range invariant)
5. FOR ALL CSV exports, parsing the exported CSV and summing the `revenueNetCents` column SHALL equal the `revenueNetCents` from the corresponding analytics API response (round-trip property for CSV export)
6. FOR ALL bulk visibility toggle operations with N court IDs, `successCount + failureCount` SHALL equal N (completeness invariant)
7. FOR ALL support ticket status transitions, the resulting status SHALL be reachable from the previous status according to the defined state machine (state machine invariant)
8. FOR ALL global search queries, results SHALL only contain entities owned by the authenticated court owner (isolation property). PLATFORM_ADMIN searches are exempt from this constraint
9. FOR ALL feature flag updates, reading the flag immediately after update SHALL return the new value (read-after-write consistency)
10. FOR ALL audit log queries filtered by date range `[from, to]`, every returned entry's `createdAt` SHALL be within the range `from <= createdAt <= to` (filter correctness)
11. FOR ALL analytics `previousPeriodComparison` calculations, the comparison period length SHALL equal the requested period length (symmetric comparison)
12. FOR ALL iCal exports (already implemented in Phase 4), parsing the generated iCalendar string and re-serializing it SHALL produce a valid iCalendar document (round-trip property)
13. FOR ALL GDPR data exports, the exported data SHALL contain all courts owned by the requesting user — the set of court IDs in the export SHALL equal the set of court IDs from `GET /api/courts` for that user (completeness property)
14. FOR ALL session revocation operations, listing active sessions via `GET /api/users/me/sessions` immediately after revoking a session SHALL NOT include the revoked session ID (revocation consistency)
15. FOR ALL reminder rule evaluations, rules with `active = false` SHALL never produce notification events (disabled rule isolation property)

### Requirement 21: GDPR Data Export API

**User Story:** As a court owner, I want to request a full export of all my personal data in compliance with GDPR Article 20, so that I can exercise my right to data portability.

#### Acceptance Criteria

1. WHEN a court owner requests a data export via `POST /api/users/me/data-export`, THE Platform_Service SHALL trigger an asynchronous export generation job that collects all personal data, court data, booking history, and revenue data for the authenticated court owner
   - **Output (success):** `202 Accepted` with `{ exportId: UUID, status: "PROCESSING", requestedAt: ISO8601 }`
   - **Output (rate limited):** `429 Too Many Requests` with `{ error: "DATA_EXPORT_RATE_LIMIT_EXCEEDED", maxExportsPerDay: 10, retryAfterSeconds: number }`
2. THE Platform_Service SHALL generate the export in both JSON and CSV formats, bundled as a ZIP archive containing: `profile.json`, `courts.json`, `courts.csv`, `bookings.json`, `bookings.csv`, `revenue.json`, `revenue.csv`
3. THE Platform_Service SHALL rate-limit data export requests to a maximum of 10 per day per court owner, tracked in Redis with key `gdpr-export-limit:{courtOwnerId}` and a 24-hour TTL
4. WHEN a data export request is submitted, THE Platform_Service SHALL record a `DATA_EXPORT_REQUESTED` entry in the `court_owner_audit_logs` table with the export ID and request timestamp
5. WHEN the export generation is complete, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event notifying the court owner that the export is ready for download
6. WHEN a court owner retrieves a completed export via `GET /api/users/me/data-exports/{exportId}`, THE Platform_Service SHALL return the download URL for the ZIP archive
   - **Output (ready):** `200 OK` with `{ exportId: UUID, status: "READY", downloadUrl: string, expiresAt: ISO8601 }`
   - **Output (processing):** `200 OK` with `{ exportId: UUID, status: "PROCESSING" }`
   - **Output (not found):** `404 Not Found`
7. THE Platform_Service SHALL store the generated export in DigitalOcean Spaces under `exports/{userId}/{exportId}.zip` with a 7-day expiry

### Requirement 22: Active Sessions Management API

**User Story:** As a court owner, I want to view and manage my active sessions, so that I can monitor account access and revoke sessions from unrecognized devices.

#### Acceptance Criteria

1. WHEN a court owner requests their active sessions via `GET /api/users/me/sessions`, THE Platform_Service SHALL return a list of active sessions for the authenticated user
   - **Output:** `200 OK` with:
     ```
     {
       sessions: [{
         sessionId: UUID,
         deviceType: string,
         browser: string,
         ipAddress: string (partially masked, e.g., "192.168.***.***"),
         lastActivity: ISO8601,
         current: boolean
       }]
     }
     ```
2. WHEN a court owner revokes a session via `DELETE /api/users/me/sessions/{sessionId}`, THE Platform_Service SHALL immediately invalidate the refresh token associated with that session
   - **Output (success):** `204 No Content`
   - **Output (not found):** `404 Not Found`
   - **Output (current session):** `422 Unprocessable Entity` with `{ error: "CANNOT_REVOKE_CURRENT_SESSION", message: "Use logout to end the current session" }`
3. THE Platform_Service SHALL prevent court owners from revoking their current active session via this endpoint — the court owner must use the logout flow instead
4. WHEN a session is revoked, THE Platform_Service SHALL record a `SESSION_REVOKED` entry in the `court_owner_audit_logs` table with the revoked session ID and device info

### Requirement 23: Profile Photo Upload API

**User Story:** As a court owner, I want to upload a profile photo or business logo, so that my business profile looks professional and recognizable to customers.

#### Acceptance Criteria

1. WHEN a court owner uploads a profile photo via `POST /api/users/me/profile-photo`, THE Platform_Service SHALL validate the file and store it in DigitalOcean Spaces
   - **Accepted formats:** JPEG, PNG, WebP
   - **Maximum file size:** 2 MB
   - **Storage path:** `profiles/{userId}/{uuid}.{ext}`
   - **Output (success):** `200 OK` with `{ profileImageUrl: string (CDN URL) }`
   - **Output (invalid format):** `400 Bad Request` with `{ error: "INVALID_IMAGE_FORMAT", acceptedFormats: ["JPEG", "PNG", "WEBP"] }`
   - **Output (file too large):** `400 Bad Request` with `{ error: "FILE_TOO_LARGE", maxSizeBytes: 2097152 }`
2. WHEN a profile photo is uploaded successfully, THE Platform_Service SHALL update the `profile_image_url` column in the `users` table with the CDN URL
3. WHEN a court owner removes their profile photo via `DELETE /api/users/me/profile-photo`, THE Platform_Service SHALL delete the file from DigitalOcean Spaces and set `profile_image_url` to null in the `users` table
   - **Output (success):** `204 No Content`
   - **Output (no photo):** `404 Not Found`
4. WHEN a new profile photo is uploaded and a previous photo exists, THE Platform_Service SHALL delete the previous photo from DigitalOcean Spaces before storing the new one
