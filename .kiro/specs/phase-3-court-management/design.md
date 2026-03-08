# Design Document — Phase 3: Court Management

## Overview

Phase 3 implements the complete court management subsystem for the Court Booking Platform, building on the authentication and user management foundation established in Phase 2. This phase delivers court CRUD operations, geospatial discovery, availability management, holiday calendars, court owner verification, pricing rules, availability caching, weather integration, and customer personalization features.

### Key Capabilities

1. **Court Lifecycle Management**: Full CRUD operations for courts with image uploads to DigitalOcean Spaces, optimistic locking, and audit logging
2. **Geospatial Discovery**: PostGIS-powered location-based search with radius and bounding box queries
3. **Availability System**: Recurring weekly windows, date-specific overrides, and Redis-cached slot computation
4. **Holiday Calendar**: Greek national holidays with Orthodox Easter calculation, custom holidays, and bulk application
5. **Court Owner Verification**: Document submission workflow with admin review and SLA monitoring
6. **Pricing & Cancellation**: Time-based pricing multipliers and tiered cancellation policies
7. **Weather Integration**: OpenWeatherMap forecasts with caching and booking recommendations
8. **Customer Personalization**: Favorites, preferences, and skill levels
9. **Event-Driven Architecture**: Kafka publishing for court updates and notification requests; consumption of booking events for cache invalidation
10. **Bulk Operations**: Multi-court pricing and availability updates
11. **Smart Reminders & Notifications**: Reminder rule CRUD, notification channel preferences, court owner dashboard

### Architecture Principles

Following the hexagonal architecture (Buckpal pattern) established in Phase 2:
- **Domain Layer**: Pure Java entities and value objects with business logic
- **Application Layer**: Use cases (ports) and services orchestrating domain operations
- **Adapter Layer**: Web controllers, persistence adapters, Kafka producers/consumers, external API clients

## Architecture

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Platform Service                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         Adapter Layer (In)                               │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │   Court      │ │ Availability │ │   Holiday    │ │ Verification │   │   │
│  │  │ Controller   │ │ Controller   │ │ Controller   │ │ Controller   │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │   Weather    │ │  Favorites   │ │ Preferences  │ │   Bulk Ops   │   │   │
│  │  │ Controller   │ │ Controller   │ │ Controller   │ │ Controller   │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │  Dashboard   │ │  Reminder    │ │ Notification │ │   Internal   │   │   │
│  │  │ Controller   │ │  Rules Ctrl  │ │  Prefs Ctrl  │ │ Controller   │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │   │
│  │  │              BookingEventKafkaConsumer                           │  │   │
│  │  └──────────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Application Layer                                 │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │ CourtMgmt    │ │ Availability │ │   Holiday    │ │ Verification │   │   │
│  │  │ Service      │ │ Service      │ │ Service      │ │ Service      │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │   Pricing    │ │   Weather    │ │  Favorites   │ │ CourtSearch  │   │   │
│  │  │  Service     │ │  Service     │ │  Service     │ │  Service     │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │  BulkOps     │ │  Dashboard   │ │  Reminder    │ │ Notification │   │   │
│  │  │  Service     │ │  Service     │ │ Rule Service │ │ Prefs Service│   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                          Domain Layer                                    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │    Court     │ │ Availability │ │   Holiday    │ │ Verification │   │   │
│  │  │   (Entity)   │ │   Window     │ │  Template    │ │   Request    │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │ PricingRule  │ │ Cancellation │ │   Weather    │ │  AuditLog    │   │   │
│  │  │              │ │    Tier      │ │  Forecast    │ │   Entry      │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │ ReminderRule │ │ Notification │ │ CourtDefaults│ │  SkillLevel  │   │   │
│  │  │              │ │  Preference  │ │              │ │              │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │   │
│  │  │              OrthodoxEasterCalculator (Domain Service)           │  │   │
│  │  └──────────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Adapter Layer (Out)                               │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │   Court      │ │ Availability │ │   Holiday    │ │ Verification │   │   │
│  │  │ Persistence  │ │ Persistence  │ │ Persistence  │ │ Persistence  │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │ PricingRule  │ │ Cancellation │ │  Favorites   │ │  SkillLevel  │   │   │
│  │  │ Persistence  │ │ Persistence  │ │ Persistence  │ │ Persistence  │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │ CourtDefault │ │ ReminderRule │ │ Notification │ │   Booking    │   │   │
│  │  │ Persistence  │ │ Persistence  │ │Prefs Persist.│ │  Check Port  │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│  │  │    Redis     │ │   Kafka      │ │DigitalOcean │ │OpenWeatherMap│   │   │
│  │  │   Cache      │ │  Publisher   │ │   Spaces    │ │    Client    │   │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Scheduled Jobs                                    │   │
│  │  ┌──────────────────────────────┐ ┌──────────────────────────────────┐  │   │
│  │  │ RecurringHolidayGeneration   │ │ VerificationSlaReminderJob       │  │   │
│  │  │ Job (@Scheduled, yearly)     │ │ (@Scheduled, hourly check)       │  │   │
│  │  │ Req 11.2: Generates holiday  │ │ Req 17.8: Publishes NOTIFICATION │  │   │
│  │  │ overrides for new year from  │ │ _REQUESTED when pending > 48h    │  │   │
│  │  │ active subscriptions with    │ │ Runs within Platform Service     │  │   │
│  │  │ auto_renew = true            │ │ (not Quartz/Transaction Service) │  │   │
│  │  └──────────────────────────────┘ └──────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        ▼                              ▼                              ▼
┌───────────────┐            ┌───────────────┐            ┌───────────────┐
│  PostgreSQL   │            │     Redis     │            │     Kafka     │
│   (PostGIS)   │            │    Cache      │            │    Broker     │
└───────────────┘            └───────────────┘            └───────────────┘
        │
        ▼
┌───────────────┐            ┌───────────────┐
│  transaction  │            │ DigitalOcean  │
│    schema     │◄───────────│    Spaces     │
│  (read-only)  │            │     (S3)      │
└───────────────┘            └───────────────┘
```


### Kafka Event Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Kafka Topics                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  court-update-events                    notification-events                  │
│  ┌────────────────────┐                ┌────────────────────┐               │
│  │ COURT_UPDATED      │                │ NOTIFICATION_      │               │
│  │ PRICING_UPDATED    │                │ REQUESTED          │               │
│  │ AVAILABILITY_      │                │                    │               │
│  │   UPDATED          │                │ Types:             │               │
│  │ CANCELLATION_      │                │ VERIFICATION_      │               │
│  │   POLICY_UPDATED   │                │   SUBMITTED        │               │
│  │ COURT_DELETED      │                │ VERIFICATION_      │               │
│  │ STRIPE_CONNECT_    │                │   SLA_REMINDER     │               │
│  │   STATUS_CHANGED   │                │ (+ approved,       │               │
│  └────────────────────┘                │   rejected,        │               │
│           │                            │   holiday, booking)│               │
│           ▼                            └────────────────────┘               │
│  ┌────────────────────┐                         │                            │
│  │ Transaction Service│                         ▼                            │
│  │ (Phase 4 consumer) │                ┌────────────────────┐               │
│  └────────────────────┘                │ Notification       │               │
│                                        │ Processor (Phase 5)│               │
│  booking-events (consumed)             └────────────────────┘               │
│  ┌────────────────────┐                                                     │
│  │ BOOKING_CREATED    │───────────────► Platform Service                    │
│  │ BOOKING_CONFIRMED  │                 (cache invalidation)                │
│  │ BOOKING_CANCELLED  │                                                     │
│  │ BOOKING_MODIFIED   │                                                     │
│  └────────────────────┘                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scheduled Jobs

| Job | Schedule | Description | Requirement |
|-----|----------|-------------|-------------|
| `RecurringHolidayGenerationJob` | Annually (configurable cron, e.g., `0 0 2 1 1 *` — Jan 1 at 02:00) | Generates `availability_overrides` for the new year from active `court_holiday_subscriptions` with `auto_renew = true`. Publishes `NOTIFICATION_REQUESTED` events to notify court owners with a summary. | Req 11.2, 11.3 |
| `VerificationSlaReminderJob` | Hourly (e.g., `0 0 * * * *`) | Checks for `verification_requests` with status `PENDING_REVIEW` and `submitted_at` older than 48 hours. Publishes `NOTIFICATION_REQUESTED` events to notify `PLATFORM_ADMIN` users. | Req 17.8 |

Both jobs use Spring `@Scheduled` annotations within Platform Service (not Quartz, which is in Transaction Service).

## Components and Interfaces

### Incoming Ports (Use Cases)

#### Court Management

```java
// Court CRUD operations
public interface CreateCourtUseCase {
    CourtId createCourt(CreateCourtCommand command);
}

public interface UpdateCourtUseCase {
    Court updateCourt(UpdateCourtCommand command);
}

public interface DeleteCourtUseCase {
    void deleteCourt(DeleteCourtCommand command);
}

public interface GetCourtQuery {
    Optional<Court> getCourtById(CourtId courtId);
    Optional<Court> getCourtByIdForOwner(CourtId courtId, UserId ownerId);
    List<Court> getCourtsByOwner(UserId ownerId);
}

// Court image management
public interface ManageCourtImagesUseCase {
    List<String> uploadImages(UploadImagesCommand command);
    void deleteImage(DeleteImageCommand command);
    void reorderImages(ReorderImagesCommand command);
}

// Court visibility
public interface UpdateCourtVisibilityUseCase {
    Court updateVisibility(UpdateVisibilityCommand command);
}

// Confirmation mode (Req 2.9)
public interface UpdateConfirmationModeUseCase {
    Court updateConfirmationMode(UpdateConfirmationModeCommand command);
}

// Waitlist config (Req 2.10)
public interface UpdateWaitlistConfigUseCase {
    Court updateWaitlistConfig(UpdateWaitlistConfigCommand command);
}
```

#### Court Search & Discovery

```java
public interface SearchCourtsUseCase {
    CourtSearchResult searchCourts(SearchCourtsQuery query);
}

public interface GetMapDataUseCase {
    MapDataResult getMapData(MapDataQuery query);
}

public interface GetCourtDetailUseCase {
    CourtDetailResult getCourtDetail(CourtDetailQuery query);
}
```

#### Availability Management

```java
public interface ManageAvailabilityWindowsUseCase {
    List<AvailabilityWindow> getWindows(CourtId courtId);
    List<AvailabilityWindow> updateWindows(UpdateWindowsCommand command);
}

public interface ManageAvailabilityOverridesUseCase {
    AvailabilityOverride createOverride(CreateOverrideCommand command);
    List<AvailabilityOverride> getOverrides(GetOverridesQuery query);
    void deleteOverride(DeleteOverrideCommand command);
}

public interface GetAvailabilityUseCase {
    AvailabilityResult getAvailability(GetAvailabilityQuery query);
}
```

#### Holiday Management

```java
public interface GetNationalHolidaysUseCase {
    List<NationalHoliday> getNationalHolidays();
}

public interface ManageCustomHolidaysUseCase {
    HolidayTemplate createCustomHoliday(CreateCustomHolidayCommand command);
    List<HolidayTemplate> getCustomHolidays(UserId ownerId);
    HolidayTemplate updateCustomHoliday(UpdateCustomHolidayCommand command);
    void deleteCustomHoliday(DeleteCustomHolidayCommand command);
}

public interface ApplyHolidaysUseCase {
    HolidayApplicationResult applyHolidays(ApplyHolidaysCommand command);
}

public interface GetHolidayCalendarUseCase {
    HolidayCalendarResult getCalendar(HolidayCalendarQuery query);
}

// Court-specific holiday management (Req 10.8, 10.9)
public interface ManageCourtHolidaysUseCase {
    List<CourtHolidaySubscription> getCourtHolidays(CourtId courtId, UserId ownerId);
    void removeCourtHoliday(RemoveCourtHolidayCommand command);
}
```

#### Verification Workflow

```java
public interface SubmitVerificationUseCase {
    VerificationRequest submitVerification(SubmitVerificationCommand command);
}

public interface ReviewVerificationUseCase {
    VerificationRequest approveVerification(ApproveVerificationCommand command);
    VerificationRequest rejectVerification(RejectVerificationCommand command);
}

public interface GetVerificationQueueQuery {
    Page<VerificationRequest> getPendingVerifications(Pageable pageable);
}

// Court owner checks own status (Req 17)
public interface GetVerificationStatusUseCase {
    VerificationStatus getVerificationStatus(UserId courtOwnerId);
}

// Document management (Req 17.12)
public interface ManageVerificationDocumentsUseCase {
    void deleteDocument(DeleteVerificationDocumentCommand command);
}
```

#### Pricing & Cancellation

```java
public interface ManagePricingRulesUseCase {
    List<PricingRule> getPricingRules(CourtId courtId);
    List<PricingRule> updatePricingRules(UpdatePricingRulesCommand command);
}

public interface ManageCancellationTiersUseCase {
    List<CancellationTier> getCancellationTiers(CourtId courtId);
    List<CancellationTier> updateCancellationTiers(UpdateCancellationTiersCommand command);
}

public interface CalculatePriceUseCase {
    PriceCalculation calculatePrice(CalculatePriceQuery query);
}
```

#### Weather Integration

```java
public interface GetWeatherForecastUseCase {
    WeatherForecastResult getWeatherForecast(WeatherForecastQuery query);
}
```

#### Customer Personalization

```java
public interface ManageFavoritesUseCase {
    void addFavorite(UserId userId, CourtId courtId);
    void removeFavorite(UserId userId, CourtId courtId);
    List<FavoriteCourt> getFavorites(GetFavoritesQuery query);
}

public interface ManagePreferencesUseCase {
    UserPreferences getPreferences(UserId userId);
    UserPreferences updatePreferences(UpdatePreferencesCommand command);
}

public interface ManageSkillLevelsUseCase {
    List<SkillLevel> getSkillLevels(UserId userId);
    List<SkillLevel> updateSkillLevels(UpdateSkillLevelsCommand command);
}
```

#### Bulk Operations (Req 35)

```java
public interface BulkCourtOperationsUseCase {
    BulkOperationResult bulkUpdatePricing(BulkPricingCommand command);
    BulkOperationResult bulkUpdateAvailabilityWindows(BulkAvailabilityCommand command);
}
```

#### Notification Preferences (Req 37)

```java
public interface ManageNotificationPreferencesUseCase {
    List<NotificationPreference> getPreferences(UserId courtOwnerId);
    List<NotificationPreference> updatePreferences(UpdateNotificationPreferencesCommand command);
}
```

#### Reminder Rules (Req 38)

```java
public interface ManageReminderRulesUseCase {
    ReminderRule createRule(CreateReminderRuleCommand command);
    List<ReminderRule> getRules(GetReminderRulesQuery query);
    ReminderRule updateRule(UpdateReminderRuleCommand command);
    void deleteRule(DeleteReminderRuleCommand command);
    ReminderRule setEnabled(SetReminderRuleEnabledCommand command);
}
```

#### Dashboard (Req 40)

```java
public interface GetDashboardUseCase {
    DashboardResult getDashboard(UserId courtOwnerId);
}
```

#### Internal / Service-to-Service (Req 31, 39.4)

```java
public interface GetStripeConnectStatusUseCase {
    StripeConnectStatusResult getStripeConnectStatus(UserId userId);
}
```


### Outgoing Ports (SPIs)

```java
// ── Persistence Ports ──

public interface LoadCourtPort {
    Optional<Court> loadById(CourtId id);
    Optional<Court> loadByIdWithOwnerCheck(CourtId id, UserId ownerId);
    List<Court> loadByOwnerId(UserId ownerId);
    List<Court> searchByLocation(GeoSearchCriteria criteria);
}

public interface SaveCourtPort {
    CourtId save(Court court);
    void delete(CourtId courtId);
}

public interface LoadAvailabilityPort {
    List<AvailabilityWindow> loadWindowsByCourtId(CourtId courtId);
    List<AvailabilityOverride> loadOverridesByCourtIdAndDateRange(CourtId courtId, LocalDate from, LocalDate to);
}

public interface SaveAvailabilityPort {
    void saveWindows(CourtId courtId, List<AvailabilityWindow> windows);
    AvailabilityOverride saveOverride(AvailabilityOverride override);
    void deleteOverride(UUID overrideId);
}

// Holiday persistence (holiday_templates + court_holiday_subscriptions)
public interface HolidayPersistencePort {
    List<HolidayTemplate> loadNationalHolidays();
    List<HolidayTemplate> loadCustomHolidaysByOwner(UserId ownerId);
    Optional<HolidayTemplate> loadById(UUID holidayId);
    HolidayTemplate save(HolidayTemplate template);
    void delete(UUID holidayId);
    List<CourtHolidaySubscription> loadSubscriptionsByCourtId(CourtId courtId);
    List<CourtHolidaySubscription> loadActiveAutoRenewSubscriptions();
    CourtHolidaySubscription saveSubscription(CourtHolidaySubscription subscription);
    void deleteSubscription(UUID subscriptionId);
}

// Verification persistence
public interface VerificationPersistencePort {
    VerificationRequest save(VerificationRequest request);
    Optional<VerificationRequest> loadById(UUID requestId);
    Optional<VerificationRequest> loadPendingByOwnerId(UserId ownerId);
    Page<VerificationRequest> loadByStatus(VerificationStatus status, Pageable pageable);
    List<VerificationRequest> loadPendingOlderThan(Instant threshold);
    VerificationStatus loadLatestStatusByOwnerId(UserId ownerId);
}

// Pricing rule persistence
public interface PricingRulePersistencePort {
    List<PricingRule> loadByCourtId(CourtId courtId);
    void replaceAll(CourtId courtId, List<PricingRule> rules);
}

// Cancellation tier persistence
public interface CancellationTierPersistencePort {
    List<CancellationTier> loadByCourtId(CourtId courtId);
    void replaceAll(CourtId courtId, List<CancellationTier> tiers);
}

// Favorites persistence
public interface FavoritesPersistencePort {
    void save(UserId userId, CourtId courtId);
    void delete(UserId userId, CourtId courtId);
    List<FavoriteCourt> loadByUserId(UserId userId);
    boolean exists(UserId userId, CourtId courtId);
}

// Skill level persistence
public interface SkillLevelPersistencePort {
    List<SkillLevel> loadByUserId(UserId userId);
    void upsert(UserId userId, List<SkillLevel> skillLevels);
}

// Court defaults persistence
public interface CourtDefaultsPersistencePort {
    Optional<CourtDefaults> loadByOwnerId(UserId ownerId);
    CourtDefaults upsert(CourtDefaults defaults);
}

// Reminder rule persistence
public interface ReminderRulePersistencePort {
    ReminderRule save(ReminderRule rule);
    Optional<ReminderRule> loadById(UUID ruleId);
    List<ReminderRule> loadByOwnerId(UserId ownerId);
    List<ReminderRule> loadByOwnerIdAndCourtId(UserId ownerId, CourtId courtId);
    void delete(UUID ruleId);
    boolean existsByOwnerAndTypeAndCourt(UserId ownerId, String ruleType, CourtId courtId);
}

// Notification preferences persistence
public interface NotificationPreferencesPersistencePort {
    List<NotificationPreference> loadByOwnerId(UserId ownerId);
    void upsert(UserId ownerId, List<NotificationPreference> preferences);
}

// Cross-schema booking check (reads from transaction.bookings view) — Req 2.2, 3.1, 12.4
public interface BookingCheckPort {
    int countFutureConfirmedBookings(CourtId courtId);
    List<AffectedBooking> findFutureBookingsAffectedByChange(CourtId courtId, List<String> changedFields);
    boolean hasBookingsOnDate(CourtId courtId, LocalDate date);
}

// ── External Service Ports ──

public interface ImageStoragePort {
    List<String> uploadImages(CourtId courtId, List<ImageUpload> images);
    void deleteImage(String imageUrl);
    void deleteAllCourtImages(CourtId courtId);
}

public interface WeatherApiPort {
    Optional<WeatherForecast> getForecast(double latitude, double longitude, LocalDate date, LocalTime time);
}

// ── Caching Ports ──

public interface AvailabilityCachePort {
    Optional<AvailabilityResult> get(CourtId courtId, LocalDate date, int durationMinutes);
    void put(CourtId courtId, LocalDate date, int durationMinutes, AvailabilityResult result);
    void invalidate(CourtId courtId, LocalDate date);
    void invalidateAllForCourt(CourtId courtId);
}

public interface WeatherCachePort {
    Optional<WeatherForecast> get(double latitude, double longitude, LocalDate date, LocalTime time);
    void put(double latitude, double longitude, LocalDate date, LocalTime time, WeatherForecast forecast);
}

// Dashboard cache (Req 40.5) — Redis with 1-minute TTL, key: dashboard:{courtOwnerId}
public interface DashboardCachePort {
    Optional<DashboardResult> get(UserId courtOwnerId);
    void put(UserId courtOwnerId, DashboardResult result);
    void invalidate(UserId courtOwnerId);
}

// ── Event Publishing Ports ──

public interface CourtEventPublisherPort {
    void publishCourtUpdated(CourtUpdatedEvent event);
    void publishPricingUpdated(PricingUpdatedEvent event);
    void publishAvailabilityUpdated(AvailabilityUpdatedEvent event);
    void publishCancellationPolicyUpdated(CancellationPolicyUpdatedEvent event);
    void publishCourtDeleted(CourtDeletedEvent event);
    void publishStripeConnectStatusChanged(StripeConnectStatusChangedEvent event);
}

public interface NotificationEventPublisherPort {
    void publishNotificationRequested(NotificationRequestedEvent event);
}

// ── Audit Logging Port ──

public interface AuditLogPort {
    void log(AuditLogEntry entry);
    Page<AuditLogEntry> getLogsByOwnerId(UserId ownerId, AuditLogQuery query, Pageable pageable);
}
```

### Web Controllers

| Controller | Endpoints | Description |
|------------|-----------|-------------|
| `CourtController` | `POST/PUT/DELETE /api/courts`, `GET /api/courts/{id}`, `PUT /api/courts/{id}/confirmation-mode`, `PUT /api/courts/{id}/waitlist-config` | Court CRUD + config |
| `CourtSearchController` | `GET /api/courts`, `GET /api/courts/map` | Search & discovery |
| `CourtImageController` | `POST/DELETE /api/courts/{id}/images`, `PUT /api/courts/{id}/images/order` | Image management |
| `AvailabilityController` | `GET/PUT /api/courts/{id}/availability/*` | Windows & overrides |
| `HolidayController` | `GET/POST/PUT/DELETE /api/holidays/*`, `GET /api/courts/{id}/holidays`, `DELETE /api/courts/{id}/holidays/{holidayId}` | Holiday management |
| `VerificationController` | `POST /api/verification/submit`, `DELETE /api/verification/documents/{documentId}` | Court owner verification |
| `AdminVerificationController` | `GET/POST /api/admin/verifications/*` | Admin review |
| `PricingController` | `GET/PUT /api/courts/{id}/pricing-rules` | Pricing rules |
| `CancellationController` | `GET/PUT /api/courts/{id}/cancellation-tiers` | Cancellation policy |
| `WeatherController` | `GET /api/weather` | Weather forecasts |
| `FavoritesController` | `GET/POST/DELETE /api/favorites/*` | Customer favorites |
| `PreferencesController` | `GET/PUT /api/preferences` | User preferences |
| `SkillLevelController` | `GET/PUT /api/skill-levels` | Skill levels |
| `CourtDefaultsController` | `GET/PUT /api/court-defaults` | Court owner defaults |
| `AuditLogController` | `GET /api/audit-logs` | Audit log retrieval |
| `DashboardController` | `GET /api/dashboard` | Court owner dashboard |
| `BulkCourtController` | `PUT /api/courts/bulk/pricing`, `PUT /api/courts/bulk/availability-windows` | Bulk operations |
| `NotificationPreferencesController` | `GET/PUT /api/notification-preferences` | Notification channels |
| `ReminderRulesController` | `GET/POST/PUT/DELETE /api/reminder-rules`, `PUT /api/reminder-rules/{id}/enabled` | Reminder rules |
| `InternalCourtController` | `GET /internal/courts/*` | Service-to-service |
| `InternalUserController` | `GET /internal/users/{id}/stripe-connect-status` | Stripe Connect status |


## Data Models

### Domain Entities

#### Court (Rich Domain Entity)

```java
public class Court {
    private final CourtId id;
    private final UserId ownerId;
    private String nameEl;
    private String nameEn;
    private String descriptionEl;
    private String descriptionEn;
    private CourtType courtType;
    private LocationType locationType;
    private GeoLocation location;
    private String address;
    private ZoneId timezone;
    private int basePriceCents;
    private int durationMinutes;
    private int maxCapacity;
    private ConfirmationMode confirmationMode;
    private int confirmationTimeoutHours;
    private boolean waitlistEnabled;
    private Set<Amenity> amenities;
    private List<String> imageUrls;
    private BigDecimal averageRating;
    private int totalReviews;
    private boolean visible;
    private int version;
    private Instant createdAt;
    private Instant updatedAt;

    // Business methods
    public void updateDetails(CourtUpdateDetails details) { ... }
    public void setVisibility(boolean visible, boolean ownerVerified, String stripeConnectStatus) { ... }
    public void addImages(List<String> urls) { ... }
    public void removeImage(String url) { ... }
    public void reorderImages(List<String> orderedUrls) { ... }
    public void validateForDeletion(int futureBookingCount, int pendingPayoutAmount) { ... }
}
```

#### VerificationRequest (Domain Entity — Req 17)

```java
public class VerificationRequest {
    private final UUID id;
    private final UserId courtOwnerId;
    private String businessName;
    private String taxId;
    private BusinessType businessType;       // SOLE_PROPRIETOR, COMPANY, ASSOCIATION
    private String businessAddress;
    private String proofDocumentUrl;
    private VerificationStatus status;       // PENDING_REVIEW, APPROVED, REJECTED
    private UserId reviewedBy;               // PLATFORM_ADMIN who reviewed
    private String reviewNotes;
    private String rejectionReason;
    private UUID previousRequestId;          // Links to rejected request on re-submission (Req 17.6)
    private Instant submittedAt;
    private Instant reviewedAt;

    // Business methods
    public void approve(UserId adminId, String notes) { ... }
    public void reject(UserId adminId, String reason) { ... }
    public boolean isPending() { return status == VerificationStatus.PENDING_REVIEW; }
    public boolean isOverSla(Duration slaThreshold) {
        return isPending() && submittedAt.plus(slaThreshold).isBefore(Instant.now());
    }
}
```

#### AuditLogEntry (Domain Entity — Req 27)

```java
public class AuditLogEntry {
    private final UUID id;
    private final UserId courtOwnerId;
    private final CourtId courtId;           // nullable — not all actions are court-specific
    private final String action;             // COURT_CREATED, COURT_UPDATED, COURT_DELETED, etc.
    private final String entityType;         // COURT, PRICING_RULE, AVAILABILITY_WINDOW, etc.
    private final UUID entityId;
    private final Map<String, Object> changes; // JSONB with before/after snapshots
    private final String ipAddress;
    private final String userAgent;
    private final Instant createdAt;

    // Append-only: no setters, no update methods (Req 27.3)
}
```

#### ReminderRule (Domain Entity — Req 38)

```java
public class ReminderRule {
    private final UUID id;
    private final UserId courtOwnerId;
    private final CourtId courtId;           // nullable — null means applies to all owner's courts
    private RuleType ruleType;               // BOOKING_REMINDER, PENDING_CONFIRMATION, LOW_OCCUPANCY, DAILY_SUMMARY
    private Integer triggerHoursBefore;      // required for BOOKING_REMINDER, PENDING_CONFIRMATION
    private LocalTime triggerTime;           // required for DAILY_SUMMARY
    private Integer thresholdPercent;        // required for LOW_OCCUPANCY
    private Set<NotificationChannel> channels; // PUSH, EMAIL
    private boolean enabled;
    private Instant createdAt;
    private Instant updatedAt;

    // Validation per rule type (Req 38.5)
    public void validate() { ... }
}
```

#### NotificationPreference (Domain Entity — Req 37)

```java
public class NotificationPreference {
    private final UUID id;
    private final UserId courtOwnerId;
    private String eventType;                // VERIFICATION_SUBMITTED, VERIFICATION_APPROVED, etc.
    private boolean pushEnabled;
    private boolean emailEnabled;
    private boolean inAppEnabled;

    // At least one channel must be enabled (Req 37.5)
    public void validate() {
        if (!pushEnabled && !emailEnabled && !inAppEnabled) {
            throw new AtLeastOneChannelRequiredException(eventType);
        }
    }
}
```

#### CourtDefaults (Domain Entity — Req 25)

```java
public class CourtDefaults {
    private final UserId courtOwnerId;
    private Integer defaultDurationMinutes;
    private ConfirmationMode defaultConfirmationMode;
    private Integer defaultConfirmationTimeoutHours;
    private List<CancellationTier> defaultCancellationTiers;
    private Set<Amenity> defaultAmenities;
    private Instant updatedAt;
}
```

### Value Objects

```java
public record CourtId(UUID value) {
    public CourtId {
        requireNonNull(value, "Court ID cannot be null");
    }
}

public record GeoLocation(double latitude, double longitude) {
    public GeoLocation {
        if (latitude < -90 || latitude > 90) {
            throw new IllegalArgumentException("Latitude must be between -90 and 90");
        }
        if (longitude < -180 || longitude > 180) {
            throw new IllegalArgumentException("Longitude must be between -180 and 180");
        }
    }
    
    public double distanceKm(GeoLocation other) {
        // Haversine formula implementation
    }
}

public record AvailabilityWindow(
    UUID id,
    DayOfWeek dayOfWeek,
    LocalTime startTime,
    LocalTime endTime
) {
    public AvailabilityWindow {
        requireNonNull(dayOfWeek);
        requireNonNull(startTime);
        requireNonNull(endTime);
        if (!endTime.isAfter(startTime)) {
            throw new IllegalArgumentException("End time must be after start time");
        }
    }
    
    public boolean overlaps(AvailabilityWindow other) {
        return this.dayOfWeek == other.dayOfWeek &&
               this.startTime.isBefore(other.endTime) &&
               other.startTime.isBefore(this.endTime);
    }
}

public record AvailabilityOverride(
    UUID id,
    CourtId courtId,
    LocalDate date,
    LocalTime startTime,  // null for full-day block
    LocalTime endTime,    // null for full-day block
    String reason,
    OverrideSource source,
    UUID holidayTemplateId,
    Instant createdAt
) {
    public boolean isFullDay() {
        return startTime == null && endTime == null;
    }
}

public record PricingRule(
    UUID id,
    DayOfWeek dayOfWeek,
    LocalTime startTime,
    LocalTime endTime,
    BigDecimal multiplier,
    String label
) {
    public PricingRule {
        if (multiplier.compareTo(new BigDecimal("0.10")) < 0 ||
            multiplier.compareTo(new BigDecimal("5.00")) > 0) {
            throw new IllegalArgumentException("Multiplier must be between 0.10 and 5.00");
        }
    }
    
    public int applyTo(int basePriceCents) {
        return multiplier.multiply(BigDecimal.valueOf(basePriceCents))
                        .setScale(0, RoundingMode.HALF_UP)
                        .intValue();
    }
}

public record CancellationTier(
    UUID id,
    int thresholdHours,
    int refundPercent,
    int sortOrder
) {
    public CancellationTier {
        if (refundPercent < 0 || refundPercent > 100) {
            throw new IllegalArgumentException("Refund percent must be between 0 and 100");
        }
    }
}

public record HolidayTemplate(
    UUID id,
    UserId ownerId,  // null for national holidays
    String nameEl,
    String nameEn,
    DatePattern datePattern,
    boolean isNational,
    boolean isActive,
    Instant createdAt
) {}

public record CourtHolidaySubscription(
    UUID id,
    CourtId courtId,
    UUID holidayTemplateId,
    boolean autoRenew,
    Instant createdAt
) {}

public record DatePattern(String pattern) {
    public DatePattern {
        if (!pattern.matches("^(FIXED:\\d{2}-\\d{2}|EASTER_OFFSET:-?\\d+)$")) {
            throw new IllegalArgumentException("Invalid date pattern: " + pattern);
        }
    }
    
    public LocalDate calculateDate(int year) {
        if (pattern.startsWith("FIXED:")) {
            String[] parts = pattern.substring(6).split("-");
            return LocalDate.of(year, Integer.parseInt(parts[0]), Integer.parseInt(parts[1]));
        } else {
            int offset = Integer.parseInt(pattern.substring(14));
            return OrthodoxEasterCalculator.calculate(year).plusDays(offset);
        }
    }
}

public record SkillLevel(
    UserId userId,
    CourtType courtType,
    int level,       // 1-7
    String label     // Beginner, Advanced Beginner, ..., Professional
) {
    public SkillLevel {
        if (level < 1 || level > 7) {
            throw new IllegalArgumentException("Skill level must be between 1 and 7");
        }
    }
}

public record WeatherForecast(
    double temperature,
    double feelsLike,
    int humidity,
    double windSpeed,
    int precipitationProbability,
    String weatherCondition,
    String weatherIcon,
    WeatherRecommendation recommendation
) {}

public enum WeatherRecommendation {
    OUTDOOR_IDEAL,
    OUTDOOR_OK,
    INDOOR_RECOMMENDED
}
```

### Domain Services

#### OrthodoxEasterCalculator (Req 8.3)

```java
/**
 * Calculates Orthodox Easter Sunday using the Meeus Julian algorithm.
 * Pure domain service — no framework dependencies.
 */
public class OrthodoxEasterCalculator {

    /**
     * Calculates the date of Orthodox Easter Sunday for the given year.
     * Implements the Meeus Julian algorithm for the Julian calendar,
     * then converts to the Gregorian calendar.
     *
     * @param year the year (e.g., 2025)
     * @return the Gregorian date of Orthodox Easter Sunday
     */
    public static LocalDate calculate(int year) {
        int a = year % 4;
        int b = year % 7;
        int c = year % 19;
        int d = (19 * c + 15) % 30;
        int e = (2 * a + 4 * b - d + 34) % 7;
        int month = (d + e + 114) / 31;  // 3 = March, 4 = April
        int day = ((d + e + 114) % 31) + 1;

        // Julian date → Gregorian conversion (add 13 days for 1900-2099)
        LocalDate julianDate = LocalDate.of(year, month, day);
        return julianDate.plusDays(13);
    }
}
```

### Enums

```java
public enum CourtType {
    TENNIS(60, 4),
    PADEL(90, 4),
    BASKETBALL(60, 10),
    FOOTBALL_5X5(60, 10);
    
    private final int defaultDurationMinutes;
    private final int defaultMaxCapacity;
}

public enum LocationType { INDOOR, OUTDOOR }

public enum ConfirmationMode { INSTANT, MANUAL }

public enum Amenity {
    PARKING, SHOWERS, EQUIPMENT_RENTAL, LIGHTING,
    CHANGING_ROOMS, WIFI, CAFE, PRO_SHOP
}

public enum OverrideSource { MANUAL, HOLIDAY_NATIONAL, HOLIDAY_CUSTOM }

public enum VerificationStatus { NOT_SUBMITTED, PENDING_REVIEW, APPROVED, REJECTED }

public enum BusinessType { SOLE_PROPRIETOR, COMPANY, ASSOCIATION }

public enum RuleType { BOOKING_REMINDER, PENDING_CONFIRMATION, LOW_OCCUPANCY, DAILY_SUMMARY }

public enum NotificationChannel { PUSH, EMAIL }

public enum StripeConnectStatus { NOT_STARTED, PENDING, ACTIVE, RESTRICTED, DISABLED }
```


### Kafka Event Schemas

#### Court Update Events (court-update-events topic)

Partition key: `courtId` (except `STRIPE_CONNECT_STATUS_CHANGED` which uses `courtOwnerId`).

```json
{
  "eventId": "UUID",
  "eventType": "COURT_UPDATED | PRICING_UPDATED | AVAILABILITY_UPDATED | CANCELLATION_POLICY_UPDATED | COURT_DELETED | STRIPE_CONNECT_STATUS_CHANGED",
  "timestamp": "ISO8601",
  "courtId": "UUID",
  "version": 1,
  "payload": { }
}
```

Event type payloads:

| Event Type | Payload | Partition Key |
|------------|---------|---------------|
| `COURT_UPDATED` | `{ changedFields: [string], court: { /* full court object */ } }` | `courtId` |
| `PRICING_UPDATED` | `{ courtId, basePriceCents, pricingRules: [...] }` | `courtId` |
| `AVAILABILITY_UPDATED` | `{ courtId, changeType: "RECURRING_WINDOWS_UPDATED" \| "OVERRIDE_CREATED" \| "OVERRIDE_DELETED", affectedDateRange?: { from, to } }` | `courtId` |
| `CANCELLATION_POLICY_UPDATED` | `{ courtId, tiers: [...] }` | `courtId` |
| `COURT_DELETED` | `{ courtId, ownerId, deletedAt }` | `courtId` |
| `STRIPE_CONNECT_STATUS_CHANGED` | `{ userId, previousStatus, newStatus, stripeConnectAccountId?, timestamp }` | `courtOwnerId` |

#### Notification Events (notification-events topic)

Partition key: `userId`.

```json
{
  "eventId": "UUID",
  "eventType": "NOTIFICATION_REQUESTED",
  "timestamp": "ISO8601",
  "payload": {
    "notificationType": "VERIFICATION_SUBMITTED | VERIFICATION_APPROVED | VERIFICATION_REJECTED | VERIFICATION_SLA_REMINDER | HOLIDAY_GENERATION_COMPLETED | BOOKING_CANCELLED_BY_HOLIDAY",
    "recipientUserId": "UUID",
    "title": "string",
    "body": "string",
    "language": "el | en",
    "channels": { "push": true, "email": true, "inApp": true },
    "data": {
      "deepLink": "string",
      "referenceId": "UUID",
      "referenceType": "VERIFICATION | HOLIDAY | BOOKING"
    }
  }
}
```

Notification types:

| Type | Trigger | Recipient | Req |
|------|---------|-----------|-----|
| `VERIFICATION_SUBMITTED` | Court owner submits verification | PLATFORM_ADMIN users | 17.3 |
| `VERIFICATION_APPROVED` | Admin approves verification | Court owner | 17.4 |
| `VERIFICATION_REJECTED` | Admin rejects verification | Court owner | 17.5 |
| `VERIFICATION_SLA_REMINDER` | Pending > 48 hours (scheduled job) | PLATFORM_ADMIN users | 17.8 |
| `HOLIDAY_GENERATION_COMPLETED` | Recurring holiday job generates overrides | Court owner | 11.3 |
| `BOOKING_CANCELLED_BY_HOLIDAY` | Holiday application cancels bookings | Affected customer | 10.11, 12.6 |

#### Booking Events (booking-events topic — consumed)

Expected event payload structure for cache invalidation (Req 28):

```json
{
  "eventId": "UUID",
  "eventType": "BOOKING_CREATED | BOOKING_CONFIRMED | BOOKING_CANCELLED | BOOKING_MODIFIED",
  "timestamp": "ISO8601",
  "courtId": "UUID",
  "date": "ISO8601-date",
  "startTime": "HH:mm",
  "endTime": "HH:mm",
  "bookingId": "UUID"
}
```

For `BOOKING_MODIFIED`, the payload includes both old and new court/date combinations:

```json
{
  "eventType": "BOOKING_MODIFIED",
  "courtId": "UUID",
  "date": "ISO8601-date",
  "startTime": "HH:mm",
  "endTime": "HH:mm",
  "bookingId": "UUID",
  "previousCourtId": "UUID",
  "previousDate": "ISO8601-date"
}
```

Consumer group: `platform-service-booking-consumer`. Config: `auto.offset.reset=latest`.

### Redis Cache Keys

| Cache | Key Pattern | TTL | Description |
|-------|-------------|-----|-------------|
| Availability | `availability:{courtId}:{date}:{durationMinutes}` | 5 min | Computed slots |
| Weather | `weather:{lat_2dp}:{lng_2dp}:{date}:{hour}` | 10 min | Forecast data |
| Dashboard | `dashboard:{courtOwnerId}` | 1 min | Dashboard summary (Req 40.5) |

### Flyway Migrations

| Migration | Description | Requirement |
|-----------|-------------|-------------|
| `V3__seed_national_holidays.sql` | Seeds the 13 Greek national holidays into `holiday_templates` with `is_national = true`, `owner_id = NULL`. Includes both FIXED and EASTER_OFFSET date patterns. | Req 8.6 |
| `V3.1__create_cross_schema_views.sql` | Creates `platform.v_court_summary`, `platform.v_court_cancellation_tiers`, `platform.v_user_skill_level` views and GRANTs SELECT to `transaction_service_role`. | Req 30 |
| `V3.2__create_booking_check_view.sql` | Creates a read-only view or grants SELECT on `transaction.bookings` for cross-schema booking conflict checks. Until Phase 4 deploys bookings, queries return empty results. | Req 2.2, 3.1 |

### Cross-Schema Views

Created via Flyway migration for Transaction Service read-only access:

```sql
-- v_court_summary: Court data for booking validation
CREATE VIEW platform.v_court_summary AS
SELECT id, owner_id, name_el, name_en, court_type, location_type, 
       location, timezone, base_price_cents, duration_minutes, 
       max_capacity, confirmation_mode, confirmation_timeout_hours, 
       waitlist_enabled, visible, version
FROM platform.courts WHERE visible = TRUE;
GRANT SELECT ON platform.v_court_summary TO transaction_service_role;

-- v_court_cancellation_tiers: Cancellation policy for refund calculation
CREATE VIEW platform.v_court_cancellation_tiers AS
SELECT court_id, threshold_hours, refund_percent, sort_order
FROM platform.cancellation_tiers ORDER BY court_id, sort_order;
GRANT SELECT ON platform.v_court_cancellation_tiers TO transaction_service_role;

-- v_user_skill_level: Skill levels for match matching
CREATE VIEW platform.v_user_skill_level AS
SELECT user_id, court_type, level FROM platform.skill_levels;
GRANT SELECT ON platform.v_user_skill_level TO transaction_service_role;
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Court Creation Round-Trip

*For any* valid court creation request with valid coordinates, court type, pricing, and bilingual content, creating the court and then retrieving it by ID should return a court with all the same field values.

**Validates: Requirements 1.1, 1.9**

### Property 2: Court Visibility Determined by Owner Status

*For any* court owner and court, the court's `visible` field should be `true` if and only if the owner's `verified` status is `true` AND the owner's `stripe_connect_status` is `ACTIVE`. Courts created by unverified owners or owners without active Stripe Connect should have `visible = false`.

**Validates: Requirements 1.2, 1.3, 33.5, 39.2**

### Property 3: Court Defaults Application

*For any* court owner with configured defaults and any court creation request that omits fields covered by defaults, the created court should have those fields populated from the owner's defaults.

**Validates: Requirements 1.4, 25.3**

### Property 4: Coordinate Validation

*For any* latitude value outside [-90, 90] or longitude value outside [-180, 180], court creation or search requests should be rejected with a validation error.

**Validates: Requirements 1.5, 13.10**

### Property 5: Positive Integer Field Validation

*For any* `basePriceCents` value that is zero, negative, or non-integer, or `durationMinutes` that is zero or negative, or `maxCapacity` that is zero or negative, court creation should be rejected with a validation error.

**Validates: Requirements 1.6, 1.7, 1.8**

### Property 6: Amenity Enum Validation

*For any* amenity value not in the standard enum set {PARKING, SHOWERS, EQUIPMENT_RENTAL, LIGHTING, CHANGING_ROOMS, WIFI, CAFE, PRO_SHOP}, court creation should be rejected with a validation error listing invalid values.

**Validates: Requirements 1.13**

### Property 7: Bilingual Content Length Validation

*For any* `nameEl` shorter than 3 or longer than 100 characters, or `nameEn` (if provided) shorter than 3 or longer than 100 characters, or descriptions shorter than 10 or longer than 2000 characters, court creation should be rejected with a validation error.

**Validates: Requirements 1.14**

### Property 8: HTML Sanitization Preserves Safe Tags

*For any* description containing HTML, after sanitization the result should contain only safe tags (`<p>`, `<br>`, `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>`) and no other tags or attributes. Unsafe tags like `<script>`, `<iframe>`, `onclick` attributes should be stripped.

**Validates: Requirements 1.15**

### Property 9: Optimistic Locking Rejects Stale Versions

*For any* court update request with a `version` value that does not match the current version in the database, the update should be rejected with a 409 Conflict error containing the current version.

**Validates: Requirements 2.1**

### Property 10: Version Increment on Update

*For any* successful court update, the resulting court's `version` should be exactly the previous version plus one.

**Validates: Requirements 2.5**

### Property 11: Court Deletion Blocked by Future Bookings

*For any* court with future confirmed bookings, deletion should be rejected with a `COURT_HAS_FUTURE_BOOKINGS` error including the booking count and next booking date.

**Validates: Requirements 3.1**

### Property 12: Image Format and Size Validation

*For any* image upload with a format not in {JPEG, PNG, WebP} or size exceeding 10MB, the upload should be rejected with an appropriate validation error.

**Validates: Requirements 4.1**

### Property 13: EXIF Metadata Stripping

*For any* uploaded image containing EXIF data (GPS coordinates, camera info, timestamps), the stored image should have all EXIF metadata removed.

**Validates: Requirements 4.2**

### Property 14: Image Count Limit

*For any* court with N existing images, an upload request that would result in more than 20 total images should be rejected with a MAX_IMAGES_EXCEEDED error.

**Validates: Requirements 4.9**

### Property 15: Court Type Defaults

*For any* court creation request that omits `durationMinutes` or `maxCapacity`, the created court should have the default values for its court type: TENNIS (60min, 4), PADEL (90min, 4), BASKETBALL (60min, 10), FOOTBALL_5X5 (60min, 10).

**Validates: Requirements 5.1, 5.2, 5.3**

### Property 16: Availability Window Time Ordering

*For any* availability window where `endTime` is not strictly after `startTime`, the window creation should be rejected with a validation error.

**Validates: Requirements 6.3**

### Property 17: Availability Window Overlap Detection

*For any* two availability windows for the same court and day of week where the time ranges overlap, the update should be rejected with an OVERLAPPING_WINDOWS error listing the conflicts.

**Validates: Requirements 6.4**

### Property 18: Availability Override Duplicate Prevention

*For any* attempt to create an availability override with the same court, date, and time range as an existing override, the creation should be rejected.

**Validates: Requirements 7.4**

### Property 19: Orthodox Easter Calculation Correctness

*For any* year between 2020 and 2050, the calculated Orthodox Easter date should match the known correct date from the Orthodox calendar.

**Validates: Requirements 8.3**

### Property 20: Easter-Relative Holiday Date Calculation

*For any* holiday with an EASTER_OFFSET pattern and any year, the calculated date should be exactly the Orthodox Easter date for that year plus the offset days.

**Validates: Requirements 8.2**

### Property 21: Custom Holiday Name Uniqueness

*For any* court owner, attempting to create two custom holidays with the same `nameEl` should result in the second creation being rejected with a HOLIDAY_NAME_EXISTS error.

**Validates: Requirements 9.2**

### Property 22: Bulk Holiday Application Atomicity

*For any* bulk holiday application request, if any court in the request fails validation (e.g., not owned by requester), no courts should be modified and the entire operation should be rolled back.

**Validates: Requirements 10.2**

### Property 23: Geospatial Search Radius Constraint

*For any* court search with specified coordinates and radius, all returned courts should have a calculated distance less than or equal to the specified radius in kilometers.

**Validates: Requirements 13.1**

### Property 24: Distance Calculation Accuracy

*For any* court search result, the `distanceKm` field should match the actual geodesic distance from the search coordinates to the court's location within a tolerance of 0.1 km.

**Validates: Requirements 13.2**

### Property 25: Search Returns Only Visible Courts

*For any* court search request, no court with `visible = false` should appear in the results.

**Validates: Requirements 13.8**

### Property 26: Search Result Sorting

*For any* search with a `sortBy` parameter, results should be ordered correctly: `distance` (ascending), `price` (ascending), `rating` (descending, nulls last), `name` (ascending alphabetical).

**Validates: Requirements 13.12**

### Property 27: Zero-Result Suggestions

*For any* search returning zero results, the response should include a `suggestion` field with one of `EXPAND_RADIUS`, `REMOVE_FILTERS`, or `TRY_DIFFERENT_DATE` based on which constraint is most restrictive.

**Validates: Requirements 13.9**

### Property 28: Verification Re-submission Linking

*For any* verification re-submission after rejection, the new request should have `previousRequestId` linking to the rejected request, enabling reviewers to see the history.

**Validates: Requirements 17.6**

### Property 29: Pricing Rule Multiplier Range

*For any* pricing rule with a multiplier less than 0.10 or greater than 5.00, the rule creation should be rejected with a validation error.

**Validates: Requirements 18.2**

### Property 30: Pricing Rule Overlap Detection

*For any* two pricing rules for the same court and day of week where the time ranges overlap, the update should be rejected with an OVERLAPPING_PRICING_RULES error.

**Validates: Requirements 18.3**

### Property 31: Price Calculation Correctness

*For any* base price and applicable pricing rule multiplier, the effective price should equal `basePriceCents * multiplier` rounded to the nearest integer.

**Validates: Requirements 18.8**

### Property 32: Cancellation Tier Refund Percent Range

*For any* cancellation tier with a `refundPercent` less than 0 or greater than 100, the tier configuration should be rejected with a validation error.

**Validates: Requirements 19.2**

### Property 33: Cancellation Tier Ordering Validation

*For any* set of cancellation tiers, if a tier with fewer threshold hours has a higher refund percent than a tier with more threshold hours, the configuration should be rejected with an INVALID_CANCELLATION_TIERS error.

**Validates: Requirements 19.4**

### Property 34: Default Cancellation Policy

*For any* court with no cancellation tiers configured, the effective cancellation policy should be 100% refund regardless of timing.

**Validates: Requirements 19.8**

### Property 35: Availability Cache Round-Trip

*For any* availability request that results in a cache miss, the computed result should be cached such that an immediate subsequent request for the same court, date, and duration returns the cached data (cache hit) with identical slot information.

**Validates: Requirements 20.2, 20.3, 20.4**

### Property 36: Cache Invalidation on Config Changes

*For any* availability window, override, or pricing rule change, the relevant cache entries for the affected court should be invalidated so that subsequent availability requests recompute from the database.

**Validates: Requirements 20.5**

### Property 37: Weather Recommendation Logic

*For any* weather forecast, the recommendation should be:
- OUTDOOR_IDEAL if: clear/sunny, 15-30°C, precipitation < 20%, wind < 20 km/h
- OUTDOOR_OK if: partly cloudy, 10-35°C, precipitation < 40%, wind < 30 km/h
- INDOOR_RECOMMENDED otherwise

**Validates: Requirements 21.3**

### Property 38: Forecast Window Validation

*For any* weather forecast request with a date more than 7 days in the future, the response should indicate `available: false` with an appropriate message.

**Validates: Requirements 21.6**

### Property 39: Favorites Operations Idempotency

*For any* user and court, adding the same favorite multiple times should succeed without error, and removing a non-existent favorite should succeed without error.

**Validates: Requirements 22.1, 22.2**

### Property 40: Skill Level Range Validation

*For any* skill level value less than 1 or greater than 7, the skill level update should be rejected with a validation error.

**Validates: Requirements 24.4**

### Property 41: Audit Log Append-Only

*For any* audit log entry, the system should never allow UPDATE or DELETE operations on the `court_owner_audit_logs` table. All entries are immutable once written.

**Validates: Requirements 27.3**

### Property 42: Kafka Non-Blocking Publish

*For any* operation that publishes Kafka events (court updates, pricing changes, availability changes, notifications), the operation should succeed even if the Kafka broker is unavailable. The event should be logged and skipped without blocking the triggering operation.

**Validates: Requirements 26.8**

### Property 43: Timezone Interpretation Correctness

*For any* availability window with times specified in a court's timezone, the computed slots should reflect the correct local times regardless of server timezone.

**Validates: Requirements 34.1**

### Property 44: DST Transition Handling

*For any* date that is a DST transition day in the court's timezone, availability slots should correctly handle the transition: no slots during skipped hours (spring forward), and single occurrence of repeated hours (fall back).

**Validates: Requirements 34.3**

### Property 45: Timezone Validation

*For any* timezone string that is not a valid IANA timezone identifier, court creation or update should be rejected with an INVALID_TIMEZONE error.

**Validates: Requirements 34.6**

### Property 46: Bulk Operation Ownership Validation

*For any* bulk operation (pricing update, availability update) where the authenticated user does not own all specified courts, the entire operation should be rejected with a NOT_OWNER_OF_ALL_COURTS error listing unauthorized court IDs.

**Validates: Requirements 35.3**

### Property 47: Bulk Operation Limit

*For any* bulk operation request specifying more than 50 courts, the request should be rejected with a BULK_LIMIT_EXCEEDED error.

**Validates: Requirements 35.6**

### Property 48: Notification Channel Minimum Constraint

*For any* notification preference update that would disable all channels (push, email, inApp) for an event type, the update should be rejected with an AT_LEAST_ONE_CHANNEL_REQUIRED error.

**Validates: Requirements 37.5**

### Property 49: Reminder Rule Type Validation

*For any* reminder rule, the required fields based on rule type should be validated:
- BOOKING_REMINDER: requires `triggerHoursBefore` (1-168)
- PENDING_CONFIRMATION: requires `triggerHoursBefore` (1-48)
- LOW_OCCUPANCY: requires `thresholdPercent` (1-100)
- DAILY_SUMMARY: requires `triggerTime` (HH:mm)

**Validates: Requirements 38.5**


## Error Handling

### Error Response Format

All Phase 3 endpoints use a standardized error response format (Req 36.1):

```json
{
  "error": "ERROR_CODE",
  "message": "Human-readable error description",
  "details": { },
  "timestamp": "2025-01-15T10:30:00Z",
  "path": "/api/courts/123",
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Error Code Categories

| Category | HTTP Status | Error Codes |
|----------|-------------|-------------|
| Authorization | 403 | `AUTHZ_INSUFFICIENT_ROLE`, `AUTHZ_NOT_OWNER`, `AUTHZ_NOT_VERIFIED`, `NOT_OWNER_OF_ALL_COURTS` |
| Validation | 400 | `INVALID_COORDINATES`, `INVALID_TIMEZONE`, `INVALID_AMENITY`, `INVALID_IMAGE_FORMAT`, `IMAGE_TOO_LARGE`, `MAX_IMAGES_EXCEEDED`, `OVERLAPPING_WINDOWS`, `OVERLAPPING_PRICING_RULES`, `INVALID_CANCELLATION_TIERS`, `INVALID_DATE_RANGE`, `BULK_LIMIT_EXCEEDED`, `AT_LEAST_ONE_CHANNEL_REQUIRED`, `INVALID_IMAGE_URLS` |
| Conflict | 409 | `CONCURRENT_MODIFICATION`, `COURT_HAS_FUTURE_BOOKINGS`, `COURT_HAS_PENDING_PAYOUTS`, `VERIFICATION_ALREADY_PENDING`, `VERIFICATION_ALREADY_APPROVED`, `HOLIDAY_NAME_EXISTS`, `RULE_TYPE_EXISTS` |
| Not Found | 404 | `COURT_NOT_FOUND`, `IMAGE_NOT_FOUND`, `HOLIDAY_NOT_FOUND`, `OVERRIDE_NOT_FOUND`, `VERIFICATION_NOT_FOUND` |
| Business Logic | 422 | `COURT_OWNER_NOT_VERIFIED`, `STRIPE_CONNECT_NOT_ACTIVE`, `CHANGE_AFFECTS_BOOKINGS` |
| External Service | 503 | `WEATHER_SERVICE_UNAVAILABLE`, `STORAGE_SERVICE_UNAVAILABLE` |

### Exception Hierarchy

```java
// Base exception for Phase 3
public abstract class CourtManagementException extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> details;
}

// Validation exceptions (400)
public class CoordinateValidationException extends CourtManagementException { }
public class AmenityValidationException extends CourtManagementException { }
public class ImageValidationException extends CourtManagementException { }
public class OverlapValidationException extends CourtManagementException { }
public class InvalidDateRangeException extends CourtManagementException { }
public class BulkLimitExceededException extends CourtManagementException { }
public class AtLeastOneChannelRequiredException extends CourtManagementException { }
public class InvalidImageUrlsException extends CourtManagementException { }

// Conflict exceptions (409)
public class ConcurrentModificationException extends CourtManagementException { }
public class CourtHasFutureBookingsException extends CourtManagementException { }
public class CourtHasPendingPayoutsException extends CourtManagementException { }
public class HolidayNameExistsException extends CourtManagementException { }
public class VerificationAlreadyPendingException extends CourtManagementException { }
public class VerificationAlreadyApprovedException extends CourtManagementException { }
public class RuleTypeExistsException extends CourtManagementException { }

// Not Found exceptions (404)
public class CourtNotFoundException extends CourtManagementException { }
public class VerificationNotFoundException extends CourtManagementException { }

// Business logic exceptions (422)
public class CourtOwnerNotVerifiedException extends CourtManagementException { }
public class StripeConnectNotActiveException extends CourtManagementException { }

// External service exceptions (503)
public class WeatherServiceUnavailableException extends CourtManagementException { }
public class StorageServiceUnavailableException extends CourtManagementException { }
```

### Graceful Degradation

| Scenario | Behavior |
|----------|----------|
| Redis unavailable | Fall through to PostgreSQL, return `cacheStatus: "MISS"` |
| OpenWeatherMap unavailable | Return `{ available: false, message: "Weather data temporarily unavailable" }` |
| Kafka unavailable | Log event, continue operation (non-blocking publish) — Req 26.8 |
| DigitalOcean Spaces unavailable | Return 503 with retry guidance |

## Testing Strategy

### Dual Testing Approach

Phase 3 employs both unit tests and property-based tests for comprehensive coverage:

- **Unit tests**: Specific examples, edge cases, integration points
- **Property tests**: Universal properties across randomized inputs (minimum 100 iterations)

### Property-Based Testing Configuration

Library: **jqwik** (JUnit 5 compatible property-based testing for Java)

```java
// Test configuration
@PropertyDefaults(tries = 100)
public class CourtPropertyTests {
    
    // Feature: phase-3-court-management, Property 1: Court Creation Round-Trip
    @Property
    void courtCreationRoundTrip(@ForAll @ValidCourtRequest CreateCourtCommand command) {
        CourtId id = createCourtUseCase.createCourt(command);
        Court retrieved = getCourtQuery.getCourtById(id).orElseThrow();
        
        assertThat(retrieved.getNameEl()).isEqualTo(command.nameEl());
        assertThat(retrieved.getBasePriceCents()).isEqualTo(command.basePriceCents());
        // ... verify all fields
    }
}
```

Each property test MUST:
- Run minimum 100 iterations
- Reference its design document property via tag comment
- Tag format: `Feature: phase-3-court-management, Property {number}: {property_text}`
- Be implemented as a SINGLE property-based test per correctness property

### Test Slice Strategy

| Component | Test Type | Annotation |
|-----------|-----------|------------|
| Domain entities & value objects | Unit | Plain JUnit 5 |
| `OrthodoxEasterCalculator` | Unit + Property | Plain JUnit 5 + jqwik |
| Use case services | Unit + Property | `@ExtendWith(MockitoExtension.class)` |
| Controllers | Slice | `@WebMvcTest` |
| Repositories | Slice | `@DataJpaTest` + Testcontainers |
| Kafka consumers | Integration | `@SpringBootTest` + Testcontainers |
| External APIs | Integration | `@RestClientTest` or WireMock |
| Geospatial queries | Integration | `@DataJpaTest` + PostGIS container |
| Scheduled jobs | Integration | `@SpringBootTest` with clock mocking |

### Test Data Generators

```java
// Custom Arbitraries for property-based testing (jqwik)
public class CourtArbitraries {
    
    @Provide
    Arbitrary<GeoLocation> validGeoLocation() {
        return Combinators.combine(
            Arbitraries.doubles().between(-90, 90),
            Arbitraries.doubles().between(-180, 180)
        ).as(GeoLocation::new);
    }
    
    @Provide
    Arbitrary<AvailabilityWindow> validAvailabilityWindow() {
        return Combinators.combine(
            Arbitraries.of(DayOfWeek.values()),
            Arbitraries.integers().between(6, 20).map(h -> LocalTime.of(h, 0)),
            Arbitraries.integers().between(1, 4)
        ).as((day, start, duration) -> 
            new AvailabilityWindow(null, day, start, start.plusHours(duration)));
    }
    
    @Provide
    Arbitrary<PricingRule> validPricingRule() {
        return Combinators.combine(
            Arbitraries.of(DayOfWeek.values()),
            Arbitraries.integers().between(6, 20).map(h -> LocalTime.of(h, 0)),
            Arbitraries.integers().between(1, 4),
            Arbitraries.bigDecimals().between(new BigDecimal("0.10"), new BigDecimal("5.00"))
        ).as((day, start, duration, mult) -> 
            new PricingRule(null, day, start, start.plusHours(duration), mult, null));
    }
    
    @Provide
    Arbitrary<SkillLevel> validSkillLevel() {
        return Combinators.combine(
            Arbitraries.create(() -> new UserId(UUID.randomUUID())),
            Arbitraries.of(CourtType.values()),
            Arbitraries.integers().between(1, 7)
        ).as((userId, courtType, level) -> 
            new SkillLevel(userId, courtType, level, SkillLevel.labelFor(level)));
    }
}
```

### Integration Test Infrastructure

```java
@SpringBootTest
@Testcontainers
class CourtIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgis/postgis:15-3.3")
        .withDatabaseName("courtbooking")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

### External API Mocking

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class WeatherServiceIntegrationTest {
    
    @Test
    void shouldReturnWeatherForecast() {
        stubFor(get(urlPathMatching("/data/2.5/forecast.*"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                      "list": [{
                        "main": {"temp": 22, "feels_like": 21, "humidity": 65},
                        "wind": {"speed": 3.5},
                        "pop": 0.1,
                        "weather": [{"main": "Clear", "icon": "01d"}]
                      }]
                    }
                    """)));
        
        WeatherForecastResult result = weatherService.getForecast(37.9838, 23.7275, 
            LocalDate.now().plusDays(1), LocalTime.of(14, 0));
        
        assertThat(result.recommendation()).isEqualTo(WeatherRecommendation.OUTDOOR_IDEAL);
    }
}
```

## External Service Integrations

### DigitalOcean Spaces (Image Storage)

```yaml
# Configuration
digitalocean:
  spaces:
    endpoint: https://fra1.digitaloceanspaces.com
    region: fra1
    bucket: court-booking-images
    cdn-endpoint: https://court-booking-images.fra1.cdn.digitaloceanspaces.com
    access-key: ${DO_SPACES_ACCESS_KEY}
    secret-key: ${DO_SPACES_SECRET_KEY}
```

**Storage Path Convention**: `courts/{courtId}/{uuid}.{extension}`
**Verification Documents**: `verification/{courtOwnerId}/{uuid}.{extension}`

**Operations**:
- Upload with EXIF stripping (Req 4.2)
- Generate CDN URLs
- Delete individual images
- Bulk delete on court deletion

### OpenWeatherMap (Weather Forecasts)

```yaml
# Configuration
openweathermap:
  api-key: ${OPENWEATHERMAP_API_KEY}
  base-url: https://api.openweathermap.org/data/2.5
  rate-limit: 60  # calls per minute (free tier)
  cache-ttl: 10m
```

**Endpoints Used**:
- `/forecast` - 5-day/3-hour forecast

**Rate Limiting**: Token bucket with 60 requests/minute, graceful degradation on limit exceeded.

### Kafka Topics

| Topic | Direction | Partition Key | Consumer Group |
|-------|-----------|---------------|----------------|
| `court-update-events` | Publish | `courtId` (or `courtOwnerId` for STRIPE_CONNECT_STATUS_CHANGED) | - |
| `notification-events` | Publish | `userId` | - |
| `booking-events` | Consume | `courtId` | `platform-service-booking-consumer` |


## API Endpoint Summary

### Public Endpoints (No Auth)

| Method | Path | Description | Req |
|--------|------|-------------|-----|
| GET | `/api/holidays/national` | List national holidays with calculated dates | 8.4 |
| GET | `/api/weather` | Get weather forecast | 21.2 |

### Customer Endpoints (CUSTOMER, COURT_OWNER)

| Method | Path | Description | Req |
|--------|------|-------------|-----|
| GET | `/api/courts` | Search courts by location | 13.1 |
| GET | `/api/courts/map` | Aggregated map data | 14.1 |
| GET | `/api/courts/{id}` | Court detail | 15.1 |
| GET | `/api/courts/{id}/availability` | Available slots | 20.2 |
| POST | `/api/favorites/{courtId}` | Add favorite | 22.1 |
| DELETE | `/api/favorites/{courtId}` | Remove favorite | 22.2 |
| GET | `/api/favorites` | List favorites | 22.3 |
| GET | `/api/preferences` | Get preferences | 23.2 |
| PUT | `/api/preferences` | Update preferences | 23.1 |
| GET | `/api/skill-levels` | Get skill levels | 24.2 |
| PUT | `/api/skill-levels` | Update skill levels | 24.1 |

### Court Owner Endpoints (COURT_OWNER)

| Method | Path | Description | Req |
|--------|------|-------------|-----|
| POST | `/api/courts` | Create court | 1.1 |
| PUT | `/api/courts/{id}` | Update court | 2.1 |
| DELETE | `/api/courts/{id}` | Delete court | 3.1 |
| PUT | `/api/courts/{id}/visibility` | Toggle visibility | 33.1 |
| PUT | `/api/courts/{id}/confirmation-mode` | Update confirmation mode | 2.9 |
| PUT | `/api/courts/{id}/waitlist-config` | Update waitlist config | 2.10 |
| POST | `/api/courts/{id}/images` | Upload images | 4.1 |
| DELETE | `/api/courts/{id}/images/{imageId}` | Delete image | 4.7 |
| PUT | `/api/courts/{id}/images/order` | Reorder images | 4.8 |
| GET | `/api/courts/{id}/availability/windows` | Get availability windows | 6.1 |
| PUT | `/api/courts/{id}/availability/windows` | Update availability windows | 6.2 |
| POST | `/api/courts/{id}/availability/overrides` | Create override | 7.1 |
| GET | `/api/courts/{id}/availability/overrides` | List overrides | 7.2 |
| DELETE | `/api/courts/{id}/availability/overrides/{overrideId}` | Delete override | 7.3 |
| GET | `/api/courts/{id}/holidays` | List court holiday subscriptions | 10.8 |
| DELETE | `/api/courts/{id}/holidays/{holidayId}` | Remove court holiday | 10.9 |
| GET | `/api/courts/{id}/pricing-rules` | Get pricing rules | 18.4 |
| PUT | `/api/courts/{id}/pricing-rules` | Update pricing rules | 18.5 |
| GET | `/api/courts/{id}/cancellation-tiers` | Get cancellation tiers | 19.5 |
| PUT | `/api/courts/{id}/cancellation-tiers` | Update cancellation tiers | 19.1 |
| GET | `/api/courts/mine` | List owned courts | 16.1 |
| POST | `/api/verification/submit` | Submit verification | 17.1 |
| DELETE | `/api/verification/documents/{documentId}` | Delete verification document | 17.12 |
| POST | `/api/holidays/custom` | Create custom holiday | 9.1 |
| GET | `/api/holidays/custom` | List custom holidays | 9.3 |
| PUT | `/api/holidays/custom/{holidayId}` | Update custom holiday | 9.4 |
| DELETE | `/api/holidays/custom/{holidayId}` | Delete custom holiday | 9.5 |
| POST | `/api/holidays/apply` | Bulk apply holidays | 10.1 |
| GET | `/api/holidays/calendar` | Holiday calendar | 12.1 |
| GET | `/api/court-defaults` | Get court defaults | 25.2 |
| PUT | `/api/court-defaults` | Update court defaults | 25.1 |
| GET | `/api/audit-logs` | Audit log | 27.4 |
| GET | `/api/dashboard` | Dashboard summary | 40.1 |
| PUT | `/api/courts/bulk/pricing` | Bulk pricing update | 35.1 |
| PUT | `/api/courts/bulk/availability-windows` | Bulk availability update | 35.2 |
| GET | `/api/notification-preferences` | Get notification prefs | 37.2 |
| PUT | `/api/notification-preferences` | Update notification prefs | 37.1 |
| POST | `/api/reminder-rules` | Create reminder rule | 38.1 |
| GET | `/api/reminder-rules` | List reminder rules | 38.2 |
| PUT | `/api/reminder-rules/{id}` | Update reminder rule | 38.3 |
| DELETE | `/api/reminder-rules/{id}` | Delete reminder rule | 38.4 |
| PUT | `/api/reminder-rules/{id}/enabled` | Enable/disable rule | 38.6 |

### Admin Endpoints (PLATFORM_ADMIN)

| Method | Path | Description | Req |
|--------|------|-------------|-----|
| GET | `/api/admin/verifications` | Verification queue | 17.7, 17.11 |
| POST | `/api/admin/verifications/{id}/approve` | Approve verification | 17.4 |
| POST | `/api/admin/verifications/{id}/reject` | Reject verification | 17.5 |

### Internal Endpoints (Service-to-Service)

| Method | Path | Description | Req |
|--------|------|-------------|-----|
| GET | `/internal/courts/{id}/validate-slot` | Validate booking slot | 31.1 |
| GET | `/internal/courts/{id}/calculate-price` | Calculate price | 31.2 |
| GET | `/internal/users/{id}/stripe-connect-status` | Stripe Connect status | 39.4 |
