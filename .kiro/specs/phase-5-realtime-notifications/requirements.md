# Requirements Document — Phase 5: Real-time & Notifications

## Introduction

Phase 5 implements the complete real-time communication and notification delivery subsystem for the Court Booking Platform, targeting both `court-booking-transaction-service` (primary) and `court-booking-platform-service` (WebSocket availability broadcasting). This phase delivers the Kafka notification event consumer that processes `NOTIFICATION_REQUESTED` events and routes them to the appropriate delivery channels, Firebase Cloud Messaging (FCM) integration for mobile push notifications, SendGrid integration for email notifications, STOMP-over-WebSocket infrastructure with JWT authentication and Redis Pub/Sub for horizontal scaling, W3C Web Push API with VAPID keys for the admin web portal, device token and browser push subscription management, notification persistence for the in-app notification center, do-not-disturb scheduling and urgency-based delivery enforcement, real-time court availability broadcasting via WebSocket triggered by booking events, and booking reminder scheduling via Quartz jobs.

This phase builds on Phase 4 (Booking & Payments), which established the booking lifecycle, payment processing, Stripe Connect, recurring bookings, and Quartz-based scheduled jobs. Phase 4 publishes `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic via `NotificationEventKafkaPublisher` and `NoOpNotificationEventPublisher`, but no consumer processes them yet. Phase 3 (Court Management) established notification preferences management (`NotificationPreferencesController`, `NotificationPreference` domain model), court owner reminder rules (`ReminderRule`, `ReminderRulesController`), and the `BookingEventKafkaConsumer` in Platform Service for availability cache invalidation. Phase 5 extends the `BookingEventKafkaConsumer` to also broadcast availability updates via WebSocket.

**Master requirements coverage:** Req 13 (Asynchronous Notification System — all 25 acceptance criteria), Req 19 (Performance, Scalability, and Caching — WebSocket/real-time criteria 11, 12, 22, 25, 26).

**Dual-channel delivery rationale:** WebSocket connections only work when the app has an active TCP connection to the server. When the mobile app is backgrounded, killed by the OS, or the phone is locked, the WebSocket connection is dead. FCM uses a system-level persistent connection managed by the OS (Google Play Services on Android, APNs on iOS) that works even when the app is closed. Both channels are needed:
- **WebSocket**: Real-time in-app updates when the app is active (instant, no OS intermediary)
- **FCM/APNs**: Push notifications when the app is backgrounded or closed (OS-level delivery)
- **Web Push**: Browser notifications when the admin portal tab is closed or minimized
- **Email (SendGrid)**: Asynchronous notifications for booking confirmations, receipts, and reminders

**Existing infrastructure leveraged by Phase 5:**
- Both services already publish `NOTIFICATION_REQUESTED` events to the `notification-events` Kafka topic (28 notification types defined in `kafka-event-contracts.json`)
- Platform Service publishes: VERIFICATION_APPROVED/REJECTED, SUBSCRIPTION_*, SUPPORT_TICKET_UPDATED, WEATHER_ALERT, REMINDER_ALERT
- Transaction Service publishes: BOOKING_CONFIRMED/CANCELLED/REJECTED, BOOKING_PENDING_CONFIRMATION, BOOKING_REMINDER, PAYMENT_*, REFUND_*, PENDING_CONFIRMATION_REMINDER, RECURRING_BOOKING_PRICE_CHANGED
- Notification preferences are already managed in Platform Service (`NotificationPreferencesController`, per-event-type channel config with push/email/inApp toggles)
- Court owner reminder rules are already implemented in Platform Service (`ReminderRule` with BOOKING_REMINDER, PENDING_CONFIRMATION, LOW_OCCUPANCY, DAILY_SUMMARY types)
- `PendingConfirmationReminderJob` and `BookingCompletionJob` Quartz jobs already exist in Transaction Service (skeleton implementations publishing notification events)
- Redis is already used for caching, slot holds, idempotency, and webhook deduplication
- `BookingEventKafkaConsumer` in Platform Service already consumes `booking-events` for availability cache invalidation
- Database tables `notifications` and `device_tokens` were created in Phase 1b Flyway migrations

**Platform Service deliverables in Phase 5:**
Phase 5 is primarily a Transaction Service phase, but the following Platform Service changes are required:
- New internal endpoint: `GET /internal/users/{userId}/notification-preferences` (added to `InternalUserController`) — returns notification preferences and DND configuration for cross-service consumption
- New internal endpoint: `GET /internal/courts/{courtId}/reminder-rules` (added to `InternalCourtController`) — returns court reminder rules for cross-service consumption
- Extend `BookingEventKafkaConsumer` to handle `SLOT_HELD` and `SLOT_RELEASED` events (currently only handles BOOKING_CREATED/CONFIRMED/CANCELLED/MODIFIED) and publish availability updates to Redis Pub/Sub
- Add `availability_update_latency_seconds` histogram metric to Platform Service
- Add DND configuration storage to the `platform.court_owner_notif_prefs` table (new columns: `dnd_enabled`, `dnd_start_time`, `dnd_end_time`, `dnd_timezone`)

**Scope boundaries:**
- Notification preferences CRUD is already implemented in Phase 3 (Platform Service). Phase 5 consumes these preferences when routing notifications.
- Reminder rule configuration is already implemented in Phase 3 (Platform Service). Phase 5 implements the actual reminder delivery.
- Kafka event publishing is already implemented in Phases 3-4. Phase 5 implements the consumer side.
- Analytics event publishing to `analytics-events` topic is Phase 6.
- Security hardening (abuse detection, notification spam prevention) is Phase 7.
- Open match notifications, waitlist notifications, and split payment notifications are ⏳ Phase 10.
- The following notification types from `kafka-event-contracts.json` are defined but NOT delivered in Phase 5 (deferred to their respective feature phases): WAITLIST_SLOT_AVAILABLE, WAITLIST_EXPIRED, MATCH_JOIN_REQUEST, MATCH_JOIN_APPROVED, MATCH_JOIN_DECLINED, MATCH_PLAYER_LEFT, SPLIT_PAYMENT_REQUESTED, SPLIT_PAYMENT_RECEIVED, SPLIT_PAYMENT_EXPIRED. The Notification Processor SHALL gracefully ignore these types if consumed.

## Glossary

- **Transaction_Service**: The Spring Boot microservice responsible for bookings, payments, notifications, and scheduled jobs. Hosts the notification processor, WebSocket infrastructure, FCM adapter, SendGrid adapter, and Web Push adapter. Deployed to `court-booking-transaction-service`.
- **Platform_Service**: The Spring Boot microservice responsible for authentication, authorization, user management, courts, availability, and weather. Hosts the `BookingEventKafkaConsumer` which Phase 5 extends to broadcast availability updates via WebSocket. Deployed to `court-booking-platform-service`.
- **Notification_Processor**: Kafka consumer in Transaction Service that reads `NOTIFICATION_REQUESTED` events from the `notification-events` topic and routes them to the appropriate delivery channels (FCM, WebSocket, SendGrid, Web Push) based on user preferences, app state, and urgency level.
- **FCM**: Firebase Cloud Messaging — Google's cross-platform messaging service for delivering push notifications to Android and iOS devices via a system-level persistent connection managed by the OS.
- **APNs**: Apple Push Notification service — Apple's push notification delivery system, accessed through FCM's unified API for cross-platform delivery.
- **SendGrid**: Twilio SendGrid — email delivery service used for booking confirmations, receipts, reminders, and court owner notifications. Accessed via SendGrid Java SDK.
- **STOMP**: Simple Text Oriented Messaging Protocol — a sub-protocol layered over WebSocket providing message framing, subscriptions, and destinations. Used for structured real-time messaging.
- **WebSocket**: Full-duplex communication protocol over a single TCP connection. Used for real-time in-app notifications and availability updates when the client has an active connection.
- **Redis_Pub_Sub**: Redis publish/subscribe messaging used as the WebSocket message broker backend, enabling horizontal scaling of WebSocket connections across multiple Transaction Service pods.
- **Web_Push**: W3C Push API standard for delivering notifications to web browsers. Uses VAPID (Voluntary Application Server Identification) keys for server authentication. Used for the admin web portal.
- **VAPID_Keys**: Voluntary Application Server Identification keys — an ECDSA key pair used to authenticate the application server with the browser push service. The public key is shared with the browser; the private key signs push messages.
- **Device_Token**: A unique identifier issued by FCM for a specific app installation on a specific device. Stored in the `transaction.device_tokens` table, encrypted at rest. Used to target push notifications.
- **Browser_Push_Subscription**: A JSON object containing the browser push endpoint URL, p256dh key, and auth secret. Stored in the `transaction.device_tokens` table with `token_type = 'WEB_PUSH'`. Used to target Web Push notifications.
- **Notification_Preference**: Per-event-type channel configuration (push/email/inApp toggles) managed by Platform Service. Consumed by the Notification Processor to determine delivery channels.
- **Urgency_Level**: Classification of notification importance: CRITICAL (always delivered immediately, bypasses DND), STANDARD (respects do-not-disturb windows), PROMOTIONAL (respects opt-out preferences).
- **Do_Not_Disturb**: User-configured quiet hours during which STANDARD and PROMOTIONAL notifications are queued and delivered after the quiet period ends. CRITICAL notifications bypass DND.
- **Notification_Center**: In-app UI backed by the `transaction.notifications` table where users can view their notification history, mark notifications as read, and manage notification state.
- **Exponential_Backoff**: Retry strategy where the delay between retries increases exponentially (e.g., 1s, 2s, 4s, 8s, max 60s) to avoid overwhelming a failing service.
- **Heartbeat**: Periodic STOMP HEARTBEAT frames (30-second interval) used to detect stale WebSocket connections. Connections are closed after 2 missed heartbeats (60 seconds of silence).
- **Graceful_Shutdown**: During rolling deployments, Transaction Service sends a DISCONNECT frame with a reconnect signal to connected WebSocket clients, then waits up to 30 seconds for connections to drain before shutting down.
- **Client_Reconnection**: Exponential backoff strategy for WebSocket reconnection on the client side (1s, 2s, 4s, max 30s) to avoid thundering herd on server restart.

## Requirements


### Requirement 1: Notification Event Processing and Channel Routing

**User Story:** As a platform operator, I want a centralized notification processor that consumes notification events from Kafka and routes them to the appropriate delivery channels, so that users receive timely notifications through the right medium based on their preferences and app state.

#### Acceptance Criteria

1. WHEN a `NOTIFICATION_REQUESTED` event is consumed from the `notification-events` Kafka topic, THE Notification_Processor SHALL determine the delivery channels based on the event's `channels` array, the user's notification preferences (from Platform Service), the user's current app state (WebSocket connection status), and the notification's urgency level
2. WHEN the Notification_Processor starts, THE Notification_Processor SHALL join the `transaction-service-notification-consumer` consumer group and consume from all partitions of the `notification-events` topic with `auto.offset.reset=latest`
3. WHEN a notification event is processed, THE Notification_Processor SHALL persist the notification in the `transaction.notifications` table with status `PENDING` before attempting delivery, and update the status to `DELIVERED`, `PARTIALLY_DELIVERED`, or `FAILED` after delivery attempts complete
4. WHEN a notification event has `urgency = 'CRITICAL'`, THE Notification_Processor SHALL deliver the notification immediately through all available channels regardless of user preferences and do-not-disturb settings
5. WHEN a notification event has `urgency = 'STANDARD'` and the user has active do-not-disturb hours, THE Notification_Processor SHALL queue the notification and deliver it after the quiet period ends
6. WHEN a notification event has `urgency = 'PROMOTIONAL'` and the user has opted out of the notification type, THE Notification_Processor SHALL skip delivery and mark the notification as `SKIPPED_OPT_OUT` in the notifications table
7. WHEN the target user has an active WebSocket connection, THE Notification_Processor SHALL deliver the notification via WebSocket (in-app) as the primary channel
8. WHEN the target user does not have an active WebSocket connection and has registered FCM device tokens, THE Notification_Processor SHALL deliver the notification via FCM push
9. WHEN the notification event's `channels` array includes `EMAIL`, THE Notification_Processor SHALL deliver the notification via SendGrid email in addition to push/WebSocket delivery
10. WHEN the notification event's `channels` array includes `WEB_PUSH` and the target user has registered browser push subscriptions, THE Notification_Processor SHALL deliver the notification via Web Push API
11. THE Notification_Processor SHALL use consumer-side idempotency based on the event's `eventId` field to prevent duplicate notification delivery. Duplicate events SHALL be logged and skipped. THE Notification_Processor SHALL store processed `eventId` values in Redis with a 7-day TTL using the key pattern `notification:processed:{eventId}`
12. WHEN a malformed notification event is consumed, THE Notification_Processor SHALL log the error at WARN level and skip the event without blocking the consumer
13. THE Notification_Processor SHALL process notification events with a target latency of less than 500ms from Kafka consumption to delivery initiation

### Requirement 2: FCM Device Token Management

**User Story:** As a mobile app user, I want my device to be registered for push notifications when I log in, so that I receive important booking updates even when the app is closed or backgrounded.

#### Acceptance Criteria

1. WHEN a user logs into the mobile app, THE Transaction_Service SHALL accept a device token registration via `POST /api/notifications/devices` and associate the FCM token with the authenticated user's ID
   - **Input:** `{ token: string, platform: enum(ANDROID|IOS), deviceId?: string }`
   - **Output (success):** `201 Created` with `{ id: UUID, userId: UUID, platform: string, registeredAt: ISO8601 }`
   - **Output (duplicate token):** `200 OK` with existing registration (idempotent)
   - **Output (invalid token):** `400 Bad Request` with `{ error: "INVALID_DEVICE_TOKEN" }`
2. WHEN a user logs in from multiple devices, THE Transaction_Service SHALL maintain multiple device registration tokens per user ID in the `transaction.device_tokens` table with `token_type = 'FCM'`
3. WHEN a user logs out, THE Transaction_Service SHALL remove the device token for the current device via `DELETE /api/notifications/devices/{deviceId}`
   - **Output (success):** `204 No Content`
   - **Output (not found):** `404 Not Found`
4. WHEN a user retrieves their registered devices, THE Transaction_Service SHALL return the list via `GET /api/notifications/devices`
   - **Output:** `200 OK` with `{ devices: [{ id: UUID, platform: string, deviceId: string, registeredAt: ISO8601, lastUsedAt: ISO8601 }] }` — token values are never returned in API responses
5. THE Transaction_Service SHALL store device registration tokens in the `transaction.device_tokens` table with encryption at rest using AES-256-GCM, with the encryption key managed via environment variable `DEVICE_TOKEN_ENCRYPTION_KEY`
6. WHEN FCM reports a device token as invalid (via the `messaging/registration-token-not-registered` error), THE Transaction_Service SHALL automatically remove the invalid token from the `device_tokens` table
7. WHEN a device token has not been used for push delivery in 90 days, THE Transaction_Service SHALL mark it as expired and remove it during the next cleanup cycle
8. THE Transaction_Service SHALL enforce a maximum of 10 device tokens per user. WHEN a user registers an 11th device, THE Transaction_Service SHALL remove the oldest (least recently used) token

### Requirement 3: FCM Push Notification Delivery

**User Story:** As a mobile app user, I want to receive push notifications about booking confirmations, cancellations, and reminders even when the app is closed, so that I never miss important updates.

#### Acceptance Criteria

1. WHEN the Notification_Processor routes a notification to the FCM channel, THE Transaction_Service SHALL send the notification to all registered FCM device tokens for the target user using the Firebase Admin SDK. WHEN a user has multiple device tokens, THE Transaction_Service SHALL use FCM's `sendEachForMulticast` API to send to all tokens in a single API call
2. WHEN sending an FCM push notification, THE Transaction_Service SHALL construct the message with the localized `title` and `body` from the notification event (using the user's preferred language from `v_user_basic.language`), the `data` payload for client-side deep linking, and the appropriate Android/iOS platform-specific configuration
3. WHEN FCM push delivery fails for a specific device token, THE Transaction_Service SHALL retry with exponential backoff (1s, 2s, 4s, 8s, 16s, max 60s) for up to 3 retry attempts before marking the delivery as failed for that token
4. WHEN all FCM delivery attempts fail for all device tokens of a user, THE Transaction_Service SHALL update the notification status to `PUSH_FAILED` in the notifications table and log the failure
5. WHEN FCM returns a `messaging/registration-token-not-registered` error for a token, THE Transaction_Service SHALL remove the invalid token immediately (within the same processing cycle) and continue delivery to remaining tokens
6. THE Transaction_Service SHALL configure FCM messages with appropriate priority: `HIGH` priority for CRITICAL urgency notifications (wakes the device), `NORMAL` priority for STANDARD and PROMOTIONAL notifications
7. THE Transaction_Service SHALL include a `click_action` field in FCM messages pointing to the deep link URL from the notification event's `data.deepLink` field, enabling direct navigation when the user taps the notification


### Requirement 4: WebSocket Infrastructure with STOMP and JWT Authentication

**User Story:** As a mobile app or admin portal user, I want a persistent real-time connection to the server, so that I receive instant in-app notifications and availability updates without polling.

#### Acceptance Criteria

1. THE Transaction_Service SHALL expose a STOMP-over-WebSocket endpoint at `/ws` using Spring WebSocket with SockJS fallback, routed through NGINX Ingress via the `/ws/*` path prefix
2. WHEN a client initiates a WebSocket connection, THE Transaction_Service SHALL authenticate the connection by validating the JWT token provided as a query parameter (`/ws?token={jwt}`) or in the STOMP CONNECT frame headers. Invalid or expired tokens SHALL result in connection rejection with a `401` close code
3. WHEN a WebSocket connection is authenticated, THE Transaction_Service SHALL extract the user ID and roles from the JWT claims and associate them with the WebSocket session for authorization of topic subscriptions
4. THE Transaction_Service SHALL support the following STOMP destination prefixes for client subscriptions:
   - `/user/queue/notifications` — personal notification delivery (user-specific)
   - `/topic/courts/{courtId}/availability` — real-time availability updates for a specific court
   - `/topic/courts/{courtId}/bookings` — real-time booking status updates for court owners
5. WHEN a client subscribes to `/user/queue/notifications`, THE Transaction_Service SHALL deliver notifications targeted at the authenticated user via this personal queue
6. WHEN a client subscribes to `/topic/courts/{courtId}/availability`, THE Transaction_Service SHALL validate that the court exists and deliver real-time availability updates for that court
7. WHEN a client subscribes to `/topic/courts/{courtId}/bookings`, THE Transaction_Service SHALL validate that the authenticated user is the court owner and reject unauthorized subscriptions with a STOMP ERROR frame
8. THE Transaction_Service SHALL configure STOMP heartbeat at 30-second intervals (server-to-client and client-to-server). WHEN 2 consecutive heartbeats are missed (60 seconds of silence), THE Transaction_Service SHALL close the WebSocket connection
9. WHEN a WebSocket connection is closed unexpectedly, THE Transaction_Service SHALL clean up the session from the in-memory connection registry and update the user's connection status
10. THE Transaction_Service SHALL track active WebSocket connections in an in-memory concurrent map keyed by user ID, supporting multiple concurrent connections per user (e.g., mobile app + admin portal)

### Requirement 5: Redis Pub/Sub for Horizontal WebSocket Scaling

**User Story:** As a platform operator, I want WebSocket messages to be delivered to clients regardless of which pod they are connected to, so that the system scales horizontally without message loss.

#### Acceptance Criteria

1. THE Transaction_Service SHALL use Redis Pub/Sub as the message broker for WebSocket message distribution across multiple pods, using dedicated Redis channels per STOMP destination
2. WHEN a notification or availability update needs to be broadcast, THE Transaction_Service SHALL publish the message to the appropriate Redis Pub/Sub channel. All Transaction Service pods subscribed to that channel SHALL receive the message and deliver it to locally connected WebSocket clients
3. THE Transaction_Service SHALL use the following Redis channel naming convention:
   - `ws:user:{userId}:notifications` — for user-specific notification delivery
   - `ws:court:{courtId}:availability` — for court availability broadcasts
   - `ws:court:{courtId}:bookings` — for court booking status broadcasts
4. WHEN Redis becomes unavailable, THE Transaction_Service SHALL degrade gracefully: WebSocket connections SHALL continue on the local pod, notifications SHALL be delivered to locally connected clients only, and a `redis_pubsub_unavailable` metric SHALL be incremented
5. WHEN Redis recovers from an outage, THE Transaction_Service SHALL automatically re-establish Pub/Sub subscriptions without requiring pod restarts. THE Transaction_Service SHALL use a Redis connection listener to detect reconnection and re-subscribe to all active channels
6. THE WebSocket infrastructure SHALL support up to 10,000 concurrent connections per Transaction Service pod, validated through load testing
7. THE Transaction_Service SHALL publish a `websocket_connections_active` gauge metric and a `websocket_messages_delivered_total` counter metric for monitoring

### Requirement 6: Real-time Availability Broadcasting

**User Story:** As a customer browsing court availability, I want to see availability updates in real-time without refreshing, so that I can make booking decisions based on current information.

#### Acceptance Criteria

1. WHEN a booking event (BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, BOOKING_MODIFIED, SLOT_HELD, SLOT_RELEASED) is consumed from the `booking-events` Kafka topic by Platform Service, THE Platform_Service SHALL publish an availability update message to the Redis Pub/Sub channel `ws:court:{courtId}:availability` in addition to invalidating the availability cache. THE Platform_Service uses the same Redis instance as Transaction Service (configured via shared `REDIS_HOST` and `REDIS_PORT` environment variables). NOTE: The existing `BookingEventKafkaConsumer` currently handles BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, and BOOKING_MODIFIED. Phase 5 SHALL extend it to also handle SLOT_HELD and SLOT_RELEASED events for availability broadcasting. BOOKING_COMPLETED is intentionally excluded from availability broadcasting because the slot was already occupied — completion does not change availability state
2. WHEN a Transaction Service pod receives an availability update from Redis Pub/Sub, THE Transaction_Service SHALL broadcast the update to all WebSocket clients subscribed to `/topic/courts/{courtId}/availability`. NOTE: Platform Service does NOT maintain WebSocket connections directly; all WebSocket connections are managed by Transaction Service
3. THE availability update message SHALL include: `{ courtId: UUID, date: ISO8601-date, eventType: string, affectedSlot: { startTime: HH:mm, endTime: HH:mm }, updatedAt: ISO8601 }` — sufficient for the client to update its local availability display. Valid `eventType` values: `BOOKING_CREATED`, `BOOKING_CONFIRMED`, `BOOKING_CANCELLED`, `BOOKING_MODIFIED`, `SLOT_HELD`, `SLOT_RELEASED`
4. WHEN a booking is created or cancelled, THE availability update SHALL propagate to connected WebSocket clients within 2 seconds end-to-end (from Kafka event production to WebSocket delivery)
5. THE Platform_Service SHALL measure and report the `availability_update_latency_seconds` histogram metric (from Kafka event timestamp to Redis Pub/Sub publish) for monitoring the 2-second SLA
6. WHEN a court owner confirms or rejects a pending booking, THE Transaction_Service SHALL publish a booking status update to the Redis Pub/Sub channel `ws:court:{courtId}:bookings` for real-time admin portal updates. WHEN a booking is modified (BOOKING_MODIFIED), THE Transaction_Service SHALL also publish a booking status update to the same channel

### Requirement 7: In-App Notification Delivery via WebSocket

**User Story:** As a user with the app open, I want to receive notifications instantly within the app without a push notification banner, so that I have a seamless in-app experience.

#### Acceptance Criteria

1. WHEN the Notification_Processor routes a notification to the IN_APP channel and the target user has an active WebSocket connection, THE Transaction_Service SHALL deliver the notification via the user's personal STOMP queue `/user/queue/notifications`
2. THE WebSocket notification message SHALL include: `{ notificationId: UUID, type: string, urgency: string, title: string, body: string, data: object, createdAt: ISO8601, read: false }` — using the user's preferred language for title and body
3. WHEN WebSocket delivery fails (connection dropped between check and send), THE Transaction_Service SHALL fall back to FCM push notification delivery for the same notification
4. WHEN a user has multiple active WebSocket connections (e.g., mobile app and admin portal), THE Transaction_Service SHALL deliver the notification to all active connections
5. THE Transaction_Service SHALL provide a `GET /api/notifications` endpoint for retrieving the user's notification history from the notifications table
   - **Input:** `GET /api/notifications?page=0&size=20&unreadOnly=false&type={notificationType}`
   - **Output:** `200 OK` with `{ content: [{ notificationId: UUID, type: string, urgency: string, title: string, body: string, data: object, read: boolean, createdAt: ISO8601 }], page: number, size: number, totalElements: number, totalPages: number }`
   - **Pagination defaults:** page=0, size=20 (max 100), sort=createdAt DESC
6. THE Transaction_Service SHALL provide a `POST /api/notifications/{notificationId}/read` endpoint for marking a notification as read
   - **Output (success):** `200 OK` with `{ notificationId: UUID, read: true, readAt: ISO8601 }`
   - **Output (not found):** `404 Not Found`
7. THE Transaction_Service SHALL provide a `POST /api/notifications/read-all` endpoint for marking all notifications as read
   - **Output:** `200 OK` with `{ updatedCount: number }`
8. THE Transaction_Service SHALL provide a `GET /api/notifications/unread-count` endpoint for retrieving the count of unread notifications
   - **Output:** `200 OK` with `{ unreadCount: number }`


### Requirement 8: Email Notification Delivery via SendGrid

**User Story:** As a court owner, I want to receive email notifications for booking events, so that I have a reliable record of all booking activity even when I'm not using the app or portal.

#### Acceptance Criteria

1. WHEN the Notification_Processor routes a notification to the EMAIL channel, THE Transaction_Service SHALL send the email via the SendGrid API using the SendGrid Java SDK
2. WHEN sending an email notification, THE Transaction_Service SHALL use pre-defined SendGrid dynamic templates for each notification type, with template variables populated from the notification event's `title`, `body`, and `data` fields
3. THE Transaction_Service SHALL support the following email notification categories with dedicated SendGrid templates:
   - Booking confirmation (customer + court owner)
   - Booking cancellation (customer + court owner)
   - Booking rejection (customer)
   - Booking reminder (customer)
   - Payment receipt (customer)
   - Refund confirmation (customer)
   - Pending confirmation alert (court owner)
   - Pending confirmation reminder (court owner)
   
   The following notification types do NOT have dedicated email templates in Phase 5 and SHALL NOT be delivered via email even if the `channels` array includes `EMAIL`: PAYMENT_RECEIVED (covered by payment receipt), PAYOUT_COMPLETED (Stripe sends its own payout emails), PAYMENT_DISPUTE (handled via Stripe dashboard notifications), RECURRING_BOOKING_PRICE_CHANGED (push/in-app only for MVP). Email templates for these types may be added in future phases.
4. WHEN SendGrid email delivery fails, THE Transaction_Service SHALL retry with exponential backoff (2s, 4s, 8s) for up to 3 retry attempts before marking the email delivery as failed
5. THE Transaction_Service SHALL configure SendGrid with a verified sender domain and use the platform's `noreply@` address as the sender
6. WHEN an email notification is sent, THE Transaction_Service SHALL include an unsubscribe link in the email footer that links to the notification preferences page in the admin portal or mobile app
7. THE Transaction_Service SHALL use the user's preferred language (from `v_user_basic.language`) to select the appropriate email template localization (Greek or English)

### Requirement 9: Web Push Notifications for Admin Portal

**User Story:** As a court owner using the admin web portal, I want to receive browser push notifications for new bookings and pending confirmations even when the portal tab is closed, so that I can respond to booking requests promptly.

#### Acceptance Criteria

1. THE Transaction_Service SHALL generate and store a VAPID key pair (ECDSA P-256) for Web Push authentication, with the private key stored as an environment variable `VAPID_PRIVATE_KEY` and the public key exposed via `GET /api/notifications/web-push/vapid-key`
   - **Output:** `200 OK` with `{ publicKey: string }` (Base64url-encoded VAPID public key)
2. WHEN a court owner grants browser push permission in the admin portal, THE Transaction_Service SHALL accept the browser push subscription via `POST /api/notifications/web-push/subscribe`
   - **Input:** `{ endpoint: string, keys: { p256dh: string, auth: string } }`
   - **Output (success):** `201 Created` with `{ subscriptionId: UUID, registeredAt: ISO8601 }`
   - **Output (duplicate endpoint):** `200 OK` with existing subscription (idempotent)
3. THE Transaction_Service SHALL store browser push subscriptions in the `transaction.device_tokens` table with `token_type = 'WEB_PUSH'` and the subscription JSON (endpoint, p256dh, auth) encrypted at rest
4. WHEN the Notification_Processor routes a notification to the WEB_PUSH channel, THE Transaction_Service SHALL send the notification to all registered browser push subscriptions for the target user using the Web Push protocol (RFC 8030) with VAPID authentication (RFC 8292)
5. WHEN a browser push delivery fails with a `410 Gone` or `404 Not Found` response from the push service, THE Transaction_Service SHALL remove the expired subscription from the `device_tokens` table
6. WHEN a court owner explicitly unsubscribes from browser push via `DELETE /api/notifications/web-push/subscribe`, THE Transaction_Service SHALL remove the subscription
   - **Output (success):** `204 No Content`
7. WHEN a browser push subscription has not received a successful delivery in 90 days, THE Transaction_Service SHALL mark it as expired and remove it during the next cleanup cycle

### Requirement 10: Notification Preference Enforcement and Do-Not-Disturb

**User Story:** As a user, I want to control which notifications I receive and through which channels, and I want to set quiet hours during which non-urgent notifications are held, so that I am not disturbed at inappropriate times.

#### Acceptance Criteria

1. WHEN the Notification_Processor processes a notification event, THE Notification_Processor SHALL fetch the user's notification preferences from Platform Service (via internal API `GET /internal/users/{userId}/notification-preferences`) and apply channel filtering based on the per-event-type push/email/inApp toggles
2. WHEN a user has disabled a specific channel (e.g., `pushEnabled = false`) for a notification type, THE Notification_Processor SHALL skip delivery on that channel for that notification type, unless the notification urgency is CRITICAL
3. WHEN a notification has `urgency = 'CRITICAL'`, THE Notification_Processor SHALL deliver through all available channels regardless of user preference toggles and do-not-disturb settings. CRITICAL notifications include: booking cancellations, payment failures, Stripe Connect status changes to RESTRICTED
4. WHEN a notification has `urgency = 'STANDARD'` and the current time falls within the user's configured do-not-disturb window, THE Notification_Processor SHALL queue the notification in the `transaction.notifications` table with status `DND_QUEUED` and a `deliver_after` timestamp set to the end of the DND window
5. WHEN a notification has `urgency = 'PROMOTIONAL'` and the user has opted out of the notification type, THE Notification_Processor SHALL skip delivery entirely and record the notification with status `SKIPPED_OPT_OUT`
6. THE Transaction_Service SHALL provide a `PUT /api/notifications/dnd` endpoint for users to configure do-not-disturb windows. This endpoint SHALL delegate to Platform Service via internal API `PUT /internal/users/{userId}/dnd` because DND configuration is owned by Platform Service (stored in the `platform.court_owner_notif_prefs` table alongside notification preferences). This ensures the Notification Processor can fetch DND state from a single internal API call (Req 20.1).
   - **Input:** `{ enabled: boolean, startTime: HH:mm, endTime: HH:mm, timezone: string }`
   - **Output (success):** `200 OK` with the saved DND configuration
   - DND windows that span midnight (e.g., 22:00–07:00) SHALL be supported
7. THE Transaction_Service SHALL run a Quartz job (`DndQueueFlushJob`) every 15 minutes that checks for `DND_QUEUED` notifications whose `deliver_after` timestamp has passed and delivers them through the normal channel routing. THE job SHALL process notifications in batches of 100, ordered by `created_at ASC` (FIFO), and continue processing remaining notifications if individual deliveries fail

### Requirement 11: Booking Notification Lifecycle

**User Story:** As a customer or court owner, I want to receive appropriate notifications at each stage of the booking lifecycle, so that I stay informed about booking status changes, upcoming bookings, and required actions.

#### Acceptance Criteria

1. WHEN a booking enters `PENDING_CONFIRMATION` status, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the court owner with `notificationType = 'BOOKING_PENDING_CONFIRMATION'`, `urgency = 'STANDARD'`, and `channels = ['PUSH', 'EMAIL', 'IN_APP', 'WEB_PUSH']`
2. WHEN a pending booking is not confirmed within the configured reminder intervals (1h, 4h, 12h after creation), THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events with `notificationType = 'PENDING_CONFIRMATION_REMINDER'` and `urgency = 'STANDARD'` to remind the court owner
3. WHEN a booking is confirmed by the court owner, THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events to notify the customer with `notificationType = 'BOOKING_CONFIRMED'` and `urgency = 'CRITICAL'`
4. WHEN a booking is cancelled by any party (customer, court owner, or system timeout), THE Transaction_Service SHALL publish `NOTIFICATION_REQUESTED` events to notify all affected parties with `notificationType = 'BOOKING_CANCELLED'` and `urgency = 'CRITICAL'`
5. WHEN a booking is rejected by the court owner, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer with `notificationType = 'BOOKING_REJECTED'` and `urgency = 'CRITICAL'`
6. WHEN a refund is processed, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer with `notificationType = 'REFUND_COMPLETED'` and `urgency = 'STANDARD'`
7. WHEN a booking reminder is due (based on court owner's configured `BOOKING_REMINDER` reminder rules), THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer with `notificationType = 'BOOKING_REMINDER'` and `urgency = 'STANDARD'`, delivered through both push (if app closed) and WebSocket (if app active)
8. WHEN a payment fails during booking creation, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event to notify the customer with `notificationType = 'PAYMENT_FAILED'` and `urgency = 'CRITICAL'`
9. FOR ALL notification events published by the Transaction_Service, THE Transaction_Service SHALL include localized `title` and `body` in both Greek (`el`) and English (`en`), and the `data` object SHALL include `bookingId`, `courtId`, and `deepLink` for client-side navigation


### Requirement 12: Multi-Channel Delivery Coordination

**User Story:** As a platform operator, I want notifications to be delivered through the optimal combination of channels based on user state and preferences, so that users receive notifications reliably without redundant alerts.

#### Acceptance Criteria

1. WHEN a notification targets a user who has the app active (WebSocket connected), THE Notification_Processor SHALL deliver via WebSocket as the primary channel and skip FCM push delivery to avoid duplicate alerts (push + in-app)
2. WHEN a notification targets a user who does not have an active WebSocket connection, THE Notification_Processor SHALL deliver via FCM push to all registered device tokens
3. WHEN WebSocket delivery is attempted but fails (connection dropped between state check and send), THE Notification_Processor SHALL fall back to FCM push delivery within the same processing cycle
4. WHEN a notification targets a court owner, THE Notification_Processor SHALL deliver via all registered channels: browser push (if admin portal is closed), WebSocket (if admin portal is open), email (always for booking events), and FCM mobile push (if the court owner has the mobile app installed)
5. THE Notification_Processor SHALL track delivery status per channel in the `transaction.notifications` table: `{ pushStatus: enum, webSocketStatus: enum, emailStatus: enum, webPushStatus: enum }` where each status is one of `PENDING`, `DELIVERED`, `FAILED`, `SKIPPED`
6. THE Notification_Processor SHALL publish delivery metrics: `notification_delivery_total` counter (labeled by channel and status), `notification_delivery_latency_seconds` histogram (labeled by channel), and `notification_delivery_failures_total` counter (labeled by channel and error type)

### Requirement 13: WebSocket Connection Lifecycle and Graceful Shutdown

**User Story:** As a user, I want my real-time connection to recover automatically from network interruptions and server deployments, so that I don't miss notifications during brief outages.

#### Acceptance Criteria

1. WHEN a Transaction Service pod begins a rolling deployment shutdown, THE Transaction_Service SHALL send a STOMP MESSAGE to all connected clients on `/user/queue/system` with `{ type: "RECONNECT", message: "Server restarting, please reconnect", retryAfterMs: 5000 }` and then wait up to 30 seconds for connections to drain before forcefully closing remaining connections
2. WHEN a WebSocket connection is closed by the server (deployment, error, or heartbeat timeout), THE client SHALL reconnect using exponential backoff: 1s, 2s, 4s, 8s, 16s, max 30s. The server SHALL accept reconnections and restore subscriptions based on the JWT identity
3. WHEN a client reconnects after a disconnection, THE Transaction_Service SHALL deliver any notifications that were persisted in the `transaction.notifications` table during the disconnection period (notifications with `webSocketStatus = 'PENDING'` for that user)
4. THE Transaction_Service SHALL register a Spring `@PreDestroy` lifecycle hook that initiates the graceful WebSocket shutdown sequence (reconnect signal → 30-second drain → force close)
5. THE Transaction_Service SHALL log WebSocket connection events (connect, disconnect, subscribe, unsubscribe) at INFO level with user ID, session ID, and remote address for debugging and audit purposes

### Requirement 14: Notification Persistence and History

**User Story:** As a user, I want to view my past notifications in an in-app notification center, so that I can review booking updates and important messages I may have missed.

#### Acceptance Criteria

1. THE Transaction_Service SHALL persist every processed notification in the `transaction.notifications` table with fields: `id`, `user_id`, `notification_type`, `urgency`, `title`, `body`, `data` (JSONB), `read` (boolean, default false), `delivery_status` (JSONB with per-channel status), `created_at`, `read_at`, `deliver_after` (nullable, for DND-queued notifications)
2. THE Transaction_Service SHALL retain notifications in the database for 90 days. A daily Quartz job (`NotificationCleanupJob`) SHALL delete notifications older than 90 days
3. WHEN a notification is delivered via any channel, THE Transaction_Service SHALL update the per-channel delivery status in the `delivery_status` JSONB column
4. THE Transaction_Service SHALL index the `notifications` table on `(user_id, created_at DESC)` for efficient paginated retrieval and on `(user_id, read)` for unread count queries
5. THE Transaction_Service SHALL support a maximum of 1000 notifications per user in the notification center. WHEN the limit is reached, THE Transaction_Service SHALL delete the oldest read notifications to make room for new ones. IF all 1000 notifications are unread, THE Transaction_Service SHALL delete the oldest unread notifications to make room. The 1000-per-user limit takes precedence over the 90-day retention policy — a notification may be deleted before 90 days if the user's notification count exceeds 1000

### Requirement 15: Booking Reminder Scheduling

**User Story:** As a customer, I want to receive reminders before my upcoming bookings, so that I don't forget about my reservations.

#### Acceptance Criteria

1. WHEN a booking is confirmed (status transitions to `CONFIRMED`), THE Transaction_Service SHALL schedule a Quartz trigger for the booking reminder based on the court owner's configured `BOOKING_REMINDER` reminder rules (fetched from Platform Service via internal API `GET /internal/courts/{courtId}/reminder-rules`)
2. WHEN a booking reminder trigger fires, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event with `notificationType = 'BOOKING_REMINDER'`, `urgency = 'STANDARD'`, and `channels = ['PUSH', 'IN_APP']`
3. WHEN a booking is cancelled before the reminder fires, THE Transaction_Service SHALL unschedule the associated Quartz trigger to prevent sending a reminder for a cancelled booking
4. WHEN no `BOOKING_REMINDER` rule is configured for a court, THE Transaction_Service SHALL use a default reminder of 2 hours before the booking start time
5. THE Transaction_Service SHALL use the court's timezone (from `v_court_summary.timezone`) for all reminder time calculations
6. WHEN a booking reminder is delivered, THE Transaction_Service SHALL include the booking details (court name, date, time, location) in the notification body and a deep link to the booking detail screen

### Requirement 16: Pending Confirmation Reminder Delivery

**User Story:** As a court owner, I want to receive escalating reminders for unconfirmed bookings, so that I don't miss booking requests and cause automatic cancellations.

#### Acceptance Criteria

1. THE Transaction_Service SHALL run the `PendingConfirmationReminderJob` Quartz job hourly to check for bookings in `PENDING_CONFIRMATION` status that have passed the configured reminder thresholds (1h, 4h, 12h after creation)
2. WHEN a pending booking passes a reminder threshold, THE Transaction_Service SHALL publish a `NOTIFICATION_REQUESTED` event with `notificationType = 'PENDING_CONFIRMATION_REMINDER'`, `urgency = 'STANDARD'`, and `channels = ['PUSH', 'EMAIL', 'IN_APP', 'WEB_PUSH']` targeting the court owner
3. THE Transaction_Service SHALL track which reminder thresholds have been sent for each pending booking (using a `reminder_state` JSONB column or separate tracking table) to prevent duplicate reminders
4. WHEN a pending booking is confirmed or rejected, THE Transaction_Service SHALL stop sending further reminders for that booking
5. THE pending confirmation reminder notification SHALL include the booking details, the time remaining before auto-cancellation, and a deep link to the booking confirmation screen in the admin portal


### Requirement 17: Notification Delivery Reliability and Observability

**User Story:** As a platform operator, I want reliable notification delivery with comprehensive monitoring, so that I can detect and resolve delivery issues before they impact users.

#### Acceptance Criteria

1. THE Notification_Processor SHALL implement at-least-once delivery semantics: notifications are persisted before delivery attempts, and delivery status is tracked per channel. Failed deliveries are retried according to channel-specific retry policies
2. WHEN a notification delivery fails on all channels after exhausting retries, THE Notification_Processor SHALL mark the notification as `FAILED` in the notifications table and publish a `notification_delivery_failure` metric with labels for `notificationType`, `urgency`, and `failureReason`
3. THE Transaction_Service SHALL expose the following Prometheus metrics for notification monitoring:
   - `notification_events_consumed_total` — counter of events consumed from Kafka (labeled by `notificationType`)
   - `notification_delivery_total` — counter of delivery attempts (labeled by `channel`, `status`)
   - `notification_delivery_latency_seconds` — histogram of delivery latency (labeled by `channel`)
   - `notification_delivery_failures_total` — counter of failed deliveries (labeled by `channel`, `errorType`)
   - `notification_dnd_queued_total` — counter of notifications queued due to DND
   - `websocket_connections_active` — gauge of active WebSocket connections
   - `websocket_messages_delivered_total` — counter of WebSocket messages sent
   - `redis_pubsub_messages_total` — counter of Redis Pub/Sub messages published and received
4. THE Transaction_Service SHALL log notification delivery events at INFO level with structured fields: `notificationId`, `userId`, `notificationType`, `channel`, `status`, `latencyMs`
5. WHEN the Kafka consumer lag for the `notification-events` topic exceeds 1000 messages, THE Transaction_Service SHALL log a WARN-level alert and increment a `notification_consumer_lag_high` metric

### Requirement 18: Device Token and Subscription Cleanup

**User Story:** As a platform operator, I want expired and invalid device tokens and browser push subscriptions to be cleaned up automatically, so that the system does not waste resources on undeliverable notifications.

#### Acceptance Criteria

1. THE Transaction_Service SHALL run a daily Quartz job (`DeviceTokenCleanupJob`) that removes device tokens and browser push subscriptions that have not been used for successful delivery in 90 days
2. WHEN FCM reports a token as invalid during push delivery (error code `messaging/registration-token-not-registered` or `messaging/invalid-registration-token`), THE Transaction_Service SHALL remove the token immediately within the same processing cycle
3. WHEN a Web Push delivery returns `410 Gone` or `404 Not Found`, THE Transaction_Service SHALL remove the browser push subscription immediately within the same processing cycle
4. THE `DeviceTokenCleanupJob` SHALL log the number of expired tokens removed per run and publish a `device_tokens_cleaned_total` metric
5. WHEN a user's last device token is removed (no remaining tokens), THE Transaction_Service SHALL log a WARN-level message indicating the user has no registered devices for push delivery

### Requirement 19: WebSocket Performance and Scalability

**User Story:** As a platform operator, I want the WebSocket infrastructure to handle the expected concurrent connection load with low latency, so that real-time features remain responsive as the platform grows.

#### Acceptance Criteria

1. THE WebSocket infrastructure SHALL support up to 10,000 concurrent connections per Transaction Service pod, with connection acceptance rate of at least 100 new connections per second
2. WHEN a booking is created or cancelled, THE availability update SHALL propagate to connected WebSocket clients within 2 seconds end-to-end, measured from the Kafka event production timestamp to the WebSocket frame delivery timestamp
3. THE Transaction_Service SHALL configure the WebSocket server with: maximum frame size of 64KB, maximum message buffer size of 512KB, send timeout of 10 seconds, and transport-level message size limit of 128KB
4. THE Transaction_Service SHALL use a dedicated thread pool for WebSocket message broadcasting (separate from the HTTP request thread pool) to prevent WebSocket traffic from impacting REST API performance
5. WHEN a WebSocket client sends messages faster than the server can process (slow consumer), THE Transaction_Service SHALL buffer up to 100 messages per session and drop older messages if the buffer is full, logging a WARN-level message for the dropped messages
6. THE Transaction_Service SHALL monitor and report `websocket_connection_duration_seconds` histogram to track connection lifetimes and detect abnormal disconnection patterns

### Requirement 20: Internal API Contracts for Cross-Service Communication

**User Story:** As a platform operator, I want well-defined internal API contracts between services, so that the Notification Processor can reliably fetch user preferences, reminder rules, and user language settings.

#### Acceptance Criteria

1. THE Platform_Service SHALL expose `GET /internal/users/{userId}/notification-preferences` returning the user's notification preferences and do-not-disturb configuration:
   - **Output:** `200 OK` with:
     ```json
     {
       "preferences": [
         { "eventType": "BOOKING_CONFIRMED", "pushEnabled": true, "emailEnabled": true, "inAppEnabled": true },
         { "eventType": "BOOKING_CANCELLED", "pushEnabled": true, "emailEnabled": true, "inAppEnabled": true }
       ],
       "doNotDisturb": {
         "enabled": boolean,
         "startTime": "HH:mm",
         "endTime": "HH:mm",
         "timezone": "Europe/Athens"
       }
     }
     ```
   - **Output (user not found):** `404 Not Found`
2. THE Platform_Service SHALL expose `GET /internal/courts/{courtId}/reminder-rules` returning the court's configured reminder rules:
   - **Output:** `200 OK` with:
     ```json
     {
       "rules": [
         { "ruleType": "BOOKING_REMINDER", "enabled": true, "hoursBefore": 2 },
         { "ruleType": "PENDING_CONFIRMATION", "enabled": true, "hoursBefore": 12 }
       ]
     }
     ```
   - **Output (court not found):** `404 Not Found`
   - **NOTE:** The `hoursBefore` field matches the `ReminderRule.triggerHoursBefore` domain model field. For `PENDING_CONFIRMATION` rules, the `hoursBefore` value indicates when the first reminder is sent; subsequent escalation reminders (1h, 4h, 12h before timeout) are system-wide and not configurable per court.
3. THE Transaction_Service SHALL access user language preference via the existing cross-schema view `v_user_basic.language` (already available from Phase 1b migrations). THE Notification_Processor SHALL use this language field to select localized notification content
4. THE internal API endpoints SHALL be secured with service-to-service authentication using the `X-Internal-Api-Key` header (consistent with existing internal endpoints from Phase 3), validated against the `INTERNAL_API_KEY` environment variable. In staging/production with Istio, mTLS provides authentication and no application-level header is needed
5. THE Platform_Service SHALL respond to internal API requests within 100ms p99 latency. THE Notification_Processor SHALL cache notification preferences (including DND configuration) in Redis with a 5-minute TTL to reduce cross-service calls. NOTE: This means preference changes may take up to 5 minutes to take effect on notification delivery — this is an acceptable trade-off for reducing cross-service call volume
6. WHEN the Platform Service internal API is unreachable (network error, timeout, or 5xx response), THE Notification_Processor SHALL fall back to the Redis-cached preferences if available. IF no cached preferences exist for the user, THE Notification_Processor SHALL deliver the notification through all channels specified in the event's `channels` array (default-open behavior) to avoid silently dropping notifications. THE Notification_Processor SHALL log the internal API failure at WARN level and increment a `notification_preference_fetch_failures_total` metric
7. THE Platform_Service SHALL also expose `PUT /internal/users/{userId}/dnd` for Transaction Service to delegate DND configuration updates. This endpoint SHALL update the `dnd_enabled`, `dnd_start_time`, `dnd_end_time`, and `dnd_timezone` columns in the `platform.court_owner_notif_prefs` table and invalidate the Redis preference cache for the user

### Requirement 21: Notification Localization and Template Management

**User Story:** As a platform operator, I want notification content to be properly localized and template management to be clearly defined, so that users receive notifications in their preferred language.

#### Acceptance Criteria

1. THE notification event publisher (Transaction Service or Platform Service) SHALL include localized `title` and `body` in the Kafka event payload as objects with language keys:
   ```json
   {
     "title": { "el": "Επιβεβαίωση Κράτησης", "en": "Booking Confirmed" },
     "body": { "el": "Η κράτησή σας επιβεβαιώθηκε", "en": "Your booking has been confirmed" }
   }
   ```
   THE Notification_Processor SHALL select the appropriate language based on `v_user_basic.language` (defaulting to `el` if not set)
2. THE Transaction_Service SHALL manage SendGrid email template IDs via environment variables with the following naming convention:
   - `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EL` / `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EN`
   - `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EL` / `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EN`
   - `SENDGRID_TEMPLATE_BOOKING_REJECTED_EL` / `SENDGRID_TEMPLATE_BOOKING_REJECTED_EN`
   - `SENDGRID_TEMPLATE_BOOKING_REMINDER_EL` / `SENDGRID_TEMPLATE_BOOKING_REMINDER_EN`
   - `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EL` / `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EN`
   - `SENDGRID_TEMPLATE_REFUND_COMPLETED_EL` / `SENDGRID_TEMPLATE_REFUND_COMPLETED_EN`
   - `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EL` / `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EN`
   - `SENDGRID_TEMPLATE_PENDING_REMINDER_EL` / `SENDGRID_TEMPLATE_PENDING_REMINDER_EN`
3. THE SendGrid templates SHALL be managed in the SendGrid dashboard (not in code). THE Transaction_Service SHALL pass template variables via the SendGrid API's `dynamic_template_data` field
4. THE template variable schema for booking-related emails SHALL include: `customerName`, `courtName`, `bookingDate`, `bookingTime`, `totalAmount`, `currency`, `bookingId`, `deepLink`, `unsubscribeLink`

### Requirement 22: Notification Deduplication and Idempotency

**User Story:** As a platform operator, I want notification delivery to be idempotent, so that duplicate Kafka events do not result in duplicate notifications to users.

#### Acceptance Criteria

1. THE Notification_Processor SHALL store processed `eventId` values in Redis with a 7-day TTL for deduplication, using the key pattern `notification:processed:{eventId}`
2. WHEN a notification event is consumed, THE Notification_Processor SHALL check Redis for the `eventId` before processing. IF the `eventId` exists in Redis, THE Notification_Processor SHALL log the duplicate at DEBUG level and skip processing
3. THE Notification_Processor SHALL set the Redis deduplication key AFTER successfully persisting the notification to the database (but before delivery attempts), ensuring at-least-once delivery semantics
4. THE Transaction_Service SHALL publish a `notification_duplicates_skipped_total` counter metric for monitoring duplicate event frequency

### Requirement 23: DND Queue Flush Job Details

**User Story:** As a platform operator, I want the DND queue flush job to process queued notifications efficiently and reliably.

#### Acceptance Criteria

1. THE `DndQueueFlushJob` SHALL process DND-queued notifications in batches of 100 notifications per execution cycle, ordered by `created_at ASC` (FIFO)
2. WHEN processing a batch of DND-queued notifications, THE `DndQueueFlushJob` SHALL update each notification's status from `DND_QUEUED` to `PENDING` and route it through the normal channel delivery logic
3. WHEN a notification delivery fails during DND queue flush, THE `DndQueueFlushJob` SHALL mark the notification as `FAILED` and continue processing remaining notifications in the batch (no batch rollback)
4. THE `DndQueueFlushJob` SHALL publish the following metrics:
   - `dnd_queue_flush_processed_total` — counter of notifications processed per run
   - `dnd_queue_flush_failed_total` — counter of notifications that failed during flush
   - `dnd_queue_depth` — gauge of remaining DND-queued notifications after each run

### Requirement 24: Notification Center API Pagination and Limits

**User Story:** As a user, I want the notification center API to have sensible defaults and limits for pagination.

#### Acceptance Criteria

1. THE `GET /api/notifications` endpoint SHALL enforce the following pagination constraints:
   - Default page size: 20
   - Maximum page size: 100
   - Page numbers are 0-indexed
   - Sort order: `createdAt DESC` (newest first)
2. WHEN a client requests a page size greater than 100, THE Transaction_Service SHALL cap the page size at 100 and return results accordingly (no error)
3. THE `GET /api/notifications` endpoint SHALL support an optional `type` query parameter for filtering by notification type (e.g., `?type=BOOKING_CONFIRMED`)

### Requirement 25: FCM Batch Sending for Multi-Device Users

**User Story:** As a platform operator, I want FCM push notifications to be sent efficiently when a user has multiple devices registered.

#### Acceptance Criteria

1. WHEN delivering a push notification to a user with multiple registered FCM device tokens, THE Transaction_Service SHALL use FCM's `sendEachForMulticast` API (Firebase Admin SDK) to send to all tokens in a single API call
2. THE Transaction_Service SHALL process the `BatchResponse` from FCM to identify which tokens succeeded and which failed, updating delivery status per token
3. WHEN FCM returns partial success (some tokens delivered, some failed), THE Transaction_Service SHALL mark the notification as `PARTIALLY_DELIVERED` and log the failed tokens with their error codes

### Requirement 26: Notification Retry State and Persistence

**User Story:** As a platform operator, I want notification retry behavior to be clearly defined, so that I understand how retries work and what happens on pod restarts.

#### Acceptance Criteria

1. THE Notification_Processor SHALL perform retries in-memory within the same Kafka message processing cycle. Retries do NOT survive pod restarts
2. WHEN all retry attempts are exhausted for a channel, THE Notification_Processor SHALL update the notification's per-channel status to `FAILED` in the database and move to the next channel
3. THE maximum retry duration per notification (across all channels) SHALL NOT exceed 5 minutes. IF retries are still in progress after 5 minutes, THE Notification_Processor SHALL mark remaining channels as `FAILED` and commit the Kafka offset
4. THE Transaction_Service SHALL NOT implement a separate retry queue or dead-letter queue for failed notifications in Phase 5. Failed notifications are logged and recorded in the database for manual investigation

### Requirement 27: WebSocket Authentication Error Handling

**User Story:** As a client developer, I want clear error handling for WebSocket authentication failures, so that I can implement proper retry and token refresh logic.

#### Acceptance Criteria

1. WHEN WebSocket authentication fails due to an expired JWT token, THE Transaction_Service SHALL send a STOMP ERROR frame with:
   ```json
   { "message": "Token expired", "code": "TOKEN_EXPIRED" }
   ```
   and close the connection with WebSocket close code `4001` (per `websocket-message-contracts.json`: "Authentication expired — token refresh failed or timed out")
2. WHEN WebSocket authentication fails due to an invalid JWT token (malformed, wrong signature, or insufficient permissions), THE Transaction_Service SHALL send a STOMP ERROR frame with:
   ```json
   { "message": "Authentication failed", "code": "INVALID_TOKEN" }
   ```
   and close the connection with WebSocket close code `4002` (per `websocket-message-contracts.json`: "Authorization denied — insufficient permissions for requested channel")
3. WHEN a client receives a `TOKEN_EXPIRED` error, THE client SHOULD refresh the JWT token and reconnect. THE client SHALL NOT retry with the same expired token
4. THE Transaction_Service SHALL NOT support token refresh during an active WebSocket session. Clients MUST disconnect and reconnect with a new token when the current token expires
5. THE Transaction_Service SHALL log authentication failures at WARN level with the remote address and failure reason (but NOT the token itself)

### Requirement 28: Platform Service to Transaction Service WebSocket Flow

**User Story:** As a platform operator, I want the real-time availability update flow to be clearly documented, so that I understand how booking events flow from Platform Service to WebSocket clients.

#### Acceptance Criteria

1. THE real-time availability update flow SHALL follow this sequence:
   - Transaction Service publishes booking event (BOOKING_CREATED, BOOKING_CONFIRMED, BOOKING_CANCELLED, BOOKING_MODIFIED, SLOT_HELD, SLOT_RELEASED) to `booking-events` Kafka topic
   - Platform Service `BookingEventKafkaConsumer` consumes the event
   - Platform Service invalidates the availability cache in Redis
   - Platform Service publishes availability update to Redis Pub/Sub channel `ws:court:{courtId}:availability`
   - Transaction Service (all pods) receives the Redis Pub/Sub message
   - Transaction Service broadcasts to WebSocket clients subscribed to `/topic/courts/{courtId}/availability`
2. THE Platform_Service SHALL use the same Redis instance as Transaction Service for Pub/Sub messaging (configured via `REDIS_HOST` and `REDIS_PORT` environment variables)
3. THE Platform_Service SHALL NOT maintain WebSocket connections directly. All WebSocket connections are managed by Transaction Service

### Requirement 29: Quartz Job Implementation Clarification

**User Story:** As a developer, I want clarity on which Quartz jobs are being implemented vs. extended in Phase 5.

#### Acceptance Criteria

1. THE following Quartz jobs SHALL be IMPLEMENTED (new code) in Phase 5:
   - `DndQueueFlushJob` — processes DND-queued notifications every 15 minutes
   - `NotificationCleanupJob` — deletes notifications older than 90 days, runs daily
   - `DeviceTokenCleanupJob` — removes expired device tokens, runs daily
   - `BookingReminderJob` — fires scheduled booking reminders (individual triggers per booking)
2. THE following Quartz jobs already exist as skeletons from Phase 4 and SHALL be EXTENDED (implementation completed) in Phase 5:
   - `PendingConfirmationReminderJob` — already exists, Phase 5 implements the notification publishing logic
   - `BookingCompletionJob` — already exists, Phase 5 implements the notification publishing logic for completed bookings
3. THE Quartz job implementations SHALL use the existing `QuartzConfig` from Phase 4 and register jobs via `@PostConstruct` initialization

### Requirement 30: WebSocket Notification Payload Size Limits

**User Story:** As a platform operator, I want WebSocket notification payloads to have defined size limits to prevent memory issues.

#### Acceptance Criteria

1. THE maximum notification payload size for WebSocket delivery SHALL be 16KB (after JSON serialization). Notifications exceeding this size SHALL be truncated (body field) with an ellipsis indicator
2. THE `data` field in WebSocket notifications SHALL be limited to 4KB. IF the original notification event's `data` field exceeds 4KB, THE Notification_Processor SHALL include only essential fields (`bookingId`, `courtId`, `deepLink`) and omit additional metadata
3. THE Transaction_Service SHALL log a WARN-level message when a notification payload is truncated, including the original size and notification ID


### Requirement 31: Platform Service Internal API for DND Configuration

**User Story:** As a platform operator, I want DND configuration to be stored alongside notification preferences in Platform Service, so that the Notification Processor can fetch all preference data from a single internal API call.

#### Acceptance Criteria

1. THE Platform_Service SHALL store do-not-disturb configuration in the `platform.court_owner_notif_prefs` table with columns: `dnd_enabled` (boolean, default false), `dnd_start_time` (TIME), `dnd_end_time` (TIME), `dnd_timezone` (VARCHAR, default 'Europe/Athens'). A Flyway migration SHALL add these columns
2. THE Platform_Service SHALL expose `PUT /internal/users/{userId}/dnd` accepting `{ enabled: boolean, startTime: HH:mm, endTime: HH:mm, timezone: string }` and returning `200 OK` with the saved configuration. This endpoint is called by Transaction Service's `PUT /api/notifications/dnd` endpoint
3. THE `GET /internal/users/{userId}/notification-preferences` response (Req 20.1) SHALL include the `doNotDisturb` object populated from these columns. IF no DND configuration exists for the user, the response SHALL return `doNotDisturb: { enabled: false, startTime: null, endTime: null, timezone: "Europe/Athens" }`
4. WHEN DND configuration is updated via the internal API, THE Platform_Service SHALL invalidate the Redis preference cache for that user (key pattern `notification:prefs:{userId}`) to ensure the Notification Processor picks up the change promptly
