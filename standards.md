# Standards & API Reference

> Project: Data Lineage & Observability · Candidate #188 · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO 8000 — Data Quality**
- URL: https://www.iso.org/standard/81745.html
- ISO 8000 is the international standard for data quality and master data. It defines quality data as "portable data that meets stated requirements" and specifies requirements for the standard exchange of master data. Part 1 (ISO 8000-1:2022) provides an overview; Part 150 covers data quality management roles and responsibilities. Relevant to a data lineage tool because lineage metadata must itself meet data quality requirements to be reliable for compliance reporting.

**ISO/IEC 25012 — Data Quality Model**
- URL: https://www.iso.org/standard/35736.html
- Defines a general data quality model for data held in structured information systems. Specifies quality characteristics including accuracy, completeness, consistency, currentness (freshness), and traceability. The traceability and currentness dimensions directly map to the freshness and lineage pillars of a data observability system.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Requires documented controls over information assets; data lineage supports the asset inventory and audit trail requirements. Relevant to regulated deployments of a lineage tool where the lineage metadata store itself must meet security controls.

---

### W3C & IETF Standards

**W3C PROV — Provenance Data Model (PROV-DM)**
- URL: https://www.w3.org/TR/prov-overview/
- The W3C PROV family (PROV-DM, PROV-O, PROV-N, PROV-XML) defines a conceptual model for representing provenance — the origin, movement, and transformation of data — as a directed graph of Entities, Activities, and Agents. PROV-O is the OWL2 ontology mapping, enabling PROV data to be published as RDF Linked Data. Data lineage is a specialisation of provenance; OpenLineage's conceptual model is aligned with PROV-DM terminology, making PROV a useful reference for data model design.

**PROV-JSONLD (W3C Member Submission, August 2024)**
- URL: https://www.w3.org/submissions/2024/SUBM-prov-jsonld-20240825/
- A JSON-LD 1.1 serialisation of the PROV data model submitted to W3C in 2024. Provides a lightweight, semantically typed format for exchanging provenance/lineage metadata that is also processable as Linked Data. Relevant as an interoperability format for lineage events in federated or scientific data environments.

**RFC 9110 — HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- Defines HTTP request/response semantics underpinning the OpenLineage HTTP transport and all REST API designs in this space. Relevant for implementing the lineage event ingestion endpoint and REST API.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard for compact, URL-safe claims representation used widely for API authentication tokens (Bearer tokens) in data observability APIs (Sifflet, OpenMetadata, DataHub). Relevant for designing authentication for the project's REST API.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- Standard OAuth 2.0 framework used by Collibra, Alation, and Monte Carlo for API authentication. Recommended for the project's authentication design to align with enterprise identity provider (IdP) requirements.

---

### Data Model & API Specifications

**OpenLineage Specification (Linux Foundation)**
- URL: https://openlineage.io / https://github.com/OpenLineage/OpenLineage
- The canonical open specification for lineage metadata collection. Defines Job, Run, and Dataset entities and their metadata facets using JSON Schema (OpenLineage.json) and provides an OpenAPI 3.x definition (OpenLineage.yml) for HTTP-based implementations. The HTTP endpoint accepts POST to `/api/v1/lineage`; a batch endpoint accepts multiple events in a single request. Custom facets use project-namespaced prefixes to avoid collision. Key facets relevant to observability include: `ColumnLineageDatasetFacet`, `DatasetQualityMetricsDatasetFacet`, `ParentRunFacet`, and `JobDependenciesRunFacet`. This specification should be the primary data ingestion API for the project.

**OpenAPI Specification 3.1 (Linux Foundation / OpenAPI Initiative)**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry-standard format for describing REST APIs. OpenMetadata, OpenLineage, and most commercial tools publish OpenAPI 3.x specifications for their APIs. The project's REST API should be designed as an OpenAPI 3.1 document to enable automatic SDK generation, documentation, and integration tooling.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12/schema
- Used by OpenLineage as the formalism for its specification files (one JSON Schema file per facet type, versioned independently). Relevant for designing extensible lineage event facets and validating ingested events.

**GraphQL Specification (GraphQL Foundation)**
- URL: https://spec.graphql.org/
- Used by Monte Carlo (all APIs are GraphQL POST to a single endpoint) and DataHub (primary API surface for search and lineage queries). Relevant for designing the lineage graph traversal and impact analysis query API, where graph-shaped queries map naturally to GraphQL.

**OpenMetadata Standards (REST API Schemas)**
- URL: https://openmetadatastandards.org/schemas/api/rest-api/
- OpenMetadata publishes an open metadata standard schema covering data assets, lineage entities, and quality metrics. Provides a reference data model for metadata entity types beyond what OpenLineage specifies.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://openid.net/connect/
- OAuth 2.0 is the de facto standard for delegated API authorisation; OpenID Connect adds identity layer on top. Used by Collibra, Atlan, and Alation for enterprise SSO integration. Relevant for the project's multi-tenant SaaS authentication design.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP's catalogue of the most critical API security risks including broken object-level authorisation, excessive data exposure, and injection. Relevant for securing the lineage ingestion endpoint, which may be publicly accessible and receives potentially sensitive metadata about data pipelines.

**NIST SP 800-188 — De-Identification of Government Datasets**
- URL: https://csrc.nist.gov/publications/detail/sp/800-188/final
- Provides guidance on de-identification for datasets flowing through anonymisation processes. Relevant for lineage tracking of personal data that traverses de-identification steps, particularly under GDPR Article 30 compliance requirements.

---

### Regulatory Frameworks

**EU AI Act (Regulation 2024/1689)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689
- Requires organisations deploying high-risk AI systems to document data origins, transformations, and quality metrics. Full application from August 2, 2026. Data lineage automation is a prerequisite for compliance; a lineage tool should be capable of generating the technical documentation required under Articles 9–17 directly from captured lineage events.

**Digital Operational Resilience Act — DORA (Regulation 2022/2554)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32022R2554
- In force from January 17, 2025. Mandates real-time lineage tracking and incident reporting for approximately 22,000 EU financial entities. Requires auditable data flow documentation and traceability of data used in risk reporting. A lineage tool serving regulated financial institutions must maintain tamper-evident lineage records and support structured incident export.

**GDPR — Article 30 (Records of Processing Activities)**
- URL: https://gdpr-info.eu/art-30-gdpr/
- Requires organisations processing personal data to maintain records of processing activities including purposes, categories of data, recipients, and transfers. Data lineage that tracks personal data flows across systems fulfils Article 30 requirements when combined with appropriate classification metadata.

**BCBS 239 — Principles for Effective Risk Data Aggregation and Risk Reporting**
- URL: https://www.bis.org/publ/bcbs239.htm
- Basel Committee standard requiring systemically important banks to demonstrate accuracy, completeness, timeliness, and adaptability of risk data aggregation. Lineage is a critical enabler: banks must show end-to-end provenance of risk metrics from source systems through aggregation to regulatory reports. A lineage tool must support structured lineage export in formats acceptable to banking regulators.

---

## Similar Products — Developer Documentation & APIs

### Monte Carlo

- **Description:** Commercial data observability platform monitoring freshness, volume, schema, distribution, and lineage with ML-driven anomaly detection. Market leader valued at $1.6B+.
- **API Documentation:** https://docs.getmontecarlo.com/docs/api
- **API Reference (GraphQL schema):** https://apidocs.getmontecarlo.com/
- **SDKs/Libraries:** Python SDK (pycarlo) — https://pypi.org/project/pycarlo/; CLI available
- **Developer Guide:** https://docs.getmontecarlo.com/docs/developer-resources
- **Standards:** GraphQL (all endpoints are POST to single GraphQL endpoint; no REST-style GET endpoints)
- **Authentication:** API Key (created in settings); passed as HTTP header

---

### DataHub

- **Description:** Open-source metadata platform from LinkedIn/Acryl Data. Full-text search, column-level lineage, data contracts, and OpenLineage ingestion. Apache 2.0.
- **API Documentation:** https://docs.datahub.com/docs/api/datahub-apis
- **GraphQL API:** https://docs.datahub.com/docs/api/graphql/overview
- **Lineage Tutorial:** https://docs.datahub.com/docs/api/tutorials/lineage
- **SDKs/Libraries:** Python SDK (datahub) — https://pypi.org/project/acryl-datahub/; Java SDK available
- **Developer Guide:** https://docs.datahub.com
- **Standards:** GraphQL (primary); REST Metadata Service API (low-level); OpenLineage HTTP compatible
- **Authentication:** API Key (Bearer token); OIDC/SAML for UI

---

### OpenMetadata

- **Description:** Open-source unified metadata platform with 120+ connectors, column-level lineage, observability integration, and visual lineage editor. Apache 2.0 with Collate managed cloud tier.
- **API Documentation:** https://docs.open-metadata.org/latest/main-concepts/metadata-standard/apis
- **API Reference:** https://docs.open-metadata.org/v1.12.x/api-reference
- **SDKs/Libraries:** Python SDK, Java SDK, Go SDK — all at https://docs.open-metadata.org
- **Developer Guide:** https://docs.open-metadata.org
- **Standards:** REST (OpenAPI 3.x specification published); OpenLineage ingestion endpoint
- **Authentication:** JWT Bearer token; SAML/OIDC for SSO

---

### Sifflet

- **Description:** Commercial data observability platform focused on freshness, volume, schema, and distribution monitoring with AI-recommended monitor coverage and business-impact-ranked alerts.
- **API Documentation:** https://docs.siffletdata.com/
- **API Reference:** https://docs.siffletdata.com/reference/getsiffletruleoverview (example endpoint)
- **SDKs/Libraries:** CLI (`sifflet`) — https://docs.siffletdata.com/docs/cli-command-line-interface
- **Developer Guide:** https://docs.siffletdata.com/docs/overview
- **Standards:** REST; tenant-scoped endpoints (`https://{tenant}.siffletdata.com/api/v1/...`)
- **Authentication:** Bearer token (Access Token generated in UI; valid 2 years)

---

### Atlan

- **Description:** Active metadata platform built on JanusGraph supporting 25–50M+ assets with real-time lineage discovery, embedded collaboration, and OpenLineage-compatible ingestion.
- **API Documentation:** https://docs.atlan.com
- **Developer Reference:** https://developer.atlan.com
- **SDKs/Libraries:** Python SDK; Java SDK; TypeScript SDK — all at https://developer.atlan.com
- **Developer Guide:** https://developer.atlan.com/models/montecarlo/ (example integration)
- **Standards:** REST and GraphQL; OpenLineage ingestion compatible
- **Authentication:** API Key (Bearer token); SAML for SSO

---

### Alation

- **Description:** Commercial data catalog with SQL log analysis for lineage inference, Open Data Quality Framework API, and behavioral metadata combining usage, lineage, and quality signals.
- **API Documentation:** https://developer.alation.com
- **Lineage API Reference:** https://developer.alation.com/dev/reference/lineage-overview
- **Open Data Quality Framework Guide:** https://developer.alation.com/dev/docs/alation-open-data-quality-initiative-guide
- **SDKs/Libraries:** No official SDK; REST API documented for direct integration
- **Developer Guide:** https://developer.alation.com
- **Standards:** REST
- **Authentication:** API Token (generated in user profile); OAuth 2.0 for enterprise integrations

---

### Marquez (OpenLineage Reference Backend)

- **Description:** Open-source OpenLineage-compatible metadata service providing lineage storage, REST API, and visualisation UI. LF AI & Data project; Apache 2.0.
- **API Documentation:** https://marquezproject.ai
- **GitHub:** https://github.com/MarquezProject/marquez
- **SDKs/Libraries:** Python client library; Java client library (community-maintained)
- **Developer Guide:** https://www.astronomer.io/docs/learn/marquez (Astronomer integration guide)
- **Standards:** REST; OpenLineage HTTP spec on port 5000; admin API on port 5001
- **Authentication:** No authentication layer out of the box; relies on external proxy/network controls

---

### OpenLineage (Standard + Client Libraries)

- **Description:** Linux Foundation open specification and reference client libraries for lineage event collection from Airflow, Spark, dbt, Flink, and other data processing tools.
- **Specification:** https://openlineage.io/docs/
- **GitHub:** https://github.com/OpenLineage/OpenLineage
- **OpenAPI Spec:** https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.yml
- **JSON Schema:** https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.json
- **SDKs/Libraries:** Python, Java, Scala client libraries; Airflow provider (`apache-airflow-providers-openlineage`)
- **Standards:** OpenAPI 3.x (HTTP); JSON Schema Draft 2020-12 (facets)
- **Authentication:** Configurable per transport; HTTP transport supports Bearer token or API Key injection

---

### Collibra

- **Description:** Enterprise data governance and lineage platform with automated lineage scanning across 40+ sources, governance workflow engine, and AI governance module for AI models and agents.
- **API Documentation:** https://developer.collibra.com (authentication required for full access)
- **Product Documentation:** https://productresources.collibra.com/docs/collibra/latest/Content/CollibraDataLineage/co_collibra-data-lineage.htm
- **SDKs/Libraries:** Python SDK; Java SDK
- **Standards:** REST; OAuth 2.0 for API authentication
- **Authentication:** OAuth 2.0 (client credentials flow for machine-to-machine); SAML for SSO

---

## Notes

**Emerging areas not yet standardised:**
- Streaming lineage (Kafka, Flink, Spark Streaming) — OpenLineage has begun adding streaming facets but the specification is not yet as mature as batch coverage. No interoperability standard exists across streaming platforms.
- ML feature store lineage — tools like Feast and Tecton expose lineage APIs but there is no cross-platform standard analogous to OpenLineage for ML feature provenance.
- Unstructured data lineage — lineage for documents, images, and audio processed through AI pipelines is largely absent from all current standards and tools.

**OpenLineage adoption trajectory:**
OpenLineage is rapidly becoming the de facto cross-platform lineage API, supported natively by Airflow, Spark, dbt, Google Dataplex, Amazon SageMaker, DataHub, OpenMetadata, and Marquez. Any new lineage tool should treat OpenLineage ingestion as a table-stakes requirement, not an optional integration.

**PROV-JSONLD opportunity:**
The W3C PROV-JSONLD submission (August 2024) provides a JSON-LD serialisation that could bridge OpenLineage events to semantic web and Linked Data ecosystems — useful for regulatory reporting contexts where RDF-based knowledge graphs are required (e.g., EU AI Act technical documentation).
