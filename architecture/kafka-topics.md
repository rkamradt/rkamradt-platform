# Kafka Topics Registry

## Overview

This document serves as the master registry for all Kafka topics used across the platform. Each topic represents a stream of events for a specific domain concern.

## Topic Naming Convention

Format: `{domain}.{entity}.{event-type}`

- **domain**: Business domain (e.g., `manufacturing`, `vehicle`, `inventory`)
- **entity**: The subject of the event (e.g., `bom`, `parts`, `work`)
- **event-type**: What happened (e.g., `created`, `updated`, `completed`)

Examples:
- `manufacturing.bom.created`
- `vehicle.location.updated`
- `inventory.parts.depleted`

## Topics Registry

| Topic Name | Producer Service | Consumer Service(s) | Event Schema | Partitions | Retention | Notes |
|------------|------------------|---------------------|--------------|------------|-----------|-------|
| `manufacturing.bom.created` | manufacturing-service | inventory-service (future), planning-service (future) | [BomCreatedEvent](event-schemas.md#bomcreatedevent) | 3 | 7 days | Emitted when new Bill of Materials is created |
| `manufacturing.parts.ordered` | manufacturing-service | inventory-service (future), supplier-service (future) | [PartsOrderedEvent](event-schemas.md#partsorderedevent) | 3 | 7 days | Parts order placed with supplier |
| `manufacturing.work.scheduled` | manufacturing-service | shop-floor-service (future), workforce-service (future) | [WorkScheduledEvent](event-schemas.md#workscheduledevent) | 3 | 7 days | Manufacturing work scheduled on assembly line |
| `manufacturing.assembly.completed` | manufacturing-service | quality-service (future), inventory-service (future) | [AssemblyCompletedEvent](event-schemas.md#assemblycompletedevent) | 3 | 7 days | Assembly process completed |
| `manufacturing.parts.received` | manufacturing-service | inventory-service (future) | [PartsReceivedEvent](event-schemas.md#partsreceivedevent) | 3 | 7 days | Parts received from supplier |

## Topic Configuration Guidelines

### Partitions

- **Default**: 3 partitions for new topics
- **Partition Key**: Aggregate ID (e.g., BOM ID, Order ID) to guarantee ordering
- **Scale-up**: Increase partitions based on throughput needs (cannot decrease)

### Retention

- **Default**: 7 days for most topics
- **Audit topics**: 30+ days for compliance/debugging
- **High-volume topics**: 2-3 days to manage storage
- **Event-sourced topics**: Infinite (compact topics)

### Replication Factor

- **Development**: 1 (single Kafka broker)
- **Production**: 3 (minimum for HA)

## Event Ordering Guarantees

Kafka guarantees ordering **within a partition**. To maintain event ordering:

1. Use consistent partition key (e.g., always use `bomId` for BOM-related events)
2. Single producer instance per logical entity (or use idempotent producer)
3. Consumer reads from assigned partitions sequentially

## Adding New Topics

When creating a new topic:

1. **Choose appropriate name** following naming convention
2. **Document in this registry** with producer, consumers, schema
3. **Define event schema** in [event-schemas.md](event-schemas.md)
4. **Create topic** via admin tools or auto-creation (dev only)
5. **Update service documentation** in respective service repos

## Topic Ownership

Each topic has one **primary producer** service that owns the event definition. Other services may emit to the same topic only if:
- They are emitting the same logical event type
- The event schema is compatible
- There is clear coordination on event semantics

## Deprecated Topics

| Topic Name | Deprecated Date | Replacement | Migration Deadline | Notes |
|------------|----------------|-------------|-------------------|-------|
| _(none yet)_ | - | - | - | - |

## Topic Aliases and Mirrors

For cross-cluster replication or migration:

| Alias/Mirror | Source Topic | Destination Cluster | Purpose |
|--------------|--------------|---------------------|---------|
| _(none yet)_ | - | - | - |

## Monitoring and Alerts

Key metrics to monitor per topic:

- **Lag**: Consumer lag per partition (alert if > 10,000 messages)
- **Throughput**: Messages/sec produced and consumed
- **Error rate**: Failed message processing attempts
- **Partition skew**: Uneven distribution across partitions

## Troubleshooting Guide

### High Consumer Lag

1. Check consumer processing time
2. Increase consumer instances (up to partition count)
3. Optimize consumer logic
4. Consider increasing partitions (requires rebalance)

### Uneven Partition Distribution

1. Review partition key strategy
2. Check for hot keys (specific IDs with high traffic)
3. Consider randomization or compound keys

### Message Loss

1. Verify producer `acks=all` configuration
2. Check replication factor meets requirements
3. Review error handling and retry logic
4. Check disk space on Kafka brokers

## Future Topics (Planned)

| Topic Name | Target Service | Purpose | Priority |
|------------|----------------|---------|----------|
| `vehicle.location.updated` | vehicle-service | Real-time vehicle location tracking | High |
| `inventory.parts.depleted` | inventory-service | Alert when parts inventory low | Medium |
| `quality.inspection.completed` | quality-service | Quality inspection results | Medium |
| `supplier.shipment.dispatched` | supplier-service | Supplier shipment tracking | Low |

## References

- [Event Schemas](event-schemas.md) - Detailed event payload contracts
- [Kafka Conventions](../conventions/kafka-conventions.md) - Event design patterns and standards
- [Service Map](service-map.md) - Service responsibilities and interactions
