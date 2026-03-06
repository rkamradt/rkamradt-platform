# Event Schemas

## Overview

This document defines the payload structure for all events published on Kafka topics. Each schema includes:
- Field definitions and types
- Required vs optional fields
- Validation rules
- Example payloads
- Versioning information

## Schema Evolution Rules

1. **Backward Compatible Changes** (safe):
   - Adding optional fields
   - Removing optional fields
   - Widening field types (int32 → int64)

2. **Breaking Changes** (require new version):
   - Removing required fields
   - Changing field types
   - Renaming fields
   - Changing field semantics

3. **Versioning Strategy**:
   - Include `schemaVersion` field in every event
   - Use semantic versioning (e.g., "1.0.0")
   - Maintain backward compatibility within major version
   - Create new topic for major version changes (e.g., `manufacturing.bom.created.v2`)

## Common Fields

All events should include these standard fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `eventId` | string (UUID) | Yes | Unique identifier for this event instance |
| `eventType` | string | Yes | Event type name (e.g., "BomCreated") |
| `eventTimestamp` | string (ISO-8601) | Yes | When the event occurred (UTC) |
| `schemaVersion` | string | Yes | Schema version (semantic versioning) |
| `aggregateId` | string | Yes | ID of the domain aggregate (used for partitioning) |
| `aggregateType` | string | Yes | Type of aggregate (e.g., "Bom", "Order") |
| `causationId` | string (UUID) | No | ID of event/command that caused this event |
| `correlationId` | string (UUID) | No | ID for tracing related events across services |
| `metadata` | object | No | Additional context (user, system, etc.) |

---

## Manufacturing Domain Events

### BomCreatedEvent

**Topic**: `manufacturing.bom.created`

**Description**: Emitted when a new Bill of Materials is created in the manufacturing system.

**Schema Version**: 1.0.0

**Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `bomId` | string (UUID) | Yes | Unique identifier for the BOM |
| `productId` | string | Yes | Product identifier being manufactured |
| `productName` | string | Yes | Human-readable product name |
| `version` | string | Yes | BOM version (e.g., "1.0", "2.1") |
| `status` | enum | Yes | BOM status: DRAFT, APPROVED, OBSOLETE |
| `parts` | array | Yes | List of parts required (see PartItem below) |
| `estimatedCost` | decimal | No | Estimated total cost |
| `currency` | string | No | Currency code (ISO 4217, e.g., "USD") |
| `createdBy` | string | Yes | User/system that created the BOM |
| `notes` | string | No | Additional notes or comments |

**PartItem Structure**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `partId` | string | Yes | Part identifier |
| `partName` | string | Yes | Part name/description |
| `quantity` | integer | Yes | Number of units required |
| `unitCost` | decimal | No | Cost per unit |
| `supplier` | string | No | Preferred supplier |

**Example Payload**:

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "BomCreated",
  "eventTimestamp": "2026-03-05T14:30:00Z",
  "schemaVersion": "1.0.0",
  "aggregateId": "bom-12345",
  "aggregateType": "Bom",
  "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "bomId": "bom-12345",
  "productId": "PROD-001",
  "productName": "Widget Deluxe",
  "version": "1.0",
  "status": "APPROVED",
  "parts": [
    {
      "partId": "PART-101",
      "partName": "Aluminum Frame",
      "quantity": 2,
      "unitCost": 15.50,
      "supplier": "MetalCo"
    },
    {
      "partId": "PART-102",
      "partName": "Steel Bolt M6",
      "quantity": 8,
      "unitCost": 0.25,
      "supplier": "FastenerSupply"
    }
  ],
  "estimatedCost": 33.00,
  "currency": "USD",
  "createdBy": "engineer-jane",
  "notes": "Q2 production run"
}
```

---

### PartsOrderedEvent

**Topic**: `manufacturing.parts.ordered`

**Description**: Emitted when parts are ordered from a supplier.

**Schema Version**: 1.0.0

**Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `orderId` | string (UUID) | Yes | Unique order identifier |
| `bomId` | string (UUID) | Yes | Associated BOM ID |
| `supplierId` | string | Yes | Supplier identifier |
| `supplierName` | string | Yes | Supplier name |
| `orderDate` | string (ISO-8601) | Yes | When order was placed |
| `expectedDeliveryDate` | string (ISO-8601) | No | Expected delivery date |
| `parts` | array | Yes | Parts being ordered (see PartItem) |
| `totalCost` | decimal | Yes | Total order cost |
| `currency` | string | Yes | Currency code (ISO 4217) |
| `priority` | enum | No | Order priority: LOW, NORMAL, HIGH, URGENT |
| `shippingAddress` | object | No | Delivery address details |

**Example Payload**:

```json
{
  "eventId": "660e8400-e29b-41d4-a716-446655440001",
  "eventType": "PartsOrdered",
  "eventTimestamp": "2026-03-05T15:00:00Z",
  "schemaVersion": "1.0.0",
  "aggregateId": "order-67890",
  "aggregateType": "PartsOrder",
  "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "orderId": "order-67890",
  "bomId": "bom-12345",
  "supplierId": "SUP-001",
  "supplierName": "MetalCo",
  "orderDate": "2026-03-05T15:00:00Z",
  "expectedDeliveryDate": "2026-03-12T00:00:00Z",
  "parts": [
    {
      "partId": "PART-101",
      "partName": "Aluminum Frame",
      "quantity": 10,
      "unitCost": 15.50
    }
  ],
  "totalCost": 155.00,
  "currency": "USD",
  "priority": "NORMAL"
}
```

---

### WorkScheduledEvent

**Topic**: `manufacturing.work.scheduled`

**Description**: Emitted when manufacturing work is scheduled on a production line.

**Schema Version**: 1.0.0

**Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workOrderId` | string (UUID) | Yes | Work order identifier |
| `bomId` | string (UUID) | Yes | Associated BOM ID |
| `productionLineId` | string | Yes | Production line identifier |
| `scheduledStartTime` | string (ISO-8601) | Yes | Scheduled start time |
| `scheduledEndTime` | string (ISO-8601) | Yes | Scheduled end time |
| `quantity` | integer | Yes | Number of units to produce |
| `assignedTo` | array[string] | No | Worker/team IDs assigned |
| `priority` | enum | No | Priority: LOW, NORMAL, HIGH, URGENT |

**Example Payload**:

```json
{
  "eventId": "770e8400-e29b-41d4-a716-446655440002",
  "eventType": "WorkScheduled",
  "eventTimestamp": "2026-03-05T16:00:00Z",
  "schemaVersion": "1.0.0",
  "aggregateId": "work-11111",
  "aggregateType": "WorkOrder",
  "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "workOrderId": "work-11111",
  "bomId": "bom-12345",
  "productionLineId": "LINE-A",
  "scheduledStartTime": "2026-03-15T08:00:00Z",
  "scheduledEndTime": "2026-03-15T16:00:00Z",
  "quantity": 10,
  "assignedTo": ["worker-123", "worker-456"],
  "priority": "NORMAL"
}
```

---

### AssemblyCompletedEvent

**Topic**: `manufacturing.assembly.completed`

**Description**: Emitted when an assembly process is completed.

**Schema Version**: 1.0.0

**Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workOrderId` | string (UUID) | Yes | Work order identifier |
| `bomId` | string (UUID) | Yes | Associated BOM ID |
| `productionLineId` | string | Yes | Production line used |
| `completedAt` | string (ISO-8601) | Yes | Actual completion time |
| `quantityCompleted` | integer | Yes | Units successfully assembled |
| `quantityDefective` | integer | No | Units with defects |
| `completedBy` | array[string] | No | Workers who completed assembly |
| `duration` | integer | No | Actual duration in minutes |
| `notes` | string | No | Completion notes or issues |

**Example Payload**:

```json
{
  "eventId": "880e8400-e29b-41d4-a716-446655440003",
  "eventType": "AssemblyCompleted",
  "eventTimestamp": "2026-03-15T16:00:00Z",
  "schemaVersion": "1.0.0",
  "aggregateId": "work-11111",
  "aggregateType": "WorkOrder",
  "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "workOrderId": "work-11111",
  "bomId": "bom-12345",
  "productionLineId": "LINE-A",
  "completedAt": "2026-03-15T16:00:00Z",
  "quantityCompleted": 10,
  "quantityDefective": 0,
  "completedBy": ["worker-123", "worker-456"],
  "duration": 480,
  "notes": "Smooth production run"
}
```

---

### PartsReceivedEvent

**Topic**: `manufacturing.parts.received`

**Description**: Emitted when parts are received from a supplier.

**Schema Version**: 1.0.0

**Fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `receiptId` | string (UUID) | Yes | Receipt identifier |
| `orderId` | string (UUID) | Yes | Associated order ID |
| `receivedAt` | string (ISO-8601) | Yes | When parts were received |
| `receivedBy` | string | Yes | User/system that recorded receipt |
| `parts` | array | Yes | Parts received (see ReceivedPart) |
| `condition` | enum | Yes | GOOD, DAMAGED, PARTIAL |
| `notes` | string | No | Receipt notes or issues |

**ReceivedPart Structure**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `partId` | string | Yes | Part identifier |
| `quantityOrdered` | integer | Yes | Quantity originally ordered |
| `quantityReceived` | integer | Yes | Quantity actually received |
| `condition` | enum | Yes | GOOD, DAMAGED |

**Example Payload**:

```json
{
  "eventId": "990e8400-e29b-41d4-a716-446655440004",
  "eventType": "PartsReceived",
  "eventTimestamp": "2026-03-12T10:30:00Z",
  "schemaVersion": "1.0.0",
  "aggregateId": "receipt-22222",
  "aggregateType": "PartsReceipt",
  "correlationId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "receiptId": "receipt-22222",
  "orderId": "order-67890",
  "receivedAt": "2026-03-12T10:30:00Z",
  "receivedBy": "warehouse-bob",
  "parts": [
    {
      "partId": "PART-101",
      "quantityOrdered": 10,
      "quantityReceived": 10,
      "condition": "GOOD"
    }
  ],
  "condition": "GOOD",
  "notes": "All parts in good condition"
}
```

---

## Vehicle Domain Events

### VehicleLocationUpdatedEvent

**Topic**: `vehicle.location.updated` _(planned)_

**Description**: Emitted when a vehicle's location is updated.

**Schema Version**: 1.0.0 _(draft)_

**Fields**: _(to be defined)_

---

## Schema Registry Integration

### Development

For local development, schemas are documented here. Consider implementing:
- JSON Schema validation in producers/consumers
- Contract testing between services
- Schema compatibility checks in CI/CD

### Production (Future)

Consider using a schema registry:
- **Confluent Schema Registry**: Avro/Protobuf support
- **Apicurio Registry**: Open-source alternative
- **AWS Glue Schema Registry**: If using AWS

Benefits:
- Centralized schema management
- Automatic compatibility checking
- Schema versioning and evolution
- Consumer/producer validation

## Validation

### Producer Responsibilities

- Validate payload against schema before publishing
- Include all required fields
- Use correct data types
- Generate unique `eventId` for each event
- Set `eventTimestamp` accurately

### Consumer Responsibilities

- Handle missing optional fields gracefully
- Validate schema version compatibility
- Implement dead letter queue for invalid messages
- Log schema validation failures

## Testing Events

### Sample Event Generator

Create test events using:
```bash
kafka-console-producer --topic manufacturing.bom.created --bootstrap-server localhost:9092
```

### Integration Testing

- Use Testcontainers with Kafka for integration tests
- Verify event schema compatibility
- Test consumer behavior with various payloads
- Test error handling with malformed events

## References

- [Kafka Topics](kafka-topics.md) - Topic registry and ownership
- [Kafka Conventions](../conventions/kafka-conventions.md) - Event design patterns
- [Service Map](service-map.md) - Producers and consumers per service
