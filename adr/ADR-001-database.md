# ADR-001: PostgreSQL + PostGIS for Physician Database

**Date:** 2026-02-28
**Status:** Accepted
**Author:** Frank Reynolds

## Context

MedSales requires a primary datastore for ~5M physician records with:
- Geospatial queries (radius search, nearby, territory polygons)
- Complex joins across 10+ CMS source tables linked by NPI
- Multi-tenant CRM data with row-level isolation
- Full-text search on names and organizations
- Capacity for 10M+ records and 100M+ CRM entries
- Sub-second query performance for mobile API

Options considered:
1. **PostgreSQL + PostGIS** — Relational + geospatial in one engine
2. **MySQL + separate geo service** — Relational, limited native geo support
3. **MongoDB + geo indexes** — Document store with 2dsphere indexes
4. **PostgreSQL + Elasticsearch** — Relational for storage, ES for search

## Decision

**PostgreSQL 16 + PostGIS 3.4** as the sole primary datastore for v1.

### Rationale

1. **Geospatial is first-class.** PostGIS `ST_DWithin()` with GIST indexes on geography handles sub-second radius queries on 5M+ points. This is proven at much larger scale (OpenStreetMap runs on PostGIS).

2. **Relational model fits the data.** Physician data is inherently relational — one NPI joins to addresses, taxonomies, Part B services, Part D drugs, Open Payments, hospital affiliations. PostgreSQL handles these joins efficiently with proper indexing.

3. **Multi-tenancy via RLS.** PostgreSQL Row-Level Security provides org-level CRM data isolation without application-level filtering bugs. One policy, enforced at the database layer.

4. **Text search built in.** `pg_trgm` extension provides fuzzy name search without a separate search engine. Good enough for v1.

5. **Spring Boot ecosystem.** Spring Data JPA + Hibernate Spatial have mature PostGIS support. No custom drivers or adapters needed.

6. **Operational simplicity.** One database engine to operate, backup, monitor, and tune. Adding Elasticsearch or MongoDB doubles the operational surface for questionable v1 benefit.

7. **Proven at scale.** PostGIS handles planetary-scale geodata. 5M physician records is a rounding error.

## Consequences

### Positive
- Single datastore — simpler ops, simpler debugging, simpler backups
- PostGIS spatial performance meets all NFRs
- RLS provides strong multi-tenant isolation
- Mature Spring Boot integration
- Team familiarity — everyone knows PostgreSQL

### Negative
- Complex multi-attribute search (geo + specialty + drug + procedure + payments) may slow down with highly combinatorial queries at scale — monitor and consider Elasticsearch in v2 (see ADR-002)
- PostgreSQL `pg_trgm` is "good enough" text search, not "great" text search — no relevance scoring, no synonyms, no fuzzy phonetic matching
- Vertical scaling has limits — if we outgrow a single writer, we need read replicas or sharding (unlikely at 5M records)

### Risks
- NPPES full reload (8GB) under 4 hours requires careful batch tuning — use `COPY` command, disable indexes during load, rebuild after
- Geocode columns add storage overhead (~40 bytes/row × 5M = ~200MB — negligible)

---

*PostgreSQL does the job. It's done the job for 30 years. I'm not gonna overthink this. — Frank*
