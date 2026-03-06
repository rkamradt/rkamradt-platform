# Java Coding Conventions

## Overview

This document defines coding standards for Java-based services in the rkamradt-platform. These conventions prioritize readability, maintainability, and consistency across the polyrepo architecture.

## Language Version

- **Target**: Java 17 LTS (minimum)
- **Future**: Java 21 LTS when dependencies support it
- Use modern Java features: records, pattern matching, text blocks, etc.

## Framework Standards

### Spring Boot

- **Version**: 3.x (Spring Framework 6.x)
- **Packaging**: Jar (not War)
- **Dependency Injection**: Constructor injection (preferred), avoid field injection
- **Configuration**: `application.yml` over `application.properties`

### Project Structure

```
src/
├── main/
│   ├── java/
│   │   └── net/kamradtfamily/{service-name}/
│   │       ├── {ServiceName}Application.java
│   │       ├── config/              # Configuration classes
│   │       ├── controller/          # REST controllers
│   │       ├── service/             # Business logic
│   │       ├── repository/          # Data access
│   │       ├── domain/              # Domain models, entities
│   │       ├── event/               # Event models and handlers
│   │       │   ├── producer/        # Kafka producers
│   │       │   └── consumer/        # Kafka consumers
│   │       ├── dto/                 # Data Transfer Objects
│   │       ├── exception/           # Custom exceptions
│   │       └── util/                # Utility classes
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       └── db/migration/            # Flyway/Liquibase scripts
└── test/
    ├── java/
    │   └── net/kamradtfamily/{service-name}/
    │       ├── integration/         # Integration tests
    │       ├── unit/                # Unit tests
    │       └── testcontainers/      # Testcontainers tests
    └── resources/
        └── application-test.yml
```

## Naming Conventions

### Packages

- **Base package**: `net.kamradtfamily.{service-name}`
- **Examples**:
  - `net.kamradtfamily.manufacturing`
  - `net.kamradtfamily.vehicleevent`
- Use lowercase, singular nouns for package names
- Avoid abbreviations unless widely understood

### Classes

- **Controllers**: `{Entity}Controller` (e.g., `BomController`)
- **Services**: `{Entity}Service` (e.g., `BomService`)
- **Repositories**: `{Entity}Repository` (e.g., `BomRepository`)
- **Entities**: Noun representing domain concept (e.g., `Bom`, `PartsOrder`)
- **DTOs**: `{Entity}Request`, `{Entity}Response` (e.g., `CreateBomRequest`)
- **Events**: `{Entity}{Action}Event` (e.g., `BomCreatedEvent`)
- **Event Handlers**: `{Event}Handler` (e.g., `BomCreatedEventHandler`)
- **Kafka Producers**: `{Domain}EventProducer` (e.g., `ManufacturingEventProducer`)
- **Kafka Consumers**: `{Topic}Consumer` (e.g., `InventoryEventConsumer`)

### Methods

- Use verbs or verb phrases: `createBom()`, `findById()`, `publishEvent()`
- Boolean methods: `isActive()`, `hasPermission()`, `canProcess()`
- REST handler methods: `create()`, `findById()`, `update()`, `delete()`

### Constants

- ALL_CAPS_WITH_UNDERSCORES
- Group related constants in dedicated classes or enums

```java
public class KafkaTopics {
    public static final String BOM_CREATED = "manufacturing.bom.created";
    public static final String PARTS_ORDERED = "manufacturing.parts.ordered";
}
```

## Code Style

### Formatting

- **Indentation**: 4 spaces (no tabs)
- **Line length**: 120 characters maximum
- **Braces**: K&R style (opening brace on same line)

```java
public class ExampleService {
    public void exampleMethod() {
        if (condition) {
            // code
        } else {
            // code
        }
    }
}
```

### Import Organization

1. Java standard library imports
2. Third-party library imports
3. Spring Framework imports
4. Internal project imports

Use wildcard imports only for static imports of test assertions.

## Constructor Injection

Always prefer constructor injection over field injection:

**Good**:
```java
@Service
public class BomService {
    private final BomRepository bomRepository;
    private final EventProducer eventProducer;

    public BomService(BomRepository bomRepository, EventProducer eventProducer) {
        this.bomRepository = bomRepository;
        this.eventProducer = eventProducer;
    }
}
```

**Avoid**:
```java
@Service
public class BomService {
    @Autowired
    private BomRepository bomRepository;  // Field injection - avoid
}
```

## Domain Modeling

### Entities

Use JPA entities for database persistence:

```java
@Entity
@Table(name = "bom")
public class Bom {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String productId;

    @Column(nullable = false)
    private String version;

    @Enumerated(EnumType.STRING)
    private BomStatus status;

    @OneToMany(mappedBy = "bom", cascade = CascadeType.ALL)
    private List<BomPart> parts = new ArrayList<>();

    // Getters, setters, equals, hashCode
}
```

### Records for DTOs

Use Java records for immutable DTOs:

```java
public record CreateBomRequest(
    String productId,
    String productName,
    String version,
    List<PartItem> parts
) {
    // Compact constructor for validation
    public CreateBomRequest {
        Objects.requireNonNull(productId, "productId must not be null");
        Objects.requireNonNull(parts, "parts must not be null");
        if (parts.isEmpty()) {
            throw new IllegalArgumentException("parts must not be empty");
        }
    }
}
```

### Value Objects

Use records for value objects:

```java
public record Money(BigDecimal amount, String currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}
```

## REST API Design

### Controller Structure

```java
@RestController
@RequestMapping("/api/bom")
public class BomController {
    private final BomService bomService;

    public BomController(BomService bomService) {
        this.bomService = bomService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public BomResponse create(@Valid @RequestBody CreateBomRequest request) {
        return bomService.createBom(request);
    }

    @GetMapping("/{id}")
    public BomResponse findById(@PathVariable UUID id) {
        return bomService.findById(id);
    }
}
```

### Response Entities

Use `ResponseEntity` for complex responses:

```java
@GetMapping("/{id}")
public ResponseEntity<BomResponse> findById(@PathVariable UUID id) {
    return bomService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

## Exception Handling

### Custom Exceptions

```java
public class BomNotFoundException extends RuntimeException {
    public BomNotFoundException(UUID id) {
        super("BOM not found with id: " + id);
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BomNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleBomNotFound(BomNotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationError(MethodArgumentNotValidException ex) {
        // Extract validation errors
        return new ErrorResponse("Validation failed", extractErrors(ex));
    }
}
```

## Validation

Use Bean Validation (JSR-380):

```java
public record CreateBomRequest(
    @NotBlank(message = "productId is required")
    String productId,

    @NotBlank(message = "productName is required")
    String productName,

    @NotEmpty(message = "parts list must not be empty")
    @Valid
    List<PartItem> parts
) {}
```

## Logging

### Use SLF4J with Logback

```java
@Service
public class BomService {
    private static final Logger log = LoggerFactory.getLogger(BomService.class);

    public BomResponse createBom(CreateBomRequest request) {
        log.info("Creating BOM for product: {}", request.productId());

        try {
            // business logic
            log.debug("BOM created successfully: {}", bom.getId());
            return response;
        } catch (Exception ex) {
            log.error("Failed to create BOM for product: {}", request.productId(), ex);
            throw ex;
        }
    }
}
```

### Log Levels

- **ERROR**: System failures, exceptions requiring immediate attention
- **WARN**: Recoverable issues, deprecated usage
- **INFO**: Important business events (BOM created, order placed)
- **DEBUG**: Detailed diagnostic information
- **TRACE**: Very detailed diagnostic (usually disabled)

## Testing

### Unit Tests

```java
@ExtendWith(MockitoExtension.class)
class BomServiceTest {

    @Mock
    private BomRepository bomRepository;

    @Mock
    private EventProducer eventProducer;

    @InjectMocks
    private BomService bomService;

    @Test
    void shouldCreateBomSuccessfully() {
        // Given
        CreateBomRequest request = new CreateBomRequest(...);
        Bom savedBom = new Bom(...);
        when(bomRepository.save(any(Bom.class))).thenReturn(savedBom);

        // When
        BomResponse response = bomService.createBom(request);

        // Then
        assertThat(response.id()).isNotNull();
        verify(eventProducer).publish(any(BomCreatedEvent.class));
    }
}
```

### Integration Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
class BomControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldCreateBomViaApi() throws Exception {
        CreateBomRequest request = new CreateBomRequest(...);

        mockMvc.perform(post("/api/bom")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists());
    }
}
```

### Testcontainers

```java
@SpringBootTest
@Testcontainers
class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:latest")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void shouldPublishEventToKafka() {
        // Test implementation
    }
}
```

## Configuration

### Application Properties

Use YAML for better readability:

```yaml
spring:
  application:
    name: manufacturing-service

  datasource:
    url: jdbc:postgresql://localhost:5432/manufacturing
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: manufacturing-service
      auto-offset-reset: earliest

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## Dependencies

### Build Tool: Maven or Gradle

**Maven** (pom.xml):
```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.2.0</spring-boot.version>
</properties>
```

**Gradle** (build.gradle):
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

java {
    sourceCompatibility = '17'
}
```

### Essential Dependencies

- Spring Boot Starter Web
- Spring Boot Starter Data JPA
- Spring Kafka
- PostgreSQL Driver
- Lombok (optional, use judiciously)
- Spring Boot Starter Validation
- Spring Boot Starter Actuator
- Micrometer (metrics)

## Documentation

### JavaDoc

Document public APIs:

```java
/**
 * Creates a new Bill of Materials (BOM) for a product.
 *
 * @param request the BOM creation request containing product details and parts
 * @return the created BOM with generated ID
 * @throws IllegalArgumentException if request is invalid
 */
public BomResponse createBom(CreateBomRequest request) {
    // implementation
}
```

### OpenAPI/Swagger

Add Springdoc OpenAPI:

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Manufacturing Service API")
                .version("1.0.0")
                .description("API for managing manufacturing workflows"));
    }
}
```

## Security

### Input Validation

- Always validate user input
- Use Bean Validation annotations
- Sanitize inputs for SQL/XSS vulnerabilities

### Sensitive Data

- Never log passwords, tokens, or sensitive data
- Use environment variables for secrets
- Consider Spring Vault for secret management

### SQL Injection

- Use JPA/prepared statements (never string concatenation)
- Parameterize queries

## Performance

### Database Queries

- Use pagination for large result sets
- Add appropriate indexes
- Use fetch joins to avoid N+1 queries
- Monitor query performance with `show-sql` (dev only)

### Caching

Consider caching for:
- Frequently read, rarely updated data
- Expensive computations
- External API responses

```java
@Cacheable(value = "boms", key = "#id")
public BomResponse findById(UUID id) {
    // implementation
}
```

## References

- [API Conventions](api-conventions.md) - REST API standards
- [Kafka Conventions](kafka-conventions.md) - Event-driven patterns
- [GitOps](gitops.md) - CI/CD and deployment
