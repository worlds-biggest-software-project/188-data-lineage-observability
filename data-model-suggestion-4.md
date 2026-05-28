# Data Model Suggestion 4: Graph-Relational (PostgreSQL + Property Graph Layer)

> Project: Data Lineage & Observability · Created: 2026-05-20

## Philosophy

This model uses a dedicated property graph layer — implemented as `graph_nodes` and `graph_edges` tables in PostgreSQL — for lineage traversal, impact analysis, and relationship discovery, while maintaining standard relational tables for operational CRUD (runs, monitors, alerts, compliance). The graph layer treats every addressable entity (job, dataset, field, dashboard, ML model) as a node and every relationship (produces, consumes, derives_from, owns, monitors) as a typed, directed edge with properties.

Data lineage is fundamentally a graph problem. The questions users ask — "What is upstream of this dashboard?", "What is affected if this table schema changes?", "Show me the path from source to this metric" — are graph traversal queries. A normalized relational model can answer these with recursive CTEs, but performance degrades as hop count and graph size increase. A graph-native model makes multi-hop traversal a first-class operation.

This design is inspired by Atlan's use of JanusGraph for 25-50M+ asset lineage graphs, DataHub's Entity-Relationship metadata graph (backed by Elasticsearch and Neo4j/PostgreSQL), and Neo4j's use in data lineage at organizations like ING and Goldman Sachs. However, rather than requiring a separate graph database (Neo4j, JanusGraph), this model implements the graph layer in PostgreSQL using a property graph pattern — making it deployable as a single-database system while preserving graph query semantics.

**Best for:** Large-scale deployments with complex, multi-hop lineage queries; environments where impact analysis and dependency discovery are the primary use cases; teams that need graph-native query performance without operating a separate graph database.

**Trade-offs:**
- (+) Multi-hop lineage traversal is the native operation, not a workaround
- (+) Any entity can be connected to any other entity — maximum relationship flexibility
- (+) Impact analysis and dependency queries are fast regardless of graph depth
- (+) New entity types and relationship types added without schema changes
- (+) Single PostgreSQL database — no separate graph DB to operate
- (-) Graph queries via recursive CTEs are less intuitive than Cypher/Gremlin
- (-) PostgreSQL graph traversal will not match Neo4j performance at very large scale (100M+ nodes)
- (-) Property graph pattern adds indirection — entity details require JOINs to node properties
- (-) More complex application code to maintain graph consistency alongside relational tables
- (-) Graph visualization requires building traversal into the API layer

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage (LF AI & Data) | Jobs, Runs, Datasets ingested and stored as graph nodes; OpenLineage relationships (input/output) become typed edges |
| W3C PROV-DM | Entity, Activity, Agent → graph node types; wasGeneratedBy, used, wasAssociatedWith → edge types; direct conceptual mapping |
| ISO/IEC 25012 | Traceability is the native property of the graph; freshness tracked via monitor nodes |
| GQL/ISO 39075 | Graph query patterns anticipate the emerging ISO GQL standard for property graph queries |
| GDPR Article 30 | Personal data flow tracking via `CONTAINS_PII` edge type and graph traversal of data flows |
| DORA (EU 2022/2554) | Impact analysis via graph traversal provides the data flow documentation DORA requires |
| EU AI Act | AI models linked to training datasets via `TRAINED_ON` edges; full provenance via graph walk |
| OpenMetadata | Entity type system inspired by OpenMetadata's 800+ metadata types |

---

## Graph Layer Tables

### Graph Nodes

```sql
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(64) NOT NULL,              -- JOB, DATASET, FIELD, DASHBOARD, ML_MODEL, USER, NAMESPACE, MONITOR
    qualified_name VARCHAR(1024) NOT NULL UNIQUE, -- globally unique identifier
        -- Examples:
        -- 'namespace://airflow-prod'
        -- 'dataset://snowflake-prod/analytics.orders_daily'
        -- 'field://snowflake-prod/analytics.orders_daily/revenue'
        -- 'job://airflow-prod/etl.transform_orders'
        -- 'dashboard://looker/exec_revenue'
        -- 'ml_model://mlflow/fraud_detector_v3'
    
    display_name VARCHAR(512),
    description TEXT,
    
    -- Typed properties stored as JSONB
    properties JSONB NOT NULL DEFAULT '{}',
    -- Dataset example:
    -- {
    --   "namespace": "snowflake://prod.warehouse",
    --   "name": "analytics.orders_daily",
    --   "type": "DB_TABLE",
    --   "source_system": "snowflake",
    --   "schema": {"fields": [...]},
    --   "row_count": 145230,
    --   "last_updated_at": "2026-05-20T14:30:00Z",
    --   "freshness_status": "OK"
    -- }
    --
    -- Job example:
    -- {
    --   "namespace": "airflow://prod",
    --   "name": "etl.transform_orders",
    --   "job_type": "BATCH",
    --   "latest_run_state": "COMPLETE",
    --   "owner": "data-eng-team",
    --   "sql": "INSERT INTO analytics.orders_daily SELECT ..."
    -- }
    
    -- Classification and tagging
    tags TEXT[],
    classifications TEXT[],                       -- PII, CONFIDENTIAL, PUBLIC, etc.
    
    -- Lifecycle
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_nodes_qname ON graph_nodes(qualified_name);
CREATE INDEX idx_nodes_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_nodes_tags ON graph_nodes USING GIN (tags);
CREATE INDEX idx_nodes_classifications ON graph_nodes USING GIN (classifications);
CREATE INDEX idx_nodes_active ON graph_nodes(is_active) WHERE is_active = true;
```

### Graph Edges

```sql
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(64) NOT NULL,
        -- Lineage edges:
        --   PRODUCES        (job → dataset)
        --   CONSUMES        (job → dataset)
        --   DERIVES_FROM    (dataset → dataset, field → field)
        --   TRANSFORMS_TO   (field → field, with transformation info)
        --
        -- Structural edges:
        --   CONTAINS_FIELD  (dataset → field)
        --   BELONGS_TO      (job/dataset → namespace)
        --   HAS_VERSION     (dataset → dataset_version node)
        --
        -- Ownership edges:
        --   OWNED_BY        (dataset/job → user)
        --   MONITORS        (monitor → dataset)
        --   ASSIGNED_TO     (alert → user)
        --
        -- Compliance edges:
        --   CONTAINS_PII    (dataset → field, with classification)
        --   GOVERNED_BY     (dataset → processing_activity)
        --   TRAINED_ON      (ml_model → dataset)
    
    -- Edge properties
    properties JSONB NOT NULL DEFAULT '{}',
    -- DERIVES_FROM example:
    -- {
    --   "transformation_type": "AGGREGATE",
    --   "transformation": "SUM(amount) GROUP BY date",
    --   "via_job": "job://airflow-prod/etl.transform_orders",
    --   "last_run_id": "d46e465b-...",
    --   "last_run_time": "2026-05-20T14:30:00Z"
    -- }
    --
    -- CONTAINS_PII example:
    -- {
    --   "pii_type": "email",
    --   "sensitivity": "high",
    --   "legal_basis": "consent"
    -- }
    
    -- Weight for ranking (e.g., frequency of data flow)
    weight NUMERIC(10,4) DEFAULT 1.0,
    
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Prevent duplicate edges of the same type between same nodes
    UNIQUE (source_node_id, target_node_id, edge_type)
);

CREATE INDEX idx_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_edges_target_type ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_edges_properties ON graph_edges USING GIN (properties);
CREATE INDEX idx_edges_active ON graph_edges(is_active) WHERE is_active = true;
```

---

## Operational Tables (Relational)

### Runs

```sql
CREATE TABLE runs (
    id UUID PRIMARY KEY,                         -- OpenLineage runId
    job_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    parent_run_id UUID REFERENCES runs(id),
    
    state VARCHAR(32) NOT NULL DEFAULT 'START',
    nominal_start_time TIMESTAMPTZ,
    nominal_end_time TIMESTAMPTZ,
    actual_start_time TIMESTAMPTZ,
    actual_end_time TIMESTAMPTZ,
    duration_ms BIGINT,
    
    -- Run facets and input/output snapshots
    facets JSONB DEFAULT '{}',
    input_datasets JSONB DEFAULT '[]',
    output_datasets JSONB DEFAULT '[]',
    
    -- Error info (denormalized for fast queries)
    error_message TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_job ON runs(job_node_id);
CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_parent ON runs(parent_run_id);
CREATE INDEX idx_runs_start ON runs(actual_start_time);
```

### Run-to-Dataset Mapping (for graph edge creation)

```sql
CREATE TABLE run_io (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id),
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    io_type VARCHAR(8) NOT NULL,                 -- INPUT, OUTPUT
    facets JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_run_io_run ON run_io(run_id);
CREATE INDEX idx_run_io_dataset ON run_io(dataset_node_id);
CREATE INDEX idx_run_io_type ON run_io(io_type);
```

---

## Observability Tables

### Monitors

```sql
CREATE TABLE monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_node_id UUID NOT NULL REFERENCES graph_nodes(id),  -- monitor node in graph
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),  -- monitored dataset
    monitor_type VARCHAR(32) NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT true,
    
    config JSONB NOT NULL DEFAULT '{}',
    baseline JSONB,
    
    last_evaluated_at TIMESTAMPTZ,
    last_status VARCHAR(16),
    consecutive_failures INTEGER DEFAULT 0,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_monitors_dataset ON monitors(dataset_node_id);
CREATE INDEX idx_monitors_type ON monitors(monitor_type);
CREATE INDEX idx_monitors_status ON monitors(last_status);
```

### Anomaly Alerts

```sql
CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_node_id UUID REFERENCES graph_nodes(id),  -- alert as a graph node (for assignment edges)
    monitor_id UUID NOT NULL REFERENCES monitors(id),
    dataset_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    
    severity VARCHAR(16) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'OPEN',
    title VARCHAR(512) NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    
    -- Impact analysis (computed via graph traversal, cached here)
    downstream_impact JSONB,
    -- Example:
    -- {
    --   "affected_count": 12,
    --   "affected_datasets": ["dataset://snowflake/analytics.revenue", ...],
    --   "affected_dashboards": ["dashboard://looker/exec_revenue"],
    --   "max_hop_distance": 4
    -- }
    
    assigned_to UUID REFERENCES users(id),
    resolved_at TIMESTAMPTZ,
    resolution_notes TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_monitor ON anomaly_alerts(monitor_id);
CREATE INDEX idx_alerts_dataset ON anomaly_alerts(dataset_node_id);
CREATE INDEX idx_alerts_status ON anomaly_alerts(status);
CREATE INDEX idx_alerts_severity ON anomaly_alerts(severity);
CREATE INDEX idx_alerts_created ON anomaly_alerts(created_at);
```

---

## Compliance Tables

### Compliance Records

```sql
CREATE TABLE compliance_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    record_type VARCHAR(64) NOT NULL,
    scope_namespace VARCHAR(512),
    data JSONB NOT NULL,
    data_hash VARCHAR(64) NOT NULL,
    status VARCHAR(32) DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_type ON compliance_records(record_type);
CREATE INDEX idx_compliance_data ON compliance_records USING GIN (data);
```

---

## Access Control

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_node_id UUID REFERENCES graph_nodes(id),  -- user as graph node
    email VARCHAR(256) NOT NULL UNIQUE,
    display_name VARCHAR(256),
    external_id VARCHAR(256),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(128) NOT NULL UNIQUE,
    permissions JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    scope_node_id UUID REFERENCES graph_nodes(id),  -- scope to namespace node
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
    entity_qualified_name VARCHAR(1024),          -- graph node qualified name
    changes JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Example Queries

### Multi-hop downstream impact analysis

```sql
-- Find all entities affected by a change to a specific dataset,
-- traversing up to 10 hops downstream
WITH RECURSIVE downstream AS (
    -- Start from the changed dataset
    SELECT
        ge.target_node_id AS node_id,
        ge.edge_type,
        1 AS depth,
        ARRAY[ge.source_node_id, ge.target_node_id] AS path
    FROM graph_edges ge
    WHERE ge.source_node_id = (
        SELECT id FROM graph_nodes
        WHERE qualified_name = 'dataset://snowflake-prod/raw.orders'
    )
    AND ge.edge_type IN ('PRODUCES', 'DERIVES_FROM', 'TRANSFORMS_TO')
    AND ge.is_active = true

    UNION ALL

    -- Traverse downstream
    SELECT
        ge.target_node_id,
        ge.edge_type,
        d.depth + 1,
        d.path || ge.target_node_id
    FROM graph_edges ge
    JOIN downstream d ON ge.source_node_id = d.node_id
    WHERE d.depth < 10
      AND ge.is_active = true
      AND ge.edge_type IN ('PRODUCES', 'DERIVES_FROM', 'TRANSFORMS_TO')
      AND NOT ge.target_node_id = ANY(d.path)  -- cycle prevention
)
SELECT
    gn.qualified_name,
    gn.node_type,
    gn.display_name,
    d.depth AS hop_distance,
    d.edge_type
FROM downstream d
JOIN graph_nodes gn ON gn.id = d.node_id
ORDER BY d.depth, gn.node_type;
```

### Find all PII data flows (GDPR compliance)

```sql
-- Trace all datasets containing PII and their downstream consumers
WITH RECURSIVE pii_flow AS (
    -- Start from fields tagged as PII
    SELECT
        ge_field.source_node_id AS dataset_id,
        ge_field.target_node_id AS field_id,
        ge_field.properties->>'pii_type' AS pii_type,
        1 AS depth,
        ARRAY[ge_field.source_node_id] AS path
    FROM graph_edges ge_field
    WHERE ge_field.edge_type = 'CONTAINS_PII'

    UNION ALL

    -- Follow data downstream
    SELECT
        ge.target_node_id,
        NULL,
        pf.pii_type,
        pf.depth + 1,
        pf.path || ge.target_node_id
    FROM graph_edges ge
    JOIN pii_flow pf ON ge.source_node_id = pf.dataset_id
    WHERE ge.edge_type IN ('DERIVES_FROM', 'PRODUCES')
      AND ge.is_active = true
      AND NOT ge.target_node_id = ANY(pf.path)
      AND pf.depth < 15
)
SELECT DISTINCT
    gn.qualified_name,
    gn.node_type,
    pf.pii_type,
    pf.depth
FROM pii_flow pf
JOIN graph_nodes gn ON gn.id = pf.dataset_id
ORDER BY pf.pii_type, pf.depth;
```

### Find shortest path between two datasets

```sql
WITH RECURSIVE path_search AS (
    SELECT
        ge.target_node_id,
        ARRAY[ge.source_node_id, ge.target_node_id] AS path,
        ARRAY[ge.edge_type] AS edge_types,
        1 AS depth
    FROM graph_edges ge
    WHERE ge.source_node_id = (SELECT id FROM graph_nodes WHERE qualified_name = 'dataset://snowflake-prod/raw.clickstream')
      AND ge.is_active = true

    UNION ALL

    SELECT
        ge.target_node_id,
        ps.path || ge.target_node_id,
        ps.edge_types || ge.edge_type,
        ps.depth + 1
    FROM graph_edges ge
    JOIN path_search ps ON ge.source_node_id = ps.target_node_id
    WHERE ps.depth < 20
      AND ge.is_active = true
      AND NOT ge.target_node_id = ANY(ps.path)
)
SELECT path, edge_types, depth
FROM path_search
WHERE target_node_id = (SELECT id FROM graph_nodes WHERE qualified_name = 'dashboard://looker/exec_revenue')
ORDER BY depth
LIMIT 1;
```

### Graph statistics

```sql
-- Node count by type
SELECT node_type, COUNT(*) AS count
FROM graph_nodes WHERE is_active = true
GROUP BY node_type ORDER BY count DESC;

-- Edge count by type
SELECT edge_type, COUNT(*) AS count
FROM graph_edges WHERE is_active = true
GROUP BY edge_type ORDER BY count DESC;

-- Most connected nodes (hubs)
SELECT gn.qualified_name, gn.node_type,
       COUNT(DISTINCT ge_out.id) AS outgoing,
       COUNT(DISTINCT ge_in.id) AS incoming,
       COUNT(DISTINCT ge_out.id) + COUNT(DISTINCT ge_in.id) AS total_connections
FROM graph_nodes gn
LEFT JOIN graph_edges ge_out ON ge_out.source_node_id = gn.id AND ge_out.is_active = true
LEFT JOIN graph_edges ge_in ON ge_in.target_node_id = gn.id AND ge_in.is_active = true
WHERE gn.is_active = true
GROUP BY gn.id, gn.qualified_name, gn.node_type
ORDER BY total_connections DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | Nodes and edges — the core of the model |
| Runs (Operational) | 2 | Run records and run I/O mapping |
| Observability | 2 | Monitors and anomaly alerts |
| Compliance | 1 | Unified compliance records |
| Access Control | 4 | Users, roles, user_roles, API tokens |
| Audit | 1 | Partitioned audit log |
| **Total** | **12** | Lowest table count of all models |

---

## Key Design Decisions

1. **Property graph in PostgreSQL** — Rather than requiring Neo4j or JanusGraph as a separate database, the graph is implemented as two PostgreSQL tables (`graph_nodes` and `graph_edges`). This keeps the deployment footprint to a single database while enabling graph traversal via recursive CTEs.

2. **Universal node types** — Jobs, datasets, fields, dashboards, ML models, users, monitors, and namespaces are all nodes in the same graph. This allows any entity to be connected to any other entity without schema changes — adding a new entity type is just a new `node_type` value.

3. **Typed, directed edges with properties** — Every edge has a type (PRODUCES, CONSUMES, DERIVES_FROM, CONTAINS_PII, TRAINED_ON, etc.) and JSONB properties. This means the graph can express not just "A is connected to B" but "A PRODUCES B with transformation SUM(amount) at time T".

4. **Graph layer for traversal, relational tables for operations** — The graph is optimized for read-heavy traversal queries. Write-heavy operational data (runs, monitors, alerts) lives in traditional relational tables that reference graph nodes by ID.

5. **Impact analysis via graph traversal** — When an anomaly is detected, the system traverses the graph downstream from the affected dataset to compute impact. The result is cached in `anomaly_alerts.downstream_impact` for fast retrieval.

6. **PII flow tracking as graph edges** — GDPR compliance becomes a graph traversal problem: tag fields with `CONTAINS_PII` edges, then traverse downstream to find all datasets that inherit PII exposure. This is more powerful than tag-based approaches because it follows actual data flow paths.

7. **Cycle prevention in queries** — All recursive CTEs include path-tracking arrays and `NOT ... = ANY(path)` checks to prevent infinite loops in cyclic graphs.

8. **Lowest table count (12)** — The property graph pattern absorbs entity and relationship complexity into just two tables, resulting in the simplest schema of all four models while supporting the most flexible relationship modeling.
