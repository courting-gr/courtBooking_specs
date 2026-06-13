# Requirements Document

**Phase 8: Mobile App (Flutter)**

## Introduction

Phase 8 delivers the cross-platform mobile application for the Court Booking Platform, targeting the `court-booking-mobile-app` repository (Flutter for iOS, Android, and Web). The app is a **client** of the existing backend services — it consumes the Platform Service and Transaction Service REST APIs and the Transaction Service STOMP-over-WebSocket interface. It does not implement business logic that is owned by the services; instead it presents the customer and court owner journeys, enforces client-side resilience, and degrades gracefully under poor connectivity.

The app serves two audiences:

- **Customers** — discover courts on a map, view court details and weather, book and pay for time slots via the Stripe SDK, manage bookings (including recurring), manage their profile and preferences, and receive notifications.
- **Court owners** — manage courts, view a booking calendar, create manual bookings, confirm/reject pending bookings, flag no-shows, mark manual bookings as paid externally, complete verification and Stripe Connect onboarding, and review analytics on the go. (The React admin web portal from Phase 6 remains the primary court-owner surface; the mobile app provides an on-the-go subset.)

This phase can begin in parallel with Phases 5–6 once the Phase 2–4 APIs are stable, using mock or staging APIs initially.

**Master requirements coverage:**
- **Req 31** (Mobile App Client Resilience, Offline Behavior, and Error Handling) — all 46 acceptance criteria, the API Error Response Contract, and the HTTP Status Code Handling table.
- **Req 1** (auth, roles, JWT claims, biometric, account management, GDPR deletion, OAuth provider linking, logout/refresh-token revocation).
- **Req 2 / Req 37** (court management surface for owners, owner verification, holidays affecting availability), **Req 3 / Req 4 / Req 5** (discovery, map, weather, search journey, personalization/preferences), **Req 6** (weather), **Req 7 / Req 19 / Req 35** (real-time availability, WebSocket auth and channel rules).
- **Req 8–12, Req 14** (atomic booking, pending confirmation, payments, Stripe Connect onboarding for owners, manual bookings, recurring bookings, modifications/cancellations/refunds, no-show, mark-paid-externally, booking history).
- **Req 13** (notifications), **Req 15** (analytics surface), **Req 28** (multi-language/i18n), **Req 29** (API contracts, Dart codegen, versioning), **Req 30** (support tickets).
- Mobile UX derived from the **Customer User Journey** (Steps 1–9, including Step 8a Booking Error Recovery and the Mobile App Error Handling and Connectivity Principles) and the **Court Owner Admin Journey**.

**Contracts this app consumes (sources of truth — not re-specified here):**
- `docs/api/openapi-platform-service.yaml` — auth, users/profile/preferences/skill-levels, courts, availability, holidays, weather, map, favorites, ratings, pricing, cancellation policy, analytics, dashboard, settings, verification, Stripe Connect, support.
- `docs/api/openapi-transaction-service.yaml` — bookings (list/detail/create/modify/cancel/confirm/reject/pending/recurring/no-show/mark-paid/bulk/calendar), payments (methods/setup-intent/refund/dispute), notifications/device tokens, receipts, manual bookings, Stripe Connect.
- `docs/api/websocket-message-contracts.json` — STOMP destinations (`/topic/courts/{courtId}/availability`, `/user/queue/bookings`, `/user/queue/notifications`, `/user/queue/system`), server message types (`AVAILABILITY_UPDATE`, `AVAILABILITY_SNAPSHOT`, `BOOKING_STATUS_UPDATE`, `NOTIFICATION`, `TOKEN_EXPIRING`, `SERVER_SHUTDOWN`, `ERROR`), the client `TOKEN_REFRESH` message on `/app/token-refresh`, close codes `4001`–`4005`, heartbeat (30s, 2 missed), and the reconnection strategy (1s, 2s, 4s, 8s, 16s, max 30s).
- `docs/api/kafka-event-contracts.json` — referenced only to understand which server-side events surface in the app as WebSocket messages and push notifications; the app does not consume Kafka directly.

**Scope boundaries:**
- The app generates its API client (Dart) from the OpenAPI specifications (Req 29 `API_Code_Generation`); endpoint/field shapes are authoritative in those specs.
- Server-side validation, pricing, conflict resolution, payment capture, refund calculation, and the reconciliation job remain authoritative on the backend. Client-side validation and refund/price previews are UX optimizations only (Req 31.46).
- The state-changing idempotency header is `X-Idempotency-Key` (UUID v4); the server returns the same response for duplicate keys within 24 hours.
- **Phase 2 (deferred) features** are accommodated in the app architecture and navigation but NOT required for MVP. Acceptance criteria for these are marked **⏳ PHASE 2**, consistent with the master Phase Roadmap: open match join/create (Req 24), split payments (Req 25), waitlist (Req 26), court ratings and reviews (Req 2.19–2.20), promo codes and dynamic pricing configuration (Req 27), scheduled reports (Req 15.6–15.9).
- **Subscription stub strategy:** per the master Phase Roadmap, all court owners are treated as "Subscribed" until Phase 10. Trial banners, trial countdown UI, subscription tier selection, and Stripe Billing screens are **⏳ PHASE 2** and are not implemented in this phase. The `subscriptionStatus` field returned by `GET /api/users/me` is read but not enforced by the app.

**Out of scope / deferred (not exposed by current contracts):**
- A bulk multi-device "active sessions" list with per-session revoke is not exposed by the current auth contracts; the app supports per-device logout via `POST /api/auth/logout` and full re-authentication. A dedicated session list is deferred.
- A self-service personal-data export endpoint is not exposed by the current contracts; the app surfaces GDPR account deletion (`POST /api/users/me/delete`) and directs export requests to the Support Hub. A data-export screen is deferred.

## Glossary

Terms below reuse the master requirements Glossary; mobile-specific terms are added where needed.

- **Mobile_App**: The Flutter application for iOS, Android, and Web, deployed from `court-booking-mobile-app`. The subject system for all requirements in this document, referred to as **THE Mobile_App**.
- **Platform_Service**: Backend microservice handling authentication, users, profiles, preferences, skill levels, courts, availability, holidays, weather, map aggregation, favorites, ratings, pricing, cancellation policy, analytics, dashboard, settings, verification, Stripe Connect, and support. Consumed by the Mobile_App via REST.
- **Transaction_Service**: Backend microservice handling bookings, payments, notifications, and the WebSocket interface. Consumed by the Mobile_App via REST and STOMP-over-WebSocket.
- **Customer**: User who discovers, books, and pays for court reservations through the Mobile_App.
- **Court_Owner**: User who owns and manages courts; uses the Mobile_App for on-the-go court and booking management.
- **OAuth_Login**: Authentication via Google, Facebook, or Apple, resulting in JWT issuance by the Platform_Service (`POST /api/auth/login`).
- **Linked_Provider**: An OAuth provider associated with the user's account, listed via `GET /api/auth/providers` and added via `POST /api/auth/providers/link`.
- **Biometric_Authentication**: Fingerprint or face recognition that unlocks the stored Refresh_Token from the device Secure_Enclave to obtain a new Access_Token without re-entering OAuth credentials.
- **Secure_Enclave**: Device-level secure storage (iOS Keychain / Android Keystore) used to store the Refresh_Token and other sensitive values.
- **Access_Token**: Short-lived JWT (15 minutes) sent as a Bearer token on API requests.
- **Refresh_Token**: Long-lived token (30 days) exchanged via `POST /api/auth/refresh` to obtain a new Access_Token; rotated on each use with replay detection.
- **Silent_Token_Refresh**: Transparent background refresh of an expired Access_Token without interrupting the user's current action.
- **Terms_and_Conditions**: Legal agreement a new user must accept before proceeding past first login.
- **Account_Screen**: Screen where a user views and edits profile fields and manages account-level actions (`GET /api/users/me`, `PUT /api/users/me`, `POST /api/auth/logout`, `POST /api/users/me/delete`).
- **User_Profile**: The authenticated user's profile from `GET /api/users/me` (id, email, name, phone, role, language, verified, stripeConnected, subscriptionStatus, trialDaysRemaining).
- **Account_Deletion**: GDPR right-to-be-forgotten request via `POST /api/users/me/delete` (cancels future bookings, processes refunds, anonymizes personal data).
- **User_Preferences**: Customer personalization from `GET /api/users/me/preferences` (`preferredDays`, `preferredStartTime`, `preferredEndTime`, `maxSearchDistanceKm`, `notificationPreferences`, `doNotDisturbStart`, `doNotDisturbEnd`).
- **Notification_Preferences**: The customer notification toggles within User_Preferences (`bookingEvents`, `favoriteCourtAlerts`, `promotionalNotifications`, `emailNotifications`, `pushNotifications`).
- **Owner_Notification_Preferences**: Court-owner per-event-type channel preferences, do-not-disturb, and email digest from `GET/PUT /api/settings/notification-preferences`.
- **Skill_Level**: Per-sport-type player skill (1–7) from `GET/PUT /api/users/me/skill-levels`.
- **Map_View**: Full-screen map of court markers and (⏳ Phase 2) open match markers, with Court_Filter tabs, an indoor/outdoor toggle, and a Distance_Filter.
- **Court_Filter**: Tab-based filtering of courts by type (All, Tennis, Padel, Basketball, Football) plus Favorites, on the Map_View.
- **Distance_Filter**: Search radius control mapped to the `radiusKm` parameter of `GET /api/courts/map` (default 10 km).
- **Map_Bounds**: Visible geographic area on the map (`minLat`, `maxLat`, `minLng`, `maxLng`) that determines which courts are loaded via `GET /api/courts/map`.
- **Marker_Clustering**: Grouping of nearby court markers into a single cluster marker that expands on zoom.
- **Favorite_Court**: Court marked by a Customer for quick access (add via `PUT /api/users/me/favorites/{courtId}`, remove via `DELETE /api/users/me/favorites/{courtId}`).
- **Weather_Forecast**: Weather prediction for a selected date/time within the 7-day window (`GET /api/weather`), with indoor/outdoor recommendation.
- **Court_Detail_Screen**: Screen showing a court's images, type, amenities, availability slots, weather (for outdoor courts), cancellation policy, and (⏳ Phase 2) ratings.
- **Booking_Flow**: The sequence slot selection → booking confirmation → payment → result/receipt.
- **Stripe_SDK**: The Stripe Flutter SDK used to collect and confirm payment (cards, Apple Pay, Google Pay) for `POST /api/bookings`.
- **Idempotency_Key**: Client-generated UUID v4 sent in the `X-Idempotency-Key` header on state-changing requests (booking creation, payment, cancellation) to prevent duplicate operations on retry; deduplicated server-side for 24 hours.
- **Booking_Confirmation_Mode**: Court-level setting (Instant or Manual) that determines whether a paid booking is `CONFIRMED` immediately or set to `PENDING_CONFIRMATION`.
- **My_Bookings**: Screen listing the Customer's bookings grouped as upcoming, past, cancelled, rejected, and pending confirmation (`GET /api/bookings`).
- **Recurring_Booking_Group**: A set of bookings created together via `POST /api/bookings/recurring` and viewed via `GET /api/bookings/recurring/{recurringGroupId}`.
- **Booking_Status**: One of `CONFIRMED`, `PENDING_CONFIRMATION`, `CANCELLED`, `COMPLETED`, `REJECTED`.
- **No_Show_Flag**: A boolean attribute set on a booking by a Court_Owner via `POST /api/bookings/{bookingId}/no-show`.
- **Mark_Paid_Externally**: Action by a Court_Owner to record out-of-band payment on a manual booking via `POST /api/bookings/{bookingId}/mark-paid`.
- **Cancellation_Policy**: Refund tier configuration for a court from `GET /api/courts/{courtId}/cancellation-policy`.
- **Refund_Preview**: A client-side estimate of the refund amount derived from the Cancellation_Policy and time-to-booking, shown before the user confirms a cancellation; the actual refund is returned by `POST /api/bookings/{bookingId}/cancel`.
- **Digital_Receipt**: Payment confirmation document retrieved via `GET /api/bookings/{bookingId}/receipt`.
- **Owner_Dashboard**: Court-owner summary from `GET /api/dashboard` (today's bookings, pending confirmations, revenue, reminder alerts).
- **Reminder_Alert**: A court-owner "action required" item surfaced on the Owner_Dashboard (e.g., unpaid booking approaching its date), originating from server-side reminder rules (Req 15b).
- **Booking_Calendar**: Court owner screen showing bookings across owned courts, with iCal export via `GET /api/bookings/calendar/ical`.
- **Manual_Booking**: Booking created by a Court_Owner without customer payment (`POST /api/bookings/manual`), blocking a slot for walk-in/phone reservations.
- **Verification_Request**: A court owner's business-document submission via `POST /api/verification`, with status read via `GET /api/verification`.
- **Verification_Status**: One of `NOT_SUBMITTED`, `PENDING_REVIEW`, `APPROVED`, `REJECTED` (per `GET /api/verification`).
- **Stripe_Connect_Onboarding**: Hosted Stripe flow initiated via `POST /api/users/me/stripe-connect/onboard`, with status read via `GET /api/users/me/stripe-connect`.
- **Stripe_Account_Status**: The court-owner Stripe Connect account state from `GET /api/users/me/stripe-connect` (`status`, `requiresAction`, `payoutSchedule`, `nextPayoutDate`).
- **Push_Notification**: Notification delivered via FCM (Android) / APNs (iOS) when the app is backgrounded or closed.
- **In_App_Notification**: Real-time notification delivered via the WebSocket `NOTIFICATION` message on `/user/queue/notifications` when the app is active.
- **Notification_Center**: In-app screen listing the user's notification history (`GET /api/notifications`), with read tracking.
- **Device_Registration_Token**: FCM/APNs token registered with the Transaction_Service via `POST /api/notifications/device` to target push notifications.
- **Deep_Link**: An in-app navigation target (e.g., `/bookings/{id}`) opened from a push notification or external link, carried in the `data.deepLink` field of a NOTIFICATION message.
- **Offline_Queue**: Local queue (max 50 items) of pending user actions (favorite toggles, preference saves) synced to the server when connectivity returns.
- **Cache_TTL**: Time-to-live (24 hours) for locally cached data (SQLite/Hive); after expiry, data is stale and refreshed on next access.
- **Skeleton_UI**: Placeholder shimmer layout shown while content loads, matching expected content structure.
- **Empty_State**: A screen state shown when a query returns zero results, providing actionable guidance.
- **Optimistic_UI_Update**: UI pattern where the interface updates immediately on user action before server confirmation, reverting if the server request fails.
- **Exponential_Backoff**: Retry strategy where wait time doubles between attempts (1s, 2s, 4s, 8s, 16s, max 30s), with jitter to avoid Thundering_Herd.
- **Thundering_Herd**: Many clients reconnecting simultaneously after an outage; mitigated by random jitter on reconnection delays.
- **Partial_Failure**: A response where some sections succeed and others fail (e.g., `GET /api/courts/map` returns courts but weather failed; `207 Multi-Status` on recurring bookings).
- **API_Error_Response**: Standard backend error body `{ error, message, details, timestamp, correlationId }`; the Mobile_App displays `message` to users and uses `error` for programmatic handling.
- **Correlation_Id**: The `correlationId` field of an API_Error_Response, included in support ticket submissions for debugging.
- **Diagnostic_Log**: Sanitized app logs (network requests, errors, navigation events) optionally attached to a Support_Ticket, with secrets and PII removed.
- **Support_Hub**: In-app interface providing FAQ (static, bundled with the app), support ticket submission (`POST /api/support/tickets`), and ticket tracking.
- **Euro_Cents**: Integer minor-unit amounts (e.g., `amountCents`) returned by the services and formatted by the Mobile_App as localized Euro currency for display.
- **Localized_Content**: User-facing strings and server content (including court names/descriptions in `el`/`en`) presented in the user's preferred language.
- **App_Permission**: A device capability gated by OS permission — location, notifications, biometrics, calendar, camera/photos.
- **Certificate_Pinning**: Client validation of the backend's TLS certificate/public key to resist man-in-the-middle interception.
- **Telemetry_Event**: A non-PII client analytics event (screen view, action, performance timing) emitted by the Mobile_App.
- **Crash_Report**: A non-PII diagnostic report captured on an unhandled error or crash.
- **API_Version_Header**: The API version advertised by the services and read by the Mobile_App to detect incompatibility (Req 29 versioning).
- **Add_To_Calendar**: Device calendar integration for a confirmed booking.
- **Share_Sheet**: The native OS share interface for sharing booking details.
- **Get_Directions**: Handoff of a court address/coordinates to the device's maps application.
- **Cold_Start**: App launch from a fully terminated state.

## Requirements

### Requirement 1: Authentication and Onboarding Screens

*Derived from Req 1 (auth flows, biometric, provider linking), Customer User Journey Step 1, and Court Owner Admin Journey Step 1.*

**User Story:** As a returning or new user, I want to log in with OAuth, set up biometric unlock, and accept terms, so that I can access the app securely and quickly on subsequent launches.

#### Acceptance Criteria

1. WHEN an unauthenticated user opens THE Mobile_App, THE Mobile_App SHALL display a login screen offering OAuth_Login options for Google, Facebook, and Apple obtained from `GET /api/auth/providers`.
2. WHEN a user completes OAuth_Login, THE Mobile_App SHALL exchange the result for an Access_Token and Refresh_Token via `POST /api/auth/login` and store the Refresh_Token in the Secure_Enclave.
3. WHERE a new user has not selected a role, THE Mobile_App SHALL prompt the user to select the CUSTOMER or COURT_OWNER role during first registration.
4. WHEN a new user authenticates for the first time, THE Mobile_App SHALL require acceptance of the Terms_and_Conditions before navigating to the home screen.
5. WHERE a returning user has a Refresh_Token stored in the Secure_Enclave, THE Mobile_App SHALL offer Biometric_Authentication to unlock the Refresh_Token and obtain a new Access_Token via `POST /api/auth/refresh`.
6. IF Biometric_Authentication fails 3 consecutive times, THEN THE Mobile_App SHALL fall back to the OAuth_Login screen.
7. IF the Refresh_Token is expired, revoked, or replay is detected, THEN THE Mobile_App SHALL redirect to the OAuth_Login screen and display "Your session has expired, please log in again".
8. IF an OAuth provider is unavailable, THEN THE Mobile_App SHALL display an error with a retry control and the remaining provider options, without blocking the login screen.
9. IF the device has no internet connectivity on the login screen, THEN THE Mobile_App SHALL display a "No internet connection" banner with a retry control and disable the OAuth_Login buttons until connectivity is restored.
10. IF the Terms_and_Conditions content is unreachable, THEN THE Mobile_App SHALL display the cached Terms_and_Conditions version with its "last updated" date and allow acceptance of the cached version.
11. WHERE a user has multiple Linked_Providers, THE Mobile_App SHALL allow login through any Linked_Provider.

### Requirement 2: Account and Profile Management

*Derived from Req 1 (account management, GDPR deletion, provider linking, logout); consumes `GET /api/users/me`, `PUT /api/users/me`, `GET /api/auth/providers`, `POST /api/auth/providers/link`, `POST /api/auth/logout`, `POST /api/users/me/delete`.*

**User Story:** As a user, I want to view and edit my profile, manage my linked login providers, log out, and delete my account, so that I stay in control of my personal data and access.

#### Acceptance Criteria

1. WHEN the user opens the Account_Screen, THE Mobile_App SHALL display the User_Profile from `GET /api/users/me`, including name, email, phone, role, and language.
2. WHEN the user edits name, email, phone, or language and saves, THE Mobile_App SHALL submit the change via `PUT /api/users/me` and update the displayed profile on success.
3. IF `PUT /api/users/me` returns `400`, THEN THE Mobile_App SHALL display the field-level errors from the API_Error_Response `details` and retain the user's entered values.
4. WHEN the user opens the linked-providers section, THE Mobile_App SHALL display the Linked_Providers from `GET /api/auth/providers` and SHALL allow adding a provider via `POST /api/auth/providers/link`.
5. WHEN the user logs out, THE Mobile_App SHALL call `POST /api/auth/logout` to revoke the current device's Refresh_Token, clear the Secure_Enclave entry and local caches, and return to the OAuth_Login screen.
6. WHEN the user requests Account_Deletion, THE Mobile_App SHALL present a confirmation describing the consequences (cancellation of future bookings, refunds processed per policy, anonymization of personal data) before calling `POST /api/users/me/delete`.
7. WHEN `POST /api/users/me/delete` returns `202`, THE Mobile_App SHALL display a "Deletion in progress" confirmation, log the user out, and return to the OAuth_Login screen.
8. IF `POST /api/users/me/delete` returns `409` (court owner has unresolved future bookings), THEN THE Mobile_App SHALL display the blocking reason and direct the Court_Owner to resolve future bookings before retrying.
9. WHERE the user requests a personal-data export, THE Mobile_App SHALL direct the request to the Support_Hub, because a self-service export endpoint is not provided by the current contracts.
10. WHILE the device is offline, THE Mobile_App SHALL display the cached User_Profile read-only and disable profile edit, logout, and deletion with a "Requires internet" indicator.

### Requirement 3: Customer Personalization and Preferences

*Derived from Req 4/Req 5 (personalization), Customer User Journey Step 9; consumes `GET /api/users/me/preferences`, `PUT /api/users/me/preferences`, `GET /api/users/me/skill-levels`, `PUT /api/users/me/skill-levels`.*

**User Story:** As a customer, I want to set my preferred playing days and times, search distance, notification preferences, do-not-disturb hours, and skill levels, so that the app matches my habits and respects my preferences.

#### Acceptance Criteria

1. WHEN the user opens preferences, THE Mobile_App SHALL load User_Preferences via `GET /api/users/me/preferences` and display `preferredDays`, `preferredStartTime`, `preferredEndTime`, `maxSearchDistanceKm`, `notificationPreferences`, `doNotDisturbStart`, and `doNotDisturbEnd`.
2. WHEN the user changes any preference, THE Mobile_App SHALL apply the change locally immediately and sync via `PUT /api/users/me/preferences`, queuing the sync in the Offline_Queue if connectivity is unavailable (see Requirement 24).
3. WHEN the user sets `maxSearchDistanceKm`, THE Mobile_App SHALL use that value as the default Distance_Filter radius on the Map_View.
4. WHEN the user configures Notification_Preferences (`bookingEvents`, `favoriteCourtAlerts`, `promotionalNotifications`, `emailNotifications`, `pushNotifications`), THE Mobile_App SHALL persist them via `PUT /api/users/me/preferences`.
5. WHEN the user configures do-not-disturb hours (`doNotDisturbStart`, `doNotDisturbEnd`), THE Mobile_App SHALL persist them via `PUT /api/users/me/preferences`.
6. WHEN the user opens skill levels, THE Mobile_App SHALL load Skill_Levels via `GET /api/users/me/skill-levels` and SHALL allow editing each sport's level on the 1–7 scale via `PUT /api/users/me/skill-levels`.
7. WHEN the user changes the app language between Greek (`el`) and English (`en`), THE Mobile_App SHALL update the language preference via `PUT /api/users/me` and apply the new language without an app restart (see Requirement 21).
8. IF a preferences sync fails, THEN THE Mobile_App SHALL retain the local change, show a "Pending sync" indicator, and retry on the next connectivity or app launch.

### Requirement 4: Map View with Court Markers and Distance Filter

*Derived from Customer User Journey Step 3; consumes `GET /api/courts/map` (params: `latitude`, `longitude`, `radiusKm`, `minLat`/`maxLat`/`minLng`/`maxLng`, `courtType`, `locationType`, `date`, `startTime`).*

**User Story:** As a customer, I want to see courts on an interactive map with type filters, a distance radius, and clustering, so that I can quickly find courts near me.

#### Acceptance Criteria

1. WHEN the Map_View opens, THE Mobile_App SHALL center the map on the user's current location and request markers for the visible Map_Bounds via `GET /api/courts/map`, including the required `date` parameter for the selected booking date.
2. THE Mobile_App SHALL render court markers color-coded by court type and highlight Favorite_Court markers with a distinct icon.
3. THE Mobile_App SHALL provide Court_Filter tabs (All Courts, Tennis, Padel, Basketball, Football, and Favorites when the user has favorites) and an indoor/outdoor toggle, and SHALL update the displayed markers when a filter changes.
4. THE Mobile_App SHALL provide a Distance_Filter control that sets the `radiusKm` parameter (default 10 km) and SHALL reload markers when the radius changes.
5. WHEN multiple courts occupy the same map area, THE Mobile_App SHALL apply Marker_Clustering and expand clusters as the user zooms in.
6. WHEN the user pans or zooms the map, THE Mobile_App SHALL reload markers for the new Map_Bounds, debounced to a maximum of 1 court search request per 500ms.
7. WHEN the user taps a court marker, THE Mobile_App SHALL display a bottom-sheet preview showing court name, type, indoor/outdoor badge, distance, price per session, and availability indicator, with a "View Details" control.
8. IF location permission is denied or location services are unavailable, THEN THE Mobile_App SHALL center the map on a default city (Athens), display an "Enable location for better results" banner, and allow manual location search via the search bar.
9. IF the court search request fails, THEN THE Mobile_App SHALL keep map tiles visible and display a "Couldn't load courts" banner with a retry control.
10. IF map tiles fail to load, THEN THE Mobile_App SHALL display a fallback list view of courts sorted by distance with a "Map unavailable" message.
11. WHEN the map area contains no courts within the current filters and radius, THE Mobile_App SHALL display an Empty_State suggesting the user widen the Distance_Filter or change filters (see Requirement 29).
12. ⏳ **PHASE 2** — WHERE open matches exist within the Map_Bounds, THE Mobile_App SHALL render open match markers with a "Join" badge and display a match preview (creator skill level, players joined/spots remaining, cost per player) when tapped.

### Requirement 5: Court Discovery — Search, Filter, Favorites, and Weather

*Derived from Customer User Journey Steps 2–3; consumes `GET /api/courts`, `GET /api/weather`, `GET /api/users/me/favorites`, `PUT /api/users/me/favorites/{courtId}`, `DELETE /api/users/me/favorites/{courtId}`.*

**User Story:** As a customer, I want to search and filter courts, save favorites, and preview weather for my selected date, so that I can choose the best court and time.

#### Acceptance Criteria

1. WHEN the user selects a date and time, THE Mobile_App SHALL request the Weather_Forecast for that date via `GET /api/weather` and display temperature, rain probability, wind, and an indoor/outdoor recommendation.
2. WHEN the selected date is beyond the 7-day forecast window, THE Mobile_App SHALL display a "Forecast not yet available" message instead of weather data.
3. WHEN the user searches by location or address, THE Mobile_App SHALL query `GET /api/courts` and display matching courts.
4. WHEN the user marks a court as a Favorite_Court, THE Mobile_App SHALL apply an Optimistic_UI_Update immediately and sync via `PUT /api/users/me/favorites/{courtId}` (expecting `204`) in the background.
5. WHEN the user removes a Favorite_Court, THE Mobile_App SHALL apply an Optimistic_UI_Update immediately and sync via `DELETE /api/users/me/favorites/{courtId}` in the background.
6. IF a favorite add or remove sync request fails, THEN THE Mobile_App SHALL revert the favorite state and display an error toast.
7. WHEN the user opens the favorites list, THE Mobile_App SHALL load it from `GET /api/users/me/favorites`.
8. IF the Weather_Forecast request fails or exceeds 3 seconds, THEN THE Mobile_App SHALL display a "Weather unavailable" placeholder with a retry control and SHALL NOT block the search flow.
9. WHILE the device is offline, THE Mobile_App SHALL allow date and time selection from local state and display "Connect to see forecast" in the weather section.
10. WHEN a court search returns no results, THE Mobile_App SHALL display an Empty_State with guidance to adjust the search terms, date, or filters (see Requirement 29).

### Requirement 6: Court Detail Screen

*Derived from Customer User Journey Step 5; consumes `GET /api/courts/{courtId}`, `GET /api/courts/{courtId}/availability`, `GET /api/courts/{courtId}/cancellation-policy`, `GET /api/weather`, `GET /api/courts/{courtId}/ratings`.*

**User Story:** As a customer, I want to view full court details with images, amenities, availability, weather, and cancellation policy, so that I can decide whether to book.

#### Acceptance Criteria

1. WHEN the user opens the Court_Detail_Screen, THE Mobile_App SHALL display the court image carousel, Localized_Content name and description, type, indoor/outdoor badge, amenities, address with a Get_Directions control, distance, price breakdown, and Cancellation_Policy summary from `GET /api/courts/{courtId}` and `GET /api/courts/{courtId}/cancellation-policy`.
2. WHEN the Court_Detail_Screen loads, THE Mobile_App SHALL request available time slots for the selected date via `GET /api/courts/{courtId}/availability` and display them with an availability indicator (available, pending, unavailable).
3. WHERE the court is outdoor, THE Mobile_App SHALL display the Weather_Forecast for the selected date alongside the court details.
4. THE Mobile_App SHALL load images, availability, and (⏳ Phase 2) ratings as independent sections so that a failure in one section does not block the others.
5. IF an individual section fails to load, THEN THE Mobile_App SHALL display a per-section error state with a retry control and keep successfully loaded sections visible.
6. IF court images fail to load, THEN THE Mobile_App SHALL display a placeholder image with the court-type icon and retry the image load on scroll-into-view.
7. WHEN the user taps the Get_Directions control, THE Mobile_App SHALL hand the court coordinates and address to the device maps application (see Requirement 30).
8. WHEN the user taps a heart control, THE Mobile_App SHALL toggle the Favorite_Court state per Requirement 5.
9. ⏳ **PHASE 2** — THE Mobile_App SHALL display the court's average rating and recent reviews from `GET /api/courts/{courtId}/ratings`.

### Requirement 7: Booking Flow — Slot Selection, Confirmation, Payment, and Receipt

*Derived from Customer User Journey Steps 6–8; consumes `POST /api/bookings` (with `X-Idempotency-Key`), `GET /api/payments/methods`, `POST /api/payments/setup-intent`, `GET /api/courts/{courtId}/cancellation-policy`, `GET /api/bookings/{bookingId}/receipt`, and the Stripe_SDK.*

**User Story:** As a customer, I want to select a slot, confirm booking details with the cancellation policy, pay securely, and receive a receipt, so that I can reserve a court with confidence.

#### Acceptance Criteria

1. WHEN the user selects an available slot and proceeds, THE Mobile_App SHALL display a booking confirmation summary showing court, date, time, duration, number-of-people selector (bounded by court capacity), total price, the Cancellation_Policy summary, and the Booking_Confirmation_Mode indicator ("Instant confirmation" or "Requires owner approval").
2. WHEN the user proceeds to payment, THE Mobile_App SHALL present saved payment methods from `GET /api/payments/methods` and an option to add a new card via the Stripe_SDK and `POST /api/payments/setup-intent`.
3. WHERE Apple Pay or Google Pay is available on the device, THE Mobile_App SHALL offer it as a payment option; otherwise THE Mobile_App SHALL hide it and show card entry only.
4. WHEN the user confirms payment, THE Mobile_App SHALL create the booking via `POST /api/bookings` including a client-generated `X-Idempotency-Key` and the Stripe_SDK payment confirmation.
5. WHEN `POST /api/bookings` returns `201` for an instant-confirmation court, THE Mobile_App SHALL display a confirmation screen with booking details, an Add_To_Calendar option, a Share_Sheet option, and access to the Digital_Receipt via `GET /api/bookings/{bookingId}/receipt`.
6. WHEN `POST /api/bookings` returns `201` for a manual-confirmation court, THE Mobile_App SHALL display a "Booking pending — awaiting owner confirmation" screen indicating the payment is held and that a Push_Notification will follow on the owner's decision.
7. WHEN a Stripe 3D Secure authentication is required, THE Mobile_App SHALL handle the redirect flow and wait up to 5 minutes for the user to complete authentication before timing out.
8. IF payment fails with a card decline or `402`, THEN THE Mobile_App SHALL display the decline reason from the API_Error_Response `message` field and offer "Try another payment method".
9. IF `POST /api/bookings` returns `422` (`BOOKING_WINDOW_EXCEEDED` or `MINIMUM_NOTICE_NOT_MET`), THEN THE Mobile_App SHALL display the corresponding business-rule message and SHALL NOT retry automatically.
10. IF `POST /api/bookings` returns `409` with alternative slots in the `BookingConflictResponse`, THEN THE Mobile_App SHALL display "This time slot was just booked by someone else" and present the returned alternative slots (see Requirement 16).
11. ⏳ **PHASE 2** — THE Mobile_App SHALL offer an "open match" toggle (with skill-range selector), a "split payment" option (invite players, show per-person cost), a promo code field, and a "Join Waitlist" action when a slot is fully booked.

### Requirement 8: My Bookings Management

*Derived from Customer User Journey Step 9, Req 14; consumes `GET /api/bookings`, `GET /api/bookings/{bookingId}`, `PUT /api/bookings/{bookingId}/modify`, `POST /api/bookings/{bookingId}/cancel`, `GET /api/courts/{courtId}/cancellation-policy`.*

**User Story:** As a customer, I want to view and manage my bookings grouped by status, see refund previews before cancelling, and modify reservations, so that I can track and change my bookings.

#### Acceptance Criteria

1. WHEN the user opens My_Bookings, THE Mobile_App SHALL retrieve bookings via `GET /api/bookings` and group them by Booking_Status as upcoming (`CONFIRMED`), pending confirmation (`PENDING_CONFIRMATION`), past (`COMPLETED`), cancelled (`CANCELLED`), and rejected (`REJECTED`).
2. WHEN the user filters bookings, THE Mobile_App SHALL pass the supported query parameters (`status`, `fromDate`, `toDate`, `courtId`, `courtType`, `page`, `size`) to `GET /api/bookings`, noting that the `status` filter accepts only `CONFIRMED`, `PENDING_CONFIRMATION`, `CANCELLED`, or `COMPLETED`.
3. WHEN the user taps a booking, THE Mobile_App SHALL display full booking details from `GET /api/bookings/{bookingId}`, including payment status, No_Show_Flag, and cancellation policy.
4. WHEN the user initiates a cancellation, THE Mobile_App SHALL display a Refund_Preview computed from `GET /api/courts/{courtId}/cancellation-policy` and the time remaining before the booking, and SHALL require confirmation before proceeding.
5. WHEN the user confirms a cancellation, THE Mobile_App SHALL display a "Cancelling..." Optimistic_UI_Update, call `POST /api/bookings/{bookingId}/cancel` with an `X-Idempotency-Key`, and display the actual refund amount from the response or revert on failure.
6. WHEN the user modifies a booking, THE Mobile_App SHALL submit the new date/time (and duration where applicable) via `PUT /api/bookings/{bookingId}/modify` and update the displayed booking on success.
7. IF a modification returns `409 Conflict`, THEN THE Mobile_App SHALL display "This time slot is no longer available" and refresh the court's availability so the user can pick another time.
8. WHEN a `BOOKING_STATUS_UPDATE` WebSocket message is received for a listed booking, THE Mobile_App SHALL update that booking's status, No_Show_Flag, payment status, and refund amount in the list without a full reload.
9. WHILE the device is offline, THE Mobile_App SHALL display cached booking history with a "Last updated X minutes ago" label and disable cancel and modify actions.
10. WHEN the user has no bookings in a group, THE Mobile_App SHALL display an Empty_State for that group with guidance to discover and book a court (see Requirement 29).

### Requirement 9: Recurring Bookings (Customer)

*Derived from Req 14 (recurring bookings), Customer Journey Step 9; consumes `POST /api/bookings/recurring` (with `X-Idempotency-Key`), `GET /api/bookings/recurring/{recurringGroupId}`.*

**User Story:** As a customer, I want to create and manage recurring bookings as a group, so that I can reserve a regular slot and handle partial conflicts clearly.

#### Acceptance Criteria

1. WHERE the user chooses to repeat a booking, THE Mobile_App SHALL collect the recurrence configuration (frequency, day(s), and end condition) and submit it via `POST /api/bookings/recurring` with an `X-Idempotency-Key`.
2. WHEN `POST /api/bookings/recurring` returns `201`, THE Mobile_App SHALL display the created Recurring_Booking_Group and its individual instances.
3. WHEN `POST /api/bookings/recurring` returns `207 Multi-Status`, THE Mobile_App SHALL display a breakdown of successful and conflicting dates with reasons and offer "Confirm partial booking" or "Cancel all" options.
4. WHEN the user opens a Recurring_Booking_Group, THE Mobile_App SHALL load its instances via `GET /api/bookings/recurring/{recurringGroupId}` and display each instance's date, time, and Booking_Status.
5. WHEN the user cancels a single instance of a Recurring_Booking_Group, THE Mobile_App SHALL call `POST /api/bookings/{bookingId}/cancel` for that instance with an `X-Idempotency-Key` and update only that instance.
6. WHILE the device is offline, THE Mobile_App SHALL display the cached Recurring_Booking_Group read-only and disable creation and cancellation actions.

### Requirement 10: Notifications — Push Handling, In-App Center, and Deep Linking

*Derived from Req 13 (client side), Req 31.42, the WebSocket `NOTIFICATION` contract, and Customer Journey Step 9; consumes `POST /api/notifications/device`, `GET /api/notifications`, `POST /api/notifications/{notificationId}/read`, `POST /api/notifications/read-all`.*

**User Story:** As a user, I want to receive push and in-app notifications and review them in a notification center, so that I stay informed about my bookings.

#### Acceptance Criteria

1. WHEN a user grants notification permission, THE Mobile_App SHALL obtain the FCM/APNs Device_Registration_Token and register it via `POST /api/notifications/device`.
2. WHILE the app is active and the WebSocket is connected, THE Mobile_App SHALL receive In_App_Notification messages on `/user/queue/notifications` and surface them in-app.
3. WHILE the app is backgrounded or closed, THE Mobile_App SHALL display Push_Notification messages delivered via FCM/APNs.
4. WHEN the user opens the Notification_Center, THE Mobile_App SHALL list notification history via `GET /api/notifications` and indicate read/unread state.
5. WHEN the user reads a notification, THE Mobile_App SHALL mark it read via `POST /api/notifications/{notificationId}/read`, and SHALL support marking all read via `POST /api/notifications/read-all`.
6. WHEN a notification carries a `data.deepLink` value, THE Mobile_App SHALL open the corresponding in-app screen (e.g., `/bookings/{id}`) when the notification is tapped (see Requirement 17).
7. WHEN a Push_Notification about a booking status change is received while backgrounded, THE Mobile_App SHALL update the local booking cache so the data is fresh when the user next opens the app.
8. THE Mobile_App SHALL render the supported `notificationType` values (including `BOOKING_CONFIRMED`, `BOOKING_CANCELLED`, `BOOKING_REJECTED`, `BOOKING_REMINDER`, `PAYMENT_RECEIVED`, `PAYMENT_FAILED`, `REFUND_COMPLETED`, `PAYOUT_COMPLETED`, `VERIFICATION_APPROVED`, `VERIFICATION_REJECTED`, `REMINDER_ALERT`, `WEATHER_ALERT`, `SUPPORT_TICKET_UPDATED`) with appropriate titles, bodies, and deep links.
9. THE Mobile_App SHALL gracefully ignore notification types it does not handle (e.g., ⏳ Phase 2 `MATCH_*`, `SPLIT_PAYMENT_*`, `WAITLIST_*` types) without crashing or displaying malformed content.
10. WHEN the Notification_Center has no entries, THE Mobile_App SHALL display an Empty_State (see Requirement 29).

### Requirement 11: Court Owner Operations — Courts, Dashboard, Calendar, Bookings, and Analytics

*Derived from the Court Owner Admin Journey (Steps 4–8), Req 9 (manual bookings), Req 14 (no-show, mark-paid, history), Req 15 (analytics), Req 15b (reminder alerts); consumes `GET /api/courts/owner/me`, `GET /api/courts/{courtId}`, `PUT /api/courts/{courtId}`, `GET /api/dashboard`, `GET /api/bookings`, `GET /api/bookings/pending`, `POST /api/bookings/{bookingId}/confirm`, `POST /api/bookings/{bookingId}/reject`, `POST /api/bookings/manual`, `POST /api/bookings/{bookingId}/no-show`, `POST /api/bookings/{bookingId}/mark-paid`, `GET /api/bookings/calendar/ical`, `GET /api/analytics/revenue`, `GET /api/analytics/usage`, `GET/PUT /api/settings/notification-preferences`.*

**User Story:** As a court owner, I want to manage courts, view a dashboard and calendar, create manual and recurring bookings, act on pending bookings, flag no-shows, mark manual bookings paid, and review analytics from my phone, so that I can run my facility on the go.

#### Acceptance Criteria

1. WHEN a COURT_OWNER opens the owner area, THE Mobile_App SHALL display the owner's courts from `GET /api/courts/owner/me` and the Owner_Dashboard summary from `GET /api/dashboard` (today's bookings, pending confirmations, revenue, and Reminder_Alerts).
2. WHEN the Owner_Dashboard reports Reminder_Alerts, THE Mobile_App SHALL display an "action required" badge with the count and link each alert to the relevant booking.
3. WHEN the owner opens the Booking_Calendar, THE Mobile_App SHALL display bookings across owned courts from `GET /api/bookings` color-coded by Booking_Status, with filters for court, status, and date range.
4. WHEN the owner views the pending bookings queue, THE Mobile_App SHALL list bookings from `GET /api/bookings/pending` with customer detail, requested time, held amount, and a countdown to auto-cancellation, and SHALL allow confirm via `POST /api/bookings/{bookingId}/confirm` or reject via `POST /api/bookings/{bookingId}/reject`.
5. WHEN the owner creates a Manual_Booking, THE Mobile_App SHALL submit `POST /api/bookings/manual` with the selected court, slot, and optional customer contact info, applying the same client-side slot validation as customer bookings.
6. WHERE the owner makes a Manual_Booking recurring, THE Mobile_App SHALL include the recurrence configuration in the `POST /api/bookings/manual` request and handle a `207 Multi-Status` response with a per-date success/conflict breakdown.
7. WHEN the owner flags a booking as a no-show, THE Mobile_App SHALL call `POST /api/bookings/{bookingId}/no-show` and reflect the No_Show_Flag in the booking detail.
8. WHEN the owner marks a Manual_Booking as paid externally, THE Mobile_App SHALL call `POST /api/bookings/{bookingId}/mark-paid` and update the displayed payment status.
9. WHEN the owner edits a court, THE Mobile_App SHALL submit updates via `PUT /api/courts/{courtId}` and surface server validation errors (e.g., capacity below existing bookings, future-booking conflicts) inline without losing entered data.
10. WHEN the owner opens analytics, THE Mobile_App SHALL display revenue and usage summaries from `GET /api/analytics/revenue` and `GET /api/analytics/usage`.
11. WHERE the owner requests calendar export, THE Mobile_App SHALL retrieve the iCal file via `GET /api/bookings/calendar/ical` and hand it to the device's calendar/share sheet (see Requirement 30).
12. WHEN the owner opens owner notification settings, THE Mobile_App SHALL load Owner_Notification_Preferences via `GET /api/settings/notification-preferences` and persist changes via `PUT /api/settings/notification-preferences`.
13. ⏳ **PHASE 2** — THE Mobile_App SHALL display trial countdown banners, subscription tier selection, and scheduled-report configuration for court owners; until Phase 10 all court owners are treated as subscribed and these screens are not shown.

### Requirement 12: Court Owner Verification and Stripe Connect Onboarding

*Derived from Req 2 (owner verification), Req 11a / Req 1 (Stripe Connect onboarding and `stripeConnected` gating); consumes `POST /api/verification`, `GET /api/verification`, `GET /api/users/me/stripe-connect`, `POST /api/users/me/stripe-connect/onboard`, `GET /api/users/me/stripe-connect/payouts`, `PUT /api/users/me/stripe-connect/payout-schedule`.*

**User Story:** As a court owner, I want to submit verification documents, track verification status, and complete Stripe Connect onboarding, so that my courts become visible and I can accept customer bookings and receive payouts.

#### Acceptance Criteria

1. WHERE a COURT_OWNER has not submitted verification, THE Mobile_App SHALL display a "Verification required" banner and SHALL allow submitting business documents via `POST /api/verification`.
2. WHEN the owner submits a Verification_Request, THE Mobile_App SHALL display the resulting Verification_Status from `GET /api/verification` (`PENDING_REVIEW`, `APPROVED`, or `REJECTED`).
3. IF `POST /api/verification` returns `409` (a request is already pending), THEN THE Mobile_App SHALL display "A verification request is already pending review" and show the current status.
4. WHILE the Verification_Status is `PENDING_REVIEW`, THE Mobile_App SHALL display a pending banner and SHALL indicate that courts are not yet publicly visible.
5. WHEN the Verification_Status is `REJECTED`, THE Mobile_App SHALL display the rejection reason from `GET /api/verification` and allow resubmission via `POST /api/verification`.
6. WHEN the owner opens payment onboarding, THE Mobile_App SHALL display the Stripe_Account_Status from `GET /api/users/me/stripe-connect` (`status`, `requiresAction`, `payoutSchedule`, `nextPayoutDate`).
7. WHERE the Stripe_Account_Status is not active or `requiresAction` is true, THE Mobile_App SHALL display a status banner and initiate Stripe_Connect_Onboarding via `POST /api/users/me/stripe-connect/onboard`, opening the returned hosted onboarding URL.
8. WHEN the user returns to the app after Stripe_Connect_Onboarding, THE Mobile_App SHALL refresh the Stripe_Account_Status via `GET /api/users/me/stripe-connect` and allow resuming onboarding if it is incomplete.
9. WHEN the owner views payouts, THE Mobile_App SHALL display payout history from `GET /api/users/me/stripe-connect/payouts` and SHALL allow changing the payout schedule via `PUT /api/users/me/stripe-connect/payout-schedule`.
10. WHERE a Court_Owner is unverified or Stripe Connect is not active, THE Mobile_App SHALL keep customer-booking-dependent owner features in a consistent disabled or informational state aligned with the owner's sub-state.

### Requirement 13: Real-Time Updates via WebSocket Client Resilience

*Derived from Req 31.20–24, Req 31.33, Req 35, and the WebSocket message contracts (`/topic/courts/{courtId}/availability`, `/user/queue/bookings`, `/user/queue/notifications`, `/user/queue/system`, `AVAILABILITY_UPDATE`, `AVAILABILITY_SNAPSHOT`, reconnection strategy, close codes `4001`–`4005`).*

**User Story:** As a user viewing live data, I want real-time availability and booking updates that recover gracefully from disconnections, so that I see accurate information.

#### Acceptance Criteria

1. WHEN the user views a court's availability, THE Mobile_App SHALL connect to the Transaction_Service WebSocket at `/ws?token=<jwt>` and subscribe to `/topic/courts/{courtId}/availability`, `/user/queue/bookings`, `/user/queue/notifications`, and `/user/queue/system`.
2. WHEN an `AVAILABILITY_UPDATE` or `AVAILABILITY_SNAPSHOT` message is received, THE Mobile_App SHALL replace its local slot state for the affected date(s) entirely (non-incremental) as specified by the contract.
3. WHEN the WebSocket connection drops, THE Mobile_App SHALL display a subtle "Live updates paused" indicator on screens that depend on real-time data.
4. WHEN the WebSocket connection drops, THE Mobile_App SHALL attempt automatic reconnection with Exponential_Backoff (1s, 2s, 4s, 8s, 16s, max 30s) plus ±500ms jitter to avoid Thundering_Herd.
5. WHEN the WebSocket connection is re-established, THE Mobile_App SHALL fetch a full availability refresh for the currently viewed court and dismiss the "Live updates paused" indicator.
6. WHILE the WebSocket has been down for more than 60 seconds, THE Mobile_App SHALL poll availability every 10 seconds for the currently viewed court via `GET /api/courts/{courtId}/availability`.
7. WHEN displayed availability becomes stale (WebSocket disconnected or more than 30 seconds since the last update), THE Mobile_App SHALL display an "Availability may have changed" indicator and refresh availability when the user taps "Book Now".
8. WHEN the server sends a `SERVER_SHUTDOWN` message or closes with code `4005`, THE Mobile_App SHALL reconnect after the suggested `reconnectAfterMs` delay using Exponential_Backoff.
9. WHEN the server closes the connection with code `4001` (authentication expired), THE Mobile_App SHALL obtain a fresh Access_Token and reconnect, falling back to the login flow if token refresh fails (see Requirement 18).
10. WHEN the app transitions from background to foreground, THE Mobile_App SHALL re-establish a dropped WebSocket connection and refresh the current screen's real-time data.
11. THE Mobile_App SHALL keep WebSocket message sends within the server limit of 10 messages per second and SHALL treat connection-level `ERROR` messages (codes such as `RATE_LIMITED`, `INVALID_SUBSCRIPTION`) as non-fatal warnings unless `willDisconnect` is true.

### Requirement 14: Offline Data Caching and Graceful Degradation

*Derived from Req 31.7–11 and the Mobile App Offline Behavior principles.*

**User Story:** As a user with intermittent connectivity, I want to browse cached data offline and have read-only access preserved, so that the app remains useful without a connection.

#### Acceptance Criteria

1. THE Mobile_App SHALL cache locally (SQLite or Hive) with a 24-hour Cache_TTL: court search results (last 3 searches), Court_Detail_Screen data (last 10 viewed), User_Profile, User_Preferences, Favorite_Court list, booking history (last 50 bookings), and weather forecasts (last fetched).
2. WHILE the device is offline, THE Mobile_App SHALL allow browsing of cached court details, viewing booking history, accessing favorite courts, and reading bundled FAQ articles.
3. WHEN displaying cached data, THE Mobile_App SHALL show a "Last updated X minutes ago" label.
4. WHEN cached data is older than the Cache_TTL, THE Mobile_App SHALL show a "Data may be outdated" warning and prioritize refresh on the next connectivity.
5. WHILE the device is offline, THE Mobile_App SHALL disable booking creation, payment, cancellation, modification, rating submission, waitlist join, and match join, showing a disabled state with a "Requires internet" tooltip.

### Requirement 15: Network Resilience, Loading States, and Error Handling

*Derived from Req 31.1–6, Req 31.12–16, Req 31.38–39, the API Error Response Contract, and the HTTP Status Code Handling table.*

**User Story:** As a user on a variable network, I want responsive loading states, clear errors, and sensible retries, so that the app feels reliable.

#### Acceptance Criteria

1. WHEN THE Mobile_App detects no internet connectivity, THE Mobile_App SHALL display a persistent "No connection" banner on every screen and disable all network-dependent actions.
2. WHEN connectivity is restored, THE Mobile_App SHALL dismiss the "No connection" banner, refresh the current screen's data, and process the Offline_Queue.
3. WHEN an API request fails with a transient error (5xx, timeout, or network error) on a GET request, THE Mobile_App SHALL auto-retry once after 2 seconds with Exponential_Backoff; for non-GET requests, THE Mobile_App SHALL display an error with a manual "Retry" control.
4. WHEN an API response takes less than 300ms, THE Mobile_App SHALL NOT show a loading indicator.
5. WHEN an API response takes between 300ms and 2 seconds, THE Mobile_App SHALL show Skeleton_UI placeholders matching the expected layout.
6. WHEN an API response takes between 2 and 5 seconds, THE Mobile_App SHALL show a prominent loading overlay with a contextual message (e.g., "Searching for courts...").
7. WHEN a non-payment API request exceeds 10 seconds, THE Mobile_App SHALL auto-cancel the request and show "Something went wrong" with retry and "Contact Support" options.
8. WHEN an API response carries an API_Error_Response body, THE Mobile_App SHALL display the localized `message` field to the user and use the `error` code for programmatic handling.
9. IF a response is `403`, THEN THE Mobile_App SHALL show "You don't have permission" and SHALL NOT retry; IF a response is `404`, THEN THE Mobile_App SHALL show "This item is no longer available" and navigate back or remove the item from cache.
10. WHEN a response is `429`, THE Mobile_App SHALL read the `Retry-After` header, show a countdown timer, and disable the action until the timer expires.
11. WHEN the aggregated map endpoint (`GET /api/courts/map`) returns Partial_Failure data, THE Mobile_App SHALL display the available data and show placeholders for failed sections (e.g., "Weather unavailable") without blocking the screen.
12. THE Mobile_App SHALL throttle availability refresh to a maximum of 1 request per 5 seconds per court.

### Requirement 16: Payment Flow Resilience

*Derived from Req 31.25–30, Req 31.33, and Customer Journey Step 8a (Booking Error Recovery). Note: the services do not expose a `PENDING_PAYMENT` booking status filter, so incomplete-booking detection is client-side.*

**User Story:** As a customer paying for a booking, I want the payment flow to prevent duplicates and recover from interruptions, so that I am never double-charged and never lose a confirmed booking.

#### Acceptance Criteria

1. WHEN the "Confirm and Pay" control is tapped, THE Mobile_App SHALL immediately disable the control and show a spinner to prevent double submission, re-enabling only on explicit failure.
2. THE Mobile_App SHALL include a client-generated `X-Idempotency-Key` (UUID v4) with every booking creation request so that retries do not create duplicate bookings within the server's 24-hour deduplication window.
3. WHEN a payment request exceeds 10 seconds but may have been processed server-side, THE Mobile_App SHALL display "Your payment is being processed — please do not retry" and poll the booking via `GET /api/bookings` every 5 seconds for up to 60 seconds before offering a "Contact Support" option with the payment reference.
4. WHEN a payment fails with a retryable error (network timeout or Stripe temporary error), THE Mobile_App SHALL allow one manual retry using the same `X-Idempotency-Key`; if the retry also fails, THE Mobile_App SHALL show "Payment could not be processed" with "Try different payment method" and "Contact Support" options.
5. WHEN THE Mobile_App is killed or crashes during an active payment flow, THE Mobile_App SHALL persist the in-flight booking's `X-Idempotency-Key` and, on next launch, reconcile by re-issuing the booking creation request with the same key (which the server deduplicates) or by checking recent bookings via `GET /api/bookings`, then display a "You have an incomplete booking" banner with resume or cancel options.
6. IF payment succeeded but booking creation failed server-side, THEN THE Mobile_App SHALL display "Your payment was received but we encountered an issue creating your booking. Your payment is safe", show the payment reference, and rely on the server-side reconciliation job and a follow-up Push_Notification for resolution.
7. WHEN a booking creation returns `409 Conflict`, THE Mobile_App SHALL display "This time slot was just booked by someone else", present the alternative slots from the `BookingConflictResponse`, offer "Pick another time" (refreshing availability), and, where waitlist is enabled, a "Join Waitlist" option (⏳ Phase 2).
8. ⏳ **PHASE 2** — WHEN a recurring booking creation returns `207 Multi-Status`, THE Mobile_App SHALL display a breakdown of successful and conflicting dates with "Confirm partial booking" or "Cancel all" options (also covered for recurring bookings in Requirement 9).

### Requirement 17: Deep Linking from Notifications and External Links

*Derived from Req 31.42, the WebSocket `NOTIFICATION` `data.deepLink` field, and Customer/Court Owner journeys.*

**User Story:** As a user, I want tapping a notification to take me directly to the relevant screen, so that I can act on it without navigating manually.

#### Acceptance Criteria

1. WHEN the user taps a Push_Notification or In_App_Notification that carries a `data.deepLink`, THE Mobile_App SHALL navigate to the corresponding in-app screen (e.g., a booking detail, court detail, or support ticket).
2. WHILE the user is unauthenticated when a Deep_Link is opened, THE Mobile_App SHALL route to the OAuth_Login screen and, after successful authentication, navigate to the original Deep_Link target.
3. IF a Deep_Link references a resource that no longer exists, THEN THE Mobile_App SHALL display "This item is no longer available" and navigate to a safe default screen.
4. WHEN a Deep_Link is opened from the device's Cold_Start state, THE Mobile_App SHALL complete app initialization and authentication before resolving the Deep_Link target.

### Requirement 18: Session Management and Background App Behavior

*Derived from Req 31.34–37 and Req 31.41–43, Req 35, and the WebSocket `TOKEN_EXPIRING` / `TOKEN_REFRESH` contract on `/user/queue/system` and `/app/token-refresh`.*

**User Story:** As a user, I want my session to refresh transparently and my place in the app to be preserved, so that I am not interrupted by token expiry or app backgrounding.

#### Acceptance Criteria

1. WHEN any API call returns `401 Unauthorized`, THE Mobile_App SHALL attempt a Silent_Token_Refresh using the stored Refresh_Token without interrupting the user.
2. WHEN the Silent_Token_Refresh succeeds, THE Mobile_App SHALL transparently retry the original failed request.
3. IF the Silent_Token_Refresh fails (Refresh_Token expired, revoked, or replay detected), THEN THE Mobile_App SHALL redirect to the login screen with "Your session has expired, please log in again" and preserve the current navigation state for restoration after re-login.
4. THE Mobile_App SHALL proactively refresh the Access_Token when it is within 60 seconds of expiration.
5. WHEN the WebSocket sends a `TOKEN_EXPIRING` message on `/user/queue/system`, THE Mobile_App SHALL obtain a fresh Access_Token and send a `TOKEN_REFRESH` message on `/app/token-refresh` within 60 seconds to keep the connection authenticated.
6. WHEN THE Mobile_App returns to the foreground after more than 5 minutes in the background, THE Mobile_App SHALL refresh the Access_Token and the current screen's data.
7. WHEN THE Mobile_App is force-killed and relaunched by an authenticated user, THE Mobile_App SHALL restore the last active screen and refresh its data.

### Requirement 19: Client-Side Validation

*Derived from Req 31.44–46.*

**User Story:** As a user, I want immediate feedback on invalid input, so that I can correct mistakes before submitting, while trusting the server as the final authority.

#### Acceptance Criteria

1. THE Mobile_App SHALL validate, before sending API requests, that the booking date/time is in the future, the booking duration matches the court configuration, the number of people does not exceed court capacity, and any promo code is alphanumeric and 3–20 characters.
2. WHEN client-side validation fails, THE Mobile_App SHALL show inline field-level error messages and SHALL NOT make an API call.
3. THE Mobile_App SHALL treat all server-side validation responses (e.g., `400`, `409`, `422`) as authoritative and SHALL display server-provided field errors from the API_Error_Response `details` even when client-side validation passed.

### Requirement 20: Optimistic UI and Offline Action Queue

*Derived from Req 31.17–19 and the Optimistic UI principles.*

**User Story:** As a user, I want quick actions like favoriting and preference changes to feel instant and survive connectivity gaps, so that the app is responsive.

#### Acceptance Criteria

1. WHEN the user toggles a Favorite_Court, THE Mobile_App SHALL apply an Optimistic_UI_Update and sync in the background, reverting the UI and showing an error toast if the sync fails.
2. WHEN the user updates preferences (language, search distance, notification settings), THE Mobile_App SHALL apply the change locally immediately and queue the server sync, retrying on the next app launch if the sync fails.
3. THE Mobile_App SHALL maintain an Offline_Queue of pending sync operations (maximum 50 items) and process them in order when connectivity is available.
4. THE Mobile_App SHALL display a "Pending sync" indicator on actions that are queued but not yet confirmed by the server.

### Requirement 21: Localization and Internationalization

*Derived from Req 28 (multi-language), Req 31 Error Message Localization, and the API Error Response Contract.*

**User Story:** As a Greek- or English-speaking user, I want all content, dates, currency, and error messages in my language and local formats, so that the app is understandable.

#### Acceptance Criteria

1. WHEN THE Mobile_App first launches, THE Mobile_App SHALL detect the device language and default to Greek (`el`) or English (`en`) accordingly.
2. WHEN the user changes the app language, THE Mobile_App SHALL update all UI text immediately without an app restart and persist the choice via `PUT /api/users/me`.
3. THE Mobile_App SHALL send the user's preferred language in the `Accept-Language` header on all API requests.
4. WHEN an API_Error_Response is received, THE Mobile_App SHALL display its localized `message` field rather than a raw HTTP status or technical error.
5. THE Mobile_App SHALL localize client-generated messages (network errors, timeouts, validation messages) in the user's preferred language.
6. THE Mobile_App SHALL present errors in friendly, non-technical language (e.g., "We couldn't complete your booking" rather than "HTTP 500 Internal Server Error").
7. WHEN displaying Localized_Content such as court names and descriptions, THE Mobile_App SHALL show the user's preferred language (`el`/`en`) and, when content is unavailable in that language, fall back to the court owner's primary language with a "translated content not available" indicator.
8. THE Mobile_App SHALL format Euro_Cents amounts as localized Euro currency and SHALL format dates, times, and numbers according to the active locale.

### Requirement 22: Support Hub and Diagnostics

*Derived from Req 30 (customer support), Customer/Court Owner Journey Step 9, and the API Error Response Contract `correlationId`; consumes `POST /api/support/tickets`, `GET /api/support/tickets`, `GET /api/support/tickets/{ticketId}`, `POST /api/support/tickets/{ticketId}/messages`, `POST /api/support/tickets/{ticketId}/close`.*

**User Story:** As a user with a problem, I want to read FAQs and submit a support ticket with relevant context, so that I can get help quickly.

#### Acceptance Criteria

1. THE Mobile_App SHALL provide a Support_Hub with searchable FAQ articles bundled with the app (updated via app releases).
2. WHEN the user submits a Support_Ticket via `POST /api/support/tickets`, THE Mobile_App SHALL auto-attach relevant context (e.g., the related booking) and the Correlation_Id from any recent API_Error_Response.
3. WHERE the user opts to attach diagnostics, THE Mobile_App SHALL include a sanitized Diagnostic_Log with the ticket, with secrets and PII removed.
4. WHEN the user opens an existing ticket, THE Mobile_App SHALL display the conversation thread from `GET /api/support/tickets/{ticketId}` and allow replies via `POST /api/support/tickets/{ticketId}/messages`.
5. WHEN the user closes a resolved ticket, THE Mobile_App SHALL call `POST /api/support/tickets/{ticketId}/close` and reflect the closed state.
6. WHEN the user opens the ticket list, THE Mobile_App SHALL load it from `GET /api/support/tickets` and display an Empty_State when no tickets exist (see Requirement 29).
7. WHILE the device is offline, THE Mobile_App SHALL allow reading bundled FAQ articles and SHALL disable ticket submission with a "Requires internet" indicator.

### Requirement 23: Accessibility

*Derived from the platform accessibility-compliance intent for client applications.*

**User Story:** As a user who relies on assistive technology or accessibility settings, I want the app to be usable with a screen reader, scalable fonts, sufficient contrast, and adequate tap targets, so that I can book courts independently.

#### Acceptance Criteria

1. THE Mobile_App SHALL provide screen-reader labels (semantic labels) for all interactive controls, images, and status indicators.
2. WHEN the device font scale is increased, THE Mobile_App SHALL scale text accordingly and SHALL keep content readable and operable without truncation that hides essential information.
3. THE Mobile_App SHALL meet a minimum text contrast ratio of 4.5:1 for normal text and 3:1 for large text against its background.
4. THE Mobile_App SHALL provide interactive tap targets of at least 44x44 logical pixels.
5. WHEN conveying status (availability, booking state, errors), THE Mobile_App SHALL NOT rely on color alone and SHALL include text or iconography.
6. WHEN a screen loads or an error occurs, THE Mobile_App SHALL announce the change to assistive technology via accessibility live-region semantics.
7. THE Mobile_App SHALL support the device's reduce-motion setting by limiting non-essential animation.

### Requirement 24: App Permissions Handling

*Derived from device-capability needs across the journeys (location, notifications, biometrics, calendar, camera/photos).*

**User Story:** As a user, I want the app to request device permissions in context and handle denial gracefully, so that I understand why a permission is needed and the app remains usable without it.

#### Acceptance Criteria

1. WHEN a feature first needs location, notifications, biometrics, calendar, or camera/photos access, THE Mobile_App SHALL request the corresponding App_Permission in context with an explanation of why it is needed.
2. IF location permission is denied, THEN THE Mobile_App SHALL fall back to manual location search and default-city centering per Requirement 4.
3. IF notification permission is denied, THEN THE Mobile_App SHALL continue to deliver In_App_Notifications and SHALL surface a setting to enable push notifications later.
4. IF biometric permission or enrollment is unavailable, THEN THE Mobile_App SHALL fall back to OAuth_Login per Requirement 1.
5. IF calendar permission is denied, THEN THE Mobile_App SHALL offer the Share_Sheet or an iCal file as an alternative to Add_To_Calendar (see Requirement 30).
6. IF camera or photos permission is denied when attaching a support attachment or profile photo, THEN THE Mobile_App SHALL display an explanation and a link to device settings.
7. WHERE a required permission was permanently denied, THE Mobile_App SHALL display guidance and a deep link to the OS app settings.

### Requirement 25: Client Security and Data Protection

*Derived from Req 1 (Secure_Enclave token storage), Req 33/Req 34 (security hardening, data protection) applied to the client.*

**User Story:** As a user, I want my credentials and payment interactions protected on my device, so that my account and financial data are safe.

#### Acceptance Criteria

1. THE Mobile_App SHALL store the Refresh_Token only in the Secure_Enclave and SHALL NOT persist the Access_Token or Refresh_Token in plain-text storage or application logs.
2. THE Mobile_App SHALL transmit all API and WebSocket traffic over TLS.
3. THE Mobile_App SHOULD apply Certificate_Pinning to the backend endpoints to resist man-in-the-middle interception, with a configured fallback to avoid lockout on certificate rotation.
4. THE Mobile_App SHALL exclude tokens, payment details, and personal data from the Diagnostic_Log and SHALL sanitize logs before attaching them to a Support_Ticket.
5. WHILE a payment or card-entry screen is displayed, THE Mobile_App SHALL set the OS secure-screen flag to suppress screenshots and exclude the screen from the app switcher preview where the platform supports it.
6. WHERE the device is detected as jailbroken or rooted, THE Mobile_App SHOULD warn the user that secure storage guarantees may be reduced.
7. WHEN the app is backgrounded, THE Mobile_App SHALL obscure sensitive content (payment and account screens) in the app switcher preview.

### Requirement 26: Client Analytics, Telemetry, and Crash Reporting

*Derived from the platform observability intent (Req 16) applied to the client.*

**User Story:** As a product and engineering team, I want privacy-respecting client analytics and crash reports, so that I can improve the app without collecting sensitive data.

#### Acceptance Criteria

1. THE Mobile_App SHALL emit Telemetry_Events for screen views, key actions, and performance timings without including PII or payment data.
2. WHEN an unhandled error or crash occurs, THE Mobile_App SHALL capture a Crash_Report with a stack trace and non-PII context and report it on the next available connectivity.
3. WHERE a Correlation_Id is available from a recent API_Error_Response, THE Mobile_App SHALL include it in the associated Telemetry_Event or Crash_Report for cross-system tracing.
4. THE Mobile_App SHALL provide a setting allowing the user to opt out of analytics and crash reporting.
5. THE Mobile_App SHALL NOT transmit Telemetry_Events or Crash_Reports to any endpoint other than the project's designated observability destination.

### Requirement 27: API Client Generation, Versioning, and Forward Compatibility

*Derived from Req 29 (OpenAPI contracts, Dart codegen, semantic versioning, deprecation) and the Phase 2 roadmap forward-compatibility notes.*

**User Story:** As a developer, I want the Dart API client generated from the OpenAPI contracts and the app to tolerate API evolution, so that backend changes do not break the app and Phase 2 endpoints can be added safely.

#### Acceptance Criteria

1. THE Mobile_App SHALL generate its Dart API client from `docs/api/openapi-platform-service.yaml` and `docs/api/openapi-transaction-service.yaml` using openapi-generator.
2. WHEN deserializing API responses, THE Mobile_App SHALL ignore unknown or newer fields rather than failing, to remain forward-compatible with additive contract changes.
3. WHEN THE Mobile_App receives an API_Version_Header indicating an unsupported major version, THE Mobile_App SHALL display an "Update required" screen and direct the user to update the app.
4. THE Mobile_App SHALL gracefully ignore unknown enum values (e.g., new `notificationType` or status values) rather than crashing.
5. WHERE Phase 2 endpoints (matches, split payments, waitlist, promo codes) are not yet available, THE Mobile_App SHALL keep the corresponding UI hidden or disabled and SHALL NOT call those endpoints.

### Requirement 28: Platform Coverage and Platform-Specific Behavior

*Derived from the repository description (Flutter for iOS, Android, and Web) and platform payment/push/biometric differences.*

**User Story:** As a user on iOS, Android, or Web, I want the app to use the right platform capabilities, so that payment, push, and biometrics work natively on my device.

#### Acceptance Criteria

1. THE Mobile_App SHALL run on iOS, Android, and Web from a single Flutter codebase.
2. WHERE the platform is iOS, THE Mobile_App SHALL offer Apple Pay (when available) and use APNs for Push_Notifications and the iOS biometric APIs (Face ID / Touch ID) for Biometric_Authentication.
3. WHERE the platform is Android, THE Mobile_App SHALL offer Google Pay (when available) and use FCM for Push_Notifications and the Android biometric APIs for Biometric_Authentication.
4. WHERE the platform is Web, THE Mobile_App SHALL gracefully disable capabilities not supported on Web (e.g., native biometrics, native push where unavailable) and present supported alternatives without errors.
5. THE Mobile_App SHALL register the platform-appropriate Device_Registration_Token via `POST /api/notifications/device`.

### Requirement 29: Empty States and Zero-Results

*Derived from Req 3.6 (suggestions on no results) and the journeys' loading/empty handling.*

**User Story:** As a user, I want clear, actionable guidance when a screen has no content, so that I know what to do next.

#### Acceptance Criteria

1. WHEN a court search or map query returns zero courts, THE Mobile_App SHALL display an Empty_State suggesting the user widen the Distance_Filter, change filters, or adjust the search terms.
2. WHEN My_Bookings has no bookings in a group, THE Mobile_App SHALL display an Empty_State for that group with a control to discover and book a court.
3. WHEN the Notification_Center has no entries, THE Mobile_App SHALL display an Empty_State indicating there are no notifications yet.
4. WHEN the favorites list is empty, THE Mobile_App SHALL display an Empty_State guiding the user to add favorites from court details or the map.
5. WHEN the Support_Hub ticket list is empty, THE Mobile_App SHALL display an Empty_State with a control to create a new ticket.

### Requirement 30: Device Integrations — Add to Calendar, Share, and Get Directions

*Derived from Customer Journey Step 8 (add-to-calendar, share) and Step 5 (get directions); consumes `GET /api/bookings/calendar/ical` for the owner calendar export.*

**User Story:** As a user, I want to add bookings to my calendar, share booking details, and get directions to a court, so that I can integrate bookings with my device.

#### Acceptance Criteria

1. WHEN the user chooses Add_To_Calendar for a confirmed booking, THE Mobile_App SHALL create a device calendar event with the court name, address, date, and time after obtaining calendar permission per Requirement 24.
2. WHEN the user chooses to share a booking, THE Mobile_App SHALL present the native Share_Sheet with the booking summary.
3. WHEN the user taps Get_Directions on a court, THE Mobile_App SHALL hand the court address and coordinates to the device's maps application.
4. WHEN a Court_Owner exports the Booking_Calendar, THE Mobile_App SHALL retrieve the iCal file via `GET /api/bookings/calendar/ical` and present it to the device's calendar or Share_Sheet.

### Requirement 31: Non-Functional Requirements — Performance, Footprint, and Efficiency

*Derived from Req 19 (performance/caching), Req 31 latency tiers, and mobile resource constraints.*

**User Story:** As a user, I want the app to start quickly, respond within reasonable latency budgets, and be efficient with data and battery, so that it is pleasant to use on a phone.

#### Acceptance Criteria

1. THE Mobile_App SHALL align its loading-state thresholds with the latency tiers of Requirement 15 (no indicator under 300ms, Skeleton_UI 300ms–2s, overlay 2s–5s, auto-cancel non-payment requests over 10s).
2. WHEN performing a Cold_Start on a representative mid-range device, THE Mobile_App SHALL display interactive content (authenticated home or login) within 3 seconds.
3. THE Mobile_App SHALL debounce map pan/zoom court searches to at most 1 request per 500ms and throttle availability refresh to at most 1 request per 5 seconds per court, to limit data and battery usage.
4. THE Mobile_App SHALL bound local cache growth consistent with the Cache_TTL limits in Requirement 14 (last 3 searches, last 10 court details, last 50 bookings) and evict the oldest entries when limits are exceeded.
5. WHILE the app is backgrounded, THE Mobile_App SHALL release the WebSocket connection and SHALL NOT poll availability, re-establishing real-time updates on return to foreground per Requirement 13.
6. THE Mobile_App SHALL load court images at sizes appropriate to the display and SHALL lazy-load off-screen images to conserve data.
