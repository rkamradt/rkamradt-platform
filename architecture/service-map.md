# Service Map

## Overview

This document provides a comprehensive catalog of all services in the platform, their responsibilities, interactions, and technical details.

## Active Services

### Manufacturing Service

**Repository**: `~/github/manufacturing-service`

**Domain**: Manufacturing workflow orchestration

**Responsibility**:
- Manages complete manufacturing lifecycle from BOM creation to assembly completion
- Orchestrates parts ordering and work scheduling
- Tracks assembly progress and completion
- Publishes manufacturing events for downstream consumers

**Technology Stack**:
- Language: Java 17+
- Framework: Spring Boot
- Database: PostgreSQL (assumed)
- Messaging: Kafka producer

**REST API Endpoints**:
- `POST /api/bom` - Create Bill of Materials
- `GET /api/bom/{id}` - Retrieve BOM details
- `POST /api/orders` - Create parts order
- `POST /api/work-schedule` - Schedule manufacturing work
- `POST /api/assembly/complete` - Mark assembly as completed
- `POST /api/parts/receive` - Record parts receipt

**Events Published**:
- `manufacturing.bom.created` - New BOM created
- `manufacturing.parts.ordered` - Parts order placed
- `manufacturing.work.scheduled` - Work scheduled on production line
- `manufacturing.assembly.completed` - Assembly process completed
- `manufacturing.parts.received` - Parts received from supplier

**Events Consumed**: _(none currently)_

**Dependencies**:
- Kafka cluster
- PostgreSQL database

**Status**: ✅ Active

---

### Vehicle Event API

**Repository**: `~/github/vehicleevent-api`

**Domain**: Vehicle event ingestion and querying

**Responsibility**:
- Ingests vehicle-related events from various sources
- Provides query API for vehicle event history
- May aggregate and transform vehicle data

**Technology Stack**:
- Language: TBD (Java/Spring Boot assumed)
- Framework: TBD
- Database: TBD
- Messaging: Kafka consumer/producer (assumed)

**REST API Endpoints**: _(to be documented)_

**Events Published**: _(to be documented)_

**Events Consumed**: _(to be documented)_

**Dependencies**:
- Kafka cluster
- Database (type TBD)

**Status**: ⚠️ In Development

---

## Planned Services

### Inventory Service

**Domain**: Parts and materials inventory management

**Responsibility**:
- Track inventory levels across warehouses
- Monitor stock levels and alert on depletion
- Manage inventory reservations
- Process inventory updates from manufacturing and suppliers

**Events Published** (planned):
- `inventory.parts.depleted` - Stock level below threshold
- `inventory.parts.reserved` - Parts reserved for order
- `inventory.parts.stocked` - New parts added to inventory

**Events Consumed** (planned):
- `manufacturing.parts.ordered` - Reserve inventory
- `manufacturing.parts.received` - Update inventory levels
- `supplier.shipment.received` - Add to inventory

**Priority**: High

---

### Supplier Service

**Domain**: Supplier relationship and order management

**Responsibility**:
- Manage supplier catalog and relationships
- Process supplier orders
- Track shipments and deliveries
- Handle supplier performance metrics

**Events Published** (planned):
- `supplier.order.placed` - Order sent to supplier
- `supplier.shipment.dispatched` - Supplier shipped parts
- `supplier.shipment.received` - Shipment arrived

**Events Consumed** (planned):
- `manufacturing.parts.ordered` - Initiate supplier order

**Priority**: Medium

---

### Quality Service

**Domain**: Quality assurance and inspection

**Responsibility**:
- Perform quality inspections
- Track quality metrics
- Manage defect reporting
- Trigger rework or rejection flows

**Events Published** (planned):
- `quality.inspection.completed` - Inspection results
- `quality.defect.reported` - Defect identified
- `quality.rework.scheduled` - Rework needed

**Events Consumed** (planned):
- `manufacturing.assembly.completed` - Trigger inspection

**Priority**: Medium

---

### Planning Service

**Domain**: Production planning and scheduling

**Responsibility**:
- Optimize production schedules
- Balance workload across production lines
- Forecast resource needs
- Generate production plans from BOMs

**Events Published** (planned):
- `planning.schedule.created` - New production schedule
- `planning.schedule.updated` - Schedule adjustments

**Events Consumed** (planned):
- `manufacturing.bom.created` - Factor into planning
- `manufacturing.assembly.completed` - Update schedule progress

**Priority**: Low

---

## Service Interaction Patterns

### Choreography (Event-Driven)

Most service interactions use choreography where services react to events:

```
Manufacturing Service           Inventory Service
        │                              │
        ├─ parts.ordered ──────────────▶│
        │                              ├─ reserve inventory
        │                              ├─ inventory.reserved
        │                              │
        ├─ parts.received ─────────────▶│
        │                              ├─ update stock
        │                              └─ inventory.updated
```

### Orchestration (Request-Response)

Synchronous REST calls used for:
- UI-driven queries requiring immediate response
- Admin operations
- Health checks

### Hybrid Patterns

Complex workflows may combine both:
1. API call initiates workflow
2. Service emits events
3. Consumers react asynchronously
4. Final status queryable via API

## Service Communication Matrix

| From Service ↓ / To Service → | Manufacturing | Vehicle Event API | Inventory | Supplier | Quality |
|-------------------------------|---------------|-------------------|-----------|----------|---------|
| **Manufacturing**             | -             | ❌                | Event     | Event    | Event   |
| **Vehicle Event API**         | ❌            | -                 | ❌        | ❌       | ❌      |
| **Inventory**                 | Event         | ❌                | -         | Event    | ❌      |
| **Supplier**                  | Event         | ❌                | Event     | -        | ❌      |
| **Quality**                   | Event         | ❌                | ❌        | ❌       | -       |

Legend:
- **Event**: Asynchronous communication via Kafka
- **REST**: Synchronous HTTP API call
- **❌**: No direct communication

## Cross-Cutting Concerns

### API Gateway

**Status**: Planned

All external API traffic should route through a gateway providing:
- Authentication/authorization
- Rate limiting
- Request/response transformation
- API versioning

### Service Discovery

**Status**: TBD

Options:
- Kubernetes DNS (native)
- Consul/Eureka (if needed)
- Hardcoded endpoints (dev/simple deployments)

### Configuration Management

Each service manages configuration via:
- Application properties (checked into repo)
- Environment variables (deployment-specific)
- ConfigMaps/Secrets (Kubernetes)
- Potentially: Spring Cloud Config Server

## Adding a New Service

Checklist:

1. **Define Domain Boundary**
   - [ ] Identify domain responsibility
   - [ ] Define service boundaries
   - [ ] Document in this file

2. **Create Repository**
   - [ ] Create service repo from template
   - [ ] Add CLAUDE.md with service context
   - [ ] Set up basic project structure

3. **Define Contracts**
   - [ ] Document REST API endpoints
   - [ ] Define events published/consumed
   - [ ] Add event schemas to `event-schemas.md`
   - [ ] Register topics in `kafka-topics.md`

4. **Implementation**
   - [ ] Implement core business logic
   - [ ] Set up database schema
   - [ ] Configure Kafka integration
   - [ ] Add health checks

5. **Deployment**
   - [ ] Create Dockerfile
   - [ ] Set up CI/CD pipeline
   - [ ] Create Kubernetes manifests
   - [ ] Configure monitoring/logging

6. **Documentation**
   - [ ] Update this service map
   - [ ] Create service summary in `services/`
   - [ ] Document integration points
   - [ ] Add runbook for operations

## Service Health Dashboard

| Service | Status | Health Endpoint | Metrics | Logs | Repository |
|---------|--------|----------------|---------|------|------------|
| Manufacturing | ✅ Active | `/actuator/health` | Prometheus | stdout | [Link](~/github/manufacturing-service) |
| Vehicle Event API | ⚠️ Dev | TBD | TBD | TBD | [Link](~/github/vehicleevent-api) |

## References

- [Kafka Topics](kafka-topics.md) - Topic registry and event streams
- [Event Schemas](event-schemas.md) - Event payload definitions
- [Architecture Overview](overview.md) - High-level architecture
- [API Conventions](../conventions/api-conventions.md) - API design standards
