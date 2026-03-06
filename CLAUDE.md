# rkamradt-platform: Master Context for Claude Code

## Overview

This is a **spine repository** for the rkamradt-platform — a personal microservices sandbox for exploring event-driven architecture, Kafka streaming, and GitOps workflows.

**Important:** This repository contains NO production code. It exists solely to provide cross-repo context and orchestration for Claude Code sessions that span multiple microservices.

## Platform Architecture

The platform consists of multiple independent microservices communicating via Kafka events. Each service:
- Lives in its own Git repository (polyrepo architecture)
- Has its own CLAUDE.md with service-specific context
- Publishes and consumes events following shared conventions
- Deploys independently via GitHub Actions and ArgoCD

## Known Services

| Service | Repository Path | Purpose |
|---------|----------------|---------|
| vehicleevent-api | `~/github/vehicleevent-api` | Vehicle event ingestion and API |
| manufacturing-service | `~/github/manufacturing-service` | Manufacturing workflow orchestration |

## Kafka Event Registry

All services communicate via Kafka. See [`architecture/kafka-topics.md`](architecture/kafka-topics.md) for the complete topic registry including:
- `manufacturing.bom.created` - Bill of Materials created events
- `manufacturing.parts.ordered` - Parts order events
- `manufacturing.work.scheduled` - Work scheduling events
- `manufacturing.assembly.completed` - Assembly completion events
- `manufacturing.parts.received` - Parts receipt confirmation events

## Working Across Services

### Adding Service Context to Your Session

When working on tasks that span multiple services, use:

```
/add-dir ../vehicleevent-api
/add-dir ../manufacturing-service
```

This loads service-specific context from their individual CLAUDE.md files.

### Task Management

For cross-service work, create a task file in `tasks/` using the template at [`tasks/TEMPLATE.md`](tasks/TEMPLATE.md). This helps track:
- Which repos are in scope
- Key architectural decisions
- Implementation phases across services
- Integration points and dependencies

## Coding Conventions

All services follow standardized conventions documented in `conventions/`:

- **[Java Conventions](conventions/java-conventions.md)** - Spring Boot, package structure, naming standards
- **[Kafka Conventions](conventions/kafka-conventions.md)** - Event naming, partitioning, error handling patterns
- **[API Conventions](conventions/api-conventions.md)** - REST API design standards
- **[GitOps](conventions/gitops.md)** - CI/CD with GitHub Actions, ArgoCD, GitHub Packages

## Architecture Documentation

Detailed architecture documentation is available in `architecture/`:

- **[Overview](architecture/overview.md)** - System architecture narrative
- **[Service Map](architecture/service-map.md)** - All services, their purpose, and interactions
- **[Kafka Topics](architecture/kafka-topics.md)** - Master registry of all topics
- **[Event Schemas](architecture/event-schemas.md)** - Shared event payload contracts

## Repository Structure

```
rkamradt-platform/
├── CLAUDE.md                    # This file - loaded first by Claude
├── README.md                    # Human-readable project overview
├── architecture/                # System architecture documentation
│   ├── overview.md
│   ├── kafka-topics.md
│   ├── service-map.md
│   └── event-schemas.md
├── conventions/                 # Coding and design standards
│   ├── java-conventions.md
│   ├── kafka-conventions.md
│   ├── api-conventions.md
│   └── gitops.md
├── tasks/                       # Cross-service task tracking
│   └── TEMPLATE.md
└── services/                    # Per-service context summaries
    ├── vehicleevent-api.md
    └── manufacturing-service.md
```

## Getting Started

1. **For single-service work**: Navigate to the service repo and work there
2. **For cross-service work**: Start here, add relevant service directories to context, create a task file
3. **When adding new services**: Update this file, add entry to `architecture/service-map.md`, create service summary in `services/`
4. **When defining new events**: Add to `architecture/kafka-topics.md` and `architecture/event-schemas.md`

## Philosophy

This platform is a learning sandbox optimizing for:
- **Experimentation** over production perfection
- **Polyglot architecture** - right tool for the job
- **Event-driven patterns** - loose coupling, eventual consistency
- **GitOps workflows** - infrastructure as code, automated deployments
- **Observability** - understanding system behavior through events and metrics
