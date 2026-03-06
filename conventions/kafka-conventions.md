# Kafka Conventions

## Overview

This document defines standards and patterns for Kafka-based event-driven communication across the rkamradt-platform services.

## Event Design Principles

### 1. Events are Facts

- Events represent things that **have happened** (past tense)
- Events are **immutable** - once published, they cannot be changed
- Events should be **self-contained** with all necessary context

### 2. Events vs Commands

- **Events**: Notifications of state changes (e.g., `BomCreated`, `PartsOrdered`)
- **Commands**: Requests to perform actions (e.g., `CreateBom`, `OrderParts`)
- This platform uses primarily events; commands may be REST API calls

### 3. Event Granularity

- Events should represent a **single domain fact**
- Avoid "god events" with unrelated data
- Balance between too coarse (hard to consume) and too fine-grained (event explosion)

## Topic Design

### Naming Convention

**Format**: `{domain}.{entity}.{event-type}`

- **domain**: Business domain (lowercase, e.g., `manufacturing`, `inventory`)
- **entity**: Subject of event (lowercase, singular, e.g., `bom`, `parts`)
- **event-type**: What happened (lowercase, past tense, e.g., `created`, `updated`, `deleted`)

**Examples**:
- `manufacturing.bom.created`
- `manufacturing.parts.ordered`
- `inventory.stock.depleted`

### Topic vs Aggregate

- Generally one topic per event type
- Alternative: One topic per aggregate with event type in payload
- Platform standard: **One topic per event type** for clarity

### Partitioning Strategy

**Partition Key**: Use the aggregate ID (e.g., BOM ID, Order ID)

**Benefits**:
- Guarantees **ordering** for events related to the same aggregate
- Enables **parallel processing** across different aggregates
- Allows **consumer scaling** up to partition count

**Example**:
```java
@Bean
public NewTopic bomCreatedTopic() {
    return TopicBuilder.name("manufacturing.bom.created")
        .partitions(3)  // Start with 3, scale as needed
        .replicas(1)    // Dev: 1, Prod: 3
        .build();
}

// When publishing
ProducerRecord<String, BomCreatedEvent> record = new ProducerRecord<>(
    "manufacturing.bom.created",
    event.getBomId().toString(),  // Partition key = aggregate ID
    event
);
```

### Retention Policy

| Use Case | Retention | Compaction |
|----------|-----------|------------|
| Regular events | 7 days | No |
| Audit/compliance | 30-90 days | No |
| Event sourcing | Infinite | Yes (optional) |
| High-volume events | 2-3 days | No |

Configure per topic:
```java
@Bean
public NewTopic bomCreatedTopic() {
    return TopicBuilder.name("manufacturing.bom.created")
        .partitions(3)
        .configs(Map.of(
            TopicConfig.RETENTION_MS_CONFIG, "604800000"  // 7 days
        ))
        .build();
}
```

## Event Schema Design

### Standard Fields

Every event MUST include:

```java
public abstract class BaseEvent {
    private UUID eventId;              // Unique event instance ID
    private String eventType;          // Event type name
    private Instant eventTimestamp;    // When event occurred (UTC)
    private String schemaVersion;      // Schema version (semantic versioning)
    private String aggregateId;        // ID of the domain aggregate
    private String aggregateType;      // Type of aggregate
    private UUID correlationId;        // For tracing related events (optional)
    private UUID causationId;          // ID of event/command that caused this (optional)
    private Map<String, String> metadata;  // Additional context (optional)
}
```

### Event Classes

```java
public class BomCreatedEvent extends BaseEvent {
    private UUID bomId;
    private String productId;
    private String productName;
    private String version;
    private BomStatus status;
    private List<PartItem> parts;
    private BigDecimal estimatedCost;
    private String currency;
    private String createdBy;
    private String notes;

    // Constructor, getters, setters
}
```

Or use records:

```java
public record BomCreatedEvent(
    UUID eventId,
    String eventType,
    Instant eventTimestamp,
    String schemaVersion,
    UUID bomId,  // aggregateId
    String productId,
    String productName,
    String version,
    BomStatus status,
    List<PartItem> parts,
    BigDecimal estimatedCost,
    String currency,
    String createdBy,
    String notes
) {}
```

### Schema Evolution

**Backward Compatible** (safe):
- Adding optional fields
- Adding default values
- Widening types (int → long)

**Breaking Changes** (require new version):
- Removing fields
- Renaming fields
- Changing field types
- Changing field semantics

**Versioning Strategy**:
1. Include `schemaVersion` in every event
2. Consumers must handle multiple versions
3. For breaking changes, create new topic (e.g., `manufacturing.bom.created.v2`)

## Producer Patterns

### Spring Kafka Producer

```java
@Service
public class ManufacturingEventProducer {
    private static final Logger log = LoggerFactory.getLogger(ManufacturingEventProducer.class);
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ManufacturingEventProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishBomCreated(BomCreatedEvent event) {
        String topic = "manufacturing.bom.created";
        String key = event.getBomId().toString();

        log.info("Publishing event to topic {}: {}", topic, event.getEventId());

        kafkaTemplate.send(topic, key, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event {}: {}", event.getEventId(), ex.getMessage(), ex);
                    // Handle failure (retry, dead letter queue, etc.)
                } else {
                    log.debug("Event {} published successfully to partition {}",
                        event.getEventId(),
                        result.getRecordMetadata().partition());
                }
            });
    }
}
```

### Transactional Outbox Pattern

For reliable event publishing with database transactions:

```java
@Transactional
public BomResponse createBom(CreateBomRequest request) {
    // 1. Save entity to database
    Bom bom = bomRepository.save(toBom(request));

    // 2. Save event to outbox table (same transaction)
    OutboxEvent outboxEvent = new OutboxEvent(
        UUID.randomUUID(),
        "manufacturing.bom.created",
        bom.getId().toString(),  // partition key
        toJson(new BomCreatedEvent(...))
    );
    outboxRepository.save(outboxEvent);

    // 3. Separate process reads outbox and publishes to Kafka
    return toBomResponse(bom);
}
```

### Producer Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # Wait for all replicas to acknowledge
      retries: 3
      properties:
        enable.idempotence: true  # Exactly-once semantics
        max.in.flight.requests.per.connection: 5
```

## Consumer Patterns

### Spring Kafka Consumer

```java
@Service
public class BomCreatedEventConsumer {
    private static final Logger log = LoggerFactory.getLogger(BomCreatedEventConsumer.class);
    private final InventoryService inventoryService;

    public BomCreatedEventConsumer(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @KafkaListener(
        topics = "manufacturing.bom.created",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(BomCreatedEvent event) {
        log.info("Received BomCreatedEvent: {}", event.getEventId());

        try {
            inventoryService.checkPartsAvailability(event);
            log.debug("Successfully processed event: {}", event.getEventId());
        } catch (Exception ex) {
            log.error("Failed to process event {}: {}", event.getEventId(), ex.getMessage(), ex);
            throw ex;  // Trigger retry or DLQ
        }
    }
}
```

### Idempotent Consumers

Consumers MUST be idempotent (safe to process same event multiple times):

```java
@Transactional
public void consume(BomCreatedEvent event) {
    // Check if already processed (using event ID)
    if (processedEventRepository.existsById(event.getEventId())) {
        log.debug("Event {} already processed, skipping", event.getEventId());
        return;
    }

    // Process event
    inventoryService.checkPartsAvailability(event);

    // Mark as processed
    processedEventRepository.save(new ProcessedEvent(event.getEventId(), Instant.now()));
}
```

### Consumer Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest
      enable-auto-commit: false  # Manual commit for better control
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: net.kamradtfamily.*
        spring.json.type.mapping: BomCreatedEvent:net.kamradtfamily.inventory.event.BomCreatedEvent
    listener:
      ack-mode: manual  # Commit after successful processing
```

### Error Handling

#### Retry with Backoff

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, BomCreatedEvent> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, BomCreatedEvent> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setCommonErrorHandler(new DefaultErrorHandler(
        new FixedBackOff(1000L, 3L)  // 3 retries with 1 second backoff
    ));
    return factory;
}
```

#### Dead Letter Queue (DLQ)

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaOperations<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
        template,
        (record, ex) -> {
            // Send to DLQ topic
            return new TopicPartition(record.topic() + ".DLQ", record.partition());
        }
    );

    return new DefaultErrorHandler(
        recoverer,
        new FixedBackOff(1000L, 3L)
    );
}
```

#### DLQ Monitoring

Create DLQ consumers to monitor and handle failed events:

```java
@KafkaListener(
    topics = "manufacturing.bom.created.DLQ",
    groupId = "dlq-monitor"
)
public void consumeDLQ(ConsumerRecord<String, BomCreatedEvent> record) {
    log.error("Event failed after retries: topic={}, partition={}, offset={}",
        record.topic(), record.partition(), record.offset());

    // Alert, log to monitoring system, manual review, etc.
}
```

## Event Choreography Patterns

### Simple Event Chain

Service A emits event → Service B consumes and emits event → Service C consumes

```
Manufacturing Service          Inventory Service          Supplier Service
        │                             │                         │
        ├─ bom.created ───────────────▶│                         │
        │                             ├─ check stock             │
        │                             ├─ parts.reserved          │
        │                             │                         │
        ├─ parts.ordered ──────────────┴────────────────────────▶│
        │                                                       ├─ order from supplier
```

### Event Fan-Out

Multiple services consume the same event:

```
Manufacturing Service
        │
        ├─ bom.created ────┬─────────▶ Inventory Service
        │                  │
        │                  ├─────────▶ Planning Service
        │                  │
        │                  └─────────▶ Cost Analysis Service
```

### Correlation ID

Track related events across service boundaries:

```java
// Service A creates initial correlation ID
UUID correlationId = UUID.randomUUID();
BomCreatedEvent bomEvent = new BomCreatedEvent(..., correlationId, null);
producer.publish(bomEvent);

// Service B propagates correlation ID
@KafkaListener(topics = "manufacturing.bom.created")
public void consume(BomCreatedEvent bomEvent) {
    PartsReservedEvent partsEvent = new PartsReservedEvent(
        ...,
        bomEvent.getCorrelationId(),  // Propagate
        bomEvent.getEventId()         // This event caused the new event
    );
    producer.publish(partsEvent);
}
```

## Testing

### Embedded Kafka

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"manufacturing.bom.created"})
class ManufacturingEventProducerTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Test
    void shouldPublishBomCreatedEvent() {
        // Test implementation
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
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void shouldConsumeAndProcessEvent() {
        // Test implementation
    }
}
```

### Consumer Testing

```java
@Test
void shouldHandleBomCreatedEvent() {
    BomCreatedEvent event = new BomCreatedEvent(...);

    consumer.consume(event);

    verify(inventoryService).checkPartsAvailability(event);
}
```

## Monitoring and Observability

### Metrics

Track these key metrics:

- **Producer**:
  - Messages sent per topic
  - Send failures
  - Send latency

- **Consumer**:
  - Consumer lag per partition
  - Processing time per message
  - Processing failures
  - DLQ message count

### Logging

Include in log messages:

- Event ID
- Correlation ID
- Topic name
- Partition/offset (for troubleshooting)

```java
log.info("Processing event: eventId={}, correlationId={}, topic={}, partition={}, offset={}",
    event.getEventId(),
    event.getCorrelationId(),
    record.topic(),
    record.partition(),
    record.offset());
```

### Distributed Tracing

Use OpenTelemetry/Sleuth for distributed tracing:

```java
@KafkaListener(topics = "manufacturing.bom.created")
@NewSpan("process-bom-created")  // Create new trace span
public void consume(BomCreatedEvent event) {
    // Processing
}
```

## Best Practices

### DO

✅ Make events immutable
✅ Include all necessary context in event
✅ Use correlation IDs for tracing
✅ Make consumers idempotent
✅ Version your schemas
✅ Use dead letter queues
✅ Monitor consumer lag
✅ Partition by aggregate ID
✅ Log event processing (success and failures)

### DON'T

❌ Put sensitive data (passwords, tokens) in events
❌ Use Kafka as a database
❌ Create circular event dependencies
❌ Assume events are processed immediately
❌ Ignore DLQ messages
❌ Let consumer lag grow unbounded
❌ Skip error handling
❌ Use Kafka for request-response patterns (use REST)

## Troubleshooting

### High Consumer Lag

1. Check consumer processing time
2. Increase consumer instances (up to partition count)
3. Optimize consumer code
4. Increase topic partitions (requires rebalancing)

### Duplicate Events

- Ensure consumers are idempotent
- Check producer `enable.idempotence` setting
- Verify exactly-once semantics configuration

### Lost Events

- Check producer `acks` configuration (should be `all`)
- Verify replication factor
- Check disk space on Kafka brokers

## References

- [Event Schemas](../architecture/event-schemas.md) - Event payload definitions
- [Kafka Topics](../architecture/kafka-topics.md) - Topic registry
- [Java Conventions](java-conventions.md) - Java coding standards
