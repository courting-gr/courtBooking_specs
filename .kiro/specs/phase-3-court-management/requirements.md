# Requirements Document — Phase 3: Court Management

## Introduction

Phase 3 implements the complete court management subsystem for the Court Booking Platform, targeting the `court-booking-platform-service`. This phase delivers court CRUD operations with image uploads, geospatial court discovery with PostGIS, availability window and override management, holiday calendar management (including Orthodox Easter calculation), court owner verification workflow, pricing rules and cancellation tiers, availability caching in Redis, weather forecast integration via OpenWeatherMap, customer favorites and preferences, court owner default settings, Kafka event publishing to the `court-update-events` topic, consumption of booking events from the `booking-events` Kafka topic for availability cache invalidation, and publishing of `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic for forward compatibility with Phase 5 (Notifications).

This phase builds on Phase 2 (Auth & User Management), which established OAuth authentication, JWT token issuance with role-based claims (including `verified`, `stripeConnected`, `subscriptionStatus` for COURT_OWNER), refresh token rotation, role-based access control enforcement, user profile management, and admin user management. Phase 1b established the database schema (including `courts`, `availability_windows`, `availability_overrides`, `holiday_templates`, `court_holiday_subscriptions`, `favorites`, `preferences`, `skill_levels`, `pricing_rules`, `cancellation_tiers`, `verification_requests`, `court_owner_audit_logs`, `court_defaults` tables), Flyway migrations, and CI/CD pipelines.

**Master requirements coverage:** Req 2 (Court Registration and Management), Req 3 (Location-Based Court Discovery), Req 4 (Customer Search and Booking User Journey — search/discovery aspects), Req 5 (User Personalization and Preferences), Req 6 (Weather Forecast Integration), Req 7 (Real-Time Availability Management — availability caching), Req 37 (Holiday Calendar Management).

**Subscription stub strategy:** Until Phase 10, all court owners default to `ACTIVE` subscription status. The `subscriptionStatus` JWT claim is included but subscription enforcement is deferred. Court owners can create and manage courts regardless of subscription status.

**Scope boundaries:**
- Court ratings and reviews (Req 2.19–2.20) are ⏳ Phase 10.
- Court ownership transfer (Req 2.29–2.32) is ⏳ Phase 10. Phase 3 does not implement any stub endpoints for court transfer — requests to transfer-related paths will return `404 Not Found`.
- Dynamic pricing configuration (Req 27.7–27.11) is ⏳ Phase 10, but `pricing_rules` table and base price multipliers ARE in scope.
- Promo code management is ⏳ Phase 10.
- Waitlist functionality is ⏳ Phase 10, but the `waitlist_enabled` field on courts IS in scope.
- Real-time WebSocket availability broadcasts (Req 7.2, 7.4, 7.5) are Phase 5 (Real-time & Notifications). Phase 3 implements the availability cache and REST API.
- Booking conflict checks for court deletion/updates coordinate with Transaction Service via cross-schema views (read-only access to `transaction.bookings`). Until Phase 4 deploys bookings, these checks return no conflicts.
- Stripe Connect onboarding is Phase 4. Phase 3 reads `stripe_connect_status` from the user record but does not manage Stripe Connect.
- Notification delivery (email, push, in-app) is Phase 5. Phase 3 publishes `NOTIFICATION_REQUESTED` events to Kafka for forward compatibility but they are not consumed until Phase 5.
- Security hardening (abuse detection, IP blocklist) is Phase 7.

## Glossary

- **Platform_Service**: The Spring Boot microservice responsible for authentication, authorization, user management, courts, availability, weather, analytics, and support. Deployed to `court-booking-platform-service`.
- **Transaction_Service**: The Spring Boot microservice responsible for bookings, payments, and notifications. Validates JWT tokens independently using the shared RS256 public key.
- **Court**: A bookable sports facility registered by a COURT_OWNER, with attributes including type, location, pricing, availability, and images. Stored in the `courts` table.
- **Court_Type**: One of TENNIS, PADEL, BASKETBALL, or FOOTBALL_5X5. Determines default duration and capacity.
- **Location_Type**: Either INDOOR or OUTDOOR. Affects weather relevance and search filtering.
- **Availability_Window**: A recurring weekly time slot during which a court is open for bookings. Stored in the `availability_windows` table.
- **Availability_Override**: A date-specific block that overrides the recurring schedule (maintenance, holidays, private events). Stored in the `availability_overrides` table. `start_time` and `end_time` are NULL for full-day blocks.
- **Holiday_Template**: A pre-defined national holiday or court-owner-created custom holiday with a date pattern (fixed or Easter-relative). Stored in the `holiday_templates` table.
- **Court_Holiday_Subscription**: A link between a court and a holiday template enabling automatic annual override generation. Stored in the `court_holiday_subscriptions` table.
- **Orthodox_Easter**: The date of Easter Sunday calculated per the Orthodox (Julian) calendar, which differs from Western Easter. Used as the reference point for moveable Greek national holidays.
- **Verification_Request**: A court owner's submission of business documents for platform verification. Stored in the `verification_requests` table with status PENDING_REVIEW, APPROVED, or REJECTED.
- **Pricing_Rule**: A time-based price multiplier applied to a court's base price for specific days and time ranges (e.g., peak/off-peak). Stored in the `pricing_rules` table.
- **Cancellation_Tier**: A time-based refund percentage tier defining the cancellation policy for a court. Stored in the `cancellation_tiers` table.
- **Court_Owner_Audit_Log**: An immutable record of court owner actions (court creation, pricing updates, availability changes). Stored in the `court_owner_audit_logs` table. Append-only.
- **Court_Defaults**: Default settings a court owner configures to be applied to newly created courts (duration, confirmation mode, cancellation tiers, amenities). Stored in the `court_defaults` table.
- **Spaces**: DigitalOcean Spaces object storage service used for court images and verification documents. Accessed via S3-compatible API with CDN distribution.
- **PostGIS**: PostgreSQL spatial extension enabling geospatial queries (radius search, bounding box, distance calculations) on court locations stored as `GEOMETRY(Point, 4326)`.
- **Availability_Cache**: Redis-backed cache of computed court availability slots with a 5-minute TTL. Invalidated on booking events (consumed from `booking-events` Kafka topic) and availability configuration changes.
- **Weather_Cache**: Redis-backed cache of weather forecast data from OpenWeatherMap with configurable TTL (default 10 minutes). Keyed by location coordinates, date, and time.
- **OpenWeatherMap**: External weather API provider used for forecast data (temperature, precipitation, wind, humidity). Supports 7-day forecast window.
- **Kafka_Court_Update_Events**: The `court-update-events` Kafka topic where Platform Service publishes court configuration changes (COURT_UPDATED, PRICING_UPDATED, AVAILABILITY_UPDATED, CANCELLATION_POLICY_UPDATED, COURT_DELETED). Consumed by Transaction Service.
- **Kafka_Notification_Events**: The `notification-events` Kafka topic where Platform Service publishes `NOTIFICATION_REQUESTED` events for verification status changes, SLA reminders, holiday application booking cancellations, and recurring holiday generation notifications. Partition key is `userId`. Not consumed until Phase 5.
- **Optimistic_Locking**: Concurrency control mechanism using a `version` column on the `courts` table. Updates include the current version; mismatches indicate concurrent modification.
- **EXIF_Stripping**: The process of removing EXIF metadata (GPS coordinates, camera info, timestamps) from uploaded images before storage, for privacy protection.

## Requirements


### Requirement 1: Court Registration and Creation

**User Story:** As a court owner, I want to register new sports courts with flexible configuration for type, location, duration, capacity, and pricing, so that customers can discover and book them.

#### Acceptance Criteria

1. WHEN a court owner submits a new court registration via `POST /api/courts`, THE Platform_Service SHALL validate court information and store it with geospatial coordinates in the `courts` table
   - **Input:** `{ nameEl: string, nameEn?: string, descriptionEl?: string, descriptionEn?: string, courtType: enum(TENNIS|PADEL|BASKETBALL|FOOTBALL_5X5), locationType: enum(INDOOR|OUTDOOR), address: string, latitude: number, longitude: number, durationMinutes: number, maxCapacity: number, basePriceCents: number, confirmationMode: enum(INSTANT|MANUAL), confirmationTimeoutHours?: number, waitlistEnabled: boolean, amenities?: string[], timezone?: string }`
   - `confirmationTimeoutHours` defaults to 24 when not provided (maps to `courts.confirmation_timeout_hours DEFAULT 24`)
   - **Output (success):** `201 Created` with `{ courtId: UUID, name: string, visible: boolean, version: 0, createdAt: ISO8601 }`
   - **Output (validation error):** `400 Bad Request` with `{ errors: [{ field: string, message: string }] }`
   - **Output (not court owner):** `403 Forbidden` with `{ error: "AUTHZ_INSUFFICIENT_ROLE" }`
2. WHEN a court owner creates a court and the court owner is not verified (`users.verified = false`), THE Platform_Service SHALL create the court with `visible = false` and return a `warning: "COURT_OWNER_NOT_VERIFIED"` field in the response
3. WHEN a court owner creates a court and the court owner is verified, THE Platform_Service SHALL set `visible = true` on the court
4. WHEN a court owner has configured court defaults in the `court_defaults` table, THE Platform_Service SHALL pre-populate new court fields (duration, confirmation mode, confirmation timeout, cancellation tiers, amenities) from the defaults when those fields are not explicitly provided in the request
5. THE Platform_Service SHALL validate that `latitude` is between -90 and 90 and `longitude` is between -180 and 180
6. THE Platform_Service SHALL validate that `basePriceCents` is a positive integer
7. THE Platform_Service SHALL validate that `durationMinutes` is a positive integer (typical values: 60, 90, 120)
8. THE Platform_Service SHALL validate that `maxCapacity` is a positive integer
9. THE Platform_Service SHALL store the court location as a PostGIS `GEOMETRY(Point, 4326)` using `ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)`
10. THE Platform_Service SHALL default `timezone` to `Europe/Athens` when not provided, and validate that the provided timezone is a valid IANA timezone identifier
11. WHEN a court is created, THE Platform_Service SHALL record a `COURT_CREATED` entry in the `court_owner_audit_logs` table with the full court details as the `changes` JSONB field
12. WHEN a court is created, THE Platform_Service SHALL publish a `COURT_UPDATED` event to the `court-update-events` Kafka topic with all court fields
13. THE Platform_Service SHALL validate that `amenities` values are from the standard amenities enum: `PARKING`, `SHOWERS`, `EQUIPMENT_RENTAL`, `LIGHTING`, `CHANGING_ROOMS`, `WIFI`, `CAFE`, `PRO_SHOP`. Invalid amenity values SHALL return `400 Bad Request` with `{ error: "INVALID_AMENITY", invalidValues: [string], validValues: [string] }`
14. THE Platform_Service SHALL validate bilingual content: `nameEl` is required and must be between 3-100 characters; `nameEn` is optional but if provided must be between 3-100 characters; `descriptionEl` and `descriptionEn` are optional but if provided must be between 10-2000 characters
15. WHEN `descriptionEl` or `descriptionEn` contains HTML content, THE Platform_Service SHALL sanitize the HTML to allow only safe tags (`<p>`, `<br>`, `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>`) and strip all other tags and attributes to prevent XSS attacks

### Requirement 2: Court Update and Modification

**User Story:** As a court owner, I want to update my court's details including name, type, pricing, and configuration, so that I can keep court information accurate and adjust settings as needed.

#### Acceptance Criteria

1. WHEN a court owner submits a court update via `PUT /api/courts/{courtId}`, THE Platform_Service SHALL validate changes and apply them using optimistic locking
   - **Input:** Partial update body (same fields as create, all optional) plus `version: number` for optimistic locking
   - **Output (success):** `200 OK` with updated court object including incremented `version` number
   - **Output (version conflict):** `409 Conflict` with `{ error: "CONCURRENT_MODIFICATION", currentVersion: number }`
2. WHEN a court update changes fields that affect future bookings (courtType, durationMinutes, maxCapacity), THE Platform_Service SHALL check for future confirmed bookings via the `transaction.bookings` cross-schema view and return `422 Unprocessable Entity` with `{ error: "CHANGE_AFFECTS_BOOKINGS", affectedBookings: [{ bookingId, date, impact: string }], requiresConfirmation: true }` if conflicts exist
3. WHEN a court update that affects bookings is re-submitted with `{ confirmed: true }`, THE Platform_Service SHALL apply the changes
4. WHEN a court owner attempts to update a court they do not own, THE Platform_Service SHALL return `403 Forbidden` with `{ error: "AUTHZ_INSUFFICIENT_ROLE" }`
5. WHEN a court is updated, THE Platform_Service SHALL increment the `version` column and update the `updated_at` timestamp
6. WHEN a court is updated, THE Platform_Service SHALL record the changes in the `court_owner_audit_logs` table with before/after snapshots in the `changes` JSONB field
7. WHEN a court is updated, THE Platform_Service SHALL publish a `COURT_UPDATED` event to the `court-update-events` Kafka topic with the list of changed fields
8. WHEN a court's `basePriceCents` is updated, THE Platform_Service SHALL additionally publish a `PRICING_UPDATED` event to the `court-update-events` Kafka topic
9. WHEN a court owner updates the `confirmationMode` via `PUT /api/courts/{courtId}/confirmation-mode`, THE Platform_Service SHALL apply the change only to future bookings and publish a `COURT_UPDATED` event
10. WHEN a court owner updates the `waitlistEnabled` flag via `PUT /api/courts/{courtId}/waitlist-config`, THE Platform_Service SHALL update the field and publish a `COURT_UPDATED` event

### Requirement 3: Court Deletion

**User Story:** As a court owner, I want to remove courts that are no longer in use, so that customers do not see outdated listings.

#### Acceptance Criteria

1. WHEN a court owner submits a court deletion via `DELETE /api/courts/{courtId}`, THE Platform_Service SHALL validate that no future confirmed bookings exist before allowing deletion
   - **Output (success):** `204 No Content`
   - **Output (has future bookings):** `409 Conflict` with `{ error: "COURT_HAS_FUTURE_BOOKINGS", bookingCount: number, nextBookingDate: ISO8601 }`
   - **Output (has pending payouts):** `409 Conflict` with `{ error: "COURT_HAS_PENDING_PAYOUTS", pendingAmount: number }`
2. WHEN a court is deleted, THE Platform_Service SHALL remove the court record and cascade-delete related `availability_windows`, `availability_overrides`, `pricing_rules`, `cancellation_tiers`, and `court_holiday_subscriptions` entries (via ON DELETE CASCADE), and delete associated images from DigitalOcean Spaces (see Requirement 4 AC6)
3. WHEN a court is deleted, THE Platform_Service SHALL record a `COURT_DELETED` entry in the `court_owner_audit_logs` table
4. WHEN a court is deleted, THE Platform_Service SHALL publish a `COURT_DELETED` event to the `court-update-events` Kafka topic
5. WHEN a court is deleted, THE Platform_Service SHALL invalidate the availability cache entries for that court in Redis
6. WHEN a court owner attempts to delete a court they do not own, THE Platform_Service SHALL return `403 Forbidden`


### Requirement 4: Court Image Management

**User Story:** As a court owner, I want to upload and manage photos of my courts, so that customers can see what the facilities look like before booking.

#### Acceptance Criteria

1. WHEN a court owner uploads images via `POST /api/courts/{courtId}/images`, THE Platform_Service SHALL validate file formats and sizes before storing them in DigitalOcean Spaces
   - **Accepted formats:** JPEG, PNG, WebP
   - **Maximum file size:** 10 MB per image
   - **Maximum images per upload:** 10
   - **Output (success):** `200 OK` with `{ imageUrls: [string] }` containing CDN URLs
   - **Output (invalid format):** `400 Bad Request` with `{ error: "INVALID_IMAGE_FORMAT", acceptedFormats: ["JPEG", "PNG", "WEBP"] }`
   - **Output (file too large):** `400 Bad Request` with `{ error: "IMAGE_TOO_LARGE", maxSizeBytes: 10485760 }`
2. WHEN images are uploaded, THE Platform_Service SHALL strip EXIF metadata (GPS coordinates, camera info, timestamps) from the images before storing them in Spaces
3. WHEN images are stored in Spaces, THE Platform_Service SHALL generate CDN-accessible URLs and append them to the court's `image_urls` array
4. THE Platform_Service SHALL store images in Spaces under the path `courts/{courtId}/{uuid}.{extension}`
5. WHEN a court owner attempts to upload images for a court they do not own, THE Platform_Service SHALL return `403 Forbidden`
6. WHEN a court is deleted, THE Platform_Service SHALL delete associated images from Spaces
7. WHEN a court owner deletes an image via `DELETE /api/courts/{courtId}/images/{imageId}`, THE Platform_Service SHALL remove the image from DigitalOcean Spaces and update the court's `image_urls` array
   - **Output (success):** `204 No Content`
   - **Output (image not found):** `404 Not Found` with `{ error: "IMAGE_NOT_FOUND" }`
   - **Output (not owner):** `403 Forbidden`
8. WHEN a court owner reorders images via `PUT /api/courts/{courtId}/images/order`, THE Platform_Service SHALL update the `image_urls` array to reflect the new order
   - **Input:** `{ imageUrls: [string] }` — the complete list of image URLs in the desired order
   - **Output (success):** `200 OK` with `{ imageUrls: [string] }`
   - **Output (invalid URLs):** `400 Bad Request` with `{ error: "INVALID_IMAGE_URLS", message: "Provided URLs do not match existing court images" }`
9. THE Platform_Service SHALL enforce a maximum of 20 images per court. Attempts to upload beyond this limit SHALL return `400 Bad Request` with `{ error: "MAX_IMAGES_EXCEEDED", maxImages: 20, currentCount: number }`

### Requirement 5: Court Type Configuration

**User Story:** As a court owner, I want to configure court-type-specific defaults for duration and capacity, so that bookings are pre-configured with sensible values for each sport.

#### Acceptance Criteria

1. THE Platform_Service SHALL support the following court types with default configurations:
   - TENNIS: 60 minutes duration, 4 max capacity
   - PADEL: 90 minutes duration, 4 max capacity
   - BASKETBALL: 60 minutes duration, 10 max capacity
   - FOOTBALL_5X5: 60 minutes duration, 10 max capacity
2. WHEN a court owner creates a court without specifying `durationMinutes`, THE Platform_Service SHALL apply the default duration for the selected court type
3. WHEN a court owner creates a court without specifying `maxCapacity`, THE Platform_Service SHALL apply the default capacity for the selected court type
4. WHEN a court owner specifies `durationMinutes` or `maxCapacity` explicitly, THE Platform_Service SHALL use the provided values, overriding the court type defaults
5. WHEN a court owner changes the `courtType` of an existing court, THE Platform_Service SHALL validate that existing future bookings are compatible with the new type's constraints and warn the court owner of any incompatibilities

### Requirement 6: Availability Window Management

**User Story:** As a court owner, I want to define recurring weekly availability schedules for my courts, so that customers know when courts are open for booking.

#### Acceptance Criteria

1. WHEN a court owner retrieves availability windows via `GET /api/courts/{courtId}/availability/windows`, THE Platform_Service SHALL return all recurring weekly time slots for the court
2. WHEN a court owner updates availability windows via `PUT /api/courts/{courtId}/availability/windows`, THE Platform_Service SHALL replace the existing windows with the provided set
   - **Input:** `{ windows: [{ dayOfWeek: enum(MONDAY..SUNDAY), startTime: HH:mm, endTime: HH:mm }] }`
   - **Output (success):** `200 OK` with the updated windows
   - **Output (overlapping):** `400 Bad Request` with `{ error: "OVERLAPPING_WINDOWS", conflicts: [{ dayOfWeek, window1, window2 }] }`
3. THE Platform_Service SHALL validate that `endTime` is after `startTime` for each availability window
4. THE Platform_Service SHALL validate that no two availability windows overlap for the same court and day of week, enforced by the `uq_availability_no_overlap` EXCLUDE constraint
5. WHEN availability windows are updated, THE Platform_Service SHALL invalidate the availability cache in Redis for the affected court
6. WHEN availability windows are updated, THE Platform_Service SHALL publish an `AVAILABILITY_UPDATED` event to the `court-update-events` Kafka topic with `changeType: "RECURRING_WINDOWS_UPDATED"`
7. WHEN availability windows are updated, THE Platform_Service SHALL record the changes in the `court_owner_audit_logs` table

### Requirement 7: Availability Override Management

**User Story:** As a court owner, I want to block specific dates or time ranges for maintenance, private events, or holidays, so that customers cannot book during those periods.

#### Acceptance Criteria

1. WHEN a court owner creates an availability override via `POST /api/courts/{courtId}/availability/overrides`, THE Platform_Service SHALL create an entry in the `availability_overrides` table
   - **Input:** `{ date: ISO8601-date, startTime?: HH:mm, endTime?: HH:mm, reason?: string }`
   - **Output (success):** `201 Created` with the override details including generated `id`
   - When `startTime` and `endTime` are NULL, the override blocks the entire day
2. WHEN a court owner retrieves overrides via `GET /api/courts/{courtId}/availability/overrides`, THE Platform_Service SHALL return overrides filtered by optional `fromDate` and `toDate` query parameters
3. WHEN a court owner deletes an override via `DELETE /api/courts/{courtId}/availability/overrides/{overrideId}`, THE Platform_Service SHALL remove the override entry
4. THE Platform_Service SHALL prevent duplicate `availability_override` entries for the same court, date, and time range
5. WHEN an override is created with `source = MANUAL`, THE Platform_Service SHALL set the `source` column to `MANUAL` and leave `holiday_template_id` as NULL
6. WHEN an availability override is created or deleted, THE Platform_Service SHALL invalidate the availability cache in Redis for the affected court and date
7. WHEN an availability override is created or deleted, THE Platform_Service SHALL publish an `AVAILABILITY_UPDATED` event to the `court-update-events` Kafka topic with `changeType: "OVERRIDE_CREATED"` or `"OVERRIDE_DELETED"` and the affected date range
8. WHEN an availability override is created or deleted, THE Platform_Service SHALL record the change in the `court_owner_audit_logs` table


### Requirement 8: Holiday Calendar — National Holiday Templates

**User Story:** As a court owner, I want to see pre-defined Greek national holidays with their dates calculated automatically, so that I can easily block my courts for public holidays without manual date entry.

#### Greek National Holidays (Pre-defined)

| Holiday | Date Pattern | Type |
|---------|--------------|------|
| New Year's Day | January 1 | Fixed |
| Epiphany | January 6 | Fixed |
| Clean Monday | 48 days before Orthodox Easter | Moveable |
| Independence Day | March 25 | Fixed |
| Orthodox Good Friday | Friday before Orthodox Easter (EASTER_OFFSET:-2) | Moveable |
| Orthodox Easter Sunday | Calculated per Orthodox calendar (EASTER_OFFSET:0) | Moveable |
| Orthodox Easter Monday | Day after Orthodox Easter (EASTER_OFFSET:1) | Moveable |
| Labour Day | May 1 | Fixed |
| Whit Monday (Holy Spirit Monday) | 50 days after Orthodox Easter (EASTER_OFFSET:50) | Moveable |
| Assumption of Mary | August 15 | Fixed |
| Ochi Day | October 28 | Fixed |
| Christmas Day | December 25 | Fixed |
| Second Day of Christmas | December 26 | Fixed |

#### Acceptance Criteria

1. THE Platform_Service SHALL provide a pre-defined registry of Greek national holidays stored in the `holiday_templates` table with `is_national = true` and `owner_id = NULL`
2. THE Platform_Service SHALL calculate moveable holiday dates based on Orthodox Easter using the `date_pattern` column:
   - `FIXED:MM-DD` for fixed-date holidays (e.g., `FIXED:12-25` for Christmas)
   - `EASTER_OFFSET:N` for Easter-relative holidays (e.g., `EASTER_OFFSET:0` for Easter Sunday, `EASTER_OFFSET:-48` for Clean Monday, `EASTER_OFFSET:50` for Whit Monday)
3. THE Platform_Service SHALL implement the Orthodox Easter calculation algorithm (Meeus Julian algorithm) to compute Easter Sunday for the current year and up to 2 years in advance
4. WHEN a user requests national holidays via `GET /api/holidays/national`, THE Platform_Service SHALL return all active national holiday templates with their calculated dates for the current year and the next 2 years
   - **Output:** `{ holidays: [{ id: UUID, nameEl: string, nameEn: string, datePattern: string, dates: { "2025": "2025-04-20", "2026": "2026-04-12", "2027": "2027-05-02" } }] }`
   - This endpoint is public (no authentication required)
5. THE Platform_Service SHALL allow PLATFORM_ADMIN users to add, modify, or disable national holiday templates via admin API endpoints
6. THE Platform_Service SHALL seed the 13 Greek national holidays listed above as part of a Flyway migration (`V3__seed_national_holidays.sql`)

### Requirement 9: Holiday Calendar — Custom Holidays

**User Story:** As a court owner, I want to create my own custom holidays (e.g., annual maintenance week, local festivals), so that I can manage recurring closures specific to my business.

#### Acceptance Criteria

1. WHEN a court owner creates a custom holiday via `POST /api/holidays/custom`, THE Platform_Service SHALL store it in the `holiday_templates` table with `is_national = false` and `owner_id` set to the court owner's user ID
   - **Input:** `{ nameEl: string, nameEn?: string, datePattern: string, timeRange?: { startTime: HH:mm, endTime: HH:mm } }`
   - **Output (success):** `201 Created` with the custom holiday details
   - **Output (duplicate name):** `409 Conflict` with `{ error: "HOLIDAY_NAME_EXISTS" }`
2. THE Platform_Service SHALL validate that custom holiday names are unique per court owner, enforced by the `uq_holiday_templates_owner_name` unique constraint on `(owner_id, name_el)`
3. WHEN a court owner retrieves custom holidays via `GET /api/holidays/custom`, THE Platform_Service SHALL return all custom holidays owned by the authenticated court owner with calculated dates
4. WHEN a court owner updates a custom holiday via `PUT /api/holidays/custom/{holidayId}`, THE Platform_Service SHALL validate ownership and apply the changes
5. WHEN a court owner deletes a custom holiday via `DELETE /api/holidays/custom/{holidayId}`, THE Platform_Service SHALL prompt for override handling via the `deleteOverrides` query parameter:
   - `KEEP`: Delete the holiday template but keep existing `availability_overrides` entries
   - `DELETE_ALL`: Delete the holiday template and all related `availability_overrides` entries
   - `DELETE_FUTURE`: Delete the holiday template and only future-dated `availability_overrides` entries
6. THE Platform_Service SHALL support two recurrence patterns for custom holidays:
   - Fixed date: `FIXED:MM-DD` (same date every year)
   - Easter-relative: `EASTER_OFFSET:N` (calculated based on Orthodox Easter)
7. WHEN a court owner attempts to modify or delete a custom holiday they do not own, THE Platform_Service SHALL return `403 Forbidden`


### Requirement 10: Holiday Calendar — Bulk Holiday Application

**User Story:** As a court owner managing multiple courts, I want to apply holidays to several courts at once, so that I can efficiently manage seasonal closures without repeating the process for each court.

#### Acceptance Criteria

1. WHEN a court owner submits a bulk holiday application via `POST /api/holidays/apply`, THE Platform_Service SHALL create `availability_overrides` entries for each selected court and holiday combination
   - **Input:** `{ holidayIds: [UUID], courtIds: [UUID], years: [number], conflictResolution?: enum(SKIP|CANCEL_BOOKINGS|ABORT) }`
   - **Output (success):** `200 OK` with `{ courtsAffected: number, holidaysApplied: number, overridesCreated: number }`
   - **Output (conflicts):** `409 Conflict` with `{ conflicts: [{ courtId, courtName, date, bookings: [{ bookingId, customerName, startTime, endTime, amountCents }] }] }`
2. WHEN applying holidays in bulk, THE Platform_Service SHALL process all courts atomically — either all succeed or none are modified (database transaction)
3. WHEN a bulk application encounters conflicts with existing confirmed bookings, THE Platform_Service SHALL report the conflicts and allow the court owner to choose:
   - `SKIP`: Apply holiday to non-conflicting dates only
   - `CANCEL_BOOKINGS`: Cancel conflicting bookings with full refund (publishes cancellation events to Transaction Service)
   - `ABORT`: Abort the entire operation
4. WHEN holidays are applied to courts, THE Platform_Service SHALL create `court_holiday_subscriptions` entries linking each court to the holiday template with `auto_renew = true` by default
5. WHEN holidays are applied, THE Platform_Service SHALL set the `source` column on created `availability_overrides` to `HOLIDAY_NATIONAL` or `HOLIDAY_CUSTOM` and set `holiday_template_id` to the template's ID
6. WHEN holidays are applied, THE Platform_Service SHALL set the `reason` field on created `availability_overrides` to the holiday name (in the court owner's preferred language)
7. THE Platform_Service SHALL prevent duplicate `availability_override` entries for the same court, date, and time range when applying holidays
8. WHEN a court owner views holidays applied to a specific court via `GET /api/courts/{courtId}/holidays`, THE Platform_Service SHALL return all holiday subscriptions for that court with calculated dates
9. WHEN a court owner removes a holiday from a court via `DELETE /api/courts/{courtId}/holidays/{holidayId}`, THE Platform_Service SHALL remove the `court_holiday_subscriptions` entry and handle overrides per the `deleteOverrides` query parameter (default: `DELETE_FUTURE`)
10. THE Platform_Service SHALL NOT allow holiday application to dates with bookings in `PENDING_CONFIRMATION` status without first resolving those bookings
11. WHEN bookings are cancelled due to holiday application (via `conflictResolution: CANCEL_BOOKINGS`), THE Platform_Service SHALL publish `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic to notify affected customers with the cancellation reason, holiday name, and refund amount

### Requirement 11: Holiday Calendar — Recurring Annual Holidays

**User Story:** As a court owner, I want holidays to automatically recur each year without manual re-application, so that I do not need to remember to block courts for the same holidays every year.

#### Acceptance Criteria

1. WHEN a court owner applies a recurring holiday, THE Platform_Service SHALL create `availability_overrides` entries for the current year and optionally for future years up to a configurable horizon (default: 2 years ahead)
2. THE Platform_Service SHALL implement a scheduled job that generates holiday instances for the new year based on active `court_holiday_subscriptions` with `auto_renew = true`
3. WHEN the scheduled job generates new holiday instances, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic to notify the court owner with a summary of generated holiday dates
4. THE Platform_Service SHALL allow court owners to opt-out of automatic recurrence for specific holidays by setting `auto_renew = false` on the `court_holiday_subscriptions` entry
5. WHEN a recurring holiday template is modified, THE Platform_Service SHALL prompt whether to update only future override instances or leave existing instances unchanged
6. THE Platform_Service SHALL track which `availability_overrides` entries were created from recurring holidays via the `holiday_template_id` foreign key, enabling bulk updates or deletions
7. WHEN a holiday-generated override is manually edited by the court owner, THE Platform_Service SHALL change its `source` to `MANUAL` and set `holiday_template_id` to NULL, excluding it from future bulk holiday updates


### Requirement 12: Holiday Calendar — Calendar View and Conflict Detection

**User Story:** As a court owner, I want a calendar view showing all blocked dates across my courts with conflict detection, so that I can manage holidays and overrides without accidentally disrupting customer bookings.

#### Acceptance Criteria

1. WHEN a court owner requests the holiday calendar via `GET /api/holidays/calendar`, THE Platform_Service SHALL return calendar data for the specified date range showing all blocked dates across the court owner's courts
   - **Input (query params):** `fromDate: ISO8601-date, toDate: ISO8601-date, courtId?: UUID`
   - **Output:** `{ entries: [{ date: ISO8601-date, courtId: UUID, courtName: string, source: enum(MANUAL|HOLIDAY_NATIONAL|HOLIDAY_CUSTOM), holidayName?: string, startTime?: HH:mm, endTime?: HH:mm, hasBookings: boolean }] }`
2. THE Platform_Service SHALL differentiate between national holidays (`source: HOLIDAY_NATIONAL`), custom holidays (`source: HOLIDAY_CUSTOM`), and manual overrides (`source: MANUAL`) in the calendar response
3. WHEN a court owner has multiple courts, THE Platform_Service SHALL support filtering the calendar by individual court via the `courtId` query parameter, or return all courts aggregated when no filter is applied
4. THE Platform_Service SHALL include a `hasBookings` flag on each calendar entry indicating whether existing customer bookings exist on that date for that court. This flag is resolved via the `transaction.bookings` cross-schema view (read-only access). Until Phase 4 deploys bookings, `hasBookings` SHALL always return `false`.
5. WHEN a court owner attempts to apply a holiday that conflicts with existing confirmed bookings, THE Platform_Service SHALL display the list of conflicting bookings with customer names, dates, times, and amounts in the conflict response
6. WHEN bookings are cancelled due to holiday application (via `conflictResolution: CANCEL_BOOKINGS`), THE Platform_Service SHALL publish `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic to notify affected customers with the cancellation reason and refund amount

### Requirement 13: Location-Based Court Discovery

**User Story:** As a customer, I want to discover courts near my location using map-based search with filters, so that I can find convenient booking options.

#### Acceptance Criteria

1. WHEN a customer searches for courts via `GET /api/courts`, THE Platform_Service SHALL execute geospatial queries using PostGIS `ST_DWithin` to find courts within the specified radius
   - **Required params:** `latitude: number, longitude: number`
   - **Optional params:** `radiusKm: number (default 10), courtType: enum, locationType: enum, date: ISO8601-date, startTime: HH:mm, minLat/maxLat/minLng/maxLng: number (map bounds), page: number, size: number (default 50, max 200)`
   - **Output:** `{ courts: [CourtSummary], totalCount: number, page: number, size: number }`
   - THE Platform_Service SHALL enforce a maximum page size of 200 to prevent abuse. Requests with `size > 200` SHALL be clamped to 200.
2. WHEN search results are returned, THE Platform_Service SHALL include distance calculations (in kilometers) from the search coordinates using PostGIS `ST_Distance` with geography cast
3. WHEN map bounds parameters (`minLat`, `maxLat`, `minLng`, `maxLng`) are provided, THE Platform_Service SHALL use `ST_MakeEnvelope` to filter courts within the visible map region instead of radius search
4. WHEN a `date` parameter is provided, THE Platform_Service SHALL filter results to only include courts that have available time slots on that date (checking availability windows minus overrides minus existing bookings)
5. WHEN a `startTime` parameter is provided along with `date`, THE Platform_Service SHALL filter results to only include courts that have an available slot starting at or after the specified `startTime` on that date
6. WHEN a `courtType` filter is applied, THE Platform_Service SHALL filter results to courts matching the specified type
7. WHEN a `locationType` filter is applied, THE Platform_Service SHALL filter results to courts matching INDOOR or OUTDOOR
8. THE Platform_Service SHALL only return courts with `visible = true` in search results
9. WHEN the search returns zero results, THE Platform_Service SHALL return an empty results array with a `suggestion` field: `{ courts: [], suggestion: "EXPAND_RADIUS" | "REMOVE_FILTERS" | "TRY_DIFFERENT_DATE" }` based on which constraint is most restrictive
10. WHEN the search request includes invalid coordinates (latitude outside -90 to 90, longitude outside -180 to 180), THE Platform_Service SHALL return `400 Bad Request` with `{ error: "INVALID_COORDINATES", message: "Latitude must be between -90 and 90, longitude between -180 and 180" }`
11. THE Platform_Service SHALL prioritize favorite courts in search results for authenticated customers by including an `isFavorite: boolean` field on each court in the response
12. THE Platform_Service SHALL support sorting search results via `sortBy` and `sortOrder` query parameters:
    - `sortBy`: `distance` (default), `price`, `rating`, `name`
    - `sortOrder`: `asc` (default for distance, price), `desc` (default for rating)
    - When `sortBy=distance`, results are sorted by distance from the search coordinates
    - When `sortBy=price`, results are sorted by `basePriceCents`
    - When `sortBy=rating`, results are sorted by `averageRating` (null values sorted last)
    - When `sortBy=name`, results are sorted alphabetically by the localized name


### Requirement 14: Aggregated Map Endpoint

**User Story:** As a customer using the map view, I want a single API call that returns courts, open matches, and weather data, so that the app can render the map efficiently without multiple round trips.

#### Acceptance Criteria

1. WHEN a customer requests the aggregated map endpoint via `GET /api/courts/map`, THE Platform_Service SHALL return courts, open matches (stub: empty array until Phase 10), and weather data in a single response
   - **Required params:** `latitude: number, longitude: number, date: ISO8601-date`
   - **Optional params:** `radiusKm: number, minLat/maxLat/minLng/maxLng: number, courtType: enum, locationType: enum, startTime: HH:mm`
   - **Output:** `{ courts: [CourtMapMarker], openMatches: [MatchMapMarker] | null, weather: WeatherSummary | null, warnings: [string] }`
2. WHEN the map endpoint partially fails (e.g., courts load but weather fails), THE Platform_Service SHALL return the successful data with `null` for failed sections and include a `warnings` array: `{ courts: [...], openMatches: null, weather: null, warnings: ["WEATHER_UNAVAILABLE", "MATCHES_UNAVAILABLE"] }`
3. THE Platform_Service SHALL return court map markers with minimal data for efficient rendering: `{ courtId, latitude, longitude, courtType, locationType, name, basePriceCents, averageRating, isFavorite }`. Note: `averageRating` SHALL be `null` until Phase 10 (Court Ratings) is implemented.
4. THE Platform_Service SHALL support the same geospatial filtering (radius, map bounds, court type, location type) as the court search endpoint

### Requirement 15: Court Detail Retrieval

**User Story:** As a customer, I want to view detailed information about a specific court including images, amenities, availability, pricing, and cancellation policy, so that I can make an informed booking decision.

#### Acceptance Criteria

1. WHEN a customer requests court details via `GET /api/courts/{courtId}`, THE Platform_Service SHALL return the full court object including images, amenities, pricing rules, cancellation tiers, and availability summary
   - **Output:** `{ court: CourtDetail, availabilityWindows: [AvailabilityWindow], pricingRules: [PricingRule], cancellationTiers: [CancellationTier], weather?: WeatherSummary, isFavorite: boolean }`
2. WHEN the court detail is requested with a `date` query parameter, THE Platform_Service SHALL include available time slots for that date computed from availability windows minus overrides minus existing bookings
3. WHEN the court is OUTDOOR and a `date` parameter is provided within the 7-day forecast window, THE Platform_Service SHALL include weather forecast data for that date
4. THE Platform_Service SHALL return `404 Not Found` when the requested court does not exist or has `visible = false` (for non-owner requests)
5. WHEN the requesting user is the court owner, THE Platform_Service SHALL return the court detail regardless of `visible` status
6. THE Platform_Service SHALL include the `isFavorite` field for authenticated customers (default `false` for unauthenticated requests)
7. THE Platform_Service SHALL include cancellation tier details sorted by `sort_order` so the client can display the cancellation policy summary

### Requirement 16: Court Owner Court Listing

**User Story:** As a court owner, I want to see all my registered courts with their status and key metrics, so that I can manage my portfolio efficiently.

#### Acceptance Criteria

1. WHEN a court owner requests their courts via `GET /api/courts/mine`, THE Platform_Service SHALL return all courts owned by the authenticated court owner regardless of `visible` status
   - **Output:** `{ courts: [{ courtId: UUID, nameEl: string, nameEn?: string, courtType: enum, locationType: enum, visible: boolean, basePriceCents: number, createdAt: ISO8601, updatedAt: ISO8601 }] }`
2. THE Platform_Service SHALL support optional filtering by `courtType` and `locationType` query parameters
3. THE Platform_Service SHALL support sorting by `name`, `createdAt`, `updatedAt`, or `basePriceCents` via the `sortBy` and `sortOrder` query parameters
4. THE Platform_Service SHALL include a `visible` field on each court indicating whether it is publicly discoverable
5. WHEN a court owner has no courts, THE Platform_Service SHALL return an empty array with `200 OK`


### Requirement 17: Court Owner Verification Workflow

**User Story:** As a court owner, I want to submit my business documents for verification, so that my courts become publicly visible and customers can discover them.

#### Acceptance Criteria

1. WHEN a court owner submits verification documents via `POST /api/verification/submit`, THE Platform_Service SHALL create a verification request with status `PENDING_REVIEW`
   - **Input:** `{ businessName: string, taxId: string, businessType: enum(SOLE_PROPRIETOR|COMPANY|ASSOCIATION), businessAddress: string, proofDocument: File (PDF/JPEG/PNG, max 10MB) }`
   - **Output (success):** `201 Created` with `{ verificationRequestId: UUID, status: "PENDING_REVIEW", submittedAt: ISO8601 }`
   - **Output (already pending):** `409 Conflict` with `{ error: "VERIFICATION_ALREADY_PENDING", existingRequestId: UUID }`
2. THE Platform_Service SHALL store the proof document in DigitalOcean Spaces under the path `verification/{courtOwnerId}/{uuid}.{extension}`
3. WHEN a verification request is submitted, THE Platform_Service SHALL publish a `NOTIFICATION_REQUESTED` event to the `notification-events` Kafka topic to notify PLATFORM_ADMIN users of the new pending request
4. WHEN a PLATFORM_ADMIN approves a verification request via `POST /api/admin/verifications/{requestId}/approve`, THE Platform_Service SHALL update the court owner's `verified` status to `true`, set all their courts to `visible = true`, and publish a `NOTIFICATION_REQUESTED` event to notify the court owner
   - **Input:** `{ notes?: string }`
   - **Output:** `200 OK` with `{ requestId: UUID, status: "APPROVED", courtOwnerId: UUID, approvedAt: ISO8601 }`
5. WHEN a PLATFORM_ADMIN rejects a verification request via `POST /api/admin/verifications/{requestId}/reject`, THE Platform_Service SHALL update the request status and publish a `NOTIFICATION_REQUESTED` event to notify the court owner with the rejection reason
   - **Input:** `{ reason: string }` (required)
   - **Output:** `200 OK` with `{ requestId: UUID, status: "REJECTED", reason: string, rejectedAt: ISO8601 }`
6. WHEN a court owner re-submits after rejection, THE Platform_Service SHALL create a new verification request with `previous_request_id` linking to the rejected request so reviewers can see the history
7. THE Platform_Service SHALL provide a verification queue endpoint for PLATFORM_ADMIN users via `GET /api/admin/verifications?status=PENDING_REVIEW` returning all pending requests sorted by submission date
8. THE Platform_Service SHALL implement a verification SLA reminder: WHEN a verification request has been pending for more than 48 hours, a `NOTIFICATION_REQUESTED` event SHALL be published to notify PLATFORM_ADMIN users. This is implemented as a Spring `@Scheduled` job within Platform Service (not Quartz, which is in Transaction Service).
9. WHEN a court owner updates business information that was part of the original verification (business name, tax ID, business address), THE Platform_Service SHALL flag the profile for re-verification review without revoking current verified status — courts remain visible during re-review
10. THE Platform_Service SHALL record all verification status changes in the `court_owner_audit_logs` table
11. THE Platform_Service SHALL support pagination for the admin verification queue via `GET /api/admin/verifications` with query parameters `page: number (default 0)`, `size: number (default 20, max 100)`, returning `{ requests: [VerificationRequest], totalCount: number, page: number, size: number }`
12. WHEN a court owner needs to replace a verification document, THE Platform_Service SHALL support `DELETE /api/verification/documents/{documentId}` to remove the existing document before uploading a new one
    - **Output (success):** `204 No Content`
    - **Output (document not found):** `404 Not Found`
    - **Output (verification already approved):** `409 Conflict` with `{ error: "VERIFICATION_ALREADY_APPROVED" }`

### Requirement 18: Pricing Rules Management

**User Story:** As a court owner, I want to configure time-based pricing rules with peak/off-peak multipliers, so that I can optimize revenue based on demand patterns.

#### Acceptance Criteria

1. WHEN a court owner creates pricing rules via `POST /api/courts/{courtId}/pricing-rules`, THE Platform_Service SHALL store them in the `pricing_rules` table
   - **Input:** `{ rules: [{ dayOfWeek: enum(MONDAY..SUNDAY), startTime: HH:mm, endTime: HH:mm, multiplier: number, label?: string }] }`
   - **Output (success):** `200 OK` with the created pricing rules
   - **Output (overlapping):** `400 Bad Request` with `{ error: "OVERLAPPING_PRICING_RULES", conflicts: [{ dayOfWeek, rule1, rule2 }] }`
2. THE Platform_Service SHALL validate that `multiplier` is between 0.10 and 5.00 (10% to 500% of base price)
3. THE Platform_Service SHALL validate that pricing rules do not overlap for the same court and day of week
4. WHEN a court owner retrieves pricing rules via `GET /api/courts/{courtId}/pricing-rules`, THE Platform_Service SHALL return all pricing rules for the court
5. WHEN a court owner updates pricing rules via `PUT /api/courts/{courtId}/pricing-rules`, THE Platform_Service SHALL replace the existing rules with the provided set
6. WHEN pricing rules are created or updated, THE Platform_Service SHALL publish a `PRICING_UPDATED` event to the `court-update-events` Kafka topic
7. WHEN pricing rules are created or updated, THE Platform_Service SHALL record the changes in the `court_owner_audit_logs` table
8. THE Platform_Service SHALL calculate the effective price for a given time slot as: `basePriceCents * multiplier` (rounded to nearest integer). If no pricing rule matches the time slot, the base price applies (multiplier = 1.00)

### Requirement 19: Cancellation Policy Management

**User Story:** As a court owner, I want to configure time-based cancellation tiers with different refund percentages, so that I can balance customer flexibility with revenue protection.

#### Acceptance Criteria

1. WHEN a court owner configures cancellation tiers via `PUT /api/courts/{courtId}/cancellation-tiers`, THE Platform_Service SHALL store them in the `cancellation_tiers` table
   - **Input:** `{ tiers: [{ thresholdHours: number, refundPercent: number }] }`
   - **Output (success):** `200 OK` with the configured tiers sorted by `thresholdHours` descending
   - **Output (invalid):** `400 Bad Request` with `{ error: "INVALID_CANCELLATION_TIERS", message: string }`
2. THE Platform_Service SHALL validate that `refundPercent` is between 0 and 100
3. THE Platform_Service SHALL validate that `thresholdHours` values are unique per court (enforced by `uq_cancellation_tiers_court_threshold` unique constraint)
4. THE Platform_Service SHALL validate that refund percentages decrease as threshold hours decrease (e.g., 24h+ = 100%, 12-24h = 50%, <12h = 0%)
5. WHEN a court owner retrieves cancellation tiers via `GET /api/courts/{courtId}/cancellation-tiers`, THE Platform_Service SHALL return all tiers sorted by `sort_order`
6. WHEN cancellation tiers are updated, THE Platform_Service SHALL publish a `CANCELLATION_POLICY_UPDATED` event to the `court-update-events` Kafka topic
7. WHEN cancellation tiers are updated, THE Platform_Service SHALL record the changes in the `court_owner_audit_logs` table
8. WHEN a court has no cancellation tiers configured, THE Platform_Service SHALL apply a default policy of 100% refund (full refund regardless of timing)


### Requirement 20: Availability Caching

**User Story:** As a customer, I want fast availability checks when searching for courts, so that I can quickly see which time slots are open without waiting for complex database queries.

#### Acceptance Criteria

1. THE Platform_Service SHALL cache computed availability slots in Redis with a 5-minute TTL
2. WHEN a customer requests availability for a court and date via `GET /api/courts/{courtId}/availability?date={date}`, THE Platform_Service SHALL first check the Redis cache before computing availability from the database
   - **Output:** `{ courtId: UUID, date: ISO8601-date, slots: [{ startTime: HH:mm, endTime: HH:mm, status: enum(AVAILABLE|BOOKED|BLOCKED), priceCents: number }], cacheStatus: enum(HIT|MISS) }`
3. WHEN the cache contains a valid entry (cache HIT), THE Platform_Service SHALL return the cached data without querying the database
4. WHEN the cache does not contain a valid entry (cache MISS), THE Platform_Service SHALL compute availability from `availability_windows` minus `availability_overrides` minus existing bookings, cache the result, and return it with `cacheStatus: "MISS"`
5. WHEN an availability window, override, or pricing rule is created, updated, or deleted, THE Platform_Service SHALL invalidate the relevant cache entries for the affected court
6. WHEN a booking event is received from the `booking-events` Kafka topic, THE Platform_Service SHALL invalidate the cache entry for the affected court and date
7. WHEN Redis is unavailable, THE Platform_Service SHALL fall through to direct PostgreSQL queries and include `cacheStatus: "MISS"` in the response
8. THE Platform_Service SHALL support an optional `durationMinutes` query parameter to override the court's default duration when computing available slots (e.g., a 90-minute court can show 60-minute slots if requested)
9. THE Platform_Service SHALL use the cache key pattern `availability:{courtId}:{date}:{durationMinutes}` — including `durationMinutes` since AC8 allows overriding the court's default duration, which produces different slot sets
10. THE Platform_Service SHALL log cache hit/miss ratios as metrics for monitoring

### Requirement 21: Weather Forecast Integration

**User Story:** As a customer, I want to see weather forecasts for court locations when browsing and booking, so that I can choose between indoor and outdoor courts based on expected conditions.

#### Acceptance Criteria

1. THE Platform_Service SHALL integrate with the OpenWeatherMap API to fetch weather forecast data for court locations
2. WHEN a customer requests weather data via `GET /api/weather?latitude={lat}&longitude={lng}&date={date}&time={time}`, THE Platform_Service SHALL return forecast data for the specified location, date, and time
   - **Output:** `{ temperature: number, feelsLike: number, humidity: number, windSpeed: number, precipitationProbability: number, weatherCondition: string, weatherIcon: string, recommendation: enum(OUTDOOR_IDEAL|OUTDOOR_OK|INDOOR_RECOMMENDED) }`
3. THE Platform_Service SHALL determine the `recommendation` based on weather conditions:
   - `OUTDOOR_IDEAL`: Clear/sunny, temperature 15-30°C, precipitation < 20%, wind < 20 km/h
   - `OUTDOOR_OK`: Partly cloudy, temperature 10-35°C, precipitation < 40%, wind < 30 km/h
   - `INDOOR_RECOMMENDED`: Rain, extreme temperatures, high wind, or precipitation > 40%
4. THE Platform_Service SHALL cache weather data in Redis with a configurable TTL (default 10 minutes) using the cache key `weather:{lat_rounded}:{lng_rounded}:{date}:{time_rounded}` — including the time component rounded to the nearest hour since the API accepts a `time` parameter and forecasts vary by time of day
5. THE Platform_Service SHALL round coordinates to 2 decimal places for cache key generation to group nearby locations
6. WHEN the requested date is beyond the 7-day forecast window, THE Platform_Service SHALL return `{ available: false, message: "Forecast not yet available — check back closer to your booking date" }`
7. WHEN the OpenWeatherMap API is unavailable or returns an error, THE Platform_Service SHALL return `{ available: false, message: "Weather data temporarily unavailable" }` without blocking the calling operation
8. THE Platform_Service SHALL configure the OpenWeatherMap API key via environment variable (`OPENWEATHERMAP_API_KEY`) and support rate limiting to stay within the API plan limits
9. THE Platform_Service SHALL enforce rate limiting for OpenWeatherMap API calls: maximum 60 calls per minute for the free tier. When the rate limit is exceeded, THE Platform_Service SHALL return cached data if available (even if stale), or return `{ available: false, message: "Weather service rate limit exceeded — try again shortly", retryAfterSeconds: number }` if no cached data exists


### Requirement 22: Customer Favorites

**User Story:** As a customer, I want to save courts as favorites so that I can quickly find and book them again.

#### Acceptance Criteria

1. WHEN a customer adds a court to favorites via `POST /api/favorites/{courtId}`, THE Platform_Service SHALL create an entry in the `favorites` table
   - **Output (success):** `201 Created`
   - **Output (already favorited):** `200 OK` (idempotent)
2. WHEN a customer removes a court from favorites via `DELETE /api/favorites/{courtId}`, THE Platform_Service SHALL remove the entry from the `favorites` table
   - **Output (success):** `204 No Content`
   - **Output (not favorited):** `204 No Content` (idempotent)
3. WHEN a customer retrieves their favorites via `GET /api/favorites`, THE Platform_Service SHALL return all favorited courts with summary information
   - **Output:** `{ favorites: [{ courtId: UUID, nameEl: string, nameEn?: string, courtType: enum, locationType: enum, basePriceCents: number, distanceKm?: number, addedAt: ISO8601 }] }`
4. THE Platform_Service SHALL include the `isFavorite: boolean` field on court objects in search results and court detail responses for authenticated customers
5. WHEN a favorited court is deleted by its owner, THE Platform_Service SHALL cascade-delete the favorite entry (via ON DELETE CASCADE on the `favorites` table)
6. WHEN a customer retrieves favorites via `GET /api/favorites` with `latitude` and `longitude` query parameters, THE Platform_Service SHALL calculate and include `distanceKm` for each favorited court using PostGIS `ST_Distance` with geography cast. When coordinates are not provided, `distanceKm` SHALL be `null`

### Requirement 23: User Preferences and Personalization

**User Story:** As a customer, I want to save my preferred playing days, times, and search distance, so that the app pre-fills my search with my usual preferences.

#### Acceptance Criteria

1. WHEN a customer updates preferences via `PUT /api/preferences`, THE Platform_Service SHALL store them in the `preferences` table (upsert)
   - **Input:** `{ preferredDays?: [enum(MONDAY..SUNDAY)], preferredStartTime?: HH:mm, preferredEndTime?: HH:mm, maxSearchDistanceKm?: number, notifyBookingEvents?: boolean, notifyFavoriteAlerts?: boolean, notifyPromotional?: boolean, notifyEmail?: boolean, notifyPush?: boolean, dndStart?: HH:mm, dndEnd?: HH:mm }`
   - **Output:** `200 OK` with the updated preferences
2. WHEN a customer retrieves preferences via `GET /api/preferences`, THE Platform_Service SHALL return the stored preferences or defaults if none are set
3. THE Platform_Service SHALL apply default values for preferences that are not explicitly set:
   - `maxSearchDistanceKm`: 10.0
   - `notifyBookingEvents`: true
   - `notifyFavoriteAlerts`: true
   - `notifyPromotional`: true
   - `notifyEmail`: true
   - `notifyPush`: true
4. WHEN a customer's preferences include `preferredDays` and `preferredStartTime`/`preferredEndTime`, THE Platform_Service SHALL use these as default search parameters when the customer does not explicitly specify search criteria
5. WHEN a customer configures do-not-disturb hours (`dndStart`, `dndEnd`), THE Platform_Service SHALL include these in the preferences response for Phase 5 notification delivery to respect
6. THE Platform_Service SHALL support Phase 3-specific notification preferences in the preferences input:
   - `notifyVerificationStatus?: boolean` (default: true) — notify court owners of verification status changes
   - `notifyHolidayGeneration?: boolean` (default: true) — notify court owners when recurring holidays are generated
7. WHEN a court owner has `notifyVerificationStatus = false`, THE Platform_Service SHALL NOT publish `NOTIFICATION_REQUESTED` events for verification status changes for that user
8. WHEN a court owner has `notifyHolidayGeneration = false`, THE Platform_Service SHALL NOT publish `NOTIFICATION_REQUESTED` events for recurring holiday generation summaries for that user

### Requirement 24: Skill Level Management

**User Story:** As a customer, I want to set my skill level for different sports, so that I can be matched with appropriate players in open matches.

#### Acceptance Criteria

1. WHEN a customer sets their skill level via `PUT /api/skill-levels`, THE Platform_Service SHALL store entries in the `skill_levels` table (upsert per court type)
   - **Input:** `{ skillLevels: [{ courtType: enum(TENNIS|PADEL|BASKETBALL|FOOTBALL_5X5), level: number(1-7) }] }`
   - **Output:** `200 OK` with the updated skill levels
2. WHEN a customer retrieves their skill levels via `GET /api/skill-levels`, THE Platform_Service SHALL return all stored skill levels
   - **Output:** `{ skillLevels: [{ courtType: enum, level: number, label: string }] }`
3. THE Platform_Service SHALL map skill level numbers to labels:
   - 1: Beginner
   - 2: Advanced Beginner
   - 3: Intermediate
   - 4: Upper Intermediate
   - 5: Advanced
   - 6: Expert
   - 7: Professional
4. THE Platform_Service SHALL validate that `level` is between 1 and 7 (enforced by CHECK constraint on the `skill_levels` table)
5. WHEN a customer has not set a skill level for a court type, THE Platform_Service SHALL return no entry for that type (not a default value)


### Requirement 25: Court Owner Default Settings

**User Story:** As a court owner managing multiple courts, I want to configure default settings that are automatically applied to new courts, so that I don't have to repeat the same configuration for each court.

#### Acceptance Criteria

1. WHEN a court owner updates default settings via `PUT /api/court-defaults`, THE Platform_Service SHALL store them in the `court_defaults` table (upsert)
   - **Input:** `{ defaultDurationMinutes?: number, defaultConfirmationMode?: enum(INSTANT|MANUAL), defaultConfirmationTimeoutHours?: number, defaultCancellationTiers?: [{ thresholdHours: number, refundPercent: number }], defaultAmenities?: string[] }`
   - **Output:** `200 OK` with the updated defaults
2. WHEN a court owner retrieves default settings via `GET /api/court-defaults`, THE Platform_Service SHALL return the stored defaults or an empty object if none are configured
3. WHEN a court owner creates a new court and has configured defaults, THE Platform_Service SHALL apply the defaults to fields not explicitly provided in the court creation request (as specified in Requirement 1 AC4)
4. THE Platform_Service SHALL validate default cancellation tiers using the same rules as Requirement 19 (refund percentages decrease as threshold hours decrease)
5. WHEN default settings are updated, THE Platform_Service SHALL record the changes in the `court_owner_audit_logs` table

### Requirement 26: Kafka Event Publishing

**User Story:** As a platform developer, I want court configuration changes published to Kafka, so that Transaction Service can stay synchronized with court data.

#### Acceptance Criteria

1. THE Platform_Service SHALL publish events to the `court-update-events` Kafka topic using a standard event envelope:
   ```json
   {
     "eventId": "UUID",
     "eventType": "COURT_UPDATED | PRICING_UPDATED | AVAILABILITY_UPDATED | CANCELLATION_POLICY_UPDATED | COURT_DELETED | STRIPE_CONNECT_STATUS_CHANGED",
     "timestamp": "ISO8601",
     "courtId": "UUID",
     "payload": { ... }
   }
   ```
   Note: `STRIPE_CONNECT_STATUS_CHANGED` is listed in the event envelope definition for completeness per the system design. It will be implemented when Stripe Connect onboarding is added in Phase 4.
2. THE Platform_Service SHALL use `courtId` as the Kafka partition key to guarantee event ordering per court
3. THE Platform_Service SHALL publish `COURT_UPDATED` events when court details are created or modified (Requirement 1 AC12, Requirement 2 AC7)
4. THE Platform_Service SHALL publish `PRICING_UPDATED` events when base price or pricing rules change (Requirement 2 AC8, Requirement 18 AC6)
5. THE Platform_Service SHALL publish `AVAILABILITY_UPDATED` events when availability windows or overrides change (Requirement 6 AC6, Requirement 7 AC7)
6. THE Platform_Service SHALL publish `CANCELLATION_POLICY_UPDATED` events when cancellation tiers change (Requirement 19 AC6)
7. THE Platform_Service SHALL publish `COURT_DELETED` events when a court is deleted (Requirement 3 AC4)
8. THE Platform_Service SHALL NOT block on Kafka publish failures — if the broker is unavailable, the event SHALL be logged and the triggering operation SHALL still succeed
9. THE Platform_Service SHALL include a `version` field in `COURT_UPDATED` events matching the court's optimistic locking version, enabling consumers to detect out-of-order events

### Requirement 27: Court Owner Audit Logging

**User Story:** As a court owner, I want an immutable audit trail of all changes made to my courts and settings, so that I can track modifications and resolve disputes.

#### Acceptance Criteria

1. THE Platform_Service SHALL record entries in the `court_owner_audit_logs` table for the following actions:
   - `COURT_CREATED`: Court registration (Requirement 1 AC11)
   - `COURT_UPDATED`: Court detail changes (Requirement 2 AC6)
   - `COURT_DELETED`: Court deletion (Requirement 3 AC3)
   - `PRICING_UPDATED`: Pricing rule changes (Requirement 18 AC7)
   - `AVAILABILITY_CHANGED`: Availability window or override changes (Requirement 6 AC7, Requirement 7 AC8)
   - `CANCELLATION_POLICY_UPDATED`: Cancellation tier changes (Requirement 19 AC7)
   - `SETTINGS_UPDATED`: Court owner default settings changes (Requirement 25 AC5)
   - `VERIFICATION_STATUS_CHANGED`: Verification request status changes (Requirement 17 AC10)
2. Each audit log entry SHALL include: `court_owner_id`, `court_id` (if applicable), `action`, `entity_type`, `entity_id`, `changes` (JSONB with before/after snapshots), `ip_address`, `user_agent`, and `created_at`
3. THE `court_owner_audit_logs` table SHALL be append-only — no UPDATE or DELETE operations permitted (enforced at application level)
4. WHEN a court owner retrieves their audit log via `GET /api/audit-logs`, THE Platform_Service SHALL return entries sorted by `created_at` descending with pagination support
   - **Optional params:** `courtId: UUID, action: string, fromDate: ISO8601, toDate: ISO8601, page: number, size: number (default 50)`
   - **Output:** `{ entries: [AuditLogEntry], totalCount: number, page: number, size: number }`
5. THE Platform_Service SHALL only return audit log entries belonging to the authenticated court owner


### Requirement 28: Booking Event Consumption for Cache Invalidation

**User Story:** As a platform developer, I want Platform Service to consume booking events from Kafka, so that the availability cache is invalidated when bookings are created, confirmed, or cancelled.

#### Acceptance Criteria

1. THE Platform_Service SHALL consume events from the `booking-events` Kafka topic using a Spring Kafka consumer with consumer group `platform-service-booking-consumer`
2. THE Platform_Service SHALL handle the following booking event types for cache invalidation:
   - `BOOKING_CREATED`: Invalidate availability cache for the booked court and date
   - `BOOKING_CONFIRMED`: Invalidate availability cache for the booked court and date
   - `BOOKING_CANCELLED`: Invalidate availability cache for the booked court and date
   - `BOOKING_MODIFIED`: Invalidate availability cache for both the old and new court/date combinations
3. WHEN a booking event is received, THE Platform_Service SHALL invalidate the Redis cache entry matching `availability:{courtId}:{date}:*` (all duration variants for that court and date)
4. THE Platform_Service SHALL handle deserialization errors gracefully — malformed events SHALL be logged and skipped without blocking the consumer
5. THE Platform_Service SHALL configure the consumer with `auto.offset.reset=latest` to avoid reprocessing historical events on first startup
6. Note: Until Phase 4 deploys the Transaction Service booking flow, no events will be published to the `booking-events` topic. The consumer is registered in Phase 3 for forward compatibility.

### Requirement 29: Search and Discovery API Contracts

**User Story:** As a platform developer, I want well-defined API contracts for all search and discovery endpoints, so that the mobile app team can develop against stable interfaces.

#### Acceptance Criteria

1. THE Platform_Service SHALL define the following API response models:
   - `CourtSummary`: `{ courtId, nameEl, nameEn, courtType, locationType, address, latitude, longitude, basePriceCents, averageRating, totalReviews, distanceKm, isFavorite, visible }`
   - `CourtMapMarker`: `{ courtId, latitude, longitude, courtType, locationType, name, basePriceCents, averageRating, isFavorite }`
   - `CourtDetail`: Full court object with all fields from the `courts` table plus computed fields
   - `AvailabilitySlot`: `{ startTime, endTime, status, priceCents }`
   - `WeatherSummary`: `{ temperature, feelsLike, humidity, windSpeed, precipitationProbability, weatherCondition, weatherIcon, recommendation }`
2. THE Platform_Service SHALL return `averageRating` as `null` and `totalReviews` as `0` until Phase 10 (Court Ratings) is implemented
3. THE Platform_Service SHALL use consistent pagination across all list endpoints: `{ items: [...], totalCount: number, page: number, size: number }`
4. THE Platform_Service SHALL use consistent error response format: `{ error: string, message?: string, details?: object }`
5. THE Platform_Service SHALL return all monetary amounts in euro cents (integer) consistent with the database schema and Stripe convention
6. THE Platform_Service SHALL support bilingual responses — when the requesting user's `language` preference is `en`, return `nameEn` fields; otherwise return `nameEl` fields. The `Accept-Language` header or user preference determines the language.

### Requirement 30: Cross-Schema Views (Flyway Migration)

**User Story:** As a platform developer, I want cross-schema database views created as part of Phase 3 Flyway migrations, so that Transaction Service can perform read-only lookups of court and cancellation data without HTTP round-trips.

#### Acceptance Criteria

1. THE Platform_Service SHALL create the `v_court_summary` view in a Flyway migration, exposing: `id, owner_id, name_el, name_en, court_type, location_type, location, timezone, base_price_cents, duration_minutes, max_capacity, confirmation_mode, confirmation_timeout_hours, waitlist_enabled, visible, version` from `platform.courts` WHERE `visible = TRUE`, and GRANT SELECT to `transaction_service_role`
2. THE Platform_Service SHALL create the `v_court_cancellation_tiers` view in a Flyway migration, exposing: `court_id, threshold_hours, refund_percent, sort_order` from `platform.cancellation_tiers` ORDER BY `court_id, sort_order`, and GRANT SELECT to `transaction_service_role`
3. THE Platform_Service SHALL create the `v_user_skill_level` view in a Flyway migration, exposing: `user_id, court_type, level` from `platform.skill_levels`, and GRANT SELECT to `transaction_service_role`
4. Note: `v_user_basic` was created in Phase 2 migrations and is not repeated here

### Requirement 31: Internal Service-to-Service API Stubs

**User Story:** As a platform developer, I want internal API endpoint stubs registered in Phase 3, so that Transaction Service can call them when Phase 4 (Booking & Payments) is implemented.

#### Acceptance Criteria

1. THE Platform_Service SHALL register an internal endpoint `GET /internal/courts/{courtId}/validate-slot` that accepts query parameters `date`, `startTime`, `endTime` and returns slot availability validation. Until full implementation, this endpoint SHALL return `{ valid: true, courtId: UUID, date: string, startTime: string, endTime: string }` for any existing court.
2. THE Platform_Service SHALL register an internal endpoint `GET /internal/courts/{courtId}/calculate-price` that accepts query parameters `date`, `startTime`, `endTime` and returns the calculated price applying pricing rules. This endpoint SHALL be fully functional in Phase 3 since pricing rules are in scope.
3. Internal endpoints SHALL use the `/internal/*` path prefix, SHALL NOT be exposed through NGINX Ingress, and SHALL only be reachable within the Kubernetes cluster network
4. Internal endpoint authentication SHALL use API key validation (`X-Internal-Api-Key` header) in dev/test environments, with mTLS via Istio in staging/production (per system design)

### Requirement 32: Notification Event Publishing Pattern

**User Story:** As a platform developer, I want a consistent pattern for publishing notification events from Platform Service, so that Phase 5 (Notifications) can consume them without changes to Phase 3 code.

#### Acceptance Criteria

1. THE Platform_Service SHALL publish `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic using the standard event envelope with `userId` as the partition key
2. THE Platform_Service SHALL publish `NOTIFICATION_REQUESTED` events for the following Phase 3 scenarios:
   - Verification request submitted (notify PLATFORM_ADMIN users)
   - Verification request approved (notify court owner)
   - Verification request rejected (notify court owner with rejection reason)
   - Verification SLA reminder (notify PLATFORM_ADMIN users when pending > 48 hours)
   - Bulk holiday application booking cancellations (notify affected customers)
   - Recurring holiday generation completed (notify court owner with summary)
3. Each `NOTIFICATION_REQUESTED` event payload SHALL include: `notificationType: string, recipientUserId: UUID, title: string, body: string, language: enum(el|en), data: { deepLink?: string, referenceId?: UUID, referenceType?: string }`
4. THE Platform_Service SHALL NOT block on Kafka publish failures — if the `notification-events` topic is unavailable, the notification event SHALL be logged and skipped (the triggering operation SHALL still succeed)
5. Note: These events are published for forward compatibility. They will not be consumed until Phase 5 (Real-time & Notifications) deploys the notification processor in Transaction Service.



### Requirement 33: Court Visibility Management

**User Story:** As a court owner, I want to manually control the visibility of my courts, so that I can temporarily hide courts without deleting them.

#### Acceptance Criteria

1. WHEN a court owner toggles court visibility via `PUT /api/courts/{courtId}/visibility`, THE Platform_Service SHALL update the `visible` field on the court
   - **Input:** `{ visible: boolean }`
   - **Output (success):** `200 OK` with `{ courtId: UUID, visible: boolean, updatedAt: ISO8601 }`
   - **Output (not owner):** `403 Forbidden`
   - **Output (not verified):** `422 Unprocessable Entity` with `{ error: "COURT_OWNER_NOT_VERIFIED", message: "Courts cannot be made visible until owner verification is approved" }` — unverified court owners can only set `visible = false`
2. WHEN a court's visibility is changed, THE Platform_Service SHALL publish a `COURT_UPDATED` event to the `court-update-events` Kafka topic with `{ changedFields: ["visible"] }`
3. WHEN a court's visibility is changed, THE Platform_Service SHALL record the change in the `court_owner_audit_logs` table
4. WHEN a court is set to `visible = false`, THE Platform_Service SHALL NOT affect existing confirmed bookings — customers can still access their booking details and the booking proceeds as scheduled
5. THE Platform_Service SHALL NOT allow setting `visible = true` for courts owned by unverified court owners (enforced at application level in addition to AC1)


### Requirement 34: Timezone Handling in Availability

**User Story:** As a court owner operating in Greece, I want availability windows and overrides to be interpreted in my court's local timezone, so that schedules are accurate regardless of server timezone or daylight saving time changes.

#### Acceptance Criteria

1. THE Platform_Service SHALL interpret all availability window times (`startTime`, `endTime`) in the court's configured `timezone` (default: `Europe/Athens`)
2. THE Platform_Service SHALL interpret all availability override times in the court's configured `timezone`
3. WHEN computing available slots for a given date, THE Platform_Service SHALL account for daylight saving time (DST) transitions:
   - On DST spring-forward dates (e.g., last Sunday of March in Greece), slots during the skipped hour SHALL NOT be generated
   - On DST fall-back dates (e.g., last Sunday of October in Greece), slots during the repeated hour SHALL be generated once (using the first occurrence)
4. THE Platform_Service SHALL store all timestamps in the database in UTC but convert to/from the court's local timezone for display and input
5. WHEN a customer views availability, THE Platform_Service SHALL return slot times in the court's local timezone with an explicit `timezone` field in the response: `{ courtId, date, timezone: "Europe/Athens", slots: [...] }`
6. THE Platform_Service SHALL validate that the `timezone` field on court creation/update is a valid IANA timezone identifier (e.g., `Europe/Athens`, `Europe/London`). Invalid timezones SHALL return `400 Bad Request` with `{ error: "INVALID_TIMEZONE", message: "Timezone must be a valid IANA timezone identifier" }`


### Requirement 35: Bulk Court Operations

**User Story:** As a court owner managing multiple courts, I want to perform bulk operations like updating pricing or availability across several courts at once, so that I can efficiently manage my portfolio.

#### Acceptance Criteria

1. WHEN a court owner submits a bulk pricing update via `PUT /api/courts/bulk/pricing`, THE Platform_Service SHALL apply the pricing rules to all specified courts atomically
   - **Input:** `{ courtIds: [UUID], pricingRules: [{ dayOfWeek: enum, startTime: HH:mm, endTime: HH:mm, multiplier: number }] }`
   - **Output (success):** `200 OK` with `{ courtsUpdated: number, courtIds: [UUID] }`
   - **Output (partial ownership):** `403 Forbidden` with `{ error: "NOT_OWNER_OF_ALL_COURTS", unauthorizedCourtIds: [UUID] }`
2. WHEN a court owner submits a bulk availability window update via `PUT /api/courts/bulk/availability-windows`, THE Platform_Service SHALL apply the availability windows to all specified courts atomically
   - **Input:** `{ courtIds: [UUID], windows: [{ dayOfWeek: enum, startTime: HH:mm, endTime: HH:mm }] }`
   - **Output (success):** `200 OK` with `{ courtsUpdated: number, courtIds: [UUID] }`
3. THE Platform_Service SHALL validate that the authenticated court owner owns ALL specified courts before applying bulk operations. If any court is not owned by the requester, the entire operation SHALL be rejected.
4. WHEN bulk operations are applied, THE Platform_Service SHALL publish individual Kafka events for each affected court (not a single bulk event) to maintain event ordering guarantees per court
5. WHEN bulk operations are applied, THE Platform_Service SHALL record individual audit log entries for each affected court
6. THE Platform_Service SHALL limit bulk operations to a maximum of 50 courts per request. Requests exceeding this limit SHALL return `400 Bad Request` with `{ error: "BULK_LIMIT_EXCEEDED", maxCourts: 50, requestedCourts: number }`


### Requirement 36: Error Response Standardization

**User Story:** As a mobile app developer, I want consistent and predictable error responses from all Phase 3 endpoints, so that I can implement robust error handling in the app.

#### Acceptance Criteria

1. THE Platform_Service SHALL use the following standard error response format for all Phase 3 endpoints:
   ```json
   {
     "error": "ERROR_CODE",
     "message": "Human-readable error description",
     "details": { ... },
     "timestamp": "ISO8601",
     "path": "/api/endpoint/path",
     "traceId": "UUID"
   }
   ```
2. THE Platform_Service SHALL use the following error codes for Phase 3 endpoints:

   **Authentication/Authorization (4xx):**
   - `AUTHZ_INSUFFICIENT_ROLE`: User lacks required role for the operation
   - `AUTHZ_NOT_OWNER`: User is not the owner of the requested resource
   - `AUTHZ_NOT_VERIFIED`: Court owner verification required for this operation

   **Validation Errors (400):**
   - `INVALID_COORDINATES`: Latitude/longitude out of valid range
   - `INVALID_TIMEZONE`: Invalid IANA timezone identifier
   - `INVALID_AMENITY`: Amenity value not in standard enum
   - `INVALID_IMAGE_FORMAT`: Image format not supported
   - `IMAGE_TOO_LARGE`: Image exceeds maximum file size
   - `MAX_IMAGES_EXCEEDED`: Court has reached maximum image limit
   - `OVERLAPPING_WINDOWS`: Availability windows overlap
   - `OVERLAPPING_PRICING_RULES`: Pricing rules overlap
   - `INVALID_CANCELLATION_TIERS`: Cancellation tier configuration invalid
   - `INVALID_DATE_RANGE`: Date range parameters invalid
   - `BULK_LIMIT_EXCEEDED`: Bulk operation exceeds maximum items

   **Conflict Errors (409):**
   - `CONCURRENT_MODIFICATION`: Optimistic locking version mismatch
   - `COURT_HAS_FUTURE_BOOKINGS`: Cannot delete court with future bookings
   - `COURT_HAS_PENDING_PAYOUTS`: Cannot delete court with pending payouts
   - `VERIFICATION_ALREADY_PENDING`: Verification request already in progress
   - `VERIFICATION_ALREADY_APPROVED`: Cannot modify approved verification
   - `HOLIDAY_NAME_EXISTS`: Custom holiday name already exists

   **Not Found Errors (404):**
   - `COURT_NOT_FOUND`: Requested court does not exist
   - `IMAGE_NOT_FOUND`: Requested image does not exist
   - `HOLIDAY_NOT_FOUND`: Requested holiday template does not exist
   - `OVERRIDE_NOT_FOUND`: Requested availability override does not exist
   - `VERIFICATION_NOT_FOUND`: Requested verification request does not exist

   **Business Logic Errors (422):**
   - `COURT_OWNER_NOT_VERIFIED`: Operation requires verified court owner status
   - `CHANGE_AFFECTS_BOOKINGS`: Update would affect existing bookings

   **External Service Errors (503):**
   - `WEATHER_SERVICE_UNAVAILABLE`: OpenWeatherMap API unavailable
   - `STORAGE_SERVICE_UNAVAILABLE`: DigitalOcean Spaces unavailable

3. THE Platform_Service SHALL include a `traceId` (UUID) in all error responses to enable log correlation for debugging
4. THE Platform_Service SHALL log all error responses with the `traceId`, request details, and stack trace (for 5xx errors) for operational monitoring
5. THE Platform_Service SHALL return localized error messages based on the `Accept-Language` header when available, defaulting to English


---

### Requirement 37: Court Owner Notification Channel Preferences

**User Story:** As a court owner, I want to configure which notification channels (push, email, in-app) I receive for each event type, so that I can control how I'm notified about different activities.

#### Acceptance Criteria

1. WHEN a court owner updates notification channel preferences via `PUT /api/notification-preferences`, THE Platform_Service SHALL store them in the `court_owner_notification_preferences` table (upsert per event type)
   - **Input:** `{ preferences: [{ eventType: string, pushEnabled: boolean, emailEnabled: boolean, inAppEnabled: boolean }] }`
   - **Output:** `200 OK` with the updated preferences
   - **Valid event types for Phase 3:** `VERIFICATION_SUBMITTED`, `VERIFICATION_APPROVED`, `VERIFICATION_REJECTED`, `VERIFICATION_SLA_REMINDER`, `HOLIDAY_GENERATION_COMPLETED`, `BOOKING_CANCELLED_BY_HOLIDAY`
2. WHEN a court owner retrieves notification channel preferences via `GET /api/notification-preferences`, THE Platform_Service SHALL return all configured preferences with defaults for unconfigured event types
   - **Output:** `{ preferences: [{ eventType: string, pushEnabled: boolean, emailEnabled: boolean, inAppEnabled: boolean }] }`
3. THE Platform_Service SHALL apply default values for notification channel preferences that are not explicitly configured:
   - `pushEnabled`: true
   - `emailEnabled`: true
   - `inAppEnabled`: true
4. WHEN publishing a `NOTIFICATION_REQUESTED` event, THE Platform_Service SHALL check the court owner's notification channel preferences and include the enabled channels in the event payload: `{ channels: { push: boolean, email: boolean, inApp: boolean } }`
5. THE Platform_Service SHALL enforce that at least one channel remains enabled for each event type. Attempts to disable all channels for an event type SHALL return `400 Bad Request` with `{ error: "AT_LEAST_ONE_CHANNEL_REQUIRED", eventType: string }`
6. WHEN a court owner has not configured preferences for an event type, THE Platform_Service SHALL use the default (all channels enabled) when publishing notifications


### Requirement 38: Smart Reminder Rules Management

**User Story:** As a court owner, I want to configure smart alert rules for my courts, so that I receive timely reminders about pending confirmations, low occupancy, and daily summaries.

#### Acceptance Criteria

1. WHEN a court owner creates a reminder rule via `POST /api/reminder-rules`, THE Platform_Service SHALL store it in the `reminder_rules` table
   - **Input:** `{ courtId?: UUID, ruleType: enum(BOOKING_REMINDER|PENDING_CONFIRMATION|LOW_OCCUPANCY|DAILY_SUMMARY), triggerHoursBefore?: number, triggerTime?: HH:mm, thresholdPercent?: number, channels: [enum(PUSH|EMAIL)] }`
   - **Output (success):** `201 Created` with the created rule including generated `id`
   - **Output (duplicate):** `409 Conflict` with `{ error: "RULE_TYPE_EXISTS", message: "A rule of this type already exists for this court" }`
   - When `courtId` is NULL, the rule applies to all courts owned by the court owner
2. WHEN a court owner retrieves reminder rules via `GET /api/reminder-rules`, THE Platform_Service SHALL return all rules owned by the authenticated court owner
   - **Optional params:** `courtId: UUID, ruleType: string`
   - **Output:** `{ rules: [ReminderRule] }`
3. WHEN a court owner updates a reminder rule via `PUT /api/reminder-rules/{ruleId}`, THE Platform_Service SHALL validate ownership and apply the changes
4. WHEN a court owner deletes a reminder rule via `DELETE /api/reminder-rules/{ruleId}`, THE Platform_Service SHALL remove the rule
5. THE Platform_Service SHALL validate rule configuration based on rule type:
   - `BOOKING_REMINDER`: requires `triggerHoursBefore` (1-168 hours)
   - `PENDING_CONFIRMATION`: requires `triggerHoursBefore` (1-48 hours)
   - `LOW_OCCUPANCY`: requires `thresholdPercent` (1-100)
   - `DAILY_SUMMARY`: requires `triggerTime` (HH:mm format)
6. THE Platform_Service SHALL allow enabling/disabling rules via `PUT /api/reminder-rules/{ruleId}/enabled` with input `{ enabled: boolean }`
7. Note: The actual reminder triggering and notification delivery is Phase 6 (Admin, Analytics & Support). Phase 3 implements the rule CRUD operations and stores the configuration for Phase 6 to consume.


### Requirement 39: Stripe Connect Status Integration

**User Story:** As a platform developer, I want Phase 3 to read and expose Stripe Connect status for court owners, so that court visibility and payment eligibility can be determined correctly.

#### Acceptance Criteria

1. THE Platform_Service SHALL read the `stripe_connect_status` field from the `users` table when determining court visibility logic
2. WHEN a court owner's `stripe_connect_status` is `NOT_STARTED`, `PENDING`, `RESTRICTED`, or `DISABLED`, THE Platform_Service SHALL NOT allow setting `visible = true` on their courts, returning `422 Unprocessable Entity` with `{ error: "STRIPE_CONNECT_NOT_ACTIVE", status: string, message: "Stripe Connect onboarding must be completed before courts can be made visible" }`
3. WHEN a court owner retrieves their profile via `GET /api/users/me`, THE Platform_Service SHALL include the `stripeConnectStatus` field in the response
4. THE Platform_Service SHALL expose an internal endpoint `GET /internal/users/{userId}/stripe-connect-status` for Transaction Service to check Stripe Connect status before payment capture
   - **Output:** `{ userId: UUID, stripeConnectStatus: enum(NOT_STARTED|PENDING|ACTIVE|RESTRICTED|DISABLED), stripeConnectAccountId?: string }`
5. Note: Stripe Connect onboarding and webhook handling is Phase 4. Phase 3 only reads the status field that will be populated by Phase 4.
6. THE Platform_Service SHALL include `stripeConnectStatus` in the court owner's court listing response (`GET /api/courts/mine`) as a top-level field to help court owners understand why their courts may not be visible


### Requirement 40: Court Owner Dashboard Stub

**User Story:** As a court owner, I want a dashboard endpoint that provides a summary of my courts and pending actions, so that I can quickly see what needs my attention.

#### Acceptance Criteria

1. WHEN a court owner requests their dashboard via `GET /api/dashboard`, THE Platform_Service SHALL return a summary of their courts and pending actions
   - **Output:** 
   ```json
   {
     "summary": {
       "totalCourts": number,
       "visibleCourts": number,
       "hiddenCourts": number,
       "verificationStatus": enum(NOT_SUBMITTED|PENDING_REVIEW|APPROVED|REJECTED),
       "stripeConnectStatus": enum(NOT_STARTED|PENDING|ACTIVE|RESTRICTED|DISABLED)
     },
     "pendingActions": [
       { "type": string, "count": number, "description": string }
     ],
     "todaysBookings": null,
     "revenueThisMonth": null,
     "occupancyRate": null
   }
   ```
2. THE Platform_Service SHALL populate `pendingActions` with Phase 3-relevant items:
   - `{ type: "VERIFICATION_REQUIRED", count: 1, description: "Complete verification to make courts visible" }` — when `verificationStatus` is `NOT_SUBMITTED` or `REJECTED`
   - `{ type: "STRIPE_CONNECT_REQUIRED", count: 1, description: "Complete Stripe Connect onboarding to receive payments" }` — when `stripeConnectStatus` is `NOT_STARTED`
   - `{ type: "HIDDEN_COURTS", count: N, description: "N courts are hidden and not visible to customers" }` — when `hiddenCourts > 0`
3. THE Platform_Service SHALL return `todaysBookings`, `revenueThisMonth`, and `occupancyRate` as `null` until Phase 6 (Admin, Analytics & Support) implements the analytics API
4. THE Platform_Service SHALL return `403 Forbidden` if the requesting user is not a COURT_OWNER
5. THE Platform_Service SHALL cache the dashboard response in Redis with a 1-minute TTL to avoid repeated database queries, using cache key `dashboard:{courtOwnerId}`


### Requirement 41: Stripe Connect Status Changed Event Handling

**User Story:** As a platform developer, I want Platform Service to publish Stripe Connect status change events, so that Transaction Service can update its cache and make correct payment eligibility decisions.

#### Acceptance Criteria

1. WHEN the `stripe_connect_status` field is updated on a user record (via Phase 4 webhook handling), THE Platform_Service SHALL publish a `STRIPE_CONNECT_STATUS_CHANGED` event to the `court-update-events` Kafka topic
   - **Event payload:** `{ userId: UUID, previousStatus: string, newStatus: string, stripeConnectAccountId?: string, timestamp: ISO8601 }`
   - **Partition key:** `courtOwnerId` (to maintain ordering with other court-related events)
2. Note: The actual Stripe webhook handling that triggers this event is Phase 4. Phase 3 defines the event schema and ensures the publishing infrastructure is in place.
3. THE Platform_Service SHALL include `STRIPE_CONNECT_STATUS_CHANGED` in the Kafka event envelope definition (Requirement 26) for completeness
4. THE Platform_Service SHALL log all Stripe Connect status changes in the `court_owner_audit_logs` table with action `STRIPE_CONNECT_STATUS_CHANGED` and the before/after status in the `changes` JSONB field
