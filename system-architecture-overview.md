# System Architecture Overview
## Medical Sales Intelligence & CRM Platform

**Author:** Frank Reynolds, DevOps & Solutions Architect  
**Date:** February 28, 2026  
**Version:** 1.0

---

## 1. Executive Summary

This document describes the high-level system architecture for the Medical Sales Intelligence & CRM Platform — a mobile-first application that aggregates publicly available CMS healthcare data to build rich physician profiles, supports territory management and route planning, and provides CRM functionality for pharmaceutical and medical device sales reps.

The architecture is designed for:
- **~5 million physician records** with multi-source data joins
- **10,000+ concurrent users** (field reps, managers, admins)
- **Mobile-first** with offline capability
- **Multi-tenant SaaS** with organization-level data isolation
- **99.9% uptime** SLA

---

## 2. System Context

```mermaid
graph TD
    subgraph Users
        Rep[📱 Field Sales Rep<br>Mobile App]
        Mgr[💻 Territory Manager<br>Web App]
        Admin[🔧 Sales Ops / Admin<br>Web App]
    end

    subgraph Platform["Medical Sales Intelligence Platform"]
        API[Spring Boot API Gateway]
        Worker[Data Ingestion Workers]
    end

    subgraph "External Data Sources (Free)"
        CMS1[NPPES NPI Registry]
        CMS2[Provider Data Catalog]
        CMS3[Medicare Part B Utilization]
        CMS4[Medicare Part D Prescribers]
        CMS5[Open Payments]
        CMS6[Hospital Compare]
        CMS7[Facility Affiliation Data]
    end

    subgraph "External Services"
        GEO[Google Maps / Mapbox<br>Geocoding API]
        PUSH[Firebase Cloud Messaging<br>Push Notifications]
        AUTH[Identity Provider<br>SSO / SAML 2.0]
    end

    Rep -->|HTTPS| API
    Mgr -->|HTTPS| API
    Admin -->|HTTPS| API

    Worker -->|Download CSV| CMS1
    Worker -->|Download CSV| CMS2
    Worker -->|Download CSV| CMS3
    Worker -->|Download CSV| CMS4
    Worker -->|Download CSV| CMS5
    Worker -->|Download CSV| CMS6
    Worker -->|Download CSV| CMS7

    Worker -->|Geocode addresses| GEO
    API -->|Send notifications| PUSH
    API -->|Authenticate| AUTH
```

---

## 3. Architecture Principles

| # | Principle | Rationale |
|---|-----------|-----------|
| 1 | **API-first** | Single API layer serves mobile, web, and future integrations |
| 2 | **Data layer separation** | Public CMS data (read-heavy, shared) is separated from CRM data (read-write, tenant-isolated) |
| 3 | **Horizontal scalability** | Stateless API servers behind a load balancer; scale by adding instances |
| 4 | **Offline-first mobile** | Core physician profiles and CRM logging work without connectivity |
| 5 | **Multi-tenant by design** | Organization-level data isolation from day one, not bolted on later |
| 6 | **Automate everything** | Data ingestion, deployments, monitoring — no manual processes |
| 7 | **Free data, zero licensing cost** | Core data foundation is 100% public CMS data. No vendor lock-in on data |

---

## 4. High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        Mobile[Mobile App<br>React Native<br>iOS + Android]
        Web[Web App<br>React SPA<br>Manager Dashboard]
    end

    subgraph "Edge Layer"
        CDN[CloudFront CDN<br>Static Assets]
        ALB[Application Load Balancer<br>TLS Termination]
    end

    subgraph "Application Layer"
        API1[Spring Boot API<br>Instance 1]
        API2[Spring Boot API<br>Instance 2]
        APIN[Spring Boot API<br>Instance N]
    end

    subgraph "Caching Layer"
        Redis[Redis Cluster<br>Session + Query Cache]
    end

    subgraph "Data Layer"
        PG_PUB[(PostgreSQL + PostGIS<br>Physician Data<br>Read Replicas)]
        PG_CRM[(PostgreSQL<br>CRM Data<br>Per-Org Isolation)]
        S3[S3 / Object Storage<br>CMS Raw Files<br>Exports]
    end

    subgraph "Background Processing"
        Queue[SQS / RabbitMQ<br>Job Queue]
        Ingest[Data Ingestion Workers<br>Spring Batch]
        Notify[Notification Worker]
    end

    subgraph "Observability"
        Logs[CloudWatch / ELK<br>Centralized Logging]
        Metrics[Prometheus + Grafana<br>Metrics & Dashboards]
        Alerts[PagerDuty / OpsGenie<br>Alerting]
    end

    Mobile --> CDN
    Web --> CDN
    Mobile --> ALB
    Web --> ALB
    ALB --> API1
    ALB --> API2
    ALB --> APIN
    API1 --> Redis
    API1 --> PG_PUB
    API1 --> PG_CRM
    API2 --> Redis
    API2 --> PG_PUB
    API2 --> PG_CRM
    Queue --> Ingest
    Ingest --> PG_PUB
    Ingest --> S3
    API1 --> Queue
    Notify --> PUSH[Push Service]
    Queue --> Notify
    API1 --> Logs
    API1 --> Metrics
    Metrics --> Alerts
```

---

## 5. Component Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Mobile App** | React Native (iOS + Android) | Field rep primary interface |
| **Web App** | React SPA | Manager dashboards, admin, reporting |
| **API Gateway** | Spring Boot 3.x | REST API, auth, business logic |
| **Physician Database** | PostgreSQL 16 + PostGIS 3.4 | Physician master data, geospatial queries |
| **CRM Database** | PostgreSQL 16 | Multi-tenant CRM data (calls, tasks, pipeline) |
| **Cache** | Redis 7.x Cluster | Session management, query caching, rate limiting |
| **Data Ingestion** | Spring Batch | CMS file download, parsing, loading, geocoding |
| **Job Queue** | Amazon SQS or RabbitMQ | Async job processing (notifications, exports, ingestion) |
| **Object Storage** | Amazon S3 | Raw CMS files, CSV exports, backups |
| **CDN** | CloudFront | Static assets, mobile app bundles |
| **Load Balancer** | AWS ALB | TLS termination, health checks, routing |
| **Push Notifications** | Firebase Cloud Messaging | Task reminders, manager alerts |
| **Monitoring** | Prometheus + Grafana | Metrics, dashboards |
| **Logging** | ELK Stack or CloudWatch | Centralized log aggregation |
| **Alerting** | PagerDuty / OpsGenie | Incident notification |

---

## 6. Key Architectural Decisions

Detailed Architecture Decision Records are maintained in the `/adr` directory:

- **[ADR-001](adr/ADR-001-database.md)** — PostgreSQL + PostGIS as primary database
- **[ADR-002](adr/ADR-002-search.md)** — Search strategy (PostGIS + pg_trgm for v1, Elasticsearch for v2)
- **[ADR-003](adr/ADR-003-mobile.md)** — React Native for cross-platform mobile

---

## 7. Data Flow Overview

### 7.1 Data Ingestion Flow

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant Worker as Ingestion Worker
    participant CMS as CMS Data Source
    participant S3 as Object Storage
    participant DB as PostgreSQL
    participant GEO as Geocoding API

    Scheduler->>Worker: Trigger ingestion job
    Worker->>CMS: Download CSV file
    CMS-->>Worker: CSV data (~2-10 GB)
    Worker->>S3: Archive raw file
    Worker->>Worker: Parse, validate, transform
    Worker->>DB: Upsert records (batch)
    Worker->>Worker: Detect address changes
    Worker->>GEO: Geocode new/changed addresses
    GEO-->>Worker: lat/lng coordinates
    Worker->>DB: Update geocoded coordinates
    Worker->>DB: Update data_lineage log
```

### 7.2 Physician Profile Request Flow

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant ALB as Load Balancer
    participant API as Spring Boot API
    participant Cache as Redis Cache
    participant DB as PostgreSQL

    App->>ALB: GET /api/v1/physicians/{npi}
    ALB->>API: Forward request
    API->>Cache: Check cache (physician:{npi})
    alt Cache hit
        Cache-->>API: Cached profile
    else Cache miss
        API->>DB: Query physician + joins
        DB-->>API: Physician data
        API->>Cache: Store in cache (TTL 1hr)
    end
    API-->>ALB: 200 OK — Physician profile JSON
    ALB-->>App: Response
```

### 7.3 Nearby Physician Search Flow

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant API as Spring Boot API
    participant DB as PostgreSQL + PostGIS

    App->>API: GET /api/v1/physicians/nearby?lat=X&lng=Y&radius=10mi
    API->>DB: SELECT * FROM physicians<br>WHERE ST_DWithin(geom, point, radius)<br>AND specialty IN (...)<br>ORDER BY distance
    DB-->>API: Result set (up to 500)
    API-->>App: 200 OK — GeoJSON physician list
```

---

## 8. Security Architecture

```mermaid
graph LR
    subgraph "Client"
        App[Mobile / Web App]
    end

    subgraph "Edge"
        WAF[AWS WAF]
        ALB[ALB + TLS 1.2+]
    end

    subgraph "Auth"
        IDP[Identity Provider<br>OAuth 2.0 / SAML]
        JWT[JWT Token Validation]
    end

    subgraph "API"
        RBAC[Role-Based Access Control<br>Rep / Manager / Admin]
        TENANT[Tenant Isolation<br>org_id filtering]
        AUDIT[Audit Logger]
    end

    subgraph "Data"
        ENC[AES-256 At Rest]
        TLS[TLS In Transit]
    end

    App -->|HTTPS| WAF --> ALB --> JWT --> RBAC --> TENANT
    TENANT --> AUDIT
    RBAC --> ENC
```

| Layer | Mechanism |
|-------|-----------|
| **Transport** | TLS 1.2+ everywhere |
| **Authentication** | OAuth 2.0 / SAML 2.0 with MFA |
| **Authorization** | RBAC: Rep (own data), Manager (team), Admin (org) |
| **Tenant Isolation** | `org_id` column on all CRM tables; enforced at query layer |
| **Data at Rest** | AES-256 encryption (RDS, S3) |
| **Audit** | All admin actions + data exports logged; 2-year retention |
| **API Security** | Rate limiting, input validation, JWT expiry |

---

## 9. Infrastructure & Deployment

See [Deployment Architecture](deployment-architecture.md) for full details.

**Target Cloud:** AWS (recommended) or GCP  
**Containerization:** Docker  
**Orchestration:** ECS Fargate or EKS  
**CI/CD:** GitHub Actions  
**IaC:** Terraform

---

## 10. Scalability Considerations

| Dimension | Approach |
|-----------|----------|
| **API throughput** | Horizontal scaling behind ALB; stateless instances |
| **Database reads** | PostgreSQL read replicas for physician data queries |
| **Database writes** | CRM writes go to primary; partitioned by org_id |
| **Caching** | Redis cluster for physician profiles, search results |
| **Data ingestion** | Parallel batch workers; partitioned file processing |
| **Storage** | S3 for raw files; PG for queryable data |
| **Search at scale** | PostGIS for v1; Elasticsearch for v2 if needed |

The system is designed to handle **10,000 concurrent users** and **10 million physician records** without architecture changes.

---

*Architecture is never done. This is v1. It works. We ship it, we watch it, we fix what breaks. That's the job.*
