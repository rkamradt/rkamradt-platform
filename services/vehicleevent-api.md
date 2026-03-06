# Vehicle Event API Service

## Overview

**Repository**: `~/github/vehicleevent-api`

**Domain**: Vehicle event ingestion and querying

**Status**: ⚠️ In Development

## Purpose

The Vehicle Event API service handles ingestion, storage, and querying of vehicle-related events. It provides an API for external systems to submit vehicle events and query historical vehicle data.

## Technical Stack

- **Language**: TBD (Java/Spring Boot assumed)
- **Framework**: TBD
- **Database**: TBD
- **Messaging**: Kafka (consumer/producer)

## API Endpoints

_(To be documented)_

### Planned Endpoints

- `POST /api/vehicles/{id}/events` - Submit vehicle event
- `GET /api/vehicles/{id}/events` - Retrieve vehicle event history
- `GET /api/vehicles/{id}` - Get vehicle details
- `GET /api/events` - Query all events (with filtering)

## Kafka Integration

### Events Published

_(To be defined)_

**Planned**:
- `vehicle.location.updated` - Vehicle location change
- `vehicle.status.changed` - Vehicle status update
- `vehicle.telemetry.received` - Telemetry data received

### Events Consumed

_(To be defined)_

## Database Schema

_(To be documented)_

**Planned Tables**:
- `vehicles` - Vehicle master data
- `vehicle_events` - Event history
- `vehicle_locations` - Location tracking

## Configuration

_(To be documented)_

### Environment Variables

- `DATABASE_URL` - Database connection string
- `KAFKA_BOOTSTRAP_SERVERS` - Kafka cluster
- `API_KEY` - API authentication key

## Development

### Local Setup

```bash
cd ~/github/vehicleevent-api

# Build
./mvnw clean install

# Run
./mvnw spring-boot:run
```

### Testing

```bash
# Unit tests
./mvnw test

# Integration tests
./mvnw verify
```

## Deployment

**Kubernetes Namespace**: `vehicleevent`

**Replicas**:
- Dev: 1
- Staging: 2
- Prod: 3

**ArgoCD Application**: `vehicleevent-api-{env}`

## Dependencies

### External

- Kafka cluster
- Database (type TBD)
- External vehicle data providers (if any)

### Internal

- None currently

## Monitoring

**Health Check**: `/actuator/health`

**Metrics**: `/actuator/metrics`, `/actuator/prometheus`

**Logs**: stdout (aggregated via Kubernetes)

## Common Operations

### View Logs

```bash
kubectl logs -n vehicleevent deployment/vehicleevent-api --tail=100 -f
```

### Check Health

```bash
curl http://vehicleevent-api/actuator/health
```

### Scale Replicas

```bash
kubectl scale deployment vehicleevent-api -n vehicleevent --replicas=3
```

## Known Issues

_(To be documented as they arise)_

## Roadmap

- [ ] Define API contracts (OpenAPI spec)
- [ ] Implement core event ingestion
- [ ] Implement event querying
- [ ] Add vehicle location tracking
- [ ] Integrate with external data sources
- [ ] Set up monitoring and alerting
- [ ] Performance testing and optimization

## References

- [Service Map](../architecture/service-map.md) - Service interactions
- [API Conventions](../conventions/api-conventions.md) - API design standards
- [Kafka Conventions](../conventions/kafka-conventions.md) - Event patterns

## Contact

**Owner**: Randal Kamradt
**Slack**: _(N/A - personal project)_
**On-Call**: _(N/A - personal project)_
