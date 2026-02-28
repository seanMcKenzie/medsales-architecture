# MedSales Architecture

Architecture documentation for the Medical Sales Intelligence & CRM Platform.

## Documents

| Document | Description |
|----------|-------------|
| [System Architecture Overview](system-architecture-overview.md) | High-level system context, principles, data flows, security |
| [Component Diagram](component-diagram.md) | Application components, domain services, API endpoints |
| [Data Architecture](data-architecture.md) | Database schema, ingestion pipeline, caching, backup |
| [Deployment Architecture](deployment-architecture.md) | AWS infrastructure, Docker, CI/CD, monitoring, DR |
| [Geocoding Service](geocoding-service.md) | Dedicated geocoding microservice — multi-provider, caching, batch processing, cost optimization |

## Architecture Decision Records

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](adr/ADR-001-database.md) | PostgreSQL + PostGIS as Primary Database | Accepted |
| [ADR-002](adr/ADR-002-search.md) | Search Strategy — PostGIS for v1, Elasticsearch for v2 | Accepted |
| [ADR-003](adr/ADR-003-mobile.md) | React Native for Cross-Platform Mobile | Accepted |

## Author

Frank Reynolds, DevOps & Solutions Architect  
February 28, 2026
