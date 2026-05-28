# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Data Lineage & Observability · Created: 2026-05-20

## Philosophy

This model follows a fully normalized relational design where every OpenLineage concept — Job, Run, Dataset, Facet — maps to a dedicated table with explicit foreign key relationships. The schema mirrors the OpenLineage specification closely, making it the most standards-aligned option. Every facet type gets its own table rather than being stored as JSON, enabling strong type checking and SQL-native querying across all metadata dimensions.

This approach is modeled after how Marquez structures its PostgreSQL backend, but extends it with dedicated tables for observability (freshness monitors, anomaly alerts, SLA tracking) and compliance (DORA incident records, EU AI Act documentation). The W3C PROV-DM concepts of Entity, Activity, and Agent map cleanly onto Dataset, Run, and Job/Owner respectively.

Real-world systems using this pattern include Marquez (OpenLineage reference backend), Alation's internal catalog schema, and traditional data catalog platforms like Apache Atlas. It is best suited for teams that prioritize query flexibility, referential integrity, and standards compliance over write throughput.

**Best for:** Regulated environments (DORA, EU AI Act) where query flexibility, referential integrity, and audit compliance are more important than write throughput.

**Trade-offs:**
- (+) Full referential integrity — broken relationships are impossible
- (+) Standard SQL queries for all lineage traversal and analytics
- (+) Column-level lineage is first-class, not buried in JSON
- (+) Easy to extend with new tables for new entity types
- (-) High table count (~45+) increases schema management complexity
- (-) Many JOINs required for common queries (run + all facets)
- (-) Schema migrations needed for every new facet type — less flexible than JSONB
- (-) Write amplification: a single OpenLineage event fans out to many INSERT statements
- (-) Facet extensibility requires DDL changes, not just new JSON keys

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage (LF AI & Data) | Direct 1:1 mapping: `jobs`, `runs`, `datasets` tables mirror OpenLineage Job, Run, Dataset entities; each standard facet gets a dedicated table |
| W3C PROV-DM | Entity→Dataset, Activity→Run, Agent→Job/Owner conceptual alignment; `was_derived_from` modeled via `dataset_lineage_edges` |
| ISO/IEC 25012 | Freshness (currentness) and traceability quality dimensions modeled as `freshness_monitors` and `lineage_edges` tables |
| ISO 3166-1/2 | `jurisdictions` reference table for DORA/GDPR data residency tracking |
| GDPR Article 30 | `processing_activities` table for records of processing; linked to datasets via foreign keys |
| DORA (EU 2022/2554) | `dora_incidents` table for structured incident reporting with tamper-evident hashing |
| EU AI Act | `ai_system_registrations` table linking AI models to their training dataset lineage |
| BCBS 239 | Risk data aggregation lineage tracked via standard `lineage_edges` with `regulatory_scope` classification |
| RFC 6749 / OpenID Connect | `api_tokens` and `oauth_clients` tables for authentication |
| JSON Schema Draft 2020-12 | Facet validation schemas stored in `facet_type_registry` |

---

## Core Lineage Tables

### Namespaces

```sql
CREATE TABLE namespaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(512) NOT NULL UNIQUE,       -- e.g. 'postgres://prod-db:5432'
    description TEXT,
    owner_id UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_namespaces_name ON namespaces(name);
```

### Jobs

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id UUID NOT NULL REFERENCES namespaces(id),
    name VARCHAR(512) NOT NULL,              -- e.g. 'etl.transform_orders'
    description TEXT,
    current_version_id UUID,                 -- points to latest job_versions row
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, name)
);

CREATE INDEX idx_jobs_namespace ON jobs(namespace_id);
CREATE INDEX idx_jobs_name ON jobs(name);
```

### Job Versions

```sql
CREATE TABLE job_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(id),
    version_hash VARCHAR(64) NOT NULL,       -- SHA-256 of job definition
    source_code_location TEXT,               -- e.g. 'git://repo/path/to/dag.py#L42'
    source_code_hash VARCHAR(64),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_versions_job ON job_versions(job_id);
```

### Runs

```sql
CREATE TABLE runs (
    id UUID PRIMARY KEY,                     -- OpenLineage runId (client-provided UUID)
    job_version_id UUID NOT NULL REFERENCES job_versions(id),
    parent_run_id UUID REFERENCES runs(id),  -- OpenLineage ParentRunFacet
    nominal_start_time TIMESTAMPTZ,          -- scheduled start (NominalTimeRunFacet)
    nominal_end_time TIMESTAMPTZ,
    actual_start_time TIMESTAMPTZ,
    actual_end_time TIMESTAMPTZ,
    state VARCHAR(32) NOT NULL DEFAULT 'START',  -- START, RUNNING, COMPLETE, FAIL, ABORT
    external_id VARCHAR(512),                -- e.g. Airflow dag_run_id
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_job_version ON runs(job_version_id);
CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_parent ON runs(parent_run_id);
CREATE INDEX idx_runs_nominal_start ON runs(nominal_start_time);
CREATE INDEX idx_runs_actual_start ON runs(actual_start_time);
```

### Datasets

```sql
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id UUID NOT NULL REFERENCES namespaces(id),
    name VARCHAR(512) NOT NULL,              -- e.g. 'public.orders'
    type VARCHAR(32) NOT NULL DEFAULT 'DB_TABLE',  -- DB_TABLE, STREAM, FILE
    description TEXT,
    current_version_id UUID,
    physical_name VARCHAR(512),              -- actual table/file name if different
    source_system VARCHAR(256),              -- e.g. 'snowflake', 'bigquery'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace_id, name)
);

CREATE INDEX idx_datasets_namespace ON datasets(namespace_id);
CREATE INDEX idx_datasets_name ON datasets(name);
CREATE INDEX idx_datasets_type ON datasets(type);
```

### Dataset Versions

```sql
CREATE TABLE dataset_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    version_hash VARCHAR(64) NOT NULL,
    run_id UUID REFERENCES runs(id),         -- the run that produced this version
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dataset_versions_dataset ON dataset_versions(dataset_id);
CREATE INDEX idx_dataset_versions_run ON dataset_versions(run_id);
```

### Dataset Fields (Column-Level Metadata)

```sql
CREATE TABLE dataset_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    name VARCHAR(256) NOT NULL,              -- column name
    type VARCHAR(128),                       -- e.g. 'VARCHAR(255)', 'INT8', 'JSONB'
    ordinal_position INTEGER,
    description TEXT,
    is_nullable BOOLEAN DEFAULT true,
    is_primary_key BOOLEAN DEFAULT false,
    tags TEXT[],                             -- classification tags
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dataset_fields_version ON dataset_fields(dataset_version_id);
CREATE INDEX idx_dataset_fields_name ON dataset_fields(name);
```

---

## Lineage Edge Tables

### Table-Level Lineage

```sql
CREATE TABLE lineage_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id),
    input_dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    output_dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    edge_type VARCHAR(32) NOT NULL DEFAULT 'TRANSFORM',  -- TRANSFORM, COPY, AGGREGATE, JOIN
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_edges_run ON lineage_edges(run_id);
CREATE INDEX idx_lineage_edges_input ON lineage_edges(input_dataset_version_id);
CREATE INDEX idx_lineage_edges_output ON lineage_edges(output_dataset_version_id);
```

### Column-Level Lineage

```sql
CREATE TABLE column_lineage_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lineage_edge_id UUID NOT NULL REFERENCES lineage_edges(id),
    input_field_id UUID NOT NULL REFERENCES dataset_fields(id),
    output_field_id UUID NOT NULL REFERENCES dataset_fields(id),
    transformation_type VARCHAR(32) NOT NULL DEFAULT 'IDENTITY',
        -- IDENTITY, RENAME, TRANSFORM, AGGREGATE, CONDITIONAL
    transformation_description TEXT,         -- e.g. 'COALESCE(a.name, b.name)'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_col_lineage_input ON column_lineage_edges(input_field_id);
CREATE INDEX idx_col_lineage_output ON column_lineage_edges(output_field_id);
CREATE INDEX idx_col_lineage_edge ON column_lineage_edges(lineage_edge_id);
```

---

## Facet Tables

### Run Facets

```sql
CREATE TABLE run_facet_nominal_time (
    run_id UUID PRIMARY KEY REFERENCES runs(id),
    nominal_start_time TIMESTAMPTZ NOT NULL,
    nominal_end_time TIMESTAMPTZ
);

CREATE TABLE run_facet_error_message (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id),
    message TEXT NOT NULL,
    programming_language VARCHAR(64),
    stack_trace TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_run_error_run ON run_facet_error_message(run_id);

CREATE TABLE run_facet_external_query (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id),
    external_query_id VARCHAR(512) NOT NULL,
    source VARCHAR(256) NOT NULL,            -- e.g. 'snowflake', 'bigquery'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_run_ext_query_run ON run_facet_external_query(run_id);
```

### Dataset Facets

```sql
CREATE TABLE dataset_facet_schema (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    schema_hash VARCHAR(64) NOT NULL,        -- hash of full schema for change detection
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dataset_facet_data_quality (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    row_count BIGINT,
    byte_count BIGINT,
    null_count JSONB,                        -- {"column_name": null_count, ...}
    distinct_count JSONB,
    min_values JSONB,
    max_values JSONB,
    quantiles JSONB,
    source_tool VARCHAR(128),               -- 'great_expectations', 'soda', 'dbt_test'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dq_dataset_version ON dataset_facet_data_quality(dataset_version_id);

CREATE TABLE dataset_facet_storage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    storage_layer VARCHAR(128),              -- 'iceberg', 'delta', 'hudi', 'parquet'
    file_format VARCHAR(64),
    compression VARCHAR(64),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Job Facets

```sql
CREATE TABLE job_facet_sql (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_version_id UUID NOT NULL REFERENCES job_versions(id),
    query TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_facet_source_code (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_version_id UUID NOT NULL REFERENCES job_versions(id),
    language VARCHAR(64),
    source_code TEXT,
    source_code_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_facet_ownership (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_version_id UUID NOT NULL REFERENCES job_versions(id),
    owner_id UUID REFERENCES users(id),
    ownership_type VARCHAR(64) NOT NULL DEFAULT 'RESPONSIBLE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_ownership_owner ON job_facet_ownership(owner_id);
```

---

## Observability Tables

### Freshness Monitors

```sql
CREATE TABLE freshness_monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    expected_interval_seconds INTEGER NOT NULL,  -- ML-derived or manual
    tolerance_seconds INTEGER NOT NULL DEFAULT 0,
    is_ml_derived BOOLEAN NOT NULL DEFAULT false,
    enabled BOOLEAN NOT NULL DEFAULT true,
    last_evaluated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_freshness_dataset ON freshness_monitors(dataset_id);
```

### Volume Monitors

```sql
CREATE TABLE volume_monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    expected_row_count_min BIGINT,
    expected_row_count_max BIGINT,
    baseline_model VARCHAR(64) DEFAULT 'rolling_avg',  -- rolling_avg, prophet, static
    is_ml_derived BOOLEAN NOT NULL DEFAULT false,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_volume_dataset ON volume_monitors(dataset_id);
```

### Schema Change Tracking

```sql
CREATE TABLE schema_changes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    old_version_id UUID REFERENCES dataset_versions(id),
    new_version_id UUID NOT NULL REFERENCES dataset_versions(id),
    change_type VARCHAR(32) NOT NULL,        -- COLUMN_ADDED, COLUMN_REMOVED, TYPE_CHANGED, COLUMN_RENAMED
    field_name VARCHAR(256) NOT NULL,
    old_type VARCHAR(128),
    new_type VARCHAR(128),
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schema_changes_dataset ON schema_changes(dataset_id);
CREATE INDEX idx_schema_changes_detected ON schema_changes(detected_at);
```

### Anomaly Alerts

```sql
CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    monitor_type VARCHAR(32) NOT NULL,       -- FRESHNESS, VOLUME, SCHEMA, DISTRIBUTION
    severity VARCHAR(16) NOT NULL DEFAULT 'MEDIUM',  -- LOW, MEDIUM, HIGH, CRITICAL
    status VARCHAR(32) NOT NULL DEFAULT 'OPEN',      -- OPEN, ACKNOWLEDGED, RESOLVED, FALSE_POSITIVE
    title VARCHAR(512) NOT NULL,
    description TEXT,
    expected_value TEXT,
    actual_value TEXT,
    downstream_impact_count INTEGER DEFAULT 0,  -- count of affected downstream datasets
    assigned_to UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_dataset ON anomaly_alerts(dataset_id);
CREATE INDEX idx_anomaly_status ON anomaly_alerts(status);
CREATE INDEX idx_anomaly_severity ON anomaly_alerts(severity);
CREATE INDEX idx_anomaly_created ON anomaly_alerts(created_at);
```

### Downstream Impact Cache

```sql
CREATE TABLE downstream_impact (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_dataset_id UUID NOT NULL REFERENCES datasets(id),
    affected_dataset_id UUID NOT NULL REFERENCES datasets(id),
    hop_count INTEGER NOT NULL,              -- distance in lineage graph
    path_ids UUID[] NOT NULL,                -- ordered array of dataset IDs in path
    last_computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_impact_source ON downstream_impact(source_dataset_id);
CREATE INDEX idx_impact_affected ON downstream_impact(affected_dataset_id);
```

---

## Compliance Tables

### Processing Activities (GDPR Article 30)

```sql
CREATE TABLE processing_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(512) NOT NULL,
    purpose TEXT NOT NULL,
    legal_basis VARCHAR(128) NOT NULL,       -- 'consent', 'legitimate_interest', 'legal_obligation', etc.
    data_categories TEXT[] NOT NULL,          -- ['personal_data', 'sensitive_data', 'financial']
    retention_period_days INTEGER,
    jurisdiction_code VARCHAR(8),            -- ISO 3166-1 alpha-2
    controller_name VARCHAR(256),
    processor_name VARCHAR(256),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE processing_activity_datasets (
    processing_activity_id UUID NOT NULL REFERENCES processing_activities(id),
    dataset_id UUID NOT NULL REFERENCES datasets(id),
    role VARCHAR(32) NOT NULL DEFAULT 'INPUT',  -- INPUT, OUTPUT, STORAGE
    PRIMARY KEY (processing_activity_id, dataset_id)
);
```

### DORA Incident Records

```sql
CREATE TABLE dora_incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_type VARCHAR(64) NOT NULL,      -- DATA_BREACH, SERVICE_DISRUPTION, DATA_LOSS
    severity VARCHAR(16) NOT NULL,
    description TEXT NOT NULL,
    affected_datasets UUID[] NOT NULL,       -- array of dataset IDs
    root_cause_run_id UUID REFERENCES runs(id),
    detection_time TIMESTAMPTZ NOT NULL,
    resolution_time TIMESTAMPTZ,
    report_hash VARCHAR(64) NOT NULL,        -- SHA-256 for tamper evidence
    reported_to_authority BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dora_type ON dora_incidents(incident_type);
CREATE INDEX idx_dora_detection ON dora_incidents(detection_time);
```

### AI System Registration (EU AI Act)

```sql
CREATE TABLE ai_system_registrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    system_name VARCHAR(512) NOT NULL,
    risk_category VARCHAR(32) NOT NULL,      -- MINIMAL, LIMITED, HIGH, UNACCEPTABLE
    purpose TEXT NOT NULL,
    provider_name VARCHAR(256),
    training_dataset_ids UUID[],             -- references to datasets used for training
    validation_dataset_ids UUID[],
    documentation_url TEXT,
    registered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_reg_risk ON ai_system_registrations(risk_category);
```

---

## Access Control Tables

### Users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(256) NOT NULL UNIQUE,
    display_name VARCHAR(256),
    external_id VARCHAR(256),                -- IdP subject ID
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,       -- 'admin', 'editor', 'viewer', 'compliance_officer'
    description TEXT,
    permissions JSONB NOT NULL DEFAULT '[]'
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    namespace_id UUID REFERENCES namespaces(id),  -- NULL = global role
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, namespace_id)
);

CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    token_hash VARCHAR(128) NOT NULL,        -- bcrypt hash of bearer token
    name VARCHAR(256),
    scopes TEXT[] NOT NULL DEFAULT '{"read"}',
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_tokens_user ON api_tokens(user_id);
CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(64) NOT NULL,             -- CREATE, UPDATE, DELETE, QUERY, LOGIN
    entity_type VARCHAR(64) NOT NULL,        -- 'dataset', 'job', 'run', 'monitor', etc.
    entity_id UUID,
    details JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Example Queries

### Trace column lineage upstream (recursive CTE)

```sql
WITH RECURSIVE upstream AS (
    -- Start from the target column
    SELECT cle.input_field_id, cle.output_field_id,
           cle.transformation_type, 1 AS depth
    FROM column_lineage_edges cle
    JOIN dataset_fields df ON df.id = cle.output_field_id
    WHERE df.name = 'revenue'
      AND df.dataset_version_id = (
          SELECT current_version_id FROM datasets WHERE name = 'analytics.dashboard_revenue'
      )

    UNION ALL

    -- Walk upstream
    SELECT cle.input_field_id, cle.output_field_id,
           cle.transformation_type, u.depth + 1
    FROM column_lineage_edges cle
    JOIN upstream u ON cle.output_field_id = u.input_field_id
    WHERE u.depth < 20  -- prevent infinite loops
)
SELECT df_in.name AS source_column,
       d_in.name AS source_dataset,
       u.transformation_type,
       u.depth
FROM upstream u
JOIN dataset_fields df_in ON df_in.id = u.input_field_id
JOIN dataset_versions dv ON dv.id = df_in.dataset_version_id
JOIN datasets d_in ON d_in.id = dv.dataset_id
ORDER BY u.depth;
```

### Find all datasets affected by a schema change

```sql
SELECT di.affected_dataset_id, d.name, d.namespace_id, di.hop_count
FROM downstream_impact di
JOIN datasets d ON d.id = di.affected_dataset_id
WHERE di.source_dataset_id = '...'  -- the changed dataset
ORDER BY di.hop_count;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Lineage (namespaces, jobs, runs, datasets) | 8 | Including versions tables |
| Lineage Edges (table + column level) | 2 | Separate tables for table and column lineage |
| Dataset Fields | 1 | Column-level metadata |
| Facets (run, dataset, job) | 8 | One table per standard facet type |
| Observability (monitors, alerts, schema changes) | 5 | Freshness, volume, schema, anomalies, impact |
| Compliance (GDPR, DORA, AI Act) | 5 | Processing activities, incidents, AI registration |
| Access Control (users, roles, tokens) | 4 | RBAC with namespace scoping |
| Audit | 1 | Append-only audit log |
| **Total** | **34** | |

---

## Key Design Decisions

1. **OpenLineage entity fidelity** — Jobs, Runs, and Datasets are first-class tables with the same semantics as the OpenLineage specification, ensuring ingested events map without lossy transformation.

2. **Dedicated facet tables instead of JSONB** — Each standard OpenLineage facet (nominal time, error message, SQL, data quality) gets its own table. This enables type-safe queries and indexing at the cost of requiring DDL changes for new facet types.

3. **Versioned entities** — Both Jobs and Datasets maintain version histories via `job_versions` and `dataset_versions` tables, enabling temporal queries ("what was the schema on date X?") without event replay.

4. **Separate table-level and column-level lineage edges** — Column lineage edges reference the parent table-level edge, allowing queries at either granularity without self-joins.

5. **Pre-computed downstream impact** — The `downstream_impact` table is a materialized cache of graph traversal results, trading storage for query speed on impact analysis.

6. **UUID primary keys throughout** — OpenLineage uses UUIDs for run IDs (client-generated); all other entities follow suit for consistency and distributed-system compatibility.

7. **Compliance as first-class tables** — GDPR processing activities, DORA incidents, and EU AI Act registrations are dedicated tables rather than metadata tags, making regulatory reporting queryable with standard SQL.

8. **Namespace-scoped RBAC** — Roles can be scoped to a specific namespace (e.g., a team's data domain) or global, supporting multi-tenant deployments.
