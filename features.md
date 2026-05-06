# Data Lineage & Observability — Feature & Functionality Survey

> Candidate #188 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Monte Carlo | Commercial SaaS | Proprietary / Custom enterprise pricing | https://www.montecarlodata.com |
| Atlan | Commercial SaaS | Proprietary / From ~$20k/yr | https://atlan.com |
| Alation | Commercial SaaS | Proprietary / ~$120/user/month + platform | https://www.alation.com |
| Sifflet | Commercial SaaS | Proprietary / Custom | https://www.siffletdata.com |
| Collibra | Commercial SaaS | Proprietary / Custom enterprise | https://www.collibra.com |
| DataHub | Open-source + Cloud | Apache 2.0 / Acryl Data managed cloud | https://datahubproject.io |
| OpenMetadata | Open-source + SaaS | Apache 2.0 / Collate managed cloud | https://open-metadata.org |
| OpenLineage | Open-source standard | Apache 2.0 | https://openlineage.io |
| Marquez | Open-source | Apache 2.0 | https://marquezproject.ai |
| dbt (Cloud + Core) | Open-source + SaaS | Apache 2.0 (Core) / dbt Cloud from $100/seat/month | https://www.getdbt.com |
| Apache Atlas | Open-source | Apache 2.0 | https://atlas.apache.org |
| Egeria | Open-source standard | Apache 2.0 | https://egeria-project.org |

---

## Feature Analysis by Solution

### Monte Carlo

**Core features**
- Five-pillar observability: freshness, volume, schema, distribution, and lineage monitoring
- ML-driven automated anomaly detection with no threshold configuration required
- Field-level (column-level) lineage from source to downstream reports and dashboards
- Root cause analysis linking anomalies back to specific jobs, tables, or schema changes
- Data downtime alerting with incident management and ownership assignment
- Circuit-breaker pipeline controls to halt jobs on quality breach
- Data quality monitoring dashboards and SLA tracking
- 250+ native integrations with warehouses, pipelines, and BI tools
- Incident timeline and collaboration tooling (Slack, Jira, PagerDuty)
- AI observability extending monitoring to ML model inputs and output drift

**Differentiating features**
- "Data downtime" framing as a reliability metric analogous to system uptime
- Automatic baseline learning per dataset without manual rule definition
- Anomaly correlation across freshness, volume, schema, and distribution simultaneously
- API-first design: full GraphQL API, Python SDK (pycarlo), and CLI powering all features

**UX patterns**
- Dashboard-centric onboarding; guided connector setup reduces time-to-first-alert
- Incident feed is the main operational surface; lineage graph is accessed from incidents
- Progressive disclosure: table-level lineage shown first, field-level on drill-down

**Integration points**
- GraphQL API (all UI features are API-backed); REST-style webhook delivery
- Python SDK (pycarlo) for programmatic automation
- Native integrations: Snowflake, BigQuery, Redshift, Databricks, dbt, Airflow, Looker, Tableau
- Atlan integration surfaces Monte Carlo incidents inside Atlan catalogue
- AWS, Azure, GCP marketplace listings

**Known gaps**
- Complex column-level lineage (JSON unpacking, lateral joins) requires additional manual config
- Enterprise-only pricing excludes SMBs and mid-market teams
- No on-premises deployment; cloud-only
- Users report limited customisation of alerting templates; basic data models that limit some workflows

**Licence / IP notes**
- Fully proprietary; no source-available component. All integrations are via documented API only.

---

### Atlan

**Core features**
- Active metadata platform built on JanusGraph; handles 25–50M+ asset lineage graphs
- Real-time lineage discovery that scans and refreshes as transformations change
- Deep integrations with Snowflake, Databricks, BigQuery, dbt, Looker, Tableau
- Collaborative governance: annotations, tagging, cross-team discussions on lineage maps
- Business glossary surfaced within lineage context
- Data quality indicator overlays from partner tools (Monte Carlo, Soda, etc.)
- Metadata orchestration — propagates changes across connected systems
- AI-assisted asset description generation and rule suggestion
- Role-based access control and policy enforcement on metadata

**Differentiating features**
- Embedded collaboration: lineage and metadata surface in Slack, BI tools, query editors without context-switching
- Open, interoperable metadata lakehouse architecture allowing custom entity types
- "Context layer for AI" positioning — tracks AI models, agents, and training datasets as first-class assets

**UX patterns**
- Workspace-oriented: each team gets a collaborative governance workspace
- Metadata appears where engineers already work (Slack, Looker, dbt) via deep integrations
- Search-centric discovery; lineage accessed from asset profiles

**Integration points**
- REST and GraphQL APIs; Python SDK with type-safe entities
- 100+ connectors via the Atlan connector framework
- OpenLineage-compatible ingestion pipeline
- Webhook event system for metadata change propagation
- Monte Carlo, Soda, Great Expectations quality integrations

**Known gaps**
- High entry price excludes smaller teams
- Relies on partner tools (Monte Carlo, Soda) for anomaly detection; not native
- Some users report complex initial setup and connector configuration time

**Licence / IP notes**
- Fully proprietary SaaS; open-source connector contributions accepted under CLA.

---

### Alation

**Core features**
- Data catalog with automatic metadata harvesting from 120+ connectors
- SQL query log analysis and natural-language parsing for table- and column-level lineage inference
- Open Data Quality Framework — standardised API for ingesting quality signals from external tools
- Behavioral metadata: combines usage, lineage, and quality to prioritise alerts
- Business glossary and policy management with stewardship workflows
- Conversation and annotation layer within the catalog
- DORA compliance reporting dashboards for regulated industries
- Agentic data intelligence mode for automated governance task execution (2026)

**Differentiating features**
- Open Data Quality Framework lets any quality tool push results in; creates a unified system of record
- Behavioral metadata (combining usage + lineage + quality) gives context-ranked alerts, not just signal alerts
- "Agentic" governance layer launching in 2026 automates stewardship decisions

**UX patterns**
- Search-and-browse catalog as primary surface; stewardship queues for data owners
- Lineage accessed from asset detail pages with column-level drill-down

**Integration points**
- REST API (Lineage API, Dataflows API) documented at developer.alation.com
- Open Data Quality Framework API for third-party quality tool integration
- 120+ connectors for databases, ETL, BI, and cloud warehouses
- SAML/SSO authentication; enterprise audit logging

**Known gaps**
- Complex implementation and high cost; significant professional services engagement often required
- Some users report difficult UI navigation; steep learning curve
- Feature parity gaps between cloud and self-hosted versions

**Licence / IP notes**
- Fully proprietary; Open Data Quality Framework API is openly documented for integration partner use.

---

### Sifflet

**Core features**
- Freshness, volume, schema, and distribution monitoring across pipelines and dashboards
- AI-recommended monitors based on risk, change frequency, and downstream impact
- Field-level lineage for Snowflake and other cloud warehouses
- Impact analysis enriched with ownership and downstream usage context
- Priority-ranked anomaly alerts tied to business SLA impact, not just technical severity
- Automatic monitor adaptation as stack changes, avoiding manual rule maintenance
- OpenLineage-compatible ingestion for pipeline metadata
- dbt, Looker, Airflow, Tableau, and warehouse integrations
- CLI for programmatic interaction

**Differentiating features**
- Business-impact prioritisation: distinguishes schema drift from a broken exec dashboard automatically
- AI-recommended monitor coverage eliminates gap discovery lag
- Enriches every issue with ownership, lineage context, and SLA metadata before alerting

**UX patterns**
- Monitor-centric UI; coverage gaps surfaced proactively on a dashboard
- Anomalies enriched with business context before engineer sees them
- Richer alert templates planned for Q1 2026 roadmap

**Integration points**
- REST API with Bearer token authentication; tenant-scoped endpoints
- CLI (`sifflet` command) for scripting and CI/CD integration
- dbt artifact ingestion API for native dbt lineage
- Airflow operator and OpenLineage transport support

**Known gaps**
- Newer entrant; connector breadth still growing compared to Monte Carlo
- No on-premises deployment
- Column-level lineage primarily Snowflake-focused; other warehouses at table level only

**Licence / IP notes**
- Fully proprietary SaaS.

---

### Collibra

**Core features**
- Technical and business lineage mapped end-to-end across source, ETL, and BI layers
- Automated lineage scanning for 40+ data sources, ETL tools, and BI platforms
- Business glossary, data catalog, and policy management in a single platform
- Workflow engine for governance lifecycle (certification, stewardship, issue resolution)
- AI governance capability for cataloguing and monitoring AI models and agents (2026)
- Data quality monitoring with automated rule generation via AI
- Workflow versioning and workspace organisation (2026)
- Enhanced Databricks and Azure lineage (2026 update)
- Regulatory compliance templates for financial services, healthcare, and government

**Differentiating features**
- Deepest governance workflow engine of any commercial platform
- AI governance module treats AI assets as first-class governed entities alongside data assets
- Cloud-only lineage product integrates with self-hosted Collibra Platform via connector

**UX patterns**
- Governance-workflow-first: assets enter certification queues; stewards manage lifecycles
- Business users interact primarily through glossary and policy interfaces; engineers via catalog
- Lineage accessed as one tab within the broader asset profile

**Integration points**
- REST API for all catalog, lineage, and workflow operations
- Python SDK; Java SDK
- SAML/SSO; OAuth 2.0 for API authentication
- 40+ out-of-box connectors; partner integration framework for custom connectors

**Known gaps**
- Significant implementation cost; typically requires SI partner engagement
- Complex UI; steep learning curve for non-governance-specialist users
- G2 and Gartner reviewers note slow issue resolution and limited self-service customisation

**Licence / IP notes**
- Fully proprietary. Cloud-only lineage product; self-hosted platform sold separately.

---

### DataHub

**Core features**
- Open-source metadata graph platform with entity-relationship model for all data assets
- Column-level and table-level lineage across datasets, data jobs, dashboards, and charts
- Elasticsearch-backed full-text search and lineage graph traversal
- OpenLineage protocol ingestion support
- GraphQL API for search, browse, and lineage queries
- Python SDK for programmatic ingestion and lineage definition
- Metadata quality monitoring and data contract enforcement
- 50+ open-source ingestion connectors (Snowflake, BigQuery, Redshift, Spark, Airflow, dbt)
- Role-based access control and fine-grained metadata policies

**Differentiating features**
- LinkedIn-originated and battle-tested at petabyte scale
- Fully open-source (Apache 2.0) with active community and commercial Acryl Data cloud tier
- Native data contracts feature for schema and quality assertions on datasets
- Python SDK is the recommended path for lineage (GraphQL mutations are UI-only)

**UX patterns**
- Search-and-browse catalog with lineage graph visualisation
- Schema history timeline shows column additions, removals, and type changes
- Lineage graph is interactive; hop traversal with depth control

**Integration points**
- GraphQL API (primary) for all entity and lineage queries
- REST Metadata Service API for low-level entity management
- Python SDK (datahub-sdk) for ingestion, lineage, and programmatic access
- OpenLineage HTTP transport compatible
- Kafka event streaming for real-time metadata propagation

**Known gaps**
- Requires significant engineering effort to deploy and maintain at scale
- No native anomaly detection; relies on partner integrations (Anomalo, Monte Carlo, Soda)
- GraphQL mutations not designed for high-throughput bulk operations
- UI less polished than commercial alternatives; limited business-user tooling

**Licence / IP notes**
- Apache 2.0. Acryl Data (managed cloud) is proprietary layer on top. No patent encumbrances identified.

---

### OpenMetadata

**Core features**
- Unified metadata platform: discovery, observability, governance, and lineage in one open-source system
- In-depth column-level lineage with no-code visual lineage editor for manual corrections
- Observability layer with data quality test results displayed inline on lineage graphs
- 120+ turnkey connectors across databases, pipelines, dashboards, and ML systems
- REST API with OpenAPI specification and Python, Java, and Go SDKs
- Business glossary, tags, and classification applied directly in lineage context
- Schema evolution tracking and lineage impact notifications
- Profiler for data statistics and distribution monitoring

**Differentiating features**
- Visual no-code lineage editor allowing manual lineage additions without code
- Quality test results overlaid directly on lineage graph nodes
- OpenAPI-first architecture: everything accessible through typed REST API
- Collate managed cloud adds AI SDK for programmatic governance automation

**UX patterns**
- Explore → Asset Profile → Lineage tab flow
- Column-level lineage togglable from the lineage graph view
- Observability alerts appear as annotations on the asset profile

**Integration points**
- REST API (OpenAPI 3.x specification)
- Python SDK, Java SDK, Go SDK (all officially maintained)
- OpenLineage-compatible ingestion endpoint
- Webhook events for metadata changes
- Slack, MS Teams alerting integrations

**Known gaps**
- Community support only for open-source tier; some enterprise features require Collate managed cloud
- Smaller connector ecosystem than DataHub for niche data sources
- Less widely deployed in large enterprise environments compared to Collibra or Atlan

**Licence / IP notes**
- Apache 2.0. No patent encumbrances identified. Collate (managed cloud layer) is proprietary.

---

### OpenLineage

**Core features**
- Open vendor-neutral specification (JSON Schema + OpenAPI) for lineage event collection
- Defines core entities: Job, Run, Dataset and associated metadata facets
- Column-level lineage facets for Spark, Airflow, and dbt
- Extensible via custom facets using a project-namespaced prefix scheme
- HTTP transport: POST events to `/api/v1/lineage`; batch endpoint for multi-event requests
- Parent-child run relationship via `ParentRunFacet` enabling cross-tool job hierarchies
- `JobDependenciesRunFacet` for tracking dependencies between jobs (2026 addition)
- `DatasetQualityMetricsDatasetFacet` for attaching quality metrics to datasets
- Maintained under Linux Foundation AI & Data; governed by open spec process

**Differentiating features**
- The only cross-platform lineage API standard with broad producer adoption (Airflow, Spark, dbt, Flink)
- Producers emit events; any compatible backend can consume them — no vendor lock-in
- JSON Schema formalisation enables strong tooling and validation

**UX patterns**
- Developer-facing specification and integration library; no end-user UI (Marquez provides the reference UI)
- Python, Java, and Scala client libraries for integration authoring

**Integration points**
- HTTP POST API (OpenAPI spec published at openlineage.io)
- Airflow provider (`apache-airflow-providers-openlineage`)
- Spark integration (`openlineage-spark` library)
- dbt integration (`openlineage-dbt`)
- Backends: Marquez, DataHub, OpenMetadata, Google Dataplex, Amazon SageMaker

**Known gaps**
- Specification only; requires a compatible backend to store and visualise events
- Streaming lineage (Kafka, Flink) coverage still maturing compared to batch coverage

**Licence / IP notes**
- Apache 2.0. Linux Foundation governed. No patent encumbrances identified.

---

### Marquez

**Core features**
- Reference implementation of the OpenLineage API backend
- Stores and visualises OpenLineage job, run, and dataset events
- REST API for querying stored lineage metadata
- Dataset-level and job-level lineage graph visualisation UI
- Data observability dashboard: event statistics over 24hr and 7-day windows
- Job list view with latest run status and duration per job
- Metrics endpoint (`/metrics`) and health check (`/healthcheck`) for ops monitoring
- Configurable backend storage (PostgreSQL)

**Differentiating features**
- Simplest path to a fully working OpenLineage backend with UI, requiring no commercial tools
- Lightweight, self-hostable, suitable for teams wanting OpenLineage without DataHub complexity

**UX patterns**
- Web UI focused on lineage graph browsing and run history
- Observability dashboard surfaces recent event volume and anomaly counts at a glance

**Integration points**
- REST API (OpenLineage HTTP spec) on port 5000
- Admin API on port 5001 for health and metrics
- PostgreSQL backend for persistent storage

**Known gaps**
- Reference implementation — limited governance, cataloguing, and quality features
- No authentication layer out of the box; security must be added externally
- Low governance community activity compared to DataHub or OpenMetadata
- Column-level lineage display less mature than commercial tools

**Licence / IP notes**
- Apache 2.0. Linux Foundation AI & Data project. No patent encumbrances identified.

---

### dbt

**Core features**
- SQL transformation framework with automatic DAG lineage from model dependencies
- Column-level lineage in dbt Cloud Enterprise (dbt Explorer)
- Column evolution lineage lens distinguishing passthrough, rename, and transform operations
- Inherited column descriptions propagated from source through unchanged downstream columns
- Impact analysis for identifying downstream models, dashboards, and tests affected by changes
- Data tests and assertions co-located with model definitions
- dbt Catalog for lineage exploration and documentation

**Differentiating features**
- Lineage is a first-class output of the transformation authoring workflow — not a post-hoc overlay
- Column description inheritance reduces documentation effort for passthrough columns
- OpenLineage event emission for cross-tool lineage federation

**UX patterns**
- Engineers define models in SQL; lineage is generated automatically from `ref()` and `source()` macros
- Explorer UI allows lineage browsing and test result viewing alongside model documentation
- Column-level lineage toggled on demand in Explorer

**Integration points**
- dbt Cloud REST API for triggering runs and retrieving metadata
- dbt Semantic Layer for querying metrics over lineage-aware models
- OpenLineage integration for emitting events to compatible backends
- dbt-core Python library for programmatic model compilation and graph access

**Known gaps**
- Column-level lineage requires dbt Cloud Enterprise; not available in dbt Core (open source)
- Lineage scope is limited to dbt models; upstream source systems and BI tools require external tools
- JSON unpacking, lateral joins, and complex SQL patterns cause column lineage gaps

**Licence / IP notes**
- dbt Core is Apache 2.0. dbt Cloud is proprietary. Column-level lineage is a proprietary Cloud Enterprise feature.

---

### Apache Atlas

**Core features**
- Metadata classification, lineage, and discovery for Hadoop ecosystem (Hive, HBase, Kafka, Sqoop)
- Entity type system for defining custom metadata schemas
- Classification propagation along lineage paths
- REST API for all entity and lineage CRUD operations
- Search across entities using DSL and full-text search
- Audit log for all metadata changes
- Integration with Apache Ranger for policy enforcement

**Differentiating features**
- Native integration with Hadoop/Hive ecosystem — strongest open-source lineage for on-premises big data
- Classification propagation along lineage is unique: sensitivity labels flow downstream automatically

**UX patterns**
- Web UI for browsing entities, lineage graphs, and classifications
- Primarily used by data engineers and governance admins; limited business-user tooling

**Integration points**
- REST API for entities, lineage, search, and classification
- Hooks into Apache Hive, HBase, Sqoop, Kafka, and Storm via native integrations
- Apache Ranger integration for policy-based access control

**Known gaps**
- Primarily Hadoop-centric; limited cloud data warehouse and modern pipeline integrations
- UI is outdated and difficult for non-engineers
- Column-level lineage support is limited compared to modern cloud tools
- Active development pace slower than DataHub or OpenMetadata

**Licence / IP notes**
- Apache 2.0. No patent encumbrances identified.

---

### Egeria

**Core features**
- Open metadata standards platform defining 800+ metadata types across enterprise data assets
- Open APIs, event formats, and types enabling metadata exchange across heterogeneous tools without vendor lock-in
- Metadata exchange via open connectors integrated with catalogs, ETL platforms, and BI tools
- OpenLineage standard support for lineage event capture via HTTP and proxy backend
- Governance engine for automated classification, validation, and lineage tracking
- REST APIs and connectors for enterprise tool integration

**Differentiating features**
- Uniquely focused on metadata interoperability as a standard, not a product — avoids any single-vendor dependency
- 800+ metadata type definitions cover AI models, pipelines, databases, BI reports, and governance artefacts
- LF AI & Data project under open governance; adopted by IBM, ING, and major financial institutions

**UX patterns**
- Primarily a developer and integration-layer platform; limited end-user UI
- Visual governance workbench available but not the primary interface

**Integration points**
- REST APIs (Open Metadata and Governance APIs)
- OpenLineage HTTP and proxy backend
- Connectors for Apache Atlas, Jupyter, Unity Catalog, and others

**Known gaps**
- High implementation complexity; typically requires specialist expertise
- Limited community documentation and tutorials compared to DataHub or OpenMetadata
- No SaaS or managed cloud option

**Licence / IP notes**
- Apache 2.0. Linux Foundation governed. No patent encumbrances identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Table-level lineage from source to BI (every mature tool now provides this)
- Schema change detection and alerting
- Freshness monitoring with configurable thresholds or ML-derived baselines
- Native integration with Snowflake, BigQuery, Databricks, dbt, and Airflow
- Incident management with ownership assignment and Slack/Jira notification
- Column-level lineage (now expected at entry level for commercial tools; maturing in open-source)
- REST or GraphQL API for programmatic access
- Role-based access control on metadata and lineage

### Differentiating Features
- AI-driven anomaly detection with no manual rule configuration (Monte Carlo, Sifflet)
- Business-impact-ranked alerts (Sifflet)
- Embedded collaboration surfacing metadata within third-party tools (Atlan)
- Active metadata propagation — changes flow downstream automatically (Atlan)
- Open Data Quality Framework ingesting quality signals from any external tool (Alation)
- AI governance module tracking AI models and agents as governed assets (Collibra, Atlan)
- Scalable lineage graphs (25–50M+ assets) with JanusGraph backend (Atlan)
- Visual no-code lineage editor for manual corrections (OpenMetadata)
- Column description inheritance along passthrough paths (dbt)

### Underserved Areas / Opportunities
- Streaming pipeline lineage (Kafka, Flink) — most tools focus on batch; real-time lineage is immature
- Cross-tool lineage federation without requiring a single central platform — OpenLineage partially addresses this but adoption is fragmented
- Business-user-accessible lineage — current UIs are engineered for data engineers, not analysts or compliance officers
- Natural-language lineage exploration — users cannot ask "where does this metric come from?" in plain language
- Automated EU AI Act / DORA compliance documentation generated directly from lineage metadata
- Column-level lineage for complex SQL patterns (JSON unpacking, lateral joins, window functions)
- SMB / self-service tier — no well-supported open-source tool provides the full observability + lineage + catalogue bundle without significant engineering overhead
- Lineage coverage for unstructured data, APIs, and ML feature stores — largely absent from current tools

### AI-Augmentation Candidates
- Anomaly detection: replace manual threshold rules with ML-derived baselines per dataset (already attempted by Monte Carlo and Sifflet; opportunity for open-source equivalent)
- Root cause analysis: AI that traces a quality alert back through lineage and identifies the probable origin automatically
- Impact analysis narration: LLM generating a plain-language description of downstream impact from a schema change
- Compliance report generation: producing EU AI Act / DORA / BCBS 239 documentation directly from captured lineage metadata
- Lineage gap detection: AI identifying coverage blind spots — tables or columns with no lineage metadata — and prioritising collection
- Monitor recommendation: suggesting the right monitoring rules per dataset based on usage, change frequency, and downstream criticality (Sifflet has this; opportunity for open-source)

---

## Legal & IP Summary

All major open-source tools in this space (DataHub, OpenMetadata, OpenLineage, Marquez, Apache Atlas, Egeria) are licensed under Apache 2.0, providing permissive terms suitable for commercial derivative use. No patent encumbrances were identified in any of the open-source projects surveyed. Commercial tools (Monte Carlo, Atlan, Alation, Sifflet, Collibra) are fully proprietary; their APIs are publicly documented for integration partner use, and inspecting or reproducing their implementation is not permissible. OpenLineage's JSON Schema specification and OpenAPI definition are Apache 2.0 and can be implemented freely. The OpenMetadata Standards project publishes REST API schemas under open terms. No copyright or licensing concerns arise from building a new open-source tool that implements the OpenLineage specification or is compatible with OpenMetadata.

---

## Recommended Feature Scope

**Must-have (MVP)**
- OpenLineage-compatible event ingestion (HTTP POST endpoint matching the published specification)
- Table-level and column-level lineage graph storage and visualisation
- Freshness, volume, and schema change monitoring with ML-derived baselines (no manual rule configuration)
- Priority-ranked anomaly alerting with downstream impact context drawn from lineage graph
- Connectors for dbt, Airflow, Snowflake, BigQuery, and Databricks (covers 80%+ of target deployments)
- REST API with OpenAPI specification for all lineage and alert operations

**Should-have (v1.1)**
- Cross-tool lineage federation consuming OpenLineage events from multiple producers simultaneously
- Natural-language lineage exploration interface (LLM-powered "where does this column come from?" queries)
- Automated EU AI Act / DORA compliance report generation from captured lineage metadata
- Visual no-code lineage editor for manual lineage corrections and gap-filling
- Quality metric facet ingestion from external tools (Great Expectations, Soda, dbt tests)

**Nice-to-have (backlog)**
- Streaming pipeline lineage for Kafka and Flink
- Column-level lineage for complex SQL patterns (JSON unpacking, lateral joins)
- AI governance module tracking ML models, training datasets, and agent pipelines as governed assets
- Business glossary and classification propagation along lineage paths
- Embedded metadata widgets surfaced within BI tools and Slack
