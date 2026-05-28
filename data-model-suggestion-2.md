# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Data Lineage & Observability · Created: 2026-05-20

## Philosophy

This model treats every piece of lineage metadata as an immutable event in an append-only store. The event store IS the lineage — there is no separate "lineage table" because lineage events are the source of truth. Materialized read models (projections) are rebuilt from the event stream to serve different query patterns: a graph projection for lineage traversal, a time-series projection for observability dashboards, and a compliance projection for regulatory reports.

This is a natural fit for data lineage because OpenLineage already emits events (START, RUNNING, COMPLETE, FAIL, ABORT) with metadata facets attached. Rather than decomposing these events into normalized tables and losing their temporal ordering, this model preserves them exactly as received and derives all queryable state from projections. The W3C PROV-DM concept of provenance as a record of "what happened" aligns perfectly with event sourcing's philosophy.

Event sourcing is used by financial trading platforms (audit trail requirements), healthcare record systems (immutable patient history), and blockchain ledgers. In the data lineage context, it provides tamper-evident audit trails essential for DORA compliance, temporal queries ("what did the lineage look like on January 15th?"), and the ability to replay events to build new analytical views without touching the source of truth.

**Best for:** Environments requiring tamper-evident audit trails (DORA, BCBS 239), temporal lineage queries, and the ability to add new analytical views without schema migrations.

**Trade-offs:**
- (+) Complete, immutable audit trail — every event ever received is preserved
- (+) Tamper-evident with cryptographic hash chains for DORA compliance
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) New read models can be added by replaying the event stream — no ETL
- (+) Schema evolution is simple: new event types coexist with old ones
- (-) Read queries hit projections, not the source of truth — eventual consistency
- (-) Projection rebuild can be slow for large event stores (millions of events)
- (-) More complex operational model: must manage event store + projection infrastructure
- (-) Ad-hoc queries on the event store require JSON parsing; structured queries need projections
- (-) Storage growth is unbounded — events are never deleted

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage (LF AI & Data) | Events are stored verbatim as received from OpenLineage producers; the event store schema wraps the OpenLineage JSON envelope |
| W3C PROV-DM | Provenance as a temporal record of Entity/Activity/Agent interactions maps directly to the event-sourced philosophy |
| DORA (EU 2022/2554) | Tamper-evident hash chain on events provides the auditable, immutable data flow documentation DORA requires |
| BCBS 239 | Full event history enables regulators to trace risk data aggregation at any historical point |
| EU AI Act | AI system data provenance built from event replay — complete training data lineage from event history |
| GDPR Article 30 | Processing activity records derived from event projections with temporal accuracy |
| ISO/IEC 25012 | Traceability quality dimension is the native property of an event-sourced system |
| JSON Schema Draft 2020-12 | Event payloads validated against OpenLineage JSON Schema before storage |

---

## Event Store (Source of Truth)

### Lineage Events

```sql
CREATE TABLE lineage_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number BIGSERIAL NOT NULL UNIQUE,    -- global ordering
    event_time TIMESTAMPTZ NOT NULL,              -- when the event occurred
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),-- when we received it
    event_type VARCHAR(32) NOT NULL,              -- LINEAGE, QUALITY, SCHEMA_CHANGE, ALERT
    source_type VARCHAR(32) NOT NULL,             -- OPENLINEAGE, MANUAL, QUALITY_TOOL, SYSTEM
    
    -- OpenLineage envelope fields (denormalized for fast filtering)
    job_namespace VARCHAR(512),
    job_name VARCHAR(512),
    run_id UUID,                                  -- OpenLineage run ID
    run_state VARCHAR(32),                        -- START, RUNNING, COMPLETE, FAIL, ABORT
    
    -- Full event payload
    payload JSONB NOT NULL,                       -- complete OpenLineage event JSON
    
    -- Tamper evidence (DORA compliance)
    payload_hash VARCHAR(64) NOT NULL,            -- SHA-256 of payload
    previous_hash VARCHAR(64),                    -- hash of previous event (chain)
    
    -- Partitioning support
    event_date DATE NOT NULL GENERATED ALWAYS AS (event_time::DATE) STORED
) PARTITION BY RANGE (event_date);

-- Create monthly partitions
CREATE TABLE lineage_events_2026_01 PARTITION OF lineage_events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE lineage_events_2026_02 PARTITION OF lineage_events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created by scheduled job

CREATE INDEX idx_events_sequence ON lineage_events(sequence_number);
CREATE INDEX idx_events_time ON lineage_events(event_time);
CREATE INDEX idx_events_type ON lineage_events(event_type);
CREATE INDEX idx_events_job ON lineage_events(job_namespace, job_name);
CREATE INDEX idx_events_run ON lineage_events(run_id);
CREATE INDEX idx_events_state ON lineage_events(run_state);
CREATE INDEX idx_events_payload ON lineage_events USING GIN (payload);
```

### Event Payload Example

```json
-- Example payload stored in lineage_events.payload:
{
  "eventType": "COMPLETE",
  "eventTime": "2026-05-20T14:30:00Z",
  "run": {
    "runId": "d46e465b-d358-4b5c-8e0c-4a26c7e1a5f2",
    "facets": {
      "nominalTime": {
        "nominalStartTime": "2026-05-20T14:00:00Z",
        "nominalEndTime": "2026-05-20T14:30:00Z"
      }
    }
  },
  "job": {
    "namespace": "airflow://prod",
    "name": "etl.transform_orders",
    "facets": {
      "sql": {"query": "INSERT INTO analytics.orders_daily SELECT ..."}
    }
  },
  "inputs": [
    {
      "namespace": "snowflake://prod.warehouse",
      "name": "raw.orders",
      "facets": {
        "schema": {
          "fields": [
            {"name": "order_id", "type": "INT8"},
            {"name": "amount", "type": "NUMERIC(12,2)"}
          ]
        }
      }
    }
  ],
  "outputs": [
    {
      "namespace": "snowflake://prod.warehouse",
      "name": "analytics.orders_daily",
      "facets": {
        "dataQualityMetrics": {
          "rowCount": 145230,
          "bytes": 28400000
        }
      }
    }
  ]
}
```

---

## Projection: Event Metadata (For Replay Tracking)

### Projection Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(128) PRIMARY KEY,     -- 'graph', 'observability', 'compliance'
    last_sequence_number BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ,
    status VARCHAR(32) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, REBUILDING, PAUSED, FAILED
    error_message TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projection: Lineage Graph (Read Model)

These tables are derived entirely from the event store and can be rebuilt by replaying events.

### Graph Nodes

```sql
CREATE TABLE graph_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace VARCHAR(512) NOT NULL,
    name VARCHAR(512) NOT NULL,
    latest_run_id UUID,
    latest_run_state VARCHAR(32),
    latest_run_time TIMESTAMPTZ,
    owner VARCHAR(256),
    description TEXT,
    source_event_id UUID NOT NULL,                -- last event that updated this row
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace, name)
);

CREATE TABLE graph_datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace VARCHAR(512) NOT NULL,
    name VARCHAR(512) NOT NULL,
    type VARCHAR(32) NOT NULL DEFAULT 'DB_TABLE',
    latest_schema JSONB,                          -- current schema facet
    latest_row_count BIGINT,
    latest_byte_count BIGINT,
    last_updated_at TIMESTAMPTZ,                  -- last time data changed
    source_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace, name)
);

CREATE INDEX idx_graph_datasets_ns ON graph_datasets(namespace);
CREATE INDEX idx_graph_datasets_name ON graph_datasets(name);

CREATE TABLE graph_dataset_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES graph_datasets(id) ON DELETE CASCADE,
    name VARCHAR(256) NOT NULL,
    type VARCHAR(128),
    ordinal_position INTEGER,
    description TEXT,
    tags TEXT[],
    UNIQUE (dataset_id, name)
);

CREATE INDEX idx_graph_fields_dataset ON graph_dataset_fields(dataset_id);
```

### Graph Edges

```sql
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_dataset_id UUID NOT NULL REFERENCES graph_datasets(id) ON DELETE CASCADE,
    target_dataset_id UUID NOT NULL REFERENCES graph_datasets(id) ON DELETE CASCADE,
    job_id UUID NOT NULL REFERENCES graph_jobs(id) ON DELETE CASCADE,
    edge_type VARCHAR(32) NOT NULL DEFAULT 'TRANSFORM',
    latest_run_id UUID,
    latest_run_time TIMESTAMPTZ,
    source_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_dataset_id, target_dataset_id, job_id)
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_dataset_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_dataset_id);
CREATE INDEX idx_graph_edges_job ON graph_edges(job_id);

CREATE TABLE graph_column_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_id UUID NOT NULL REFERENCES graph_edges(id) ON DELETE CASCADE,
    source_field_id UUID NOT NULL REFERENCES graph_dataset_fields(id) ON DELETE CASCADE,
    target_field_id UUID NOT NULL REFERENCES graph_dataset_fields(id) ON DELETE CASCADE,
    transformation_type VARCHAR(32) NOT NULL DEFAULT 'IDENTITY',
    transformation_description TEXT,
    UNIQUE (edge_id, source_field_id, target_field_id)
);

CREATE INDEX idx_col_edges_source ON graph_column_edges(source_field_id);
CREATE INDEX idx_col_edges_target ON graph_column_edges(target_field_id);
```

---

## Projection: Observability (Read Model)

### Freshness State

```sql
CREATE TABLE obs_freshness (
    dataset_id UUID PRIMARY KEY REFERENCES graph_datasets(id) ON DELETE CASCADE,
    last_arrival_time TIMESTAMPTZ,
    expected_interval_seconds INTEGER,
    is_ml_derived BOOLEAN DEFAULT false,
    consecutive_late_count INTEGER DEFAULT 0,
    status VARCHAR(16) NOT NULL DEFAULT 'OK',     -- OK, LATE, CRITICAL
    source_event_id UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_obs_freshness_status ON obs_freshness(status);
```

### Volume Baselines

```sql
CREATE TABLE obs_volume_baselines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES graph_datasets(id) ON DELETE CASCADE,
    window_start TIMESTAMPTZ NOT NULL,
    window_end TIMESTAMPTZ NOT NULL,
    row_count BIGINT,
    byte_count BIGINT,
    rolling_avg_row_count NUMERIC(18,2),
    rolling_stddev_row_count NUMERIC(18,2),
    source_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_obs_volume_dataset ON obs_volume_baselines(dataset_id);
CREATE INDEX idx_obs_volume_window ON obs_volume_baselines(window_start, window_end);
```

### Anomaly Stream

```sql
CREATE TABLE obs_anomalies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES graph_datasets(id) ON DELETE CASCADE,
    anomaly_type VARCHAR(32) NOT NULL,            -- FRESHNESS, VOLUME, SCHEMA, DISTRIBUTION
    severity VARCHAR(16) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'OPEN',
    title VARCHAR(512) NOT NULL,
    description TEXT,
    expected_value TEXT,
    actual_value TEXT,
    downstream_count INTEGER DEFAULT 0,
    assigned_to VARCHAR(256),
    resolved_at TIMESTAMPTZ,
    source_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_obs_anomalies_dataset ON obs_anomalies(dataset_id);
CREATE INDEX idx_obs_anomalies_status ON obs_anomalies(status);
CREATE INDEX idx_obs_anomalies_type ON obs_anomalies(anomaly_type);
CREATE INDEX idx_obs_anomalies_created ON obs_anomalies(created_at);
```

---

## Projection: Compliance (Read Model)

### Regulatory Lineage Snapshots

```sql
CREATE TABLE compliance_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_type VARCHAR(64) NOT NULL,           -- DORA_REPORT, AI_ACT_DOCUMENTATION, GDPR_ROPA
    snapshot_time TIMESTAMPTZ NOT NULL,
    scope_namespace VARCHAR(512),                 -- NULL = all namespaces
    content JSONB NOT NULL,                       -- structured report content
    content_hash VARCHAR(64) NOT NULL,            -- SHA-256 for tamper evidence
    generated_by VARCHAR(128) NOT NULL DEFAULT 'system',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_type ON compliance_snapshots(snapshot_type);
CREATE INDEX idx_compliance_time ON compliance_snapshots(snapshot_time);
```

### Processing Activity Projection (GDPR)

```sql
CREATE TABLE compliance_processing_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_namespace VARCHAR(512) NOT NULL,
    dataset_name VARCHAR(512) NOT NULL,
    contains_personal_data BOOLEAN DEFAULT false,
    data_categories TEXT[],
    processing_purposes TEXT[],
    legal_basis VARCHAR(128),
    retention_days INTEGER,
    cross_border_transfers TEXT[],                 -- ISO 3166 country codes
    last_updated_from_event_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_pa_ns ON compliance_processing_activities(dataset_namespace);
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    key_hash VARCHAR(128) NOT NULL,
    name VARCHAR(256),
    scopes TEXT[] NOT NULL DEFAULT '{"read"}',
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

---

## Example Queries

### Temporal lineage query: "What did the lineage look like on January 15th?"

```sql
-- Replay events up to a point in time to see historical state
SELECT
    payload->>'job'->>'namespace' AS job_namespace,
    payload->>'job'->>'name' AS job_name,
    jsonb_array_elements(payload->'inputs')->>'name' AS input_dataset,
    jsonb_array_elements(payload->'outputs')->>'name' AS output_dataset
FROM lineage_events
WHERE event_time <= '2026-01-15T23:59:59Z'
  AND event_type = 'LINEAGE'
  AND run_state = 'COMPLETE'
ORDER BY event_time DESC;
```

### Verify tamper-evidence chain

```sql
SELECT e1.event_id, e1.payload_hash, e1.previous_hash,
       e2.payload_hash AS expected_previous,
       CASE WHEN e1.previous_hash = e2.payload_hash
            THEN 'VALID' ELSE 'TAMPERED' END AS chain_status
FROM lineage_events e1
LEFT JOIN lineage_events e2
    ON e2.sequence_number = e1.sequence_number - 1
WHERE e1.sequence_number BETWEEN 1000 AND 2000
ORDER BY e1.sequence_number;
```

### Rebuild a projection from scratch

```sql
-- Step 1: Mark projection as rebuilding
UPDATE projection_checkpoints
SET status = 'REBUILDING', last_sequence_number = 0
WHERE projection_name = 'graph';

-- Step 2: Truncate projection tables
TRUNCATE graph_jobs, graph_datasets, graph_dataset_fields,
         graph_edges, graph_column_edges CASCADE;

-- Step 3: Replay events (done by application projection worker)
-- The worker reads events in sequence_number order and applies
-- each event to rebuild the projection tables.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Partitioned by month; single source of truth |
| Projection Management | 1 | Tracks replay position per projection |
| Graph Projection | 5 | Jobs, datasets, fields, table edges, column edges |
| Observability Projection | 3 | Freshness, volume baselines, anomalies |
| Compliance Projection | 2 | Snapshots and processing activities |
| Access Control | 2 | Users and API keys |
| **Total** | **14** | Plus monthly partitions for event store |

---

## Key Design Decisions

1. **Single event store as source of truth** — All OpenLineage events are stored verbatim in one partitioned table. No information is lost during ingestion. Projections are derived views that can be rebuilt.

2. **Hash chain for tamper evidence** — Each event's `previous_hash` references the prior event's `payload_hash`, creating a blockchain-like chain that DORA auditors can verify. Any tampering breaks the chain.

3. **Monthly partitioning** — The event store is partitioned by `event_date` for efficient range queries and storage management. Old partitions can be archived to cold storage without affecting current operations.

4. **Projections are disposable** — The graph, observability, and compliance projections are rebuiltable from the event stream. This means new analytical views can be added by creating a new projection worker — no schema migration needed on the event store.

5. **Denormalized filter columns on event store** — `job_namespace`, `job_name`, `run_id`, and `run_state` are extracted from the JSON payload into indexed columns for fast filtering without JSON parsing.

6. **Eventual consistency accepted** — Projections lag behind the event store by the processing delay of the projection worker. For a lineage system (not a transaction processor), this is acceptable.

7. **GIN index on payload** — Enables JSONB containment queries directly on the event store for ad-hoc investigations, though projections are preferred for routine queries.

8. **Lean table count** — Only 14 tables total (versus 34+ in the normalized model). The event store absorbs complexity that would otherwise require dedicated facet tables.
