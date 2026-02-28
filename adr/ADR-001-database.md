# ADR-001: PostgreSQL + PostGIS as Primary Database

**Date:** 2026-02-28  
**Status:** Accepted  
**Author:** Frank Reynolds, DevOps & Solutions Architect

## Context

The Medical Sales Intelligence Platform requires a database that can:

1. Store ~5 million physician records with rich multi-source data joins
2. Support geospatial queries (radius search: "show me all cardiologists within 10 miles")
3. Handle complex multi-criteria filtering across 10+ data dimensions
4. Support multi-tenant CRM data with row-level security
5. Scale to 10,000 concurrent users with sub-3-second search response times
6. Handle batch ingestion of 8-10 GB CSV files without impacting read performance

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL + PostGIS** | Mature, proven at scale; native geospatial with PostGIS; ACID compliance; rich indexing (B-tree, GIN, GiST, pg_trgm); RLS for multi-tenancy; excellent tooling; managed options (RDS, Cloud SQL) | Single-node write bottleneck; geospatial + full-text combo queries may need tuning |
| **MySQL + spatial** | Familiar; managed options | Weaker spatial support; no native pg_trgm equivalent; no RLS |
| **MongoDB** | Flexible schema; native geospatial; horizontal sharding | No ACID across collections; schema flexibility is a liability for structured CMS data; harder to do complex joins |
| **DynamoDB** | Serverless scaling; managed | Terrible for ad-hoc queries; no geospatial; no joins; would need complete redesign of query patterns |
| **Elasticsearch (primary)** | Excellent search + geo; fast faceted queries | Not a system of record; no ACID; complex to manage; data loss risk |

## Decision

**PostgreSQL 16 with PostGIS 3.4** as the primary database for both physician intelligence data and CRM data.

Key reasons:
- **PostGIS** provides production-grade geospatial indexing (`ST_DWithin`, `GIST` indexes) that handles radius queries on 5M+ points with sub-second performance
- **pg_trgm** extension provides fuzzy text search on physician names without a separate search engine
- **Row-Level Security (RLS)** provides database-enforced multi-tenant isolation for CRM data
- **Table partitioning** handles large datasets like Part B (250M rows) efficiently — partition by data_year
- **Read replicas** handle read-heavy physician queries while primary handles CRM writes
- **Managed hosting** (AWS RDS) reduces operational burden with automated backups, failover, patching
- **Cost-effective** — no per-query licensing; predictable instance-based pricing
- The team has Spring Boot + JPA + PostgreSQL experience — no ramp-up time

## Consequences

### Positive
- Single database technology to manage (reduced ops complexity)
- Strong consistency guarantees for CRM data
- Proven at this scale (5M records is well within PostgreSQL comfort zone)
- Rich ecosystem: pgAdmin, pg_dump, logical replication, etc.
- PostGIS is the industry standard for geospatial data

### Negative
- If search query patterns become very complex (10+ filter dimensions with full-text + geo), we may need Elasticsearch as a secondary search layer (see ADR-002)
- Write scaling is vertical (bigger instance) not horizontal — acceptable for our write volume
- Batch ingestion of 8 GB files requires careful transaction management to avoid table bloat; use staging tables + COPY + upsert pattern

### Risks
- Part B table at 250M rows needs partitioning from day one — cannot retrofit easily
- Geocoded data doubles index size; budget for ~100 GB of indexes on physician data
- Connection pooling (PgBouncer) needed at 10,000 concurrent users

## Follow-Up
- Implement connection pooling (PgBouncer or RDS Proxy) from launch
- Partition Part B and Part D tables by `data_year`
- Monitor query performance; if multi-criteria search exceeds 3s p95, evaluate Elasticsearch (ADR-002)
