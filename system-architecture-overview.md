# MedSales — System Architecture Overview

**Author:** Frank Reynolds, DevOps & Solutions Architect
**Date:** 2026-02-28
**Version:** 1.0

---

## 1. Overview

MedSales is a mobile-first Medical Sales Intelligence & CRM platform that aggregates publicly available CMS physician data (~5M records) and serves it to pharmaceutical and medical device sales reps through a real-time, location-aware mobile app with integrated CRM capabilities.

Here's the deal: we pull free federal data, enrich it, geocode it, shove it into a spatial database, and serve it through a Spring Boot API to native mobile apps. No commercial data licensing. No $500K/year IQVIA contracts. Just public CMS data, good engineering, and a rep who can see every doctor within 10 miles before they walk in the door.

---

## 2. Major Components

### 2.1 Data Ingestion Pipeline
- **Batch ETL jobs** that pull CSV datasets from CMS (NPPES, Provider Data Catalog, Part B, Part D, Open Payments, Hospital Compare, Facility Affiliations)
- Scheduled via Spring Batch or standalone ETL service
- Monthly/quarterly/annual cadence depending on source
- NPI-keyed deduplication and cross-source reconciliation
- Specialty normalization across three different naming conventions (NUCC, CMS plain-English, Open Payments pipe-delimited)
- Schema validation to catch CMS format changes before they blow up prod (see RISK-007)

### 2.2 Geocoding Service
- Processes ~5M addresses on initial load, ~200K/month incremental
- Google Maps Geocoding API or Mapbox (budget-dependent)
- Cache-first: only re-geocode when address changes
- Populates `geo_lat`/`geo_lng` on PhysicianAddress and Hospital records

### 2.3 PostgreSQL + PostGIS Database
- **Primary datastore** for all physician, hospital, utilization, prescribing, and payments data
- PostGIS extension for geospatial queries (`ST_DWithin()`, geography indexes)
- Handles radius search, nearby physician queries, territory polygon containment
- Separate schema for multi-tenant CRM data (org-level isolation)
- `pg_trgm` extension for fuzzy text search on names and organizations

### 2.4 Spring Boot API
- RESTful API serving all mobile and web clients
- Endpoints for: physician profiles, search/filter, map data, CRM CRUD, dashboards, admin
- JWT-based authentication with MFA support
- RBAC: Rep, Manager, Admin roles
- Horizontal scaling behind a load balancer
- Rate limiting and API key support for integrations

### 2.5 Mobile Applications (iOS + Android)
- Native or cross-platform (see ADR-003)
- GPS-aware: nearby physician map, distance calculations
- Offline mode: cached profiles, offline call logging, sync on reconnect
- Push notifications for task reminders
- Route export: AI prompt generation + Google Maps / Apple Maps deep links

### 2.6 Web Companion (Phase 3)
- Manager-focused dashboard
- Territory management, team roll-ups, reporting
- React or similar SPA consuming the same Spring Boot API

---

## 3. Tech Stack

| Layer | Technology |
|---|---|
| **Backend API** | Spring Boot 3.x (Java 21) |
| **Database** | PostgreSQL 16 + PostGIS 3.4 |
| **Search** | PostGIS spatial indexes + pg_trgm (v1); Elasticsearch considered for v2 |
| **Batch Processing** | Spring Batch |
| **Cache** | Redis |
| **Message Queue** | RabbitMQ or Kafka (for async ingestion events) |
| **Mobile** | React Native (recommended — see ADR-003) |
| **Geocoding** | Google Maps Geocoding API / Mapbox |
| **Auth** | Spring Security + JWT; SAML 2.0/OAuth 2.0 SSO (Phase 3) |
| **Containerization** | Docker + Docker Compose |
| **CI/CD** | GitHub Actions |
| **Monitoring** | Prometheus + Grafana |
| **Reverse Proxy** | Nginx |

---

## 4. Data Flow

```
CMS Data Sources (NPPES, Provider Data, Part B/D, Open Payments, Hospital Compare)
    │
    ▼
┌─────────────────────────────────────┐
│  Data Ingestion Pipeline            │
│  (Spring Batch ETL)                 │
│  - Download CSVs                    │
│  - Validate schema                  │
│  - Parse, normalize, deduplicate    │
│  - NPI-keyed joins                  │
│  - Specialty normalization          │
│  - Geocode new/changed addresses    │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  PostgreSQL + PostGIS               │
│  - Physician master records (5M+)   │
│  - Hospital records                 │
│  - Part B/D utilization data        │
│  - Open Payments                    │
│  - CRM data (multi-tenant)          │
│  - Geospatial indexes               │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Spring Boot API                    │
│  - REST endpoints                   │
│  - Auth (JWT + RBAC)                │
│  - Redis cache layer                │
│  - Business logic                   │
└──────┬──────────────────┬───────────┘
       │                  │
       ▼                  ▼
┌──────────────┐  ┌──────────────────┐
│  Mobile App  │  │  Web Companion   │
│  (iOS/Droid) │  │  (Phase 3)       │
│  - GPS map   │  │  - Manager dash  │
│  - Profiles  │  │  - Reporting     │
│  - CRM       │  │  - Territory mgmt│
│  - Offline   │  │                  │
└──────────────┘  └──────────────────┘
```

---

## 5. Key Design Decisions

1. **PostgreSQL + PostGIS over a separate search engine** — For v1, PostGIS handles spatial queries on 5M records with sub-second performance. No need to add Elasticsearch complexity yet. (See ADR-001, ADR-002)

2. **NPI as canonical key** — Every CMS dataset joins on NPI. It's the spine of the entire data model. One physician, one NPI, all data linked.

3. **Multi-tenant CRM isolation** — CRM data is org-scoped. Every CRM table carries an `org_id`. Row-level security in Postgres enforces isolation.

4. **Batch ingestion, not streaming** — CMS publishes bulk files on fixed schedules (monthly, quarterly, annually). There's no real-time feed. Batch ETL is the right pattern.

5. **Mobile-first, API-first** — The API serves both mobile and web. Mobile is the primary consumer. The API is the product.

---

## 6. Non-Functional Targets

- **Profile load:** < 2s on 4G
- **Radius search:** < 3s for 50-mile radius
- **Map render:** < 2s for 500 pins
- **Concurrent users:** 10,000+
- **Uptime:** 99.9%
- **RPO:** 1 hour / **RTO:** 4 hours
- **NPPES full ingestion:** < 4 hours for ~8GB file
- **CRM capacity:** 100M+ call log entries

---

*Built by Frank. It'll work.*
