# Implementation Plan: Phase 1b — Service Scaffolding

## Overview

This implementation plan delivers three repositories for the court booking platform: `court-booking-common` (shared Java library), `court-booking-platform-service` (Platform Service), and `court-booking-transaction-service` (Transaction Service). The deliverable is both services running on DigitalOcean Kubernetes (DOKS) — empty but healthy — with CI/CD pipelines green, local development environment working, and Flyway migrations applied.

The implementation follows a logical progression: common library first (as a dependency), then both services in parallel, followed by infrastructure (Docker, K8s, CI/CD), and finally local development setup.

## Tasks

- [ ] 1. Set up Common Library (court-booking-common)
  - [ ] 1.1 Create Gradle project structure with build.gradle.kts
    - Create `court-booking-common/build.gradle.kts` with group `gr.courtbooking`, name `court-booking-common`, Java 25
    - Configure Spring Boot starter dependencies as `compileOnly` scope
    - Add Jackson, JUnit 5, AssertJ, and jqwik dependencies
    - Configure `publishing` block with `maven-publish` plugin for GitHub Packages
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [x] 1.2 Implement DTO enum types
    - [x] 1.2.1 Create CourtType enum (TENNIS, PADEL, BASKETBALL, FOOTBALL_5X5)
      - _Requirements: 2.2_
    - [x] 1.2.2 Create LocationType enum (INDOOR, OUTDOOR)
      - _Requirements: 2.2_
    - [x] 1.2.3 Create BookingStatus enum (PENDING_CONFIRMATION, CONFIRMED, CANCELLED, COMPLETED, REJECTED)
      - _Requirements: 2.2_
    - [x] 1.2.4 Create PaymentStatus enum (NOT_REQUIRED, PENDING, AUTHORIZED, CAPTURED, REFUNDED, PARTIALLY_REFUNDED, FAILED, PAID_EXTERNALLY)
      - _Requirements: 2.2_
    - [x] 1.2.5 Create UserRole enum (CUSTOMER, COURT_OWNER, SUPPORT_AGENT, PLATFORM_ADMIN)
      - _Requirements: 2.2_
    - [x] 1.2.6 Create DayOfWeekEnum enum (MONDAY through SUNDAY)
      - _Requirements: 2.2_
    - [x] 1.2.7 Create Language enum (EL, EN)
      - _Requirements: 2.2_

  - [x] 1.3 Implement DTO record classes
    - [x] 1.3.1 Create CourtSummaryDto record with Jackson annotations
      - _Requirements: 2.1, 2.3, 2.4_
    - [x] 1.3.2 Create UserBasicDto record with Jackson annotations
      - _Requirements: 2.1, 2.3, 2.4_
    - [x] 1.3.3 Create BookingDto record with monetary fields using Cents suffix
      - _Requirements: 2.1, 2.3, 2.4_
    - [x] 1.3.4 Create PaymentDto record with monetary fields using Cents suffix
      - _Requirements: 2.1, 2.3, 2.4_
    - [x] 1.3.5 Create PricingRuleDto record
      - _Requirements: 2.1, 2.3, 2.4_

  - [x] 1.4 Write property test for DTO JSON serialization (Property 1)
    - **Property 1: DTO JSON Serialization Produces camelCase Field Names**
    - **Validates: Requirements 2.3**

  - [x] 1.5 Write property test for monetary field naming convention (Property 2)
    - **Property 2: Monetary Fields Use Cents Suffix Convention**
    - **Validates: Requirements 2.4**

  - [x] 1.6 Implement Kafka event infrastructure
    - [x] 1.6.1 Create EventEnvelope<T> generic record with validation
      - _Requirements: 3.1, 3.2_
    - [x] 1.6.2 Create KafkaTopics constants class with all topic names
      - _Requirements: 3.1_

  - [x] 1.7 Implement booking event payload classes
    - [x] 1.7.1 Create BookingCreatedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.2 Create BookingConfirmedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.3 Create BookingCancelledPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.4 Create BookingModifiedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.5 Create BookingCompletedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.6 Create SlotHeldPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.7.7 Create SlotReleasedPayload record
      - _Requirements: 3.3, 3.4_

  - [x] 1.8 Implement notification event payload classes
    - [x] 1.8.1 Create NotificationRequestedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.8.2 Create NotificationData record
      - _Requirements: 3.3, 3.4_

  - [x] 1.9 Implement court update event payload classes
    - [x] 1.9.1 Create CourtUpdatedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.9.2 Create PricingUpdatedPayload record with nested PricingRule record
      - _Requirements: 3.3, 3.4_
    - [x] 1.9.3 Create AvailabilityUpdatedPayload record with nested DateRange and Override records
      - _Requirements: 3.3, 3.4_
    - [x] 1.9.4 Create CancellationPolicyUpdatedPayload record with nested Tier record
      - _Requirements: 3.3, 3.4_
    - [x] 1.9.5 Create CourtDeletedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.9.6 Create StripeConnectStatusChangedPayload record
      - _Requirements: 3.3, 3.4_

  - [x] 1.10 Implement match event payload classes
    - [x] 1.10.1 Create MatchCreatedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.10.2 Create MatchUpdatedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.10.3 Create MatchClosedPayload record
      - _Requirements: 3.3, 3.4_

  - [x] 1.11 Implement waitlist event payload classes
    - [x] 1.11.1 Create WaitlistSlotFreedPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.11.2 Create WaitlistHoldExpiredPayload record
      - _Requirements: 3.3, 3.4_

  - [x] 1.12 Implement analytics event payload classes
    - [x] 1.12.1 Create BookingAnalyticsPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.12.2 Create RevenueAnalyticsPayload record
      - _Requirements: 3.3, 3.4_
    - [x] 1.12.3 Create PromoCodeRedeemedPayload record
      - _Requirements: 3.3, 3.4_

  - [x] 1.13 Implement security event payload classes
    - [x] 1.13.1 Create SecurityAlertPayload record with nested AlertMetadata record
      - _Requirements: 3.3, 3.4_

  - [x] 1.14 Write property test for event serialization round-trip (Property 3)
    - **Property 3: Event Payload Serialization Round-Trip**
    - **Validates: Requirements 3.4, 3.5**

  - [x] 1.15 Implement exception classes
    - [x] 1.15.1 Create CourtBookingException base class with errorCode and httpStatus fields
      - _Requirements: 4.1, 4.2_
    - [x] 1.15.2 Create ResourceNotFoundException subclass (404)
      - _Requirements: 4.1, 4.2_
    - [x] 1.15.3 Create ConflictException subclass (409)
      - _Requirements: 4.1, 4.2_
    - [x] 1.15.4 Create ValidationException subclass (400)
      - _Requirements: 4.1, 4.2_
    - [x] 1.15.5 Create UnauthorizedException subclass (401)
      - _Requirements: 4.1, 4.2_
    - [x] 1.15.6 Create ForbiddenException subclass (403)
      - _Requirements: 4.1, 4.2_

  - [x] 1.16 Implement utility classes
    - [x] 1.16.1 Create MoneyUtil class with add, subtract, multiplyByPercentage methods
      - _Requirements: 4.3_
    - [x] 1.16.2 Add calculatePlatformFee and calculateNetAmount methods to MoneyUtil
      - _Requirements: 4.3_
    - [x] 1.16.3 Create ErrorResponse record for consistent API error responses
      - _Requirements: 4.4_

  - [x] 1.17 Write property tests for MoneyUtil arithmetic (Property 4)
    - **Property 4: MoneyUtil Arithmetic Correctness**
    - **Validates: Requirements 4.3**

  - [x] 1.18 Create GitHub Actions CI workflow for common library
    - Create `.github/workflows/ci.yml` for build, test, and conditional publish to GitHub Packages
    - Configure workflow to publish only on push to `main` branch
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

- [x] 2. Checkpoint - Common Library Complete
  - Ensure all tests pass, ask the user if questions arise.

- [x] 3. Set up Platform Service (court-booking-platform-service)
  - [x] 3.1 Create Gradle project structure with build.gradle.kts
    - Create `court-booking-platform-service/build.gradle.kts` with Spring Boot 4.x plugin, Java 25
    - Add dependencies: Spring Boot Web, Actuator, Data JPA, Data Redis, Spring Kafka, Flyway, PostgreSQL, Hibernate Spatial, springdoc-openapi, jqwik, Testcontainers
    - Configure GitHub Packages repository for common library dependency
    - Configure `tasks.test` block with `useJUnitPlatform()` for JUnit 5 and jqwik
    - _Requirements: 6.1, 6.2, 23.1, 23.2, 23.3, 23.4, 23.5_

  - [x] 3.2 Create hexagonal architecture package structure
    - [x] 3.2.1 Create domain packages (domain/model, domain/exception, domain/service)
      - _Requirements: 6.3_
    - [x] 3.2.2 Create application packages (application/port/in, application/port/out, application/service)
      - _Requirements: 6.3_
    - [x] 3.2.3 Create adapter/in packages (adapter/in/web, adapter/in/kafka)
      - _Requirements: 6.3_
    - [x] 3.2.4 Create adapter/out packages (adapter/out/persistence, adapter/out/kafka, adapter/out/redis)
      - _Requirements: 6.3_
    - [x] 3.2.5 Create config package
      - _Requirements: 6.3_

  - [x] 3.3 Create Spring Boot main application class
    - Create `PlatformServiceApplication` with `@SpringBootApplication` annotation
    - _Requirements: 6.4_

  - [x] 3.4 Create placeholder health info controller
    - Create `HealthInfoController` at `/api/health/info` returning service name and version
    - _Requirements: 6.5_

  - [x] 3.5 Configure Spring profiles - shared defaults
    - [x] 3.5.1 Create application.yml with JPA, Flyway, Actuator, springdoc defaults
      - _Requirements: 9.1_
    - [x] 3.5.2 Configure Hibernate Spatial dialect for PostGIS
      - _Requirements: 9.1_
    - [x] 3.5.3 Configure graceful shutdown settings
      - _Requirements: 26.1, 26.3, 26.6_

  - [x] 3.6 Configure Spring profiles - local environment
    - [x] 3.6.1 Create application-local.yml with localhost PostgreSQL connection (port 5432)
      - _Requirements: 9.3_
    - [x] 3.6.2 Configure local Redis connection (port 6379)
      - _Requirements: 9.3_
    - [x] 3.6.3 Configure local Kafka connection (port 9092)
      - _Requirements: 9.3_
    - [x] 3.6.4 Set server port to 8081 for local development
      - _Requirements: 9.3_

  - [x] 3.7 Configure Spring profiles - dev environment
    - [x] 3.7.1 Create application-dev.yml with environment variable placeholders
      - _Requirements: 9.5_
    - [x] 3.7.2 Configure platform_service database user
      - _Requirements: 9.5, 25.1_

  - [x] 3.8 Configure Spring profiles - test environment
    - [x] 3.8.1 Create application-test.yml with environment variable placeholders
      - _Requirements: 9.7_
    - [x] 3.8.2 Configure platform_service database user
      - _Requirements: 9.7, 25.1_

  - [x] 3.9 Configure Spring profiles - staging environment
    - [x] 3.9.1 Create application-staging.yml with environment variable placeholders
      - _Requirements: 9.9_
    - [x] 3.9.2 Configure platform_service database user
      - _Requirements: 9.9, 25.1_

  - [x] 3.10 Configure Spring profiles - production environment
    - [x] 3.10.1 Create application-prod.yml with environment variable placeholders
      - _Requirements: 9.10_
    - [x] 3.10.2 Configure platform_service database user
      - _Requirements: 9.10, 25.1_
    - [x] 3.10.3 Disable Swagger UI in prod profile
      - _Requirements: 31.5_

  - [x] 3.11 Configure Spring Boot Actuator health endpoints
    - [x] 3.11.1 Configure liveness probe at `/actuator/health/liveness`
      - _Requirements: 8.1_
    - [x] 3.11.2 Configure readiness probe at `/actuator/health/readiness` with db, redis, kafka checks
      - _Requirements: 8.2_
    - [x] 3.11.3 Configure startup probe at `/actuator/health`
      - _Requirements: 8.3_
    - [x] 3.11.4 Configure health endpoint exposure settings
      - _Requirements: 8.7_

  - [x] 3.12 Configure structured JSON logging
    - [x] 3.12.1 Create logback-spring.xml with console appender for local profile
      - _Requirements: 27.1, 27.4_
    - [x] 3.12.2 Configure JSON encoder appender for cloud profiles (dev, test, staging, prod)
      - _Requirements: 27.1, 27.2_
    - [x] 3.12.3 Include MDC fields: traceId, spanId, requestId, userId, service
      - _Requirements: 27.2_
    - [x] 3.12.4 Configure log levels per profile
      - _Requirements: 27.5_

  - [x] 3.13 Implement RequestIdFilter for MDC logging
    - Create filter to extract `X-Request-ID` header and store in MDC
    - _Requirements: 27.3_

  - [x] 3.14 Write property test for structured log entries (Property 5)
    - **Property 5: Structured Log Entries Contain Required Fields**
    - **Validates: Requirements 27.1, 27.2, 27.3**

  - [x] 3.15 Configure OpenAPI/springdoc
    - [x] 3.15.1 Create OpenApiConfig class with API info
      - _Requirements: 31.1, 31.2_
    - [x] 3.15.2 Configure API docs path at /v3/api-docs
      - _Requirements: 31.3_
    - [x] 3.15.3 Configure Swagger UI path at /swagger-ui.html
      - _Requirements: 31.4_

  - [x] 3.16 Implement global exception handler
    - Create `GlobalExceptionHandler` with `@RestControllerAdvice`
    - Map domain exceptions to HTTP responses with ErrorResponse
    - _Requirements: 4.4_

- [x] 4. Set up Transaction Service (court-booking-transaction-service)
  - [x] 4.1 Create Gradle project structure with build.gradle.kts
    - Create `court-booking-transaction-service/build.gradle.kts` with Spring Boot 4.x plugin, Java 25
    - Add dependencies: Spring Boot Web, Actuator, Data JPA, Data Redis, WebSocket, Spring Kafka, Flyway, PostgreSQL, Quartz, springdoc-openapi, jqwik, Testcontainers
    - Configure GitHub Packages repository for common library dependency
    - _Requirements: 7.1, 7.2, 23.1, 23.2, 23.3, 23.4, 23.5_

  - [x] 4.2 Create hexagonal architecture package structure
    - [x] 4.2.1 Create domain packages (domain/model, domain/exception, domain/service)
      - _Requirements: 7.3_
    - [x] 4.2.2 Create application packages (application/port/in, application/port/out, application/service)
      - _Requirements: 7.3_
    - [x] 4.2.3 Create adapter/in packages (adapter/in/web, adapter/in/kafka, adapter/in/websocket)
      - _Requirements: 7.3_
    - [x] 4.2.4 Create adapter/out packages (adapter/out/persistence, adapter/out/kafka, adapter/out/redis, adapter/out/stripe)
      - _Requirements: 7.3_
    - [x] 4.2.5 Create config package
      - _Requirements: 7.3_

  - [x] 4.3 Create Spring Boot main application class
    - Create `TransactionServiceApplication` with `@SpringBootApplication` annotation
    - _Requirements: 7.4_

  - [x] 4.4 Create placeholder health info controller
    - Create `HealthInfoController` at `/api/health/info` returning service name and version
    - _Requirements: 7.5_

  - [x] 4.5 Configure Spring profiles - shared defaults
    - [x] 4.5.1 Create application.yml with JPA, Flyway, Actuator, springdoc defaults
      - _Requirements: 9.2_
    - [x] 4.5.2 Configure Quartz scheduler settings for clustered mode
      - _Requirements: 28.1, 28.2, 28.3_
    - [x] 4.5.3 Configure graceful shutdown settings
      - _Requirements: 26.2, 26.4, 26.6_

  - [x] 4.6 Configure Spring profiles - local environment
    - [x] 4.6.1 Create application-local.yml with localhost PostgreSQL connection
      - _Requirements: 9.4_
    - [x] 4.6.2 Configure local Redis connection
      - _Requirements: 9.4_
    - [x] 4.6.3 Configure local Kafka connection
      - _Requirements: 9.4_
    - [x] 4.6.4 Set server port to 8082 for local development
      - _Requirements: 9.4_
    - [x] 4.6.5 Configure Flyway to target transaction schema
      - _Requirements: 9.4_

  - [x] 4.7 Configure Spring profiles - dev environment
    - [x] 4.7.1 Create application-dev.yml with environment variable placeholders
      - _Requirements: 9.6_
    - [x] 4.7.2 Configure transaction_service database user
      - _Requirements: 9.6, 25.2_

  - [x] 4.8 Configure Spring profiles - test environment
    - [x] 4.8.1 Create application-test.yml with environment variable placeholders
      - _Requirements: 9.8_
    - [x] 4.8.2 Configure transaction_service database user
      - _Requirements: 9.8, 25.2_

  - [x] 4.9 Configure Spring profiles - staging environment
    - [x] 4.9.1 Create application-staging.yml with environment variable placeholders
      - _Requirements: 9.9_
    - [x] 4.9.2 Configure transaction_service database user
      - _Requirements: 9.9, 25.2_

  - [x] 4.10 Configure Spring profiles - production environment
    - [x] 4.10.1 Create application-prod.yml with environment variable placeholders
      - _Requirements: 9.10_
    - [x] 4.10.2 Configure transaction_service database user
      - _Requirements: 9.10, 25.2_
    - [x] 4.10.3 Disable Swagger UI in prod profile
      - _Requirements: 31.5_

  - [x] 4.11 Configure Spring Boot Actuator health endpoints
    - [x] 4.11.1 Configure liveness probe at `/actuator/health/liveness`
      - _Requirements: 8.4_
    - [x] 4.11.2 Configure readiness probe at `/actuator/health/readiness`
      - _Requirements: 8.5_
    - [x] 4.11.3 Configure startup probe at `/actuator/health`
      - _Requirements: 8.6_

  - [x] 4.12 Configure Quartz Scheduler for clustered mode
    - [x] 4.12.1 Configure JDBC job store with isClustered=true
      - _Requirements: 28.1, 28.2_
    - [x] 4.12.2 Set instanceId=AUTO for unique scheduler instance IDs
      - _Requirements: 28.3_
    - [x] 4.12.3 Configure thread pool with configurable thread count
      - _Requirements: 28.4, 28.5_

  - [x] 4.13 Configure structured JSON logging
    - [x] 4.13.1 Create logback-spring.xml with console appender for local profile
      - _Requirements: 27.1, 27.4_
    - [x] 4.13.2 Configure JSON encoder appender for cloud profiles
      - _Requirements: 27.1, 27.2_

  - [x] 4.14 Implement RequestIdFilter for MDC logging
    - Create filter to extract `X-Request-ID` header and store in MDC
    - _Requirements: 27.3_

  - [x] 4.15 Configure OpenAPI/springdoc
    - [x] 4.15.1 Create OpenApiConfig class with API info
      - _Requirements: 31.1, 31.2_
    - [x] 4.15.2 Configure API docs and Swagger UI paths
      - _Requirements: 31.3, 31.4_

  - [x] 4.16 Implement global exception handler
    - Create `GlobalExceptionHandler` with `@RestControllerAdvice`
    - _Requirements: 4.4_

- [x] 5. Checkpoint - Service Scaffolding Complete
  - Ensure all tests pass, ask the user if questions arise.


- [x] 6. Create Flyway Database Migrations - Platform Service
  - [x] 6.1 Create Platform Service V1 migration - Enable PostGIS
    - Create `src/main/resources/db/migration/platform/V1__create_platform_schema.sql`
    - Enable PostGIS extension
    - _Requirements: 10.1_

  - [x] 6.2 Create Platform Service V1 migration - Users and authentication tables
    - [x] 6.2.1 Create users table with all columns and constraints
      - _Requirements: 10.2_
    - [x] 6.2.2 Create indexes on users table (email, role, status, stripe_connect, stripe_customer)
      - _Requirements: 10.3_
    - [x] 6.2.3 Create oauth_providers table with unique constraint
      - _Requirements: 10.2_
    - [x] 6.2.4 Create refresh_tokens table with indexes
      - _Requirements: 10.2_
    - [x] 6.2.5 Create verification_requests table with indexes
      - _Requirements: 10.2_

  - [x] 6.3 Create Platform Service V1 migration - Courts table
    - [x] 6.3.1 Create courts table with PostGIS geometry column
      - _Requirements: 10.2_
    - [x] 6.3.2 Create GIST index on courts.location for spatial queries
      - _Requirements: 10.3_
    - [x] 6.3.3 Create indexes on courts (owner_id, court_type, location_type, visible)
      - _Requirements: 10.3_
    - [x] 6.3.4 Create composite index on courts (court_type, location_type) WHERE visible = TRUE
      - _Requirements: 10.3_

  - [x] 6.4 Create Platform Service V1 migration - Availability tables
    - [x] 6.4.1 Create availability_windows table with CHECK constraint on end_time > start_time
      - _Requirements: 10.2, 10.4_
    - [x] 6.4.2 Create availability_overrides table with indexes
      - _Requirements: 10.2_
    - [x] 6.4.3 Create holiday_templates table with unique index
      - _Requirements: 10.2_
    - [x] 6.4.4 Create court_holiday_subscriptions table with unique constraint
      - _Requirements: 10.2_
    - [x] 6.4.5 Add FK from availability_overrides to holiday_templates
      - _Requirements: 10.2_

  - [x] 6.5 Create Platform Service V1 migration - User preferences tables
    - [x] 6.5.1 Create favorites table with composite primary key
      - _Requirements: 10.2_
    - [x] 6.5.2 Create preferences table with user_id as primary key
      - _Requirements: 10.2_
    - [x] 6.5.3 Create skill_levels table with composite primary key and CHECK constraint
      - _Requirements: 10.2, 10.4_

  - [x] 6.6 Create Platform Service V1 migration - Pricing tables
    - [x] 6.6.1 Create pricing_rules table with CHECK constraint on multiplier range
      - _Requirements: 10.2, 10.4_
    - [x] 6.6.2 Create special_date_pricing table with unique index
      - _Requirements: 10.2_
    - [x] 6.6.3 Create cancellation_tiers table with unique index on court_id, threshold_hours
      - _Requirements: 10.2_

  - [x] 6.7 Create Platform Service V1 migration - Promo codes tables
    - [x] 6.7.1 Create promo_codes table with CHECK constraints
      - _Requirements: 10.2, 10.4_
    - [x] 6.7.2 Create promo_code_redemptions table with unique index
      - _Requirements: 10.2_

  - [x] 6.8 Create Platform Service V1 migration - Support tables
    - [x] 6.8.1 Create support_tickets table with CHECK constraints and indexes
      - _Requirements: 10.2, 10.4_
    - [x] 6.8.2 Create support_messages table with index on ticket_id
      - _Requirements: 10.2_
    - [x] 6.8.3 Create support_attachments table with index on message_id
      - _Requirements: 10.2_

  - [x] 6.9 Create Platform Service V1 migration - Court owner tables
    - [x] 6.9.1 Create court_owner_audit_logs table with indexes
      - _Requirements: 10.2_
    - [x] 6.9.2 Create reminder_rules table with CHECK constraint and indexes
      - _Requirements: 10.2, 10.4_
    - [x] 6.9.3 Create court_owner_notification_preferences table with unique index
      - _Requirements: 10.2_
    - [x] 6.9.4 Create court_defaults table
      - _Requirements: 10.2_

  - [x] 6.10 Create Platform Service V1 migration - Security tables
    - [x] 6.10.1 Create security_alerts table with CHECK constraints
      - _Requirements: 10.2, 10.4_
    - [x] 6.10.2 Create indexes on security_alerts (status, severity, user_id, ip_address, type)
      - _Requirements: 10.3_
    - [x] 6.10.3 Create ip_blocklist table with indexes
      - _Requirements: 10.2_
    - [x] 6.10.4 Create failed_auth_attempts table with indexes
      - _Requirements: 10.2_

  - [x] 6.11 Create Platform Service V1 migration - Misc tables
    - [x] 6.11.1 Create translations table with unique index on key, language
      - _Requirements: 10.2_
    - [x] 6.11.2 Create feature_flags table with unique constraint on flag_key
      - _Requirements: 10.2_
    - [x] 6.11.3 Create court_ratings table with unique index on booking_id
      - _Requirements: 10.2_

  - [x] 6.12 Create Platform Service V2 migration - Cross-schema views
    - [x] 6.12.1 Create v_court_summary view
      - _Requirements: 10.5_
    - [x] 6.12.2 Create v_user_basic view
      - _Requirements: 10.5_
    - [x] 6.12.3 Create v_court_cancellation_tiers view
      - _Requirements: 10.5_
    - [x] 6.12.4 Create v_user_skill_level view
      - _Requirements: 10.5_
    - [x] 6.12.5 Grant SELECT on views to transaction_service role
      - _Requirements: 10.5_

- [x] 7. Create Flyway Database Migrations - Transaction Service
  - [x] 7.1 Create Transaction Service V1 migration - Bookings table
    - [x] 7.1.1 Create bookings table with all columns and CHECK constraints
      - _Requirements: 11.1, 11.4_
    - [x] 7.1.2 Create indexes on bookings (court_id, user_id, date, status, court_owner_id)
      - _Requirements: 11.3_
    - [x] 7.1.3 Create EXCLUDE constraint for non-overlapping bookings
      - _Requirements: 11.4_

  - [x] 7.2 Create Transaction Service V1 migration - Payments table
    - [x] 7.2.1 Create payments table with CHECK constraints
      - _Requirements: 11.1, 11.4_
    - [x] 7.2.2 Create indexes on payments (booking_id, user_id, status, stripe_payment_intent_id)
      - _Requirements: 11.3_

  - [x] 7.3 Create Transaction Service V1 migration - Audit logs table
    - [x] 7.3.1 Create audit_logs table with CHECK constraint on actor_type
      - _Requirements: 11.1, 11.4_
    - [x] 7.3.2 Create indexes on audit_logs (entity_type, entity_id, actor_id, created_at)
      - _Requirements: 11.3_

  - [x] 7.4 Create Transaction Service V1 migration - Notifications table
    - [x] 7.4.1 Create notifications table with CHECK constraints
      - _Requirements: 11.1, 11.4_
    - [x] 7.4.2 Create indexes on notifications (user_id, status, created_at)
      - _Requirements: 11.3_
    - [x] 7.4.3 Create device_tokens table with unique constraint
      - _Requirements: 11.1_

  - [x] 7.5 Create Transaction Service V1 migration - Waitlist table
    - [x] 7.5.1 Create waitlists table with CHECK constraint on status
      - _Requirements: 11.1, 11.4_
    - [x] 7.5.2 Create indexes on waitlists (court_id, date, user_id, status)
      - _Requirements: 11.3_

  - [x] 7.6 Create Transaction Service V1 migration - Open matches tables
    - [x] 7.6.1 Create open_matches table with CHECK constraints
      - _Requirements: 11.1, 11.4_
    - [x] 7.6.2 Create indexes on open_matches (court_id, date, status, creator_user_id)
      - _Requirements: 11.3_
    - [x] 7.6.3 Create match_players table with unique constraint
      - _Requirements: 11.1_
    - [x] 7.6.4 Create match_join_requests table with CHECK constraint
      - _Requirements: 11.1, 11.4_
    - [x] 7.6.5 Create match_messages table with index
      - _Requirements: 11.1_

  - [x] 7.7 Create Transaction Service V1 migration - Split payments tables
    - [x] 7.7.1 Create split_payments table with CHECK constraint on status
      - _Requirements: 11.1, 11.4_
    - [x] 7.7.2 Create split_payment_shares table with CHECK constraint
      - _Requirements: 11.1, 11.4_
    - [x] 7.7.3 Create indexes on split_payment_shares (split_payment_id, user_id)
      - _Requirements: 11.3_

  - [x] 7.8 Create Transaction Service V2 migration - Quartz tables
    - [x] 7.8.1 Create QRTZ_JOB_DETAILS table
      - _Requirements: 11.5_
    - [x] 7.8.2 Create QRTZ_TRIGGERS table with FK to JOB_DETAILS
      - _Requirements: 11.5_
    - [x] 7.8.3 Create QRTZ_SIMPLE_TRIGGERS table with FK to TRIGGERS
      - _Requirements: 11.5_
    - [x] 7.8.4 Create QRTZ_CRON_TRIGGERS table with FK to TRIGGERS
      - _Requirements: 11.5_
    - [x] 7.8.5 Create QRTZ_SIMPROP_TRIGGERS table with FK to TRIGGERS
      - _Requirements: 11.5_
    - [x] 7.8.6 Create QRTZ_BLOB_TRIGGERS table with FK to TRIGGERS
      - _Requirements: 11.5_
    - [x] 7.8.7 Create QRTZ_CALENDARS table
      - _Requirements: 11.5_
    - [x] 7.8.8 Create QRTZ_PAUSED_TRIGGER_GRPS table
      - _Requirements: 11.5_
    - [x] 7.8.9 Create QRTZ_FIRED_TRIGGERS table
      - _Requirements: 11.5_
    - [x] 7.8.10 Create QRTZ_SCHEDULER_STATE table
      - _Requirements: 11.5_
    - [x] 7.8.11 Create QRTZ_LOCKS table
      - _Requirements: 11.5_
    - [x] 7.8.12 Create all Quartz indexes (idx_qrtz_j_req_recovery, idx_qrtz_j_grp, idx_qrtz_t_j, etc.)
      - _Requirements: 11.5_

- [x] 8. Checkpoint - Database Migrations Complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Create Dockerfiles
  - [ ] 9.1 Create Platform Service Dockerfile
    - [ ] 9.1.1 Create build stage with Gradle 9.3 and Eclipse Temurin 23
      - _Requirements: 12.1, 12.3_
    - [ ] 9.1.2 Copy build.gradle.kts and settings.gradle.kts first for dependency caching
      - _Requirements: 12.4_
    - [ ] 9.1.3 Create runtime stage with Eclipse Temurin 21 JRE Alpine
      - _Requirements: 12.3_
    - [ ] 9.1.4 Configure non-root user (appuser:appgroup) for security
      - _Requirements: 12.5_
    - [ ] 9.1.5 Configure JAVA_OPTS environment variable with G1GC settings
      - _Requirements: 12.6_
    - [ ] 9.1.6 Add HEALTHCHECK instruction
      - _Requirements: 12.7_

  - [ ] 9.2 Create Transaction Service Dockerfile
    - [ ] 9.2.1 Create build stage with Gradle 9.3 and Eclipse Temurin 23
      - _Requirements: 12.2, 12.3_
    - [ ] 9.2.2 Copy build.gradle.kts and settings.gradle.kts first for dependency caching
      - _Requirements: 12.4_
    - [ ] 9.2.3 Create runtime stage with Eclipse Temurin 21 JRE Alpine
      - _Requirements: 12.3_
    - [ ] 9.2.4 Configure non-root user for security
      - _Requirements: 12.5_
    - [ ] 9.2.5 Configure JAVA_OPTS environment variable
      - _Requirements: 12.6_
    - [ ] 9.2.6 Add HEALTHCHECK instruction
      - _Requirements: 12.7_

- [ ] 10. Create Kubernetes Manifests - Platform Service Base
  - [ ] 10.1 Create Platform Service base kustomization.yaml
    - Create `k8s/base/kustomization.yaml` referencing deployment, service, configmap
    - _Requirements: 15.1_

  - [ ] 10.2 Create Platform Service base deployment.yaml
    - [ ] 10.2.1 Configure Deployment metadata and selector
      - _Requirements: 15.2_
    - [ ] 10.2.2 Configure container with image and ports
      - _Requirements: 15.2_
    - [ ] 10.2.3 Configure envFrom for ConfigMap reference
      - _Requirements: 15.2_
    - [ ] 10.2.4 Configure env for secrets (db-password, redis-password, kafka-sasl-config)
      - _Requirements: 15.2_
    - [ ] 10.2.5 Configure resource requests (cpu: 250m, memory: 512Mi)
      - _Requirements: 15.3_
    - [ ] 10.2.6 Configure resource limits (cpu: 500m, memory: 1Gi)
      - _Requirements: 15.3_
    - [ ] 10.2.7 Configure liveness probe (initialDelaySeconds: 30, periodSeconds: 10, failureThreshold: 3)
      - _Requirements: 15.4_
    - [ ] 10.2.8 Configure readiness probe (initialDelaySeconds: 15, periodSeconds: 5, failureThreshold: 3)
      - _Requirements: 15.5_
    - [ ] 10.2.9 Configure startup probe (initialDelaySeconds: 60, periodSeconds: 10, failureThreshold: 12)
      - _Requirements: 15.6_
    - [ ] 10.2.10 Set terminationGracePeriodSeconds: 45
      - _Requirements: 15.7, 26.5_

  - [ ] 10.3 Create Platform Service base service.yaml
    - Configure ClusterIP service on port 8080
    - _Requirements: 15.2_

  - [ ] 10.4 Create Platform Service base configmap.yaml
    - Configure SPRING_PROFILES_ACTIVE, SERVER_PORT, JAVA_OPTS
    - _Requirements: 15.2_

- [ ] 11. Create Kubernetes Manifests - Platform Service Overlays
  - [ ] 11.1 Create Platform Service dev overlay
    - [ ] 11.1.1 Create k8s/overlays/dev/kustomization.yaml
      - _Requirements: 18.1, 24.1_
    - [ ] 11.1.2 Create k8s/overlays/dev/configmap-patch.yaml with dev profile and connection strings
      - _Requirements: 18.1, 24.1_
    - [ ] 11.1.3 Create k8s/overlays/dev/deployment-patch.yaml with replicas: 1
      - _Requirements: 18.1, 24.1_

  - [ ] 11.2 Create Platform Service test overlay
    - [ ] 11.2.1 Create k8s/overlays/test/kustomization.yaml
      - _Requirements: 18.1, 24.2_
    - [ ] 11.2.2 Create k8s/overlays/test/configmap-patch.yaml with test profile
      - _Requirements: 18.1, 24.2_
    - [ ] 11.2.3 Create k8s/overlays/test/deployment-patch.yaml with replicas: 1
      - _Requirements: 18.1, 24.2_

  - [ ] 11.3 Create Platform Service staging overlay
    - [ ] 11.3.1 Create k8s/overlays/staging/kustomization.yaml
      - _Requirements: 18.1, 24.3_
    - [ ] 11.3.2 Create k8s/overlays/staging/configmap-patch.yaml with staging profile
      - _Requirements: 18.1, 24.3_
    - [ ] 11.3.3 Create k8s/overlays/staging/deployment-patch.yaml with replicas: 2
      - _Requirements: 18.1, 24.3_

  - [ ] 11.4 Create Platform Service production overlay
    - [ ] 11.4.1 Create k8s/overlays/production/kustomization.yaml
      - _Requirements: 18.1, 24.4_
    - [ ] 11.4.2 Create k8s/overlays/production/configmap-patch.yaml with prod profile and production connection strings
      - _Requirements: 18.1, 24.4_
    - [ ] 11.4.3 Create k8s/overlays/production/deployment-patch.yaml with replicas: 2, increased resources
      - _Requirements: 18.1, 24.4_

- [ ] 12. Create Kubernetes Manifests - Transaction Service Base
  - [ ] 12.1 Create Transaction Service base kustomization.yaml
    - Create `k8s/base/kustomization.yaml` referencing deployment, service, configmap
    - _Requirements: 16.1_

  - [ ] 12.2 Create Transaction Service base deployment.yaml
    - [ ] 12.2.1 Configure Deployment metadata and selector
      - _Requirements: 16.2_
    - [ ] 12.2.2 Configure container with image and ports
      - _Requirements: 16.2_
    - [ ] 12.2.3 Configure envFrom for ConfigMap reference
      - _Requirements: 16.2_
    - [ ] 12.2.4 Configure env for secrets
      - _Requirements: 16.2_
    - [ ] 12.2.5 Configure resource requests and limits
      - _Requirements: 16.3_
    - [ ] 12.2.6 Configure liveness probe
      - _Requirements: 16.4_
    - [ ] 12.2.7 Configure readiness probe
      - _Requirements: 16.5_
    - [ ] 12.2.8 Configure startup probe
      - _Requirements: 16.6_
    - [ ] 12.2.9 Set terminationGracePeriodSeconds: 45
      - _Requirements: 16.7, 26.5_

  - [ ] 12.3 Create Transaction Service base service.yaml
    - Configure ClusterIP service on port 8080
    - _Requirements: 16.2_

  - [ ] 12.4 Create Transaction Service base configmap.yaml
    - Configure SPRING_PROFILES_ACTIVE, SERVER_PORT, JAVA_OPTS
    - _Requirements: 16.2_

- [ ] 13. Create Kubernetes Manifests - Transaction Service Overlays
  - [ ] 13.1 Create Transaction Service dev overlay
    - [ ] 13.1.1 Create k8s/overlays/dev/kustomization.yaml
      - _Requirements: 18.2, 24.1_
    - [ ] 13.1.2 Create k8s/overlays/dev/configmap-patch.yaml
      - _Requirements: 18.2, 24.1_
    - [ ] 13.1.3 Create k8s/overlays/dev/deployment-patch.yaml
      - _Requirements: 18.2, 24.1_

  - [ ] 13.2 Create Transaction Service test overlay
    - [ ] 13.2.1 Create k8s/overlays/test/kustomization.yaml
      - _Requirements: 18.2, 24.2_
    - [ ] 13.2.2 Create k8s/overlays/test/configmap-patch.yaml
      - _Requirements: 18.2, 24.2_
    - [ ] 13.2.3 Create k8s/overlays/test/deployment-patch.yaml
      - _Requirements: 18.2, 24.2_

  - [ ] 13.3 Create Transaction Service staging overlay
    - [ ] 13.3.1 Create k8s/overlays/staging/kustomization.yaml
      - _Requirements: 18.2, 24.3_
    - [ ] 13.3.2 Create k8s/overlays/staging/configmap-patch.yaml
      - _Requirements: 18.2, 24.3_
    - [ ] 13.3.3 Create k8s/overlays/staging/deployment-patch.yaml
      - _Requirements: 18.2, 24.3_

  - [ ] 13.4 Create Transaction Service production overlay
    - [ ] 13.4.1 Create k8s/overlays/production/kustomization.yaml
      - _Requirements: 18.2, 24.4_
    - [ ] 13.4.2 Create k8s/overlays/production/configmap-patch.yaml
      - _Requirements: 18.2, 24.4_
    - [ ] 13.4.3 Create k8s/overlays/production/deployment-patch.yaml
      - _Requirements: 18.2, 24.4_

- [ ] 14. Create shared Ingress resource
  - [ ] 14.1 Configure Ingress metadata with annotations
    - [ ] 14.1.1 Set kubernetes.io/ingress.class: nginx
      - _Requirements: 17.1_
    - [ ] 14.1.2 Set cert-manager.io/cluster-issuer: letsencrypt-prod
      - _Requirements: 17.5_
    - [ ] 14.1.3 Configure proxy settings (body-size, read-timeout)
      - _Requirements: 17.1_

  - [ ] 14.2 Configure TLS settings
    - Configure TLS with host api.courtbooking.gr and secret name
    - _Requirements: 17.5_

  - [ ] 14.3 Configure Platform Service routes
    - [ ] 14.3.1 Route /api/auth to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.2 Route /api/users to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.3 Route /api/courts to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.4 Route /api/weather to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.5 Route /api/analytics to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.6 Route /api/promo-codes to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.7 Route /api/feature-flags to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.8 Route /api/admin to platform-service
      - _Requirements: 17.2_
    - [ ] 14.3.9 Route /api/support to platform-service
      - _Requirements: 17.2_

  - [ ] 14.4 Configure Transaction Service routes
    - [ ] 14.4.1 Route /api/bookings to transaction-service
      - _Requirements: 17.3_
    - [ ] 14.4.2 Route /api/payments to transaction-service
      - _Requirements: 17.3_
    - [ ] 14.4.3 Route /api/notifications to transaction-service
      - _Requirements: 17.3_
    - [ ] 14.4.4 Route /api/waitlist to transaction-service
      - _Requirements: 17.3_
    - [ ] 14.4.5 Route /api/matches to transaction-service
      - _Requirements: 17.3_
    - [ ] 14.4.6 Route /api/split-payments to transaction-service
      - _Requirements: 17.3_

  - [ ] 14.5 Configure WebSocket route
    - Route /ws to transaction-service
    - _Requirements: 17.4, 17.6_

- [ ] 15. Checkpoint - Kubernetes Manifests Complete
  - Ensure all tests pass, ask the user if questions arise.


- [ ] 16. Create CI/CD Pipelines - Platform Service CI Workflow
  - [ ] 16.1 Create Platform Service CI workflow file
    - Create `.github/workflows/ci.yml` triggered on PRs to develop/main
    - _Requirements: 13.1_

  - [ ] 16.2 Configure PostgreSQL service container
    - [ ] 16.2.1 Configure postgis/postgis:15-3.3 image
      - _Requirements: 13.2, 20.1_
    - [ ] 16.2.2 Configure environment variables (POSTGRES_USER, PASSWORD, DB)
      - _Requirements: 13.2_
    - [ ] 16.2.3 Configure health check options
      - _Requirements: 13.2_

  - [ ] 16.3 Configure CI workflow steps
    - [ ] 16.3.1 Add checkout step
      - _Requirements: 13.2_
    - [ ] 16.3.2 Add Java 25 setup step with Gradle cache
      - _Requirements: 13.2_
    - [ ] 16.3.3 Add step to create test schemas (platform schema, PostGIS extension)
      - _Requirements: 13.2_
    - [ ] 16.3.4 Add build and test step with environment variables
      - _Requirements: 13.2_
    - [ ] 16.3.5 Add Flyway validate step
      - _Requirements: 13.2_
    - [ ] 16.3.6 Add Docker image build step
      - _Requirements: 13.2_

  - [ ] 16.4 Configure Trivy vulnerability scan
    - [ ] 16.4.1 Add Trivy action with SARIF output format
      - _Requirements: 21.1, 21.2_
    - [ ] 16.4.2 Configure severity filter (CRITICAL, HIGH)
      - _Requirements: 21.3_
    - [ ] 16.4.3 Configure exit-code: 1 to fail on vulnerabilities
      - _Requirements: 21.4_
    - [ ] 16.4.4 Add step to upload Trivy results as artifact
      - _Requirements: 21.2_

- [ ] 17. Create CI/CD Pipelines - Platform Service Deploy Workflow
  - [ ] 17.1 Create Platform Service deploy workflow file
    - Create `.github/workflows/deploy.yml` triggered on push to develop
    - _Requirements: 13.3_

  - [ ] 17.2 Create build-and-push job
    - [ ] 17.2.1 Configure checkout and Java 25 setup steps
      - _Requirements: 13.3_
    - [ ] 17.2.2 Add Gradle build step (./gradlew clean build -x test)
      - _Requirements: 13.3_
    - [ ] 17.2.3 Add doctl installation step
      - _Requirements: 13.3_
    - [ ] 17.2.4 Add DOCR login step
      - _Requirements: 13.3_
    - [ ] 17.2.5 Add Docker build and push step with SHA tag
      - _Requirements: 13.3_
    - [ ] 17.2.6 Configure output for image_tag
      - _Requirements: 13.3_

  - [ ] 17.3 Create deploy-dev job
    - [ ] 17.3.1 Configure job dependency on build-and-push
      - _Requirements: 13.4_
    - [ ] 17.3.2 Configure environment: dev
      - _Requirements: 13.4, 22.1_
    - [ ] 17.3.3 Add checkout step
      - _Requirements: 13.4_
    - [ ] 17.3.4 Add doctl installation and kubectl configuration steps
      - _Requirements: 13.4_
    - [ ] 17.3.5 Add kustomize edit set image step
      - _Requirements: 13.4_
    - [ ] 17.3.6 Add kubectl apply step
      - _Requirements: 13.4_
    - [ ] 17.3.7 Add kubectl rollout status step with timeout
      - _Requirements: 13.4_

  - [ ] 17.4 Create deploy-test job
    - [ ] 17.4.1 Configure job dependency on deploy-dev
      - _Requirements: 13.5_
    - [ ] 17.4.2 Configure environment: test
      - _Requirements: 13.5, 22.1_
    - [ ] 17.4.3 Add deployment steps (checkout, doctl, kubectl, kustomize, apply, rollout)
      - _Requirements: 13.5_

  - [ ] 17.5 Create trigger-qa job
    - [ ] 17.5.1 Configure job dependency on deploy-test
      - _Requirements: 13.6_
    - [ ] 17.5.2 Add repository_dispatch action to trigger QA functional regression
      - _Requirements: 13.6, 22.3_
    - [ ] 17.5.3 Configure client-payload with service, version, environment, triggered_by, run_url
      - _Requirements: 13.6_

  - [ ] 17.6 Create deploy-staging job
    - [ ] 17.6.1 Configure job dependency on trigger-qa
      - _Requirements: 13.7_
    - [ ] 17.6.2 Configure environment: staging with URL
      - _Requirements: 13.7, 22.1_
    - [ ] 17.6.3 Add deployment steps
      - _Requirements: 13.7_
    - [ ] 17.6.4 Add repository_dispatch to trigger QA staging validation
      - _Requirements: 13.7, 22.3_

  - [ ] 17.7 Create deploy-production job
    - [ ] 17.7.1 Configure job dependency on deploy-staging
      - _Requirements: 13.8_
    - [ ] 17.7.2 Configure environment: production with URL and manual approval
      - _Requirements: 13.8, 22.1, 22.4_
    - [ ] 17.7.3 Add step to tag release candidate (docker tag with version)
      - _Requirements: 13.9_
    - [ ] 17.7.4 Add deployment steps with production overlay
      - _Requirements: 13.10_
    - [ ] 17.7.5 Add kubectl rollout status step
      - _Requirements: 13.10_

- [ ] 18. Create CI/CD Pipelines - Transaction Service CI Workflow
  - [ ] 18.1 Create Transaction Service CI workflow file
    - Create `.github/workflows/ci.yml` triggered on PRs to develop/main
    - _Requirements: 14.1_

  - [ ] 18.2 Configure PostgreSQL service container
    - [ ] 18.2.1 Configure postgis/postgis:15-3.3 image
      - _Requirements: 14.2, 20.2_
    - [ ] 18.2.2 Configure environment variables
      - _Requirements: 14.2_
    - [ ] 18.2.3 Configure health check options
      - _Requirements: 14.2_

  - [ ] 18.3 Configure CI workflow steps
    - [ ] 18.3.1 Add checkout step
      - _Requirements: 14.2_
    - [ ] 18.3.2 Add Java 25 setup step with Gradle cache
      - _Requirements: 14.2_
    - [ ] 18.3.3 Add step to create platform schema first (for cross-schema view references)
      - _Requirements: 14.2, 20.3_
    - [ ] 18.3.4 Add step to create transaction schema
      - _Requirements: 14.2_
    - [ ] 18.3.5 Add build and test step
      - _Requirements: 14.2_
    - [ ] 18.3.6 Add Flyway validate step
      - _Requirements: 14.2_
    - [ ] 18.3.7 Add Docker image build step
      - _Requirements: 14.2_

  - [ ] 18.4 Configure Trivy vulnerability scan
    - [ ] 18.4.1 Add Trivy action with SARIF output format
      - _Requirements: 21.1, 21.2_
    - [ ] 18.4.2 Configure severity filter and exit-code
      - _Requirements: 21.3, 21.4_
    - [ ] 18.4.3 Add step to upload Trivy results
      - _Requirements: 21.2_

- [ ] 19. Create CI/CD Pipelines - Transaction Service Deploy Workflow
  - [ ] 19.1 Create Transaction Service deploy workflow file
    - Create `.github/workflows/deploy.yml` triggered on push to develop
    - _Requirements: 14.3_

  - [ ] 19.2 Create build-and-push job
    - [ ] 19.2.1 Configure checkout and Java 25 setup steps
      - _Requirements: 14.3_
    - [ ] 19.2.2 Add Gradle build step
      - _Requirements: 14.3_
    - [ ] 19.2.3 Add doctl installation and DOCR login steps
      - _Requirements: 14.3_
    - [ ] 19.2.4 Add Docker build and push step
      - _Requirements: 14.3_

  - [ ] 19.3 Create deploy-dev job
    - [ ] 19.3.1 Configure job dependency and environment
      - _Requirements: 14.4, 22.2_
    - [ ] 19.3.2 Add deployment steps
      - _Requirements: 14.4_

  - [ ] 19.4 Create deploy-test job
    - [ ] 19.4.1 Configure job dependency and environment
      - _Requirements: 14.5, 22.2_
    - [ ] 19.4.2 Add deployment steps
      - _Requirements: 14.5_

  - [ ] 19.5 Create trigger-qa job
    - [ ] 19.5.1 Configure repository_dispatch for QA functional regression
      - _Requirements: 14.6, 22.3_

  - [ ] 19.6 Create deploy-staging job
    - [ ] 19.6.1 Configure job dependency and environment with manual approval
      - _Requirements: 14.7, 22.2_
    - [ ] 19.6.2 Add deployment steps
      - _Requirements: 14.7_
    - [ ] 19.6.3 Add repository_dispatch for QA staging validation
      - _Requirements: 14.7, 22.3_

  - [ ] 19.7 Create deploy-production job
    - [ ] 19.7.1 Configure job dependency and environment with manual approval
      - _Requirements: 14.8, 22.2, 22.4_
    - [ ] 19.7.2 Add release candidate tagging step
      - _Requirements: 14.9_
    - [ ] 19.7.3 Add deployment steps
      - _Requirements: 14.10_

- [ ] 20. Checkpoint - CI/CD Pipelines Complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 21. Set up Local Development Environment
  - [ ] 21.1 Create Docker Compose configuration
    - [ ] 21.1.1 Create infrastructure/docker-compose.yml file
      - _Requirements: 19.1_
    - [ ] 21.1.2 Configure PostgreSQL 15 + PostGIS 3.3 service
      - _Requirements: 19.1_
    - [ ] 21.1.3 Configure PostgreSQL health check
      - _Requirements: 19.1_
    - [ ] 21.1.4 Configure PostgreSQL volume mount for init script
      - _Requirements: 19.1_
    - [ ] 21.1.5 Configure Redis 7 Alpine service
      - _Requirements: 19.1_
    - [ ] 21.1.6 Configure Redis health check
      - _Requirements: 19.1_
    - [ ] 21.1.7 Configure Zookeeper service
      - _Requirements: 19.1_
    - [ ] 21.1.8 Configure Kafka (Confluent) service with dependency on Zookeeper
      - _Requirements: 19.1_
    - [ ] 21.1.9 Configure Kafka health check
      - _Requirements: 19.1_
    - [ ] 21.1.10 Configure named volumes for data persistence
      - _Requirements: 19.1_

  - [ ] 21.2 Create database init script
    - [ ] 21.2.1 Create infrastructure/init-db.sql file
      - _Requirements: 19.2_
    - [ ] 21.2.2 Add CREATE SCHEMA statements for platform and transaction
      - _Requirements: 19.2_
    - [ ] 21.2.3 Add CREATE EXTENSION postgis statement
      - _Requirements: 19.2_
    - [ ] 21.2.4 Add GRANT statements for dev user
      - _Requirements: 19.2_
    - [ ] 21.2.5 Add ALTER USER to set default search_path
      - _Requirements: 19.2_

  - [ ] 21.3 Create Platform Service sample data seeding migration
    - [ ] 21.3.1 Create V999__seed_local_data.sql in platform/local directory
      - _Requirements: 29.1_
    - [ ] 21.3.2 Add sample users (customer, court owner)
      - _Requirements: 29.2_
    - [ ] 21.3.3 Add sample courts with PostGIS geometry
      - _Requirements: 29.2_
    - [ ] 21.3.4 Add sample availability windows
      - _Requirements: 29.2_
    - [ ] 21.3.5 Add sample pricing rules
      - _Requirements: 29.2_
    - [ ] 21.3.6 Add sample cancellation tiers
      - _Requirements: 29.2_
    - [ ] 21.3.7 Ensure idempotent seeding with ON CONFLICT clauses
      - _Requirements: 29.4_

  - [ ] 21.4 Create Transaction Service sample data seeding migration
    - [ ] 21.4.1 Create V999__seed_local_data.sql in transaction/local directory
      - _Requirements: 29.1_
    - [ ] 21.4.2 Add sample bookings
      - _Requirements: 29.3_
    - [ ] 21.4.3 Ensure idempotent seeding with ON CONFLICT clauses
      - _Requirements: 29.4_

  - [ ] 21.5 Configure Flyway locations for local seed data
    - [ ] 21.5.1 Update Platform Service application-local.yml to include local seed directory
      - _Requirements: 29.5_
    - [ ] 21.5.2 Update Transaction Service application-local.yml to include local seed directory
      - _Requirements: 29.5_

  - [ ] 21.6 Verify local development workflow
    - [ ] 21.6.1 Verify docker compose up -d starts all infrastructure
      - _Requirements: 19.3_
    - [ ] 21.6.2 Verify PostgreSQL health check passes
      - _Requirements: 19.3_
    - [ ] 21.6.3 Verify Redis health check passes
      - _Requirements: 19.3_
    - [ ] 21.6.4 Verify Kafka health check passes
      - _Requirements: 19.3_
    - [ ] 21.6.5 Verify Platform Service starts on port 8081 with local profile
      - _Requirements: 19.4_
    - [ ] 21.6.6 Verify Transaction Service starts on port 8082 with local profile
      - _Requirements: 19.4_
    - [ ] 21.6.7 Verify Flyway migrations apply successfully for Platform Service
      - _Requirements: 19.5_
    - [ ] 21.6.8 Verify Flyway migrations apply successfully for Transaction Service
      - _Requirements: 19.5_
    - [ ] 21.6.9 Verify Platform Service actuator health endpoint returns HTTP 200
      - _Requirements: 19.6_
    - [ ] 21.6.10 Verify Transaction Service actuator health endpoint returns HTTP 200
      - _Requirements: 19.6_

- [ ] 22. Create Repository Documentation
  - [ ] 22.1 Create Platform Service README.md
    - [ ] 22.1.1 Add project overview section
      - _Requirements: 30.1_
    - [ ] 22.1.2 Add prerequisites section (Java 25, Gradle 9.3.1, Docker)
      - _Requirements: 30.1_
    - [ ] 22.1.3 Add Quick Start section for 5-minute setup
      - _Requirements: 30.4_
    - [ ] 22.1.4 Add local setup instructions
      - _Requirements: 30.1_
    - [ ] 22.1.5 Add how to run tests section
      - _Requirements: 30.1_
    - [ ] 22.1.6 Add project structure explanation
      - _Requirements: 30.1_
    - [ ] 22.1.7 Add links to API spec and database schema
      - _Requirements: 30.1_

  - [ ] 22.2 Create Transaction Service README.md
    - [ ] 22.2.1 Add project overview section
      - _Requirements: 30.2_
    - [ ] 22.2.2 Add prerequisites section
      - _Requirements: 30.2_
    - [ ] 22.2.3 Add Quick Start section
      - _Requirements: 30.4_
    - [ ] 22.2.4 Add local setup instructions
      - _Requirements: 30.2_
    - [ ] 22.2.5 Add how to run tests section
      - _Requirements: 30.2_
    - [ ] 22.2.6 Add project structure explanation
      - _Requirements: 30.2_
    - [ ] 22.2.7 Add links to API spec and database schema
      - _Requirements: 30.2_

  - [ ] 22.3 Create Common Library README.md
    - [ ] 22.3.1 Add library overview section
      - _Requirements: 30.3_
    - [ ] 22.3.2 Add build and publish instructions
      - _Requirements: 30.3_
    - [ ] 22.3.3 Add Gradle dependency consumption guide with GitHub Packages authentication
      - _Requirements: 30.3_
    - [ ] 22.3.4 Add package structure explanation
      - _Requirements: 30.3_

- [ ] 23. Write Unit Tests - Common Library
  - [ ] 23.1 Write unit tests for DTO enum types
    - [ ] 23.1.1 Test CourtType enum values
      - _Requirements: 2.2_
    - [ ] 23.1.2 Test BookingStatus enum values
      - _Requirements: 2.2_
    - [ ] 23.1.3 Test PaymentStatus enum values
      - _Requirements: 2.2_

  - [ ] 23.2 Write unit tests for DTO record classes
    - [ ] 23.2.1 Test CourtSummaryDto JSON serialization
      - _Requirements: 2.3_
    - [ ] 23.2.2 Test BookingDto monetary field naming
      - _Requirements: 2.4_
    - [ ] 23.2.3 Test PaymentDto monetary field naming
      - _Requirements: 2.4_

  - [ ] 23.3 Write unit tests for EventEnvelope
    - [ ] 23.3.1 Test EventEnvelope validation (null checks)
      - _Requirements: 3.2_
    - [ ] 23.3.2 Test EventEnvelope.create factory method
      - _Requirements: 3.2_

  - [ ] 23.4 Write unit tests for exception classes
    - [ ] 23.4.1 Test CourtBookingException error code and HTTP status
      - _Requirements: 4.1_
    - [ ] 23.4.2 Test ResourceNotFoundException with resource type and ID
      - _Requirements: 4.2_
    - [ ] 23.4.3 Test ValidationException
      - _Requirements: 4.2_

  - [ ] 23.5 Write unit tests for MoneyUtil
    - [ ] 23.5.1 Test MoneyUtil.add with overflow handling
      - _Requirements: 4.3_
    - [ ] 23.5.2 Test MoneyUtil.subtract with underflow handling
      - _Requirements: 4.3_
    - [ ] 23.5.3 Test MoneyUtil.multiplyByPercentage with boundary values
      - _Requirements: 4.3_
    - [ ] 23.5.4 Test MoneyUtil.calculatePlatformFee
      - _Requirements: 4.3_
    - [ ] 23.5.5 Test MoneyUtil.calculateNetAmount
      - _Requirements: 4.3_

- [ ] 24. Write Unit Tests - Platform Service
  - [ ] 24.1 Write unit tests for HealthInfoController
    - [ ] 24.1.1 Test /api/health/info returns service name
      - _Requirements: 6.5_
    - [ ] 24.1.2 Test /api/health/info returns version
      - _Requirements: 6.5_

  - [ ] 24.2 Write unit tests for RequestIdFilter
    - [ ] 24.2.1 Test filter extracts X-Request-ID header
      - _Requirements: 27.3_
    - [ ] 24.2.2 Test filter generates UUID when header missing
      - _Requirements: 27.3_
    - [ ] 24.2.3 Test filter cleans up MDC after request
      - _Requirements: 27.3_

  - [ ] 24.3 Write unit tests for GlobalExceptionHandler
    - [ ] 24.3.1 Test ResourceNotFoundException returns 404
      - _Requirements: 4.4_
    - [ ] 24.3.2 Test ValidationException returns 400
      - _Requirements: 4.4_
    - [ ] 24.3.3 Test ConflictException returns 409
      - _Requirements: 4.4_

- [ ] 25. Write Unit Tests - Transaction Service
  - [ ] 25.1 Write unit tests for HealthInfoController
    - [ ] 25.1.1 Test /api/health/info returns service name
      - _Requirements: 7.5_
    - [ ] 25.1.2 Test /api/health/info returns version
      - _Requirements: 7.5_

  - [ ] 25.2 Write unit tests for RequestIdFilter
    - [ ] 25.2.1 Test filter extracts X-Request-ID header
      - _Requirements: 27.3_
    - [ ] 25.2.2 Test filter generates UUID when header missing
      - _Requirements: 27.3_

  - [ ] 25.3 Write unit tests for GlobalExceptionHandler
    - [ ] 25.3.1 Test exception mapping to HTTP responses
      - _Requirements: 4.4_

- [ ] 26. Write Integration Tests
  - [ ] 26.1 Write Platform Service integration tests with Testcontainers
    - [ ] 26.1.1 Configure Testcontainers PostgreSQL with PostGIS
      - _Requirements: 23.4_
    - [ ] 26.1.2 Test Flyway migrations apply successfully
      - _Requirements: 10.1_
    - [ ] 26.1.3 Test actuator health endpoint with database
      - _Requirements: 8.2_

  - [ ] 26.2 Write Transaction Service integration tests with Testcontainers
    - [ ] 26.2.1 Configure Testcontainers PostgreSQL
      - _Requirements: 23.4_
    - [ ] 26.2.2 Test Flyway migrations apply successfully
      - _Requirements: 11.1_
    - [ ] 26.2.3 Test actuator health endpoint with database
      - _Requirements: 8.5_

  - [ ] 26.3 Write controller slice tests (@WebMvcTest)
    - [ ] 26.3.1 Test Platform Service HealthInfoController
      - _Requirements: 6.5_
    - [ ] 26.3.2 Test Transaction Service HealthInfoController
      - _Requirements: 7.5_

- [ ] 27. Final Checkpoint - All Components Complete
  - Ensure all tests pass, ask the user if questions arise.
  - Verify Common Library publishes to GitHub Packages
  - Verify both services build and pass all tests
  - Verify Docker images build successfully
  - Verify Kubernetes manifests are valid with `kubectl apply --dry-run`
  - Verify local development environment works end-to-end

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- The implementation order ensures dependencies are built first (common library before services)
- Database user separation (Requirement 25) is handled via Spring profile configuration
- Log sanitization (Requirement 27.6) should be implemented as a Logback filter pattern
