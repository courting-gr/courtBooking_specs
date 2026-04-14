# Implementation Plan: Phase 5 — Real-time & Notifications

## Overview

This plan implements the complete real-time communication and notification delivery subsystem across Transaction Service (primary) and Platform Service (availability broadcasting). Tasks are ordered by dependency: domain layer → database migrations → outgoing ports → adapters → application services → incoming adapters (consumers, controllers, WebSocket, jobs) → Platform Service changes → wiring and integration.

The design uses Java throughout. jqwik 1.9.2 is already configured for property-based testing. Spring WebSocket starter and Spring Data Redis are already present in both services.

## Tasks

- [ ] 1. Domain layer — Notification and DeviceToken entities, enums, value objects (Transaction Service)
  - [ ] 1.1 Create `NotificationType` enum with all 28 notification types
    - File: `domain/model/NotificationType.java`
    - _Requirements: 1.1, 11_
  - [ ] 1.2 Create `UrgencyLevel` enum (CRITICAL, STANDARD, PROMOTIONAL)
    - File: `domain/model/UrgencyLevel.java`
    - _Requirements: 1.4, 1.5, 1.6_
  - [ ] 1.3 Create `TokenType` enum (FCM, WEB_PUSH) and `ChannelStatus` enum (PENDING, DELIVERED, FAILED, SKIPPED)
    - Files: `domain/model/TokenType.java`, `domain/model/ChannelStatus.java`
    - _Requirements: 2.5, 9.3, 12.5_
  - [ ] 1.4 Create `DeliveryStatus` value object with per-channel status tracking and overall status derivation
    - File: `domain/model/DeliveryStatus.java`
    - Methods: `markDelivered(channel)`, `markFailed(channel)`, `markSkipped(channel)`, `getOverallStatus()`
    - Overall status logic: DELIVERED if all delivered, PARTIALLY_DELIVERED if mixed, FAILED if all failed, SKIPPED if all skipped, PENDING otherwise
    - _Requirements: 12.5, 14.3_
  - [ ] 1.5 Create `Notification` rich domain entity
    - File: `domain/model/Notification.java`
    - Fields: id, userId, notificationType, urgency, title, body, data (Map), read, readAt, deliveryStatus, deliverAfter, createdAt
    - Business methods: `markRead()`, `markDelivered(channel)`, `markFailed(channel)`, `markSkipped(channel)`, `queueForDnd(deliverAfter)`, `isDndQueued()`, `getOverallStatus()`
    - _Requirements: 1.3, 14.1_
  - [ ] 1.6 Create `DeviceToken` rich domain entity
    - File: `domain/model/DeviceToken.java`
    - Fields: id, userId, encryptedToken, tokenType, platform, deviceId, registeredAt, lastUsedAt
    - Business methods: `updateLastUsed()`, `isExpired(Duration maxAge)`
    - _Requirements: 2.1, 2.5, 2.7_
  - [ ] 1.7 Create `NotificationEvent` internal record/DTO for Kafka consumer deserialization
    - File: `domain/model/NotificationEvent.java`
    - Record fields: eventId, userId, notificationType, urgency, titleEl, titleEn, bodyEl, bodyEn, channels (List<String>), data (Map<String, Object>), timestamp
    - This is the normalized internal representation that the `NotificationEventKafkaConsumer` maps raw JSON into before passing to `NotificationProcessingService.processNotificationEvent(NotificationEvent)`
    - Handles both Transaction Service format (bilingual title/body objects, channels as array) and Platform Service format (single language, channels as Map<String, Boolean>)
    - _Requirements: 1.1, 1.2_

- [ ] 2. Database migrations (Transaction Service V3–V5, Platform Service V7)
  - [ ] 2.1 Create V3 migration — Restructure notifications table for Phase 5
    - File: `src/main/resources/db/migration/transaction/V3__restructure_notifications_for_phase5.sql`
    - Drop old columns (channel, language, failure_reason, status), add new columns (urgency, read, delivery_status JSONB, deliver_after), add indexes (user_created, user_read, dnd_queued), add CHECK constraint for urgency
    - _Requirements: 14.1, 14.4_
  - [ ] 2.2 Create V4 migration — Restructure device_tokens table for Phase 5
    - File: `src/main/resources/db/migration/transaction/V4__restructure_device_tokens_for_phase5.sql`
    - Drop old columns (token, push_subscription_endpoint, push_subscription_keys, active, updated_at), add new columns (encrypted_token, token_type, last_used_at), rename created_at to registered_at, add indexes and CHECK constraints
    - _Requirements: 2.5, 9.3_
  - [ ] 2.3 Create V5 migration — Add reminder_state JSONB column to bookings table
    - File: `src/main/resources/db/migration/transaction/V5__add_reminder_state_column.sql`
    - Add `reminder_state JSONB DEFAULT '{}'` to `transaction.bookings`
    - _Requirements: 16.3_
  - [ ] 2.4 Create V7 migration in Platform Service — Add DND columns to court_owner_notif_prefs
    - File: `platform-service/src/main/resources/db/migration/platform/V7__add_dnd_columns.sql`
    - Add columns: dnd_enabled (BOOLEAN DEFAULT false), dnd_start_time (TIME), dnd_end_time (TIME), dnd_timezone (VARCHAR(50) DEFAULT 'Europe/Athens')
    - _Requirements: 31.1_

- [ ] 3. Outgoing port interfaces (Transaction Service)
  - [ ] 3.1 Create `FcmPushPort` interface with `FcmMessage`, `FcmDeliveryResult`, `TokenResult` records
    - File: `application/port/out/FcmPushPort.java`
    - _Requirements: 3.1, 3.2_
  - [ ] 3.2 Create `SendGridEmailPort` interface with `SendGridEmailRequest` record
    - File: `application/port/out/SendGridEmailPort.java`
    - _Requirements: 8.1_
  - [ ] 3.3 Create `WebPushPort` interface with `WebPushSubscription`, `WebPushPayload`, `WebPushDeliveryResult` records
    - File: `application/port/out/WebPushPort.java`
    - _Requirements: 9.4_
  - [ ] 3.4 Create `WebSocketSessionPort` interface (isUserConnected, getSessionIds, sendToUser)
    - File: `application/port/out/WebSocketSessionPort.java`
    - _Requirements: 4.10, 7.1_
  - [ ] 3.5 Create `RedisPubSubPublisherPort` interface (publishNotification, publishAvailabilityUpdate, publishBookingUpdate)
    - File: `application/port/out/RedisPubSubPublisherPort.java`
    - _Requirements: 5.1, 5.2_
  - [ ] 3.6 Create `SaveNotificationPort` and `LoadNotificationPort` interfaces
    - File: `application/port/out/SaveNotificationPort.java`, `application/port/out/LoadNotificationPort.java`
    - LoadNotificationPort methods: loadById, loadByUserId (paginated), countUnread, loadDndQueuedReady, markAllRead, deleteOlderThan, countByUserId, deleteOldestRead, **loadPendingWebSocket(userId)** — returns notifications with `webSocketStatus = PENDING` for reconnection delivery
    - _Requirements: 1.3, 7.5, 7.6, 7.7, 7.8, 13.3, 14.1, 14.5_
  - [ ] 3.7 Create `SaveDeviceTokenPort` and `LoadDeviceTokenPort` interfaces
    - File: `application/port/out/SaveDeviceTokenPort.java`, `application/port/out/LoadDeviceTokenPort.java`
    - LoadDeviceTokenPort methods: loadByUserId, loadFcmTokensByUserId, loadWebPushByUserId, loadByToken, loadById, countByUserId, deleteExpiredTokens, deleteById, deleteByToken
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  - [ ] 3.8 Create `NotificationDeduplicationPort` interface (isProcessed, markProcessed)
    - File: `application/port/out/NotificationDeduplicationPort.java`
    - _Requirements: 1.11, 22.1_
  - [ ] 3.9 Create `NotificationPreferenceCachePort` interface with `UserNotificationPreferences` record (including nested `ChannelPreference` and `DndConfiguration`)
    - File: `application/port/out/NotificationPreferenceCachePort.java`
    - The `UserNotificationPreferences.DndConfiguration` record must include an `isInDndWindow(Instant now)` method that handles same-day windows (e.g., 09:00–17:00) and overnight windows (e.g., 22:00–07:00) using the configured timezone. This is the Transaction Service's own DTO — it does NOT depend on Platform Service's `DndConfiguration` domain model
    - _Requirements: 20.5_
  - [ ] 3.10 Extend existing `PlatformServicePort` with Phase 5 methods: `getNotificationPreferences`, `getReminderRules`, `updateDnd`, and `ReminderRuleInfo` record
    - File: modify existing `application/port/out/PlatformServicePort.java`
    - _Requirements: 20.1, 20.2, 20.7_

- [ ] 4. Outgoing port interface — Platform Service (new)
  - [ ] 4.1 Create `AvailabilityPubSubPort` interface with `AvailabilityUpdateMessage` record
    - File: `platform-service/.../application/port/out/AvailabilityPubSubPort.java`
    - _Requirements: 6.1, 28.1_

- [ ] 5. Incoming port interfaces (Use Cases) — Transaction Service
  - [ ] 5.1 Create `ProcessNotificationUseCase` interface
    - File: `application/port/in/ProcessNotificationUseCase.java`
    - Method signature: `void processNotificationEvent(NotificationEvent event)` — accepts the `NotificationEvent` record defined in task 1.7
    - _Requirements: 1.1_
  - [ ] 5.2 Create device token use cases: `RegisterDeviceTokenUseCase`, `RemoveDeviceTokenUseCase`, `ListDeviceTokensQuery` with `RegisterDeviceCommand` and `DeviceTokenResult` records
    - Files: `application/port/in/RegisterDeviceTokenUseCase.java`, `application/port/in/RemoveDeviceTokenUseCase.java`, `application/port/in/ListDeviceTokensQuery.java`
    - _Requirements: 2.1, 2.3, 2.4_
  - [ ] 5.3 Create Web Push use cases: `GetVapidKeyQuery`, `SubscribeWebPushUseCase`, `UnsubscribeWebPushUseCase` with `SubscribeWebPushCommand` and `WebPushSubscriptionResult` records
    - Files: `application/port/in/GetVapidKeyQuery.java`, `application/port/in/SubscribeWebPushUseCase.java`, `application/port/in/UnsubscribeWebPushUseCase.java`
    - _Requirements: 9.1, 9.2, 9.6_
  - [ ] 5.4 Create notification center use cases: `ListNotificationsQuery`, `MarkNotificationReadUseCase`, `MarkAllNotificationsReadUseCase`, `GetUnreadCountQuery` with `NotificationFilter` and `NotificationResult` records
    - Files: `application/port/in/ListNotificationsQuery.java`, `application/port/in/MarkNotificationReadUseCase.java`, `application/port/in/MarkAllNotificationsReadUseCase.java`, `application/port/in/GetUnreadCountQuery.java`
    - _Requirements: 7.5, 7.6, 7.7, 7.8_
  - [ ] 5.5 Create `UpdateDndConfigurationUseCase` with `UpdateDndCommand` and `DndConfigurationResult` records
    - File: `application/port/in/UpdateDndConfigurationUseCase.java`
    - _Requirements: 10.6_
  - [ ] 5.6 Create `ScheduleBookingReminderUseCase` and `CancelBookingReminderUseCase` interfaces
    - Files: `application/port/in/ScheduleBookingReminderUseCase.java`, `application/port/in/CancelBookingReminderUseCase.java`
    - _Requirements: 15.1, 15.3_
  - [ ] 5.7 Create WebSocket notification and availability update DTO/record classes
    - File: `adapter/in/websocket/dto/WebSocketNotificationPayload.java` — record with fields: notificationId, type, urgency, title, body, data, createdAt, read (Req 7.2)
    - File: `adapter/in/websocket/dto/AvailabilityUpdatePayload.java` — record with fields: courtId, date, eventType, affectedSlot (startTime, endTime), updatedAt (Req 6.3)
    - File: `adapter/in/websocket/dto/BookingStatusUpdatePayload.java` — record with fields: bookingId, courtId, status, eventType, customerName, date, startTime, endTime, updatedAt
    - _Requirements: 6.3, 7.2_

- [ ] 6. Checkpoint — Ensure all domain, port, and migration code compiles
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Shared retry utility, AES-256-GCM encryption, and SendGrid template resolver (Transaction Service)
  - [ ] 7.1 Create `RetryWithBackoff` utility component
    - File: `adapter/out/RetryWithBackoff.java`
    - Generic retry helper with configurable exponential backoff: accepts initial delay, max delay, max attempts, and a retryable predicate
    - FCM retry config: initial 1s, max 60s, 3 attempts (Req 3.3)
    - SendGrid retry config: initial 2s, max 8s, 3 attempts (Req 8.4)
    - Web Push retry config: initial 2s, max 8s, 3 attempts
    - This shared utility is used by `FcmPushAdapter`, `SendGridEmailAdapter`, and `WebPushAdapter`
    - _Requirements: 3.3, 8.4_
  - [ ] 7.2 Create `DeviceTokenEncryptor` component
    - File: `adapter/out/persistence/DeviceTokenEncryptor.java`
    - AES-256-GCM with 12-byte IV prepended to ciphertext, Base64 encoded. Key from `${device-token.encryption-key}` env var (Base64-encoded 256-bit key). Fail fast on startup if key is invalid
    - _Requirements: 2.5, 9.3_
  - [ ]* 7.3 Write property test for token encryption (Property 8: Token Encryption at Rest)
    - **Property 8: Token Encryption at Rest** — For any plaintext token, encrypted ≠ plaintext, and decrypt(encrypt(token)) == token
    - **Validates: Requirements 2.5, 9.3**
  - [ ] 7.4 Create `SendGridTemplateResolver` component
    - File: `adapter/out/email/SendGridTemplateResolver.java`
    - Maps notification type → env var prefix, appends `_EL` or `_EN` suffix based on user language. Returns null for types without email templates (PAYMENT_DISPUTE, PAYOUT_COMPLETED, RECURRING_BOOKING_PRICE_CHANGED, etc.)
    - Reads 16 SendGrid template ID environment variables (8 notification types × 2 languages): `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EL`, `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EN`, `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EL`, `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EN`, `SENDGRID_TEMPLATE_BOOKING_REJECTED_EL`, `SENDGRID_TEMPLATE_BOOKING_REJECTED_EN`, `SENDGRID_TEMPLATE_BOOKING_REMINDER_EL`, `SENDGRID_TEMPLATE_BOOKING_REMINDER_EN`, `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EL`, `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EN`, `SENDGRID_TEMPLATE_REFUND_COMPLETED_EL`, `SENDGRID_TEMPLATE_REFUND_COMPLETED_EN`, `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EL`, `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EN`, `SENDGRID_TEMPLATE_PENDING_REMINDER_EL`, `SENDGRID_TEMPLATE_PENDING_REMINDER_EN`
    - _Requirements: 8.3, 21.2_
  - [ ]* 7.5 Write property test for email template resolution (Property 15: Email Template Resolution)
    - **Property 15: Email Template Resolution and Variable Completeness** — For any notification type with a defined template and any language (el/en), resolver returns non-null template ID
    - **Validates: Requirements 8.3, 21.4**

- [ ] 8. Persistence adapters — Notification and DeviceToken (Transaction Service)
  - [ ] 8.1 Create `NotificationJpaEntity` with JPA mappings for the restructured notifications table
    - File: `adapter/out/persistence/entity/NotificationJpaEntity.java`
    - Include `DeliveryStatusConverter` (JPA AttributeConverter for JSONB ↔ DeliveryStatus) and `DataMapConverter` (JSONB ↔ Map<String, Object>)
    - _Requirements: 14.1_
  - [ ] 8.2 Create `NotificationRepository` (Spring Data JPA) with custom queries for DND-queued, unread count, mark-all-read, delete-older-than, count-by-user, delete-oldest-read, **loadPendingWebSocket** (notifications with `webSocketStatus = PENDING` for reconnection delivery per Req 13.3)
    - File: `adapter/out/persistence/repository/NotificationRepository.java`
    - _Requirements: 7.5, 7.8, 10.7, 13.3, 14.2, 14.5_
  - [ ] 8.3 Create `NotificationPersistenceAdapter` implementing `SaveNotificationPort` and `LoadNotificationPort`
    - File: `adapter/out/persistence/NotificationPersistenceAdapter.java`
    - Include `NotificationPersistenceMapper` for entity ↔ domain mapping
    - Implement `loadPendingWebSocket(userId)` for reconnection delivery
    - _Requirements: 1.3, 7.5, 13.3, 14.1_
  - [ ] 8.4 Create `DeviceTokenJpaEntity` with JPA mappings for the restructured device_tokens table
    - File: `adapter/out/persistence/entity/DeviceTokenJpaEntity.java`
    - _Requirements: 2.5_
  - [ ] 8.5 Create `DeviceTokenRepository` (Spring Data JPA) with queries for user tokens by type, by encrypted token, expired token cleanup
    - File: `adapter/out/persistence/repository/DeviceTokenRepository.java`
    - _Requirements: 2.1, 2.2, 2.6, 18.1_
  - [ ] 8.6 Create `DeviceTokenPersistenceAdapter` implementing `SaveDeviceTokenPort` and `LoadDeviceTokenPort`
    - File: `adapter/out/persistence/DeviceTokenPersistenceAdapter.java`
    - Uses `DeviceTokenEncryptor` for encrypt/decrypt. Include `DeviceTokenPersistenceMapper`
    - _Requirements: 2.1, 2.5_
  - [ ] 8.7 Update `BookingJpaEntity` — add `reminderState` JSONB field with `ReminderStateConverter`
    - Modify existing `adapter/out/persistence/entity/BookingJpaEntity.java`
    - Create `ReminderStateConverter` (JPA AttributeConverter for JSONB ↔ Map<String, Instant>)
    - _Requirements: 16.3_
  - [ ] 8.8 Update `Booking` domain model — add `reminderState` field and methods (`hasReminderBeenSent`, `markReminderSent`)
    - Modify existing `domain/model/Booking.java`
    - Add `private Map<String, Instant> reminderState` field (default empty map)
    - Add to private constructor, `withId()` factory, and all `create*` factory methods (pass `Map.of()` or `new HashMap<>()` as default)
    - Add business methods: `hasReminderBeenSent(String key)`, `markReminderSent(String key, Instant sentAt)` (mutates in place, consistent with other business methods like `confirm()`, `cancel()`)
    - **BREAKING CHANGE**: `Booking.withId()` gains a new parameter — all callers must be updated: `BookingPersistenceMapper.toDomain()`, `IntegrationTestConfiguration.loadBookingPort()`, and any test code that calls `Booking.withId()` or `Booking.createCustomerBooking()` directly
    - _Requirements: 16.3_
  - [ ] 8.9 Update `BookingPersistenceMapper` to map `reminderState` between entity and domain
    - Modify existing `adapter/out/persistence/mapper/BookingPersistenceMapper.java`
    - _Requirements: 16.3_
  - [ ] 8.10 Update existing tests and `IntegrationTestConfiguration` for the new `reminderState` field
    - Modify existing `domain/model/BookingTest.java` — add test cases for `hasReminderBeenSent()` and `markReminderSent()` methods
    - Update `IntegrationTestConfiguration.loadBookingPort()` — the `Booking.createCustomerBooking()` call must be updated if the factory method signature changes, or the `reminderState` field must be set via the new constructor parameter
    - Update any existing test builders/factories that construct `Booking` or `BookingJpaEntity` to include the new `reminderState` field with a default empty map
    - Update `BookingPersistenceMapper.toDomain()` and `toEntity()` to pass/map `reminderState` (also covered in task 8.9, but verify compilation here)
    - _Requirements: 16.3_

- [ ] 9. Redis adapters — Deduplication and Preference Cache (Transaction Service)
  - [ ] 9.1 Create `NotificationDeduplicationRedisAdapter` implementing `NotificationDeduplicationPort`
    - File: `adapter/out/redis/NotificationDeduplicationRedisAdapter.java`
    - Key pattern: `notification:processed:{eventId}`, TTL: 7 days
    - _Requirements: 1.11, 22.1, 22.2, 22.3_
  - [ ] 9.2 Create `NotificationPreferenceCacheRedisAdapter` implementing `NotificationPreferenceCachePort`
    - File: `adapter/out/redis/NotificationPreferenceCacheRedisAdapter.java`
    - Key pattern: `notification:prefs:{userId}`, TTL: 5 minutes. Serialize `UserNotificationPreferences` as JSON
    - _Requirements: 20.5_

- [ ] 10. External service adapters — FCM, SendGrid, Web Push (Transaction Service)
  - [ ] 10.1 Create `FcmPushAdapter` implementing `FcmPushPort`
    - File: `adapter/out/fcm/FcmPushAdapter.java`
    - Uses Firebase Admin SDK `FirebaseMessaging.getInstance().sendEachForMulticast()`. Constructs `MulticastMessage` with localized title/body, data payload, platform-specific config (Android/iOS), priority mapping (HIGH for CRITICAL, NORMAL otherwise), click_action deep link
    - Uses `RetryWithBackoff` utility (task 7.1) for exponential backoff retry: 1s, 2s, 4s, 8s, 16s (max 60s), up to 3 attempts on transient errors (`messaging/server-unavailable`, `messaging/internal-error`)
    - Process `BatchResponse` — **remove invalid tokens immediately** when FCM returns `messaging/registration-token-not-registered` or `messaging/invalid-registration-token` error codes (call `LoadDeviceTokenPort.deleteByToken()` within the same processing cycle)
    - _Requirements: 3.1, 3.2, 3.3, 3.5, 3.6, 3.7, 18.2, 25.1_
  - [ ] 10.2 Create `NoOpFcmPushAdapter` with `@ConditionalOnMissingBean(FirebaseApp.class)`
    - File: `adapter/out/fcm/NoOpFcmPushAdapter.java`
    - Logs FCM message at INFO, returns success result
    - _Requirements: Design — NoOp pattern_
  - [ ] 10.3 Create `FirebaseConfig` — initializes `FirebaseApp` from credentials path
    - File: `config/FirebaseConfig.java`
    - Conditional on `fcm.credentials-path` property pointing to a valid file
    - _Requirements: 3.1_
  - [ ]* 10.4 Write property test for FCM priority mapping (Property 10: FCM Priority Mapping)
    - **Property 10: FCM Priority Mapping** — For any urgency level, FCM priority is HIGH when CRITICAL, NORMAL otherwise
    - **Validates: Requirements 3.6**
  - [ ] 10.5 Create `SendGridEmailAdapter` implementing `SendGridEmailPort`
    - File: `adapter/out/email/SendGridEmailAdapter.java`
    - Uses SendGrid Java SDK. Sends dynamic template emails with `dynamic_template_data`
    - Uses `RetryWithBackoff` utility (task 7.1) for exponential backoff retry: 2s, 4s, 8s, up to 3 attempts for 5xx errors
    - Configure verified sender domain: use `${sendgrid.from-email}` (noreply@courtbooking.gr) as sender address and `${sendgrid.from-name}` as sender name
    - **Include unsubscribe link** in email footer: add `unsubscribeLink` to `dynamic_template_data` pointing to the notification preferences page (constructed from notification data's `deepLink` base URL + `/settings/notifications`)
    - _Requirements: 8.1, 8.2, 8.4, 8.5, 8.6_
  - [ ] 10.6 Create `NoOpSendGridEmailAdapter` with `@ConditionalOnProperty(name = "sendgrid.api-key", havingValue = "SG.placeholder")`
    - File: `adapter/out/email/NoOpSendGridEmailAdapter.java`
    - _Requirements: Design — NoOp pattern_
  - [ ] 10.7 Create `WebPushAdapter` implementing `WebPushPort`
    - File: `adapter/out/webpush/WebPushAdapter.java`
    - Uses `nl.martijndwars:web-push` library with VAPID keys. Handles 410/404 responses for expired subscriptions (remove immediately)
    - Uses `RetryWithBackoff` utility (task 7.1) for retry on 5xx and 429 errors
    - _Requirements: 9.4, 9.5_
  - [ ] 10.8 Create `NoOpWebPushAdapter` with `@ConditionalOnProperty(name = "web-push.vapid-private-key", havingValue = "")`
    - File: `adapter/out/webpush/NoOpWebPushAdapter.java`
    - _Requirements: Design — NoOp pattern_
  - [ ] 10.9 Create `WebPushConfig` — initializes web-push library with VAPID keys
    - File: `config/WebPushConfig.java`
    - Reads `${web-push.vapid-public-key}`, `${web-push.vapid-private-key}`, `${web-push.vapid-subject}` from environment
    - Initializes `PushService` bean from `nl.martijndwars:web-push` with the VAPID key pair
    - Conditional on `web-push.vapid-private-key` being non-empty (similar to `FirebaseConfig` pattern)
    - _Requirements: 9.1, 9.4_

- [ ] 11. Checkpoint — Ensure all adapters compile and NoOp adapters are wired
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Redis Pub/Sub publisher adapter (Transaction Service)
  - [ ] 12.1 Create `RedisPubSubPublisherAdapter` implementing `RedisPubSubPublisherPort`
    - File: `adapter/out/redis/RedisPubSubPublisherAdapter.java`
    - Publishes JSON-serialized messages to Redis channels: `ws:user:{userId}:notifications`, `ws:court:{courtId}:availability`, `ws:court:{courtId}:bookings`
    - Graceful degradation: on `RedisConnectionFailureException`, fall back to local-pod-only delivery via `SimpMessagingTemplate`, increment `redis_pubsub_unavailable` metric
    - _Requirements: 5.1, 5.2, 5.3, 5.4_
  - [ ] 12.2 Create `NoOpRedisPubSubPublisher` for tests/environments without Redis
    - File: `adapter/out/redis/NoOpRedisPubSubPublisher.java`
    - _Requirements: Design — NoOp pattern_

- [ ] 13. PlatformServiceHttpClient extension (Transaction Service)
  - [ ] 13.1 Extend `PlatformServiceHttpClient` with `getNotificationPreferences(UserId)` method
    - Calls `GET /internal/users/{userId}/notification-preferences`, deserializes to `UserNotificationPreferences`
    - _Requirements: 20.1, 20.5_
  - [ ] 13.2 Extend `PlatformServiceHttpClient` with `getReminderRules(CourtId)` method
    - Calls `GET /internal/courts/{courtId}/reminder-rules`, deserializes to `List<ReminderRuleInfo>`
    - _Requirements: 20.2_
  - [ ] 13.3 Extend `PlatformServiceHttpClient` with `updateDnd(UserId, ...)` method
    - Calls `PUT /internal/users/{userId}/dnd`
    - _Requirements: 20.7_

- [ ] 14. Application services — Device Token, Notification Center, DND (Transaction Service)
  - [ ] 14.1 Create `DeviceTokenService` implementing `RegisterDeviceTokenUseCase`, `RemoveDeviceTokenUseCase`, `ListDeviceTokensQuery`
    - File: `application/service/DeviceTokenService.java`
    - Registration: encrypt token, check for duplicate (idempotent — return 200 if exists), enforce 10-token limit (remove oldest LRU on 11th), persist
    - Removal: delete by deviceId, return 404 if not found
    - List: return metadata only (never raw tokens)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.8_
  - [ ]* 14.2 Write property test for device token limit (Property 6: Device Token Limit Per User)
    - **Property 6: Device Token Limit Per User** — For any sequence of registrations, count never exceeds 10; 11th registration removes oldest LRU
    - **Validates: Requirements 2.8**
  - [ ]* 14.3 Write property test for token secrecy (Property 7: Token Secrecy in API Responses)
    - **Property 7: Token Secrecy in API Responses** — For any device token, GET response never contains raw token value
    - **Validates: Requirements 2.4**
  - [ ] 14.4 Create `NotificationQueryService` implementing `ListNotificationsQuery`, `MarkNotificationReadUseCase`, `MarkAllNotificationsReadUseCase`, `GetUnreadCountQuery`
    - File: `application/service/NotificationQueryService.java`
    - List: paginated, cap page size at min(requested, 100), sort createdAt DESC, optional type filter
    - Mark read: set read=true, readAt=now, return 404 if not found
    - Mark all read: bulk update, return count
    - Unread count: count where read=false for user
    - _Requirements: 7.5, 7.6, 7.7, 7.8, 24.1, 24.2, 24.3_
  - [ ]* 14.5 Write property test for pagination cap (Property 16: Pagination Size Cap)
    - **Property 16: Pagination Size Cap** — For any requested page size, effective size is min(requested, 100)
    - **Validates: Requirements 24.1, 24.2**
  - [ ] 14.6 Create `DndConfigurationService` implementing `UpdateDndConfigurationUseCase`
    - File: `application/service/DndConfigurationService.java`
    - Delegates to `PlatformServicePort.updateDnd()`. Supports overnight windows (e.g., 22:00–07:00)
    - _Requirements: 10.6_
  - [ ] 14.7 Create `WebPushSubscriptionService` implementing `GetVapidKeyQuery`, `SubscribeWebPushUseCase`, `UnsubscribeWebPushUseCase`
    - File: `application/service/WebPushSubscriptionService.java`
    - Subscribe: encrypt subscription JSON, store as device_token with token_type=WEB_PUSH, idempotent on duplicate endpoint
    - _Requirements: 9.1, 9.2, 9.6_

- [ ] 15. WebSocket infrastructure (Transaction Service)
  - [ ] 15.1 Create `WebSocketConfig` — STOMP-over-WebSocket configuration
    - File: `config/WebSocketConfig.java`
    - Enable simple broker for `/topic`, `/queue` with 30s heartbeat. Application destination prefix `/app`. User destination prefix `/user`. Register `/ws` endpoint with SockJS fallback. Configure transport: 128KB message size, 512KB send buffer, 10s send timeout
    - _Requirements: 4.1, 4.4, 4.8, 19.3_
  - [ ] 15.2 Create `JwtHandshakeInterceptor` — validates JWT from query param on WebSocket handshake
    - File: `adapter/in/websocket/JwtHandshakeInterceptor.java`
    - Extract token from `?token=` query param. Validate with `JwtDecoder`. Store userId and role in session attributes. Reject with 4001 (TOKEN_EXPIRED) or 4002 (INVALID_TOKEN) close codes
    - _Requirements: 4.2, 4.3, 27.1, 27.2, 27.5_
  - [ ] 15.3 Create `StompSubscriptionInterceptor` — authorizes STOMP subscriptions
    - File: `adapter/in/websocket/StompSubscriptionInterceptor.java`
    - Validate `/topic/courts/{courtId}/bookings` subscriptions: only court owner allowed. `/user/queue/*` destinations are inherently user-scoped. `/topic/courts/{courtId}/availability` is open to all authenticated users
    - _Requirements: 4.5, 4.6, 4.7_
  - [ ] 15.4 Create `WebSocketConnectionRegistry` — in-memory concurrent map of userId → Set<sessionId>
    - File: `adapter/in/websocket/WebSocketConnectionRegistry.java`
    - Methods: registerSession, removeSession, isUserConnected, getSessionIds, getTotalConnectionCount
    - _Requirements: 4.10_
  - [ ] 15.5 Create `WebSocketEventListener` — handles STOMP connect/disconnect events
    - File: `adapter/in/websocket/WebSocketEventListener.java`
    - On connect: register session, log at INFO (userId, sessionId, remoteAddr), **deliver pending notifications** via `loadNotificationPort.loadPendingWebSocket(userId)` — send each pending notification to the user's `/user/queue/notifications` destination and mark as delivered (Req 13.3)
    - On disconnect: remove session, log at INFO, clean up registry
    - _Requirements: 4.9, 13.3, 13.5_
  - [ ] 15.6 Create `WebSocketSessionAdapter` implementing `WebSocketSessionPort`
    - File: `adapter/in/websocket/WebSocketSessionAdapter.java`
    - Delegates to `WebSocketConnectionRegistry` for connection checks and `SimpMessagingTemplate` for message sending
    - _Requirements: 4.10, 7.1, 7.4_
  - [ ] 15.7 Create `WebSocketGracefulShutdown` — @PreDestroy lifecycle hook
    - File: `adapter/in/websocket/WebSocketGracefulShutdown.java`
    - Send RECONNECT message to all connected clients on `/user/queue/system`, wait up to 30s for drain, force close remaining
    - _Requirements: 13.1, 13.4_
  - [ ] 15.8 Create `WebSocketThreadPoolConfig` — dedicated thread pool for WebSocket broadcasting
    - File: `config/WebSocketThreadPoolConfig.java`
    - Core 4, max 8, queue 1000, CallerRunsPolicy
    - _Requirements: 19.4_
  - [ ] 15.9 Update `SecurityConfig` — add `/ws/**` to permitAll (WebSocket auth handled by interceptor)
    - Modify existing `config/SecurityConfig.java`
    - _Requirements: 4.1, 27_

- [ ] 16. Redis Pub/Sub listener and broadcasting (Transaction Service)
  - [ ] 16.1 Create `RedisPubSubConfig` — configure `RedisMessageListenerContainer` with WebSocket broadcast listeners and auto-reconnection
    - File: `config/RedisPubSubConfig.java`
    - **IMPORTANT**: The existing `RedisConfig.java` already defines a `RedisMessageListenerContainer` bean used by `SlotHoldExpiryListener` for keyspace notifications. Phase 5 must either: (a) modify the existing `RedisConfig` to add Pub/Sub channel subscriptions alongside the keyspace listener, or (b) create `RedisPubSubConfig` and **remove** the `redisMessageListenerContainer` bean from `RedisConfig` to avoid a duplicate bean conflict, re-registering `SlotHoldExpiryListener` in the new config. Option (b) is recommended — consolidate all Redis listener configuration in `RedisPubSubConfig`
    - Add Pub/Sub channel subscriptions for notification, availability, and booking channels
    - **Implement Redis connection listener** for auto-reconnection: detect Redis reconnection events and re-subscribe to all active channels without requiring pod restarts (Req 5.5). Use Spring Data Redis `RedisConnectionFactory` connection listener or `LettuceConnectionFactory.setValidateConnection(true)` to detect reconnection
    - _Requirements: 5.1, 5.5_
  - [ ] 16.2 Create `WebSocketBroadcastListener` — receives Redis Pub/Sub messages and broadcasts to local WebSocket clients
    - File: `adapter/in/redis/WebSocketBroadcastListener.java`
    - Routes messages from `ws:user:{userId}:notifications` → `/user/queue/notifications`, `ws:court:{courtId}:availability` → `/topic/courts/{courtId}/availability`, `ws:court:{courtId}:bookings` → `/user/queue/bookings` (resolved per user)
    - **Availability update transformation**: Maintain a local cache of slot states per court/date. On subscription, fetch current availability from Platform Service and cache locally. On delta update from Redis Pub/Sub, update local cache and broadcast full slot list. Cache eviction after 5 min of no subscribers. On cache miss for incoming delta, fetch fresh data from Platform Service before broadcasting. **NOTE**: This creates a Transaction Service → Platform Service call path during availability updates. Use async fetch with a short timeout (500ms) and circuit breaker to prevent cascading failures. If Platform Service is unreachable, broadcast the raw delta message instead of the full slot list — clients can handle incremental updates
    - **Booking status update routing**: Resolve the booking's customer userId and court owner userId from the booking update message, then route to both users' personal `/user/queue/bookings` queues (not a court-level topic)
    - _Requirements: 5.2, 6.2, 6.3, 7.1_

- [ ] 17. Checkpoint — Ensure WebSocket infrastructure compiles and Redis Pub/Sub is wired
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 18. Notification Processor pipeline (Transaction Service)
  - [ ]* 18.1 Write property test for DND queuing (Property 3: DND Queuing for STANDARD Urgency)
    - **Property 3: DND Queuing for STANDARD Urgency** — For any STANDARD notification and any DND window (including overnight), notification is queued with correct deliver_after
    - **Validates: Requirements 1.5, 10.4**
  - [ ] 18.2 Create `NotificationProcessingService` implementing `ProcessNotificationUseCase`
    - File: `application/service/NotificationProcessingService.java`
    - **Dependencies**: `NotificationDeduplicationPort`, `SaveNotificationPort`, `LoadNotificationPort`, `NotificationPreferenceCachePort`, `PlatformServicePort`, `UserBasicPort` (for user language from `v_user_basic.language`), `WebSocketSessionPort`, `FcmPushPort`, `SendGridEmailPort`, `WebPushPort`, `RedisPubSubPublisherPort`, `LoadDeviceTokenPort`, `SendGridTemplateResolver`, `MeterRegistry`
    - Implements the full pipeline: deduplicate → persist → resolve preferences → DND check → route channels → deliver → update status
    - Channel routing logic: CRITICAL bypasses all prefs/DND; STANDARD respects DND; PROMOTIONAL respects opt-out; WebSocket takes priority over FCM when connected; email always if in channels and enabled
    - Preference resolution: fetch from cache (Redis 5-min TTL), on miss call Platform Service internal API, on API failure use cached or default-open
    - Localization: select title/body by user language from `UserBasicPort.loadById(userId).language` (default `el`)
    - **DND window check**: Implement `isInDndWindow()` logic in Transaction Service (either as a static utility method on the `UserNotificationPreferences.DndConfiguration` record from task 3.9, or as a private helper in this service). The logic must handle same-day windows (e.g., 09:00–17:00) and overnight windows (e.g., 22:00–07:00) using the user's configured timezone. Do NOT depend on Platform Service's `DndConfiguration` domain model — Transaction Service has its own DTO
    - Enforce 1000-per-user notification limit (delete oldest read, then oldest unread)
    - WebSocket payload size limit: truncate body if >16KB, limit data to essential fields if >4KB
    - Timeout guard: mark remaining channels FAILED if total delivery exceeds 5 minutes
    - **FCM invalid token removal**: When processing `FcmDeliveryResult`, iterate `TokenResult` list and call `LoadDeviceTokenPort.deleteByToken()` for any token with error code `messaging/registration-token-not-registered` or `messaging/invalid-registration-token` — removal happens immediately within the same processing cycle (Req 2.6, 3.5, 18.2)
    - **Structured logging**: Log notification delivery events at INFO level with structured fields: `notificationId`, `userId`, `notificationType`, `channel`, `status`, `latencyMs` (Req 17.4)
    - **Kafka consumer lag monitoring**: Track consumer lag and log WARN + increment `notification_consumer_lag_high` metric when lag exceeds 1000 messages (Req 17.5) — implemented via Micrometer's `KafkaConsumerMetrics` or manual lag calculation in the consumer
    - _Requirements: 1.1–1.13, 2.6, 3.5, 10.1–10.5, 12.1–12.5, 14.5, 17.4, 17.5, 18.2, 21.1, 26.1–26.3, 30.1–30.3_
  - [ ]* 18.3 Write property test for channel routing (Property 1: Channel Routing Resolution)
    - **Property 1: Channel Routing Resolution** — Resolved channels are subset of event channels filtered by preferences, WebSocket priority over FCM when connected, email always when specified and enabled
    - **Validates: Requirements 1.1, 1.7, 1.8, 1.9, 1.10, 12.1, 12.2, 12.4**
  - [ ]* 18.4 Write property test for CRITICAL bypass (Property 2: CRITICAL Urgency Bypass)
    - **Property 2: CRITICAL Urgency Bypass** — CRITICAL notifications deliver to all available channels regardless of preferences and DND
    - **Validates: Requirements 1.4, 10.3**
  - [ ]* 18.5 Write property test for opt-out skip (Property 4: Opt-Out Skip for PROMOTIONAL Urgency)
    - **Property 4: Opt-Out Skip for PROMOTIONAL Urgency** — PROMOTIONAL notifications with no preference entry for the event type are skipped with SKIPPED_OPT_OUT. NOTE: The existing `NotificationPreference` domain model enforces at least one channel enabled per entry (via `validate()`), so opt-out is represented by the **absence** of a preference entry for the notification type, not by all channels being false. The test must verify: (a) when no preference entry exists for the event type → SKIPPED_OPT_OUT, (b) when a preference entry exists (even with some channels disabled) → deliver to enabled channels
    - **Validates: Requirements 1.6, 10.5**
  - [ ]* 18.6 Write property test for deduplication (Property 5: Event Deduplication Idempotency)
    - **Property 5: Event Deduplication Idempotency** — Same eventId processed twice results in exactly one notification and one delivery set
    - **Validates: Requirements 1.11, 22.1, 22.2, 22.3**
  - [ ]* 18.7 Write property test for localization (Property 9: Localization Selection)
    - **Property 9: Localization Selection** — Correct language selected based on user preference, default to `el`
    - **Validates: Requirements 3.2, 8.7, 21.1**
  - [ ]* 18.8 Write property test for notification count limit (Property 14: Notification Count Limit Per User)
    - **Property 14: Notification Count Limit Per User** — Count never exceeds 1000; oldest read deleted first, then oldest unread
    - **Validates: Requirements 14.5**
  - [ ]* 18.9 Write property test for WebSocket notification message completeness (Property 12: WS Notification Message Completeness)
    - **Property 12: WebSocket Notification Message Completeness** — All required fields present: notificationId, type, urgency, title, body, data, createdAt, read=false
    - **Validates: Requirements 7.2**
  - [ ]* 18.10 Write property test for independent channel status tracking (Property 13: Independent Channel Status Tracking)
    - **Property 13: Independent Channel Status Tracking** — Per-channel status updates are independent; overall status derived correctly
    - **Validates: Requirements 12.5, 14.3**
  - [ ]* 18.11 Write property test for WebSocket payload size limit (Property 17: WS Payload Size Limit)
    - **Property 17: WebSocket Payload Size Limit** — Serialized payload ≤ 16KB, data field ≤ 4KB
    - **Validates: Requirements 30.1, 30.2**
  - [ ] 18.12 Create `NotificationEventKafkaConsumer` — Kafka listener for `notification-events` topic
    - File: `adapter/in/kafka/NotificationEventKafkaConsumer.java`
    - Consumer group: `transaction-service-notification-consumer`, `auto.offset.reset=latest`
    - **Kafka deserializer override**: The global `application.yml` configures `value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer` for the default consumer group. This consumer uses raw JSON parsing via `ObjectMapper.readTree()`, so it MUST override the deserializer to `StringDeserializer` in the `@KafkaListener` annotation: `properties = {"value.deserializer=org.apache.kafka.common.serialization.StringDeserializer"}`
    - Deserializes raw JSON via `ObjectMapper.readTree()`, maps to `NotificationEvent` record (task 1.7) handling both Transaction Service and Platform Service publisher formats (bilingual title/body objects vs plain strings, channels as array vs map)
    - Delegates to `NotificationProcessingService.processNotificationEvent(NotificationEvent)`
    - Gracefully ignores unknown/deferred notification types (Phase 10 types)
    - **Kafka consumer lag detection**: Implement consumer lag monitoring — when lag exceeds 1000 messages, log WARN and increment `notification_consumer_lag_high` metric (Req 17.5). Use `Consumer.metrics()` or `KafkaListenerEndpointRegistry` to access lag info
    - _Requirements: 1.1, 1.2, 1.12, 1.13, 17.5_

- [ ] 19. REST controllers — Notification APIs (Transaction Service)
  - [ ] 19.1 Create `DeviceTokenController`
    - File: `adapter/in/web/DeviceTokenController.java`
    - `POST /api/notifications/devices` — register device token (201 Created or 200 OK for duplicate)
    - `GET /api/notifications/devices` — list registered devices (metadata only)
    - `DELETE /api/notifications/devices/{deviceId}` — remove device token (204 or 404)
    - _Requirements: 2.1, 2.3, 2.4_
  - [ ] 19.2 Create `NotificationController`
    - File: `adapter/in/web/NotificationController.java`
    - `GET /api/notifications` — paginated notification history with optional type filter
    - `POST /api/notifications/{id}/read` — mark notification as read
    - `POST /api/notifications/read-all` — mark all as read
    - `GET /api/notifications/unread-count` — get unread count
    - _Requirements: 7.5, 7.6, 7.7, 7.8, 24.1, 24.2, 24.3_
  - [ ] 19.3 Create `WebPushController`
    - File: `adapter/in/web/WebPushController.java`
    - `GET /api/notifications/web-push/vapid-key` — return VAPID public key
    - `POST /api/notifications/web-push/subscribe` — register browser push subscription (201 or 200 for duplicate)
    - `DELETE /api/notifications/web-push/subscribe` — unsubscribe (204)
    - _Requirements: 9.1, 9.2, 9.6_
  - [ ] 19.4 Create `DndController`
    - File: `adapter/in/web/DndController.java`
    - `PUT /api/notifications/dnd` — configure DND (delegates to Platform Service via DndConfigurationService)
    - Input: `{ enabled, startTime, endTime, timezone }`. Supports overnight windows
    - _Requirements: 10.6, 31.2_

- [ ] 20. Checkpoint — Ensure Notification Processor and REST controllers compile
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 21. Platform Service changes — Internal APIs, BookingEventKafkaConsumer extension, AvailabilityPubSub
  - [ ] 21.1 Create `DndConfiguration` value object in Platform Service domain
    - File: `platform-service/.../domain/model/DndConfiguration.java`
    - Record: enabled, startTime, endTime, timezone. Include `isInDndWindow(Instant)` method
    - _Requirements: 31.1_
  - [ ] 21.2 Extend `NotificationPreferencesPersistencePort` (or create `DndConfigurationPersistencePort`) for DND column access
    - Add methods to load and save DND config from `court_owner_notif_prefs` table
    - Update JPA entity and repository for the new DND columns
    - _Requirements: 31.1, 31.3_
  - [ ] 21.3 Create or extend use case for fetching notification preferences + DND for internal API
    - New `GetNotificationPreferencesForUserQuery` use case or extend `ManageNotificationPreferencesUseCase`
    - Loads preferences via `NotificationPreferencesPersistencePort.findByOwnerId()` and DND config from new columns
    - _Requirements: 20.1, 31.3_
  - [ ] 21.4 Create or extend use case for updating DND configuration
    - New `UpdateDndConfigurationUseCase` in Platform Service
    - Updates DND columns on `court_owner_notif_prefs`, invalidates Redis preference cache (`notification:prefs:{userId}`)
    - _Requirements: 20.7, 31.2, 31.4_
  - [ ] 21.5 Extend `InternalUserController` — add `GET /internal/users/{userId}/notification-preferences` endpoint
    - Returns preferences array + doNotDisturb object. 404 if user not found
    - **Validate `X-Internal-Api-Key` header** against `INTERNAL_API_KEY` environment variable for service-to-service authentication (consistent with existing internal endpoints from Phase 3). **NOTE**: The existing `InternalUserController` and `InternalCourtController` do NOT currently validate any API key — this is a gap from Phase 3. Create a shared `InternalApiKeyFilter` (Spring `OncePerRequestFilter` or `HandlerInterceptor`) that validates the `X-Internal-Api-Key` header on all `/internal/**` endpoints, and register it in Platform Service's `SecurityConfig` or as a `WebMvcConfigurer` interceptor. This filter is shared across tasks 21.5, 21.6, and 21.7
    - _Requirements: 20.1, 20.4, 31.3_
  - [ ] 21.6 Extend `InternalUserController` — add `PUT /internal/users/{userId}/dnd` endpoint
    - Accepts `{ enabled, startTime, endTime, timezone }`, returns saved config. Invalidates Redis cache
    - **Validate `X-Internal-Api-Key` header** for service-to-service authentication
    - _Requirements: 20.4, 20.7, 31.2_
  - [ ] 21.7 Extend `InternalCourtController` — add `GET /internal/courts/{courtId}/reminder-rules` endpoint
    - Returns rules array with ruleType, enabled, hoursBefore. 404 if court not found
    - **Validate `X-Internal-Api-Key` header** for service-to-service authentication
    - Create `GetReminderRulesForCourtQuery` use case (or extend `ManageReminderRulesUseCase`) — loads rules via `ReminderRulePersistencePort.findByOwnerIdAndCourtId()`, filtering to the court's owner
    - _Requirements: 20.2, 20.4_
  - [ ] 21.8 Create `AvailabilityRedisPubSubPublisher` implementing `AvailabilityPubSubPort`
    - File: `platform-service/.../adapter/out/redis/AvailabilityRedisPubSubPublisher.java`
    - Publishes `AvailabilityUpdateMessage` JSON to `ws:court:{courtId}:availability` Redis channel
    - _Requirements: 6.1, 28.1_
  - [ ] 21.9 Create `NoOpAvailabilityPubSubPublisher` for environments without Redis
    - File: `platform-service/.../adapter/out/redis/NoOpAvailabilityPubSubPublisher.java`
    - _Requirements: Design — NoOp pattern_
  - [ ] 21.10 Extend `BookingEventKafkaConsumer` in Platform Service
    - Add handling for `SLOT_HELD` and `SLOT_RELEASED` event types
    - For ALL booking event types (existing + new): publish availability update to Redis Pub/Sub via `AvailabilityPubSubPort`
    - **BUG FIX — BOOKING_MODIFIED field names**: The current consumer reads `oldCourtId` and `oldDate` from the JSON payload, but `BookingModifiedEvent` in Transaction Service serializes as `previousDate` and `previousStartTime` (Java record field names). This means BOOKING_MODIFIED cache invalidation for the old date is currently broken. Fix the consumer to read `previousDate` instead of `oldDate`, and `courtId` for the new court (the record doesn't have `previousCourtId` — cross-court modification is not supported). Also note: `BookingModifiedEvent` does not include `previousEndTime` — for availability updates, compute the affected slot end time from `previousStartTime` + court's `durationMinutes` (fetched from the event's `courtId`), or include `previousEndTime` in the event payload by extending the `BookingModifiedEvent` record in Transaction Service
    - Record `availability_update_latency_seconds` histogram metric
    - _Requirements: 6.1, 6.5, 28.1_
  - [ ]* 21.11 Write property test for availability update message completeness (Property 11: Availability Update Message Completeness)
    - **Property 11: Availability Update Message Completeness** — For any booking event type, generated message contains all required fields: courtId, date, eventType, affectedSlot (startTime, endTime), updatedAt
    - **Validates: Requirements 6.3**
  - [ ] 21.12 Update Platform Service test configurations for new `AvailabilityPubSubPort`
    - Update `TestPortsConfiguration.java` to add a NoOp/mock bean for `AvailabilityPubSubPort`
    - Update `Phase3PortStubsConfiguration.java` to stub the new port if needed
    - Ensure existing Platform Service integration tests (`BookingEventConsumerTest`, `Phase3EndpointSmokeTest`, etc.) continue passing with the new port dependency
    - _Requirements: Design — test configuration_

- [ ] 22. Scheduled jobs — DND flush, cleanup, reminders (Transaction Service)
  - [ ] 22.1 Create `DndQueueFlushJob` — Quartz job running every 15 minutes
    - File: `adapter/in/scheduler/DndQueueFlushJob.java`
    - Queries `DND_QUEUED` notifications where `deliver_after < now()`, batches of 100, FIFO order
    - Updates status from DND_QUEUED to PENDING, routes through normal channel delivery
    - Individual failure handling (no batch rollback). Publishes metrics: `dnd_queue_flush_processed_total`, `dnd_queue_flush_failed_total`, `dnd_queue_depth`
    - _Requirements: 10.7, 23.1, 23.2, 23.3, 23.4_
  - [ ] 22.2 Create `NotificationCleanupJob` — daily Quartz job
    - File: `adapter/in/scheduler/NotificationCleanupJob.java`
    - Deletes notifications older than 90 days. Enforces 1000-per-user limit (delete oldest read first, then oldest unread)
    - _Requirements: 14.2, 14.5_
  - [ ] 22.3 Create `DeviceTokenCleanupJob` — daily Quartz job
    - File: `adapter/in/scheduler/DeviceTokenCleanupJob.java`
    - Removes device tokens and Web Push subscriptions unused for 90 days. Logs count removed, publishes `device_tokens_cleaned_total` metric. Logs WARN if user's last token removed
    - _Requirements: 2.7, 9.7, 18.1, 18.4, 18.5_
  - [ ] 22.4 Create `BookingReminderJob` and `BookingReminderSchedulingService`
    - Files: `adapter/in/scheduler/BookingReminderJob.java`, `application/service/BookingReminderSchedulingService.java`
    - `BookingReminderSchedulingService`: on booking confirmation, fetch court's BOOKING_REMINDER rules from Platform Service, schedule Quartz trigger at `bookingStartTime - reminderHoursBefore` (default 2h if no rule). On booking cancellation, unschedule trigger. Use court timezone for calculations. **Dependencies**: `PlatformServicePort` (for reminder rules), `CourtSummaryPort` (for court timezone), `Scheduler` (Quartz), `NotificationEventPublisherPort`, `LoadBookingPort`
    - `BookingReminderJob`: when trigger fires, publish `NOTIFICATION_REQUESTED` event with type=BOOKING_REMINDER, urgency=STANDARD, channels=[PUSH, IN_APP], include booking details and deep link
    - _Requirements: 15.1, 15.2, 15.3, 15.4, 15.5, 15.6_
  - [ ] 22.5 Extend `PendingConfirmationReminderJob` — implement notification publishing
    - Modify existing `adapter/in/scheduler/PendingConfirmationReminderJob.java`
    - **Add `SaveBookingPort` dependency** — the existing skeleton only injects `LoadBookingPort`, `CourtSummaryPort`, and `NotificationEventPublisherPort`. Phase 5 needs `SaveBookingPort` to persist `reminder_state` updates after sending reminders
    - **Add new `LoadBookingPort` method**: `loadPendingForReminder(Instant createdBefore, String thresholdKey, int limit)` — queries bookings in `PENDING_CONFIRMATION` status created before the threshold time where `reminder_state` does not contain the threshold key. Add this method to `LoadBookingPort` interface, `BookingRepository` (native JPQL with `NOT (b.reminderState ? :thresholdKey)` or equivalent), and `BookingPersistenceAdapter`
    - Query bookings in PENDING_CONFIRMATION status past reminder thresholds (1h, 4h, 12h) using the new port method
    - Track sent thresholds via `reminder_state` JSONB column: after sending a reminder, call `booking.markReminderSent(thresholdKey, Instant.now())` then `saveBookingPort.save(booking)`
    - Publish `NOTIFICATION_REQUESTED` events with type=PENDING_CONFIRMATION_REMINDER, urgency=STANDARD, channels=[PUSH, EMAIL, IN_APP, WEB_PUSH]
    - Skip bookings that have been confirmed/rejected. Include time remaining before auto-cancellation and deep link
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5_
  - [ ] 22.6 ~~Extend `BookingCompletionJob`~~ — Verify existing implementation (NO-OP)
    - The existing `BookingCompletionJob` already publishes `BOOKING_COMPLETED` events to the `booking-events` Kafka topic via `bookingEventPublisherPort.publishBookingCompleted()` (implemented in Phase 4). Phase 5 does NOT add customer-facing notifications for booking completion. **This task is a verification-only step**: confirm the existing code publishes the event correctly and no changes are needed
    - _Requirements: 29.2_
  - [ ] 22.7 Register new Quartz jobs in `QuartzConfig`
    - Modify existing `config/QuartzConfig.java`
    - Register `DndQueueFlushJob` (cron: `0 0/15 * * * ?`), `NotificationCleanupJob` (cron: `0 0 2 * * ?`), `DeviceTokenCleanupJob` (cron: `0 0 3 * * ?`)
    - _Requirements: 29.1, 29.3_

- [ ] 23. Checkpoint — Ensure all scheduled jobs and Platform Service changes compile
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 24. Prometheus metrics and structured logging (Transaction Service + Platform Service)
  - [ ] 24.1 Register all notification delivery metrics in Transaction Service
    - Counters: `notification_events_consumed_total`, `notification_delivery_total`, `notification_delivery_failures_total`, `notification_dnd_queued_total`, `notification_duplicates_skipped_total`, `notification_preference_fetch_failures_total`, `notification_consumer_lag_high`, `websocket_messages_delivered_total`, `redis_pubsub_messages_total`, `redis_pubsub_unavailable`, `device_tokens_cleaned_total`, `dnd_queue_flush_processed_total`, `dnd_queue_flush_failed_total`
    - Gauges: `websocket_connections_active`, `dnd_queue_depth`
    - Histograms: `notification_delivery_latency_seconds`, `websocket_connection_duration_seconds`
    - Instrument all adapters and services with appropriate metric recording
    - _Requirements: 5.7, 12.6, 17.3, 19.6, 22.4, 23.4_
  - [ ] 24.2 Implement structured logging for notification delivery events
    - Add INFO-level structured log statements in `NotificationProcessingService` with fields: `notificationId`, `userId`, `notificationType`, `channel`, `status`, `latencyMs`
    - Add WARN-level log for Kafka consumer lag exceeding 1000 messages in `NotificationEventKafkaConsumer`
    - Add WARN-level log for notification payload truncation (original size + notificationId)
    - _Requirements: 17.4, 17.5, 30.3_
  - [ ] 24.3 Register `availability_update_latency_seconds` histogram in Platform Service
    - Measure from Kafka event timestamp to Redis Pub/Sub publish in `BookingEventKafkaConsumer`
    - _Requirements: 6.5_

- [ ] 25. Configuration — application.yml and build.gradle.kts updates
  - [ ] 25.1 Add Phase 5 configuration to Transaction Service `application.yml`
    - FCM config: `fcm.credentials-path`, `fcm.project-id`
    - SendGrid config: `sendgrid.api-key`, `sendgrid.from-email`, `sendgrid.from-name`
    - Web Push config: `web-push.vapid-public-key`, `web-push.vapid-private-key`, `web-push.vapid-subject`
    - Device token encryption: `device-token.encryption-key`
    - Notification processing: `notification.preference-cache-ttl`, `notification.dedup-ttl`, `notification.max-per-user`, `notification.retention-days`, `notification.max-retry-duration`
    - **Kafka consumer config**: Add a listener-specific configuration for the `notification-events` consumer group with `auto.offset.reset=latest`. NOTE: The `@KafkaListener` annotation in task 18.12 overrides the deserializer to `StringDeserializer` via `properties` attribute, so no global deserializer change is needed here. However, the consumer group ID (`transaction-service-notification-consumer`) should be documented in the YAML comments for clarity
    - **SendGrid template ID environment variables** (16 total — 8 notification types × 2 languages): `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EL`, `SENDGRID_TEMPLATE_BOOKING_CONFIRMED_EN`, `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EL`, `SENDGRID_TEMPLATE_BOOKING_CANCELLED_EN`, `SENDGRID_TEMPLATE_BOOKING_REJECTED_EL`, `SENDGRID_TEMPLATE_BOOKING_REJECTED_EN`, `SENDGRID_TEMPLATE_BOOKING_REMINDER_EL`, `SENDGRID_TEMPLATE_BOOKING_REMINDER_EN`, `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EL`, `SENDGRID_TEMPLATE_PAYMENT_RECEIPT_EN`, `SENDGRID_TEMPLATE_REFUND_COMPLETED_EL`, `SENDGRID_TEMPLATE_REFUND_COMPLETED_EN`, `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EL`, `SENDGRID_TEMPLATE_PENDING_CONFIRMATION_EN`, `SENDGRID_TEMPLATE_PENDING_REMINDER_EL`, `SENDGRID_TEMPLATE_PENDING_REMINDER_EN` — with placeholder values for local development
    - _Requirements: 1.2, 2.5, 8.5, 9.1, 20.5, 21.2, 22.1_
  - [ ] 25.3 Add Phase 5 configuration to Platform Service `application.yml` (if needed)
    - Add `INTERNAL_API_KEY` environment variable reference for the `InternalApiKeyFilter` (task 21.5)
    - No new Redis or Kafka configuration needed — Platform Service already has Redis and Kafka configured from Phase 3. Redis Pub/Sub publishing uses the existing `StringRedisTemplate`
    - _Requirements: 20.4_
  - [ ] 25.2 Add Phase 5 dependencies to Transaction Service `build.gradle.kts`
    - `com.google.firebase:firebase-admin:9.3.0` (FCM)
    - `com.sendgrid:sendgrid-java:4.10.2` (SendGrid)
    - `nl.martijndwars:web-push:5.1.1` (Web Push)
    - `org.bouncycastle:bcprov-jdk18on:1.78.1` (BouncyCastle for Web Push)
    - _Requirements: Design — dependency additions_

- [ ] 26. Integration wiring and booking status broadcasting (Transaction Service)
  - [ ] 26.1 Wire booking status updates to Redis Pub/Sub for real-time admin portal
    - When a booking is confirmed, rejected, or modified, publish booking status update to `ws:court:{courtId}:bookings` Redis channel via `RedisPubSubPublisherPort`
    - Integrate into existing `ConfirmBookingService`, `RejectBookingService`, `ModifyBookingService` (add `RedisPubSubPublisherPort` calls)
    - _Requirements: 6.6_
  - [ ] 26.2 Wire `ScheduleBookingReminderUseCase` into `ConfirmBookingService`
    - On booking confirmation, call `scheduleReminder(bookingId, courtId)`
    - _Requirements: 15.1_
  - [ ] 26.3 Wire `CancelBookingReminderUseCase` into `CancelBookingService`
    - On booking cancellation, call `cancelReminder(bookingId)`
    - _Requirements: 15.3_
  - [ ] 26.4 Update `GlobalExceptionHandler` in Transaction Service for Phase 5 error responses
    - Handle device token validation errors (400), notification not found (404), encryption errors
    - _Requirements: 2.1, 7.6_
  - [ ] 26.5 Update Transaction Service `IntegrationTestConfiguration` for Phase 5 ports
    - Add NoOp/mock beans for all new Phase 5 outgoing ports: `FcmPushPort`, `SendGridEmailPort`, `WebPushPort`, `WebSocketSessionPort`, `RedisPubSubPublisherPort`, `NotificationDeduplicationPort`, `NotificationPreferenceCachePort`, `SaveNotificationPort`, `LoadNotificationPort`, `SaveDeviceTokenPort`, `LoadDeviceTokenPort`
    - Ensure existing Transaction Service integration tests (`Phase4EndpointSmokeTest`, `StripeWebhookIntegrationTest`, `BookingCancellationIntegrationTest`, `ManualBookingIntegrationTest`, `CustomerBookingManualModeIntegrationTest`, `CustomerBookingInstantModeIntegrationTest`, `CourtUpdateEventsIntegrationTest`) continue passing with the new port dependencies
    - _Requirements: Design — test configuration_

- [ ] 27. Checkpoint — Full compilation and wiring verification
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 28. Unit tests for core services (Transaction Service)
  - [ ]* 28.1 Write unit tests for `NotificationProcessingService`
    - Test pipeline ordering: persist before deliver, status updates after delivery
    - Test malformed event handling (log WARN, skip)
    - Test unknown notification type handling (log DEBUG, skip)
    - Test preference resolution fallback (API failure → cached → default-open)
    - Test WebSocket fallback to FCM on connection drop
    - Test 5-minute timeout guard
    - Test FCM invalid token removal during delivery (Req 2.6, 3.5, 18.2)
    - Test structured logging output (notificationId, userId, notificationType, channel, status, latencyMs)
    - _Requirements: 1.3, 1.12, 2.6, 3.5, 12.3, 17.4, 18.2, 20.6, 26.3_
  - [ ]* 28.2 Write unit tests for `DeviceTokenService`
    - Test registration idempotency (duplicate returns 200)
    - Test 10-token limit enforcement
    - Test removal (success and not-found)
    - _Requirements: 2.1, 2.8_
  - [ ]* 28.3 Write unit tests for `DndConfiguration.isInDndWindow()`
    - Test same-day window (09:00–17:00)
    - Test overnight window (22:00–07:00)
    - Test edge cases: 23:59→00:01 transition, timezone boundaries
    - Test disabled DND, null start/end times
    - _Requirements: 10.4_
  - [ ]* 28.4 Write unit tests for `BookingReminderSchedulingService`
    - Test schedule on confirm, cancel on booking cancel
    - Test default 2h reminder when no rule configured
    - Test timezone handling
    - _Requirements: 15.1, 15.3, 15.4, 15.5_
  - [ ]* 28.5 Write unit tests for `PendingConfirmationReminderJob`
    - Test 1h, 4h, 12h threshold detection
    - Test reminder_state tracking (no duplicate reminders)
    - Test skip for confirmed/rejected bookings
    - _Requirements: 16.1, 16.2, 16.3, 16.4_
  - [ ]* 28.6 Write unit tests for `NotificationCleanupJob`
    - Test 90-day retention cleanup
    - Test 1000-per-user limit enforcement
    - _Requirements: 14.2, 14.5_
  - [ ]* 28.7 Write unit tests for `DndQueueFlushJob`
    - Test batch of 100, FIFO order
    - Test individual failure handling (no batch rollback)
    - _Requirements: 23.1, 23.2, 23.3_
  - [ ]* 28.8 Write unit tests for `WebSocketConnectionRegistry`
    - Test register, remove, multiple sessions per user
    - Test concurrent access safety
    - _Requirements: 4.10_
  - [ ]* 28.9 Write unit tests for `RetryWithBackoff` utility
    - Test exponential delay calculation (1s, 2s, 4s, 8s, 16s)
    - Test max delay cap (60s for FCM, 8s for SendGrid)
    - Test max attempts enforcement (3 attempts)
    - Test retryable predicate filtering
    - _Requirements: 3.3, 8.4_
  - [ ]* 28.10 Write unit tests for `WebSocketBroadcastListener`
    - Test availability update local cache: fetch on subscription, update on delta, evict after 5 min
    - Test booking status update routing: resolve customer userId and court owner userId, route to both personal queues
    - Test cache miss handling: fetch fresh data from Platform Service
    - _Requirements: 6.2, 6.3_

- [ ] 29. Integration tests (Transaction Service + Platform Service)
  - [ ]* 29.1 Write `NotificationProcessorIntegrationTest`
    - End-to-end: Kafka event → deduplication → DB persist → mock FCM/SendGrid delivery → status update
    - Uses Testcontainers (PostgreSQL, Redis) and embedded Kafka
    - _Requirements: 1.1, 1.3, 1.11_
  - [ ]* 29.2 Write `WebSocketIntegrationTest`
    - STOMP client connection with JWT → subscription → message delivery via Redis Pub/Sub
    - Test authentication failure (expired token, invalid token)
    - Test subscription authorization (non-owner rejected from bookings topic)
    - _Requirements: 4.1, 4.2, 4.7, 27.1, 27.2_
  - [ ]* 29.3 Write `DeviceTokenIntegrationTest`
    - REST API → DB: register, list (no raw tokens), delete
    - Verify encryption at rest (DB value ≠ plaintext)
    - _Requirements: 2.1, 2.4, 2.5_
  - [ ]* 29.4 Write `AvailabilityBroadcastIntegrationTest`
    - Kafka booking event → Platform Service consumer → Redis Pub/Sub → Transaction Service listener → WebSocket client
    - Verify end-to-end flow and message format
    - _Requirements: 6.1, 6.2, 6.3, 28.1_
  - [ ]* 29.5 Write `InternalApiIntegrationTest`
    - Transaction Service → Platform Service internal APIs: preference fetch, reminder rules, DND update
    - Verify `X-Internal-Api-Key` header validation on all internal endpoints
    - _Requirements: 20.1, 20.2, 20.4, 20.7_
  - [ ]* 29.6 Write `DndQueueFlushIntegrationTest`
    - Quartz job → DB query → delivery pipeline
    - _Requirements: 10.7, 23.1_
  - [ ]* 29.7 Write Phase 5 endpoint smoke test
    - Verify all new REST endpoints return expected status codes
    - _Requirements: 2.1, 7.5, 9.1, 10.6_

- [ ] 30. Final checkpoint — Full test suite passes
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at logical boundaries
- Property tests validate universal correctness properties from the design document (17 properties)
- Unit tests validate specific examples and edge cases
- The design uses Java throughout — no language selection needed
- jqwik 1.9.2 is already configured for property-based testing in both services
- Spring WebSocket starter and Spring Data Redis are already present in both services

### Gap Coverage Summary

The following gaps from the previous tasks.md have been addressed:
1. **Shared retry utility** (task 7.1) — explicit `RetryWithBackoff` component used by FCM, SendGrid, and Web Push adapters
2. **FCM invalid token removal** (tasks 10.1, 18.2) — explicit handling of `messaging/registration-token-not-registered` and `messaging/invalid-registration-token` in both adapter and processing service
3. **Redis Pub/Sub auto-reconnection** (task 16.1) — explicit Redis connection listener for auto-reconnection and re-subscription
4. **Availability update transformation strategy** (task 16.2) — local cache of slot states per court/date with fetch-on-subscribe, delta-update, and 5-min eviction
5. **Booking status update routing** (task 16.2) — resolve customer and court owner userIds, route to both personal queues
6. **`loadPendingWebSocket` method** (tasks 3.6, 8.2, 8.3, 15.5) — added to `LoadNotificationPort`, repository, adapter, and WebSocket event listener
7. **SendGrid unsubscribe link** (task 10.5) — explicit `unsubscribeLink` in `dynamic_template_data`
8. **SendGrid from address configuration** (task 10.5) — explicit `sendgrid.from-email` and `sendgrid.from-name` usage
9. **Kafka consumer lag monitoring** (tasks 18.2, 18.12, 24.2) — explicit lag detection, WARN logging, and metric increment
10. **Structured logging** (tasks 18.2, 24.2) — explicit INFO-level structured log fields for notification delivery
11. **Internal API security** (tasks 21.5, 21.6, 21.7) — explicit `X-Internal-Api-Key` header validation on all internal endpoints
12. **`GetReminderRulesForCourtQuery` use case** (task 21.7) — explicit use case creation for reminder rules internal API
13. **`NotificationEvent` record/DTO** (task 1.7) — explicit internal record for Kafka consumer deserialization
14. **WebSocket message DTOs** (task 5.7) — explicit DTO/record classes for WebSocket notification, availability, and booking payloads
15. **SendGrid template environment variables** (tasks 7.4, 25.1) — all 16 template ID env vars listed explicitly
16. **Existing BookingTest updates** (task 8.10) — explicit task for updating existing tests when `reminderState` field is added
17. **Platform Service test configuration** (task 21.12) — explicit task for updating `TestPortsConfiguration` and `Phase3PortStubsConfiguration`
18. **Transaction Service IntegrationTestConfiguration** (task 26.5) — explicit task for adding NoOp/mock beans for all new ports
19. **WebPushConfig for VAPID keys** (task 10.9) — explicit `WebPushConfig` class for VAPID key initialization
20. **Explicit NotificationEvent record** (task 1.7) — normalized internal DTO for both publisher formats

### Review Fixes Applied (v2)

The following issues identified during code review have been fixed:
1. **BOOKING_MODIFIED field name mismatch is a real bug** (task 21.10) — Corrected description: the Platform Service consumer reads `oldCourtId`/`oldDate` but Transaction Service serializes `previousDate`/`previousStartTime`. This is a real bug causing broken cache invalidation for the old date, not a cosmetic cleanup. Also noted the missing `previousEndTime` field in `BookingModifiedEvent`
2. **`Booking.withId()` breaking change** (task 8.8) — Added explicit callout that adding `reminderState` to the private constructor and `withId()` factory is a breaking change affecting `BookingPersistenceMapper`, `IntegrationTestConfiguration`, and all test code
3. **Missing `LoadBookingPort` query for `PendingConfirmationReminderJob`** (task 22.5) — Added explicit sub-task to create `loadPendingForReminder()` method on `LoadBookingPort`, `BookingRepository`, and `BookingPersistenceAdapter`
4. **Missing `SaveBookingPort` dependency in `PendingConfirmationReminderJob`** (task 22.5) — Added `SaveBookingPort` as an explicit dependency (the existing skeleton only has `LoadBookingPort`, `CourtSummaryPort`, `NotificationEventPublisherPort`)
5. **`BookingCompletionJob` task 22.6 was a no-op** — Changed to verification-only step since `BookingCompletionJob` already publishes `BOOKING_COMPLETED` events (implemented in Phase 4)
6. **`CourtSummaryPort` dependency for `BookingReminderSchedulingService`** (task 22.4) — Added explicit dependency list including `CourtSummaryPort` for timezone access
7. **`UserBasicPort` dependency for `NotificationProcessingService`** (task 18.2) — Added explicit dependency list including `UserBasicPort` for user language resolution
8. **Internal API key validation filter** (task 21.5) — Added explicit instruction to create a shared `InternalApiKeyFilter` since existing internal controllers don't validate any API key
9. **Opt-out property test clarification** (task 18.5) — Clarified that opt-out is represented by absence of preference entry, not all channels false, with explicit test cases
10. **Redis bean conflict** (task 16.1) — Added explicit warning about duplicate `RedisMessageListenerContainer` bean and recommended approach (consolidate in `RedisPubSubConfig`, remove from `RedisConfig`)
11. **Missing `previousEndTime` in `BookingModifiedEvent`** (task 21.10) — Noted that the record lacks `previousEndTime` and the consumer must either compute it from duration or the record must be extended
12. **Platform Service `application.yml` changes** (task 25.3) — Added new task for Platform Service configuration (INTERNAL_API_KEY env var)
13. **Availability cache circular dependency risk** (task 16.2) — Added explicit note about Transaction Service → Platform Service call path with async fetch, timeout, and circuit breaker mitigation
14. **DND check logic location** (tasks 3.9, 18.2) — Added explicit instruction that `isInDndWindow()` must be implemented on the Transaction Service's `UserNotificationPreferences.DndConfiguration` record, not depending on Platform Service's domain model
15. **Kafka consumer deserializer mismatch** (task 18.12) — Added explicit instruction to override `value-deserializer` to `StringDeserializer` in the `@KafkaListener` annotation since the global config uses `JsonDeserializer`
