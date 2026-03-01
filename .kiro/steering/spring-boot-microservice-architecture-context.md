---
inclusion: manual
---

# Spring Boot 4.x & Java 25+ Microservice Architecture Context

> When designing or implementing any Spring Boot microservice in this project, follow the patterns and standards in this document. It codifies the hexagonal architecture from Buckpal (thombergs), the testing rigor from rieckpil's Masterclass, and operational best practices from Spring PetClinic Microservices and abhisheksr01's best-practices repo.

## 1. Hexagonal Architecture (Ports & Adapters)

### Layer Separation

```
┌─────────────────────────────────────────┐
│         Infrastructure Layer            │
│  (Adapters: Web, Persistence, External) │
├─────────────────────────────────────────┤
│         Application Layer               │
│    (Use Cases, Ports/Interfaces)        │
├─────────────────────────────────────────┤
│           Domain Layer                  │
│  (Entities, Value Objects, Logic)       │
└─────────────────────────────────────────┘
```

### Dependency Rules

| Direction | Allowed? |
|-----------|----------|
| Domain → Nothing (pure Java) | ✅ |
| Application → Domain | ✅ |
| Adapter → Application + Domain | ✅ |
| Domain → Application | ❌ |
| Domain → Adapter | ❌ |
| Application → Adapter | ❌ |

### Package Structure (Buckpal Style)

```
src/main/java/com/company/bookingplatform/
├── domain/                          # Pure business logic — NO framework imports
│   ├── model/
│   │   ├── Booking.java            # Entity (rich, with business methods)
│   │   ├── BookingId.java          # Value Object (Record)
│   │   ├── BookingStatus.java      # Enum
│   │   ├── DateRange.java          # Value Object (Record)
│   │   └── Money.java              # Value Object (Record)
│   ├── exception/
│   │   └── BookingConflictException.java
│   └── service/
│       └── BookingDomainService.java  # Cross-entity domain logic
│
├── application/                     # Use Cases & Ports
│   ├── port/
│   │   ├── in/                     # Incoming Ports (Use Case interfaces)
│   │   │   ├── CreateBookingUseCase.java
│   │   │   ├── CancelBookingUseCase.java
│   │   │   └── GetBookingQuery.java
│   │   └── out/                    # Outgoing Ports (SPI interfaces)
│   │       ├── LoadBookingPort.java
│   │       ├── SaveBookingPort.java
│   │       └── SendNotificationPort.java
│   └── service/                    # Use Case implementations
│       ├── CreateBookingService.java
│       └── CancelBookingService.java
│
└── adapter/                        # Infrastructure implementations
    ├── in/
    │   └── web/
    │       ├── BookingController.java
    │       ├── dto/
    │       │   ├── CreateBookingRequest.java   # Record
    │       │   └── BookingResponse.java        # Record
    │       └── mapper/
    │           └── BookingWebMapper.java
    ├── out/
    │   ├── persistence/
    │   │   ├── entity/
    │   │   │   └── BookingJpaEntity.java
    │   │   ├── repository/
    │   │   │   └── BookingRepository.java
    │   │   ├── mapper/
    │   │   │   └── BookingPersistenceMapper.java
    │   │   └── BookingPersistenceAdapter.java
    │   ├── messaging/
    │   │   └── KafkaNotificationAdapter.java
    │   └── external/
    │       └── StripePaymentAdapter.java
    └── config/
        └── BeanConfiguration.java
```


## 2. Domain Layer Patterns

### Rich Domain Entities (NOT anemic)

Business logic lives inside domain objects. Entities are not just data holders.

Domain entities should provide two construction paths (Buckpal pattern):
1. **Creation**: Constructor for new entities (no ID yet, assigned by persistence)
2. **Reconstitution**: Static factory method for loading existing entities from the database

```java
public class Booking {
    private final BookingId id;
    private final CourtId courtId;
    private final UserId userId;
    private final DateRange dateRange;
    private BookingStatus status;

    // Creation — new booking, no ID yet
    public Booking(CourtId courtId, UserId userId, DateRange dateRange) {
        this.id = null;
        this.courtId = requireNonNull(courtId);
        this.userId = requireNonNull(userId);
        this.dateRange = requireNonNull(dateRange);
        this.status = BookingStatus.PENDING;
        validateBooking();
    }

    // Reconstitution — loading from database (used by persistence mapper)
    public static Booking withId(BookingId id, CourtId courtId, UserId userId,
                                  DateRange dateRange, BookingStatus status) {
        Booking booking = new Booking(courtId, userId, dateRange);
        // Use reflection or package-private setter to assign id and status
        return booking;
    }

    public Optional<BookingId> getId() {
        return Optional.ofNullable(this.id);
    }

    public void confirm() {
        if (status != BookingStatus.PENDING) {
            throw new IllegalStateException("Only pending bookings can be confirmed");
        }
        this.status = BookingStatus.CONFIRMED;
    }

    public void cancel() {
        if (status == BookingStatus.CANCELLED) {
            throw new IllegalStateException("Booking already cancelled");
        }
        this.status = BookingStatus.CANCELLED;
    }

    private void validateBooking() {
        if (dateRange.isInPast()) {
            throw new IllegalArgumentException("Cannot book in the past");
        }
    }
}
```

### Value Objects as Java Records

Records enforce immutability and include validation in compact constructors.

```java
public record BookingId(Long value) {
    public BookingId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("Invalid booking ID");
        }
    }
}

public record DateRange(LocalDateTime start, LocalDateTime end) {
    public DateRange {
        requireNonNull(start, "Start time cannot be null");
        requireNonNull(end, "End time cannot be null");
        if (start.isAfter(end)) {
            throw new IllegalArgumentException("Start must be before end");
        }
    }

    public boolean isInPast() {
        return end.isBefore(LocalDateTime.now());
    }

    public boolean overlaps(DateRange other) {
        return !this.end.isBefore(other.start) && !other.end.isBefore(this.start);
    }
}
```

## 3. Application Layer Patterns

### Incoming Ports (one interface per use case)

```java
public interface CreateBookingUseCase {
    BookingId createBooking(CreateBookingCommand command);
}
```

### Self-Validating Commands (Buckpal Pattern)

Commands validate their own inputs in the constructor. This keeps validation noise out of use case code.

```java
public record CreateBookingCommand(
    CourtId courtId,
    UserId userId,
    DateRange dateRange
) {
    public CreateBookingCommand {
        requireNonNull(courtId, "courtId must not be null");
        requireNonNull(userId, "userId must not be null");
        requireNonNull(dateRange, "dateRange must not be null");
    }
}
```

### Outgoing Ports (SPI)

```java
public interface LoadBookingPort {
    Optional<Booking> loadById(BookingId id);
    List<Booking> loadByFieldAndDateRange(CourtId courtId, DateRange dateRange);
}

public interface SaveBookingPort {
    BookingId save(Booking booking);
}
```

### Use Case Implementation

Use cases orchestrate domain objects and outgoing ports. They are annotated with `@Component` (or `@Service`) and `@Transactional`. The use case owns the transaction boundary — not the adapter.

```java
@Service
@Transactional
@RequiredArgsConstructor
public class CreateBookingService implements CreateBookingUseCase {

    private final LoadBookingPort loadBookingPort;
    private final SaveBookingPort saveBookingPort;
    private final SendNotificationPort notificationPort;

    @Override
    public BookingId createBooking(CreateBookingCommand command) {
        List<Booking> conflicts = loadBookingPort
            .loadByFieldAndDateRange(command.courtId(), command.dateRange());

        if (!conflicts.isEmpty()) {
            throw new BookingConflictException("Court already booked for this time");
        }

        Booking booking = new Booking(
            command.courtId(), command.userId(), command.dateRange()
        );

        BookingId bookingId = saveBookingPort.save(booking);
        notificationPort.sendBookingConfirmation(booking);
        return bookingId;
    }
}
```

### Locking Pattern (from Buckpal's SendMoneyService)

For operations requiring concurrency control (like booking a time slot), acquire a lock before mutating state and release it after persisting:

```java
@Override
public BookingId createBooking(CreateBookingCommand command) {
    timeSlotLock.lockSlot(command.courtId(), command.dateRange());
    try {
        // ... validate, create, persist
        return bookingId;
    } finally {
        timeSlotLock.releaseSlot(command.courtId(), command.dateRange());
    }
}
```

## 4. Adapter Layer Patterns

### Web Adapter (Incoming)

```java
@RestController
@RequestMapping("/api/v1/bookings")
@RequiredArgsConstructor
public class BookingController {

    private final CreateBookingUseCase createBookingUseCase;
    private final GetBookingQuery getBookingQuery;

    @PostMapping
    public ResponseEntity<BookingResponse> createBooking(
            @Valid @RequestBody CreateBookingRequest request) {
        CreateBookingCommand command = new CreateBookingCommand(
            new CourtId(request.courtId()),
            new UserId(request.userId()),
            new DateRange(request.startTime(), request.endTime())
        );
        BookingId bookingId = createBookingUseCase.createBooking(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new BookingResponse(bookingId.value()));
    }

    @GetMapping("/{id}")
    public ResponseEntity<BookingResponse> getBooking(@PathVariable Long id) {
        return getBookingQuery.getBooking(new BookingId(id))
            .map(BookingWebMapper::toResponse)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

### Persistence Adapter (Outgoing)

```java
@Component
@RequiredArgsConstructor
class BookingPersistenceAdapter implements LoadBookingPort, SaveBookingPort {

    private final BookingRepository repository;
    private final BookingPersistenceMapper mapper;

    @Override
    public Optional<Booking> loadById(BookingId id) {
        return repository.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public List<Booking> loadByFieldAndDateRange(CourtId courtId, DateRange dateRange) {
        return repository.findByCourtIdAndDateRange(
                courtId.value(), dateRange.start(), dateRange.end())
            .stream()
            .map(mapper::toDomain)
            .toList();
    }

    @Override
    public BookingId save(Booking booking) {
        BookingJpaEntity saved = repository.save(mapper.toEntity(booking));
        return new BookingId(saved.getId());
    }
}
```

### JPA Entity (Adapter-only, never leaks into Domain)

```java
@Entity
@Table(name = "bookings", indexes = {
    @Index(name = "idx_field_date", columnList = "court_id,start_time,end_time"),
    @Index(name = "idx_user", columnList = "user_id")
})
@Getter
@Setter
@NoArgsConstructor
public class BookingJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "court_id", nullable = false)
    private Long courtId;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "start_time", nullable = false)
    private LocalDateTime startTime;

    @Column(name = "end_time", nullable = false)
    private LocalDateTime endTime;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private BookingStatus status;

    @Version
    private Long version;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}
```

### Boundary Mapping (MapStruct)

```java
@Mapper(componentModel = "spring")
public interface BookingPersistenceMapper {
    Booking toDomain(BookingJpaEntity entity);
    BookingJpaEntity toEntity(Booking domain);
}
```


## 5. Code Style Rules

### Constructor Injection Only

```java
// ✅ CORRECT
@Service
@RequiredArgsConstructor
public class BookingService {
    private final BookingRepository repository;
    private final NotificationService notificationService;
}

// ❌ FORBIDDEN — no field injection
@Autowired private BookingRepository repository;
```

### Functional Programming Over Imperative

```java
// ✅ Streams and Optionals
public List<BookingResponse> getActiveBookings(Long userId) {
    return repository.findByUserId(userId).stream()
        .filter(b -> b.getStatus() != BookingStatus.CANCELLED)
        .map(this::toResponse)
        .toList();
}

// ❌ No imperative loops with mutable lists
```

### Optional Over Null

```java
// ✅ Return Optional, chain with map/orElseThrow
public BookingResponse getBookingOrThrow(BookingId id) {
    return findBooking(id)
        .map(this::toResponse)
        .orElseThrow(() -> new BookingNotFoundException(id));
}

// ❌ Never return null from a method
```

### Lombok — Sparingly

| Annotation | Where | Allowed? |
|------------|-------|----------|
| `@Slf4j` | Any Spring bean | ✅ |
| `@RequiredArgsConstructor` | Services, adapters | ✅ |
| `@Getter` / `@Setter` | JPA entities only | ✅ |
| `@NoArgsConstructor` | JPA entities only | ✅ |
| `@Data` | Anywhere | ❌ |
| `@Builder` | Anywhere | ❌ Use Records |
| `@AllArgsConstructor` | Anywhere | ❌ |

### Validation Strategy

- **Domain layer**: Validate in constructors and compact record constructors
- **API layer**: Use Bean Validation annotations (`@NotNull`, `@Positive`, `@Future`)
- **Both layers validate independently** — never trust the adapter to validate for the domain

```java
// API-level validation (adapter)
public record CreateBookingRequest(
    @NotNull @Positive Long courtId,
    @NotNull @Positive Long userId,
    @NotNull @Future LocalDateTime startTime,
    @NotNull @Future LocalDateTime endTime
) {}

// Domain-level validation (domain)
public record DateRange(LocalDateTime start, LocalDateTime end) {
    public DateRange {
        requireNonNull(start, "Start time cannot be null");
        requireNonNull(end, "End time cannot be null");
        if (start.isAfter(end)) {
            throw new IllegalArgumentException("Start must be before end");
        }
    }
}
```

## 6. Testing Standards (rieckpil Masterclass)

### Test Pyramid

```
        /\
       /E2E\      ← Few: Full @SpringBootTest (only for critical end-to-end flows)
      /------\
     /  Slice  \  ← Some: @WebMvcTest, @DataJpaTest (fast, focused)
    /----------\
   /   Unit     \ ← Many: Pure Java, no Spring context (fastest)
  /--------------\
```

### Critical Testing Rules

| Rule | Detail |
|------|--------|
| No H2 for integration tests | Use Testcontainers with real PostgreSQL |
| Avoid @SpringBootTest | Use slice tests (@WebMvcTest, @DataJpaTest) unless testing full flow |
| Test naming | `should[Behavior]When[Condition]()` |
| Assertions | Use AssertJ fluent assertions |
| Coverage target | 80%+ unit test coverage |

### Which Test Slice to Use

| What you're testing | Annotation | Mocking |
|---------------------|-----------|---------|
| Controller endpoints | `@WebMvcTest` | `@MockBean` for use cases |
| JPA repositories / queries | `@DataJpaTest` + Testcontainers | Real PostgreSQL |
| JSON serialization | `@JsonTest` | None |
| REST clients (RestTemplate) | `@RestClientTest` | `MockRestServiceServer` |
| External HTTP APIs | `@SpringBootTest` + `@AutoConfigureWireMock` | WireMock stubs |
| Kafka / async messaging | `@SpringBootTest` + Testcontainers | Awaitility for assertions |
| Full end-to-end flow | `@SpringBootTest(RANDOM_PORT)` + Testcontainers | `TestRestTemplate` |

### Unit Tests (Domain Logic)

No Spring context. Pure Java. Fast.

```java
class BookingTest {

    @Test
    void shouldRejectBookingInThePast() {
        LocalDateTime pastDate = LocalDateTime.now().minusDays(1);
        assertThatThrownBy(() -> new Booking(
            new CourtId(1L), new UserId(1L),
            new DateRange(pastDate, pastDate.plusHours(1))
        )).isInstanceOf(IllegalArgumentException.class)
          .hasMessageContaining("past");
    }
}
```

### Slice Tests

**Controller (@WebMvcTest)**
```java
@WebMvcTest(BookingController.class)
class BookingControllerTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private CreateBookingUseCase createBookingUseCase;

    @Test
    void shouldReturn201WhenBookingCreated() throws Exception {
        given(createBookingUseCase.createBooking(any()))
            .willReturn(new BookingId(1L));

        mockMvc.perform(post("/api/v1/bookings")
                .contentType(APPLICATION_JSON)
                .content("""
                    {"courtId":1,"userId":1,"startTime":"2026-03-01T10:00","endTime":"2026-03-01T11:30"}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

**Repository (@DataJpaTest + Testcontainers)**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class BookingRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private BookingRepository repository;

    @Test
    void shouldFindConflictingBookings() {
        // test with real PostgreSQL
    }
}
```

### Integration Tests (Sparingly)

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class BookingIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private TestRestTemplate restTemplate;

    @Test
    void shouldCreateBookingEndToEnd() {
        ResponseEntity<BookingResponse> response = restTemplate.postForEntity(
            "/api/v1/bookings", request, BookingResponse.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

### External API Testing (WireMock)

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class StripePaymentAdapterTest {

    @Test
    void shouldProcessPaymentSuccessfully() {
        stubFor(post("/payments").willReturn(aResponse()
            .withStatus(200)
            .withBody("{\"status\":\"SUCCESS\"}")));

        PaymentResult result = paymentAdapter.processPayment(/*...*/);
        assertThat(result.status()).isEqualTo(PaymentStatus.SUCCESS);
    }
}
```

### JSON Serialization Testing (@JsonTest)

Test Jackson annotations, custom serializers, and JSON format without loading the full context.

```java
@JsonTest
class BookingResponseTest {

    @Autowired
    private JacksonTester<BookingResponse> jacksonTester;

    @Test
    void shouldSerializeBookingResponse() throws IOException {
        BookingResponse response = new BookingResponse(1L, "CONFIRMED", "2026-03-01T10:00");
        JsonContent<BookingResponse> result = jacksonTester.write(response);

        assertThat(result).extractingJsonPathNumberValue("$.id").isEqualTo(1);
        assertThat(result).extractingJsonPathStringValue("$.status").isEqualTo("CONFIRMED");
    }
}
```

### HTTP Client Testing (@RestClientTest)

Test REST clients in isolation with a mock server — no WireMock needed.

```java
@RestClientTest(WeatherApiClient.class)
class WeatherApiClientTest {

    @Autowired private WeatherApiClient client;
    @Autowired private MockRestServiceServer mockServer;

    @Test
    void shouldReturnWeatherForecast() {
        mockServer.expect(requestTo("/forecast?lat=40.0&lon=3.0"))
            .andRespond(withSuccess("{\"temp\":22}", APPLICATION_JSON));

        WeatherForecast result = client.getForecast(40.0, 3.0);
        assertThat(result.temp()).isEqualTo(22);
    }
}
```

### Async Testing (Awaitility)

```java
@Test
void shouldSendNotificationAfterBookingCreated() {
    eventPublisher.publish(new BookingCreatedEvent(/*...*/));

    await().atMost(Duration.ofSeconds(5)).untilAsserted(() ->
        verify(notificationService).sendEmail(any())
    );
}
```


## 7. API Design Standards

### Spring Cloud Infrastructure (PetClinic Microservices)

When deploying as microservices, use these Spring Cloud components:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Config Server | Spring Cloud Config | Centralized, Git-backed configuration |
| Service Discovery | Eureka Server | Service registration and client-side load balancing |
| API Gateway | Spring Cloud Gateway | Routing, auth, rate limiting |
| Circuit Breaker | Resilience4j | Fault tolerance, fallbacks, bulkhead isolation |
| Distributed Tracing | Micrometer + OpenTelemetry | Request correlation across services |

### RESTful Conventions

```
GET    /api/v1/bookings          - List (paginated)
GET    /api/v1/bookings/{id}     - Get by ID
POST   /api/v1/bookings          - Create
PUT    /api/v1/bookings/{id}     - Full update
PATCH  /api/v1/bookings/{id}     - Partial update
DELETE /api/v1/bookings/{id}     - Delete
```

### Error Response Format

```json
{
  "timestamp": "2026-02-06T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/v1/bookings",
  "errors": [
    { "field": "startTime", "message": "Start time must be in the future" }
  ]
}
```

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(BookingConflictException.class)
    public ResponseEntity<ErrorResponse> handleConflict(BookingConflictException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("Validation failed", errors));
    }
}
```

## 8. Infrastructure & Operations

### Configuration

```yaml
spring:
  application:
    name: court-booking-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}

server:
  port: ${SERVER_PORT:8080}
  servlet:
    context-path: /api/v1

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

**Profile Strategy:**
- `local` — Docker Compose infrastructure
- `dev` — DigitalOcean managed services
- `staging` — Pre-production (mirrors prod)
- `prod` — Production on Kubernetes

### Database

- Flyway for migrations (never Liquibase in this project)
- Optimistic locking via `@Version`
- Proper composite indexes for query patterns
- HikariCP connection pooling (Spring Boot default)

### Observability

- Micrometer + Prometheus for metrics
- OpenTelemetry + Jaeger for distributed tracing
- Loki for log aggregation
- Spring Boot Actuator for health endpoints
- Structured logging with correlation IDs

```java
@Slf4j
@Service
public class CreateBookingService {
    public BookingId createBooking(CreateBookingCommand command) {
        log.info("Creating booking for court={} user={}", command.courtId(), command.userId());
        // ...
        log.debug("Booking created: {}", bookingId);
        return bookingId;
    }
}
```

### Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/v1/**").authenticated()
                .anyRequest().denyAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

### Caching (Redis)

```java
@Service
@CacheConfig(cacheNames = "courtAvailability")
@RequiredArgsConstructor
public class AvailabilityService {

    @Cacheable(key = "#courtId + '-' + #date")
    public List<TimeSlot> getAvailability(Long courtId, LocalDate date) {
        return repository.findAvailableSlots(courtId, date);
    }

    @CacheEvict(key = "#courtId + '-' + #date")
    public void invalidateAvailability(Long courtId, LocalDate date) {
        // Cache cleared on booking changes
    }
}
```

### Containerization

```dockerfile
FROM gradle:8.5-jdk21 AS builder
WORKDIR /app
COPY . .
RUN gradle clean build -x test

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### CI/CD Pipeline Stages

1. Checkout → 2. Build → 3. Unit tests → 4. Static analysis (Checkstyle, SonarQube) → 5. Security scan (OWASP, Trivy) → 6. Docker build → 7. Integration tests (Testcontainers) → 8. Push image → 9. Deploy staging → 10. Smoke tests → 11. Deploy production

## 9. Technology Stack

| Category | Technology |
|----------|-----------|
| Framework | Spring Boot 4.x, Java 25+ |
| Cloud | Spring Cloud 2024.x |
| Namespace | Jakarta EE 11+ |
| Data | Spring Data JPA, Hibernate 7.x, Flyway |
| API Docs | SpringDoc OpenAPI 3 |
| Messaging | Apache Kafka, Spring Cloud Stream |
| Metrics | Micrometer, Prometheus, Grafana |
| Tracing | OpenTelemetry, Jaeger |
| Security | Spring Security 6.x, OAuth 2.0, JWT |
| Testing | JUnit 5, Mockito, AssertJ, JSONAssert, Testcontainers, WireMock, Awaitility, LocalStack |
| Quality | Checkstyle, JaCoCo (80%+), SonarQube, OWASP Dependency Check |
| Mapping | MapStruct |
| Logging | @Slf4j (Lombok) |
| DI | Constructor injection via @RequiredArgsConstructor |

## 10. Quick Reference — Decision Rules

| When you need to... | Do this |
|---------------------|---------|
| Create a DTO or Value Object | Use a Java Record with compact constructor validation |
| Add business logic | Put it in the Domain entity, not in a service |
| Access the database | Define an outgoing Port interface, implement in a Persistence Adapter |
| Expose an API | Define an incoming Port (Use Case), implement in Application service, call from Web Adapter |
| Test a controller | Use @WebMvcTest with @MockBean for use cases |
| Test a repository | Use @DataJpaTest + Testcontainers (real PostgreSQL) |
| Test domain logic | Plain JUnit 5, no Spring context |
| Test an external API | Use WireMock or @RestClientTest |
| Test JSON serialization | Use @JsonTest |
| Test async behavior | Use Awaitility |
| Inject a dependency | Constructor injection only (@RequiredArgsConstructor) |
| Handle nulls | Return Optional, never null |
| Write a loop | Use Stream API instead |
| Create a new entity | Constructor (no ID) |
| Load entity from DB | Static factory `withId(...)` method |
| Lock a shared resource | Acquire lock → mutate → persist → release lock (Buckpal pattern) |

### When to Take Shortcuts (from Buckpal)

Not every feature needs full hexagonal ceremony. Buckpal explicitly discusses "taking shortcuts consciously":

| Shortcut | When acceptable |
|----------|----------------|
| Skip incoming port interface | Simple CRUD with no business logic |
| Skip mapping between layers | When domain and persistence models are identical |
| Use @Service directly without port | Internal utility services with no external adapters |
| Use @SpringBootTest | Only for smoke tests or critical E2E paths |

The key is to take shortcuts **consciously** and document the trade-off, not accidentally.

## 11. Reference Repository Catalog

> The following GitHub repositories serve as state-of-the-art references for building production-grade Spring Boot 4.x / Java 25+ microservices. Each repository is selected for specific architectural patterns relevant to the court booking platform. When implementing, cross-reference the appropriate repository for the pattern you need.

### Tier 1: Primary Architecture References (Already Codified Above)

| Repository | Stars | Focus Area | What to Extract |
|-----------|-------|-----------|----------------|
| [thombergs/buckpal](https://github.com/thombergs/buckpal) | ~2.5k ⭐ | Hexagonal Architecture | Package structure, ports & adapters, use case pattern, self-validating commands, conscious shortcuts, dependency inversion |
| [spring-petclinic/spring-petclinic-microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) | ~1.5k ⭐ | Spring Cloud Microservices | Service discovery (Eureka), API Gateway, Config Server, Resilience4j circuit breakers, Micrometer tracing, OpenTelemetry, Docker Compose orchestration |
| [rieckpil/testing-spring-boot-applications-masterclass](https://github.com/rieckpil/testing-spring-boot-applications-masterclass) | ~500 ⭐ | Testing Standards | @WebMvcTest, @DataJpaTest, @JsonTest, @RestClientTest, Testcontainers with real PostgreSQL, WireMock for external APIs, Awaitility for async, test naming conventions |
| [abhisheksr01/spring-boot-microservice-best-practices](https://github.com/abhisheksr01/spring-boot-microservice-best-practices) | ~300 ⭐ | DevSecOps & CI/CD | Java 25+, Gradle build, Checkstyle, JaCoCo, Hadolint, OWASP dependency check, Docker image vulnerability scanning, Kubernetes deployment, MapStruct, WireMock, Cucumber E2E tests |

### Tier 2: Microservice Patterns & Event-Driven Architecture

| Repository | Stars | Focus Area | What to Extract |
|-----------|-------|-----------|----------------|
| [microservices-patterns/ftgo-application](https://github.com/microservices-patterns/ftgo-application) | ~3.7k ⭐ | Microservice Patterns (Chris Richardson) | Saga orchestration, event-driven communication, transactional outbox pattern, API gateway, service decomposition by business capability, contract testing between services |
| [eventuate-tram/eventuate-tram-sagas](https://github.com/eventuate-tram/eventuate-tram-sagas) | ~400 ⭐ | Saga Pattern | Orchestration-based sagas for distributed transactions, compensating transactions, JDBC-based transactional messaging — directly applicable to booking + payment atomicity |
| [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) | ~17k ⭐ | Cloud-Native Microservices on Kubernetes | 11 polyglot microservices, Kubernetes manifests, Istio service mesh integration, gRPC communication, Locust load testing, Terraform deployment, Kustomize variations |

### Tier 3: Kubernetes, Observability & Infrastructure

| Repository | Stars | Focus Area | What to Extract |
|-----------|-------|-----------|----------------|
| [piomin/sample-spring-microservices-kubernetes](https://github.com/piomin/sample-spring-microservices-kubernetes) | ~414 ⭐ | Spring Cloud Kubernetes | Spring Cloud Kubernetes config, OpenFeign for inter-service calls, Spring Cloud Gateway on K8s, health probes, readiness/liveness configuration — directly applicable to DOKS deployment |
| [spring-projects/spring-modulith](https://github.com/spring-projects/spring-modulith) | ~507 ⭐ | Modular Monolith / DDD Boundaries | Module boundary enforcement, event-driven module communication, architecture verification tests — useful for enforcing domain boundaries within each service |
| [ali-bouali/spring-boot-3-jwt-security](https://github.com/ali-bouali/spring-boot-3-jwt-security) | ~1.3k ⭐ | Spring Security 6 + JWT | Spring Boot 3 + Spring Security 6 JWT authentication, OAuth2 resource server configuration, role-based access control — directly applicable to Platform Service auth module |

### Tier 4: Domain-Specific Patterns (Booking, Payments, Real-Time)

| Repository | Focus Area | What to Extract |
|-----------|-----------|----------------|
| [macrozheng/mall](https://github.com/macrozheng/mall) ~78k ⭐ | E-Commerce Platform | Spring Boot + MyBatis full e-commerce system, order management, payment integration, Redis caching strategies, Elasticsearch, Docker deployment — reference for booking/payment flow patterns |
| [macrozheng/mall-swarm](https://github.com/macrozheng/mall-swarm) ~12k ⭐ | Microservices E-Commerce | Spring Cloud Alibaba microservices edition, Spring Boot 4.x, gateway routing, service discovery, distributed transactions — reference for splitting monolith into Platform + Transaction services |
| [dandoran/spring-data-postgis-geospatial](https://github.com/dandoran/spring-data-postgis-geospatial) | PostGIS + Spring Data | Hibernate Spatial with PostGIS, Flyway migrations for spatial schema, distance queries using ST_DWithin — directly applicable to court discovery geospatial queries |
| [dnjscksdn98/spring-redis-chat-service](https://github.com/dnjscksdn98/spring-redis-chat-service) | WebSocket + Redis Pub/Sub | Spring Boot WebSocket with STOMP, Redis Pub/Sub for horizontal scaling, real-time messaging — directly applicable to availability broadcasting and in-app notifications |

### How to Use These References

When implementing a specific feature, consult the appropriate reference:

| Feature Being Implemented | Primary Reference | Secondary Reference |
|--------------------------|------------------|---------------------|
| Package structure & layer separation | Buckpal | Spring Modulith |
| Inter-service communication | FTGO Application | PetClinic Microservices |
| Booking conflict prevention (saga) | FTGO Application | Eventuate Tram Sagas |
| JWT authentication & OAuth | ali-bouali/spring-boot-3-jwt-security | PetClinic Microservices |
| Kubernetes deployment & health probes | piomin/sample-spring-microservices-kubernetes | Google Online Boutique |
| Istio service mesh & observability | Google Online Boutique | PetClinic Microservices |
| PostGIS geospatial queries | dandoran/spring-data-postgis-geospatial | — |
| WebSocket + Redis Pub/Sub scaling | dnjscksdn98/spring-redis-chat-service | — |
| Payment processing (Stripe) | macrozheng/mall (payment module) | FTGO Application |
| Redis caching strategies | macrozheng/mall | PetClinic Microservices |
| Testing (unit, slice, integration) | rieckpil Masterclass | abhisheksr01 best-practices |
| Property-based testing (jqwik) | [jqwik.net](https://jqwik.net/) | — |
| CI/CD & DevSecOps pipeline | abhisheksr01 best-practices | Google Online Boutique |
| Docker Compose local dev | PetClinic Microservices | FTGO Application |
| Locust load/stress testing | Google Online Boutique | — |
| E2E contract testing | FTGO Application | abhisheksr01 (Cucumber) |

### Key Architectural Decisions Informed by References

**From Buckpal (Hexagonal Architecture):**
- One use case = one incoming port interface
- Self-validating command objects (records with compact constructors)
- Domain entities with creation constructor + `withId()` reconstitution factory
- Take shortcuts consciously (skip ports for simple CRUD, document the trade-off)

**From FTGO Application (Microservice Patterns):**
- Saga pattern for distributed transactions spanning booking + payment
- Transactional outbox for reliable event publishing to Kafka
- API composition for aggregating data across services
- Service decomposition by business capability (Platform vs Transaction)

**From PetClinic Microservices (Spring Cloud):**
- Spring Cloud Gateway for API routing (maps to NGINX Ingress path-based routing in K8s)
- Resilience4j for circuit breaking on external service calls (Stripe, OAuth, Weather API)
- Micrometer + OpenTelemetry for distributed tracing across services
- Docker Compose for local development infrastructure

**From Google Online Boutique (Cloud-Native):**
- Kubernetes-native deployment with health probes and readiness checks
- Istio service mesh for mTLS, traffic management, and canary deployments
- Locust for load testing with realistic user flows
- Terraform for infrastructure provisioning

**From rieckpil Masterclass (Testing):**
- Test pyramid: many unit tests, some slice tests, few integration tests
- Never use H2 for integration tests — always Testcontainers with real PostgreSQL
- @WebMvcTest with @MockBean for controller tests
- @DataJpaTest + Testcontainers for repository tests
- WireMock for external API mocking (Stripe, OAuth, Weather)
- Awaitility for async event assertions (Kafka consumers, WebSocket)

**From jqwik (Property-Based Testing):**
- Use `@Property` annotation with custom `@Provide` generators
- Minimum 100 iterations per property test
- Smart generators that constrain to valid input space
- Tag format: `Feature: court-booking-platform, Property {N}: {description}`

## 12. DigitalOcean-Specific Deployment Patterns

> Since the court booking platform deploys to DigitalOcean (DOKS, Managed PostgreSQL, Managed Redis, Spaces), adapt cloud-specific patterns from the reference repositories accordingly.

### Mapping AWS/GCP Patterns to DigitalOcean

| Pattern from Reference | AWS/GCP Implementation | DigitalOcean Equivalent |
|----------------------|----------------------|------------------------|
| Service Discovery | Eureka (PetClinic) | Kubernetes DNS + NGINX Ingress |
| Config Management | Spring Cloud Config (PetClinic) | Kubernetes ConfigMaps + Sealed Secrets |
| API Gateway | Spring Cloud Gateway | NGINX Ingress Controller with path-based routing |
| Object Storage | S3 (AWS) | DigitalOcean Spaces (S3-compatible API) |
| Container Registry | ECR / GCR | DigitalOcean Container Registry |
| Managed Database | RDS / Cloud SQL | DigitalOcean Managed PostgreSQL + PostGIS |
| Managed Cache | ElastiCache / Memorystore | DigitalOcean Managed Redis |
| Event Streaming | Amazon MSK / Pub/Sub | Redpanda Serverless (Kafka-compatible, native protocol) |
| Infrastructure as Code | CloudFormation / Deployment Manager | Terraform with DigitalOcean provider |
| CI/CD | CodePipeline / Cloud Build | GitHub Actions |
| Secrets Management | AWS Secrets Manager / GCP Secret Manager | External Secrets Operator → Sealed Secrets (MVP) |
| Service Mesh | App Mesh / Anthos | Istio on DOKS (staging + production only) |
| Monitoring | CloudWatch / Cloud Monitoring | Prometheus + Grafana (self-hosted on DOKS) |
| Log Aggregation | CloudWatch Logs / Cloud Logging | Loki + Grafana (self-hosted on DOKS) |
| Distributed Tracing | X-Ray / Cloud Trace | Jaeger with OpenTelemetry (self-hosted on DOKS) |

### Spring Profile Configuration for DigitalOcean Environments

```yaml
# application-local.yml — Docker Compose infrastructure
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/courtbooking
  redis:
    host: localhost
  kafka:
    bootstrap-servers: localhost:9092

# application-dev.yml — DigitalOcean dev namespace
spring:
  datasource:
    url: jdbc:postgresql://${DO_PG_HOST}:${DO_PG_PORT}/courtbooking_dev
  redis:
    host: ${DO_REDIS_HOST}
  kafka:
    bootstrap-servers: ${REDPANDA_BOOTSTRAP_SERVERS}
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256

# application-prod.yml — DigitalOcean production cluster
spring:
  datasource:
    url: jdbc:postgresql://${DO_PG_HOST}:${DO_PG_PORT}/courtbooking_prod
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  redis:
    host: ${DO_REDIS_HOST}
  kafka:
    bootstrap-servers: ${REDPANDA_BOOTSTRAP_SERVERS}
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
```

### Kubernetes Health Probes (from piomin reference)

```yaml
# Kubernetes deployment manifest for Spring Boot service
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60
  failureThreshold: 30
  periodSeconds: 10
```

## 13. Additional Implementation Patterns

### Transactional Outbox Pattern (Reliable Event Publishing)

For reliable event publishing to Kafka, use the transactional outbox pattern. This ensures events are published exactly once, even if the application crashes after database commit but before Kafka acknowledgment.

**Outbox Table Schema:**

```sql
-- Flyway migration: V010__create_outbox_table.sql
CREATE TABLE transaction.outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,  -- e.g., 'Booking', 'Payment'
    aggregate_id    VARCHAR(100) NOT NULL,  -- e.g., booking ID
    event_type      VARCHAR(100) NOT NULL,  -- e.g., 'BOOKING_CREATED'
    topic           VARCHAR(100) NOT NULL,  -- Kafka topic name
    partition_key   VARCHAR(100) NOT NULL,  -- Kafka partition key
    payload         JSONB NOT NULL,         -- Event payload
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at    TIMESTAMPTZ,            -- NULL until published
    retry_count     INT NOT NULL DEFAULT 0,
    last_error      TEXT
);

CREATE INDEX idx_outbox_unpublished ON transaction.outbox_events (created_at) 
    WHERE published_at IS NULL;
```

**Outbox Entity:**

```java
@Entity
@Table(name = "outbox_events", schema = "transaction")
@Getter
@NoArgsConstructor
public class OutboxEvent {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;
    
    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;
    
    @Column(name = "event_type", nullable = false)
    private String eventType;
    
    @Column(nullable = false)
    private String topic;
    
    @Column(name = "partition_key", nullable = false)
    private String partitionKey;
    
    @Column(columnDefinition = "jsonb", nullable = false)
    private String payload;
    
    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
    
    @Column(name = "published_at")
    private Instant publishedAt;
    
    @Column(name = "retry_count", nullable = false)
    private int retryCount;
    
    @Column(name = "last_error")
    private String lastError;
    
    public static OutboxEvent create(String aggregateType, String aggregateId, 
                                      String eventType, String topic, 
                                      String partitionKey, Object payload) {
        OutboxEvent event = new OutboxEvent();
        event.aggregateType = aggregateType;
        event.aggregateId = aggregateId;
        event.eventType = eventType;
        event.topic = topic;
        event.partitionKey = partitionKey;
        event.payload = JsonUtils.toJson(payload);
        event.createdAt = Instant.now();
        event.retryCount = 0;
        return event;
    }
    
    public void markPublished() {
        this.publishedAt = Instant.now();
    }
    
    public void recordFailure(String error) {
        this.retryCount++;
        this.lastError = error;
    }
}
```

**Use Case with Outbox:**

```java
@Service
@Transactional
@RequiredArgsConstructor
public class CreateBookingService implements CreateBookingUseCase {

    private final SaveBookingPort saveBookingPort;
    private final OutboxRepository outboxRepository;

    @Override
    public BookingId createBooking(CreateBookingCommand command) {
        // ... validation and booking creation ...
        
        BookingId bookingId = saveBookingPort.save(booking);
        
        // Write event to outbox in same transaction
        BookingCreatedEvent event = new BookingCreatedEvent(
            bookingId.value(),
            command.courtId().value(),
            command.userId().value(),
            command.dateRange().start(),
            command.dateRange().end()
        );
        
        OutboxEvent outboxEvent = OutboxEvent.create(
            "Booking",
            bookingId.value().toString(),
            "BOOKING_CREATED",
            "booking-events",
            command.courtId().value().toString(),  // partition by courtId
            event
        );
        outboxRepository.save(outboxEvent);
        
        return bookingId;
    }
}
```

**Outbox Publisher (Scheduled Job):**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    private static final int BATCH_SIZE = 100;
    private static final int MAX_RETRIES = 5;

    @Scheduled(fixedDelay = 100)  // Poll every 100ms
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository
            .findUnpublishedOrderByCreatedAt(BATCH_SIZE);
        
        for (OutboxEvent event : pending) {
            if (event.getRetryCount() >= MAX_RETRIES) {
                log.error("Event {} exceeded max retries, moving to DLQ", event.getId());
                // Move to dead letter queue or alert
                continue;
            }
            
            try {
                kafkaTemplate.send(
                    event.getTopic(),
                    event.getPartitionKey(),
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS);  // Synchronous send
                
                event.markPublished();
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.warn("Failed to publish event {}: {}", event.getId(), e.getMessage());
                event.recordFailure(e.getMessage());
                outboxRepository.save(event);
            }
        }
    }
}
```

### Cross-Schema View Access

Transaction Service reads Platform Service data via database views. These are read-only and should never be used for writes.

**View Definitions (Platform Schema):**

```sql
-- Flyway migration in Platform Service: V020__create_cross_schema_views.sql

-- Grant Transaction Service user read access
GRANT USAGE ON SCHEMA platform TO transaction_service_user;

-- Court summary view
CREATE VIEW platform.v_court_summary AS
SELECT 
    id,
    owner_id,
    name,
    court_type,
    slot_duration_minutes,
    capacity,
    base_price_cents,
    timezone,
    confirmation_mode,
    is_visible,
    version
FROM platform.courts
WHERE deleted_at IS NULL;

GRANT SELECT ON platform.v_court_summary TO transaction_service_user;

-- User basic info view
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
WHERE deleted_at IS NULL;

GRANT SELECT ON platform.v_user_basic TO transaction_service_user;

-- Cancellation policy tiers view
CREATE VIEW platform.v_court_cancellation_tiers AS
SELECT 
    court_id,
    threshold_hours,
    refund_percent,
    sort_order
FROM platform.cancellation_tiers;

GRANT SELECT ON platform.v_court_cancellation_tiers TO transaction_service_user;
```

**JPA Entity for Cross-Schema View (Read-Only):**

```java
@Entity
@Table(name = "v_court_summary", schema = "platform")
@Immutable  // Hibernate: marks as read-only
@Getter
@NoArgsConstructor
public class CourtSummaryView {

    @Id
    private Long id;
    
    @Column(name = "owner_id")
    private Long ownerId;
    
    private String name;
    
    @Column(name = "court_type")
    @Enumerated(EnumType.STRING)
    private CourtType courtType;
    
    @Column(name = "slot_duration_minutes")
    private Integer slotDurationMinutes;
    
    private Integer capacity;
    
    @Column(name = "base_price_cents")
    private Integer basePriceCents;
    
    private String timezone;
    
    @Column(name = "confirmation_mode")
    @Enumerated(EnumType.STRING)
    private ConfirmationMode confirmationMode;
    
    @Column(name = "is_visible")
    private Boolean isVisible;
    
    private Long version;
}
```

**Repository for Cross-Schema View:**

```java
@Repository
public interface CourtSummaryViewRepository extends Repository<CourtSummaryView, Long> {
    
    Optional<CourtSummaryView> findById(Long id);
    
    List<CourtSummaryView> findByOwnerIdAndIsVisibleTrue(Long ownerId);
    
    // No save/delete methods — read-only!
}
```

**Usage in Use Case:**

```java
@Service
@RequiredArgsConstructor
public class CreateBookingService implements CreateBookingUseCase {

    private final CourtSummaryViewRepository courtSummaryView;
    
    @Override
    public BookingId createBooking(CreateBookingCommand command) {
        // Read court info from cross-schema view
        CourtSummaryView court = courtSummaryView.findById(command.courtId().value())
            .orElseThrow(() -> new CourtNotFoundException(command.courtId()));
        
        if (!court.getIsVisible()) {
            throw new CourtNotVisibleException(command.courtId());
        }
        
        // Use court.getBasePriceCents(), court.getTimezone(), etc.
        // ...
    }
}
```

### Internal API Authentication

For service-to-service calls on `/internal/*` endpoints, use API key authentication in dev/test and mTLS in staging/prod.

**Security Configuration:**

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final InternalApiKeyFilter internalApiKeyFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                // Actuator health endpoints
                .requestMatchers("/actuator/health/**").permitAll()
                
                // Internal API — authenticated by filter (API key or mTLS)
                .requestMatchers("/internal/**").authenticated()
                
                // Public API — JWT authentication
                .requestMatchers("/api/v1/**").authenticated()
                
                .anyRequest().denyAll()
            )
            .addFilterBefore(internalApiKeyFilter, UsernamePasswordAuthenticationFilter.class)
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

**Internal API Key Filter:**

```java
@Component
@Slf4j
public class InternalApiKeyFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-Internal-Api-Key";
    
    @Value("${internal.api.key:}")
    private String expectedApiKey;
    
    @Value("${internal.api.mtls-enabled:false}")
    private boolean mtlsEnabled;

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain filterChain) throws ServletException, IOException {
        
        // Only apply to /internal/* paths
        if (!request.getRequestURI().startsWith("/internal/")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        // In staging/prod with Istio, mTLS handles auth — skip API key check
        if (mtlsEnabled) {
            // Istio validates client certificate; if request reaches here, it's trusted
            setInternalServiceAuthentication();
            filterChain.doFilter(request, response);
            return;
        }
        
        // Dev/test: validate API key
        String providedKey = request.getHeader(API_KEY_HEADER);
        
        if (expectedApiKey.isBlank()) {
            log.error("INTERNAL_API_KEY not configured");
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            return;
        }
        
        if (!expectedApiKey.equals(providedKey)) {
            log.warn("Invalid internal API key from {}", request.getRemoteAddr());
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid API key");
            return;
        }
        
        setInternalServiceAuthentication();
        filterChain.doFilter(request, response);
    }
    
    private void setInternalServiceAuthentication() {
        Authentication auth = new PreAuthenticatedAuthenticationToken(
            "internal-service", null, List.of(new SimpleGrantedAuthority("ROLE_INTERNAL_SERVICE"))
        );
        SecurityContextHolder.getContext().setAuthentication(auth);
    }
}
```

**Internal API Client (Transaction Service → Platform Service):**

```java
@Component
@RequiredArgsConstructor
public class PlatformServiceClient {

    private final RestClient restClient;
    
    @Value("${platform-service.base-url}")
    private String baseUrl;
    
    @Value("${internal.api.key}")
    private String apiKey;

    public SlotValidationResult validateSlot(Long courtId, LocalDate date, 
                                              LocalTime startTime, LocalTime endTime) {
        return restClient.get()
            .uri(baseUrl + "/internal/courts/{courtId}/validate-slot", courtId)
            .header("X-Internal-Api-Key", apiKey)
            .retrieve()
            .body(SlotValidationResult.class);
    }
    
    public PriceCalculationResult calculatePrice(Long courtId, LocalDate date,
                                                  LocalTime startTime, LocalTime endTime,
                                                  String promoCode) {
        return restClient.get()
            .uri(baseUrl + "/internal/courts/{courtId}/calculate-price?date={date}&startTime={start}&endTime={end}&promoCode={promo}",
                 courtId, date, startTime, endTime, promoCode)
            .header("X-Internal-Api-Key", apiKey)
            .retrieve()
            .body(PriceCalculationResult.class);
    }
}
```

**Application Properties:**

```yaml
# application-local.yml
internal:
  api:
    key: ${INTERNAL_API_KEY:dev-secret-key-change-in-prod}
    mtls-enabled: false

# application-staging.yml / application-prod.yml
internal:
  api:
    key: ""  # Not used with mTLS
    mtls-enabled: true
```

### WebSocket + Redis Pub/Sub

For real-time availability updates and notifications across multiple pods, use WebSocket with STOMP and Redis Pub/Sub for horizontal scaling.

**WebSocket Configuration:**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Use Redis-backed broker for horizontal scaling
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();
    }
}
```

**Redis Pub/Sub Configuration:**

```java
@Configuration
@RequiredArgsConstructor
public class RedisPubSubConfig {

    private final RedisConnectionFactory connectionFactory;
    private final AvailabilityUpdateSubscriber availabilitySubscriber;
    private final NotificationSubscriber notificationSubscriber;

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer() {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        
        // Subscribe to availability updates channel
        container.addMessageListener(availabilitySubscriber, 
            new ChannelTopic("availability-updates"));
        
        // Subscribe to user notifications channel (pattern for user-specific)
        container.addMessageListener(notificationSubscriber, 
            new PatternTopic("notifications:user:*"));
        
        return container;
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

**Availability Update Publisher:**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AvailabilityBroadcaster {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public void broadcastAvailabilityUpdate(Long courtId, LocalDate date, 
                                             List<TimeSlot> availableSlots) {
        try {
            AvailabilityUpdate update = new AvailabilityUpdate(
                courtId, date, availableSlots, Instant.now()
            );
            String message = objectMapper.writeValueAsString(update);
            
            // Publish to Redis — all pods will receive this
            redisTemplate.convertAndSend("availability-updates", message);
            
            log.debug("Broadcast availability update for court {} on {}", courtId, date);
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize availability update", e);
        }
    }
}
```

**Availability Update Subscriber (Receives from Redis, Sends to WebSocket):**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AvailabilityUpdateSubscriber implements MessageListener {

    private final SimpMessagingTemplate messagingTemplate;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            String json = new String(message.getBody(), StandardCharsets.UTF_8);
            AvailabilityUpdate update = objectMapper.readValue(json, AvailabilityUpdate.class);
            
            // Send to all WebSocket subscribers watching this court
            String destination = "/topic/courts/" + update.courtId() + "/availability";
            messagingTemplate.convertAndSend(destination, update);
            
            log.debug("Forwarded availability update to WebSocket: {}", destination);
        } catch (Exception e) {
            log.error("Failed to process availability update from Redis", e);
        }
    }
}
```

**User-Specific Notification Publisher:**

```java
@Component
@RequiredArgsConstructor
public class NotificationBroadcaster {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public void sendToUser(Long userId, NotificationPayload notification) {
        try {
            String message = objectMapper.writeValueAsString(notification);
            // Publish to user-specific channel
            redisTemplate.convertAndSend("notifications:user:" + userId, message);
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize notification", e);
        }
    }
}
```

**WebSocket Controller:**

```java
@Controller
@RequiredArgsConstructor
@Slf4j
public class AvailabilityWebSocketController {

    private final AvailabilityService availabilityService;

    @MessageMapping("/courts/{courtId}/subscribe")
    @SendTo("/topic/courts/{courtId}/availability")
    public AvailabilityUpdate subscribeToCourtAvailability(
            @DestinationVariable Long courtId,
            @Payload SubscriptionRequest request) {
        
        log.info("Client subscribed to court {} availability for date {}", 
                 courtId, request.date());
        
        // Return current availability immediately
        return availabilityService.getCurrentAvailability(courtId, request.date());
    }
}
```

### Flyway Migration Conventions

**Naming Convention:**

```
V{version}__{description}.sql

Examples:
V001__create_users_table.sql
V002__create_courts_table.sql
V003__add_postgis_extension.sql
V004__create_bookings_table.sql
V005__add_booking_indexes.sql
V010__create_outbox_table.sql
V020__create_cross_schema_views.sql

Repeatable migrations (run on every change):
R__create_views.sql
R__update_functions.sql
```

**Example Migration with PostGIS:**

```sql
-- V003__add_postgis_extension.sql
-- Enable PostGIS for geospatial queries

CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- Add location column to courts table
ALTER TABLE platform.courts 
ADD COLUMN location GEOGRAPHY(POINT, 4326);

-- Create spatial index for distance queries
CREATE INDEX idx_courts_location ON platform.courts USING GIST (location);

-- Example: Find courts within 10km of a point
-- SELECT * FROM platform.courts 
-- WHERE ST_DWithin(location, ST_MakePoint(23.7275, 37.9838)::geography, 10000);
```

**Example Migration for Bookings:**

```sql
-- V004__create_bookings_table.sql

CREATE TABLE transaction.bookings (
    id                      BIGSERIAL PRIMARY KEY,
    court_id                BIGINT NOT NULL,
    user_id                 BIGINT NOT NULL,
    date                    DATE NOT NULL,
    start_time              TIME NOT NULL,
    end_time                TIME NOT NULL,
    status                  VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    payment_status          VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    total_amount_cents      INTEGER NOT NULL,
    platform_fee_cents      INTEGER NOT NULL,
    court_owner_net_cents   INTEGER NOT NULL,
    currency                VARCHAR(3) NOT NULL DEFAULT 'EUR',
    stripe_payment_intent_id VARCHAR(100),
    idempotency_key         UUID UNIQUE,
    recurring_group_id      UUID,
    promo_code_id           BIGINT,
    notes                   TEXT,
    version                 BIGINT NOT NULL DEFAULT 0,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT chk_booking_times CHECK (start_time < end_time),
    CONSTRAINT chk_booking_amounts CHECK (
        total_amount_cents >= 0 AND 
        platform_fee_cents >= 0 AND 
        court_owner_net_cents >= 0
    )
);

-- Prevent double-bookings (only for active bookings)
CREATE UNIQUE INDEX uq_bookings_slot_active 
ON transaction.bookings (court_id, date, start_time, end_time) 
WHERE status NOT IN ('CANCELLED', 'REJECTED');

-- Query indexes
CREATE INDEX idx_bookings_court_date ON transaction.bookings (court_id, date);
CREATE INDEX idx_bookings_user ON transaction.bookings (user_id);
CREATE INDEX idx_bookings_status ON transaction.bookings (status) WHERE status NOT IN ('CANCELLED', 'COMPLETED');
CREATE INDEX idx_bookings_recurring ON transaction.bookings (recurring_group_id) WHERE recurring_group_id IS NOT NULL;
CREATE INDEX idx_bookings_payment_intent ON transaction.bookings (stripe_payment_intent_id) WHERE stripe_payment_intent_id IS NOT NULL;

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_bookings_updated_at
    BEFORE UPDATE ON transaction.bookings
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### MapStruct Configuration

**Gradle Dependency:**

```groovy
dependencies {
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
}
```

**Mapper for Domain ↔ JPA Entity with Value Objects:**

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface BookingPersistenceMapper {

    @Mapping(target = "id", source = "id", qualifiedByName = "bookingIdToLong")
    @Mapping(target = "courtId", source = "courtId", qualifiedByName = "courtIdToLong")
    @Mapping(target = "userId", source = "userId", qualifiedByName = "userIdToLong")
    @Mapping(target = "startTime", source = "dateRange.start", qualifiedByName = "instantToLocalTime")
    @Mapping(target = "endTime", source = "dateRange.end", qualifiedByName = "instantToLocalTime")
    @Mapping(target = "date", source = "dateRange.start", qualifiedByName = "instantToLocalDate")
    BookingJpaEntity toEntity(Booking domain);

    @Mapping(target = "id", source = "id", qualifiedByName = "longToBookingId")
    @Mapping(target = "courtId", source = "courtId", qualifiedByName = "longToCourtId")
    @Mapping(target = "userId", source = "userId", qualifiedByName = "longToUserId")
    @Mapping(target = "dateRange", source = ".", qualifiedByName = "toDateRange")
    Booking toDomain(BookingJpaEntity entity);

    // Value Object converters
    @Named("bookingIdToLong")
    default Long bookingIdToLong(BookingId id) {
        return id != null ? id.value() : null;
    }

    @Named("longToBookingId")
    default BookingId longToBookingId(Long id) {
        return id != null ? new BookingId(id) : null;
    }

    @Named("courtIdToLong")
    default Long courtIdToLong(CourtId id) {
        return id != null ? id.value() : null;
    }

    @Named("longToCourtId")
    default CourtId longToCourtId(Long id) {
        return id != null ? new CourtId(id) : null;
    }

    @Named("userIdToLong")
    default Long userIdToLong(UserId id) {
        return id != null ? id.value() : null;
    }

    @Named("longToUserId")
    default UserId longToUserId(Long id) {
        return id != null ? new UserId(id) : null;
    }

    @Named("toDateRange")
    default DateRange toDateRange(BookingJpaEntity entity) {
        LocalDateTime start = LocalDateTime.of(entity.getDate(), entity.getStartTime());
        LocalDateTime end = LocalDateTime.of(entity.getDate(), entity.getEndTime());
        return new DateRange(start, end);
    }

    @Named("instantToLocalTime")
    default LocalTime instantToLocalTime(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.toLocalTime() : null;
    }

    @Named("instantToLocalDate")
    default LocalDate instantToLocalDate(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.toLocalDate() : null;
    }
}
```

**Mapper for Web DTOs:**

```java
@Mapper(componentModel = "spring")
public interface BookingWebMapper {

    @Mapping(target = "id", source = "id.value")
    @Mapping(target = "courtId", source = "courtId.value")
    @Mapping(target = "userId", source = "userId.value")
    @Mapping(target = "startTime", source = "dateRange.start")
    @Mapping(target = "endTime", source = "dateRange.end")
    BookingResponse toResponse(Booking domain);

    List<BookingResponse> toResponseList(List<Booking> domains);
}
```

**Web DTO (Record):**

```java
public record BookingResponse(
    Long id,
    Long courtId,
    Long userId,
    LocalDateTime startTime,
    LocalDateTime endTime,
    String status,
    String paymentStatus,
    Integer totalAmountCents,
    String currency
) {}
```

## References

- [Buckpal — thombergs](https://github.com/thombergs/buckpal) — Hexagonal Architecture reference (~2.5k ⭐, 722 forks)
- [Hexagonal Architecture with Java and Spring — reflectoring.io](https://reflectoring.io/spring-hexagonal/) — Buckpal companion article
- [Testing Spring Boot Applications Masterclass — rieckpil](https://github.com/rieckpil/testing-spring-boot-applications-masterclass) — Testing standards
- [Spring Boot Test Slices — rieckpil](https://rieckpil.de/spring-boot-test-slices-overview-and-usage/) — Complete slice test guide
- [Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) — Spring Cloud patterns
- [Spring Boot Microservice Best Practices — abhisheksr01](https://github.com/abhisheksr01/spring-boot-microservice-best-practices) — DevSecOps, Java 25+
- [FTGO Application — microservices-patterns](https://github.com/microservices-patterns/ftgo-application) — Saga, event-driven patterns (~3.7k ⭐, 1.4k forks)
- [Eventuate Tram Sagas](https://github.com/eventuate-tram/eventuate-tram-sagas) — Distributed transaction sagas
- [Google Online Boutique — microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) — Cloud-native K8s reference (~17k ⭐)
- [Spring Microservices on Kubernetes — piomin](https://github.com/piomin/sample-spring-microservices-kubernetes) — Spring Cloud Kubernetes (~414 ⭐)
- [Spring Modulith](https://github.com/spring-projects/spring-modulith) — Modular architecture with Spring Boot (~507 ⭐)
- [Spring Boot 3 JWT Security — ali-bouali](https://github.com/ali-bouali/spring-boot-3-jwt-security) — Spring Security 6 + JWT (~1.3k ⭐)
- [Mall E-Commerce — macrozheng](https://github.com/macrozheng/mall) — Full e-commerce platform (~78k ⭐)
- [Mall Swarm Microservices — macrozheng](https://github.com/macrozheng/mall-swarm) — Microservices e-commerce (~12k ⭐)
- [Spring Data PostGIS Geospatial — dandoran](https://github.com/dandoran/spring-data-postgis-geospatial) — PostGIS + Hibernate Spatial + Flyway
- [Spring Redis Chat Service — dnjscksdn98](https://github.com/dnjscksdn98/spring-redis-chat-service) — WebSocket + Redis Pub/Sub
- [jqwik — Property-Based Testing for Java](https://jqwik.net/) — PBT framework for JUnit 5
- [Testcontainers](https://testcontainers.com/) — Real database testing
- [Microservices.io — Chris Richardson](https://microservices.io/) — Microservice patterns catalog
- [OpenTelemetry](https://opentelemetry.io/) — Observability framework

---
*Content rephrased for compliance with licensing restrictions. Original sources cited above.*
