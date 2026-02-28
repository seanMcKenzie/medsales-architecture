# ADR-002: Search Infrastructure — PostGIS vs Elasticsearch

**Date:** 2026-02-28
**Status:** Accepted
**Author:** Frank Reynolds

## Context

MedSales search requirements include:
- **Geographic radius search** (physicians within X miles of a point)
- **Specialty filtering** (single or multi-select)
- **Name search** (fuzzy match on physician/org names)
- **Procedure volume filtering** (Part B HCPCS code + minimum threshold)
- **Drug prescribing filtering** (Part D drug name + minimum claims)
- **Open Payments filtering** (by company, amount, payment nature)
- **Combined multi-attribute queries** (geo + specialty + drug + payments simultaneously)
- **Sorting** by distance, billing volume, prescribing volume, payments

Target: 5M physician records, <3 second response for radius queries, 10K concurrent users.

Options:
1. **PostGIS only** — PostgreSQL spatial + `pg_trgm` for all search
2. **Elasticsearch only** — All data synced to ES, queries go through ES
3. **PostGIS + Elasticsearch hybrid** — PostgreSQL for storage/CRM, ES for search

## Decision

**PostGIS only for v1. Evaluate Elasticsearch for v2 if performance degrades under production load.**

### Why PostGIS for v1

| Criterion | PostGIS | Elasticsearch |
|---|---|---|
| Radius search on 5M points | ✅ Sub-second with GIST index | ✅ Sub-second with geo_point |
| Multi-table joins (NPI key) | ✅ Native SQL joins | ❌ Requires denormalization |
| Fuzzy name search | ⚠️ pg_trgm (adequate) | ✅ Excellent (analyzers, fuzzy) |
| Combined geo + attribute filter | ⚠️ Complex queries, adequate perf | ✅ Designed for this |
| Operational complexity | ✅ Already running PostgreSQL | ❌ Additional cluster to manage |
| Data consistency | ✅ Single source of truth | ❌ Sync lag, eventual consistency |
| Spring Boot integration | ✅ JPA + Hibernate Spatial | ✅ Spring Data Elasticsearch |
| v1 timeline impact | ✅ No additional setup | ❌ 2–4 weeks additional work |

**The deciding factors:**

1. PostGIS handles every v1 query pattern. The question isn't "can it?" — it's "will it at 10K concurrent users with complex filters?" For v1 launch traffic, the answer is yes.

2. Adding Elasticsearch for v1 means syncing 5M records + 10 child tables into a denormalized ES index, building a sync pipeline, handling consistency, and operating an ES cluster. That's a month of work for marginal v1 benefit.

3. We can add ES later without rearchitecting. PostgreSQL remains the source of truth regardless. ES is an additive optimization, not a foundational change.

### When to Add Elasticsearch (v2 Triggers)

Add Elasticsearch when any of these occur:
- P95 search latency exceeds 3s for combined geo + multi-attribute queries
- Users report slow search performance consistently
- Faceted search / aggregation requirements emerge (e.g., "show me specialty distribution in this radius")
- Full-text search quality complaints (synonyms, phonetic matching, relevance ranking)

### ES v2 Architecture (if triggered)

```
PostgreSQL (source of truth)
    │
    ├── CDC (Debezium) or scheduled sync
    │
    ▼
Elasticsearch Cluster
    ├── physician_search index (denormalized)
    │   ├── NPI, name, specialty, location (geo_point)
    │   ├── Part B top procedures (nested)
    │   ├── Part D top drugs (nested)
    │   ├── Open Payments summary (nested)
    │   └── CRM status (last contact, priority)
    │
    └── Search API queries ES
        └── Detail/CRM queries still hit PostgreSQL
```

## Consequences

### Positive
- Faster v1 delivery — no ES cluster to build and maintain
- Simpler architecture — one data engine
- No sync consistency issues
- Lower infrastructure cost for v1

### Negative
- Complex combined searches may be slower than ES equivalent
- No faceted aggregations (count by specialty in radius)
- Text search quality limited to `pg_trgm` capabilities
- If ES is needed for v2, it's a significant engineering effort (denormalization + sync pipeline)

---

*Look — I'm not adding another cluster to manage unless I have to. PostGIS handles it. If it doesn't, we add Elasticsearch later. That's the scheme. — Frank*
