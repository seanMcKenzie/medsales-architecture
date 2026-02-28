# MedSales — Component Diagram

**Author:** Frank Reynolds, DevOps & Solutions Architect
**Date:** 2026-02-28

---

## System Component Diagram

```mermaid
graph TB
    subgraph External Data Sources
        CMS_NPPES[("CMS NPPES<br/>NPI Registry")]
        CMS_PDC[("CMS Provider<br/>Data Catalog")]
        CMS_PARTB[("Medicare Part B<br/>Utilization")]
        CMS_PARTD[("Medicare Part D<br/>Prescribing")]
        CMS_OP[("Open Payments")]
        CMS_HOSP[("Hospital Compare")]
        CMS_FAC[("Facility<br/>Affiliations")]
    end

    subgraph Geocoding
        GEOCODE_API["Google Maps /<br/>Mapbox Geocoding API"]
    end

    subgraph Data Ingestion Layer
        ETL["Spring Batch<br/>ETL Pipeline"]
        SCHEMA_VAL["Schema Validator"]
        SPECIALTY_NORM["Specialty<br/>Normalizer"]
    end

    subgraph Application Layer
        API["Spring Boot API<br/>:8080"]
        AUTH["Spring Security<br/>JWT + RBAC"]
        CACHE["Redis<br/>:6379"]
        QUEUE["RabbitMQ<br/>:5672"]
    end

    subgraph Data Layer
        DB[("PostgreSQL 16<br/>+ PostGIS 3.4<br/>:5432")]
    end

    subgraph Client Layer
        MOBILE_IOS["iOS App"]
        MOBILE_ANDROID["Android App"]
        WEB["Web Companion<br/>(Phase 3)"]
    end

    subgraph Infrastructure
        NGINX["Nginx<br/>Reverse Proxy<br/>:80/:443"]
        PROMETHEUS["Prometheus<br/>:9090"]
        GRAFANA["Grafana<br/>:3000"]
    end

    %% Data Ingestion Flow
    CMS_NPPES --> ETL
    CMS_PDC --> ETL
    CMS_PARTB --> ETL
    CMS_PARTD --> ETL
    CMS_OP --> ETL
    CMS_HOSP --> ETL
    CMS_FAC --> ETL

    ETL --> SCHEMA_VAL
    SCHEMA_VAL --> SPECIALTY_NORM
    SPECIALTY_NORM --> DB
    ETL --> GEOCODE_API
    GEOCODE_API --> ETL

    %% Application Flow
    API --> DB
    API --> CACHE
    API --> QUEUE
    AUTH --> API
    QUEUE --> ETL

    %% Client Flow
    NGINX --> API
    MOBILE_IOS --> NGINX
    MOBILE_ANDROID --> NGINX
    WEB --> NGINX

    %% Monitoring
    API --> PROMETHEUS
    DB --> PROMETHEUS
    PROMETHEUS --> GRAFANA
```

---

## Service Interaction Diagram

```mermaid
sequenceDiagram
    participant Rep as Mobile App
    participant LB as Nginx
    participant API as Spring Boot API
    participant Cache as Redis
    participant DB as PostgreSQL + PostGIS

    Note over Rep: Rep opens app near hospital

    Rep->>LB: GET /api/physicians/nearby?lat=41.8&lng=-87.6&radius=10
    LB->>API: Forward request
    API->>Cache: Check cached results
    Cache-->>API: Cache miss
    API->>DB: SELECT ... WHERE ST_DWithin(geog, point, 16093)
    DB-->>API: 127 physicians
    API->>Cache: Store results (TTL 5min)
    API-->>LB: 200 OK — physician list
    LB-->>Rep: JSON response

    Rep->>LB: GET /api/physicians/{npi}/profile
    LB->>API: Forward request
    API->>Cache: Check cached profile
    Cache-->>API: Cache miss
    API->>DB: Join physician + CMS + Part B + Part D + Open Payments
    DB-->>API: Full profile
    API->>Cache: Store profile (TTL 15min)
    API-->>LB: 200 OK — full profile
    LB-->>Rep: JSON response

    Rep->>LB: POST /api/crm/calls
    LB->>API: Forward request
    API->>DB: INSERT call log
    DB-->>API: Saved
    API-->>LB: 201 Created
    LB-->>Rep: Confirmation
```

---

## Data Ingestion Flow

```mermaid
graph LR
    subgraph CMS Downloads
        A["Download CSV<br/>from CMS"]
    end

    subgraph Validation
        B["Schema<br/>Validation"]
        C["NPI Luhn<br/>Check"]
    end

    subgraph Transformation
        D["Parse &<br/>Normalize"]
        E["Specialty<br/>Mapping"]
        F["NPI Join &<br/>Dedup"]
    end

    subgraph Geocoding
        G["Address Change<br/>Detection"]
        H["Geocode API<br/>Call"]
    end

    subgraph Load
        I["Upsert to<br/>PostgreSQL"]
        J["Update Data<br/>Lineage Log"]
    end

    A --> B --> C --> D --> E --> F --> G
    G -->|Changed| H --> I
    G -->|Unchanged| I
    I --> J
```

---

*Diagrams render in GitHub, GitLab, and any Mermaid-compatible viewer.*
