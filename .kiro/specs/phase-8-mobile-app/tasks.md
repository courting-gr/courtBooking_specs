# Implementation Plan: Phase 8 — Mobile App (Flutter)

## Overview

Implements `court-booking-mobile-app`: a single Flutter codebase targeting iOS, Android, and Web that is a **pure client** of the existing Platform Service (REST) and Transaction Service (REST + STOMP-over-WebSocket). Tasks follow **feature-first Clean Architecture** with three layers per feature (presentation → domain → data), **BLoC/Cubit** (`flutter_bloc`) for state, **GetIt + Injectable** for DI, **fpdart `Either<Failure, T>`** for error handling, **Hive** for local storage/queues, a **generated `dart-dio` OpenAPI client** plus a typed **STOMP/WebSocket** client (Decision 2 — no in-app mocking), and a self-contained **Material 3 design system** (Decision 1 — Path A). The plan builds the core (codegen, network, models, caching, real-time) first, then the cross-cutting design system / motion / haptics / gestures / assets / routing / onboarding, then the customer and owner features, then resilience, security, observability, accessibility, perceived performance, and finally **integration/E2E against the real local backend** (Decision 3). The 30 Correctness Properties are implemented as property-based tests with **glados/fast_check** (≥100 iterations), each tagged `Feature: phase-8-mobile-app, Property {n}: {property text}`. All code targets Dart/Flutter.

> Property-based tests, unit tests, widget/golden tests, and integration tests are marked optional with `*` and are complementary to the core implementation tasks. The platform-native configuration, motion, haptics, gesture, asset, onboarding, perceived-performance, settings, and local-runbook tasks are **core implementation** and are not optional.

## Tasks

- [ ] 1. Project foundation, configuration, and dependency injection
  - [ ] 1.1 Scaffold the Flutter app and feature-first folder structure
    - Create the `court-booking-mobile-app` Flutter project (iOS/Android/Web targets) with the `lib/` layout from the design (`app/`, `core/`, `features/`, `shared/`)
    - Add dependencies: `flutter_bloc`, `bloc`, `freezed`, `get_it`, `injectable`, `fpdart`, `hive`/`hive_flutter`, `hydrated_bloc`, `dio`, `go_router`, `connectivity_plus`, `flutter_secure_storage`, `local_auth`, `flutter_stripe`, `firebase_messaging`, `google_maps_flutter`(+web), `cached_network_image`, `flutter_svg`, `flutter_animate`, `animations`, `intl`, `shimmer`; dev: `build_runner`, `mocktail`, `bloc_test`, `glados`/`fast_check`, `integration_test`
    - Configure the strict analyzer/lint rules and barrel-export convention per the steering baseline
    - _Requirements: 28.1, 27.1_
  - [ ] 1.2 Implement `AppConfig` build-time configuration reader
    - Read all config exclusively via `--dart-define` (platform/transaction base URLs, WS base URL, Stripe publishable key, Maps key, OAuth client ids); no secrets baked in
    - _Requirements: 25.1, 28.1_
  - [ ] 1.3 Wire GetIt + Injectable DI container and register module
    - Create `core/di/injection.dart` (`configureDependencies()` → `getIt.init()`) and `register_module.dart` providing `Dio`, `SharedPreferences`, Hive boxes, `flutter_secure_storage`, STOMP client, and SDK wrappers
    - _Requirements: 27.1_

- [ ] 2. Platform-native configuration
  - [ ] 2.1 Configure debug-only cleartext (Android) and ATS exception (iOS)
    - Add Android `network_security_config.xml` permitting cleartext to `10.0.2.2`/LAN IP in **DEBUG** builds only and reference it from the debug `AndroidManifest`; add an iOS local **ATS exception** for the dev host in **DEBUG** only; release builds use HTTPS with the exceptions absent
    - _Requirements: 25.2, 28.1_
  - [ ] 2.2 Wire Firebase/FCM (Android) and APNs (iOS) native setup
    - Add `google-services.json` + the Google Services Gradle plugin (Android) and `GoogleService-Info.plist` + APNs **sandbox** entitlement (iOS), and the background/notification capabilities required by `firebase_messaging`
    - _Requirements: 10.1, 10.3, 28.2, 28.3, 28.5_
  - [ ] 2.3 Wire the Google Maps native key
    - Add the Maps API key as `AndroidManifest` `meta-data` and via the iOS `AppDelegate` `GMSServices` provideAPIKey wiring, sourced from build-time config
    - _Requirements: 4.1, 28.1_
  - [ ] 2.4 Configure flutter_stripe native requirements
    - Set the iOS minimum deployment target required by `flutter_stripe` and the Android `MainActivity`/app theme to a Material components theme; wire Apple/Google Pay merchant config
    - _Requirements: 7.2, 28.2, 28.3_
  - [ ] 2.5 Add and configure splash and launcher icons
    - Add `flutter_native_splash` + `flutter_launcher_icons` to dependencies and configure the splash screen and app icon for all platforms (feeds the cold-start budget)
    - _Requirements: 31.2_

- [ ] 3. Generated OpenAPI client and forward-compatible decoding
  - [ ] 3.1 Generate the `dart-dio` client from both OpenAPI specs
    - Run openapi-generator against `docs/api/openapi-platform-service.yaml` → `core/api/platform/` and `openapi-transaction-service.yaml` → `core/api/transaction/`; output is never hand-edited
    - _Requirements: 27.1_
  - [ ] 3.2 Implement tolerant decoding helpers (`tolerantEnum<T>()`, unknown-field-ignoring deserialization)
    - Configure deserializers to ignore unknown/newer fields; map unrecognized enum strings (`SlotStatus`, `BookingStatus`, `notificationType`) to a safe `unknown`/ignored variant
    - _Requirements: 27.2, 27.4, 10.8, 10.9, 16.4_
  - [ ]* 3.3 Write property test for unknown-field tolerance
    - **Property 15: Deserialization ignores unknown/newer fields**
    - **Validates: Requirements 27.2**
  - [ ]* 3.4 Write property test for tolerant enum mapping
    - **Property 16: Unknown enum values map to a safe variant and never throw**
    - **Validates: Requirements 10.8, 10.9, 27.4**

- [ ] 4. Core data models and Failure hierarchy
  - [ ] 4.1 Implement domain entities and the `Failure` sealed hierarchy
    - Freezed entities: `Money`, `Slot`/`CourtAvailability`, `Booking` (with `BookingStatus`/`SlotStatus` tolerant enums); `Failure` sealed types: `ServerFailure`, `NetworkFailure`, `CacheFailure`, `ValidationFailure(fieldErrors)`, `AuthFailure`, `NotFoundFailure`, `ConflictFailure(alternativeSlots)`, `BusinessRuleFailure(code)`, `RateLimitedFailure(retryAfter)`
    - _Requirements: 15.8, 21.4, 22.2, 26.3_
  - [ ] 4.2 Implement client-owned cache/queue models with Hive `TypeAdapter`s
    - `Cached<T>` (fetchedAt + 24h `isStale`), `CacheBucket`, `QueuedAction`/`QueuedActionType`, `PendingBookingIntent`
    - _Requirements: 14.1, 31.4, 20.3, 16.5_

- [ ] 5. Network layer and Dio interceptor chain
  - [ ] 5.1 Implement Header/Idempotency interceptor
    - Inject `Accept-Language`, `traceparent`/correlation header, and a client-generated UUID v4 `X-Idempotency-Key` on every state-changing (`POST/PUT/DELETE`) booking/payment/cancellation request
    - _Requirements: 16.2, 8.5, 9.1, 21.3_
  - [ ] 5.2 Implement Auth interceptor with single-flight silent refresh
    - Attach Bearer token; on `401` perform exactly one Silent_Token_Refresh, queue concurrent 401s behind it, then transparently retry; proactively refresh within 60s of expiry; route to login (preserving nav state) on refresh failure
    - _Requirements: 18.1, 18.2, 18.3, 18.4, 1.7_
  - [ ] 5.3 Implement Retry/backoff interceptor and hard timeout
    - GET-only auto-retry once after 2s (exponential backoff); non-GET surfaces manual retry; 10s hard timeout on non-payment requests → auto-cancel + "Something went wrong"
    - _Requirements: 15.3, 15.7, 15.9, 15.10_
  - [ ] 5.4 Implement `ErrorInterceptor` mapping response bodies to typed `Failure`s
    - Map `ErrorResponse` `{ error, message, details, timestamp, correlationId }`, `BookingConflictResponse`, and `BookingConstraintError` to the `Failure` hierarchy preserving message/code/details/correlationId
    - _Requirements: 15.8, 2.3, 19.3, 22.2, 26.3_
  - [ ]* 5.5 Write property test for idempotency key generation and reuse
    - **Property 1: Idempotency key is generated for every state-changing request and reused on retry**
    - **Validates: Requirements 8.5, 9.1, 16.2, 16.4**
  - [ ]* 5.6 Write property test for the 401 silent-refresh single-flight behavior
    - **Property 2: 401 triggers a single silent refresh and transparently retries the original requests**
    - **Validates: Requirements 18.1, 18.2**
  - [ ]* 5.7 Write property test for the HTTP status handling matrix
    - **Property 13: HTTP status maps to the correct action and retryability**
    - **Validates: Requirements 15.3, 15.9, 15.10**
  - [ ]* 5.8 Write property test for lossless error-body mapping
    - **Property 14: API error bodies map losslessly to typed errors with the message and code preserved**
    - **Validates: Requirements 2.3, 15.8, 19.3, 21.4, 22.2, 26.3**

- [ ] 6. Connectivity and global network-state resilience
  - [ ] 6.1 Implement `NetworkInfo` + `ConnectivityCubit` and global "No connection" banner driver
    - Drive the persistent offline banner, disable network-dependent actions everywhere, and on restore refresh the current screen and drain the offline queue
    - _Requirements: 15.1, 15.2, 1.9, 2.10, 14.5, 22.7_
  - [ ]* 6.2 Write property test for offline action gating
    - **Property 12: Network-dependent actions are disabled exactly when offline**
    - **Validates: Requirements 1.9, 2.10, 14.5, 22.7**

- [ ] 7. Offline store, caching, and action queue
  - [ ] 7.1 Implement the Hive cache store with TTL, bounded buckets, and oldest-eviction
    - 24h `Cache_TTL`; bounded sets (searches ≤ 3, court details ≤ 10, bookings ≤ 50, profile/preferences/favorites/weather); "Last updated X ago" + "Data may be outdated" staleness
    - _Requirements: 14.1, 14.3, 14.4, 31.4_
  - [ ] 7.2 Implement the offline action queue (FIFO, max 50)
    - Enqueue favorite toggles and preference saves; process in insertion order on reconnect; "Pending sync" indicator; reject beyond 50
    - _Requirements: 20.3, 20.4, 15.2_
  - [ ]* 7.3 Write property test for cache TTL and bucket eviction
    - **Property 9: Cache respects the 24-hour TTL and per-bucket size limits with oldest-eviction**
    - **Validates: Requirements 14.1, 14.4, 31.4**
  - [ ]* 7.4 Write property test for the bounded FIFO offline queue
    - **Property 11: The offline action queue is bounded at 50 and processed in FIFO order**
    - **Validates: Requirements 20.3, 20.4, 15.2**

- [ ] 8. Real-time STOMP/WebSocket client
  - [ ] 8.1 Implement STOMP connection, subscriptions, and token lifecycle
    - Connect to `wss://<host>/ws?token=<jwt>`; subscribe to `/topic/courts/{courtId}/availability`, `/user/queue/bookings`, `/user/queue/notifications`, `/user/queue/system`; on `TOKEN_EXPIRING` send `TOKEN_REFRESH` on `/app/token-refresh` within 60s; on close `4001` refresh+reconnect (fallback to login); foreground/background lifecycle
    - _Requirements: 13.1, 18.5, 13.9, 13.10, 18.6, 31.5_
  - [ ] 8.2 Implement reconnection backoff with jitter and degradation modes
    - Backoff `1s,2s,4s,8s,16s,max 30s` + ±500ms jitter; "Live updates paused" on drop; poll `GET /availability` every 10s after >60s down; "Availability may have changed" when stale (>30s/disconnected) with refresh on "Book Now"; `SERVER_SHUTDOWN`/`4005` reconnect after `reconnectAfterMs`; full refresh + clear indicator on reconnect
    - _Requirements: 13.3, 13.4, 13.5, 13.6, 13.7, 13.8_
  - [ ] 8.3 Implement availability full-replace and booking-status patch handlers
    - On `AVAILABILITY_UPDATE`/`AVAILABILITY_SNAPSHOT` fully replace local slot state for affected dates (non-incremental); on `BOOKING_STATUS_UPDATE` patch the matching booking's `status`/`noShow`/`paymentStatus`/`refundAmount` in place
    - _Requirements: 13.2, 13.5, 8.8_
  - [ ] 8.4 Implement outbound rate limiter and ERROR-frame handling, plus shared rate/debounce limiters
    - Keep sends ≤ 10 msg/s; treat `ERROR` frames non-fatal unless `willDisconnect`; provide reusable debounce (map search ≤ 1/500ms) and throttle (availability refresh ≤ 1/5s per court) utilities
    - _Requirements: 13.11, 4.6, 15.12, 31.3_
  - [ ]* 8.5 Write property test for availability full-replace
    - **Property 4: Availability updates fully replace local slot state (non-incremental)**
    - **Validates: Requirements 13.2, 13.5**
  - [ ]* 8.6 Write property test for in-place booking-status patching
    - **Property 5: Booking status updates patch in place without disturbing the list**
    - **Validates: Requirements 8.8**
  - [ ]* 8.7 Write property test for the reconnection backoff schedule and jitter bound
    - **Property 6: Reconnection backoff follows the schedule and stays within the jitter bound**
    - **Validates: Requirements 13.4, 13.8**
  - [ ]* 8.8 Write property test for the outbound send rate cap
    - **Property 7: Outbound WebSocket sends never exceed 10 messages per second**
    - **Validates: Requirements 13.11**
  - [ ]* 8.9 Write property test for ERROR-frame fatality
    - **Property 8: WebSocket ERROR frames are fatal only when willDisconnect is true**
    - **Validates: Requirements 13.11**
  - [ ]* 8.10 Write property test for map-search and availability-refresh rate bounds
    - **Property 26: Map search and availability refresh respect their rate bounds**
    - **Validates: Requirements 4.6, 15.12, 31.3**

- [ ] 9. Checkpoint - core infrastructure
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Design system (Path A) — tokens, theme, widget catalog
  - [ ] 10.1 Implement the token layer
    - Color (`ColorScheme.fromSeed(0xFF1677FF)` + semantic roles + booking-status color map with paired `on*` foregrounds), typography scale, 4pt spacing tokens, shape radii, elevation ramp, motion tokens (durations + curves)
    - _Requirements: 23.3, 23.5, 23.2_
  - [ ] 10.2 Implement `AppTheme` light/dark, `ThemeCubit`/`LocaleCubit` (HydratedCubit), `Breakpoints`, and `MotionConfig`
    - Light/dark `ThemeData` from tokens; persisted theme-mode and locale; total breakpoint selector (phone `<600`, tablet `600–1023`, wide `≥1024`); reduce-motion-aware effective-duration function
    - _Requirements: 23.7, 28.1, 28.4, 21.2_
  - [ ] 10.3 Build the reusable widget catalog
    - Buttons (disabled+spinner + press micro-interaction), status chips (icon+label+color), skeleton/shimmer loaders, banners (`OfflineBanner`, `LiveUpdatesPausedBanner`, `StaleAvailabilityBanner`, `PendingSyncBadge`, `UpdateRequiredScreen`), bottom sheets, `EmptyState`, `SectionError`/`FullScreenError`, `SuccessCheck`, `PlatformAdaptive` wrappers
    - _Requirements: 16.1, 23.5, 15.5, 29.1, 6.5_
  - [ ]* 10.4 Write property test for contrast thresholds
    - **Property 28: Brand color roles meet the accessibility contrast thresholds**
    - **Validates: Requirements 23.3**
  - [ ]* 10.5 Write property test for reduce-motion handling
    - **Property 29: Reduce-motion disables non-essential animation**
    - **Validates: Requirements 23.7**
  - [ ]* 10.6 Write property test for responsive breakpoint selection
    - **Property 30: Responsive layout selection is a total, monotonic function of width**
    - **Validates: Requirements 28.1, 28.4**
  - [ ]* 10.7 Write property test for zero-result empty-state rendering
    - **Property 27: Zero-result responses always render an actionable empty state**
    - **Validates: Requirements 29.1, 29.2, 29.3, 29.4, 29.5**
  - [ ] 10.8 Write golden/widget tests for the catalog and key screen states
    - Loading/empty/error/loaded across light/dark, phone/tablet widths, default/2× text scale
    - _Requirements: 15.5, 23.2, 29.1_

- [ ] 11. Motion and transitions
  - [ ] 11.1 Implement the motion system driven by motion tokens
    - Page transitions wired into `PageTransitionsTheme` for go_router (Material **shared-axis** for forward/back within a tab, **fade-through** for bottom-nav tab switches, **container-transform** for card→detail expansions); **hero/shared-element** court-card→Court-Detail (image hero + container transform); **staggered fade+slide** list entrance on first paint of search/calendar/notification lists; `AnimatedSwitcher` loading/empty/error/loaded state transitions; live-data **cross-fade** when `AVAILABILITY_UPDATE` replaces slots; all motion routed through `MotionConfig` so non-essential motion collapses when reduce-motion is on (reduce-motion correctness is covered by Property 29 / task 10.5)
    - _Requirements: 23.7_
  - [ ] 11.2 Write widget tests for motion and transitions
    - Assert reduce-motion collapses non-essential animation, hero/container-transform wiring, and AnimatedSwitcher state changes
    - _Requirements: 23.7_

- [ ] 12. Haptic feedback
  - [ ] 12.1 Implement the `Haptics` helper and event→pattern map
    - Wrap `HapticFeedback` with the event→pattern map (primary/confirm tap = `selectionClick`, booking success = `mediumImpact`, decline/validation error = `heavyImpact`, pull-to-refresh trigger = `lightImpact`, swipe reveal = `selectionClick`/commit = `mediumImpact`, favorite toggle = `lightImpact`); no-ops on Web; wire into the shared button and gesture widgets
    - _Requirements: 23.7_
  - [ ] 12.2 Write unit tests for the Haptics map and Web no-op
    - Assert each event maps to the correct pattern and that all calls are no-ops on Web
    - _Requirements: 23.7_

- [ ] 13. Gesture affordances foundation
  - [ ] 13.1 Implement the shared gesture widgets
    - Shared **pull-to-refresh** (`RefreshIndicator` / Cupertino sliver refresh, respecting the Req 15.12 throttle), **swipe-action** row (reveal + required confirm step on destructive actions, never firing on a single accidental swipe), **bottom-sheet drag handle + drag-to-dismiss**, and **image carousel** (page dots, swipe + Web keyboard/pointer arrows) used across list/detail screens
    - _Requirements: 4.5, 8.4, 11.4_
  - [ ] 13.2 Write widget tests for the gesture widgets
    - Pull-to-refresh trigger, swipe reveal + confirm gating, drag-to-dismiss, carousel paging
    - _Requirements: 4.5, 8.4, 11.4_

- [ ] 14. Asset and visual language
  - [ ] 14.1 Implement the asset system and image treatment
    - Single icon family (Material Symbols, rounded) + bespoke `flutter_svg` court-type glyphs (tennis/padel/basketball/football) reused on markers/chips/placeholders; a cohesive flat, brand-tinted illustration set with light/dark variants for empty/onboarding/error states (decorative, `Semantics(excludeSemantics: true)`); court-image treatment via `cached_network_image` against the Spaces CDN (consistent aspect ratio, `radius.md` clip, bottom scrim, court-type placeholder, **progressive fade-in**)
    - _Requirements: 6.6, 31.6_
  - [ ] 14.2 Write golden tests for the visual language
    - Court-image treatment (placeholder→fade-in, scrim) and illustration set across light/dark
    - _Requirements: 6.6, 31.6_

- [ ] 15. Localization and internationalization
  - [ ] 15.1 Implement ARB localizations and locale/format infrastructure
    - el/en ARB strings, device-language detection + default, `Accept-Language` propagation, locale-aware currency/date/number formatters, localized friendly error copy, Localized_Content fallback indicator
    - _Requirements: 21.1, 21.2, 21.3, 21.4, 21.5, 21.6, 21.7, 21.8_
  - [ ]* 15.2 Write property test for Euro currency formatting
    - **Property 19: Euro_Cents render as a correct locale-formatted Euro amount**
    - **Validates: Requirements 21.8**

- [ ] 16. Routing, app shell, and bootstrap
  - [ ] 16.1 Implement `main.dart` bootstrap and root `MultiBlocProvider`
    - Bind, read config, init Hive + `HydratedBloc.storage`, `configureDependencies()`, `runZonedGuarded` + `FlutterError.onError`, mount Auth/Connectivity/Theme/Locale singletons
    - _Requirements: 26.2, 27.1_
  - [ ] 16.2 Implement go_router config, guards, and role-based shells
    - Auth guard, role guard (owner-only), verification/Stripe sub-state gating, customer + owner bottom-nav shells selected from JWT `role`
    - _Requirements: 11.1, 12.10_
  - [ ] 16.3 Implement deep-link routing and navigation state preserve/restore
    - go_router deep links, unauthenticated→login→resolve-original-target, missing-resource fallback, cold-start resolve-after-init, session-expiry state preservation, restore-last-screen on relaunch
    - _Requirements: 17.1, 17.2, 17.3, 17.4, 18.3, 18.7_
  - [ ]* 16.4 Write property test for navigation state round-trip on refresh failure
    - **Property 3: Refresh failure preserves and restores navigation state (round trip)**
    - **Validates: Requirements 1.7, 18.3, 17.2**

- [ ] 17. Onboarding and in-context permission priming
  - [ ] 17.1 Implement the first-run onboarding and permission-priming screens
    - 3-screen illustrated first-run flow (discover → book → manage) shown once, **skippable**, using the cohesive illustration set and shared-axis transitions, **reduce-motion aware**; friendly in-context **permission-priming** screens that precede each OS permission prompt (location, notifications, biometrics, calendar, camera/photos) explaining why the permission is needed before the system dialog
    - _Requirements: 24.1_
  - [ ] 17.2 Write widget tests for onboarding and permission priming
    - First-run shown once + skippable, reduce-motion collapse, priming precedes each OS prompt
    - _Requirements: 24.1_

- [ ] 18. Checkpoint - cross-cutting foundation
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 19. Authentication feature
  - [ ] 19.1 Implement auth domain/data layer
    - Repositories/use cases for `GET /api/auth/providers`, `POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/auth/logout`, `POST /api/auth/providers/link` returning `Either<Failure, T>`
    - _Requirements: 1.1, 1.2, 1.11, 2.4, 2.5_
  - [ ] 19.2 Implement secure token storage and biometric unlock
    - Refresh_Token only in Secure_Enclave via `flutter_secure_storage`; access token in memory; biometric gate via `local_auth` (3 failures → OAuth); enrollment-unavailable fallback
    - _Requirements: 1.5, 1.6, 24.4, 25.1_
  - [ ] 19.3 Implement `AuthBloc` and login/onboarding screens
    - OAuth provider buttons, role selection, Terms acceptance (cached fallback when unreachable), offline banner disabling OAuth, session-expired redirect message
    - _Requirements: 1.1, 1.3, 1.4, 1.7, 1.8, 1.9, 1.10_
  - [ ] 19.4 Write unit/widget tests for auth flows
    - bloc_test for AuthBloc state sequences; widget tests for provider load, biometric, role/terms, offline states
    - _Requirements: 1.6, 1.8, 1.9, 1.10_

- [ ] 20. Account and profile management
  - [ ] 20.1 Implement profile repository, `AccountBloc`, and Account screen
    - `GET/PUT /api/users/me` (field errors retained on 400), linked providers, logout, Account_Deletion confirm → `POST /api/users/me/delete` (202 logout / 409 blocking reason), offline read-only, export→Support Hub
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 2.10_
  - [ ] 20.2 Write unit/widget tests for account/profile
    - _Requirements: 2.3, 2.7, 2.8, 2.10_

- [ ] 21. Preferences, skill levels, and optimistic sync
  - [ ] 21.1 Implement preferences/skill repositories, cubits, and screens
    - `GET/PUT /api/users/me/preferences`, `GET/PUT /api/users/me/skill-levels`, language change via `PUT /api/users/me`; apply locally immediately, queue sync offline, "Pending sync" + retry on launch
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 20.2, 21.2_
  - [ ]* 21.2 Write property test for optimistic update keep/restore
    - **Property 10: Optimistic updates keep the new state on success and restore the original on failure**
    - **Validates: Requirements 3.2, 5.4, 5.5, 5.6, 20.1, 20.2**
  - [ ] 21.3 Write widget tests for preferences optimistic/pending-sync states
    - _Requirements: 3.8, 20.4_

- [ ] 22. Checkpoint - auth and account
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 23. Map view
  - [ ] 23.1 Implement map repository/datasource and `MapBloc`
    - `GET /api/courts/map` (bounds + required `date` param) debounced ≤ 1/500ms, distance/type/locationType filters, favorites highlight
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.6_
  - [ ] 23.2 Implement the Map screen
    - Marker clustering, court preview bottom sheet, location-permission fallback (Athens + banner + manual search), couldn't-load retry, tile-fail list fallback, empty state
    - _Requirements: 4.5, 4.7, 4.8, 4.9, 4.10, 4.11, 24.2_
  - [ ] 23.3 Write widget tests for map states and fallbacks
    - _Requirements: 4.8, 4.9, 4.10, 4.11_

- [ ] 24. Discovery, search, favorites, and weather
  - [ ] 24.1 Implement discovery repository (search, favorites, weather) and `DiscoveryBloc`
    - `GET /api/courts`, `GET /api/weather` (>7d + >3s/fail handling), favorites optimistic add/remove via `PUT/DELETE /api/users/me/favorites/{courtId}` with revert, `GET /api/users/me/favorites`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8_
  - [ ] 24.2 Implement Discovery/search screen and weather section
    - Search results, weather placeholder/unavailable, offline date/time selection, empty state
    - _Requirements: 5.9, 5.10, 29.1_
  - [ ] 24.3 Write widget tests for discovery/favorites/weather states
    - _Requirements: 5.6, 5.8, 5.10_

- [ ] 25. Court detail
  - [ ] 25.1 Implement court detail repository with independent per-section loading
    - `GET /api/courts/{id}`, `/availability`, `/cancellation-policy`, `GET /api/weather` as independent sections so one failure doesn't block others
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_
  - [ ] 25.2 Implement the Court Detail screen
    - Hero/container-transform entry, image carousel with placeholder + fade-in + on-scroll retry, slot indicators, outdoor weather, Get_Directions, favorite toggle
    - _Requirements: 6.1, 6.2, 6.3, 6.6, 6.7, 6.8_
  - [ ] 25.3 Write widget tests for court detail per-section load/error
    - _Requirements: 6.5, 6.6_

- [ ] 26. Forms and client-side validation
  - [ ] 26.1 Implement pure validators and the Form Cubit pattern
    - `validate*` for future date/time, duration matches court config, people ≤ capacity, promo alphanumeric 3–20; reactive `errorText`/`isValid`; no API call while invalid; server remains authoritative
    - _Requirements: 19.1, 19.2, 19.3_
  - [ ]* 26.2 Write property test for client-side validation correctness
    - **Property 17: Client-side validation accepts an input if and only if it satisfies the rules**
    - **Validates: Requirements 19.1, 19.2**

- [ ] 27. Checkpoint - discovery surfaces
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 28. Booking flow and payments
  - [ ] 28.1 Implement payments and booking-create repositories
    - `GET /api/payments/methods`, `POST /api/payments/setup-intent` + Stripe SDK, `POST /api/bookings` with persisted `X-Idempotency-Key`; Apple/Google Pay availability
    - _Requirements: 7.2, 7.3, 7.4, 28.2, 28.3_
  - [ ] 28.2 Implement the booking flow screens
    - Confirmation summary (capacity-bounded people, total, policy, confirmation-mode indicator), payment sheet, 3DS (5-min timeout), instant `201` (success check + Add-to-Calendar/Share/Receipt) and manual "pending — awaiting owner confirmation", decline/`402`, `409` alternatives, `422` business-rule message
    - _Requirements: 7.1, 7.5, 7.6, 7.7, 7.8, 7.9, 7.10_
  - [ ] 28.3 Implement payment resilience and crash/kill recovery
    - Disable "Confirm and Pay" + spinner (re-enable only on explicit failure); persist `PendingBookingIntent`; post-10s "payment is being processed" poll `GET /api/bookings` every 5s up to 60s; on relaunch reconcile by reissuing with same key or checking recent bookings + "incomplete booking" banner; payment-succeeded-booking-failed messaging
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5, 16.6, 16.7_
  - [ ] 28.4 Implement the refund-preview calculator
    - Client-side preview from `GET /api/courts/{courtId}/cancellation-policy` + time-to-booking (UX optimization only)
    - _Requirements: 8.4_
  - [ ]* 28.5 Write property test for refund-preview bounds and monotonicity
    - **Property 18: Refund preview is a bounded, monotonic function of the cancellation policy and time-to-booking**
    - **Validates: Requirements 8.4**
  - [ ]* 28.6 Write property test for 409 alternative-slot presentation
    - **Property 21: A 409 conflict presents exactly the alternative slots returned by the server**
    - **Validates: Requirements 7.10, 16.7**
  - [ ]* 28.7 Write property test for single in-flight payment submission
    - **Property 22: The "Confirm and Pay" control allows at most one in-flight submission**
    - **Validates: Requirements 16.1**
  - [ ]* 28.8 Write property test for in-flight payment intent persistence across restart
    - **Property 23: An in-flight payment intent survives a restart and reissues with the same key**
    - **Validates: Requirements 16.5**
  - [ ]* 28.9 Write property test for the post-timeout poll schedule
    - **Property 24: The post-timeout payment poll schedule is bounded**
    - **Validates: Requirements 16.3**
  - [ ] 28.10 Write widget tests for booking-flow result and error states
    - _Requirements: 7.5, 7.6, 7.8, 16.5_

- [ ] 29. My Bookings management
  - [ ] 29.1 Implement My Bookings repository and `BookingBloc`
    - `GET /api/bookings` with supported filters (never sending `status=REJECTED`; REJECTED group derived client-side), grouping by status, `GET /api/bookings/{id}`, cancel `POST /api/bookings/{id}/cancel` (idempotency, actual refund), modify `PUT /api/bookings/{id}/modify` (409 refresh), live `BOOKING_STATUS_UPDATE` patch integration
    - _Requirements: 8.1, 8.2, 8.3, 8.5, 8.6, 8.7, 8.8_
  - [ ] 29.2 Implement My Bookings screens
    - Grouped lists with pull-to-refresh + swipe actions, refund preview before cancel, optimistic "Cancelling…", offline read-only ("Last updated X minutes ago"), per-group empty states
    - _Requirements: 8.4, 8.9, 8.10, 29.2_
  - [ ] 29.3 Write widget tests for My Bookings states and WS patching
    - _Requirements: 8.8, 8.9, 8.10_

- [ ] 30. Recurring bookings
  - [ ] 30.1 Implement recurring booking repository and bloc
    - `POST /api/bookings/recurring` (idempotency), `201` group + instances, `207 Multi-Status` partition with reasons, `GET /api/bookings/recurring/{groupId}`, single-instance cancel, offline read-only
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_
  - [ ] 30.2 Implement recurring booking screens
    - Recurrence config, 207 success/conflict breakdown ("Confirm partial booking"/"Cancel all"), instance list and cancel
    - _Requirements: 9.2, 9.3, 9.4, 9.5_
  - [ ]* 30.3 Write property test for 207 Multi-Status date partitioning
    - **Property 20: A 207 Multi-Status recurring result partitions every requested date exactly once**
    - **Validates: Requirements 9.3, 16.8**
  - [ ] 30.4 Write widget tests for recurring 207 breakdown
    - _Requirements: 9.3, 9.6_

- [ ] 31. Checkpoint - booking and payments
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 32. Notifications and deep linking
  - [ ] 32.1 Implement notifications repository, device registration, and delivery
    - FCM/APNs token via `firebase_messaging` → `POST /api/notifications/device`; WS `NOTIFICATION` in-app; background push + local booking-cache update; `GET /api/notifications`, `POST .../read`, `/read-all`; render supported `notificationType`s, ignore unknown/Phase 2 types
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.7, 10.8, 10.9, 24.3, 28.5_
  - [ ] 32.2 Implement the Notification Center screen and deep-link tap handling
    - Read/unread list, empty state, `data.deepLink` navigation via go_router
    - _Requirements: 10.6, 10.10, 17.1, 29.3_
  - [ ] 32.3 Write widget tests for notification center and deep-link routing
    - _Requirements: 10.9, 10.10, 17.3_

- [ ] 33. Court owner operations
  - [ ] 33.1 Implement owner dashboard repository and screen
    - `GET /api/courts/owner/me`, `GET /api/dashboard` (today's bookings, pending, revenue, reminder-alert badges linking to bookings)
    - _Requirements: 11.1, 11.2_
  - [ ] 33.2 Implement owner calendar screen and iCal export
    - `GET /api/bookings` status-colored with filters; `GET /api/bookings/calendar/ical` handed to calendar/share sheet
    - _Requirements: 11.3, 11.11, 30.4_
  - [ ] 33.3 Implement pending-queue screen
    - `GET /api/bookings/pending` with countdown + swipe; confirm `POST /api/bookings/{id}/confirm`, reject `POST /api/bookings/{id}/reject`
    - _Requirements: 11.4_
  - [ ] 33.4 Implement manual booking screen
    - `POST /api/bookings/manual` with client-side slot validation; recurring `207` per-date breakdown
    - _Requirements: 11.5, 11.6, 19.1_
  - [ ] 33.5 Implement owner court edit, analytics, and notification settings screens
    - `PUT /api/courts/{id}` inline server validation without data loss; `GET /api/analytics/revenue` + `/usage`; `GET/PUT /api/settings/notification-preferences`; no-show `POST /api/bookings/{id}/no-show`; mark-paid `POST /api/bookings/{id}/mark-paid`
    - _Requirements: 11.7, 11.8, 11.9, 11.10, 11.12_
  - [ ] 33.6 Write widget tests for owner screens
    - _Requirements: 11.4, 11.6, 11.9_

- [ ] 34. Verification and Stripe Connect onboarding
  - [ ] 34.1 Implement verification repository and screen
    - `POST /api/verification`, `GET /api/verification`; not-submitted/pending/approved/rejected states; 409 already-pending; rejection reason + resubmit
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5_
  - [ ] 34.2 Implement Stripe Connect onboarding screen and payouts
    - `GET /api/users/me/stripe-connect`, `/onboard` (hosted-URL handoff + refresh-on-return + resume), `/payouts`, `PUT .../payout-schedule`
    - _Requirements: 12.6, 12.7, 12.8, 12.9_
  - [ ] 34.3 Wire owner sub-state gating
    - Keep customer-booking-dependent owner features in a consistent disabled/informational state aligned with verification/Stripe sub-state
    - _Requirements: 12.10_
  - [ ] 34.4 Write widget tests for verification and onboarding states
    - _Requirements: 12.3, 12.5, 12.8_

- [ ] 35. Checkpoint - owner surfaces
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 36. Support hub and diagnostics
  - [ ] 36.1 Implement the `LogSanitizer`
    - Redact tokens, card data, and PII from any log/diagnostic payload before emission or attachment
    - _Requirements: 25.4, 25.1_
  - [ ] 36.2 Implement Support Hub repository and screens
    - Bundled searchable FAQ; `POST /api/support/tickets` auto-attaching related booking + `correlationId` and optional sanitized diagnostics; `GET /api/support/tickets`, `/{ticketId}`, `/messages`, `/close`; empty state; offline FAQ-only
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5, 22.6, 22.7, 29.5_
  - [ ]* 36.3 Write property test for diagnostic-log sanitization
    - **Property 25: Diagnostic logs exclude all secrets and PII**
    - **Validates: Requirements 25.1, 25.4**
  - [ ] 36.4 Write widget tests for support hub states
    - _Requirements: 22.6, 22.7_

- [ ] 37. Client security hardening
  - [ ] 37.1 Implement transport/storage security and secure-screen behavior
    - TLS-only REST + WebSocket; certificate pinning with backup pin; `FLAG_SECURE`/app-switcher obfuscation on payment/account screens; jailbreak/root warning
    - _Requirements: 25.2, 25.3, 25.5, 25.6, 25.7_
  - [ ]* 37.2 Write tests for secure-screen gating and pinning configuration
    - _Requirements: 25.3, 25.5, 25.7_

- [ ] 38. Client analytics, telemetry, and crash reporting
  - [ ] 38.1 Implement telemetry and crash reporting with opt-out
    - Emit non-PII screen views/actions/timings; capture crash reports on next connectivity; attach `correlationId` when available; settings opt-out; restrict to the designated observability destination only
    - _Requirements: 26.1, 26.2, 26.3, 26.4, 26.5_
  - [ ] 38.2 Write tests for opt-out and PII exclusion
    - _Requirements: 26.4, 26.5_

- [ ] 39. Settings screen
  - [ ] 39.1 Assemble the unified Settings screen
    - Single screen wiring theme mode (via `ThemeCubit`), app language (via `LocaleCubit` + `PUT /api/users/me`), analytics/crash-reporting opt-out toggle (from task 38), and security toggles (e.g., push-enable prompt, biometric unlock) into one cohesive, navigable settings surface
    - _Requirements: 21.2, 25.5, 26.4_
  - [ ] 39.2 Write widget tests for the Settings screen
    - Theme/language switch, analytics opt-out persistence, security toggle states
    - _Requirements: 21.2, 25.5, 26.4_

- [ ] 40. API versioning and forward compatibility
  - [ ] 40.1 Implement the API version check and Update Required screen
    - Read `API_Version_Header`; on unsupported major version show "Update required" and direct to update; keep Phase 2 UI hidden/disabled and never call those endpoints
    - _Requirements: 27.3, 27.5_
  - [ ] 40.2 Write widget tests for the Update Required flow
    - _Requirements: 27.3_

- [ ] 41. Device integrations and permissions
  - [ ] 41.1 Implement Add-to-Calendar, Share, Get-Directions, and contextual permission handling
    - Calendar event creation (permission-gated, share/iCal fallback), native Share_Sheet, maps handoff; in-context permission requests for location/notifications/biometrics/calendar/camera with graceful denial + settings deep link
    - _Requirements: 30.1, 30.2, 30.3, 24.1, 24.5, 24.6, 24.7_
  - [ ] 41.2 Write widget tests for device integrations and permission denial paths
    - _Requirements: 24.5, 24.6, 30.1_

- [ ] 42. Perceived-performance and cold start
  - [ ] 42.1 Implement the perceived-performance strategy
    - Instant cached render of cached screens (court detail, My Bookings, favorites, profile) with a "Last updated X ago" label, then background refresh with **animated deltas**; **skeletons over spinners** for the 300 ms–2 s tier; **sub-300 ms = no indicator** (never flash a loader); **deferred/lazy DI** for non-critical singletons plus the native splash (task 2.5) to meet the 3 s cold-start budget on mid-range devices; display-appropriate court-image sizing + lazy-load of off-screen images
    - _Requirements: 31.2, 31.6, 14.2, 14.3, 15.4, 15.5_
  - [ ] 42.2 Write tests for perceived-performance behaviors
    - Cached-first render with "Last updated X ago", skeleton-vs-spinner selection, sub-300 ms no-indicator, lazy image loading
    - _Requirements: 14.2, 15.4, 15.5_

- [ ] 43. Accessibility wiring
  - [ ] 43.1 Apply accessibility semantics across screens
    - Semantic labels on controls/images/status, font-scale support without truncation, ≥44×44 tap targets, status conveyed by icon/label (not color alone), live-region announcements on load/error, reduce-motion honored
    - _Requirements: 23.1, 23.2, 23.4, 23.5, 23.6, 23.7_
  - [ ]* 43.2 Write accessibility audit widget tests
    - Per-screen `meetsGuideline(textContrastGuideline)`, semantic-label presence, tap-target sizes, reduce-motion collapse
    - _Requirements: 23.1, 23.3, 23.4, 23.6_

- [ ] 44. Checkpoint - resilience, security, accessibility
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 45. Local development runbook and seed data
  - [ ] 45.1 Implement `scripts/seed-local.sh`
    - Seed, via the running APIs, a **verified court owner** with a Stripe Connect **test** account, one or more **courts** with availability windows + pricing, and a **customer** with a saved Stripe **test** payment method, so a fresh `docker compose down -v` can be re-seeded quickly
    - _Requirements: 28.1_
  - [ ] 45.2 Document the per-platform `--dart-define` run guide in the mobile repo README
    - Document the per-target host mapping and run commands (Android emulator `10.0.2.2`, iOS simulator `localhost`, physical device LAN IP, Flutter Web + backend CORS allowed-origin note) and the full local-run order (infra → services on `local` → `stripe listen` → seed → launch)
    - _Requirements: 28.1, 28.4_

- [ ] 46. Integration and E2E against the real local backend
  - [ ] 46.1 Build the integration_test harness booting against the local stack
    - Wire `flutter integration_test` to the real docker-compose + both Spring services on `local`, with Stripe test mode + Stripe CLI webhook forwarding, OpenWeatherMap, FCM/APNs sandbox, OAuth test apps (no test doubles)
    - _Requirements: 28.1_
  - [ ] 46.2 Implement the customer journey E2E
    - OAuth login → map discovery → court detail (live availability over WebSocket) → booking + Stripe test-card (incl. 3DS) → receipt → cancel with real refund
    - _Requirements: 1.2, 4.1, 6.2, 7.4, 7.5, 8.5_
  - [ ] 46.3 Implement the idempotency / never-double-book E2E
    - Issue a booking, force a retry with the same `X-Idempotency-Key`, assert the server returns the same booking and no duplicate (end-to-end Property 1)
    - _Requirements: 16.2, 16.4_
  - [ ]* 46.4 Implement the token-refresh transparency E2E
    - Drive a request past access-token expiry; assert silent refresh + transparent retry with no interruption (end-to-end Property 2)
    - _Requirements: 18.1, 18.2_
  - [ ]* 46.5 Implement the availability full-replace E2E
    - Trigger a server-side booking emitting `AVAILABILITY_UPDATE`; assert client slot state equals the broadcast (end-to-end Property 4)
    - _Requirements: 13.2_
  - [ ] 46.6 Implement the court owner journey E2E
    - Verification submit → Stripe Connect onboarding (test) → manual booking → confirm/reject pending → mark-paid → analytics
    - _Requirements: 12.1, 12.7, 11.5, 11.4, 11.8, 11.10_

- [ ] 47. Final checkpoint - full suite
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional (property, unit, widget/golden, and integration tests) and can be skipped for a faster MVP; core implementation tasks are never optional. The platform-native configuration (task 2), motion/haptics/gestures/assets (tasks 11–14), onboarding (task 17), settings (task 39), perceived-performance (task 42), and local runbook/seed (task 45) are first-class core tasks.
- Each task references specific granular requirements for traceability.
- The 30 Correctness Properties are each implemented by a single property-based test using glados/fast_check at **≥ 100 iterations**, tagged `Feature: phase-8-mobile-app, Property {n}: {property text}`. Properties that are UI-shaped (10, 12, 22, 27, 28, 29, 30) are expressed over the underlying state machine / token model with a thin widget test asserting the binding.
- Per Decision 2, no in-app mocking exists; test doubles (mocktail) appear only in unit/widget tests, and all integration/E2E tasks (46.x) run against the real local backend stack (Decision 3).
- Checkpoints ensure incremental validation at natural integration boundaries.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3", "2.1", "2.5", "3.1", "4.1", "10.1", "15.1"] },
    { "id": 2, "tasks": ["2.2", "3.2", "4.2", "5.1", "6.1", "10.2", "10.3", "16.1"] },
    { "id": 3, "tasks": ["2.3", "2.4", "3.3", "3.4", "5.2", "5.3", "5.4", "6.2", "7.1", "7.2", "10.4", "10.5", "10.6", "10.7", "10.8", "12.1", "14.1", "15.2", "16.2", "26.1"] },
    { "id": 4, "tasks": ["5.5", "5.6", "5.7", "5.8", "7.3", "7.4", "8.1", "11.1", "12.2", "13.1", "16.3", "19.1", "19.2", "26.2", "36.1"] },
    { "id": 5, "tasks": ["8.2", "8.3", "8.4", "11.2", "13.2", "14.2", "16.4", "17.1", "19.3", "37.1", "38.1", "40.1"] },
    { "id": 6, "tasks": ["8.5", "8.6", "8.7", "8.8", "8.9", "8.10", "17.2", "19.4", "20.1", "21.1", "23.1", "24.1", "37.2", "38.2", "39.1", "40.2"] },
    { "id": 7, "tasks": ["20.2", "21.2", "21.3", "23.2", "24.2", "25.1", "28.1", "39.2"] },
    { "id": 8, "tasks": ["23.3", "24.3", "25.2", "28.2", "28.4", "33.1", "33.2", "33.3", "33.4", "33.5"] },
    { "id": 9, "tasks": ["25.3", "28.3", "29.1", "30.1", "32.1", "33.6", "34.1", "34.2", "36.2", "41.1"] },
    { "id": 10, "tasks": ["28.5", "28.6", "28.7", "28.8", "28.9", "29.2", "30.2", "32.2", "34.3", "36.3", "41.2", "42.1", "43.1"] },
    { "id": 11, "tasks": ["28.10", "29.3", "30.3", "30.4", "32.3", "34.4", "36.4", "42.2", "43.2", "45.1", "45.2"] },
    { "id": 12, "tasks": ["46.1"] },
    { "id": 13, "tasks": ["46.2", "46.3", "46.4", "46.5", "46.6"] }
  ]
}
```
