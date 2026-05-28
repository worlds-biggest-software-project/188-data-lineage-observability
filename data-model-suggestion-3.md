# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Data Lineage & Observability · Created: 2026-05-20

## Philosophy

This model uses strongly-typed relational tables for core entities (jobs, runs, datasets, lineage edges) while storing variable metadata — OpenLineage facets, quality metrics, custom properties — in JSONB columns. It is a pragmatic middle ground: you get referential integrity and indexed queries for the structural backbone, plus schema-free extensibility for the long tail of metadata that varies by connector, facet type, and deployment.

This approach is inspired by how OpenMetadata and DataHub handle metadata extensibility in practice. OpenLineage defines a core set of facets but explicitly supports custom facets via project-namespaced prefixes. A normalized model would need a new table for every custom facet; a pure document model would lose relational integrity. The hybrid model stores core relationships relationally and attaches facets as validated JSONB, giving the best of both worlds.

PostgreSQL's JSONB support is mature: GIN indexes enable fast containment queries, partial indexes can target specific JSON paths, and `jsonb_path_query` provides SQL/JSON path language for complex extractions. This makes JSONB columns practical for production workloads, not just a prototyping convenience.

**Best for:** Rapid MVP development, multi-connector environments where facet schemas vary widely, and teams that want relational integrity without committing to a DDL change for every new metadata type.

**Trade-offs:**
- (+) Relational integrity for core entities; JSONB flexibility for metadata
- (+) No schema migrations needed when new facet types are introduced
- (+) Custom OpenLineage facets stored natively without lossy transformation
- (+) Fewer tables than fully normalized (~20 vs ~34)
- (+) PostgreSQL GIN indexes make JSONB queries performant
- (-) JSONB columns are harder to enforce type constraints on (requires application-level validation)
- (-) Complex JSONB queries can be slower than dedicated column queries
- (-) Reporting tools may struggle with JSONB extraction compared to flat columns
- (-) Risk of JSONB columns becoming dumping grounds for unstructured data
- (-) Schema documentation requires both DDL and JSON Schema specifications

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage (LF AI & Data) | Core entities (Job, Run, Dataset) are relational tables; all facets stored as validated JSONB in `run_facets`, `job_facets`, `dataset_facets` columns |
| JSON Schema Draft 2020-12 | Facet JSONB values validated against registered JSON Schemas before storage; schema registry table holds OpenLineage facet schemas |
| W3C PROV-DM | Entity/Activity/Agent mapping preserved in relational structure; PROV-JSONLD export generated from JSONB facets |
| OpenAPI 3.1 | REST API specification references JSONB field structures as OpenAPI `additionalProperties` schemas |
| ISO/IEC 25012 | Freshness and traceability dimensions modeled as relational monitor tables; quality metrics as JSONB facets |
| GDPR Article 30 | Processing metadata stored as JSONB properties on datasets with `personal_data_classification` keys |
| DORA (EU 2022/2554) | Incident records in dedicated table; lineage event JSONB preserves full provenance chain |
| EU AI Act | AI system metadata attached as JSONB properties on job and dataset entities |

---

## Facet Schema Registry

```sql
CREATE TABLE facet_schemas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facet_name VARCHAR(256) NOT NULL UNIQUE,     -- e.g. 'columnLineage', 'dataQualityMetrics'
    facet_target VARCHAR(16) NOT NULL,           -- 'run', 'job', 'dataset', 'input_dataset', 'output_dataset'
    json_schema JSONB NOT NULL,                  -- JSON Schema Draft 2020-12 for validation
    is_standard BOOLEAN NOT NULL DEFAULT false,  -- true for OpenLineage standard facets
    namespace_prefix VARCHAR(128),               -- e.g. 'my_org' for custom facets
    version VARCHAR(32) NOT NULL DEFAULT '1.0',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_facet_schemas_target ON facet_schemas(facet_target);
```

---

## Core Lineage Tables

### Namespaces

```sql
CREATE TABLE namespaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(512) NOT NULL UNIQUE,
    description TEXT,
    properties JSONB DEFAULT '{}',               -- extensible namespace metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Jobs

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id UUID NOT NULL REFERENCES namespaces(id),
    name VARCHAR(512) NOT NULL,
    description TEXT,
    job_type VARCHAR(64),                        -- 'BATCH', 'STREAMING', 'SERVICE'
    
    -- Current state (denormalized for fast reads)
    latest_run_state VARCHAR(32),
    latest_run_time TIMESTAMPTZ,
    
    -- All job facets stored as JSONB
    facets JSONB DEFAULT '{}',
    -- Example facets value:
    -- {
    --   "sql": {"query": "SELECT ..."},
    --   "sourceCode": {"language": "python", "sourceCodeUrl": "git://..."},
    --   "ownership": [{"name": "data-eng-team", "type": "RESPONSIBLE"}],
    --   "documentation": {"description": "Daily order aggregation"}
    -- }
    
    -- Custom properties (non-OpenLineage metadata)
    properties JSONB DEFAULT '{}',
    -- Example: {"team": "analytics", "cost_center": "CC-1234", "sla_tier": "gold"}
    
    tags TEXT[],
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, name)
);

CREATE INDEX idx_jobs_namespace ON jobs(namespace_id);
CREATE INDEX idx_jobs_name ON jobs(name);
CREATE INDEX idx_jobs_facets ON jobs USING GIN (facets);
CREATE INDEX idx_jobs_properties ON jobs USING GIN (properties);
CREATE INDEX idx_jobs_tags ON jobs USING GIN (tags);
```

### Datasets

```sql
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id UUID NOT NULL REFERENCES namespaces(id),
    name VARCHAR(512) NOT NULL,
    type VARCHAR(32) NOT NULL DEFAULT 'DB_TABLE',  -- DB_TABLE, STREAM, FILE
    description TEXT,
    physical_name VARCHAR(512),
    source_system VARCHAR(256),
    
    -- Current schema (denormalized for fast access)
    current_schema JSONB,
    -- Example:
    -- {
    --   "fields": [
    --     {"name": "order_id", "type": "INT8", "description": "Primary key"},
    --     {"name": "amount", "type": "NUMERIC(12,2)", "nullable": true}
    --   ]
    -- }
    
    -- All dataset facets
    facets JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "dataQualityMetrics": {"rowCount": 145230, "bytes": 28400000},
    --   "storage": {"storageLayer": "iceberg", "fileFormat": "parquet"},
    --   "columnLineage": {...}
    -- }
    
    -- Custom properties
    properties JSONB DEFAULT '{}',
    -- Example: {"pii_classification": "confidential", "retention_days": 365}
    
    tags TEXT[],
    
    -- Observability state (denormalized)
    last_updated_at TIMESTAMPTZ,
    last_row_count BIGINT,
    freshness_status VARCHAR(16) DEFAULT 'UNKNOWN',  -- OK, LATE, CRITICAL, UNKNOWN
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, name)
);

CREATE INDEX idx_datasets_namespace ON datasets(namespace_id);
CREATE INDEX idx_datasets_name ON datasets(name);
CREATE INDEX idx_datasets_type ON datasets(type);
CREATE INDEX idx_datasets_facets ON datasets USING GIN (facets);
CREATE INDEX idx_datasets_properties ON datasets USING GIN (properties);
CREATE INDEX idx_datasets_tags ON datasets USING GIN (tags);
CREATE INDEX idx_datasets_freshness ON datasets(freshness_status);
```

### Dataset Fields

```sql
CREATE TABLE dataset_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    name VARCHAR(256) NOT NULL,
    type VARCHAR(128),
    ordinal_position INTEGER,
    description TEXT,
    is_nullable BOOLEAN DEFAULT true,
    is_primary_key BOOLEAN DEFAULT false,
    tags TEXT[],
    
    -- Field-level metadata as JSONB
    properties JSONB DEFAULT '{}',
    -- Example: {"pii_type": "email", "sensitivity": "high", "masking_rule": "hash"}
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dataset_id, name)
);

CREATE INDEX idx_fields_dataset ON dataset_fields(dataset_id);
CREATE INDEX idx_fields_properties ON dataset_fields USING GIN (properties);
```

---

## Runs and Run History

```sql
CREATE TABLE runs (
    id UUID PRIMARY KEY,                         -- OpenLineage runId (client-provided)
    job_id UUID NOT NULL REFERENCES jobs(id),
    parent_run_id UUID REFERENCES runs(id),
    
    state VARCHAR(32) NOT NULL DEFAULT 'START',
    nominal_start_time TIMESTAMPTZ,
    nominal_end_time TIMESTAMPTZ,
    actual_start_time TIMESTAMPTZ,
    actual_end_time TIMESTAMPTZ,
    duration_ms BIGINT GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (actual_end_time - actual_start_time)) * 1000
    ) STORED,
    
    -- All run facets as JSONB
    facets JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "nominalTime": {"nominalStartTime": "...", "nominalEndTime": "..."},
    --   "errorMessage": {"message": "Table not found", "programmingLanguage": "sql"},
    --   "externalQuery": {"externalQueryId": "01abc...", "source": "snowflake"},
    --   "parent": {"run": {"runId": "..."}, "job": {"namespace": "...", "name": "..."}}
    -- }
    
    -- Input/output dataset snapshots
    input_datasets JSONB DEFAULT '[]',
    -- Example: [{"namespace": "snowflake://prod", "name": "raw.orders", "facets": {...}}]
    
    output_datasets JSONB DEFAULT '[]',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_job ON runs(job_id);
CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_parent ON runs(parent_run_id);
CREATE INDEX idx_runs_nominal_start ON runs(nominal_start_time);
CREATE INDEX idx_runs_actual_start ON runs(actual_start_time);
CREATE INDEX idx_runs_facets ON runs USING GIN (facets);
```

---

## Lineage Edges

### Table-Level Lineage

```sql
CREATE TABLE lineage_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_dataset_id UUID NOT NULL REFERENCES datasets(id),
    target_dataset_id UUID NOT NULL REFERENCES datasets(id),
    job_id UUID NOT NULL REFERENCES jobs(id),
    edge_type VARCHAR(32) NOT NULL DEFAULT 'TRANSFORM',
    
    -- Latest run info (denormalized)
    latest_run_id UUID REFERENCES runs(id),
    latest_run_time TIMESTAMPTZ,
    latest_run_state VARCHAR(32),
    
    -- Edge metadata
    properties JSONB DEFAULT '{}',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_dataset_id, target_dataset_id, job_id)
);

CREATE INDEX idx_lineage_source ON lineage_edges(source_dataset_id);
CREATE INDEX idx_lineage_target ON lineage_edges(target_dataset_id);
CREATE INDEX idx_lineage_job ON lineage_edges(job_id);
```

### Column-Level Lineage

```sql
CREATE TABLE column_lineage_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lineage_edge_id UUID NOT NULL REFERENCES lineage_edges(id) ON DELETE CASCADE,
    source_field_id UUID NOT NULL REFERENCES dataset_fields(id),
    target_field_id UUID NOT NULL REFERENCES dataset_fields(id),
    transformation_type VARCHAR(32) NOT NULL DEFAULT 'IDENTITY',
    transformation_description TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (lineage_edge_id, source_field_id, target_field_id)
);

CREATE INDEX idx_col_lineage_source ON column_lineage_edges(source_field_id);
CREATE INDEX idx_col_lineage_target ON column_lineage_edges(target_field_id);
```

---

## Observability Tables

### Monitors (Unified)

```sql
CREATE TABLE monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    monitor_type VARCHAR(32) NOT NULL,            -- FRESHNESS, VOLUME, SCHEMA, DISTRIBUTION, CUSTOM
    enabled BOOLEAN NOT NULL DEFAULT true,
    is_ml_derived BOOLEAN NOT NULL DEFAULT false,
    
    -- Configuration as JSONB (varies by monitor type)
    config JSONB NOT NULL DEFAULT '{}',
    -- Freshness example: {"expected_interval_seconds": 3600, "tolerance_seconds": 300}
    -- Volume example: {"min_rows": 1000, "max_rows": 500000, "baseline_model": "rolling_avg"}
    -- Distribution example: {"columns": ["amount"], "test": "ks_test", "threshold": 0.05}
    -- Custom example: {"query": "SELECT COUNT(*) FROM ... WHERE ...", "expected": "> 0"}
    
    -- ML baseline data
    baseline JSONB,
    -- Example: {"mean": 145230, "stddev": 12400, "p95": 168000, "window_days": 30}
    
    last_evaluated_at TIMESTAMPTZ,
    last_status VARCHAR(16),                      -- OK, WARNING, CRITICAL
    consecutive_failures INTEGER DEFAULT 0,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_monitors_dataset ON monitors(dataset_id);
CREATE INDEX idx_monitors_type ON monitors(monitor_type);
CREATE INDEX idx_monitors_status ON monitors(last_status);
```

### Anomaly Alerts

```sql
CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_id UUID NOT NULL REFERENCES monitors(id),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    severity VARCHAR(16) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'OPEN',
    title VARCHAR(512) NOT NULL,
    
    -- Alert details as JSONB (flexible per monitor type)
    details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "expected": "145230 +/- 12400 rows",
    --   "actual": "3200 rows",
    --   "deviation_sigma": 11.4,
    --   "downstream_affected": [
    --     {"dataset": "analytics.revenue_daily", "hops": 2},
    --     {"dataset": "dashboards.exec_revenue", "hops": 3}
    --   ]
    -- }
    
    assigned_to UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    resolution_notes TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_monitor ON anomaly_alerts(monitor_id);
CREATE INDEX idx_alerts_dataset ON anomaly_alerts(dataset_id);
CREATE INDEX idx_alerts_status ON anomaly_alerts(status);
CREATE INDEX idx_alerts_severity ON anomaly_alerts(severity);
CREATE INDEX idx_alerts_created ON anomaly_alerts(created_at);
```

### Schema Change History

```sql
CREATE TABLE schema_changes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Change details as JSONB
    changes JSONB NOT NULL,
    -- Example:
    -- [
    --   {"type": "COLUMN_ADDED", "field": "discount_pct", "new_type": "NUMERIC(5,2)"},
    --   {"type": "TYPE_CHANGED", "field": "amount", "old_type": "INT4", "new_type": "INT8"},
    --   {"type": "COLUMN_REMOVED", "field": "legacy_code"}
    -- ]
    
    old_schema_hash VARCHAR(64),
    new_schema_hash VARCHAR(64) NOT NULL,
    triggered_by_run_id UUID REFERENCES runs(id)
);

CREATE INDEX idx_schema_changes_dataset ON schema_changes(dataset_id);
CREATE INDEX idx_schema_changes_detected ON schema_changes(detected_at);
```

---

## Compliance

### Compliance Metadata (JSONB-based)

```sql
CREATE TABLE compliance_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    record_type VARCHAR(64) NOT NULL,              -- GDPR_ROPA, DORA_INCIDENT, AI_ACT_REGISTRATION
    scope_namespace VARCHAR(512),
    
    -- All compliance data as JSONB (varies by regulation)
    data JSONB NOT NULL,
    -- GDPR_ROPA example:
    -- {
    --   "processing_purpose": "Order fulfillment",
    --   "legal_basis": "contract",
    --   "data_categories": ["customer_name", "email", "order_history"],
    --   "datasets": ["raw.customers", "raw.orders"],
    --   "retention_days": 730,
    --   "cross_border": ["DE", "US"]
    -- }
    --
    -- DORA_INCIDENT example:
    -- {
    --   "incident_type": "DATA_BREACH",
    --   "severity": "HIGH",
    --   "affected_datasets": ["..."],
    --   "root_cause_run_id": "...",
    --   "detection_time": "2026-05-20T10:00:00Z",
    --   "report_hash": "sha256:..."
    -- }
    
    data_hash VARCHAR(64) NOT NULL,               -- SHA-256 for tamper evidence
    status VARCHAR(32) DEFAULT 'ACTIVE',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_type ON compliance_records(record_type);
CREATE INDEX idx_compliance_status ON compliance_records(status);
CREATE INDEX idx_compliance_data ON compliance_records USING GIN (data);
```

---

## Access Control

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(256) NOT NULL UNIQUE,
    display_name VARCHAR(256),
    external_id VARCHAR(256),
    is_active BOOLEAN NOT NULL DEFAULT true,
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,
    permissions JSONB NOT NULL DEFAULT '[]',
    -- Example: ["lineage:read", "lineage:write", "monitors:manage", "compliance:export"]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    scope_namespace_id UUID REFERENCES namespaces(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    token_hash VARCHAR(128) NOT NULL,
    name VARCHAR(256),
    scopes TEXT[],
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(64) NOT NULL,
    entity_type VARCHAR(64) NOT NULL,
    entity_id UUID,
    changes JSONB,                               -- {"field": {"old": "...", "new": "..."}}
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Example Queries

### Find all datasets with PII classification

```sql
SELECT d.name, d.namespace_id,
       d.properties->>'pii_classification' AS classification
FROM datasets d
WHERE d.properties @> '{"pii_classification": "confidential"}'::jsonb;
```

### Extract quality metrics from dataset facets

```sql
SELECT d.name,
       d.facets->'dataQualityMetrics'->>'rowCount' AS row_count,
       d.facets->'dataQualityMetrics'->>'bytes' AS byte_count,
       d.last_updated_at
FROM datasets d
WHERE d.facets ? 'dataQualityMetrics'
ORDER BY d.last_updated_at DESC;
```

### Find runs with error facets containing a specific message

```sql
SELECT r.id, j.name AS job_name,
       r.facets->'errorMessage'->>'message' AS error,
       r.actual_start_time
FROM runs r
JOIN jobs j ON j.id = r.job_id
WHERE r.state = 'FAIL'
  AND r.facets->'errorMessage'->>'message' ILIKE '%permission denied%'
ORDER BY r.actual_start_time DESC
LIMIT 50;
```

### Upstream lineage traversal (recursive CTE)

```sql
WITH RECURSIVE upstream AS (
    SELECT le.source_dataset_id, le.target_dataset_id,
           le.job_id, 1 AS depth
    FROM lineage_edges le
    WHERE le.target_dataset_id = '...'  -- starting dataset

    UNION ALL

    SELECT le.source_dataset_id, le.target_dataset_id,
           le.job_id, u.depth + 1
    FROM lineage_edges le
    JOIN upstream u ON le.target_dataset_id = u.source_dataset_id
    WHERE u.depth < 20
)
SELECT d.name, d.namespace_id, u.depth
FROM upstream u
JOIN datasets d ON d.id = u.source_dataset_id
ORDER BY u.depth;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Registry | 1 | Facet JSON Schema validation |
| Core Lineage | 5 | Namespaces, jobs, datasets, fields, runs |
| Lineage Edges | 2 | Table-level and column-level |
| Observability | 3 | Monitors, alerts, schema changes |
| Compliance | 1 | Unified compliance records with typed JSONB |
| Access Control | 4 | Users, roles, user_roles, API tokens |
| Audit | 1 | Partitioned audit log |
| **Total** | **17** | |

---

## Key Design Decisions

1. **JSONB facets instead of facet tables** — All OpenLineage facets (standard and custom) are stored in a single `facets` JSONB column on the parent entity. This eliminates the need for DDL changes when new facet types appear and directly stores the OpenLineage JSON structure.

2. **Facet schema registry for validation** — While JSONB is schema-free at the database level, the `facet_schemas` table holds JSON Schema definitions that the application layer uses to validate incoming facets before storage. This prevents the JSONB columns from becoming unstructured dumping grounds.

3. **Relational edges, JSONB metadata** — Lineage edges are relational (with foreign keys to datasets and jobs) because graph traversal queries need indexed join columns. But edge metadata is JSONB for flexibility.

4. **Denormalized observability state** — `freshness_status`, `last_row_count`, and `last_updated_at` are denormalized onto the `datasets` table for fast dashboard queries. The authoritative values live in the monitors and run facets.

5. **Unified monitors table** — Instead of separate tables for freshness, volume, and schema monitors, a single `monitors` table uses `monitor_type` discrimination with JSONB `config` and `baseline`. This makes it trivial to add new monitor types.

6. **Unified compliance records** — Instead of separate tables per regulation (GDPR, DORA, AI Act), a single `compliance_records` table with `record_type` discrimination stores all compliance data as validated JSONB. This simplifies the schema while supporting any future regulatory requirement.

7. **GIN indexes on all JSONB columns** — Every JSONB column has a GIN index enabling fast containment queries (`@>`) and key-existence checks (`?`).

8. **17 tables total** — Roughly half the count of the fully normalized model, with the same functional coverage. The trade-off is that some queries require JSON path extraction rather than simple column references.
