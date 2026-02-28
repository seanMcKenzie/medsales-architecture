# MedSales — Data Architecture

**Author:** Frank Reynolds, DevOps & Solutions Architect
**Date:** 2026-02-28

---

## 1. Data Ingestion Pipeline

### 1.1 Source Datasets

| Dataset | Source URL | Format | Frequency | Size | Join Key |
|---|---|---|---|---|---|
| NPPES NPI Registry | download.cms.gov/nppes | CSV | Monthly full + weekly delta | 8–10 GB | NPI |
| Provider Data Catalog (National Downloadable File) | data.cms.gov/provider-data | CSV | Quarterly | ~500 MB | NPI, org_pac_id |
| Facility Affiliation Data | data.cms.gov/provider-data | CSV | Quarterly | ~200 MB | NPI, CCN |
| Hospital Compare — General Info | data.cms.gov/provider-data | CSV | Quarterly | ~5 MB | CCN |
| Medicare Part B — By Provider & Service | data.cms.gov | CSV | Annual | 2–3 GB | NPI |
| Medicare Part D — By Provider & Drug | data.cms.gov | CSV | Annual | ~3 GB | NPI |
| Open Payments — General | download.cms.gov/openpayments | CSV | Annual | ~2 GB | NPI |
| Open Payments — Research | download.cms.gov/openpayments | CSV | Annual | ~500 MB | NPI |
| Open Payments — Ownership | download.cms.gov/openpayments | CSV | Annual | ~50 MB | NPI |

### 1.2 Pipeline Architecture

```
Spring Batch Job (per dataset)
├── Step 1: Download
│   ├── HTTP GET from CMS
│   ├── Checksum validation
│   └── Store raw file in staging area
├── Step 2: Schema Validation
│   ├── Compare column headers to expected schema
│   ├── Alert + abort on unexpected changes (RISK-007)
│   └── Log validation result
├── Step 3: Parse & Transform
│   ├── CSV parsing (chunked, 10K rows/batch)
│   ├── NPI Luhn checksum validation
│   ├── Data type casting and null handling
│   ├── Specialty normalization (NUCC → canonical)
│   └── Quarantine invalid records
├── Step 4: Geocode (address datasets only)
│   ├── Detect address changes vs existing records
│   ├── Batch geocode changed addresses
│   ├── Rate limiting (50 req/sec for Google)
│   └── Cache geocode results
├── Step 5: Load
│   ├── UPSERT into PostgreSQL (ON CONFLICT NPI)
│   ├── Maintain audit columns (source_updated_at)
│   └── Update data lineage log
└── Step 6: Post-Load
    ├── VACUUM ANALYZE affected tables
    ├── Refresh materialized views
    └── Emit completion event to RabbitMQ
```

### 1.3 Ingestion Schedule

| Job | Schedule | Window |
|---|---|---|
| NPPES Full | 1st Sunday of month, 2:00 AM CT | ~4 hours |
| NPPES Weekly Delta | Every Sunday 2:00 AM CT | ~30 min |
| Provider Data Catalog | Quarterly, within 48h of CMS publish | ~1 hour |
| Facility Affiliations | Quarterly, with Provider Data Catalog | ~30 min |
| Hospital Compare | Quarterly, with Provider Data Catalog | ~10 min |
| Part B Utilization | Annual, within 72h of CMS publish | ~2 hours |
| Part D Prescribing | Annual, within 72h of CMS publish | ~2 hours |
| Open Payments | Annual, within 72h of CMS publish | ~1.5 hours |

---

## 2. Database Schema Overview

### 2.1 Schema Organization

```
medsales (database)
├── public schema — Physician intelligence data (shared, read-heavy)
│   ├── physician                  — Master record (NPI PK)
│   ├── physician_address          — Practice/mailing locations + geocodes
│   ├── physician_taxonomy         — Specialties and licenses
│   ├── physician_cms_profile      — CMS Provider Data Catalog fields
│   ├── hospital                   — Hospital master (CCN PK)
│   ├── hospital_affiliation       — Physician ↔ Hospital links
│   ├── part_b_service             — Medicare Part B utilization
│   ├── part_d_drug                — Medicare Part D prescribing
│   ├── open_payments_general      — Industry payments
│   ├── open_payments_research     — Research payments
│   ├── open_payments_ownership    — Ownership interests
│   ├── specialty_crosswalk        — NUCC ↔ CMS ↔ OpenPayments mapping
│   └── data_lineage               — Source dataset ingestion log
│
├── crm schema — CRM data (multi-tenant, org-isolated)
│   ├── organization               — Tenant record
│   ├── crm_user                   — User accounts
│   ├── crm_call                   — Call/visit logs
│   ├── crm_task                   — Follow-up tasks
│   ├── crm_opportunity            — Pipeline records
│   ├── crm_order                  — Order records
│   ├── crm_sample                 — Sample distribution logs
│   ├── product                    — Product catalog
│   ├── territory                  — Territory definitions
│   ├── territory_assignment       — Rep ↔ territory links
│   ├── physician_bookmark         — Flagged/bookmarked physicians
│   └── saved_segment              — Saved search filters
│
└── audit schema — Audit and security logs
    ├── admin_action_log           — Admin actions
    ├── data_export_log            — Export events
    └── auth_event_log             — Login/logout/MFA events
```

### 2.2 Key Indexes

```sql
-- Geospatial (PostGIS)
CREATE INDEX idx_physician_address_geog
    ON physician_address USING GIST (
        ST_SetSRID(ST_MakePoint(geo_lng, geo_lat), 4326)::geography
    );

CREATE INDEX idx_hospital_geog
    ON hospital USING GIST (
        ST_SetSRID(ST_MakePoint(geo_lng, geo_lat), 4326)::geography
    );

-- Text search (pg_trgm)
CREATE INDEX idx_physician_name_trgm
    ON physician USING GIN (
        (last_name || ' ' || first_name) gin_trgm_ops
    );

-- NPI lookups (all child tables)
CREATE INDEX idx_part_b_npi ON part_b_service (npi);
CREATE INDEX idx_part_d_npi ON part_d_drug (npi);
CREATE INDEX idx_open_payments_npi ON open_payments_general (npi);
CREATE INDEX idx_hospital_aff_npi ON hospital_affiliation (npi);
CREATE INDEX idx_physician_addr_npi ON physician_address (npi);

-- CRM queries (org-scoped)
CREATE INDEX idx_crm_call_org_npi ON crm.crm_call (org_id, npi);
CREATE INDEX idx_crm_call_org_user_date ON crm.crm_call (org_id, user_id, call_date DESC);
CREATE INDEX idx_crm_task_org_due ON crm.crm_task (org_id, user_id, due_date) WHERE NOT completed;

-- Part B/D filtering
CREATE INDEX idx_part_b_hcpcs ON part_b_service (hcpcs_code, total_services DESC);
CREATE INDEX idx_part_d_drug ON part_d_drug (generic_name, total_claims DESC);
```

### 2.3 Row-Level Security (Multi-Tenant CRM)

```sql
ALTER TABLE crm.crm_call ENABLE ROW LEVEL SECURITY;

CREATE POLICY crm_call_org_isolation ON crm.crm_call
    USING (org_id = current_setting('app.current_org_id')::UUID);
```

Applied to all CRM tables. The Spring Boot API sets `app.current_org_id` on each connection from the authenticated user's JWT.

---

## 3. Search Infrastructure

### 3.1 v1: PostgreSQL-Native

| Query Type | Implementation |
|---|---|
| Radius search (nearby physicians) | PostGIS `ST_DWithin()` with GIST index on geography |
| Specialty filter | B-tree index on normalized specialty column |
| Name search | `pg_trgm` GIN index, `similarity()` or `ILIKE` |
| Part B procedure filter | Composite index on `(hcpcs_code, total_services)` |
| Part D drug filter | Composite index on `(generic_name, total_claims)` |
| Open Payments filter | Indexes on `manufacturer_name`, `payment_amount` |
| Combined filters | Query planner combines indexes; use CTEs for complex combos |

**Expected performance:** Sub-second for single-filter queries on 5M records. 1–3s for complex multi-filter with radius. Meets NFR-002.

### 3.2 v2 Consideration: Elasticsearch

If v1 search performance degrades with complex multi-attribute queries at scale, add Elasticsearch:
- Sync physician data to ES via CDC (Change Data Capture) or scheduled refresh
- ES handles full-text + faceted + geo queries
- PostgreSQL remains source of truth
- See ADR-002 for full analysis

---

## 4. Geocoding

### 4.1 Strategy

- **Initial load:** ~5M physician addresses + ~7K hospitals
- **Incremental:** ~200K address changes/month (NPPES weekly deltas)
- **Provider:** Google Maps Geocoding API (primary) or Mapbox (cost alternative)
- **Budget:** ~$25K initial + ~$1K/month ongoing (Google standard rates)

### 4.2 Implementation

```
Address Change Detection:
1. Hash current address fields (line1 + city + state + zip)
2. Compare to stored hash on physician_address record
3. If changed → queue for geocoding
4. If new record → queue for geocoding
5. If unchanged → skip

Geocoding Process:
1. Batch addresses in groups of 100
2. Rate limit: 50 requests/second (Google limit)
3. Store lat/lng + geocode_confidence + geocode_timestamp
4. Failed geocodes → retry queue (3 attempts, exponential backoff)
5. Permanently failed → flag for manual review
```

### 4.3 Geocode Storage

```sql
-- On physician_address table
geo_lat          DECIMAL(10, 7)    -- latitude
geo_lng          DECIMAL(10, 7)    -- longitude
geo_confidence   VARCHAR(20)       -- ROOFTOP, RANGE_INTERPOLATED, etc.
geo_hash         VARCHAR(64)       -- SHA-256 of input address
geo_updated_at   TIMESTAMP         -- when geocoded
```

---

## 5. Data Quality & Reconciliation

### 5.1 NPI Validation
- Luhn checksum on all NPIs before insert
- Invalid NPIs quarantined in `data_quarantine` table
- Dashboard alert for quarantine count > threshold

### 5.2 Cross-Source Reconciliation
- NPPES address vs Provider Data Catalog address: flag discrepancies
- Precedence order: Provider Data Catalog > NPPES (Provider Data Catalog is more current for practice location)
- Open Payments NPI matching: 3–5% records may not join; fuzzy name+state fallback with low-confidence flag

### 5.3 Specialty Normalization
```
specialty_crosswalk table:
├── nucc_code          — NPPES taxonomy code (e.g., "207RC0000X")
├── nucc_desc          — NUCC description ("Cardiovascular Disease")
├── cms_specialty      — CMS plain-English ("Cardiology")
├── op_category        — Open Payments category ("Allopathic & Osteopathic Physicians")
├── op_specialty       — Open Payments specialty ("Cardiology")
├── canonical_name     — Our normalized name ("Cardiology")
└── canonical_group    — Grouping ("Internal Medicine Subspecialty")
```

### 5.4 Data Freshness Display
Every profile section shows: `Data source: [SOURCE] | Last updated: [DATE] | Data year: [YEAR]`

---

*The data is the product. If the data's wrong, everything's wrong. — Frank*
