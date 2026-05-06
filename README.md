# Data Lineage & Observability

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, OpenLineage-compatible open-source platform for end-to-end data lineage tracking, freshness monitoring, and impact analysis.

Data Lineage & Observability is a candidate project for an open-source system that captures lineage metadata across data pipelines, monitors freshness, volume, and schema, and surfaces anomalies and downstream impact in real time. It targets data engineering teams, data platform leads, and compliance officers who need automated lineage and observability without committing to enterprise-priced commercial platforms.

---

## Why Data Lineage & Observability?

- Commercial leaders such as Monte Carlo, Collibra, and Atlan are gated behind enterprise pricing — Atlan starts around $20k/yr, Alation around $120/user/month plus platform fees, and Collibra deployments routinely exceed $500k/yr — leaving SMBs and mid-market teams without a viable option.
- Existing open-source tools (DataHub, OpenMetadata, Apache Atlas, Marquez) require significant engineering effort to deploy and lack native AI-driven anomaly detection, forcing teams to bolt on partner products.
- The EU AI Act and DORA have made automated lineage and impact reporting a legal requirement for tens of thousands of EU organisations from 2025, but no open-source tool today generates the required compliance documentation directly from lineage metadata.
- Column-level lineage is now the expected baseline, yet open-source coverage of complex SQL patterns (JSON unpacking, lateral joins, window functions) and streaming pipelines (Kafka, Flink) is immature across the board.
- OpenLineage is becoming the canonical cross-platform lineage API, but adoption is fragmented and no open backend combines OpenLineage ingestion, observability, cataloguing, and AI-augmented analysis in one system.

---

## Key Features

### Lineage Capture and Graph

- OpenLineage-compatible HTTP ingestion endpoint matching the published specification
- Table-level and column-level lineage graph storage and visualisation
- Cross-tool lineage federation consuming OpenLineage events from multiple producers simultaneously
- Connectors for dbt, Airflow, Snowflake, BigQuery, and Databricks
- Visual no-code lineage editor for manual corrections and gap-filling

### Observability and Monitoring

- Freshness, volume, and schema change monitoring with ML-derived baselines, no manual rule configuration
- Priority-ranked anomaly alerting with downstream impact context drawn from the lineage graph
- Quality metric facet ingestion from external tools such as Great Expectations, Soda, and dbt tests
- Anomaly correlation across freshness, volume, schema, and distribution dimensions

### AI-Augmented Analysis

- Natural-language lineage exploration: ask "where does this column come from?" and get traced, annotated paths
- AI-generated impact analysis identifying affected downstream dashboards, models, and reports from a schema change
- Automated freshness SLA recommendations based on historical pipeline behaviour
- Lineage gap detection that prioritises tables and columns with missing metadata

### Compliance and Governance

- Automated EU AI Act and DORA compliance report generation from captured lineage metadata
- GDPR Article 30 records-of-processing support via personal-data flow tracking
- Role-based access control on metadata and lineage
- Audit logging of metadata changes

### Developer Experience

- REST API with OpenAPI specification covering all lineage and alert operations
- Designed around the OpenLineage Job, Run, and Dataset entity model with extensible facets
- Webhook event delivery for downstream automation

---

## AI-Native Advantage

The project replaces manual thresholds and rule configuration with ML-derived baselines per dataset, and uses LLM-powered exploration so analysts and compliance officers — not just data engineers — can interrogate lineage in plain language. AI-driven impact narration explains what a schema change will break across downstream dashboards and models, while automated compliance reporting generates EU AI Act and DORA technical documentation directly from captured lineage. Together these close the gap between raw lineage capture (where open-source tools stop today) and actionable, regulator-ready insight (which today only enterprise commercial platforms approximate).

---

## Tech Stack & Deployment

The project is built around the **OpenLineage** open standard (Linux Foundation AI & Data) for vendor-neutral event capture, with native producer support via Airflow, Spark, and dbt integrations. It is intended to be self-hostable in the spirit of Marquez and OpenMetadata, with a REST API defined by an OpenAPI 3.x specification. Connectors target the modern cloud data stack — Snowflake, BigQuery, Databricks, dbt, Airflow — covering the bulk of expected deployments. Interoperability with **OpenMetadata** and **Egeria** metadata standards is a design goal so the system can participate in heterogeneous metadata ecosystems rather than becoming another silo.

---

## Market Context

The data observability and cataloguing market is part of the broader $10B+ data management software segment and is among its fastest-growing sub-segments, accelerated by EU AI Act and DORA compliance mandates affecting roughly 22,000 EU financial entities. Commercial pricing ranges from ~$20k/yr (Atlan entry) to $500k+/yr (Collibra, Monte Carlo enterprise), with Monte Carlo valued above $1.6B and Collibra around $3.5B. Primary buyers are data engineering teams managing multi-source pipelines, data platform leads owning quality SLAs, and compliance teams in regulated industries needing auditable data flows.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. The major open-source tools in this space (DataHub, OpenMetadata, OpenLineage, Marquez, Apache Atlas, Egeria) are all Apache 2.0 with no identified patent encumbrances, which is the expected reference point for this project.
