# Component Diagram
## Medical Sales Intelligence & CRM Platform

**Author:** Frank Reynolds, DevOps & Solutions Architect  
**Date:** February 28, 2026  
**Version:** 1.0

---

## 1. Application Component Architecture

```mermaid
graph TB
    subgraph "Client Applications"
        subgraph "Mobile App (React Native)"
            MUI[UI Components]
            MState[Local State / Redux]
            MSync[Offline Sync Engine]
            MAuth[Auth Module]
            MGPS[GPS / Location Service]
            MNotify[Push Notification Handler]
            MDB[(SQLite<br>Offline Cache)]
        end

        subgraph "Web App (React SPA)"
            WUI[UI Components]
            WState[State Management]
            WAuth[Auth Module]
        end
    end

    subgraph "API Layer (Spring Boot 3.x)"
        subgraph "API Gateway"
            AuthFilter[Authentication Filter<br>JWT Validation]
            RateLimit[Rate Limiter]
            TenantCtx[Tenant Context Resolver]
        end

        subgraph "Domain Services"
            PhySvc[Physician Service]
            SearchSvc[Search Service]
            MapSvc[Map / Geospatial Service]
            CRMSvc[CRM Service]
            RouteSvc[Route Export Service]
            TerritSvc[Territory Service]
            DashSvc[Dashboard / Analytics Service]
            ExportSvc[Data Export Service]
            NotifSvc[Notification Service]
        end

        subgraph "Infrastructure Services"
            CacheMgr[Cache Manager]
            AuditSvc[Audit Logger]
            TenantSvc[Tenant Isolation Service]
        end
    end

    subgraph "Background Workers (Spring Batch)"
        subgraph "Data Ingestion Pipeline"
            Downloader[File Downloader]
            Parser[CSV Parser / Validator]
            Transformer[Data Transformer<br>Specialty Normalizer]
            Loader[Batch Upsert Loader]
            Geocoder[Geocoding Worker]
        end

        subgraph "Async Workers"
            NotifWorker[Push Notification Worker]
            ExportWorker[CSV Export Worker]
            SyncWorker[Mobile Sync Worker]
        end
    end

    MUI --> MState
    MState --> MSync
    MSync --> MDB
    MAuth --> AuthFilter
    MGPS --> MapSvc

    WUI --> WState
    WAuth --> AuthFilter

    AuthFilter --> RateLimit --> TenantCtx

    TenantCtx --> PhySvc
    TenantCtx --> SearchSvc
    TenantCtx --> MapSvc
    TenantCtx --> CRMSvc
    TenantCtx --> RouteSvc
    TenantCtx --> TerritSvc
    TenantCtx --> DashSvc
    TenantCtx --> ExportSvc

    PhySvc --> CacheMgr
    SearchSvc --> CacheMgr
    CRMSvc --> AuditSvc
    CRMSvc --> TenantSvc

    NotifSvc --> NotifWorker
    ExportSvc --> ExportWorker
```

---

## 2. Domain Service Breakdown

### 2.1 Physician Service

Owns the physician master record and all associated data aggregation.

```mermaid
graph LR
    subgraph "Physician Service"
        PC[PhysicianController<br>/api/v1/physicians]
        PS[PhysicianService]
        PR[PhysicianRepository]
        PA[PhysicianAggregator]
    end

    subgraph "Data Sources"
        NPPES[(NPPES Data)]
        CMS[(CMS Provider Data)]
        PartB[(Part B Utilization)]
        PartD[(Part D Prescribing)]
        OP[(Open Payments)]
        HA[(Hospital Affiliations)]
    end

    PC --> PS --> PA
    PA --> PR
    PR --> NPPES
    PR --> CMS
    PR --> PartB
    PR --> PartD
    PR --> OP
    PR --> HA
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/physicians/{npi}` | Full physician profile with all data sections |
| GET | `/physicians/{npi}/partb` | Part B utilization details |
| GET | `/physicians/{npi}/partd` | Part D prescribing details |
| GET | `/physicians/{npi}/payments` | Open Payments detail |
| GET | `/physicians/{npi}/affiliations` | Hospital affiliations |
| GET | `/physicians/{npi}/locations` | All practice locations |
| POST | `/physicians/{npi}/flag` | Bookmark / flag physician |

### 2.2 Search Service

Handles all search and filtering operations including geospatial queries.

```mermaid
graph LR
    subgraph "Search Service"
        SC[SearchController<br>/api/v1/search]
        SS[SearchService]
        SB[SearchQueryBuilder]
        GI[GeoIndex<br>PostGIS]
        TI[TextIndex<br>pg_trgm]
    end

    SC --> SS --> SB
    SB --> GI
    SB --> TI
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/search/physicians` | Multi-criteria physician search |
| GET | `/search/nearby` | Radius search from GPS coordinates |
| POST | `/search/segments` | Save a named search filter |
| GET | `/search/segments` | List saved search segments |
| GET | `/search/export` | Export search results to CSV |

### 2.3 CRM Service

All CRM operations — calls, tasks, opportunities, orders, samples.

```mermaid
graph LR
    subgraph "CRM Service"
        CC[CRMController<br>/api/v1/crm]
        CS[CRMService]
        CR[CRM Repositories]
        TI2[Tenant Isolation<br>org_id filter]
    end

    subgraph "CRM Entities"
        Calls[(Calls)]
        Tasks[(Tasks)]
        Opps[(Opportunities)]
        Orders[(Orders)]
        Samples[(Samples)]
    end

    CC --> CS --> TI2 --> CR
    CR --> Calls
    CR --> Tasks
    CR --> Opps
    CR --> Orders
    CR --> Samples
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/crm/calls` | Log a call/visit |
| GET | `/crm/calls?npi={npi}` | Call history for physician |
| POST | `/crm/tasks` | Create follow-up task |
| GET | `/crm/tasks?due=today` | Get tasks due today |
| PATCH | `/crm/tasks/{id}` | Update / complete task |
| POST | `/crm/opportunities` | Create opportunity |
| GET | `/crm/pipeline` | Pipeline view (kanban data) |
| POST | `/crm/orders` | Log an order |
| POST | `/crm/samples` | Log sample distribution |
| GET | `/crm/samples/budget` | Sample accountability report |

### 2.4 Route Export Service

Generates AI planning prompts and map deep-link URLs. No external API calls required.

```mermaid
graph LR
    subgraph "Route Export Service"
        RC[RouteController<br>/api/v1/routes]
        RS[RouteExportService]
        PG[PromptGenerator]
        MG[MapURLGenerator]
    end

    RC --> RS
    RS --> PG
    RS --> MG
```

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/routes/export` | Generate AI prompt + map URLs for selected stops |
| POST | `/routes/reorder` | Reorder stops with distance calculations |

### 2.5 Territory Service

Territory definitions and assignment management.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/territories` | Create territory |
| GET | `/territories/{id}/coverage` | Territory coverage report |
| POST | `/territories/{id}/assign` | Assign rep to territory |
| POST | `/territories/{id}/targets` | Set target physician list |

### 2.6 Dashboard Service

Aggregated metrics for rep and manager dashboards.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/dashboard/rep` | Rep activity dashboard |
| GET | `/dashboard/manager` | Manager team roll-up |
| GET | `/dashboard/territory` | Territory coverage metrics |

---

## 3. Data Ingestion Pipeline Components

```mermaid
graph TD
    subgraph "Ingestion Pipeline (Spring Batch)"
        SCHED[Cron Scheduler<br>Triggers by dataset schedule]
        DL[File Downloader<br>HTTP/SFTP to S3]
        VAL[Schema Validator<br>Detect format changes]
        PARSE[CSV Parser<br>Stream processing]
        NORM[Specialty Normalizer<br>NUCC → Standard]
        DEDUP[NPI Deduplicator<br>Luhn validation]
        LOAD[Batch Loader<br>COPY / upsert]
        GEO[Geocoding Worker<br>Address → lat/lng]
        META[Metadata Logger<br>Data lineage tracking]
    end

    SCHED --> DL --> VAL
    VAL -->|Schema OK| PARSE
    VAL -->|Schema Changed| ALERT[🚨 Alert Ops Team]
    PARSE --> NORM --> DEDUP --> LOAD --> GEO --> META
```

| Component | Responsibility | Key Detail |
|-----------|---------------|------------|
| File Downloader | Fetches CSV from CMS | Handles ~10 GB files; streams to S3 |
| Schema Validator | Checks column names/types | Catches CMS format changes before loading |
| CSV Parser | Stream-parses large files | Chunk processing, memory-efficient |
| Specialty Normalizer | Maps NUCC ↔ CMS ↔ Open Payments specialties | Uses mapping table (RISK-005) |
| NPI Deduplicator | Validates NPI format (Luhn), deduplicates | Quarantines invalid NPIs (NFR-031) |
| Batch Loader | Upserts to PostgreSQL | Uses COPY for bulk, upsert for incremental |
| Geocoding Worker | Geocodes new/changed addresses | Rate-limited; Google Maps or Mapbox |
| Metadata Logger | Records data lineage | Which file, when loaded, row counts |

---

## 4. Cross-Cutting Concerns

### 4.1 Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant Client as Mobile/Web
    participant API as API Gateway
    participant Auth as Auth Service
    participant IDP as Identity Provider

    Client->>Auth: POST /auth/login (credentials)
    Auth->>IDP: Validate credentials + MFA
    IDP-->>Auth: Identity token
    Auth-->>Client: JWT access token + refresh token
    
    Client->>API: GET /physicians/123 (Bearer JWT)
    API->>API: Validate JWT signature + expiry
    API->>API: Extract org_id, user_id, role
    API->>API: Apply RBAC policy
    API->>API: Apply tenant filter (org_id)
    API-->>Client: 200 OK (scoped data)
```

### 4.2 Offline Sync (Mobile)

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant SQLite as Local SQLite
    participant API as Backend API

    Note over App: Rep goes into a building — no signal
    App->>SQLite: Read cached physician profiles
    App->>SQLite: Write call log entry (queued)
    App->>SQLite: Write task update (queued)
    
    Note over App: Signal restored
    App->>API: POST /sync (queued changes)
    API-->>App: 200 OK (conflicts resolved)
    App->>API: GET /sync/pull (updated data)
    API-->>App: Delta updates since last sync
    App->>SQLite: Apply server updates
```

---

## 5. API Versioning Strategy

- **URL-based versioning:** `/api/v1/...`, `/api/v2/...`
- **Breaking changes** → new version
- **Additive changes** (new fields, new endpoints) → same version
- **Deprecation policy:** Old versions supported for 6 months after successor release
- **Mobile app compatibility:** API must support current version and one prior version simultaneously (mobile app stores have update lag)

---

*Every component has one job. If it does more than one job, it's two components pretending to be one. Split it.*
