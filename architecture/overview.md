# Platform Architecture Overview

## Vision

The rkamradt-platform is a personal microservices sandbox designed to explore modern event-driven architecture patterns, Kafka streaming, and GitOps workflows. It prioritizes learning and experimentation over production-grade reliability.

## Architectural Principles

### 1. Event-Driven Architecture

Services communicate primarily through Kafka events rather than synchronous REST calls. This enables:
- **Loose coupling** - Services don't need to know about each other's implementation details
- **Temporal decoupling** - Producers and consumers don't need to be online simultaneously
- **Scalability** - Events can be replayed, buffered, and processed at different rates
- **Audit trail** - Event log provides complete history of system state changes

### 2. Polyrepo Structure

Each service lives in its own Git repository with independent versioning and deployment:
- Enables team autonomy (even for a team of one experimenting with different patterns)
- Allows polyglot technology choices per service
- Simplifies reasoning about service boundaries
- Reduces blast radius of changes

### 3. Domain-Driven Design

Services are organized around business domains:
- **Manufacturing Service** - Manages the complete manufacturing workflow from BOM creation through assembly
- **Vehicle Event API** - Handles vehicle-related event ingestion and querying
- Future services will align to clear domain boundaries

### 4. API-First Design

Each service exposes:
- **REST APIs** - For synchronous queries and commands
- **Kafka Events** - For asynchronous state changes and notifications
- **OpenAPI/Swagger Specs** - For API documentation and contract-first development

## High-Level Architecture

```
┌─────────────────┐         ┌─────────────────┐
│  Vehicle Event  │         │  Manufacturing  │
│      API        │         │     Service     │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │        ┌─────────┐        │
         └───────▶│  Kafka  │◀───────┘
                  └─────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    [Topic 1]     [Topic 2]     [Topic 3]
         │             │             │
         └─────────────┴─────────────┘
                       │
              (Event Stream)
```

## Service Communication Patterns

### Event-Driven (Primary)

Services publish domain events to Kafka topics when state changes occur:
1. Service performs business logic
2. Service emits event to Kafka topic
3. Interested services consume and react to event
4. Each consumer processes independently

**Example**: Manufacturing service publishes `manufacturing.bom.created` event when a new Bill of Materials is created. Other services (inventory, planning, etc.) can subscribe and react.

### Request-Response (Secondary)

REST APIs are used for:
- Queries that need immediate response
- Admin/management operations
- External client integrations
- Health checks and diagnostics

## Data Management

### Database Per Service

Each service owns its data:
- No shared databases between services
- Services expose data through APIs and events
- Supports independent technology choices (PostgreSQL, MongoDB, etc.)

### Event Sourcing (Future)

Consider event sourcing for services where:
- Complete audit history is valuable
- Time-travel queries are needed
- State needs to be rebuilt from events

## Deployment Architecture

### GitOps with ArgoCD

```
Developer     GitHub          GitHub Actions    GitHub Packages    ArgoCD    Kubernetes
   │              │                  │                 │             │           │
   ├─ git push ──▶│                  │                 │             │           │
   │              ├─ webhook ───────▶│                 │             │           │
   │              │                  ├─ build/test ───▶│             │           │
   │              │                  │                 │             │           │
   │              │                  ├─ update manifests             │           │
   │              │◀─────────────────┤                 │             │           │
   │              │                  │                 │             │           │
   │              ├─────────────────────────────────────┬─ sync ────▶│           │
   │              │                  │                 │             ├─ deploy ─▶│
```

### Infrastructure

- **Container Runtime**: Docker
- **Orchestration**: Kubernetes (local: minikube/kind, cloud: TBD)
- **Service Mesh**: Consider Istio/Linkerd for observability
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK stack or Loki

## Technology Stack

### Backend Services
- **Primary**: Spring Boot (Java 17+)
- **Future**: Node.js, Go, Python (polyglot experimentation)

### Messaging
- **Event Streaming**: Apache Kafka
- **Schema Registry**: Confluent Schema Registry or Apicurio

### Data Stores
- **Primary**: PostgreSQL
- **Future**: Redis (caching), MongoDB (document store)

### API Gateway
- **Options**: Spring Cloud Gateway, Kong, or Nginx

## Security

### Authentication & Authorization
- JWT tokens for API authentication
- Service-to-service: mTLS or API keys
- OAuth2/OIDC for future user-facing services

### Network Security
- Services communicate within private network
- API Gateway as single external entry point
- Network policies in Kubernetes

## Observability

### Metrics
- Service-level: Spring Boot Actuator + Micrometer
- System-level: Prometheus
- Visualization: Grafana dashboards

### Distributed Tracing
- OpenTelemetry instrumentation
- Jaeger or Zipkin for trace collection

### Logging
- Structured logging (JSON)
- Centralized log aggregation
- Correlation IDs across service boundaries

## Scalability Considerations

### Horizontal Scaling
- Stateless services scale via Kubernetes replicas
- Kafka consumers scale via consumer groups
- Database scaling: read replicas, connection pooling

### Kafka Partitioning
- Events partitioned by aggregate ID (e.g., vehicle ID, order ID)
- Guarantees ordering within partition
- Enables parallel processing across partitions

## Evolution Strategy

### Adding New Services

1. Define domain boundary and service responsibility
2. Create service repository from template
3. Document in `service-map.md`
4. Define event contracts in `event-schemas.md`
5. Update Kafka topic registry

### Decomposing Services

If a service grows too large:
1. Identify subdomain boundary
2. Plan event migration strategy
3. Extract new service gradually
4. Run both services in parallel during transition
5. Migrate consumers and deprecate old events

### Technology Upgrades

- Services upgrade independently
- Kafka supports rolling upgrades with backward compatibility
- Blue-green deployments for zero-downtime updates

## Current Limitations & Future Work

### Current State
- Early stage: 2 services implemented
- Single Kafka cluster (not HA)
- Local development focus
- Basic CI/CD

### Future Enhancements
- [ ] Multi-region deployment
- [ ] Advanced observability (distributed tracing)
- [ ] Service mesh implementation
- [ ] Event schema evolution strategy
- [ ] Chaos engineering experiments
- [ ] Cost optimization
- [ ] Advanced Kafka patterns (CQRS, event sourcing)

## References

- [Service Map](service-map.md) - Complete service catalog
- [Kafka Topics](kafka-topics.md) - Event topic registry
- [Event Schemas](event-schemas.md) - Event payload contracts
