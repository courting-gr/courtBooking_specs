# Requirements Document

## Introduction

Phase 1b — Service Scaffolding builds on the completed Phase 1a foundation infrastructure to deliver runnable, healthy Spring Boot services on DigitalOcean Kubernetes (DOKS). This phase covers three repositories: `court-booking-common` (shared Java library), `court-booking-platform-service` (Platform Service), and `court-booking-transaction-service` (Transaction Service). The deliverable is both services running on DOKS (empty but healthy), CI/CD pipelines green, local development environment working, and Flyway migrations applied against the database schema defined in `database-schema.md`.

This document scopes requirements specifically to service scaffolding concerns — project structure, build configuration, database migrations, deployment manifests, CI/CD pipelines, and local development workflow. Business logic implementation is deferred to subsequent phases.

## Glossary

- **Platform_Service**: Spring Boot application (`court-booking-platform-service`) responsible for authentication, users, courts, availability, and analytics. Owns the `platform` PostgreSQL schema.
- **Transaction_Service**: Spring Boot application (`court-booking-transaction-service`) responsible for bookings, payments, notifications, and scheduled jobs. Owns the `transaction` PostgreSQL schema.
- **Common_Library**: Shared Java library (`court-booking-common`) published to GitHub Packages, consumed as a Maven dependency by both services. Contains DTOs, Kafka event classes, exception types, and utilities.
- **Flyway**: Database migration tool that manages versioned SQL scripts per service, per schema.
- **DOKS**: DigitalOcean Kubernetes Service — the managed Kubernetes cluster where services are deployed.
- **DOCR**: DigitalOcean Container Registry — private Docker image storage for service images.
- **Actuator**: Spring Boot Actuator module providing health check endpoints for Kubernetes probes.
- **Hexagonal_Architecture**: Software architecture pattern (ports and adapters) separating domain logic from infrastructure concerns.
- **CI_CD_Pipeline**: GitHub Actions workflow automating build, test, scan, push, and deploy steps.
- **Ingress**: Kubernetes NGINX Ingress resource defining path-based routing rules to services.
- **ConfigMap**: Kubernetes resource storing non-sensitive configuration data for pods.
- **SealedSecret**: Encrypted Kubernetes secret stored safely in Git, decrypted at runtime by the Sealed Secrets controller.
- **Trivy**: Container image vulnerability scanner used in CI/CD pipelines.
- **Cross_Schema_View**: Read-only PostgreSQL view in the `platform` schema granting Transaction_Service access to Platform data without direct table access.
- **Spring_Profile**: Spring Boot configuration profile (local, dev, test, staging, prod) controlling environment-specific settings.
- **Structured_Logging**: JSON-formatted log output with consistent fields (traceId, spanId, userId, requestId, timestamp, level, service name) enabling centralized log aggregation and correlation.
- **Quartz_Clustering**: Quartz Scheduler configuration using JDBC-based job store with `isClustered=true` to ensure only one pod executes each scheduled job across multiple replicas.

## Requirements

### Requirement 1: Common Library Project Setup

**User Story:** As a developer, I want a shared Java library with DTOs, Kafka event classes, exceptions, and utilities, so that both services use consistent data contracts and avoid code duplication.

#### Acceptance Criteria

1. THE Common_Library SHALL be a Maven project with `groupId` `gr.courtbooking`, `artifactId` `court-booking-common`, and Java 21 source/target compatibility.
2. THE Common_Library SHALL declare Spring Boot starter dependencies as `provided` scope so consuming services control their own Spring Boot version.
3. WHEN the Common_Library is built, THE Maven build SHALL produce a JAR artifact suitable for publishing to GitHub Packages.
4. THE Common_Library SHALL include a `distributionManagement` section in `pom.xml` configured for GitHub Packages under the repository owner's namespace.
5. THE Common_Library SHALL define a base package structure under `gr.courtbooking.common` with sub-packages: `dto`, `event`, `exception`, and `util`.

### Requirement 2: Common Library — DTO Classes

**User Story:** As a developer, I want shared DTO classes in the common library, so that both services serialize and deserialize API and inter-service data consistently.

#### Acceptance Criteria

1. THE Common_Library SHALL define DTO record classes for shared domain concepts including: `CourtSummaryDto`, `UserBasicDto`, `BookingDto`, `PaymentDto`, and `PricingRuleDto`.
2. THE Common_Library SHALL define enum types for shared domain values including: `CourtType` (TENNIS, PADEL, BASKETBALL, FOOTBALL_5X5), `LocationType` (INDOOR, OUTDOOR), `BookingStatus` (PENDING_CONFIRMATION, CONFIRMED, CANCELLED, COMPLETED, REJECTED), `PaymentStatus` (NOT_REQUIRED, PENDING, AUTHORIZED, CAPTURED, REFUNDED, PARTIALLY_REFUNDED, FAILED, PAID_EXTERNALLY), `UserRole` (CUSTOMER, COURT_OWNER, SUPPORT_AGENT, PLATFORM_ADMIN), `DayOfWeekEnum` (MONDAY through SUNDAY), and `Language` (EL, EN).
3. THE Common_Library SHALL annotate DTO classes with Jackson serialization annotations to ensure consistent JSON field naming (camelCase) across both services.
4. WHEN a DTO contains monetary amounts, THE Common_Library SHALL represent the amount as an `int` field with a name suffixed by `Cents` (e.g., `totalAmountCents`) to enforce the euro-cents convention.

### Requirement 3: Common Library — Kafka Event Classes

**User Story:** As a developer, I want shared Kafka event envelope and payload classes in the common library, so that producers and consumers use identical event schemas as defined in `kafka-event-contracts.json`.

#### Acceptance Criteria

1. THE Common_Library SHALL define an `EventEnvelope<T>` generic class with fields: `eventId` (UUID), `eventType` (String), `source` (String), `timestamp` (Instant), `traceId` (String), `spanId` (String), `correlationId` (UUID, nullable), and `payload` (T).
2. THE Common_Library SHALL define payload classes for each event type defined in `kafka-event-contracts.json`: `BookingCreatedPayload`, `BookingConfirmedPayload`, `BookingCancelledPayload`, `BookingModifiedPayload`, `BookingCompletedPayload`, `SlotHeldPayload`, `SlotReleasedPayload`, `NotificationRequestedPayload`, `CourtUpdatedPayload`, `PricingUpdatedPayload`, `AvailabilityUpdatedPayload`, `CancellationPolicyUpdatedPayload`, `CourtDeletedPayload`, `StripeConnectStatusChangedPayload`, `MatchCreatedPayload`, `MatchUpdatedPayload`, `MatchClosedPayload`, `WaitlistSlotFreedPayload`, `WaitlistHoldExpiredPayload`, `BookingAnalyticsPayload`, `RevenueAnalyticsPayload`, `PromoCodeRedeemedPayload`, and `SecurityAlertPayload`.
3. THE Common_Library SHALL define a `KafkaTopics` constants class listing all topic names: `booking-events`, `notification-events`, `court-update-events`, `match-events`, `waitlist-events`, `analytics-events`, and `security-events`.
4. WHEN an event payload class is serialized to JSON, THE Common_Library SHALL produce output conforming to the schema defined in `kafka-event-contracts.json` for that event type.
5. WHEN an event payload JSON conforming to `kafka-event-contracts.json` is deserialized, THE Common_Library SHALL produce the correct payload class instance with all fields populated.

### Requirement 4: Common Library — Exception Classes and Utilities

**User Story:** As a developer, I want shared exception types and utility classes in the common library, so that both services handle errors consistently and share common helper logic.

#### Acceptance Criteria

1. THE Common_Library SHALL define a base `CourtBookingException` class extending `RuntimeException` with fields for `errorCode` (String) and `httpStatus` (int).
2. THE Common_Library SHALL define specific exception subclasses: `ResourceNotFoundException`, `ConflictException`, `ValidationException`, `UnauthorizedException`, and `ForbiddenException`.
3. THE Common_Library SHALL define a `MoneyUtil` utility class with methods for safe euro-cents arithmetic (add, subtract, multiply by percentage, calculate platform fee).
4. THE Common_Library SHALL define an `ErrorResponse` record with fields: `errorCode` (String), `message` (String), `timestamp` (Instant), and `traceId` (String) for consistent API error responses.

### Requirement 5: Common Library — Publishing to GitHub Packages

**User Story:** As a developer, I want the common library published to GitHub Packages via CI/CD, so that both services can consume it as a versioned Maven dependency.

#### Acceptance Criteria

1. WHEN a commit is pushed to the `main` branch of the Common_Library repository, THE CI_CD_Pipeline SHALL build the library, run tests, and publish the artifact to GitHub Packages.
2. THE Common_Library SHALL use semantic versioning (major.minor.patch) for artifact versions.
3. WHEN a pull request is opened against the Common_Library repository, THE CI_CD_Pipeline SHALL compile the library and run all tests without publishing.
4. THE Common_Library SHALL include a GitHub Actions workflow file at `.github/workflows/ci.yml` that performs build, test, and conditional publish steps.
5. WHEN both services declare the Common_Library as a Maven dependency, THE Maven build SHALL resolve the artifact from GitHub Packages using repository credentials.

### Requirement 6: Spring Boot Project Scaffolding — Platform Service

**User Story:** As a developer, I want the Platform Service scaffolded as a Spring Boot 3.x application with hexagonal architecture, so that the codebase is structured for maintainability and testability from the start.

#### Acceptance Criteria

1. THE Platform_Service SHALL be a Maven project with `groupId` `gr.courtbooking`, `artifactId` `court-booking-platform-service`, Spring Boot 3.x parent, and Java 21 source/target compatibility.
2. THE Platform_Service SHALL declare Maven dependencies for: Spring Boot Web, Spring Boot Actuator, Spring Boot Data JPA, Spring Boot Data Redis, Spring Kafka, Flyway Core, PostgreSQL driver, PostGIS Hibernate Spatial dialect, springdoc-openapi, jqwik (test scope), and the Common_Library.
3. THE Platform_Service SHALL organize source code in a hexagonal architecture package structure under `gr.courtbooking.platform` with sub-packages: `domain` (entities, value objects), `domain.port.in` (use case interfaces), `domain.port.out` (repository interfaces), `application` (use case implementations), `adapter.in.web` (REST controllers), `adapter.in.kafka` (Kafka consumers), `adapter.out.persistence` (JPA repositories), `adapter.out.kafka` (Kafka producers), `adapter.out.redis` (Redis adapters), and `config` (Spring configuration classes).
4. THE Platform_Service SHALL include a Spring Boot main application class annotated with `@SpringBootApplication`.
5. THE Platform_Service SHALL include a placeholder REST controller at `/api/health/info` returning a JSON response with service name and version for manual verification.

### Requirement 7: Spring Boot Project Scaffolding — Transaction Service

**User Story:** As a developer, I want the Transaction Service scaffolded as a Spring Boot 3.x application with hexagonal architecture, so that the codebase is structured for maintainability and testability from the start.

#### Acceptance Criteria

1. THE Transaction_Service SHALL be a Maven project with `groupId` `gr.courtbooking`, `artifactId` `court-booking-transaction-service`, Spring Boot 3.x parent, and Java 21 source/target compatibility.
2. THE Transaction_Service SHALL declare Maven dependencies for: Spring Boot Web, Spring Boot Actuator, Spring Boot Data JPA, Spring Boot Data Redis, Spring Boot WebSocket, Spring Kafka, Flyway Core, PostgreSQL driver, Quartz Scheduler, springdoc-openapi, jqwik (test scope), and the Common_Library.
3. THE Transaction_Service SHALL organize source code in a hexagonal architecture package structure under `gr.courtbooking.transaction` with sub-packages: `domain` (entities, value objects), `domain.port.in` (use case interfaces), `domain.port.out` (repository interfaces), `application` (use case implementations), `adapter.in.web` (REST controllers), `adapter.in.kafka` (Kafka consumers), `adapter.in.websocket` (WebSocket handlers), `adapter.out.persistence` (JPA repositories), `adapter.out.kafka` (Kafka producers), `adapter.out.redis` (Redis adapters), `adapter.out.stripe` (Stripe client adapter), and `config` (Spring configuration classes).
4. THE Transaction_Service SHALL include a Spring Boot main application class annotated with `@SpringBootApplication`.
5. THE Transaction_Service SHALL include a placeholder REST controller at `/api/health/info` returning a JSON response with service name and version for manual verification.

### Requirement 8: Spring Boot Actuator Health Endpoints

**User Story:** As a DevOps engineer, I want both services to expose Spring Boot Actuator health endpoints, so that Kubernetes can perform liveness, readiness, and startup probes.

#### Acceptance Criteria

1. THE Platform_Service SHALL expose a liveness probe endpoint at `GET /actuator/health/liveness` that returns HTTP 200 when the application is running.
2. THE Platform_Service SHALL expose a readiness probe endpoint at `GET /actuator/health/readiness` that checks database connectivity, Redis connectivity, and Kafka broker availability, returning HTTP 200 only when all dependencies are reachable.
3. THE Platform_Service SHALL expose a startup probe endpoint at `GET /actuator/health` that returns HTTP 200 after the application has completed initialization including Flyway migrations.
4. THE Transaction_Service SHALL expose a liveness probe endpoint at `GET /actuator/health/liveness` that returns HTTP 200 when the application is running.
5. THE Transaction_Service SHALL expose a readiness probe endpoint at `GET /actuator/health/readiness` that checks database connectivity, Redis connectivity, and Kafka broker availability, returning HTTP 200 only when all dependencies are reachable.
6. THE Transaction_Service SHALL expose a startup probe endpoint at `GET /actuator/health` that returns HTTP 200 after the application has completed initialization including Flyway migrations.
7. WHEN a dependency (database, Redis, or Kafka) becomes unreachable, THE readiness probe SHALL return HTTP 503 within 10 seconds of the dependency becoming unavailable.

### Requirement 9: Spring Profiles Configuration

**User Story:** As a developer, I want environment-specific Spring profiles for local, dev, test, staging, and prod, so that each environment connects to the correct infrastructure resources with appropriate settings.

#### Acceptance Criteria

1. THE Platform_Service SHALL include Spring configuration files: `application.yml` (shared defaults), `application-local.yml`, `application-dev.yml`, `application-test.yml`, `application-staging.yml`, and `application-prod.yml`.
2. THE Transaction_Service SHALL include Spring configuration files: `application.yml` (shared defaults), `application-local.yml`, `application-dev.yml`, `application-test.yml`, `application-staging.yml`, and `application-prod.yml`.
3. WHEN the `local` profile is active, THE Platform_Service SHALL connect to PostgreSQL at `localhost:5432/courtbooking` with user `dev`, Redis at `localhost:6379`, and Kafka at `localhost:9092`.
4. WHEN the `local` profile is active, THE Transaction_Service SHALL connect to PostgreSQL at `localhost:5432/courtbooking` with user `dev`, Redis at `localhost:6379`, and Kafka at `localhost:9092`.
5. WHEN a cloud profile (dev, test, staging, prod) is active, THE Platform_Service SHALL read database URL, Redis URL, and Kafka bootstrap servers from environment variables (`SPRING_DATASOURCE_URL`, `SPRING_REDIS_HOST`, `SPRING_KAFKA_BOOTSTRAP_SERVERS`).
6. WHEN a cloud profile (dev, test, staging, prod) is active, THE Transaction_Service SHALL read database URL, Redis URL, and Kafka bootstrap servers from environment variables (`SPRING_DATASOURCE_URL`, `SPRING_REDIS_HOST`, `SPRING_KAFKA_BOOTSTRAP_SERVERS`).
7. THE Platform_Service SHALL configure Flyway to target the `platform` schema using `spring.flyway.schemas=platform` in all profiles.
8. THE Transaction_Service SHALL configure Flyway to target the `transaction` schema using `spring.flyway.schemas=transaction` in all profiles.
9. WHEN the `local` profile is active, THE Platform_Service SHALL set `spring.flyway.user` to `dev` and THE Transaction_Service SHALL set `spring.flyway.user` to `dev`.
10. WHEN a cloud profile is active, THE Platform_Service SHALL set `spring.flyway.user` to `platform_service` and THE Transaction_Service SHALL set `spring.flyway.user` to `transaction_service` to enforce schema ownership boundaries.
11. THE Platform_Service SHALL configure JPA to use the Hibernate Spatial dialect for PostGIS support in all profiles.

### Requirement 10: Flyway Database Migrations — Platform Schema

**User Story:** As a developer, I want Flyway migrations for the complete platform schema, so that all tables, indexes, constraints, and views are created consistently across environments as defined in `database-schema.md`.

#### Acceptance Criteria

1. THE Platform_Service SHALL include Flyway migration scripts under `src/main/resources/db/migration/platform/`.
2. THE Platform_Service SHALL include an initial migration (`V1__create_platform_schema.sql`) that creates all platform schema tables as defined in `database-schema.md`: `users`, `oauth_providers`, `refresh_tokens`, `verification_requests`, `courts`, `availability_windows`, `availability_overrides`, `holiday_templates`, `court_holiday_subscriptions`, `favorites`, `preferences`, `skill_levels`, `court_ratings` (Phase 2, created empty), `pricing_rules`, `special_date_pricing` (Phase 2, created empty), `cancellation_tiers`, `promo_codes` (Phase 2, created empty), `promo_code_redemptions` (Phase 2, created empty), `translations`, `feature_flags`, `support_tickets`, `support_messages`, `support_attachments`, `court_owner_audit_logs`, `reminder_rules`, `court_owner_notification_preferences`, `court_defaults`, `security_alerts`, `ip_blocklist`, and `failed_auth_attempts`.
3. THE initial migration SHALL create all indexes defined in `database-schema.md` for each platform table, including partial indexes with WHERE clauses, UNIQUE constraints, GIST indexes for PostGIS geometry columns, and EXCLUDE constraints for availability window overlap prevention.
4. THE initial migration SHALL create all CHECK constraints defined in `database-schema.md` for each platform table, including enum value checks, range checks, and cross-column validation constraints.
5. THE Platform_Service SHALL include a migration (`V2__create_cross_schema_views.sql`) that creates the cross-schema read-only views: `platform.v_court_summary`, `platform.v_user_basic`, `platform.v_court_cancellation_tiers`, and `platform.v_user_skill_level` as defined in `database-schema.md`, and grants SELECT on each view to the `transaction_service` role.
6. THE initial migration SHALL enable the PostGIS extension (`CREATE EXTENSION IF NOT EXISTS postgis`) before creating tables that use geometry columns.
7. WHEN Flyway migrations are executed, THE Platform_Service SHALL apply migrations only to the `platform` schema without modifying the `transaction` schema.
8. THE migration scripts SHALL store all monetary amounts as INTEGER columns representing euro cents, all event timestamps as TIMESTAMPTZ, booking dates as DATE, and booking times as TIME, consistent with the conventions in `database-schema.md`.

### Requirement 11: Flyway Database Migrations — Transaction Schema

**User Story:** As a developer, I want Flyway migrations for the complete transaction schema, so that all tables, indexes, and constraints are created consistently across environments as defined in `database-schema.md`.

#### Acceptance Criteria

1. THE Transaction_Service SHALL include Flyway migration scripts under `src/main/resources/db/migration/transaction/`.
2. THE Transaction_Service SHALL include an initial migration (`V1__create_transaction_schema.sql`) that creates all transaction schema tables as defined in `database-schema.md`: `bookings`, `payments`, `audit_logs`, `notifications`, `device_tokens`, `waitlists` (Phase 2, created empty), `open_matches` (Phase 2, created empty), `match_players` (Phase 2, created empty), `match_join_requests` (Phase 2, created empty), `match_messages` (Phase 2, created empty), `split_payments` (Phase 2, created empty), and `split_payment_shares` (Phase 2, created empty).
3. THE initial migration SHALL create all indexes defined in `database-schema.md` for each transaction table, including partial indexes with WHERE clauses and UNIQUE constraints.
4. THE initial migration SHALL create all CHECK constraints defined in `database-schema.md` for each transaction table, including enum value checks and cross-column validation constraints.
5. THE Transaction_Service SHALL include a migration (`V2__create_quartz_tables.sql`) that creates the Quartz Scheduler JDBC job store tables using the standard Quartz PostgreSQL DDL script, configured for clustered mode.
6. WHEN Flyway migrations are executed, THE Transaction_Service SHALL apply migrations only to the `transaction` schema without modifying the `platform` schema.
7. THE migration scripts SHALL store all monetary amounts as INTEGER columns representing euro cents, all event timestamps as TIMESTAMPTZ, booking dates as DATE, and booking times as TIME, consistent with the conventions in `database-schema.md`.

### Requirement 12: Dockerfiles — Multi-Stage Builds

**User Story:** As a DevOps engineer, I want multi-stage Dockerfiles for both services, so that production images are minimal, secure, and built reproducibly.

#### Acceptance Criteria

1. THE Platform_Service SHALL include a `Dockerfile` at the repository root that uses a multi-stage build: a Maven build stage (Java 21 base) and a runtime stage (Eclipse Temurin 21 JRE slim base).
2. THE Transaction_Service SHALL include a `Dockerfile` at the repository root that uses a multi-stage build: a Maven build stage (Java 21 base) and a runtime stage (Eclipse Temurin 21 JRE slim base).
3. WHEN the build stage executes, THE Dockerfile SHALL copy `pom.xml` first and resolve dependencies before copying source code, to maximize Docker layer caching.
4. THE runtime stage SHALL run the application as a non-root user for security.
5. THE runtime stage SHALL expose port 8080 and set the Spring Boot server port to 8080.
6. THE runtime stage SHALL configure JVM memory settings via `JAVA_OPTS` environment variable with sensible defaults for container environments (e.g., `-XX:MaxRAMPercentage=75.0`).
7. THE Dockerfile SHALL include a `HEALTHCHECK` instruction that calls the actuator health endpoint.

### Requirement 13: CI/CD Pipeline — Platform Service

**User Story:** As a developer, I want a GitHub Actions CI/CD pipeline for the Platform Service, so that every code change is automatically built, tested, scanned, and deployed through environments with appropriate gates.

#### Acceptance Criteria

1. THE Platform_Service SHALL include a GitHub Actions workflow at `.github/workflows/ci.yml` triggered on pull requests to the `develop` and `main` branches.
2. WHEN a pull request is opened, THE CI_CD_Pipeline SHALL execute: checkout, Java 21 setup, Maven dependency resolution (with GitHub Packages authentication for Common_Library), compile, unit tests, property-based tests (jqwik), Flyway migration validation (`flyway validate` against a temporary PostgreSQL container), and container image vulnerability scan (Trivy).
3. THE Platform_Service SHALL include a GitHub Actions workflow at `.github/workflows/deploy.yml` triggered on push to the `develop` branch.
4. WHEN code is pushed to `develop`, THE CI_CD_Pipeline SHALL build a Docker image, tag it with the Git commit SHA, push it to DOCR, and deploy to the `dev` Kubernetes namespace.
5. WHEN the `dev` deployment succeeds, THE CI_CD_Pipeline SHALL automatically deploy to the `test` Kubernetes namespace.
6. WHEN the `test` deployment succeeds, THE CI_CD_Pipeline SHALL trigger the cross-repo QA functional regression suite in the `court-booking-qa` repository via `repository_dispatch` event, passing service name, version (commit SHA), and target environment as payload.
7. WHEN QA passes, THE CI_CD_Pipeline SHALL deploy to the `staging` Kubernetes namespace after manual approval via a GitHub environment protection rule.
8. WHEN staging deployment succeeds, THE CI_CD_Pipeline SHALL trigger the QA smoke and stress suite in the `court-booking-qa` repository via `repository_dispatch` event.
9. WHEN staging validation passes, THE CI_CD_Pipeline SHALL tag the image as `rc-v{version}` and allow manual production deployment that promotes the tag to `v{version}`.
10. THE CI_CD_Pipeline SHALL use GitHub environments (`dev`, `test`, `staging`, `production`) with appropriate protection rules: no approval for dev/test, required reviewer approval for staging/production.

### Requirement 14: CI/CD Pipeline — Transaction Service

**User Story:** As a developer, I want a GitHub Actions CI/CD pipeline for the Transaction Service, so that every code change is automatically built, tested, scanned, and deployed through environments with appropriate gates.

#### Acceptance Criteria

1. THE Transaction_Service SHALL include a GitHub Actions workflow at `.github/workflows/ci.yml` triggered on pull requests to the `develop` and `main` branches.
2. WHEN a pull request is opened, THE CI_CD_Pipeline SHALL execute: checkout, Java 21 setup, Maven dependency resolution (with GitHub Packages authentication for Common_Library), compile, unit tests, property-based tests (jqwik), Flyway migration validation (`flyway validate` against a temporary PostgreSQL container), and container image vulnerability scan (Trivy).
3. THE Transaction_Service SHALL include a GitHub Actions workflow at `.github/workflows/deploy.yml` triggered on push to the `develop` branch.
4. WHEN code is pushed to `develop`, THE CI_CD_Pipeline SHALL build a Docker image, tag it with the Git commit SHA, push it to DOCR, and deploy to the `dev` Kubernetes namespace.
5. WHEN the `dev` deployment succeeds, THE CI_CD_Pipeline SHALL automatically deploy to the `test` Kubernetes namespace.
6. WHEN the `test` deployment succeeds, THE CI_CD_Pipeline SHALL trigger the cross-repo QA functional regression suite in the `court-booking-qa` repository via `repository_dispatch` event, passing service name, version (commit SHA), and target environment as payload.
7. WHEN QA passes, THE CI_CD_Pipeline SHALL deploy to the `staging` Kubernetes namespace after manual approval via a GitHub environment protection rule.
8. WHEN staging deployment succeeds, THE CI_CD_Pipeline SHALL trigger the QA smoke and stress suite in the `court-booking-qa` repository via `repository_dispatch` event.
9. WHEN staging validation passes, THE CI_CD_Pipeline SHALL tag the image as `rc-v{version}` and allow manual production deployment that promotes the tag to `v{version}`.
10. THE CI_CD_Pipeline SHALL use GitHub environments (`dev`, `test`, `staging`, `production`) with appropriate protection rules: no approval for dev/test, required reviewer approval for staging/production.

### Requirement 15: Kubernetes Deployment Manifests — Platform Service

**User Story:** As a DevOps engineer, I want Kubernetes deployment manifests for the Platform Service, so that the service can be deployed to DOKS with proper resource limits, health probes, and scaling configuration.

#### Acceptance Criteria

1. THE Platform_Service SHALL include a Kubernetes `Deployment` manifest that specifies: container image from DOCR, resource requests (cpu: 250m, memory: 512Mi) and limits (cpu: 500m, memory: 1Gi), and replica count of 1 for dev/test, 2 for staging/production.
2. THE Deployment manifest SHALL configure a liveness probe pointing to `GET /actuator/health/liveness` with `initialDelaySeconds: 30`, `periodSeconds: 10`, and `failureThreshold: 3`.
3. THE Deployment manifest SHALL configure a readiness probe pointing to `GET /actuator/health/readiness` with `initialDelaySeconds: 15`, `periodSeconds: 5`, and `failureThreshold: 3`.
4. THE Deployment manifest SHALL configure a startup probe pointing to `GET /actuator/health` with `initialDelaySeconds: 60`, `periodSeconds: 10`, and `failureThreshold: 12` to allow time for Flyway migrations.
5. THE Platform_Service SHALL include a Kubernetes `Service` manifest of type `ClusterIP` exposing port 8080.
6. THE Deployment manifest SHALL inject environment variables from a ConfigMap and secrets (database URL, Redis host, Kafka bootstrap servers, Spring active profile).
7. THE Deployment manifest SHALL set the `JAVA_OPTS` environment variable with container-appropriate JVM settings.

### Requirement 16: Kubernetes Deployment Manifests — Transaction Service

**User Story:** As a DevOps engineer, I want Kubernetes deployment manifests for the Transaction Service, so that the service can be deployed to DOKS with proper resource limits, health probes, and scaling configuration.

#### Acceptance Criteria

1. THE Transaction_Service SHALL include a Kubernetes `Deployment` manifest that specifies: container image from DOCR, resource requests (cpu: 250m, memory: 512Mi) and limits (cpu: 500m, memory: 1Gi), and replica count of 1 for dev/test, 2 for staging/production.
2. THE Deployment manifest SHALL configure a liveness probe pointing to `GET /actuator/health/liveness` with `initialDelaySeconds: 30`, `periodSeconds: 10`, and `failureThreshold: 3`.
3. THE Deployment manifest SHALL configure a readiness probe pointing to `GET /actuator/health/readiness` with `initialDelaySeconds: 15`, `periodSeconds: 5`, and `failureThreshold: 3`.
4. THE Deployment manifest SHALL configure a startup probe pointing to `GET /actuator/health` with `initialDelaySeconds: 60`, `periodSeconds: 10`, and `failureThreshold: 12` to allow time for Flyway migrations and Quartz table initialization.
5. THE Transaction_Service SHALL include a Kubernetes `Service` manifest of type `ClusterIP` exposing port 8080.
6. THE Deployment manifest SHALL inject environment variables from a ConfigMap and secrets (database URL, Redis host, Kafka bootstrap servers, Stripe API keys, Spring active profile).
7. THE Deployment manifest SHALL set the `JAVA_OPTS` environment variable with container-appropriate JVM settings.

### Requirement 17: Kubernetes Ingress Rules

**User Story:** As a DevOps engineer, I want Kubernetes Ingress rules for path-based routing, so that API requests are routed to the correct service based on URL path.

#### Acceptance Criteria

1. THE Kubernetes manifests SHALL include an Ingress resource that routes requests with paths `/api/auth/*`, `/api/users/*`, `/api/courts/*`, `/api/weather/*`, `/api/analytics/*`, `/api/promo-codes/*`, `/api/feature-flags/*`, `/api/admin/*`, and `/api/support/*` to the Platform_Service.
2. THE Kubernetes manifests SHALL include an Ingress resource that routes requests with paths `/api/bookings/*`, `/api/payments/*`, `/api/notifications/*`, `/api/waitlist/*`, `/api/matches/*`, and `/api/split-payments/*` to the Transaction_Service.
3. THE Ingress resource SHALL route WebSocket upgrade requests at `/ws/*` to the Transaction_Service.
4. THE Ingress resource SHALL use the `nginx.ingress.kubernetes.io` annotation class to integrate with the existing NGINX Ingress Controller deployed in Phase 1a.
5. THE Ingress resource SHALL configure TLS termination using a cert-manager ClusterIssuer annotation for automatic Let's Encrypt certificate provisioning.
6. THE Ingress resource SHALL be defined per namespace (dev, test, staging, production) with environment-specific hostnames.

### Requirement 18: Kubernetes ConfigMaps

**User Story:** As a DevOps engineer, I want Kubernetes ConfigMaps for each service per environment, so that non-sensitive configuration is managed declaratively and separately from secrets.

#### Acceptance Criteria

1. THE Kubernetes manifests SHALL include a ConfigMap for the Platform_Service in each namespace (dev, test, staging, production) containing: `SPRING_PROFILES_ACTIVE`, `SPRING_DATASOURCE_URL`, `SPRING_REDIS_HOST`, `SPRING_KAFKA_BOOTSTRAP_SERVERS`, `SERVER_PORT=8080`, and `JAVA_OPTS`.
2. THE Kubernetes manifests SHALL include a ConfigMap for the Transaction_Service in each namespace (dev, test, staging, production) containing: `SPRING_PROFILES_ACTIVE`, `SPRING_DATASOURCE_URL`, `SPRING_REDIS_HOST`, `SPRING_KAFKA_BOOTSTRAP_SERVERS`, `SERVER_PORT=8080`, and `JAVA_OPTS`.
3. WHEN the ConfigMap values differ between environments, THE manifests SHALL use Kustomize overlays to manage per-environment overrides from a shared base.

### Requirement 19: Local Development Workflow

**User Story:** As a developer, I want a working local development workflow, so that I can run both services locally against Docker Compose infrastructure and iterate quickly with hot reload.

#### Acceptance Criteria

1. WHEN a developer runs `docker compose up -d` from the `infrastructure/` directory, THE local environment SHALL start PostgreSQL 15 with PostGIS 3.3 on port 5432, Redis 7 on port 6379, and Kafka (Confluent) on port 9092, all with health checks passing.
2. WHEN the PostgreSQL container starts for the first time, THE init script SHALL create the `platform` and `transaction` schemas, grant permissions to the `dev` user, and enable the PostGIS extension.
3. WHEN a developer starts the Platform_Service with the `local` profile, THE service SHALL connect to the local Docker Compose infrastructure, run Flyway migrations against the `platform` schema, and start successfully with actuator health endpoints returning HTTP 200.
4. WHEN a developer starts the Transaction_Service with the `local` profile, THE service SHALL connect to the local Docker Compose infrastructure, run Flyway migrations against the `transaction` schema, and start successfully with actuator health endpoints returning HTTP 200.
5. WHEN both services are running locally, THE Platform_Service SHALL be accessible at `http://localhost:8081` and THE Transaction_Service SHALL be accessible at `http://localhost:8082` (different ports to allow simultaneous local execution).
6. WHEN a developer modifies Java source code while the service is running with Spring Boot DevTools, THE service SHALL automatically restart to reflect the changes.

### Requirement 20: Flyway Migration Validation in CI

**User Story:** As a developer, I want Flyway migrations validated in CI on every pull request, so that migration errors are caught before merging.

#### Acceptance Criteria

1. WHEN a pull request is opened for the Platform_Service, THE CI_CD_Pipeline SHALL start a temporary PostgreSQL container with PostGIS, create the `platform` schema, run `flyway migrate` to apply all migrations, and then run `flyway validate` to confirm migration checksums and ordering.
2. WHEN a pull request is opened for the Transaction_Service, THE CI_CD_Pipeline SHALL start a temporary PostgreSQL container with PostGIS, create the `platform` and `transaction` schemas (Transaction_Service needs the platform schema to exist for cross-schema view references), run Platform_Service migrations first, then run `flyway migrate` for the transaction schema, and then run `flyway validate`.
3. IF a Flyway migration fails validation (checksum mismatch, out-of-order version, SQL syntax error), THEN THE CI_CD_Pipeline SHALL fail the pull request check and report the specific migration error.

### Requirement 21: Container Image Security Scanning

**User Story:** As a DevOps engineer, I want container images scanned for vulnerabilities in CI, so that known security issues are detected before deployment.

#### Acceptance Criteria

1. WHEN a Docker image is built in the CI_CD_Pipeline for either service, THE pipeline SHALL scan the image using Trivy for HIGH and CRITICAL severity vulnerabilities.
2. IF Trivy detects any CRITICAL severity vulnerabilities, THEN THE CI_CD_Pipeline SHALL fail the build and report the vulnerability details.
3. IF Trivy detects HIGH severity vulnerabilities, THEN THE CI_CD_Pipeline SHALL report the vulnerabilities as warnings without failing the build.
4. THE CI_CD_Pipeline SHALL upload the Trivy scan results as a build artifact for audit purposes.

### Requirement 22: Cross-Repo QA Trigger Integration

**User Story:** As a developer, I want service deployments to automatically trigger QA test suites in the `court-booking-qa` repository, so that functional regressions are caught before promotion to higher environments.

#### Acceptance Criteria

1. WHEN the Platform_Service is deployed to the `test` environment, THE CI_CD_Pipeline SHALL trigger the `court-booking-qa` repository's `run-functional-regression` workflow via `repository_dispatch`, passing `service: court-booking-platform-service`, `version: {commit_sha}`, `environment: test`, `triggered_by: {actor}`, and `run_url: {workflow_run_url}` as payload.
2. WHEN the Transaction_Service is deployed to the `test` environment, THE CI_CD_Pipeline SHALL trigger the `court-booking-qa` repository's `run-functional-regression` workflow via `repository_dispatch`, passing `service: court-booking-transaction-service`, `version: {commit_sha}`, `environment: test`, `triggered_by: {actor}`, and `run_url: {workflow_run_url}` as payload.
3. WHEN either service is deployed to the `staging` environment, THE CI_CD_Pipeline SHALL trigger the `court-booking-qa` repository's `run-staging-validation` workflow via `repository_dispatch` with the same payload structure.
4. THE CI_CD_Pipeline SHALL use the `peter-evans/repository-dispatch@v3` GitHub Action with a `QA_DISPATCH_TOKEN` secret (GitHub PAT with `repo` scope) for cross-repo dispatch.

### Requirement 23: Maven Build Configuration Consistency

**User Story:** As a developer, I want consistent Maven build configuration across all three repositories, so that builds are reproducible and dependency versions are aligned.

#### Acceptance Criteria

1. THE Platform_Service and Transaction_Service SHALL use the same Spring Boot parent version (3.x latest stable) and Java 21 compiler settings.
2. THE Platform_Service and Transaction_Service SHALL declare the Common_Library dependency with a specific version (not SNAPSHOT for releases) and configure the GitHub Packages Maven repository for resolution.
3. THE Platform_Service and Transaction_Service SHALL include the `maven-surefire-plugin` configured to run both JUnit 5 and jqwik property-based tests.
4. THE Platform_Service and Transaction_Service SHALL include the `spring-boot-maven-plugin` for building executable JARs.
5. THE Common_Library, Platform_Service, and Transaction_Service SHALL all use the same Java 21 source and target compatibility settings.

### Requirement 24: Kubernetes Manifest Organization

**User Story:** As a DevOps engineer, I want Kubernetes manifests organized with Kustomize base and overlays, so that environment-specific configuration is managed cleanly without duplication.

#### Acceptance Criteria

1. THE Kubernetes manifests for each service SHALL be organized in a Kustomize structure with a `base/` directory containing shared Deployment, Service, and Ingress templates, and `overlays/` directories for each environment (dev, test, staging, production).
2. WHEN deploying to a specific environment, THE CI_CD_Pipeline SHALL use `kubectl apply -k overlays/{environment}/` to apply the environment-specific overlay.
3. THE environment overlays SHALL customize: replica count, resource limits, ConfigMap values, image tags, and Ingress hostnames.
4. THE Kubernetes manifests SHALL reside within each service repository under a `k8s/` directory, so that application manifests are versioned alongside application code.

### Requirement 25: Database User Separation for Cloud Environments

**User Story:** As a DevOps engineer, I want separate database users for each service in cloud environments, so that schema ownership boundaries are enforced at the database level.

#### Acceptance Criteria

1. WHEN running in a cloud profile (dev, test, staging, prod), THE Platform_Service SHALL connect to PostgreSQL using the `platform_service` database user which has full privileges on the `platform` schema and no write access to the `transaction` schema.
2. WHEN running in a cloud profile (dev, test, staging, prod), THE Transaction_Service SHALL connect to PostgreSQL using the `transaction_service` database user which has full privileges on the `transaction` schema and read-only access to the `platform` schema cross-schema views.
3. THE database user credentials for cloud environments SHALL be stored as Kubernetes Secrets (SealedSecrets encrypted in Git) and injected into pods as environment variables.
4. WHEN running in the `local` profile, both services SHALL use the shared `dev` database user for simplicity.

### Requirement 26: Service Startup and Graceful Shutdown

**User Story:** As a DevOps engineer, I want both services to start up reliably and shut down gracefully, so that rolling deployments do not cause request failures or data loss.

#### Acceptance Criteria

1. WHEN the Platform_Service starts, THE service SHALL complete Flyway migrations before accepting HTTP traffic, ensuring the readiness probe returns HTTP 503 until migrations are complete.
2. WHEN the Transaction_Service starts, THE service SHALL complete Flyway migrations and Quartz Scheduler initialization before accepting HTTP traffic, ensuring the readiness probe returns HTTP 503 until initialization is complete.
3. WHEN a SIGTERM signal is received (Kubernetes pod termination), THE Platform_Service SHALL stop accepting new requests, complete in-flight requests within a 30-second grace period, and then shut down.
4. WHEN a SIGTERM signal is received (Kubernetes pod termination), THE Transaction_Service SHALL stop accepting new requests, complete in-flight requests within a 30-second grace period, and then shut down.
5. THE Kubernetes Deployment manifests SHALL set `terminationGracePeriodSeconds: 45` to allow the 30-second application grace period plus buffer.
6. BOTH services SHALL configure `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=30s` in the shared `application.yml` to enable Spring Boot's built-in graceful shutdown support.


### Requirement 27: Structured JSON Logging Configuration

**User Story:** As a developer and DevOps engineer, I want both services to produce structured JSON logs from day one, so that logs are ready for centralized aggregation in Loki and correlation via trace IDs when the observability stack is deployed.

#### Acceptance Criteria

1. BOTH services SHALL configure structured JSON log output using Logback with a JSON encoder (e.g., `logstash-logback-encoder`) as the default log format in all profiles.
2. EVERY log entry SHALL include these top-level fields: `timestamp` (ISO 8601), `level`, `logger`, `message`, `service` (service name), `traceId` (from MDC, empty if not in a request context), `spanId` (from MDC), and `requestId` (from `X-Request-ID` header if present).
3. WHEN a request is received, BOTH services SHALL extract the `X-Request-ID` header (if present) and store it in the MDC as `requestId` for inclusion in all log entries during that request.
4. WHEN running in the `local` profile, THE services MAY use a human-readable console log format (non-JSON) for developer convenience, while cloud profiles SHALL use JSON format.
5. BOTH services SHALL configure log levels: `INFO` as default, `DEBUG` for `gr.courtbooking` packages in local/dev profiles, `WARN` for noisy third-party libraries (Hibernate SQL, Kafka internals) in all profiles.
6. BOTH services SHALL sanitize logs to exclude sensitive data: passwords, tokens, credit card numbers, and full email addresses SHALL NOT appear in log output. PII fields SHALL be masked (e.g., `k***@gmail.com`).

### Requirement 28: Quartz Scheduler Clustering Configuration

**User Story:** As a developer, I want the Transaction Service's Quartz Scheduler configured for clustered mode from the start, so that scheduled jobs execute exactly once even when multiple pod replicas are running.

#### Acceptance Criteria

1. THE Transaction_Service SHALL configure Quartz Scheduler to use the JDBC-based job store (`org.quartz.impl.jdbcjobstore.JobStoreTX`) backed by the `transaction` schema Quartz tables created in Requirement 11.5.
2. THE Transaction_Service SHALL set `org.quartz.jobStore.isClustered=true` and `org.quartz.jobStore.clusterCheckinInterval=20000` in the Quartz configuration.
3. THE Transaction_Service SHALL configure `org.quartz.scheduler.instanceId=AUTO` so each pod replica gets a unique scheduler instance ID.
4. THE Transaction_Service SHALL configure the Quartz thread pool with a configurable thread count (default: 10) via `org.quartz.threadPool.threadCount`.
5. WHEN multiple Transaction_Service pods are running, THE Quartz Scheduler SHALL ensure each scheduled job fires on exactly one pod (no duplicate execution).

### Requirement 29: Sample Data Seeding for Local Development

**User Story:** As a developer, I want sample data automatically seeded in the local database, so that I can start developing and testing features immediately without manually creating test data.

#### Acceptance Criteria

1. THE local development Docker Compose setup SHALL include a data seeding mechanism that populates both schemas with representative sample data after Flyway migrations complete.
2. THE sample data SHALL include: at least 2 users (1 CUSTOMER, 1 COURT_OWNER), at least 3 courts of different types (Tennis, Padel, Basketball) with availability windows, at least 2 bookings in different statuses (CONFIRMED, COMPLETED), and sample pricing rules and cancellation tiers.
3. THE seeding mechanism SHALL be idempotent — running it multiple times SHALL NOT create duplicate data.
4. THE seeding mechanism SHALL use a Spring Boot profile-specific `data.sql` or a dedicated `V999__seed_local_data.sql` Flyway migration that only runs in the `local` profile (using Flyway's `spring.flyway.locations` with profile-conditional paths).
5. THE sample data SHALL use realistic but clearly fake values (e.g., "Demo Tennis Club", "test@example.com") to avoid confusion with real data.

### Requirement 30: Service Repository Documentation

**User Story:** As a developer joining the project, I want clear README documentation in each service repository, so that I can set up my local environment and understand the project structure quickly.

#### Acceptance Criteria

1. THE Platform_Service repository SHALL include a `README.md` at the root with: project overview, prerequisites (Java 21, Maven, Docker), local setup instructions (Docker Compose + Spring Boot), how to run tests, project structure explanation (hexagonal architecture packages), and links to the API specification and database schema documentation.
2. THE Transaction_Service repository SHALL include a `README.md` at the root with: project overview, prerequisites, local setup instructions, how to run tests, project structure explanation, and links to the API specification and database schema documentation.
3. THE Common_Library repository SHALL include a `README.md` at the root with: library overview, how to build and publish, how to consume as a Maven dependency (including GitHub Packages authentication), and package structure explanation.
4. EACH README SHALL include a "Quick Start" section that gets a developer from zero to running services in under 5 minutes.

### Requirement 31: springdoc-openapi Configuration

**User Story:** As a developer, I want springdoc-openapi configured from the start, so that API documentation is auto-generated from annotated controllers and available for validation as endpoints are implemented in later phases.

#### Acceptance Criteria

1. BOTH services SHALL include springdoc-openapi dependency and configure it to auto-generate OpenAPI 3.1 documentation from Spring MVC annotated controllers.
2. WHEN a service is running, THE OpenAPI specification SHALL be accessible at `/v3/api-docs` (JSON) and the Swagger UI at `/swagger-ui.html`.
3. THE springdoc configuration SHALL set the API info title, version, and description matching the service name and current version.
4. WHEN running in the `prod` profile, THE Swagger UI endpoint SHALL be disabled for security (only `/v3/api-docs` remains available for internal tooling).
5. THE springdoc configuration SHALL be placed in the `config` package of each service as a Spring `@Configuration` class.
