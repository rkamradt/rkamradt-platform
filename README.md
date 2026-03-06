# rkamradt-platform

> **Spine Repository** — Contains NO production code. Used for cross-repo context and orchestration in Claude Code sessions.

## What is this?

This is the central coordination repository for a personal microservices platform. It provides:

- **Architecture documentation** - System-level design, service interactions, event contracts
- **Coding conventions** - Standards shared across all services
- **Cross-service context** - For AI-assisted development sessions spanning multiple repos
- **Task tracking** - For features/fixes that touch multiple services

## Quick Start

### For AI Sessions (Claude Code)

Load this repository first to get platform-wide context:

```bash
cd ~/github/rkamradt-platform
```

Then add individual service repos as needed:

```
/add-dir ../vehicleevent-api
/add-dir ../manufacturing-service
```

See [`CLAUDE.md`](CLAUDE.md) for detailed context instructions.

### For Humans

1. Browse `architecture/` to understand system design
2. Check `conventions/` for coding standards
3. See `services/` for per-service summaries
4. Create task files in `tasks/` for cross-service work

## Services

| Service | Repository | Description |
|---------|-----------|-------------|
| vehicleevent-api | `~/github/vehicleevent-api` | Vehicle event ingestion and API |
| manufacturing-service | `~/github/manufacturing-service` | Manufacturing workflow orchestration |

## Platform Stack

- **Event Streaming**: Apache Kafka
- **Backend Services**: Spring Boot (Java), potentially others
- **API Gateway**: TBD
- **CI/CD**: GitHub Actions
- **Deployment**: ArgoCD (GitOps)
- **Artifact Registry**: GitHub Packages

## Repository Structure

```
rkamradt-platform/
├── CLAUDE.md              # Master AI context file
├── architecture/          # System architecture docs
├── conventions/           # Coding standards
├── tasks/                 # Cross-service task tracking
└── services/              # Per-service summaries
```

## Philosophy

This is a **learning platform** optimizing for:

- ✅ Experimentation and learning
- ✅ Modern microservices patterns
- ✅ Event-driven architecture
- ✅ GitOps workflows
- ✅ AI-assisted development

Not optimizing for:

- ❌ Production-grade reliability
- ❌ Enterprise compliance
- ❌ Performance at scale

## Contributing

This is a personal sandbox, but patterns and conventions are documented for:
- Consistency across experiments
- Learning reinforcement
- Portfolio demonstration
- AI context quality

## License

MIT License - feel free to learn from or fork any patterns here.
