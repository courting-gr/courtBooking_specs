# Design Document: Phase 1b — Service Scaffolding

## Overview

This design document provides implementation-ready technical specifications for Phase 1b — Service Scaffolding. The phase delivers three repositories:

1. **court-booking-common** — Shared Java library with DTOs, Kafka event classes, exceptions, and utilities
2. **court-booking-platform-service** — Platform Service (auth, users, courts, availability, analytics)
3. **court-booking-transaction-service** — Transaction Service (bookings, payments, notifications, scheduled jobs)

The deliverable is both services running on DigitalOcean Kubernetes (DOKS) — empty but healthy — with CI/CD pipelines green, local development environment working, and Flyway migrations applied.

### Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.x |
| Build | Maven |
| Database | PostgreSQL 15 + PostGIS 3.3 |
| Messaging | Kafka (Redpanda Serverless) |
| Cache | Redis 7 |
| Container Registry | DigitalOcean Container Registry (DOCR) |
| Orchestration | DigitalOcean Kubernetes (DOKS) |
| CI/CD | GitHub Actions |
| Migrations | Flyway |
| K8s Manifests | Kustomize |
| Testing | JUnit 5, jqwik (property-based) |
| Logging | Logback with JSON encoder |
| Scheduling | Quartz Scheduler (clustered) |


## Architecture

### Repository Structure

```
court-booking-common/
├── pom.xml
├── src/main/java/gr/courtbooking/common/
│   ├── dto/
│   ├── event/
│   ├── exception/
│   └── util/
├── src/test/java/
└── .github/workflows/ci.yml

court-booking-platform-service/
├── pom.xml
├── Dockerfile
├── src/main/java/gr/courtbooking/platform/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   └── config/
├── src/main/resources/
│   ├── application.yml
│   ├── application-{profile}.yml
│   ├── db/migration/platform/
│   └── logback-spring.xml
├── src/test/java/
├── k8s/
│   ├── base/
│   └── overlays/
└── .github/workflows/

court-booking-transaction-service/
├── pom.xml
├── Dockerfile
├── src/main/java/gr/courtbooking/transaction/
│   ├── domain/
│   ├── application/
│   ├── adapter/
│   └── config/
├── src/main/resources/
│   ├── application.yml
│   ├── application-{profile}.yml
│   ├── db/migration/transaction/
│   └── logback-spring.xml
├── src/test/java/
├── k8s/
│   ├── base/
│   └── overlays/
└── .github/workflows/
```

### Hexagonal Architecture Package Structure

Both services follow the hexagonal (ports & adapters) architecture pattern:

```
gr.courtbooking.{platform|transaction}/
├── domain/
│   ├── model/              # Entities, Value Objects, Enums
│   ├── exception/          # Domain-specific exceptions
│   └── service/            # Domain services (cross-entity logic)
├── application/
│   ├── port/
│   │   ├── in/             # Use case interfaces (incoming ports)
│   │   └── out/            # Repository interfaces (outgoing ports/SPI)
│   └── service/            # Use case implementations
├── adapter/
│   ├── in/
│   │   ├── web/            # REST controllers + DTOs + mappers
│   │   ├── kafka/          # Kafka consumers
│   │   └── websocket/      # WebSocket handlers (Transaction only)
│   └── out/
│       ├── persistence/    # JPA entities + repositories + adapters
│       ├── kafka/          # Kafka producers
│       ├── redis/          # Redis cache adapters
│       └── stripe/         # Stripe client (Transaction only)
└── config/                 # Spring @Configuration classes
```


## Components and Interfaces

### 1. Common Library (court-booking-common)

#### 1.1 Maven Configuration (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>gr.courtbooking</groupId>
    <artifactId>court-booking-common</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Court Booking Common Library</name>
    <description>Shared DTOs, Kafka events, exceptions, and utilities</description>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.3.0</spring-boot.version>
        <jackson.version>2.17.0</jackson.version>
    </properties>

    <dependencies>
        <!-- Spring Boot starters as provided scope -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-json</artifactId>
            <version>${spring-boot.version}</version>
            <scope>provided</scope>
        </dependency>
        
        <!-- Jackson annotations for JSON serialization -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>${jackson.version}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
            <version>${jackson.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Test dependencies -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>net.jqwik</groupId>
            <artifactId>jqwik</artifactId>
            <version>1.8.4</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <distributionManagement>
        <repository>
            <id>github</id>
            <name>GitHub Packages</name>
            <url>https://maven.pkg.github.com/OWNER/court-booking-common</url>
        </repository>
    </distributionManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <source>21</source>
                    <target>21</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```


#### 1.2 DTO Classes

**Enums (gr.courtbooking.common.dto)**

```java
public enum CourtType {
    TENNIS, PADEL, BASKETBALL, FOOTBALL_5X5
}

public enum LocationType {
    INDOOR, OUTDOOR
}

public enum BookingStatus {
    PENDING_CONFIRMATION, CONFIRMED, CANCELLED, COMPLETED, REJECTED
}

public enum PaymentStatus {
    NOT_REQUIRED, PENDING, AUTHORIZED, CAPTURED, REFUNDED, 
    PARTIALLY_REFUNDED, FAILED, PAID_EXTERNALLY
}

public enum UserRole {
    CUSTOMER, COURT_OWNER, SUPPORT_AGENT, PLATFORM_ADMIN
}

public enum DayOfWeekEnum {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

public enum Language {
    EL, EN
}
```

**DTO Records**

```java
package gr.courtbooking.common.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.UUID;

public record CourtSummaryDto(
    UUID id,
    UUID ownerId,
    String nameEl,
    String nameEn,
    CourtType courtType,
    LocationType locationType,
    String timezone,
    @JsonProperty("basePriceCents") int basePriceCents,
    int durationMinutes,
    int maxCapacity,
    String confirmationMode,
    boolean visible
) {}

public record UserBasicDto(
    UUID id,
    String email,
    String name,
    String phone,
    UserRole role,
    Language language,
    String stripeConnectAccountId,
    String stripeConnectStatus,
    String stripeCustomerId,
    String status
) {}

public record BookingDto(
    UUID id,
    UUID courtId,
    UUID userId,
    String date,
    String startTime,
    String endTime,
    int durationMinutes,
    BookingStatus status,
    String bookingType,
    @JsonProperty("totalAmountCents") Integer totalAmountCents,
    @JsonProperty("platformFeeCents") Integer platformFeeCents,
    @JsonProperty("courtOwnerNetCents") Integer courtOwnerNetCents,
    PaymentStatus paymentStatus
) {}

public record PaymentDto(
    UUID id,
    UUID bookingId,
    UUID userId,
    @JsonProperty("amountCents") int amountCents,
    @JsonProperty("platformFeeCents") int platformFeeCents,
    @JsonProperty("courtOwnerNetCents") int courtOwnerNetCents,
    String currency,
    String status,
    String stripePaymentIntentId
) {}

public record PricingRuleDto(
    UUID id,
    UUID courtId,
    DayOfWeekEnum dayOfWeek,
    String startTime,
    String endTime,
    double multiplier,
    String label
) {}
```


#### 1.3 Kafka Event Classes

**Event Envelope**

```java
package gr.courtbooking.common.event;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.time.Instant;
import java.util.UUID;

public record EventEnvelope<T>(
    @JsonProperty("eventId") UUID eventId,
    @JsonProperty("eventType") String eventType,
    @JsonProperty("source") String source,
    @JsonProperty("timestamp") Instant timestamp,
    @JsonProperty("traceId") String traceId,
    @JsonProperty("spanId") String spanId,
    @JsonProperty("correlationId") UUID correlationId,
    @JsonProperty("payload") T payload
) {
    public EventEnvelope {
        if (eventId == null) throw new IllegalArgumentException("eventId required");
        if (eventType == null || eventType.isBlank()) throw new IllegalArgumentException("eventType required");
        if (source == null || source.isBlank()) throw new IllegalArgumentException("source required");
        if (timestamp == null) throw new IllegalArgumentException("timestamp required");
        if (traceId == null || traceId.isBlank()) throw new IllegalArgumentException("traceId required");
        if (spanId == null || spanId.isBlank()) throw new IllegalArgumentException("spanId required");
        if (payload == null) throw new IllegalArgumentException("payload required");
    }
    
    public static <T> EventEnvelope<T> create(String eventType, String source, 
            String traceId, String spanId, UUID correlationId, T payload) {
        return new EventEnvelope<>(
            UUID.randomUUID(),
            eventType,
            source,
            Instant.now(),
            traceId,
            spanId,
            correlationId,
            payload
        );
    }
}
```

**Kafka Topics Constants**

```java
package gr.courtbooking.common.event;

public final class KafkaTopics {
    private KafkaTopics() {}
    
    public static final String BOOKING_EVENTS = "booking-events";
    public static final String NOTIFICATION_EVENTS = "notification-events";
    public static final String COURT_UPDATE_EVENTS = "court-update-events";
    public static final String MATCH_EVENTS = "match-events";
    public static final String WAITLIST_EVENTS = "waitlist-events";
    public static final String ANALYTICS_EVENTS = "analytics-events";
    public static final String SECURITY_EVENTS = "security-events";
}
```

**Event Payload Classes (booking-events)**

```java
package gr.courtbooking.common.event.booking;

import java.util.UUID;

public record BookingCreatedPayload(
    UUID bookingId,
    UUID courtId,
    UUID userId,
    UUID courtOwnerId,
    String date,
    String startTime,
    String endTime,
    int durationMinutes,
    String status,
    String bookingType,
    Integer totalAmountCents,
    boolean isOpenMatch,
    boolean isSplitPayment,
    UUID recurringGroupId
) {}

public record BookingConfirmedPayload(
    UUID bookingId,
    UUID courtId,
    UUID userId,
    UUID courtOwnerId,
    String date,
    String startTime,
    String endTime,
    UUID confirmedBy,
    int capturedAmountCents,
    int platformFeeCents,
    int courtOwnerNetCents
) {}

public record BookingCancelledPayload(
    UUID bookingId,
    UUID courtId,
    UUID userId,
    UUID courtOwnerId,
    String date,
    String startTime,
    String endTime,
    String cancelledBy,
    String cancellationReason,
    int refundAmountCents,
    double refundPercentage,
    boolean waitlistEnabled
) {}

public record BookingModifiedPayload(
    UUID bookingId,
    UUID courtId,
    UUID userId,
    String previousDate,
    String previousStartTime,
    String previousEndTime,
    String newDate,
    String newStartTime,
    String newEndTime,
    int priceDifferenceCents
) {}

public record BookingCompletedPayload(
    UUID bookingId,
    UUID courtId,
    UUID courtOwnerId,
    UUID userId,
    String date,
    String startTime,
    String endTime,
    String bookingType,
    Integer totalAmountCents,
    Integer platformFeeCents,
    Integer courtOwnerNetCents,
    boolean noShow
) {}

public record SlotHeldPayload(
    UUID courtId,
    String date,
    String startTime,
    String endTime,
    String holdExpiresAt,
    UUID heldByUserId
) {}

public record SlotReleasedPayload(
    UUID courtId,
    String date,
    String startTime,
    String endTime,
    String releaseReason
) {}
```


**Event Payload Classes (notification-events)**

```java
package gr.courtbooking.common.event.notification;

import java.util.List;
import java.util.Map;
import java.util.UUID;

public record NotificationRequestedPayload(
    UUID userId,
    String notificationType,
    String urgency,
    Map<String, String> title,
    Map<String, String> body,
    NotificationData data,
    List<String> channels
) {}

public record NotificationData(
    UUID bookingId,
    UUID courtId,
    UUID matchId,
    UUID paymentId,
    UUID ticketId,
    String deepLink
) {}
```

**Event Payload Classes (court-update-events)**

```java
package gr.courtbooking.common.event.court;

import java.util.List;
import java.util.UUID;

public record CourtUpdatedPayload(
    UUID courtId,
    UUID courtOwnerId,
    List<String> changedFields,
    String courtName,
    String courtType,
    String locationType,
    Integer bookingDurationMinutes,
    Integer maxCapacity,
    String confirmationMode,
    Integer confirmationTimeoutHours,
    Boolean waitlistEnabled,
    int version
) {}

public record PricingUpdatedPayload(
    UUID courtId,
    UUID courtOwnerId,
    String courtName,
    Integer previousBasePriceCents,
    int basePriceCents,
    List<PricingRule> pricingRules,
    String effectiveFrom,
    UUID changedBy
) {
    public record PricingRule(
        String dayOfWeek,
        String startTime,
        String endTime,
        double multiplier,
        String label
    ) {}
}

public record AvailabilityUpdatedPayload(
    UUID courtId,
    UUID courtOwnerId,
    String changeType,
    DateRange affectedDateRange,
    Override override
) {
    public record DateRange(String from, String to) {}
    public record Override(UUID overrideId, String date, String startTime, String endTime, String reason) {}
}

public record CancellationPolicyUpdatedPayload(
    UUID courtId,
    UUID courtOwnerId,
    List<Tier> tiers
) {
    public record Tier(int thresholdHours, int refundPercent) {}
}

public record CourtDeletedPayload(
    UUID courtId,
    UUID courtOwnerId
) {}

public record StripeConnectStatusChangedPayload(
    UUID courtOwnerId,
    String stripeConnectAccountId,
    String previousStatus,
    String newStatus,
    boolean requiresAction,
    String restrictionReason,
    List<UUID> affectedCourtIds
) {}
```

**Event Payload Classes (match-events, waitlist-events, analytics-events, security-events)**

```java
package gr.courtbooking.common.event.match;

import java.util.UUID;

public record MatchCreatedPayload(
    UUID matchId, UUID bookingId, UUID courtId, UUID creatorUserId,
    String date, String startTime, String endTime, String courtType,
    String locationType, int maxPlayers, int currentPlayers,
    int creatorSkillLevel, Integer skillRangeMin, Integer skillRangeMax,
    boolean autoAccept, int costPerPlayerCents, double latitude, double longitude
) {}

public record MatchUpdatedPayload(
    UUID matchId, UUID courtId, String updateType,
    int currentPlayers, int maxPlayers, int costPerPlayerCents, UUID affectedUserId
) {}

public record MatchClosedPayload(UUID matchId, UUID courtId, String closeReason) {}
```

```java
package gr.courtbooking.common.event.waitlist;

import java.util.UUID;

public record WaitlistSlotFreedPayload(
    UUID courtId, String date, String startTime, String endTime,
    UUID cancelledBookingId, boolean freedByCourtOwner
) {}

public record WaitlistHoldExpiredPayload(
    UUID courtId, String date, String startTime, String endTime,
    UUID expiredWaitlistEntryId, UUID expiredUserId
) {}
```

```java
package gr.courtbooking.common.event.analytics;

import java.util.UUID;

public record BookingAnalyticsPayload(
    UUID courtId, UUID courtOwnerId, String eventType, UUID bookingId,
    String bookingType, String courtType, String date, String startTime,
    String dayOfWeek, int durationMinutes, Integer playerCount
) {}

public record RevenueAnalyticsPayload(
    UUID courtId, UUID courtOwnerId, String eventType, UUID bookingId,
    int amountCents, int platformFeeCents, int courtOwnerNetCents,
    Integer refundAmountCents, String stripePaymentIntentId, String currency
) {}

public record PromoCodeRedeemedPayload(
    UUID promoCodeId, String promoCode, UUID courtId, UUID courtOwnerId,
    UUID bookingId, UUID userId, String discountType, int discountAmountCents,
    int originalAmountCents, int finalAmountCents
) {}
```

```java
package gr.courtbooking.common.event.security;

import java.util.UUID;

public record SecurityAlertPayload(
    String alertType, String severity, UUID userId, String ipAddress,
    String description, AlertMetadata metadata
) {
    public record AlertMetadata(
        String endpoint, Integer requestCount, Integer windowSeconds,
        Integer threshold, String userAgent, String geoLocation, String stripeEventId
    ) {}
}
```


#### 1.4 Exception Classes

```java
package gr.courtbooking.common.exception;

public class CourtBookingException extends RuntimeException {
    private final String errorCode;
    private final int httpStatus;

    public CourtBookingException(String message, String errorCode, int httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public CourtBookingException(String message, String errorCode, int httpStatus, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() { return errorCode; }
    public int getHttpStatus() { return httpStatus; }
}

public class ResourceNotFoundException extends CourtBookingException {
    public ResourceNotFoundException(String message) {
        super(message, "RESOURCE_NOT_FOUND", 404);
    }
    public ResourceNotFoundException(String resourceType, Object id) {
        super(resourceType + " not found with id: " + id, "RESOURCE_NOT_FOUND", 404);
    }
}

public class ConflictException extends CourtBookingException {
    public ConflictException(String message) {
        super(message, "CONFLICT", 409);
    }
}

public class ValidationException extends CourtBookingException {
    public ValidationException(String message) {
        super(message, "VALIDATION_ERROR", 400);
    }
}

public class UnauthorizedException extends CourtBookingException {
    public UnauthorizedException(String message) {
        super(message, "UNAUTHORIZED", 401);
    }
}

public class ForbiddenException extends CourtBookingException {
    public ForbiddenException(String message) {
        super(message, "FORBIDDEN", 403);
    }
}
```

#### 1.5 Utility Classes

```java
package gr.courtbooking.common.util;

public final class MoneyUtil {
    private MoneyUtil() {}

    public static int add(int amountCents1, int amountCents2) {
        return Math.addExact(amountCents1, amountCents2);
    }

    public static int subtract(int amountCents1, int amountCents2) {
        return Math.subtractExact(amountCents1, amountCents2);
    }

    public static int multiplyByPercentage(int amountCents, int percentage) {
        if (percentage < 0 || percentage > 100) {
            throw new IllegalArgumentException("Percentage must be between 0 and 100");
        }
        return (int) Math.round(amountCents * percentage / 100.0);
    }

    public static int calculatePlatformFee(int amountCents, int feePercentage) {
        return multiplyByPercentage(amountCents, feePercentage);
    }

    public static int calculateNetAmount(int grossAmountCents, int platformFeeCents) {
        return subtract(grossAmountCents, platformFeeCents);
    }
}
```

```java
package gr.courtbooking.common.dto;

import java.time.Instant;

public record ErrorResponse(
    String errorCode,
    String message,
    Instant timestamp,
    String traceId
) {
    public ErrorResponse(String errorCode, String message, String traceId) {
        this(errorCode, message, Instant.now(), traceId);
    }
}
```


### 2. Platform Service (court-booking-platform-service)

#### 2.1 Maven Configuration (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>gr.courtbooking</groupId>
    <artifactId>court-booking-platform-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Court Booking Platform Service</name>
    <description>Platform Service - Auth, Users, Courts, Availability, Analytics</description>

    <properties>
        <java.version>21</java.version>
        <court-booking-common.version>1.0.0</court-booking-common.version>
        <springdoc.version>2.5.0</springdoc.version>
        <jqwik.version>1.8.4</jqwik.version>
        <hibernate-spatial.version>6.4.4.Final</hibernate-spatial.version>
        <logstash-logback.version>7.4</logstash-logback.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-spatial</artifactId>
            <version>${hibernate-spatial.version}</version>
        </dependency>

        <!-- Common Library -->
        <dependency>
            <groupId>gr.courtbooking</groupId>
            <artifactId>court-booking-common</artifactId>
            <version>${court-booking-common.version}</version>
        </dependency>

        <!-- API Documentation -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>${logstash-logback.version}</version>
        </dependency>

        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>net.jqwik</groupId>
            <artifactId>jqwik</artifactId>
            <version>${jqwik.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/OWNER/court-booking-common</url>
        </repository>
    </repositories>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                        <include>**/*Tests.java</include>
                        <include>**/*Properties.java</include>
                    </includes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```


#### 2.2 Spring Configuration Files

**application.yml (shared defaults)**

```yaml
spring:
  application:
    name: court-booking-platform-service
  
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.spatial.dialect.postgis.PostgisPG10Dialect
        default_schema: platform
        jdbc:
          time_zone: UTC

  flyway:
    enabled: true
    schemas: platform
    locations: classpath:db/migration/platform
    baseline-on-migrate: true

server:
  port: 8080
  shutdown: graceful

spring.lifecycle:
  timeout-per-shutdown-phase: 30s

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
      group:
        readiness:
          include: db,redis,kafka
        liveness:
          include: ping

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
  info:
    title: Court Booking Platform Service API
    version: 1.0.0
    description: Platform Service - Authentication, Users, Courts, Availability, Analytics
```

**application-local.yml**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/courtbooking
    username: dev
    password: dev
    hikari:
      maximum-pool-size: 5

  flyway:
    user: dev
    password: dev

  data:
    redis:
      host: localhost
      port: 6379

  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: platform-service-local
      auto-offset-reset: earliest
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

server:
  port: 8081

logging:
  level:
    gr.courtbooking: DEBUG
    org.hibernate.SQL: WARN
    org.apache.kafka: WARN
```

**application-dev.yml / application-test.yml / application-staging.yml / application-prod.yml**

```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: platform_service
    password: ${SPRING_DATASOURCE_PASSWORD}
    hikari:
      maximum-pool-size: 10

  flyway:
    user: platform_service
    password: ${SPRING_DATASOURCE_PASSWORD}

  data:
    redis:
      host: ${SPRING_REDIS_HOST}
      port: 6379
      password: ${SPRING_REDIS_PASSWORD:}

  kafka:
    bootstrap-servers: ${SPRING_KAFKA_BOOTSTRAP_SERVERS}
    consumer:
      group-id: platform-service-${spring.profiles.active}
      auto-offset-reset: earliest
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
      sasl.jaas.config: ${SPRING_KAFKA_SASL_JAAS_CONFIG}

logging:
  level:
    gr.courtbooking: INFO
    org.hibernate.SQL: WARN
    org.apache.kafka: WARN
```

**application-prod.yml (additional)**

```yaml
springdoc:
  swagger-ui:
    enabled: false
```


#### 2.3 Logback Configuration (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="SERVICE_NAME" source="spring.application.name"/>

    <!-- Console appender for local development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- JSON appender for cloud environments -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <customFields>{"service":"${SERVICE_NAME}"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <level>level</level>
                <logger>logger</logger>
                <message>message</message>
            </fieldNames>
        </encoder>
    </appender>

    <!-- Local profile: human-readable console output -->
    <springProfile name="local">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="gr.courtbooking" level="DEBUG"/>
    </springProfile>

    <!-- Cloud profiles: JSON structured logging -->
    <springProfile name="dev,test,staging,prod">
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
        <logger name="gr.courtbooking" level="INFO"/>
        <logger name="org.hibernate.SQL" level="WARN"/>
        <logger name="org.apache.kafka" level="WARN"/>
        <logger name="org.springframework.kafka" level="WARN"/>
    </springProfile>
</configuration>
```

#### 2.4 Main Application Class

```java
package gr.courtbooking.platform;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PlatformServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PlatformServiceApplication.class, args);
    }
}
```

#### 2.5 Placeholder Health Controller

```java
package gr.courtbooking.platform.adapter.in.web;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/health")
public class HealthInfoController {

    @Value("${spring.application.name}")
    private String serviceName;

    @Value("${info.app.version:1.0.0}")
    private String version;

    @GetMapping("/info")
    public Map<String, String> getInfo() {
        return Map.of(
            "service", serviceName,
            "version", version,
            "status", "UP"
        );
    }
}
```

#### 2.6 OpenAPI Configuration

```java
package gr.courtbooking.platform.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Value("${spring.application.name}")
    private String serviceName;

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Court Booking Platform Service API")
                .version("1.0.0")
                .description("Platform Service - Authentication, Users, Courts, Availability, Analytics"));
    }
}
```

#### 2.7 Request ID Filter (MDC for logging)

```java
package gr.courtbooking.platform.config;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(1)
public class RequestIdFilter implements Filter {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";
    private static final String REQUEST_ID_MDC_KEY = "requestId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String requestId = httpRequest.getHeader(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }
        MDC.put(REQUEST_ID_MDC_KEY, requestId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(REQUEST_ID_MDC_KEY);
        }
    }
}
```


### 3. Transaction Service (court-booking-transaction-service)

#### 3.1 Maven Configuration (pom.xml)

The Transaction Service pom.xml follows the same structure as Platform Service with these additional dependencies:

```xml
<!-- Additional dependencies for Transaction Service -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

Note: Transaction Service does NOT include hibernate-spatial since it doesn't manage PostGIS geometry columns directly.

#### 3.2 Quartz Scheduler Configuration

**application.yml (Quartz section)**

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: never  # Flyway manages schema
    properties:
      org.quartz:
        scheduler:
          instanceName: TransactionServiceScheduler
          instanceId: AUTO
        jobStore:
          class: org.quartz.impl.jdbcjobstore.JobStoreTX
          driverDelegateClass: org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
          tablePrefix: QRTZ_
          isClustered: true
          clusterCheckinInterval: 20000
          useProperties: false
        threadPool:
          class: org.quartz.simpl.SimpleThreadPool
          threadCount: ${QUARTZ_THREAD_COUNT:10}
          threadPriority: 5
```

#### 3.3 Transaction Service Main Application

```java
package gr.courtbooking.transaction;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TransactionServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(TransactionServiceApplication.class, args);
    }
}
```

#### 3.4 Transaction Service application-local.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/courtbooking
    username: dev
    password: dev
    hikari:
      maximum-pool-size: 5

  flyway:
    user: dev
    password: dev
    schemas: transaction
    locations: classpath:db/migration/transaction

  jpa:
    properties:
      hibernate:
        default_schema: transaction

  data:
    redis:
      host: localhost
      port: 6379

  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: transaction-service-local
      auto-offset-reset: earliest

server:
  port: 8082

logging:
  level:
    gr.courtbooking: DEBUG
```


## Data Models

### Flyway Migration Structure

#### Platform Service Migrations

**V1__create_platform_schema.sql** (src/main/resources/db/migration/platform/)

```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Users table
CREATE TABLE platform.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    role VARCHAR(20) NOT NULL CHECK (role IN ('CUSTOMER', 'COURT_OWNER', 'SUPPORT_AGENT', 'PLATFORM_ADMIN')),
    language VARCHAR(5) NOT NULL DEFAULT 'el' CHECK (language IN ('el', 'en')),
    verified BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_connect_account_id VARCHAR(255),
    stripe_connect_status VARCHAR(20) DEFAULT 'NOT_STARTED' CHECK (stripe_connect_status IN ('NOT_STARTED','PENDING','ACTIVE','RESTRICTED','DISABLED')),
    stripe_customer_id VARCHAR(255),
    payout_schedule VARCHAR(10) CHECK (payout_schedule IN ('DAILY','WEEKLY','MONTHLY')),
    subscription_status VARCHAR(10) NOT NULL DEFAULT 'NONE' CHECK (subscription_status IN ('TRIAL','ACTIVE','EXPIRED','NONE')),
    trial_ends_at TIMESTAMPTZ,
    stripe_subscription_id VARCHAR(255),
    business_name VARCHAR(255),
    tax_id VARCHAR(50),
    business_type VARCHAR(30) CHECK (business_type IN ('SOLE_PROPRIETOR','COMPANY','ASSOCIATION')),
    business_address VARCHAR(500),
    vat_registered BOOLEAN NOT NULL DEFAULT FALSE,
    vat_number VARCHAR(50),
    contact_phone VARCHAR(50),
    profile_image_url VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE','SUSPENDED','DELETED')),
    terms_accepted_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON platform.users(email);
CREATE INDEX idx_users_role ON platform.users(role);
CREATE INDEX idx_users_status ON platform.users(status);
CREATE INDEX idx_users_stripe_connect ON platform.users(stripe_connect_account_id) WHERE stripe_connect_account_id IS NOT NULL;
CREATE INDEX idx_users_stripe_customer ON platform.users(stripe_customer_id) WHERE stripe_customer_id IS NOT NULL;

-- OAuth Providers table
CREATE TABLE platform.oauth_providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES platform.users(id) ON DELETE CASCADE,
    provider VARCHAR(20) NOT NULL CHECK (provider IN ('GOOGLE','FACEBOOK','APPLE')),
    provider_user_id VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    linked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(provider, provider_user_id)
);

CREATE INDEX idx_oauth_providers_user_id ON platform.oauth_providers(user_id);

-- Refresh Tokens table
CREATE TABLE platform.refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES platform.users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    device_id VARCHAR(255),
    device_info VARCHAR(500),
    ip_address VARCHAR(45),
    invalidated BOOLEAN NOT NULL DEFAULT FALSE,
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON platform.refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_expires_at ON platform.refresh_tokens(expires_at);

-- Verification Requests table
CREATE TABLE platform.verification_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_owner_id UUID NOT NULL REFERENCES platform.users(id),
    business_name VARCHAR(255) NOT NULL,
    tax_id VARCHAR(50) NOT NULL,
    business_type VARCHAR(30) NOT NULL CHECK (business_type IN ('SOLE_PROPRIETOR','COMPANY','ASSOCIATION')),
    business_address VARCHAR(500) NOT NULL,
    proof_document_url VARCHAR(500) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING_REVIEW' CHECK (status IN ('PENDING_REVIEW','APPROVED','REJECTED')),
    reviewed_by UUID REFERENCES platform.users(id),
    review_notes TEXT,
    rejection_reason TEXT,
    previous_request_id UUID REFERENCES platform.verification_requests(id),
    submitted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_at TIMESTAMPTZ
);

CREATE INDEX idx_verification_requests_owner ON platform.verification_requests(court_owner_id);
CREATE INDEX idx_verification_requests_status ON platform.verification_requests(status) WHERE status = 'PENDING_REVIEW';
CREATE INDEX idx_verification_requests_submitted ON platform.verification_requests(submitted_at);

-- Courts table
CREATE TABLE platform.courts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES platform.users(id),
    name_el VARCHAR(255) NOT NULL,
    name_en VARCHAR(255),
    description_el TEXT,
    description_en TEXT,
    court_type VARCHAR(20) NOT NULL CHECK (court_type IN ('TENNIS','PADEL','BASKETBALL','FOOTBALL_5X5')),
    location_type VARCHAR(10) NOT NULL CHECK (location_type IN ('INDOOR','OUTDOOR')),
    location GEOMETRY(Point, 4326) NOT NULL,
    address VARCHAR(500) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'Europe/Athens',
    base_price_cents INTEGER NOT NULL,
    duration_minutes INTEGER NOT NULL,
    max_capacity INTEGER NOT NULL,
    confirmation_mode VARCHAR(10) NOT NULL DEFAULT 'INSTANT' CHECK (confirmation_mode IN ('INSTANT','MANUAL')),
    confirmation_timeout_hours INTEGER DEFAULT 24,
    waitlist_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    amenities TEXT[],
    image_urls TEXT[],
    average_rating DECIMAL(3,2),
    total_reviews INTEGER NOT NULL DEFAULT 0,
    visible BOOLEAN NOT NULL DEFAULT FALSE,
    version INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_courts_owner_id ON platform.courts(owner_id);
CREATE INDEX idx_courts_location ON platform.courts USING GIST(location);
CREATE INDEX idx_courts_court_type ON platform.courts(court_type);
CREATE INDEX idx_courts_location_type ON platform.courts(location_type);
CREATE INDEX idx_courts_visible ON platform.courts(visible) WHERE visible = TRUE;
CREATE INDEX idx_courts_type_location ON platform.courts(court_type, location_type) WHERE visible = TRUE;
```


**V1__create_platform_schema.sql (continued)**

```sql
-- Availability Windows table
CREATE TABLE platform.availability_windows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    day_of_week VARCHAR(10) NOT NULL CHECK (day_of_week IN ('MONDAY','TUESDAY','WEDNESDAY','THURSDAY','FRIDAY','SATURDAY','SUNDAY')),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL CHECK (end_time > start_time)
);

CREATE INDEX idx_availability_windows_court_id ON platform.availability_windows(court_id);

-- Availability Overrides table
CREATE TABLE platform.availability_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    start_time TIME,
    end_time TIME,
    reason VARCHAR(255),
    source VARCHAR(20) NOT NULL DEFAULT 'MANUAL' CHECK (source IN ('MANUAL','HOLIDAY_NATIONAL','HOLIDAY_CUSTOM')),
    holiday_template_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_availability_overrides_court_date ON platform.availability_overrides(court_id, date);
CREATE INDEX idx_availability_overrides_source ON platform.availability_overrides(source) WHERE source != 'MANUAL';

-- Holiday Templates table
CREATE TABLE platform.holiday_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES platform.users(id) ON DELETE CASCADE,
    name_el VARCHAR(255) NOT NULL,
    name_en VARCHAR(255),
    date_pattern VARCHAR(50) NOT NULL,
    is_national BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_holiday_templates_owner ON platform.holiday_templates(owner_id) WHERE owner_id IS NOT NULL;
CREATE INDEX idx_holiday_templates_national ON platform.holiday_templates(is_national) WHERE is_national = TRUE;
CREATE UNIQUE INDEX uq_holiday_templates_owner_name ON platform.holiday_templates(owner_id, name_el) WHERE owner_id IS NOT NULL;

-- Add FK for availability_overrides.holiday_template_id
ALTER TABLE platform.availability_overrides 
    ADD CONSTRAINT fk_availability_overrides_holiday 
    FOREIGN KEY (holiday_template_id) REFERENCES platform.holiday_templates(id) ON DELETE SET NULL;

CREATE INDEX idx_availability_overrides_holiday ON platform.availability_overrides(holiday_template_id) WHERE holiday_template_id IS NOT NULL;

-- Court Holiday Subscriptions table
CREATE TABLE platform.court_holiday_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    holiday_template_id UUID NOT NULL REFERENCES platform.holiday_templates(id) ON DELETE CASCADE,
    auto_renew BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(court_id, holiday_template_id)
);

CREATE INDEX idx_court_holiday_subscriptions_court ON platform.court_holiday_subscriptions(court_id);

-- Favorites table
CREATE TABLE platform.favorites (
    user_id UUID NOT NULL REFERENCES platform.users(id) ON DELETE CASCADE,
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, court_id)
);

-- Preferences table
CREATE TABLE platform.preferences (
    user_id UUID PRIMARY KEY REFERENCES platform.users(id) ON DELETE CASCADE,
    preferred_days TEXT[],
    preferred_start_time TIME,
    preferred_end_time TIME,
    max_search_distance_km DECIMAL(5,1) DEFAULT 10.0,
    notify_booking_events BOOLEAN DEFAULT TRUE,
    notify_favorite_alerts BOOLEAN DEFAULT TRUE,
    notify_promotional BOOLEAN DEFAULT TRUE,
    notify_email BOOLEAN DEFAULT TRUE,
    notify_push BOOLEAN DEFAULT TRUE,
    dnd_start TIME,
    dnd_end TIME
);

-- Skill Levels table
CREATE TABLE platform.skill_levels (
    user_id UUID NOT NULL REFERENCES platform.users(id) ON DELETE CASCADE,
    court_type VARCHAR(20) NOT NULL,
    level INTEGER NOT NULL CHECK (level BETWEEN 1 AND 7),
    PRIMARY KEY (user_id, court_type)
);

-- Court Ratings table (Phase 2, created empty)
CREATE TABLE platform.court_ratings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES platform.users(id),
    booking_id UUID NOT NULL,
    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_court_ratings_court_id ON platform.court_ratings(court_id);
CREATE UNIQUE INDEX uq_court_ratings_booking ON platform.court_ratings(booking_id);

-- Pricing Rules table
CREATE TABLE platform.pricing_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    day_of_week VARCHAR(10) NOT NULL CHECK (day_of_week IN ('MONDAY','TUESDAY','WEDNESDAY','THURSDAY','FRIDAY','SATURDAY','SUNDAY')),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL CHECK (end_time > start_time),
    multiplier DECIMAL(3,2) NOT NULL CHECK (multiplier BETWEEN 0.10 AND 5.00),
    label VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pricing_rules_court_id ON platform.pricing_rules(court_id);

-- Special Date Pricing table (Phase 2, created empty)
CREATE TABLE platform.special_date_pricing (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    multiplier DECIMAL(3,2) NOT NULL CHECK (multiplier BETWEEN 0.10 AND 5.00),
    label VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_special_date_pricing_court_date ON platform.special_date_pricing(court_id, date);

-- Cancellation Tiers table
CREATE TABLE platform.cancellation_tiers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL REFERENCES platform.courts(id) ON DELETE CASCADE,
    threshold_hours INTEGER NOT NULL,
    refund_percent INTEGER NOT NULL CHECK (refund_percent BETWEEN 0 AND 100),
    sort_order INTEGER NOT NULL
);

CREATE INDEX idx_cancellation_tiers_court_id ON platform.cancellation_tiers(court_id);
CREATE UNIQUE INDEX uq_cancellation_tiers_court_threshold ON platform.cancellation_tiers(court_id, threshold_hours);
```


**V1__create_platform_schema.sql (continued - remaining tables)**

```sql
-- Promo Codes table (Phase 2, created empty)
CREATE TABLE platform.promo_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_owner_id UUID NOT NULL REFERENCES platform.users(id),
    code VARCHAR(50) NOT NULL,
    discount_type VARCHAR(20) NOT NULL CHECK (discount_type IN ('PERCENTAGE','FIXED_AMOUNT')),
    discount_value INTEGER NOT NULL,
    min_booking_cents INTEGER,
    max_discount_cents INTEGER,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ NOT NULL,
    max_uses INTEGER,
    current_uses INTEGER NOT NULL DEFAULT 0,
    max_uses_per_user INTEGER DEFAULT 1,
    applicable_court_ids UUID[],
    applicable_court_types TEXT[],
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_promo_codes_owner ON platform.promo_codes(court_owner_id);
CREATE UNIQUE INDEX uq_promo_codes_owner_code ON platform.promo_codes(court_owner_id, code);
CREATE INDEX idx_promo_codes_active ON platform.promo_codes(court_owner_id, active) WHERE active = TRUE;

-- Promo Code Redemptions table (Phase 2, created empty)
CREATE TABLE platform.promo_code_redemptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    promo_code_id UUID NOT NULL REFERENCES platform.promo_codes(id),
    booking_id UUID NOT NULL,
    user_id UUID NOT NULL REFERENCES platform.users(id),
    discount_amount_cents INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE','ROLLED_BACK')),
    rolled_back_at TIMESTAMPTZ,
    rollback_reason VARCHAR(30) CHECK (rollback_reason IN ('BOOKING_CANCELLED','BOOKING_REFUNDED','PAYMENT_FAILED')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX uq_promo_redemption_booking ON platform.promo_code_redemptions(promo_code_id, booking_id);
CREATE INDEX idx_promo_redemptions_user ON platform.promo_code_redemptions(promo_code_id, user_id);
CREATE INDEX idx_promo_redemptions_status ON platform.promo_code_redemptions(promo_code_id, status) WHERE status = 'ACTIVE';

-- Translations table
CREATE TABLE platform.translations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(255) NOT NULL,
    language VARCHAR(5) NOT NULL CHECK (language IN ('el','en')),
    value TEXT NOT NULL
);

CREATE UNIQUE INDEX uq_translations_key_lang ON platform.translations(key, language);

-- Feature Flags table
CREATE TABLE platform.feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_key VARCHAR(100) NOT NULL UNIQUE,
    enabled BOOLEAN NOT NULL DEFAULT FALSE,
    description VARCHAR(500),
    updated_by UUID REFERENCES platform.users(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Support Tickets table
CREATE TABLE platform.support_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES platform.users(id),
    category VARCHAR(30) NOT NULL CHECK (category IN ('BOOKING','PAYMENT','COURT','ACCOUNT','TECHNICAL','OTHER')),
    subject VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN','IN_PROGRESS','WAITING_ON_USER','RESOLVED','CLOSED')),
    priority VARCHAR(10) NOT NULL DEFAULT 'NORMAL' CHECK (priority IN ('LOW','NORMAL','HIGH','URGENT')),
    assigned_to UUID REFERENCES platform.users(id),
    related_booking_id UUID,
    related_court_id UUID,
    diagnostic_data JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_support_tickets_user_id ON platform.support_tickets(user_id);
CREATE INDEX idx_support_tickets_status ON platform.support_tickets(status);
CREATE INDEX idx_support_tickets_assigned ON platform.support_tickets(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_support_tickets_created ON platform.support_tickets(created_at);

-- Support Messages table
CREATE TABLE platform.support_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id UUID NOT NULL REFERENCES platform.support_tickets(id) ON DELETE CASCADE,
    sender_id UUID NOT NULL REFERENCES platform.users(id),
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_support_messages_ticket_id ON platform.support_messages(ticket_id);

-- Support Attachments table
CREATE TABLE platform.support_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES platform.support_messages(id) ON DELETE CASCADE,
    file_url VARCHAR(500) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_support_attachments_message_id ON platform.support_attachments(message_id);

-- Court Owner Audit Logs table
CREATE TABLE platform.court_owner_audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_owner_id UUID NOT NULL REFERENCES platform.users(id),
    court_id UUID REFERENCES platform.courts(id),
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID,
    changes JSONB NOT NULL,
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_court_owner_audit_logs_owner ON platform.court_owner_audit_logs(court_owner_id);
CREATE INDEX idx_court_owner_audit_logs_court ON platform.court_owner_audit_logs(court_id) WHERE court_id IS NOT NULL;
CREATE INDEX idx_court_owner_audit_logs_created ON platform.court_owner_audit_logs(created_at);
CREATE INDEX idx_court_owner_audit_logs_action ON platform.court_owner_audit_logs(court_owner_id, action);

-- Reminder Rules table
CREATE TABLE platform.reminder_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_owner_id UUID NOT NULL REFERENCES platform.users(id),
    court_id UUID REFERENCES platform.courts(id) ON DELETE CASCADE,
    rule_type VARCHAR(30) NOT NULL CHECK (rule_type IN ('BOOKING_REMINDER','PENDING_CONFIRMATION','LOW_OCCUPANCY','DAILY_SUMMARY')),
    trigger_hours_before INTEGER,
    trigger_time TIME,
    threshold_percent INTEGER,
    channels TEXT[] NOT NULL DEFAULT '{PUSH,EMAIL}',
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reminder_rules_owner ON platform.reminder_rules(court_owner_id);
CREATE INDEX idx_reminder_rules_court ON platform.reminder_rules(court_id) WHERE court_id IS NOT NULL;
CREATE INDEX idx_reminder_rules_type ON platform.reminder_rules(court_owner_id, rule_type) WHERE enabled = TRUE;

-- Court Owner Notification Preferences table
CREATE TABLE platform.court_owner_notification_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_owner_id UUID NOT NULL REFERENCES platform.users(id),
    event_type VARCHAR(50) NOT NULL,
    push_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    email_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    in_app_enabled BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE UNIQUE INDEX uq_notification_prefs_owner_event ON platform.court_owner_notification_preferences(court_owner_id, event_type);

-- Court Defaults table
CREATE TABLE platform.court_defaults (
    court_owner_id UUID PRIMARY KEY REFERENCES platform.users(id) ON DELETE CASCADE,
    default_duration_minutes INTEGER,
    default_confirmation_mode VARCHAR(10) CHECK (default_confirmation_mode IN ('INSTANT','MANUAL')),
    default_confirmation_timeout_hours INTEGER,
    default_cancellation_tiers JSONB,
    default_amenities TEXT[],
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Security Alerts table
CREATE TABLE platform.security_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_type VARCHAR(50) NOT NULL CHECK (alert_type IN ('BOOKING_ABUSE','PAYMENT_FRAUD','SCRAPING','BRUTE_FORCE','SUSPICIOUS_LOGIN','RATE_LIMIT_EXCEEDED','WEBHOOK_REPLAY','ACCOUNT_TAKEOVER')),
    severity VARCHAR(10) NOT NULL CHECK (severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    user_id UUID REFERENCES platform.users(id),
    ip_address VARCHAR(45),
    description TEXT NOT NULL,
    metadata JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN','ACKNOWLEDGED','INVESTIGATING','RESOLVED','FALSE_POSITIVE')),
    acknowledged_by UUID REFERENCES platform.users(id),
    resolved_by UUID REFERENCES platform.users(id),
    resolution_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_security_alerts_status ON platform.security_alerts(status) WHERE status IN ('OPEN','ACKNOWLEDGED','INVESTIGATING');
CREATE INDEX idx_security_alerts_severity ON platform.security_alerts(severity) WHERE status NOT IN ('RESOLVED','FALSE_POSITIVE');
CREATE INDEX idx_security_alerts_user_id ON platform.security_alerts(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_security_alerts_ip ON platform.security_alerts(ip_address) WHERE ip_address IS NOT NULL;
CREATE INDEX idx_security_alerts_type ON platform.security_alerts(alert_type);
CREATE INDEX idx_security_alerts_created ON platform.security_alerts(created_at);

-- IP Blocklist table
CREATE TABLE platform.ip_blocklist (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ip_address VARCHAR(45) NOT NULL UNIQUE,
    cidr_range VARCHAR(49),
    reason VARCHAR(255) NOT NULL,
    blocked_by UUID NOT NULL REFERENCES platform.users(id),
    related_alert_id UUID REFERENCES platform.security_alerts(id),
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ip_blocklist_expires ON platform.ip_blocklist(expires_at) WHERE expires_at IS NOT NULL;

-- Failed Auth Attempts table
CREATE TABLE platform.failed_auth_attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ip_address VARCHAR(45) NOT NULL,
    email VARCHAR(255),
    attempt_count INTEGER NOT NULL DEFAULT 1,
    window_start TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    locked_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_failed_auth_ip ON platform.failed_auth_attempts(ip_address);
CREATE INDEX idx_failed_auth_email ON platform.failed_auth_attempts(email) WHERE email IS NOT NULL;
CREATE INDEX idx_failed_auth_locked ON platform.failed_auth_attempts(locked_until) WHERE locked_until IS NOT NULL;
```


**V2__create_cross_schema_views.sql**

```sql
-- Cross-schema views for Transaction Service read-only access

CREATE VIEW platform.v_court_summary AS
SELECT
    id,
    owner_id,
    name_el,
    name_en,
    court_type,
    location_type,
    location,
    timezone,
    base_price_cents,
    duration_minutes,
    max_capacity,
    confirmation_mode,
    confirmation_timeout_hours,
    waitlist_enabled,
    visible,
    version
FROM platform.courts
WHERE visible = TRUE;

CREATE VIEW platform.v_user_basic AS
SELECT
    id,
    email,
    name,
    phone,
    role,
    language,
    stripe_connect_account_id,
    stripe_connect_status,
    stripe_customer_id,
    vat_registered,
    vat_number,
    status
FROM platform.users
WHERE status = 'ACTIVE';

CREATE VIEW platform.v_court_cancellation_tiers AS
SELECT
    court_id,
    threshold_hours,
    refund_percent,
    sort_order
FROM platform.cancellation_tiers
ORDER BY court_id, sort_order;

CREATE VIEW platform.v_user_skill_level AS
SELECT
    user_id,
    court_type,
    level
FROM platform.skill_levels;

-- Grant SELECT on views to transaction_service role
-- Note: These grants are executed by the DBA or Terraform during initial setup
-- GRANT SELECT ON platform.v_court_summary TO transaction_service;
-- GRANT SELECT ON platform.v_user_basic TO transaction_service;
-- GRANT SELECT ON platform.v_court_cancellation_tiers TO transaction_service;
-- GRANT SELECT ON platform.v_user_skill_level TO transaction_service;
```

#### Transaction Service Migrations

**V1__create_transaction_schema.sql** (src/main/resources/db/migration/transaction/)

```sql
-- Bookings table
CREATE TABLE transaction.bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL,
    user_id UUID,
    court_owner_id UUID NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL UNIQUE,
    date DATE NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL CHECK (end_time > start_time),
    duration_minutes INTEGER NOT NULL,
    status VARCHAR(25) NOT NULL CHECK (status IN ('PENDING_CONFIRMATION','CONFIRMED','CANCELLED','COMPLETED','REJECTED')),
    booking_type VARCHAR(10) NOT NULL DEFAULT 'CUSTOMER' CHECK (booking_type IN ('CUSTOMER','MANUAL')),
    confirmation_mode VARCHAR(10) NOT NULL CHECK (confirmation_mode IN ('INSTANT','MANUAL')),
    total_amount_cents INTEGER,
    platform_fee_cents INTEGER,
    court_owner_net_cents INTEGER,
    discount_cents INTEGER DEFAULT 0,
    promo_code_id UUID,
    payment_status VARCHAR(20) NOT NULL DEFAULT 'NOT_REQUIRED' CHECK (payment_status IN ('NOT_REQUIRED','PENDING','AUTHORIZED','CAPTURED','REFUNDED','PARTIALLY_REFUNDED','FAILED','PAID_EXTERNALLY')),
    stripe_payment_intent_id VARCHAR(255),
    paid_externally BOOLEAN NOT NULL DEFAULT FALSE,
    external_payment_method VARCHAR(50),
    external_payment_notes VARCHAR(500),
    no_show BOOLEAN NOT NULL DEFAULT FALSE,
    no_show_flagged_at TIMESTAMPTZ,
    customer_name VARCHAR(255),
    customer_phone VARCHAR(50),
    notes TEXT,
    recurring_group_id UUID,
    recurring_pattern JSONB,
    cancelled_by VARCHAR(15) CHECK (cancelled_by IN ('CUSTOMER','COURT_OWNER','SYSTEM')),
    cancellation_reason TEXT,
    cancelled_at TIMESTAMPTZ,
    refund_amount_cents INTEGER,
    confirmed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bookings_court_date ON transaction.bookings(court_id, date);
CREATE INDEX idx_bookings_user_id ON transaction.bookings(user_id);
CREATE INDEX idx_bookings_court_owner_id ON transaction.bookings(court_owner_id);
CREATE INDEX idx_bookings_status ON transaction.bookings(status);
CREATE INDEX idx_bookings_date ON transaction.bookings(date);
CREATE INDEX idx_bookings_recurring_group ON transaction.bookings(recurring_group_id) WHERE recurring_group_id IS NOT NULL;
CREATE INDEX idx_bookings_payment_status ON transaction.bookings(payment_status) WHERE payment_status IN ('PENDING','AUTHORIZED');
CREATE INDEX idx_bookings_pending_confirmation ON transaction.bookings(court_owner_id, status) WHERE status = 'PENDING_CONFIRMATION';

-- Payments table
CREATE TABLE transaction.payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES transaction.bookings(id),
    user_id UUID NOT NULL,
    amount_cents INTEGER NOT NULL,
    platform_fee_cents INTEGER NOT NULL,
    court_owner_net_cents INTEGER NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'EUR',
    status VARCHAR(20) NOT NULL CHECK (status IN ('PENDING','AUTHORIZED','CAPTURED','REFUNDED','PARTIALLY_REFUNDED','FAILED','DISPUTED')),
    stripe_payment_intent_id VARCHAR(255) NOT NULL,
    stripe_charge_id VARCHAR(255),
    stripe_transfer_id VARCHAR(255),
    stripe_refund_id VARCHAR(255),
    payment_method_type VARCHAR(20),
    refund_amount_cents INTEGER,
    refund_reason TEXT,
    refunded_at TIMESTAMPTZ,
    failure_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_booking_id ON transaction.payments(booking_id);
CREATE INDEX idx_payments_user_id ON transaction.payments(user_id);
CREATE UNIQUE INDEX idx_payments_stripe_pi ON transaction.payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON transaction.payments(status);

-- Audit Logs table
CREATE TABLE transaction.audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES transaction.bookings(id),
    action VARCHAR(30) NOT NULL,
    performed_by UUID,
    performed_by_role VARCHAR(20),
    details JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_booking_id ON transaction.audit_logs(booking_id);
CREATE INDEX idx_audit_logs_created ON transaction.audit_logs(created_at);

-- Notifications table
CREATE TABLE transaction.notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    channel VARCHAR(10) NOT NULL CHECK (channel IN ('PUSH','EMAIL','IN_APP','WEB_PUSH')),
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    language VARCHAR(5) NOT NULL DEFAULT 'el' CHECK (language IN ('el','en')),
    data JSONB,
    status VARCHAR(15) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','SENT','DELIVERED','FAILED','READ')),
    read_at TIMESTAMPTZ,
    failure_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON transaction.notifications(user_id);
CREATE INDEX idx_notifications_user_unread ON transaction.notifications(user_id, status) WHERE status != 'READ';
CREATE INDEX idx_notifications_type ON transaction.notifications(notification_type);
CREATE INDEX idx_notifications_created ON transaction.notifications(created_at);

-- Device Tokens table
CREATE TABLE transaction.device_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    token VARCHAR(500) NOT NULL UNIQUE,
    platform VARCHAR(10) NOT NULL CHECK (platform IN ('IOS','ANDROID','WEB_PUSH')),
    device_id VARCHAR(255),
    push_subscription_endpoint VARCHAR(500),
    push_subscription_keys JSONB,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_device_tokens_user_id ON transaction.device_tokens(user_id) WHERE active = TRUE;
```


**V1__create_transaction_schema.sql (continued - Phase 2 tables)**

```sql
-- Waitlists table (Phase 2, created empty)
CREATE TABLE transaction.waitlists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    court_id UUID NOT NULL,
    user_id UUID NOT NULL,
    date DATE NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'WAITING' CHECK (status IN ('WAITING','NOTIFIED','HOLD_ACTIVE','BOOKED','EXPIRED','CANCELLED')),
    position INTEGER NOT NULL,
    hold_expires_at TIMESTAMPTZ,
    notified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_waitlists_court_slot ON transaction.waitlists(court_id, date, start_time) WHERE status IN ('WAITING','NOTIFIED','HOLD_ACTIVE');
CREATE INDEX idx_waitlists_user_id ON transaction.waitlists(user_id);
CREATE INDEX idx_waitlists_hold_expires ON transaction.waitlists(hold_expires_at) WHERE status = 'HOLD_ACTIVE';

-- Open Matches table (Phase 2, created empty)
CREATE TABLE transaction.open_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES transaction.bookings(id),
    court_id UUID NOT NULL,
    creator_user_id UUID NOT NULL,
    court_type VARCHAR(20) NOT NULL,
    max_players INTEGER NOT NULL,
    current_players INTEGER NOT NULL DEFAULT 1,
    skill_range_min INTEGER CHECK (skill_range_min BETWEEN 1 AND 7),
    skill_range_max INTEGER CHECK (skill_range_max BETWEEN 1 AND 7),
    auto_accept BOOLEAN NOT NULL DEFAULT FALSE,
    cost_per_player_cents INTEGER NOT NULL,
    status VARCHAR(15) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN','FULL','CANCELLED','EXPIRED')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_open_matches_court_id ON transaction.open_matches(court_id);
CREATE INDEX idx_open_matches_status ON transaction.open_matches(status) WHERE status = 'OPEN';
CREATE INDEX idx_open_matches_court_type ON transaction.open_matches(court_type) WHERE status = 'OPEN';

-- Match Players table (Phase 2, created empty)
CREATE TABLE transaction.match_players (
    match_id UUID NOT NULL REFERENCES transaction.open_matches(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    role VARCHAR(10) NOT NULL DEFAULT 'PLAYER' CHECK (role IN ('CREATOR','PLAYER')),
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (match_id, user_id)
);

-- Match Join Requests table (Phase 2, created empty)
CREATE TABLE transaction.match_join_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_id UUID NOT NULL REFERENCES transaction.open_matches(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    status VARCHAR(15) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','APPROVED','DECLINED','CANCELLED')),
    message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    responded_at TIMESTAMPTZ
);

CREATE INDEX idx_match_join_requests_match ON transaction.match_join_requests(match_id);
CREATE INDEX idx_match_join_requests_user ON transaction.match_join_requests(user_id);

-- Match Messages table (Phase 2, created empty)
CREATE TABLE transaction.match_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_id UUID NOT NULL REFERENCES transaction.open_matches(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_match_messages_match ON transaction.match_messages(match_id);

-- Split Payments table (Phase 2, created empty)
CREATE TABLE transaction.split_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL REFERENCES transaction.bookings(id),
    initiator_user_id UUID NOT NULL,
    total_amount_cents INTEGER NOT NULL,
    per_share_cents INTEGER NOT NULL,
    total_shares INTEGER NOT NULL,
    paid_shares INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','PARTIALLY_PAID','FULLY_PAID','EXPIRED','CANCELLED')),
    deadline TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_split_payments_booking ON transaction.split_payments(booking_id);
CREATE INDEX idx_split_payments_status ON transaction.split_payments(status) WHERE status IN ('PENDING','PARTIALLY_PAID');

-- Split Payment Shares table (Phase 2, created empty)
CREATE TABLE transaction.split_payment_shares (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    split_payment_id UUID NOT NULL REFERENCES transaction.split_payments(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    amount_cents INTEGER NOT NULL,
    status VARCHAR(15) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','PAID','FAILED','EXPIRED')),
    stripe_payment_intent_id VARCHAR(255),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_split_shares_split_id ON transaction.split_payment_shares(split_payment_id);
CREATE INDEX idx_split_shares_user ON transaction.split_payment_shares(user_id);
```

**V2__create_quartz_tables.sql**

```sql
-- Quartz Scheduler JDBC Job Store tables for PostgreSQL
-- Standard Quartz DDL for clustered mode

CREATE TABLE transaction.QRTZ_JOB_DETAILS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    JOB_NAME VARCHAR(200) NOT NULL,
    JOB_GROUP VARCHAR(200) NOT NULL,
    DESCRIPTION VARCHAR(250) NULL,
    JOB_CLASS_NAME VARCHAR(250) NOT NULL,
    IS_DURABLE BOOLEAN NOT NULL,
    IS_NONCONCURRENT BOOLEAN NOT NULL,
    IS_UPDATE_DATA BOOLEAN NOT NULL,
    REQUESTS_RECOVERY BOOLEAN NOT NULL,
    JOB_DATA BYTEA NULL,
    PRIMARY KEY (SCHED_NAME, JOB_NAME, JOB_GROUP)
);

CREATE TABLE transaction.QRTZ_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    JOB_NAME VARCHAR(200) NOT NULL,
    JOB_GROUP VARCHAR(200) NOT NULL,
    DESCRIPTION VARCHAR(250) NULL,
    NEXT_FIRE_TIME BIGINT NULL,
    PREV_FIRE_TIME BIGINT NULL,
    PRIORITY INTEGER NULL,
    TRIGGER_STATE VARCHAR(16) NOT NULL,
    TRIGGER_TYPE VARCHAR(8) NOT NULL,
    START_TIME BIGINT NOT NULL,
    END_TIME BIGINT NULL,
    CALENDAR_NAME VARCHAR(200) NULL,
    MISFIRE_INSTR SMALLINT NULL,
    JOB_DATA BYTEA NULL,
    PRIMARY KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME, JOB_NAME, JOB_GROUP) REFERENCES transaction.QRTZ_JOB_DETAILS(SCHED_NAME, JOB_NAME, JOB_GROUP)
);

CREATE TABLE transaction.QRTZ_SIMPLE_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    REPEAT_COUNT BIGINT NOT NULL,
    REPEAT_INTERVAL BIGINT NOT NULL,
    TIMES_TRIGGERED BIGINT NOT NULL,
    PRIMARY KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP) REFERENCES transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP)
);

CREATE TABLE transaction.QRTZ_CRON_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    CRON_EXPRESSION VARCHAR(120) NOT NULL,
    TIME_ZONE_ID VARCHAR(80),
    PRIMARY KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP) REFERENCES transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP)
);

CREATE TABLE transaction.QRTZ_SIMPROP_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    STR_PROP_1 VARCHAR(512) NULL,
    STR_PROP_2 VARCHAR(512) NULL,
    STR_PROP_3 VARCHAR(512) NULL,
    INT_PROP_1 INTEGER NULL,
    INT_PROP_2 INTEGER NULL,
    LONG_PROP_1 BIGINT NULL,
    LONG_PROP_2 BIGINT NULL,
    DEC_PROP_1 NUMERIC(13,4) NULL,
    DEC_PROP_2 NUMERIC(13,4) NULL,
    BOOL_PROP_1 BOOLEAN NULL,
    BOOL_PROP_2 BOOLEAN NULL,
    PRIMARY KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP) REFERENCES transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP)
);

CREATE TABLE transaction.QRTZ_BLOB_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    BLOB_DATA BYTEA NULL,
    PRIMARY KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP),
    FOREIGN KEY (SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP) REFERENCES transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP)
);

CREATE TABLE transaction.QRTZ_CALENDARS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    CALENDAR_NAME VARCHAR(200) NOT NULL,
    CALENDAR BYTEA NOT NULL,
    PRIMARY KEY (SCHED_NAME, CALENDAR_NAME)
);

CREATE TABLE transaction.QRTZ_PAUSED_TRIGGER_GRPS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    PRIMARY KEY (SCHED_NAME, TRIGGER_GROUP)
);

CREATE TABLE transaction.QRTZ_FIRED_TRIGGERS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    ENTRY_ID VARCHAR(95) NOT NULL,
    TRIGGER_NAME VARCHAR(200) NOT NULL,
    TRIGGER_GROUP VARCHAR(200) NOT NULL,
    INSTANCE_NAME VARCHAR(200) NOT NULL,
    FIRED_TIME BIGINT NOT NULL,
    SCHED_TIME BIGINT NOT NULL,
    PRIORITY INTEGER NOT NULL,
    STATE VARCHAR(16) NOT NULL,
    JOB_NAME VARCHAR(200) NULL,
    JOB_GROUP VARCHAR(200) NULL,
    IS_NONCONCURRENT BOOLEAN NULL,
    REQUESTS_RECOVERY BOOLEAN NULL,
    PRIMARY KEY (SCHED_NAME, ENTRY_ID)
);

CREATE TABLE transaction.QRTZ_SCHEDULER_STATE (
    SCHED_NAME VARCHAR(120) NOT NULL,
    INSTANCE_NAME VARCHAR(200) NOT NULL,
    LAST_CHECKIN_TIME BIGINT NOT NULL,
    CHECKIN_INTERVAL BIGINT NOT NULL,
    PRIMARY KEY (SCHED_NAME, INSTANCE_NAME)
);

CREATE TABLE transaction.QRTZ_LOCKS (
    SCHED_NAME VARCHAR(120) NOT NULL,
    LOCK_NAME VARCHAR(40) NOT NULL,
    PRIMARY KEY (SCHED_NAME, LOCK_NAME)
);

-- Indexes for Quartz tables
CREATE INDEX idx_qrtz_j_req_recovery ON transaction.QRTZ_JOB_DETAILS(SCHED_NAME, REQUESTS_RECOVERY);
CREATE INDEX idx_qrtz_j_grp ON transaction.QRTZ_JOB_DETAILS(SCHED_NAME, JOB_GROUP);
CREATE INDEX idx_qrtz_t_j ON transaction.QRTZ_TRIGGERS(SCHED_NAME, JOB_NAME, JOB_GROUP);
CREATE INDEX idx_qrtz_t_jg ON transaction.QRTZ_TRIGGERS(SCHED_NAME, JOB_GROUP);
CREATE INDEX idx_qrtz_t_c ON transaction.QRTZ_TRIGGERS(SCHED_NAME, CALENDAR_NAME);
CREATE INDEX idx_qrtz_t_g ON transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_GROUP);
CREATE INDEX idx_qrtz_t_state ON transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_STATE);
CREATE INDEX idx_qrtz_t_n_state ON transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP, TRIGGER_STATE);
CREATE INDEX idx_qrtz_t_n_g_state ON transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_GROUP, TRIGGER_STATE);
CREATE INDEX idx_qrtz_t_next_fire_time ON transaction.QRTZ_TRIGGERS(SCHED_NAME, NEXT_FIRE_TIME);
CREATE INDEX idx_qrtz_t_nft_st ON transaction.QRTZ_TRIGGERS(SCHED_NAME, TRIGGER_STATE, NEXT_FIRE_TIME);
CREATE INDEX idx_qrtz_t_nft_misfire ON transaction.QRTZ_TRIGGERS(SCHED_NAME, MISFIRE_INSTR, NEXT_FIRE_TIME);
CREATE INDEX idx_qrtz_t_nft_st_misfire ON transaction.QRTZ_TRIGGERS(SCHED_NAME, MISFIRE_INSTR, NEXT_FIRE_TIME, TRIGGER_STATE);
CREATE INDEX idx_qrtz_t_nft_st_misfire_grp ON transaction.QRTZ_TRIGGERS(SCHED_NAME, MISFIRE_INSTR, NEXT_FIRE_TIME, TRIGGER_GROUP, TRIGGER_STATE);
CREATE INDEX idx_qrtz_ft_trig_inst_name ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, INSTANCE_NAME);
CREATE INDEX idx_qrtz_ft_inst_job_req_rcvry ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, INSTANCE_NAME, REQUESTS_RECOVERY);
CREATE INDEX idx_qrtz_ft_j_g ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, JOB_NAME, JOB_GROUP);
CREATE INDEX idx_qrtz_ft_jg ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, JOB_GROUP);
CREATE INDEX idx_qrtz_ft_t_g ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, TRIGGER_NAME, TRIGGER_GROUP);
CREATE INDEX idx_qrtz_ft_tg ON transaction.QRTZ_FIRED_TRIGGERS(SCHED_NAME, TRIGGER_GROUP);
```


### Dockerfiles

#### Platform Service Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app

# Copy pom.xml first for dependency caching
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests -B

# Runtime stage
FROM eclipse-temurin:21-jre-alpine

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy JAR from builder
COPY --from=builder /app/target/*.jar app.jar

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# JVM settings for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Entry point
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

#### Transaction Service Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app

# Copy pom.xml first for dependency caching
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests -B

# Runtime stage
FROM eclipse-temurin:21-jre-alpine

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy JAR from builder
COPY --from=builder /app/target/*.jar app.jar

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# JVM settings for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Entry point
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Kubernetes Manifests (Kustomize Structure)

#### Directory Structure

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── configmap-patch.yaml
    │   └── deployment-patch.yaml
    ├── test/
    │   ├── kustomization.yaml
    │   ├── configmap-patch.yaml
    │   └── deployment-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── configmap-patch.yaml
    │   └── deployment-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── configmap-patch.yaml
        └── deployment-patch.yaml
```

#### Base Deployment (k8s/base/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-service
  labels:
    app: platform-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: platform-service
  template:
    metadata:
      labels:
        app: platform-service
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: platform-service
          image: registry.digitalocean.com/court-booking/platform-service:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: platform-service-config
          env:
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: platform-service-secrets
                  key: db-password
            - name: SPRING_REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: platform-service-secrets
                  key: redis-password
            - name: SPRING_KAFKA_SASL_JAAS_CONFIG
              valueFrom:
                secretKeyRef:
                  name: platform-service-secrets
                  key: kafka-sasl-config
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 12
```


#### Base Service (k8s/base/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: platform-service
  labels:
    app: platform-service
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: platform-service
```

#### Base ConfigMap (k8s/base/configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-service-config
data:
  SPRING_PROFILES_ACTIVE: "dev"
  SERVER_PORT: "8080"
  JAVA_OPTS: "-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"
```

#### Base Kustomization (k8s/base/kustomization.yaml)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

#### Production Overlay (k8s/overlays/production/kustomization.yaml)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

patches:
  - path: deployment-patch.yaml
  - path: configmap-patch.yaml

images:
  - name: registry.digitalocean.com/court-booking/platform-service
    newTag: v1.0.0
```

#### Production Deployment Patch (k8s/overlays/production/deployment-patch.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: platform-service
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 1000m
              memory: 2Gi
```

#### Production ConfigMap Patch (k8s/overlays/production/configmap-patch.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-service-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://db-postgresql-fra1-court-booking-do-user-xxxxx.db.ondigitalocean.com:25060/courtbooking?sslmode=require"
  SPRING_REDIS_HOST: "redis-fra1-court-booking-do-user-xxxxx.db.ondigitalocean.com"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "redpanda-xxxxx.any.eu-central-1.mpx.prd.cloud.redpanda.com:9092"
```

#### Ingress Resource (shared across services)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: court-booking-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  tls:
    - hosts:
        - api.courtbooking.gr
      secretName: court-booking-tls
  rules:
    - host: api.courtbooking.gr
      http:
        paths:
          # Platform Service routes
          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/courts
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/weather
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/analytics
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/promo-codes
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/feature-flags
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/admin
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          - path: /api/support
            pathType: Prefix
            backend:
              service:
                name: platform-service
                port:
                  number: 8080
          # Transaction Service routes
          - path: /api/bookings
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          - path: /api/payments
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          - path: /api/notifications
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          - path: /api/waitlist
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          - path: /api/matches
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          - path: /api/split-payments
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
          # WebSocket route
          - path: /ws
            pathType: Prefix
            backend:
              service:
                name: transaction-service
                port:
                  number: 8080
```


### CI/CD Workflows

#### Common Library CI Workflow (.github/workflows/ci.yml)

```yaml
name: Common Library CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build and Test
        run: mvn clean verify -B

      - name: Publish to GitHub Packages
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: mvn deploy -B -DskipTests
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Platform Service CI Workflow (.github/workflows/ci.yml)

```yaml
name: Platform Service CI

on:
  pull_request:
    branches: [develop, main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:15-3.3
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: courtbooking_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Create test schemas
        run: |
          PGPASSWORD=test psql -h localhost -U test -d courtbooking_test -c "CREATE SCHEMA IF NOT EXISTS platform;"
          PGPASSWORD=test psql -h localhost -U test -d courtbooking_test -c "CREATE EXTENSION IF NOT EXISTS postgis;"

      - name: Build and Test
        run: mvn clean verify -B
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/courtbooking_test
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

      - name: Flyway Validate
        run: |
          mvn flyway:validate -Dflyway.url=jdbc:postgresql://localhost:5432/courtbooking_test \
            -Dflyway.user=test -Dflyway.password=test -Dflyway.schemas=platform

      - name: Build Docker Image
        run: docker build -t platform-service:${{ github.sha }} .

      - name: Trivy Vulnerability Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: platform-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true

      - name: Upload Trivy Results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```


#### Platform Service Deploy Workflow (.github/workflows/deploy.yml)

```yaml
name: Platform Service Deploy

on:
  push:
    branches: [develop]

env:
  REGISTRY: registry.digitalocean.com/court-booking
  IMAGE_NAME: platform-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Build JAR
        run: mvn clean package -DskipTests -B
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Login to DOCR
        run: doctl registry login --expiry-seconds 600

      - name: Build and Push Docker Image
        id: meta
        run: |
          IMAGE_TAG=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker build -t $IMAGE_TAG .
          docker push $IMAGE_TAG
          echo "tags=$IMAGE_TAG" >> $GITHUB_OUTPUT

  deploy-dev:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Configure kubectl
        run: doctl kubernetes cluster kubeconfig save court-booking-cluster

      - name: Deploy to dev
        run: |
          cd k8s/overlays/dev
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl apply -k .

      - name: Wait for rollout
        run: kubectl rollout status deployment/platform-service -n dev --timeout=300s

  deploy-test:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: test
    steps:
      - uses: actions/checkout@v4

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Configure kubectl
        run: doctl kubernetes cluster kubeconfig save court-booking-cluster

      - name: Deploy to test
        run: |
          cd k8s/overlays/test
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl apply -k .

      - name: Wait for rollout
        run: kubectl rollout status deployment/platform-service -n test --timeout=300s

  trigger-qa:
    needs: deploy-test
    runs-on: ubuntu-latest
    steps:
      - name: Trigger QA Functional Regression
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.QA_DISPATCH_TOKEN }}
          repository: OWNER/court-booking-qa
          event-type: run-functional-regression
          client-payload: |
            {
              "service": "court-booking-platform-service",
              "version": "${{ github.sha }}",
              "environment": "test",
              "triggered_by": "${{ github.actor }}",
              "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }

  deploy-staging:
    needs: trigger-qa
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging-api.courtbooking.gr
    steps:
      - uses: actions/checkout@v4

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Configure kubectl
        run: doctl kubernetes cluster kubeconfig save court-booking-cluster

      - name: Deploy to staging
        run: |
          cd k8s/overlays/staging
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl apply -k .

      - name: Wait for rollout
        run: kubectl rollout status deployment/platform-service -n staging --timeout=300s

      - name: Trigger QA Staging Validation
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.QA_DISPATCH_TOKEN }}
          repository: OWNER/court-booking-qa
          event-type: run-staging-validation
          client-payload: |
            {
              "service": "court-booking-platform-service",
              "version": "${{ github.sha }}",
              "environment": "staging"
            }

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.courtbooking.gr
    steps:
      - uses: actions/checkout@v4

      - name: Install doctl
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}

      - name: Configure kubectl
        run: doctl kubernetes cluster kubeconfig save court-booking-cluster

      - name: Tag release candidate
        run: |
          doctl registry login --expiry-seconds 600
          docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:v1.0.0
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:v1.0.0

      - name: Deploy to production
        run: |
          cd k8s/overlays/production
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:v1.0.0
          kubectl apply -k .

      - name: Wait for rollout
        run: kubectl rollout status deployment/platform-service -n production --timeout=300s
```


### Local Development Setup

#### Docker Compose (infrastructure/docker-compose.yml)

```yaml
version: '3.8'

services:
  postgres:
    image: postgis/postgis:15-3.3
    container_name: court-booking-postgres
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: courtbooking
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d courtbooking"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: court-booking-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.3
    container_name: court-booking-kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 10s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.3
    container_name: court-booking-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

volumes:
  postgres_data:
  redis_data:
```

#### Database Init Script (infrastructure/init-db.sql)

```sql
-- Create schemas
CREATE SCHEMA IF NOT EXISTS platform;
CREATE SCHEMA IF NOT EXISTS transaction;

-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Grant permissions to dev user
GRANT ALL PRIVILEGES ON SCHEMA platform TO dev;
GRANT ALL PRIVILEGES ON SCHEMA transaction TO dev;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA platform TO dev;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA transaction TO dev;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA platform TO dev;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA transaction TO dev;

-- Set default schema search path
ALTER USER dev SET search_path TO platform, transaction, public;
```

#### Sample Data Seeding (V999__seed_local_data.sql)

This migration only runs in the local profile via conditional Flyway locations.

**Platform Service (src/main/resources/db/migration/platform/local/V999__seed_local_data.sql)**

```sql
-- Sample Users
INSERT INTO platform.users (id, email, name, phone, role, language, verified, status, terms_accepted_at)
VALUES 
    ('11111111-1111-1111-1111-111111111111', 'customer@example.com', 'Demo Customer', '+30 6900000001', 'CUSTOMER', 'el', false, 'ACTIVE', NOW()),
    ('22222222-2222-2222-2222-222222222222', 'owner@example.com', 'Demo Court Owner', '+30 6900000002', 'COURT_OWNER', 'el', true, 'ACTIVE', NOW())
ON CONFLICT (email) DO NOTHING;

-- Sample Courts
INSERT INTO platform.courts (id, owner_id, name_el, name_en, court_type, location_type, location, address, timezone, base_price_cents, duration_minutes, max_capacity, confirmation_mode, visible)
VALUES 
    ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', '22222222-2222-2222-2222-222222222222', 'Demo Tennis Club', 'Demo Tennis Club', 'TENNIS', 'OUTDOOR', ST_SetSRID(ST_MakePoint(23.7275, 37.9838), 4326), 'Athens, Greece', 'Europe/Athens', 2500, 60, 4, 'INSTANT', true),
    ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb', '22222222-2222-2222-2222-222222222222', 'Demo Padel Center', 'Demo Padel Center', 'PADEL', 'INDOOR', ST_SetSRID(ST_MakePoint(23.7300, 37.9850), 4326), 'Athens, Greece', 'Europe/Athens', 3000, 90, 4, 'INSTANT', true),
    ('cccccccc-cccc-cccc-cccc-cccccccccccc', '22222222-2222-2222-2222-222222222222', 'Demo Basketball Court', 'Demo Basketball Court', 'BASKETBALL', 'OUTDOOR', ST_SetSRID(ST_MakePoint(23.7250, 37.9820), 4326), 'Athens, Greece', 'Europe/Athens', 4000, 60, 10, 'MANUAL', true)
ON CONFLICT (id) DO NOTHING;

-- Sample Availability Windows (Mon-Sun 08:00-22:00)
INSERT INTO platform.availability_windows (court_id, day_of_week, start_time, end_time)
SELECT c.id, d.day, '08:00'::TIME, '22:00'::TIME
FROM platform.courts c
CROSS JOIN (VALUES ('MONDAY'), ('TUESDAY'), ('WEDNESDAY'), ('THURSDAY'), ('FRIDAY'), ('SATURDAY'), ('SUNDAY')) AS d(day)
ON CONFLICT DO NOTHING;

-- Sample Pricing Rules (Peak hours 18:00-21:00 weekdays)
INSERT INTO platform.pricing_rules (court_id, day_of_week, start_time, end_time, multiplier, label)
SELECT c.id, d.day, '18:00'::TIME, '21:00'::TIME, 1.50, 'Peak Hours'
FROM platform.courts c
CROSS JOIN (VALUES ('MONDAY'), ('TUESDAY'), ('WEDNESDAY'), ('THURSDAY'), ('FRIDAY')) AS d(day)
ON CONFLICT DO NOTHING;

-- Sample Cancellation Tiers
INSERT INTO platform.cancellation_tiers (court_id, threshold_hours, refund_percent, sort_order)
SELECT c.id, t.hours, t.percent, t.ord
FROM platform.courts c
CROSS JOIN (VALUES (48, 100, 1), (24, 50, 2), (0, 0, 3)) AS t(hours, percent, ord)
ON CONFLICT DO NOTHING;
```

**Transaction Service (src/main/resources/db/migration/transaction/local/V999__seed_local_data.sql)**

```sql
-- Sample Bookings
INSERT INTO transaction.bookings (id, court_id, user_id, court_owner_id, idempotency_key, date, start_time, end_time, duration_minutes, status, booking_type, confirmation_mode, total_amount_cents, payment_status)
VALUES 
    ('dddddddd-dddd-dddd-dddd-dddddddddddd', 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', '11111111-1111-1111-1111-111111111111', '22222222-2222-2222-2222-222222222222', 'demo-booking-1', CURRENT_DATE + INTERVAL '1 day', '10:00', '11:00', 60, 'CONFIRMED', 'CUSTOMER', 'INSTANT', 2500, 'CAPTURED'),
    ('eeeeeeee-eeee-eeee-eeee-eeeeeeeeeeee', 'bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb', '11111111-1111-1111-1111-111111111111', '22222222-2222-2222-2222-222222222222', 'demo-booking-2', CURRENT_DATE - INTERVAL '7 days', '14:00', '15:30', 90, 'COMPLETED', 'CUSTOMER', 'INSTANT', 3000, 'CAPTURED')
ON CONFLICT (idempotency_key) DO NOTHING;
```

**Flyway Configuration for Local Profile**

In `application-local.yml`:

```yaml
spring:
  flyway:
    locations: classpath:db/migration/platform,classpath:db/migration/platform/local
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: DTO JSON Serialization Produces camelCase Field Names

*For any* DTO class in the common library, when serialized to JSON using Jackson, all field names in the output SHALL be in camelCase format (e.g., `totalAmountCents`, not `total_amount_cents` or `TotalAmountCents`).

**Validates: Requirements 2.3**

### Property 2: Monetary Fields Use Cents Suffix Convention

*For any* DTO class containing monetary amounts, the field name SHALL end with `Cents` and the field type SHALL be `int` or `Integer`. This ensures consistent representation of euro-cents across all services.

**Validates: Requirements 2.4**

### Property 3: Event Payload Serialization Round-Trip

*For any* valid event payload object, serializing to JSON and deserializing back SHALL produce an object equivalent to the original. This validates that Jackson annotations are correctly configured and no data is lost during serialization.

**Validates: Requirements 3.4, 3.5**

### Property 4: MoneyUtil Arithmetic Correctness

*For any* valid euro-cents amounts (non-negative integers within safe bounds), the MoneyUtil arithmetic operations SHALL:
- `add(a, b)` returns `a + b` without overflow
- `subtract(a, b)` returns `a - b` without underflow
- `multiplyByPercentage(amount, percentage)` returns correctly rounded result for percentage in [0, 100]
- `calculatePlatformFee(amount, feePercent)` equals `multiplyByPercentage(amount, feePercent)`
- `calculateNetAmount(gross, fee)` equals `subtract(gross, fee)`

**Validates: Requirements 4.3**

### Property 5: Structured Log Entries Contain Required Fields

*For any* log entry produced by either service in cloud profiles (dev, test, staging, prod), the JSON output SHALL contain these top-level fields: `timestamp` (ISO 8601), `level`, `logger`, `message`, and `service`. When in a request context, the entry SHALL also contain `traceId`, `spanId`, and `requestId` (if X-Request-ID header was present).

**Validates: Requirements 27.1, 27.2, 27.3**

## Error Handling

### Global Exception Handler

Both services implement a `@RestControllerAdvice` that maps domain exceptions to HTTP responses:

```java
package gr.courtbooking.platform.adapter.in.web;

import gr.courtbooking.common.dto.ErrorResponse;
import gr.courtbooking.common.exception.*;
import org.slf4j.MDC;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), getTraceId()));
    }

    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<ErrorResponse> handleConflict(ConflictException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), getTraceId()));
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), getTraceId()));
    }

    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), getTraceId()));
    }

    @ExceptionHandler(ForbiddenException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(ForbiddenException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), getTraceId()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .reduce((a, b) -> a + "; " + b)
            .orElse("Validation failed");
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("VALIDATION_ERROR", message, getTraceId()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred", getTraceId()));
    }

    private String getTraceId() {
        return MDC.get("traceId") != null ? MDC.get("traceId") : "unknown";
    }
}
```

### Health Check Error Handling

The readiness probe checks database, Redis, and Kafka connectivity. If any dependency is unreachable, the probe returns HTTP 503:

```java
package gr.courtbooking.platform.config;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public KafkaHealthIndicator(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @Override
    public Health health() {
        try {
            kafkaTemplate.getProducerFactory().createProducer().metrics();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```


## Testing Strategy

### Dual Testing Approach

This phase uses both unit tests and property-based tests:

- **Unit tests**: Verify specific examples, edge cases, and integration points
- **Property-based tests (jqwik)**: Verify universal properties across many generated inputs

### Property-Based Testing Configuration

- Library: **jqwik** (Java property-based testing framework)
- Minimum iterations: **100 per property test**
- Each property test references its design document property via tag comment

### Test Structure

```
src/test/java/
├── gr/courtbooking/common/
│   ├── dto/
│   │   └── DtoSerializationProperties.java    # Property tests for JSON serialization
│   ├── event/
│   │   └── EventEnvelopeProperties.java       # Property tests for event round-trip
│   └── util/
│       └── MoneyUtilProperties.java           # Property tests for arithmetic
├── gr/courtbooking/platform/
│   ├── adapter/in/web/
│   │   └── HealthInfoControllerTest.java      # @WebMvcTest slice test
│   └── config/
│       └── LoggingConfigurationTest.java      # Verify log format
└── gr/courtbooking/transaction/
    └── config/
        └── QuartzConfigurationTest.java       # Verify Quartz clustering config
```

### Example Property Test (MoneyUtil)

```java
package gr.courtbooking.common.util;

import net.jqwik.api.*;
import static org.assertj.core.api.Assertions.*;

/**
 * Feature: phase-1b-service-scaffolding, Property 4: MoneyUtil Arithmetic Correctness
 */
class MoneyUtilProperties {

    @Property(tries = 100)
    void additionShouldBeCommutative(@ForAll @IntRange(min = 0, max = 1_000_000) int a,
                                      @ForAll @IntRange(min = 0, max = 1_000_000) int b) {
        assertThat(MoneyUtil.add(a, b)).isEqualTo(MoneyUtil.add(b, a));
    }

    @Property(tries = 100)
    void subtractionShouldBeInverseOfAddition(@ForAll @IntRange(min = 0, max = 1_000_000) int a,
                                               @ForAll @IntRange(min = 0, max = 1_000_000) int b) {
        Assume.that(a >= b);
        int sum = MoneyUtil.add(a, b);
        assertThat(MoneyUtil.subtract(sum, b)).isEqualTo(a);
    }

    @Property(tries = 100)
    void percentageMultiplicationShouldBeInRange(@ForAll @IntRange(min = 0, max = 1_000_000) int amount,
                                                  @ForAll @IntRange(min = 0, max = 100) int percentage) {
        int result = MoneyUtil.multiplyByPercentage(amount, percentage);
        assertThat(result).isBetween(0, amount);
    }

    @Property(tries = 100)
    void netAmountPlusFeeEqualsGross(@ForAll @IntRange(min = 100, max = 1_000_000) int gross,
                                      @ForAll @IntRange(min = 1, max = 30) int feePercent) {
        int fee = MoneyUtil.calculatePlatformFee(gross, feePercent);
        int net = MoneyUtil.calculateNetAmount(gross, fee);
        assertThat(MoneyUtil.add(net, fee)).isEqualTo(gross);
    }
}
```

### Example Property Test (Event Serialization Round-Trip)

```java
package gr.courtbooking.common.event;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import gr.courtbooking.common.event.booking.BookingCreatedPayload;
import net.jqwik.api.*;
import java.time.Instant;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

/**
 * Feature: phase-1b-service-scaffolding, Property 3: Event Payload Serialization Round-Trip
 */
class EventEnvelopeProperties {

    private final ObjectMapper objectMapper = new ObjectMapper()
        .registerModule(new JavaTimeModule());

    @Property(tries = 100)
    void eventEnvelopeRoundTrip(@ForAll("bookingCreatedPayloads") BookingCreatedPayload payload) throws Exception {
        EventEnvelope<BookingCreatedPayload> original = EventEnvelope.create(
            "BOOKING_CREATED",
            "transaction-service",
            UUID.randomUUID().toString(),
            UUID.randomUUID().toString().substring(0, 16),
            null,
            payload
        );

        String json = objectMapper.writeValueAsString(original);
        EventEnvelope<?> deserialized = objectMapper.readValue(json, EventEnvelope.class);

        assertThat(deserialized.eventId()).isEqualTo(original.eventId());
        assertThat(deserialized.eventType()).isEqualTo(original.eventType());
        assertThat(deserialized.source()).isEqualTo(original.source());
    }

    @Provide
    Arbitrary<BookingCreatedPayload> bookingCreatedPayloads() {
        return Combinators.combine(
            Arbitraries.create(UUID::randomUUID),  // bookingId
            Arbitraries.create(UUID::randomUUID),  // courtId
            Arbitraries.create(UUID::randomUUID),  // userId
            Arbitraries.create(UUID::randomUUID),  // courtOwnerId
            Arbitraries.strings().ofMinLength(10).ofMaxLength(10),  // date
            Arbitraries.strings().ofMinLength(5).ofMaxLength(5),    // startTime
            Arbitraries.strings().ofMinLength(5).ofMaxLength(5),    // endTime
            Arbitraries.integers().between(30, 180),                // durationMinutes
            Arbitraries.of("CONFIRMED", "PENDING_CONFIRMATION"),    // status
            Arbitraries.of("CUSTOMER", "MANUAL")                    // bookingType
        ).as((bookingId, courtId, userId, courtOwnerId, date, startTime, endTime, duration, status, type) ->
            new BookingCreatedPayload(bookingId, courtId, userId, courtOwnerId, date, startTime, endTime,
                duration, status, type, 2500, false, false, null)
        );
    }
}
```

### Slice Tests

**Controller Test (@WebMvcTest)**

```java
package gr.courtbooking.platform.adapter.in.web;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(HealthInfoController.class)
class HealthInfoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnServiceInfo() throws Exception {
        mockMvc.perform(get("/api/health/info"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.service").exists())
            .andExpect(jsonPath("$.status").value("UP"));
    }
}
```

### Integration Tests (Testcontainers)

```java
package gr.courtbooking.platform;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class PlatformServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgis/postgis:15-3.3")
        .withDatabaseName("courtbooking_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void healthEndpointShouldReturnUp() {
        ResponseEntity<String> response = restTemplate.getForEntity("/actuator/health", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

