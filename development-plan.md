# Data Lineage & Observability — Phased Development Plan

> Project: 188-data-lineage-observability · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12+ | Dominant language in the data engineering ecosystem; OpenLineage, dbt, Airflow, and all major data tools have Python SDKs; strong ML/AI library support for anomaly detection and LLM integration |
| API framework | FastAPI 0.115+ | OpenAPI 3.1 auto-generation from type annotations (required by standards.md); async support for event ingestion throughput; Pydantic v2 integration for request/response validation |
| Database | PostgreSQL 16+ | Hybrid relational + JSONB model (data-model-suggestion-3); recursive CTE support for lineage traversal; GIN indexes for JSONB facet queries; partitioning for audit logs and event history; mature, battle-tested |
| ORM / query builder | SQLAlchemy 2.0+ with Alembic | Type-safe async queries; Alembic for versioned schema migrations; supports raw SQL for recursive CTEs |
| Task queue | Celery 5.4+ with Redis broker | Async processing for anomaly detection, impact analysis computation, compliance report generation, and connector polling |
| Cache | Redis 7+ | Session cache, API rate limiting, freshness status cache, task queue broker (single dependency for multiple concerns) |
| Frontend | React 18 + TypeScript + Vite | Interactive lineage graph visualisation requires client-side rendering; React ecosystem has mature graph libraries (React Flow, D3); TypeScript for type safety |
| Graph visualisation | React Flow 12+ | Purpose-built for node-edge graph UIs; supports custom nodes, edge types, minimap, and layout algorithms needed for lineage graphs |
| ML / anomaly detection | scikit-learn + Prophet | Rolling statistical baselines (z-score, IQR) via scikit-learn; time-series anomaly detection via Prophet for freshness and volume monitoring |
| LLM integration | LiteLLM | Provider-agnostic LLM calls for natural-language lineage exploration and impact narration; supports OpenAI, Anthropic, local models |
| Authentication | OAuth 2.0 / OIDC via Authlib | Standards-compliant (RFC 6749, OpenID Connect) as required by standards.md; supports enterprise IdP integration |
| API authentication | JWT (RFC 7519) + API keys | JWT for user sessions; API keys (Bearer tokens) for programmatic access matching OpenLineage transport auth patterns |
| Containerisation | Docker + Docker Compose | Self-hostable deployment model; single docker-compose.yml for PostgreSQL, Redis, API, worker, and frontend |
| Testing | pytest + pytest-asyncio + Playwright | pytest for unit/integration; pytest-asyncio for async API tests; Playwright for frontend E2E |
| Code quality | Ruff (linting + formatting) + mypy | Ruff replaces flake8+black+isort; mypy for static type checking |
| Package manager | uv | Fast Python dependency resolution; lockfile support; replaces pip+pip-tools |
| API documentation | Redoc / Swagger UI (FastAPI built-in) | Auto-generated from OpenAPI 3.1 spec; no manual doc maintenance |
| OpenLineage validation | jsonschema 4.x | Validate incoming OpenLineage events against published JSON Schema Draft 2020-12 facet definitions |

### Data Model Selection

The **Hybrid Relational + JSONB model** (data-model-suggestion-3) is selected as the database design foundation. Rationale:

1. **Pragmatic MVP velocity** — 17 tables vs 34 (normalized) or 14 (event-sourced with projection complexity). Fastest path to a working system.
2. **Facet extensibility without DDL** — OpenLineage custom facets stored as validated JSONB; no migration needed when new connectors emit new facet types.
3. **Relational integrity where it matters** — Core entities (jobs, datasets, runs, lineage edges) have foreign keys and indexed joins for graph traversal.
4. **GIN-indexed JSONB queries** — PostgreSQL's JSONB containment queries (`@>`) are performant enough for the expected metadata query patterns.
5. **Facet schema registry** — Application-level JSON Schema validation prevents JSONB columns from becoming unstructured dumps.

Elements from other models are incorporated where appropriate:
- **Event-sourced audit trail** (from model 2) — the `audit_log` table is append-only and partitioned, providing tamper-evident history for DORA compliance.
- **Graph traversal patterns** (from model 4) — recursive CTE queries from model 4's examples are used for impact analysis and upstream tracing.

### Project Structure

```
data-lineage-observability/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── src/
│   └── lineage/
│       ├── __init__.py
│       ├── main.py                          # FastAPI application entry point
│       ├── config.py                        # Settings via pydantic-settings
│       ├── dependencies.py                  # FastAPI dependency injection
│       │
│       ├── db/
│       │   ├── __init__.py
│       │   ├── engine.py                    # SQLAlchemy async engine
│       │   ├── session.py                   # Session factory
│       │   └── models/
│       │       ├── __init__.py
│       │       ├── core.py                  # Namespaces, Jobs, Datasets, Runs
│       │       ├── fields.py                # DatasetFields
│       │       ├── lineage.py               # LineageEdges, ColumnLineageEdges
│       │       ├── observability.py          # Monitors, AnomalyAlerts, SchemaChanges
│       │       ├── compliance.py            # ComplianceRecords
│       │       ├── auth.py                  # Users, Roles, ApiTokens
│       │       └── audit.py                 # AuditLog
│       │
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── openlineage.py               # OpenLineage event Pydantic models
│       │   ├── lineage.py                   # Lineage query request/response
│       │   ├── observability.py             # Monitor/alert schemas
│       │   ├── compliance.py                # Compliance report schemas
│       │   └── auth.py                      # Auth request/response schemas
│       │
│       ├── api/
│       │   ├── __init__.py
│       │   ├── router.py                    # Root router aggregation
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── ingest.py                # POST /api/v1/lineage (OpenLineage endpoint)
│       │   │   ├── lineage.py               # Lineage graph query endpoints
│       │   │   ├── datasets.py              # Dataset CRUD + search
│       │   │   ├── jobs.py                  # Job CRUD + search
│       │   │   ├── runs.py                  # Run history endpoints
│       │   │   ├── monitors.py              # Monitor CRUD + evaluation
│       │   │   ├── alerts.py                # Alert management endpoints
│       │   │   ├── compliance.py            # Compliance report endpoints
│       │   │   ├── search.py                # Full-text search
│       │   │   └── auth.py                  # Auth endpoints
│       │   └── middleware/
│       │       ├── __init__.py
│       │       ├── auth.py                  # JWT/API key validation
│       │       ├── audit.py                 # Audit logging middleware
│       │       └── rate_limit.py            # Rate limiting
│       │
│       ├── services/
│       │   ├── __init__.py
│       │   ├── ingest.py                    # OpenLineage event processing
│       │   ├── lineage_graph.py             # Lineage traversal (upstream/downstream)
│       │   ├── impact_analysis.py           # Downstream impact computation
│       │   ├── schema_diff.py               # Schema change detection
│       │   ├── monitor_engine.py            # Monitor evaluation engine
│       │   ├── anomaly_detection.py         # ML-based anomaly detection
│       │   ├── compliance_reporter.py       # DORA/GDPR/AI Act report generation
│       │   ├── nl_explorer.py               # Natural-language lineage queries
│       │   └── facet_validator.py           # JSON Schema facet validation
│       │
│       ├── connectors/
│       │   ├── __init__.py
│       │   ├── base.py                      # Abstract connector interface
│       │   ├── dbt.py                       # dbt artifact ingestion
│       │   ├── airflow.py                   # Airflow metadata connector
│       │   ├── snowflake.py                 # Snowflake metadata connector
│       │   ├── bigquery.py                  # BigQuery metadata connector
│       │   └── databricks.py               # Databricks metadata connector
│       │
│       ├── workers/
│       │   ├── __init__.py
│       │   ├── celery_app.py                # Celery application config
│       │   ├── tasks/
│       │   │   ├── __init__.py
│       │   │   ├── monitor_evaluation.py    # Periodic monitor checks
│       │   │   ├── impact_recompute.py      # Recompute downstream impact
│       │   │   ├── baseline_training.py     # ML baseline model training
│       │   │   ├── compliance_snapshot.py   # Periodic compliance snapshots
│       │   │   └── connector_sync.py        # Connector polling tasks
│       │   └── schedules.py                 # Celery Beat schedule
│       │
│       └── migrations/
│           ├── env.py
│           └── versions/
│               └── ...                      # Alembic migration files
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── api/
│   │   │   └── client.ts                    # Auto-generated API client from OpenAPI
│   │   ├── components/
│   │   │   ├── lineage/
│   │   │   │   ├── LineageGraph.tsx          # React Flow lineage visualisation
│   │   │   │   ├── DatasetNode.tsx           # Custom dataset node
│   │   │   │   ├── JobNode.tsx               # Custom job node
│   │   │   │   └── LineageControls.tsx       # Zoom, filter, depth controls
│   │   │   ├── observability/
│   │   │   │   ├── MonitorDashboard.tsx
│   │   │   │   ├── AlertFeed.tsx
│   │   │   │   └── FreshnessTimeline.tsx
│   │   │   ├── search/
│   │   │   │   └── GlobalSearch.tsx
│   │   │   └── common/
│   │   │       └── ...
│   │   ├── pages/
│   │   │   ├── DatasetsPage.tsx
│   │   │   ├── DatasetDetailPage.tsx
│   │   │   ├── JobsPage.tsx
│   │   │   ├── LineagePage.tsx
│   │   │   ├── AlertsPage.tsx
│   │   │   ├── MonitorsPage.tsx
│   │   │   └── CompliancePage.tsx
│   │   └── stores/
│   │       └── ...                          # Zustand state stores
│   └── tests/
│       └── ...                              # Playwright E2E tests
│
└── tests/
    ├── conftest.py                          # Shared fixtures (test DB, client)
    ├── factories.py                         # Test data factories
    ├── unit/
    │   ├── test_ingest.py
    │   ├── test_lineage_graph.py
    │   ├── test_schema_diff.py
    │   ├── test_anomaly_detection.py
    │   └── test_facet_validator.py
    ├── integration/
    │   ├── test_api_ingest.py
    │   ├── test_api_lineage.py
    │   ├── test_api_monitors.py
    │   ├── test_connectors.py
    │   └── test_impact_analysis.py
    └── e2e/
        ├── test_lineage_flow.py
        └── test_alert_workflow.py
```

---

## Phase 1: Foundation & Project Skeleton

### Purpose

Establish the project scaffolding, database schema, configuration system, and development toolchain. After this phase, the project builds, tests run (even if trivially), Docker containers start, and the database schema is applied. No business logic yet — this is pure infrastructure.

### Tasks

#### 1.1 — Project Initialisation & Tooling

**What**: Create the Python project with uv, configure Ruff, mypy, pytest, and pre-commit hooks.

**Design**:

```toml
# pyproject.toml
[project]
name = "data-lineage-observability"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "sqlalchemy[asyncio]>=2.0.35",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "redis>=5.2.0",
    "celery[redis]>=5.4.0",
    "httpx>=0.28.0",
    "jsonschema>=4.23.0",
    "authlib>=1.3.0",
    "python-jose[cryptography]>=3.3.0",
    "bcrypt>=4.2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "factory-boy>=3.3.0",
    "httpx>=0.28.0",
]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "ANN", "B", "SIM", "TCH"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

```python
# src/lineage/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "Data Lineage & Observability"
    debug: bool = False
    log_level: str = "INFO"
    api_prefix: str = "/api/v1"

    # Database
    database_url: str = "postgresql+asyncpg://lineage:lineage@localhost:5432/lineage"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Auth
    jwt_secret_key: str = "CHANGE-ME-IN-PRODUCTION"
    jwt_algorithm: str = "HS256"
    jwt_expiration_minutes: int = 60
    api_key_header: str = "X-API-Key"

    # OpenLineage
    openlineage_validate_schemas: bool = True
    openlineage_max_event_size_bytes: int = 10_485_760  # 10 MB

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    model_config = {"env_prefix": "LINEAGE_", "env_file": ".env"}

settings = Settings()
```

**Testing**:
- `Unit: default_settings_load — Settings() with no env vars produces valid defaults`
- `Unit: env_override — LINEAGE_DEBUG=true overrides debug to True`
- `Unit: database_url_format — database_url contains asyncpg driver prefix`

---

#### 1.2 — Database Engine & Session Management

**What**: Configure SQLAlchemy async engine, session factory, and FastAPI dependency injection for database sessions.

**Design**:

```python
# src/lineage/db/engine.py
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine
from lineage.config import settings

def create_engine() -> AsyncEngine:
    return create_async_engine(
        settings.database_url,
        pool_size=settings.database_pool_size,
        max_overflow=settings.database_max_overflow,
        echo=settings.debug,
    )

engine = create_engine()
```

```python
# src/lineage/db/session.py
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker
from lineage.db.engine import engine

async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

```python
# src/lineage/dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from lineage.db.session import get_session

async def get_db(session: AsyncSession = Depends(get_session)) -> AsyncSession:
    return session
```

**Testing**:
- `Integration: session_lifecycle — session commits on success, rolls back on exception`
- `Integration: connection_pool — engine creates connections up to pool_size`
- `Unit: get_session_yields_async_session — dependency yields AsyncSession type`

---

#### 1.3 — Core Database Models & Initial Migration

**What**: Define SQLAlchemy ORM models for all 17 tables from the hybrid relational + JSONB data model and create the initial Alembic migration.

**Design**:

```python
# src/lineage/db/models/core.py
import uuid
from datetime import datetime
from sqlalchemy import (
    Boolean, DateTime, ForeignKey, Integer, String, Text, func
)
from sqlalchemy.dialects.postgresql import ARRAY, JSONB, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Namespace(Base):
    __tablename__ = "namespaces"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(512), unique=True, nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    jobs: Mapped[list["Job"]] = relationship(back_populates="namespace")
    datasets: Mapped[list["Dataset"]] = relationship(back_populates="namespace")

class Job(Base):
    __tablename__ = "jobs"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    namespace_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("namespaces.id"), nullable=False)
    name: Mapped[str] = mapped_column(String(512), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    job_type: Mapped[str | None] = mapped_column(String(64))
    latest_run_state: Mapped[str | None] = mapped_column(String(32))
    latest_run_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    facets: Mapped[dict] = mapped_column(JSONB, default=dict)
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    tags: Mapped[list[str] | None] = mapped_column(ARRAY(Text))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    __table_args__ = (
        {"schema": None},  # unique constraint on (namespace_id, name)
    )

    namespace: Mapped["Namespace"] = relationship(back_populates="jobs")

class Dataset(Base):
    __tablename__ = "datasets"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    namespace_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("namespaces.id"), nullable=False)
    name: Mapped[str] = mapped_column(String(512), nullable=False)
    type: Mapped[str] = mapped_column(String(32), default="DB_TABLE")
    description: Mapped[str | None] = mapped_column(Text)
    physical_name: Mapped[str | None] = mapped_column(String(512))
    source_system: Mapped[str | None] = mapped_column(String(256))
    current_schema: Mapped[dict | None] = mapped_column(JSONB)
    facets: Mapped[dict] = mapped_column(JSONB, default=dict)
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    tags: Mapped[list[str] | None] = mapped_column(ARRAY(Text))
    last_updated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    last_row_count: Mapped[int | None] = mapped_column(Integer)
    freshness_status: Mapped[str] = mapped_column(String(16), default="UNKNOWN")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    namespace: Mapped["Namespace"] = relationship(back_populates="datasets")
    fields: Mapped[list["DatasetField"]] = relationship(back_populates="dataset", cascade="all, delete-orphan")

class Run(Base):
    __tablename__ = "runs"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True)  # client-provided
    job_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("jobs.id"), nullable=False)
    parent_run_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("runs.id"))
    state: Mapped[str] = mapped_column(String(32), default="START")
    nominal_start_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    nominal_end_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    actual_start_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    actual_end_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    facets: Mapped[dict] = mapped_column(JSONB, default=dict)
    input_datasets: Mapped[list] = mapped_column(JSONB, default=list)
    output_datasets: Mapped[list] = mapped_column(JSONB, default=list)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

```python
# src/lineage/db/models/fields.py
class DatasetField(Base):
    __tablename__ = "dataset_fields"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    dataset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("datasets.id", ondelete="CASCADE"), nullable=False)
    name: Mapped[str] = mapped_column(String(256), nullable=False)
    type: Mapped[str | None] = mapped_column(String(128))
    ordinal_position: Mapped[int | None] = mapped_column(Integer)
    description: Mapped[str | None] = mapped_column(Text)
    is_nullable: Mapped[bool] = mapped_column(Boolean, default=True)
    is_primary_key: Mapped[bool] = mapped_column(Boolean, default=False)
    tags: Mapped[list[str] | None] = mapped_column(ARRAY(Text))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    dataset: Mapped["Dataset"] = relationship(back_populates="fields")
```

```python
# src/lineage/db/models/lineage.py
class LineageEdge(Base):
    __tablename__ = "lineage_edges"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    source_dataset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("datasets.id"), nullable=False)
    target_dataset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("datasets.id"), nullable=False)
    job_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("jobs.id"), nullable=False)
    edge_type: Mapped[str] = mapped_column(String(32), default="TRANSFORM")
    latest_run_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("runs.id"))
    latest_run_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    latest_run_state: Mapped[str | None] = mapped_column(String(32))
    properties: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

class ColumnLineageEdge(Base):
    __tablename__ = "column_lineage_edges"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    lineage_edge_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("lineage_edges.id", ondelete="CASCADE"), nullable=False)
    source_field_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("dataset_fields.id"), nullable=False)
    target_field_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("dataset_fields.id"), nullable=False)
    transformation_type: Mapped[str] = mapped_column(String(32), default="IDENTITY")
    transformation_description: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

Remaining models for observability, compliance, auth, and audit tables follow the same pattern from data-model-suggestion-3, including:
- `FacetSchema` — facet_schemas table
- `Monitor` — monitors table with JSONB config/baseline
- `AnomalyAlert` — anomaly_alerts table with JSONB details
- `SchemaChange` — schema_changes table with JSONB changes array
- `ComplianceRecord` — compliance_records table with JSONB data
- `User`, `Role`, `UserRole`, `ApiToken` — auth tables
- `AuditLog` — partitioned audit_log table

**Testing**:
- `Integration: create_all_tables — Alembic migration applies without errors on clean database`
- `Integration: namespace_crud — create, read, update namespace via ORM`
- `Integration: unique_constraints — duplicate (namespace_id, name) raises IntegrityError`
- `Integration: cascade_delete — deleting dataset cascades to dataset_fields`
- `Integration: jsonb_default — new Job has empty dict for facets, not NULL`

---

#### 1.4 — Docker Compose & Container Setup

**What**: Create Dockerfile for the API service and docker-compose.yml with PostgreSQL, Redis, API, and worker services.

**Design**:

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
RUN pip install uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY src/ ./src/
COPY alembic.ini ./
COPY src/lineage/migrations/ ./src/lineage/migrations/

FROM base AS api
CMD ["uv", "run", "uvicorn", "lineage.main:app", "--host", "0.0.0.0", "--port", "8000"]

FROM base AS worker
CMD ["uv", "run", "celery", "-A", "lineage.workers.celery_app", "worker", "--loglevel=info"]
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: lineage
      POSTGRES_USER: lineage
      POSTGRES_PASSWORD: lineage
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lineage"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5

  api:
    build:
      context: .
      target: api
    ports: ["8000:8000"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    environment:
      LINEAGE_DATABASE_URL: "postgresql+asyncpg://lineage:lineage@postgres:5432/lineage"
      LINEAGE_REDIS_URL: "redis://redis:6379/0"
      LINEAGE_CELERY_BROKER_URL: "redis://redis:6379/1"

  worker:
    build:
      context: .
      target: worker
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    environment:
      LINEAGE_DATABASE_URL: "postgresql+asyncpg://lineage:lineage@postgres:5432/lineage"
      LINEAGE_CELERY_BROKER_URL: "redis://redis:6379/1"
      LINEAGE_CELERY_RESULT_BACKEND: "redis://redis:6379/2"

volumes:
  pgdata:
```

**Testing**:
- `Integration: docker_build — Docker build succeeds for both api and worker targets`
- `Integration: compose_up — docker compose up starts all services; api returns 200 on GET /health`
- `Integration: migrations_on_start — API container applies Alembic migrations on startup`

---

#### 1.5 — FastAPI Application Shell & Health Endpoint

**What**: Create the FastAPI application with CORS, health check, and OpenAPI metadata.

**Design**:

```python
# src/lineage/main.py
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from lineage.config import settings
from lineage.db.engine import engine
from lineage.api.router import api_router

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # Startup: run migrations, warm caches
    yield
    # Shutdown: dispose engine
    await engine.dispose()

app = FastAPI(
    title=settings.app_name,
    version="0.1.0",
    description="OpenLineage-compatible data lineage tracking and observability platform",
    lifespan=lifespan,
    openapi_url=f"{settings.api_prefix}/openapi.json",
    docs_url=f"{settings.api_prefix}/docs",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Tighten in production
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}

app.include_router(api_router, prefix=settings.api_prefix)
```

**Testing**:
- `Integration: health_endpoint — GET /health returns 200 with {"status": "ok"}`
- `Integration: openapi_spec — GET /api/v1/openapi.json returns valid OpenAPI 3.1 document`
- `Integration: cors_headers — OPTIONS request returns Access-Control-Allow-Origin header`

---

## Phase 2: OpenLineage Event Ingestion

### Purpose

Implement the core OpenLineage-compatible event ingestion endpoint — the primary data entry point for the entire system. After this phase, the system can receive OpenLineage events from Airflow, Spark, dbt, and other producers, validate them against JSON Schema, and persist them as jobs, runs, datasets, lineage edges, and facets. This is the single most important capability.

### Tasks

#### 2.1 — OpenLineage Pydantic Models

**What**: Define Pydantic v2 models that represent the OpenLineage event specification, including all standard facets.

**Design**:

```python
# src/lineage/schemas/openlineage.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field, ConfigDict
from typing import Any

class OpenLineageRunFacets(BaseModel):
    model_config = ConfigDict(extra="allow")  # Custom facets pass through

    nominalTime: dict[str, Any] | None = None
    errorMessage: dict[str, Any] | None = None
    externalQuery: dict[str, Any] | None = None
    parent: dict[str, Any] | None = None

class OpenLineageRun(BaseModel):
    runId: UUID
    facets: OpenLineageRunFacets | None = None

class OpenLineageJobFacets(BaseModel):
    model_config = ConfigDict(extra="allow")

    sql: dict[str, Any] | None = None
    sourceCode: dict[str, Any] | None = None
    sourceCodeLocation: dict[str, Any] | None = None
    documentation: dict[str, Any] | None = None
    ownership: dict[str, Any] | None = None

class OpenLineageJob(BaseModel):
    namespace: str
    name: str
    facets: OpenLineageJobFacets | None = None

class OpenLineageDatasetFacets(BaseModel):
    model_config = ConfigDict(extra="allow")

    schema_: dict[str, Any] | None = Field(None, alias="schema")
    dataQualityMetrics: dict[str, Any] | None = None
    columnLineage: dict[str, Any] | None = None
    storage: dict[str, Any] | None = None
    dataSource: dict[str, Any] | None = None

class OpenLineageInputDataset(BaseModel):
    namespace: str
    name: str
    facets: OpenLineageDatasetFacets | None = None
    inputFacets: dict[str, Any] | None = None

class OpenLineageOutputDataset(BaseModel):
    namespace: str
    name: str
    facets: OpenLineageDatasetFacets | None = None
    outputFacets: dict[str, Any] | None = None

class OpenLineageEvent(BaseModel):
    eventType: str = Field(..., pattern="^(START|RUNNING|COMPLETE|FAIL|ABORT|OTHER)$")
    eventTime: datetime
    run: OpenLineageRun
    job: OpenLineageJob
    inputs: list[OpenLineageInputDataset] = Field(default_factory=list)
    outputs: list[OpenLineageOutputDataset] = Field(default_factory=list)
    producer: str | None = None
    schemaURL: str | None = None

class OpenLineageBatchRequest(BaseModel):
    events: list[OpenLineageEvent]
```

**Testing**:
- `Unit: valid_complete_event — full OpenLineage COMPLETE event with all facets parses correctly`
- `Unit: valid_start_event — START event with only run and job parses correctly`
- `Unit: invalid_event_type — eventType "UNKNOWN" raises ValidationError`
- `Unit: custom_facets_preserved — extra facets in run.facets are preserved (ConfigDict extra="allow")`
- `Unit: missing_run_id — event without run.runId raises ValidationError`
- `Unit: batch_request — list of 3 events parses as OpenLineageBatchRequest`

---

#### 2.2 — Facet Schema Validator

**What**: Build a service that validates OpenLineage facet JSONB values against registered JSON Schema definitions from the facet_schemas table.

**Design**:

```python
# src/lineage/services/facet_validator.py
from jsonschema import Draft202012Validator, ValidationError as JsonSchemaError
from sqlalchemy.ext.asyncio import AsyncSession
from lineage.db.models.core import FacetSchema
from typing import Any

class FacetValidator:
    def __init__(self) -> None:
        self._schema_cache: dict[str, Draft202012Validator] = {}

    async def load_schemas(self, session: AsyncSession) -> None:
        """Load all registered facet schemas into memory."""
        result = await session.execute(select(FacetSchema))
        for schema_row in result.scalars().all():
            validator = Draft202012Validator(schema_row.json_schema)
            self._schema_cache[schema_row.facet_name] = validator

    def validate_facets(
        self, facets: dict[str, Any], target: str
    ) -> list[dict[str, str]]:
        """
        Validate facets dict against registered schemas.
        Returns list of validation errors (empty if all valid).
        Unknown facets (custom, not in registry) are allowed through.
        """
        errors: list[dict[str, str]] = []
        for facet_name, facet_value in facets.items():
            validator = self._schema_cache.get(facet_name)
            if validator is None:
                continue  # Custom facet — no schema registered, allow through
            try:
                validator.validate(facet_value)
            except JsonSchemaError as e:
                errors.append({
                    "facet": facet_name,
                    "path": ".".join(str(p) for p in e.absolute_path),
                    "message": e.message,
                })
        return errors
```

**Testing**:
- `Unit: valid_standard_facet — nominalTime facet matching schema returns no errors`
- `Unit: invalid_standard_facet — nominalTime facet missing nominalStartTime returns error`
- `Unit: custom_facet_passthrough — facet not in registry returns no errors (allowed through)`
- `Unit: multiple_errors — facets with 2 invalid standard facets returns 2 errors`
- `Integration: load_schemas — loads schemas from facet_schemas table into cache`

---

#### 2.3 — Event Ingestion Service

**What**: Implement the core service that processes an OpenLineage event: upserts namespaces, jobs, datasets, dataset fields, runs, and lineage edges, and stores facets as JSONB.

**Design**:

```python
# src/lineage/services/ingest.py
from sqlalchemy.ext.asyncio import AsyncSession
from lineage.schemas.openlineage import OpenLineageEvent
from lineage.db.models.core import Namespace, Job, Dataset, Run
from lineage.db.models.fields import DatasetField
from lineage.db.models.lineage import LineageEdge, ColumnLineageEdge

class IngestService:
    async def process_event(
        self, session: AsyncSession, event: OpenLineageEvent
    ) -> dict[str, str]:
        """
        Process a single OpenLineage event:
        1. Upsert namespace for job
        2. Upsert job, merge job facets
        3. Upsert or update run (state transition: START→RUNNING→COMPLETE/FAIL/ABORT)
        4. For each input dataset: upsert namespace, upsert dataset, upsert fields from schema facet
        5. For each output dataset: same as inputs
        6. Create/update lineage edges (input→output via job)
        7. Extract column-level lineage from columnLineage facet if present
        8. Update job.latest_run_state and job.latest_run_time
        9. Update dataset.last_updated_at, dataset.last_row_count from dataQualityMetrics facet
        Returns: {"status": "ok", "run_id": str(event.run.runId)}
        """

    async def _upsert_namespace(
        self, session: AsyncSession, name: str
    ) -> Namespace:
        """INSERT ... ON CONFLICT (name) DO UPDATE SET updated_at = now()"""

    async def _upsert_job(
        self, session: AsyncSession, namespace_id: uuid.UUID,
        name: str, facets: dict
    ) -> Job:
        """INSERT ... ON CONFLICT (namespace_id, name) DO UPDATE SET facets = merged_facets"""

    async def _upsert_dataset(
        self, session: AsyncSession, namespace_id: uuid.UUID,
        name: str, facets: dict, schema_facet: dict | None
    ) -> Dataset:
        """Upsert dataset; if schema facet present, sync dataset_fields"""

    async def _sync_dataset_fields(
        self, session: AsyncSession, dataset_id: uuid.UUID,
        schema_fields: list[dict]
    ) -> list[DatasetField]:
        """Diff current fields vs schema facet fields; add new, update changed, soft-delete removed"""

    async def _upsert_run(
        self, session: AsyncSession, event: OpenLineageEvent, job_id: uuid.UUID
    ) -> Run:
        """Upsert run; merge facets; update state and timestamps based on eventType"""

    async def _create_lineage_edges(
        self, session: AsyncSession, run: Run, job_id: uuid.UUID,
        input_datasets: list[Dataset], output_datasets: list[Dataset]
    ) -> list[LineageEdge]:
        """For each (input, output) pair, upsert a lineage edge"""

    async def _extract_column_lineage(
        self, session: AsyncSession, edge: LineageEdge,
        column_lineage_facet: dict
    ) -> list[ColumnLineageEdge]:
        """Parse ColumnLineageDatasetFacet and create column-level edges"""
```

State transition rules for runs:
- `START` → sets `actual_start_time`
- `RUNNING` → no timestamp change
- `COMPLETE` → sets `actual_end_time`, state = `COMPLETE`
- `FAIL` → sets `actual_end_time`, state = `FAIL`, extracts error facet
- `ABORT` → sets `actual_end_time`, state = `ABORT`

**Testing**:
- `Integration: ingest_start_event — creates namespace, job, run with state START`
- `Integration: ingest_complete_event — updates run to COMPLETE, sets actual_end_time`
- `Integration: ingest_with_inputs_outputs — creates datasets and lineage edges`
- `Integration: ingest_with_schema_facet — creates dataset_fields from schema facet`
- `Integration: ingest_with_column_lineage — creates column_lineage_edges from columnLineage facet`
- `Integration: upsert_idempotency — same event ingested twice does not create duplicates`
- `Integration: facet_merge — subsequent events merge new facets with existing, not replace`
- `Integration: run_state_transition — START then COMPLETE updates state correctly`
- `Unit: invalid_state_transition — COMPLETE before START is handled gracefully (event ordering)`

---

#### 2.4 — OpenLineage HTTP Ingestion Endpoint

**What**: Implement the `POST /api/v1/lineage` endpoint matching the OpenLineage HTTP transport specification, plus a batch endpoint.

**Design**:

```python
# src/lineage/api/v1/ingest.py
from fastapi import APIRouter, Depends, HTTPException, status, Request
from sqlalchemy.ext.asyncio import AsyncSession
from lineage.dependencies import get_db
from lineage.schemas.openlineage import OpenLineageEvent, OpenLineageBatchRequest
from lineage.services.ingest import IngestService
from lineage.services.facet_validator import FacetValidator
from lineage.config import settings

router = APIRouter(tags=["OpenLineage Ingestion"])
ingest_service = IngestService()

@router.post(
    "/lineage",
    status_code=status.HTTP_201_CREATED,
    summary="Ingest OpenLineage event",
    description="Accepts a single OpenLineage event per the OpenLineage HTTP transport spec."
)
async def ingest_event(
    event: OpenLineageEvent,
    request: Request,
    session: AsyncSession = Depends(get_db),
) -> dict[str, str]:
    # Validate event size
    content_length = request.headers.get("content-length", "0")
    if int(content_length) > settings.openlineage_max_event_size_bytes:
        raise HTTPException(status_code=413, detail="Event too large")

    result = await ingest_service.process_event(session, event)
    return result

@router.post(
    "/lineage/batch",
    status_code=status.HTTP_201_CREATED,
    summary="Ingest batch of OpenLineage events",
)
async def ingest_batch(
    batch: OpenLineageBatchRequest,
    session: AsyncSession = Depends(get_db),
) -> dict[str, int]:
    processed = 0
    for event in batch.events:
        await ingest_service.process_event(session, event)
        processed += 1
    return {"processed": processed}
```

**Testing**:
- `Integration: post_valid_event — POST /api/v1/lineage with valid event returns 201`
- `Integration: post_invalid_event — POST with missing eventType returns 422`
- `Integration: post_batch — POST /api/v1/lineage/batch with 5 events returns {"processed": 5}`
- `Integration: post_oversized_event — event exceeding max size returns 413`
- `Integration: content_type_json — POST with non-JSON content-type returns 415`
- `E2E: airflow_event_fixture — POST real Airflow OpenLineage event fixture, verify job/dataset/run created`
- `E2E: dbt_event_fixture — POST real dbt OpenLineage event fixture, verify column lineage created`

---

## Phase 3: Lineage Graph Queries & API

### Purpose

Expose the stored lineage data through query APIs. After this phase, users and downstream tools can query upstream and downstream lineage, search for datasets and jobs, and retrieve run history. This makes the ingested data usable.

### Tasks

#### 3.1 — Lineage Graph Traversal Service

**What**: Implement recursive CTE-based lineage traversal for upstream and downstream queries at both table and column level.

**Design**:

```python
# src/lineage/services/lineage_graph.py
from dataclasses import dataclass
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

@dataclass
class LineageNode:
    dataset_id: UUID
    dataset_name: str
    namespace_name: str
    node_type: str  # DB_TABLE, STREAM, FILE
    depth: int
    freshness_status: str

@dataclass
class LineageEdgeResult:
    source: LineageNode
    target: LineageNode
    job_name: str
    edge_type: str
    latest_run_state: str | None
    latest_run_time: str | None

@dataclass
class LineageGraphResult:
    nodes: list[LineageNode]
    edges: list[LineageEdgeResult]
    root_dataset_id: UUID
    direction: str  # "upstream" | "downstream"
    max_depth_reached: bool

class LineageGraphService:
    async def get_upstream(
        self, session: AsyncSession, dataset_id: UUID, max_depth: int = 10
    ) -> LineageGraphResult:
        """
        Recursive CTE walking lineage_edges backward (target→source).
        Returns all upstream datasets and the edges connecting them.
        """
        query = text("""
            WITH RECURSIVE upstream AS (
                SELECT le.source_dataset_id, le.target_dataset_id,
                       le.job_id, le.edge_type, le.latest_run_state,
                       le.latest_run_time, 1 AS depth,
                       ARRAY[le.target_dataset_id, le.source_dataset_id] AS path
                FROM lineage_edges le
                WHERE le.target_dataset_id = :dataset_id

                UNION ALL

                SELECT le.source_dataset_id, le.target_dataset_id,
                       le.job_id, le.edge_type, le.latest_run_state,
                       le.latest_run_time, u.depth + 1,
                       u.path || le.source_dataset_id
                FROM lineage_edges le
                JOIN upstream u ON le.target_dataset_id = u.source_dataset_id
                WHERE u.depth < :max_depth
                  AND NOT le.source_dataset_id = ANY(u.path)
            )
            SELECT u.*, d.name, d.type, d.freshness_status, n.name AS ns_name,
                   j.name AS job_name
            FROM upstream u
            JOIN datasets d ON d.id = u.source_dataset_id
            JOIN namespaces n ON n.id = d.namespace_id
            JOIN jobs j ON j.id = u.job_id
            ORDER BY u.depth
        """)
        # ... execute and map to LineageGraphResult

    async def get_downstream(
        self, session: AsyncSession, dataset_id: UUID, max_depth: int = 10
    ) -> LineageGraphResult:
        """Recursive CTE walking lineage_edges forward (source→target)."""

    async def get_column_upstream(
        self, session: AsyncSession, field_id: UUID, max_depth: int = 20
    ) -> list[dict]:
        """
        Column-level upstream trace using column_lineage_edges.
        Returns transformation chain from target column back to source columns.
        """

    async def get_column_downstream(
        self, session: AsyncSession, field_id: UUID, max_depth: int = 20
    ) -> list[dict]:
        """Column-level downstream trace."""
```

**Testing**:
- `Integration: upstream_linear — A→B→C chain, upstream of C returns [B, A] with depths [1, 2]`
- `Integration: downstream_linear — A→B→C chain, downstream of A returns [B, C]`
- `Integration: upstream_diamond — A→C, B→C, upstream of C returns [A, B] both at depth 1`
- `Integration: cycle_prevention — A→B→A cycle does not cause infinite recursion`
- `Integration: max_depth_respected — 15-node chain with max_depth=5 returns only 5 nodes`
- `Integration: column_upstream — field revenue traces back through transformation chain`
- `Integration: empty_lineage — dataset with no edges returns empty graph`

---

#### 3.2 — Dataset & Job Query APIs

**What**: REST endpoints for listing, searching, and retrieving datasets, jobs, and runs.

**Design**:

```python
# src/lineage/api/v1/datasets.py
@router.get("/datasets", response_model=PaginatedResponse[DatasetSummary])
async def list_datasets(
    namespace: str | None = None,
    type: str | None = None,
    tag: str | None = None,
    search: str | None = None,
    freshness_status: str | None = None,
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=200),
    session: AsyncSession = Depends(get_db),
) -> PaginatedResponse[DatasetSummary]: ...

@router.get("/datasets/{dataset_id}", response_model=DatasetDetail)
async def get_dataset(
    dataset_id: UUID, session: AsyncSession = Depends(get_db)
) -> DatasetDetail: ...

@router.get("/datasets/{dataset_id}/fields", response_model=list[DatasetFieldSchema])
async def get_dataset_fields(
    dataset_id: UUID, session: AsyncSession = Depends(get_db)
) -> list[DatasetFieldSchema]: ...

# src/lineage/api/v1/lineage.py
@router.get("/lineage/{dataset_id}/upstream", response_model=LineageGraphResponse)
async def get_upstream_lineage(
    dataset_id: UUID,
    depth: int = Query(10, ge=1, le=50),
    session: AsyncSession = Depends(get_db),
) -> LineageGraphResponse: ...

@router.get("/lineage/{dataset_id}/downstream", response_model=LineageGraphResponse)
async def get_downstream_lineage(
    dataset_id: UUID,
    depth: int = Query(10, ge=1, le=50),
    session: AsyncSession = Depends(get_db),
) -> LineageGraphResponse: ...

@router.get("/lineage/column/{field_id}/upstream", response_model=ColumnLineageResponse)
async def get_column_upstream(
    field_id: UUID,
    depth: int = Query(20, ge=1, le=100),
    session: AsyncSession = Depends(get_db),
) -> ColumnLineageResponse: ...
```

```python
# src/lineage/schemas/lineage.py
from pydantic import BaseModel
from uuid import UUID

class DatasetSummary(BaseModel):
    id: UUID
    namespace: str
    name: str
    type: str
    freshness_status: str
    last_updated_at: datetime | None
    last_row_count: int | None
    tags: list[str]

class DatasetDetail(DatasetSummary):
    description: str | None
    facets: dict
    properties: dict
    current_schema: dict | None
    field_count: int
    upstream_count: int
    downstream_count: int

class LineageNodeResponse(BaseModel):
    dataset_id: UUID
    name: str
    namespace: str
    type: str
    depth: int
    freshness_status: str

class LineageEdgeResponse(BaseModel):
    source_dataset_id: UUID
    target_dataset_id: UUID
    job_name: str
    edge_type: str
    latest_run_state: str | None

class LineageGraphResponse(BaseModel):
    nodes: list[LineageNodeResponse]
    edges: list[LineageEdgeResponse]
    root_dataset_id: UUID
    direction: str
    max_depth_reached: bool

class PaginatedResponse(BaseModel):
    items: list
    total: int
    page: int
    page_size: int
    has_next: bool
```

**Testing**:
- `Integration: list_datasets_default — GET /datasets returns paginated list`
- `Integration: list_datasets_filter_namespace — ?namespace=snowflake://prod filters correctly`
- `Integration: list_datasets_search — ?search=orders matches dataset names containing "orders"`
- `Integration: get_dataset_detail — GET /datasets/{id} includes field_count and lineage counts`
- `Integration: get_dataset_404 — GET /datasets/{nonexistent_id} returns 404`
- `Integration: get_upstream_lineage — returns graph with correct depth ordering`
- `Integration: get_column_upstream — returns column transformation chain`
- `Integration: pagination — page=2, page_size=10 returns correct offset`

---

#### 3.3 — Run History & Job Detail APIs

**What**: Endpoints for viewing job details and run history with filtering by state and time range.

**Design**:

```python
# src/lineage/api/v1/jobs.py
@router.get("/jobs", response_model=PaginatedResponse[JobSummary])
async def list_jobs(
    namespace: str | None = None,
    state: str | None = None,
    search: str | None = None,
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=200),
    session: AsyncSession = Depends(get_db),
) -> PaginatedResponse[JobSummary]: ...

@router.get("/jobs/{job_id}", response_model=JobDetail)
async def get_job(job_id: UUID, session: AsyncSession = Depends(get_db)) -> JobDetail: ...

# src/lineage/api/v1/runs.py
@router.get("/jobs/{job_id}/runs", response_model=PaginatedResponse[RunSummary])
async def list_runs(
    job_id: UUID,
    state: str | None = None,
    since: datetime | None = None,
    until: datetime | None = None,
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=200),
    session: AsyncSession = Depends(get_db),
) -> PaginatedResponse[RunSummary]: ...

@router.get("/runs/{run_id}", response_model=RunDetail)
async def get_run(run_id: UUID, session: AsyncSession = Depends(get_db)) -> RunDetail: ...
```

```python
class RunSummary(BaseModel):
    id: UUID
    job_name: str
    state: str
    actual_start_time: datetime | None
    actual_end_time: datetime | None
    duration_ms: int | None

class RunDetail(RunSummary):
    facets: dict
    input_datasets: list[dict]
    output_datasets: list[dict]
    parent_run_id: UUID | None
    error_message: str | None  # extracted from errorMessage facet
```

**Testing**:
- `Integration: list_runs_by_state — ?state=FAIL returns only failed runs`
- `Integration: list_runs_by_time — ?since=2026-05-01 filters correctly`
- `Integration: get_run_detail — includes facets, input_datasets, output_datasets`
- `Integration: run_duration — duration_ms computed from actual_start and actual_end`
- `Integration: run_error_extraction — FAIL run has error_message populated from facet`

---

## Phase 4: Observability Engine

### Purpose

Implement the monitoring and anomaly detection system. After this phase, the platform monitors dataset freshness, volume, and schema changes, detects anomalies using ML-derived baselines, and surfaces priority-ranked alerts with downstream impact context. This is the core observability differentiator.

### Tasks

#### 4.1 — Monitor CRUD & Configuration

**What**: REST endpoints and service for creating, updating, and managing monitors (freshness, volume, schema, distribution, custom).

**Design**:

```python
# src/lineage/schemas/observability.py
from enum import Enum

class MonitorType(str, Enum):
    FRESHNESS = "FRESHNESS"
    VOLUME = "VOLUME"
    SCHEMA = "SCHEMA"
    DISTRIBUTION = "DISTRIBUTION"
    CUSTOM = "CUSTOM"

class MonitorStatus(str, Enum):
    OK = "OK"
    WARNING = "WARNING"
    CRITICAL = "CRITICAL"

class MonitorCreate(BaseModel):
    dataset_id: UUID
    monitor_type: MonitorType
    enabled: bool = True
    is_ml_derived: bool = False
    config: dict = Field(default_factory=dict)
    # FRESHNESS config: {"expected_interval_seconds": 3600, "tolerance_seconds": 300}
    # VOLUME config: {"min_rows": 1000, "max_rows": 500000, "baseline_model": "rolling_avg"}
    # SCHEMA config: {"alert_on": ["COLUMN_REMOVED", "TYPE_CHANGED"]}  # ignore COLUMN_ADDED
    # DISTRIBUTION config: {"columns": ["amount"], "test": "ks_test", "threshold": 0.05}

class MonitorResponse(BaseModel):
    id: UUID
    dataset_id: UUID
    dataset_name: str
    monitor_type: MonitorType
    enabled: bool
    is_ml_derived: bool
    config: dict
    baseline: dict | None
    last_evaluated_at: datetime | None
    last_status: MonitorStatus | None
    consecutive_failures: int
    created_at: datetime

# src/lineage/api/v1/monitors.py
@router.post("/monitors", response_model=MonitorResponse, status_code=201)
async def create_monitor(monitor: MonitorCreate, ...) -> MonitorResponse: ...

@router.get("/monitors", response_model=PaginatedResponse[MonitorResponse])
async def list_monitors(
    dataset_id: UUID | None = None,
    monitor_type: MonitorType | None = None,
    status: MonitorStatus | None = None,
    ...
) -> PaginatedResponse[MonitorResponse]: ...

@router.put("/monitors/{monitor_id}", response_model=MonitorResponse)
async def update_monitor(monitor_id: UUID, update: MonitorUpdate, ...) -> MonitorResponse: ...

@router.delete("/monitors/{monitor_id}", status_code=204)
async def delete_monitor(monitor_id: UUID, ...) -> None: ...

@router.post("/monitors/{monitor_id}/evaluate", response_model=MonitorEvaluationResult)
async def evaluate_monitor(monitor_id: UUID, ...) -> MonitorEvaluationResult: ...
```

**Testing**:
- `Integration: create_freshness_monitor — POST returns 201 with correct config`
- `Integration: create_volume_monitor — volume config stored in JSONB correctly`
- `Integration: list_monitors_by_dataset — filter by dataset_id returns correct monitors`
- `Integration: update_monitor_config — PUT updates config without losing baseline`
- `Integration: delete_monitor — DELETE returns 204, monitor gone`
- `Integration: create_monitor_invalid_dataset — nonexistent dataset_id returns 404`

---

#### 4.2 — Schema Change Detection Service

**What**: Detect and record schema changes by comparing incoming schema facets with stored dataset schemas.

**Design**:

```python
# src/lineage/services/schema_diff.py
from dataclasses import dataclass
from enum import Enum

class SchemaChangeType(str, Enum):
    COLUMN_ADDED = "COLUMN_ADDED"
    COLUMN_REMOVED = "COLUMN_REMOVED"
    TYPE_CHANGED = "TYPE_CHANGED"
    COLUMN_RENAMED = "COLUMN_RENAMED"
    NULLABLE_CHANGED = "NULLABLE_CHANGED"

@dataclass
class SchemaChange:
    change_type: SchemaChangeType
    field_name: str
    old_type: str | None
    new_type: str | None
    details: dict | None = None

class SchemaDiffService:
    def compute_diff(
        self, old_fields: list[dict], new_fields: list[dict]
    ) -> list[SchemaChange]:
        """
        Compare two schema field lists.
        Algorithm:
        1. Build name→field maps for old and new
        2. Fields in new but not old → COLUMN_ADDED
        3. Fields in old but not new → COLUMN_REMOVED
        4. Fields in both with different type → TYPE_CHANGED
        5. Fields in both with different nullable → NULLABLE_CHANGED
        Returns ordered list of changes.
        """

    async def detect_and_record(
        self, session: AsyncSession, dataset_id: UUID,
        old_schema: dict | None, new_schema: dict,
        triggered_by_run_id: UUID | None = None
    ) -> list[SchemaChange]:
        """
        Compute diff and persist to schema_changes table.
        Also triggers anomaly alert if COLUMN_REMOVED or TYPE_CHANGED detected.
        """
```

**Testing**:
- `Unit: no_changes — identical schemas return empty list`
- `Unit: column_added — new column detected as COLUMN_ADDED`
- `Unit: column_removed — missing column detected as COLUMN_REMOVED`
- `Unit: type_changed — VARCHAR→INT detected as TYPE_CHANGED`
- `Unit: nullable_changed — nullable true→false detected as NULLABLE_CHANGED`
- `Unit: multiple_changes — schema with 3 changes returns all 3`
- `Integration: changes_persisted — schema_changes table has row after detection`
- `Integration: alert_on_column_removed — COLUMN_REMOVED creates anomaly_alert`

---

#### 4.3 — ML-Based Anomaly Detection Engine

**What**: Implement statistical anomaly detection for freshness and volume monitors using rolling baselines and z-score detection.

**Design**:

```python
# src/lineage/services/anomaly_detection.py
from dataclasses import dataclass
import numpy as np

@dataclass
class AnomalyResult:
    is_anomaly: bool
    severity: str  # LOW, MEDIUM, HIGH, CRITICAL
    expected_value: str
    actual_value: str
    deviation_sigma: float
    method: str  # "z_score", "iqr", "prophet"

class AnomalyDetector:
    def detect_freshness_anomaly(
        self, last_arrival: datetime | None,
        expected_interval_seconds: int,
        tolerance_seconds: int,
        historical_intervals: list[float],  # seconds between arrivals
    ) -> AnomalyResult:
        """
        1. If last_arrival is None and expected_interval > 0 → CRITICAL
        2. Compute elapsed = now - last_arrival
        3. If historical_intervals has >= 10 points:
           - Compute rolling mean and stddev
           - z_score = (elapsed - mean) / stddev
           - severity: |z| > 4 → CRITICAL, > 3 → HIGH, > 2 → MEDIUM
        4. Else fall back to static threshold: elapsed > expected + tolerance → anomaly
        """

    def detect_volume_anomaly(
        self, current_row_count: int,
        historical_counts: list[int],
        config: dict,
    ) -> AnomalyResult:
        """
        1. If historical_counts has >= 10 points:
           - Compute rolling mean and stddev
           - z_score = (current - mean) / stddev
           - Severity based on z_score magnitude
        2. Else use config min_rows/max_rows as static bounds
        """

    def detect_distribution_anomaly(
        self, current_values: list[float],
        baseline_values: list[float],
        test: str = "ks_test",
        threshold: float = 0.05,
    ) -> AnomalyResult:
        """Kolmogorov-Smirnov or chi-squared test against baseline distribution."""

    def compute_severity(self, z_score: float) -> str:
        if abs(z_score) > 4.0:
            return "CRITICAL"
        elif abs(z_score) > 3.0:
            return "HIGH"
        elif abs(z_score) > 2.0:
            return "MEDIUM"
        return "LOW"
```

**Testing**:
- `Unit: freshness_normal — arrival within 1 stddev returns is_anomaly=False`
- `Unit: freshness_critical — arrival 5 stddev late returns CRITICAL`
- `Unit: freshness_no_history — falls back to static threshold`
- `Unit: freshness_never_arrived — None last_arrival returns CRITICAL`
- `Unit: volume_normal — row count within 1 stddev returns is_anomaly=False`
- `Unit: volume_drop — row count 10x below mean returns CRITICAL`
- `Unit: volume_spike — row count 5x above mean returns HIGH`
- `Unit: volume_static_fallback — <10 historical points uses min/max config`
- `Unit: distribution_ks_normal — matching distributions return is_anomaly=False`
- `Unit: distribution_ks_shifted — shifted distribution returns anomaly`

---

#### 4.4 — Monitor Evaluation Engine & Celery Tasks

**What**: Build the periodic monitor evaluation engine that runs all enabled monitors and creates anomaly alerts.

**Design**:

```python
# src/lineage/services/monitor_engine.py
class MonitorEngine:
    def __init__(
        self, anomaly_detector: AnomalyDetector,
        schema_diff: SchemaDiffService,
        impact_service: ImpactAnalysisService,
    ) -> None: ...

    async def evaluate_monitor(
        self, session: AsyncSession, monitor: Monitor
    ) -> AnomalyAlert | None:
        """
        Evaluate a single monitor:
        1. Fetch current state for the monitored dataset
        2. Fetch historical data points (last 30 days of run metrics)
        3. Call appropriate anomaly detection method based on monitor_type
        4. If anomaly detected:
           a. Compute downstream impact count via impact_service
           b. Create AnomalyAlert with severity and impact context
           c. Update monitor.last_status and consecutive_failures
           d. Update dataset.freshness_status if freshness monitor
        5. If no anomaly: update monitor.last_status = OK, reset consecutive_failures
        """

    async def evaluate_all(self, session: AsyncSession) -> dict[str, int]:
        """
        Evaluate all enabled monitors.
        Returns: {"evaluated": N, "anomalies_created": M}
        """

# src/lineage/workers/tasks/monitor_evaluation.py
from lineage.workers.celery_app import celery_app

@celery_app.task(name="evaluate_monitors")
def evaluate_monitors_task() -> dict[str, int]:
    """Celery task that runs MonitorEngine.evaluate_all()"""

# src/lineage/workers/schedules.py
celery_app.conf.beat_schedule = {
    "evaluate-monitors-every-5-minutes": {
        "task": "evaluate_monitors",
        "schedule": 300.0,  # 5 minutes
    },
}
```

**Testing**:
- `Integration: evaluate_freshness_monitor_ok — fresh dataset produces no alert`
- `Integration: evaluate_freshness_monitor_late — stale dataset creates FRESHNESS alert`
- `Integration: evaluate_volume_monitor_ok — normal row count produces no alert`
- `Integration: evaluate_volume_monitor_drop — row count drop creates VOLUME alert with severity`
- `Integration: evaluate_schema_monitor — schema change creates SCHEMA alert`
- `Integration: alert_has_impact_count — alert includes downstream_impact_count > 0`
- `Integration: consecutive_failures — 3 consecutive anomalies increments counter to 3`
- `Integration: evaluate_all — processes all enabled monitors, skips disabled`
- `Integration: celery_task_runs — evaluate_monitors_task executes successfully`

---

#### 4.5 — Alert Management API

**What**: REST endpoints for listing, acknowledging, resolving, and dismissing anomaly alerts.

**Design**:

```python
# src/lineage/api/v1/alerts.py
class AlertStatus(str, Enum):
    OPEN = "OPEN"
    ACKNOWLEDGED = "ACKNOWLEDGED"
    RESOLVED = "RESOLVED"
    FALSE_POSITIVE = "FALSE_POSITIVE"

@router.get("/alerts", response_model=PaginatedResponse[AlertResponse])
async def list_alerts(
    dataset_id: UUID | None = None,
    status: AlertStatus | None = None,
    severity: str | None = None,
    since: datetime | None = None,
    sort_by: str = Query("created_at", pattern="^(created_at|severity|downstream_count)$"),
    sort_order: str = Query("desc", pattern="^(asc|desc)$"),
    ...
) -> PaginatedResponse[AlertResponse]: ...

@router.get("/alerts/{alert_id}", response_model=AlertDetailResponse)
async def get_alert(alert_id: UUID, ...) -> AlertDetailResponse: ...

@router.patch("/alerts/{alert_id}/acknowledge")
async def acknowledge_alert(alert_id: UUID, ...) -> AlertResponse: ...

@router.patch("/alerts/{alert_id}/resolve")
async def resolve_alert(
    alert_id: UUID, resolution: AlertResolution, ...
) -> AlertResponse: ...

@router.patch("/alerts/{alert_id}/false-positive")
async def mark_false_positive(alert_id: UUID, ...) -> AlertResponse: ...

class AlertResponse(BaseModel):
    id: UUID
    dataset_id: UUID
    dataset_name: str
    monitor_type: str
    severity: str
    status: AlertStatus
    title: str
    details: dict
    downstream_impact_count: int
    created_at: datetime

class AlertDetailResponse(AlertResponse):
    downstream_datasets: list[dict]  # affected downstream datasets with hop count
    assigned_to: str | None
    resolution_notes: str | None
    resolved_at: datetime | None

class AlertResolution(BaseModel):
    resolution_notes: str
```

**Testing**:
- `Integration: list_alerts_default — returns alerts sorted by created_at desc`
- `Integration: list_alerts_by_severity — ?severity=CRITICAL returns only CRITICAL alerts`
- `Integration: list_alerts_sort_by_downstream — sort_by=downstream_count orders by impact`
- `Integration: acknowledge_alert — status changes to ACKNOWLEDGED`
- `Integration: resolve_alert — status changes to RESOLVED, resolution_notes stored`
- `Integration: false_positive — status changes to FALSE_POSITIVE`
- `Integration: alert_detail_with_downstream — includes list of affected downstream datasets`

---

## Phase 5: Impact Analysis & Schema Change Tracking

### Purpose

Build the downstream impact computation engine that traces how a change at one dataset propagates through the lineage graph. After this phase, every anomaly alert includes a ranked list of affected downstream datasets, dashboards, and models, and schema changes trigger proactive impact notifications.

### Tasks

#### 5.1 — Downstream Impact Analysis Service

**What**: Service that computes and caches the full downstream impact tree for any dataset.

**Design**:

```python
# src/lineage/services/impact_analysis.py
@dataclass
class ImpactedAsset:
    dataset_id: UUID
    dataset_name: str
    namespace: str
    asset_type: str  # DB_TABLE, STREAM, DASHBOARD, ML_MODEL
    hop_count: int
    path: list[UUID]  # dataset IDs in the path from source to this asset
    freshness_status: str
    tags: list[str]

@dataclass
class ImpactReport:
    source_dataset_id: UUID
    source_dataset_name: str
    affected_assets: list[ImpactedAsset]
    total_affected: int
    max_depth: int
    computed_at: datetime

class ImpactAnalysisService:
    async def compute_downstream_impact(
        self, session: AsyncSession, dataset_id: UUID, max_depth: int = 20
    ) -> ImpactReport:
        """
        Recursive CTE downstream traversal.
        Returns all affected datasets with hop count and path.
        Results are ordered by hop_count (closest first).
        """

    async def get_impact_summary(
        self, session: AsyncSession, dataset_id: UUID
    ) -> dict[str, int]:
        """
        Quick summary: {"total": N, "tables": M, "dashboards": K, "ml_models": L}
        Used for alert enrichment without full path computation.
        """

    async def refresh_impact_cache(
        self, session: AsyncSession, dataset_id: UUID
    ) -> None:
        """
        Recompute and cache impact for a dataset.
        Called after lineage edges are updated.
        """
```

**Testing**:
- `Integration: impact_linear_chain — A→B→C→D, impact of A returns [B(1), C(2), D(3)]`
- `Integration: impact_fan_out — A→[B,C,D], impact of A returns 3 assets all at depth 1`
- `Integration: impact_diamond — A→B, A→C, B→D, C→D, impact of A returns [B(1), C(1), D(2)]`
- `Integration: impact_cycle — cyclic graph terminates correctly`
- `Integration: impact_summary_counts — summary returns correct type counts`
- `Integration: impact_empty — dataset with no downstream returns empty list`
- `Integration: impact_max_depth — respects max_depth parameter`

---

#### 5.2 — Schema Change Impact Notifications

**What**: When a schema change is detected, automatically compute and attach downstream impact to the schema change alert.

**Design**:

```python
# Integration into schema_diff.py detect_and_record method:
async def detect_and_record(
    self, session: AsyncSession, dataset_id: UUID,
    old_schema: dict | None, new_schema: dict,
    triggered_by_run_id: UUID | None = None
) -> list[SchemaChange]:
    changes = self.compute_diff(old_fields, new_fields)
    if not changes:
        return []

    # Persist schema change record
    change_record = SchemaChangeModel(
        dataset_id=dataset_id,
        changes=[asdict(c) for c in changes],
        old_schema_hash=hash_schema(old_schema),
        new_schema_hash=hash_schema(new_schema),
        triggered_by_run_id=triggered_by_run_id,
    )
    session.add(change_record)

    # For breaking changes, create alert with impact
    breaking = [c for c in changes if c.change_type in (
        SchemaChangeType.COLUMN_REMOVED, SchemaChangeType.TYPE_CHANGED
    )]
    if breaking:
        impact = await self.impact_service.compute_downstream_impact(session, dataset_id)
        alert = AnomalyAlert(
            dataset_id=dataset_id,
            monitor_id=schema_monitor_id,
            severity=self._schema_severity(breaking, impact.total_affected),
            status="OPEN",
            title=f"Schema change: {len(breaking)} breaking changes detected",
            details={
                "changes": [asdict(c) for c in changes],
                "downstream_affected": impact.total_affected,
                "downstream_assets": [
                    {"name": a.dataset_name, "hops": a.hop_count}
                    for a in impact.affected_assets[:20]
                ],
            },
        )
        session.add(alert)

    return changes
```

**Testing**:
- `Integration: breaking_change_creates_alert — COLUMN_REMOVED creates alert with impact details`
- `Integration: non_breaking_change_no_alert — COLUMN_ADDED does not create alert`
- `Integration: severity_scales_with_impact — 50 downstream assets → CRITICAL severity`
- `Integration: alert_details_has_downstream_list — details JSONB contains affected asset names`
- `Integration: type_change_alert — TYPE_CHANGED creates alert`

---

## Phase 6: Authentication & Authorization

### Purpose

Add JWT-based authentication, API key support, and role-based access control. After this phase, all API endpoints require authentication, users have scoped permissions, and the audit log captures who did what.

### Tasks

#### 6.1 — User Management & API Key Service

**What**: Service for user CRUD, API key generation, and JWT token creation/validation.

**Design**:

```python
# src/lineage/services/auth.py
class AuthService:
    async def create_user(
        self, session: AsyncSession, email: str, display_name: str,
        role_names: list[str] = ["viewer"]
    ) -> User: ...

    async def create_api_key(
        self, session: AsyncSession, user_id: UUID,
        name: str, scopes: list[str], expires_in_days: int | None = None
    ) -> tuple[str, ApiToken]:
        """
        Generate API key, hash it with bcrypt, store hash.
        Returns (plaintext_key, token_record).
        Plaintext key is only returned once.
        """

    async def validate_api_key(
        self, session: AsyncSession, key: str
    ) -> User | None:
        """
        Hash the provided key, look up in api_tokens table.
        Returns user if valid and not expired, None otherwise.
        Updates last_used_at.
        """

    def create_jwt(self, user: User) -> str:
        """Create JWT with user_id, email, roles, and expiration."""

    def validate_jwt(self, token: str) -> dict:
        """Validate and decode JWT. Raises on expiry or invalid signature."""
```

**Testing**:
- `Unit: create_jwt_valid — JWT contains correct claims and is verifiable`
- `Unit: validate_jwt_expired — expired JWT raises ExpiredTokenError`
- `Unit: validate_jwt_invalid — tampered JWT raises InvalidTokenError`
- `Integration: create_api_key — key stored as bcrypt hash, plaintext returned once`
- `Integration: validate_api_key_valid — correct key returns user`
- `Integration: validate_api_key_expired — expired key returns None`
- `Integration: validate_api_key_wrong — incorrect key returns None`

---

#### 6.2 — Auth Middleware & RBAC Enforcement

**What**: FastAPI middleware and dependencies that enforce authentication on all endpoints and check role-based permissions.

**Design**:

```python
# src/lineage/api/middleware/auth.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, APIKeyHeader

bearer_scheme = HTTPBearer(auto_error=False)
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def get_current_user(
    bearer: HTTPAuthorizationCredentials | None = Security(bearer_scheme),
    api_key: str | None = Security(api_key_header),
    session: AsyncSession = Depends(get_db),
) -> User:
    """
    Authentication priority:
    1. Bearer JWT token
    2. X-API-Key header
    3. If neither → 401 Unauthorized
    """

def require_permission(permission: str):
    """
    Dependency factory that checks the current user has a role
    granting the specified permission.
    Permissions: lineage:read, lineage:write, monitors:manage,
                 alerts:manage, compliance:read, compliance:export,
                 admin:users, admin:roles
    """
    async def check(user: User = Depends(get_current_user)) -> User:
        if not user_has_permission(user, permission):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return check
```

**Testing**:
- `Integration: unauthenticated_request — no auth header returns 401`
- `Integration: valid_jwt_request — Bearer JWT grants access`
- `Integration: valid_api_key_request — X-API-Key grants access`
- `Integration: insufficient_permission — viewer calling admin endpoint returns 403`
- `Integration: admin_full_access — admin role can access all endpoints`
- `Integration: namespace_scoped_role — user with namespace-scoped role can only access that namespace`

---

#### 6.3 — Audit Logging Middleware

**What**: Middleware that logs all state-changing API calls to the audit_log table.

**Design**:

```python
# src/lineage/api/middleware/audit.py
class AuditMiddleware:
    async def __call__(self, request: Request, call_next):
        response = await call_next(request)
        if request.method in ("POST", "PUT", "PATCH", "DELETE"):
            await self._log_action(request, response)
        return response

    async def _log_action(self, request: Request, response: Response) -> None:
        """
        Insert into audit_log:
        - user_id from request.state.user
        - action derived from method (POST→CREATE, PUT→UPDATE, etc.)
        - entity_type and entity_id parsed from URL path
        - ip_address from request.client.host
        """
```

**Testing**:
- `Integration: post_creates_audit — POST /monitors creates audit_log entry with action=CREATE`
- `Integration: get_no_audit — GET requests do not create audit entries`
- `Integration: audit_has_user — audit entry has correct user_id`
- `Integration: audit_has_ip — audit entry has client IP address`

---

## Phase 7: Frontend — Lineage Visualisation

### Purpose

Build the React frontend with interactive lineage graph visualisation, dataset browsing, and alert dashboard. After this phase, users can visually explore lineage graphs, drill into datasets, and monitor alert status through a web UI.

### Tasks

#### 7.1 — Frontend Scaffolding & API Client

**What**: Set up the React/TypeScript/Vite project and auto-generate an API client from the OpenAPI spec.

**Design**:

```json
// frontend/package.json (key dependencies)
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^7.0.0",
    "@xyflow/react": "^12.0.0",
    "zustand": "^5.0.0",
    "@tanstack/react-query": "^5.60.0",
    "lucide-react": "^0.460.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.3.0",
    "openapi-typescript-codegen": "^0.29.0",
    "@playwright/test": "^1.49.0"
  }
}
```

```typescript
// frontend/src/api/client.ts (auto-generated structure)
// Generated via: npx openapi-typescript-codegen --input http://localhost:8000/api/v1/openapi.json --output src/api/generated

export interface DatasetSummary {
  id: string;
  namespace: string;
  name: string;
  type: string;
  freshness_status: string;
  last_updated_at: string | null;
  tags: string[];
}

export interface LineageGraphResponse {
  nodes: LineageNodeResponse[];
  edges: LineageEdgeResponse[];
  root_dataset_id: string;
  direction: "upstream" | "downstream";
  max_depth_reached: boolean;
}
```

**Testing**:
- `Unit: api_client_generation — openapi-typescript-codegen generates types without errors`
- `E2E: app_loads — navigating to / renders the main layout`
- `E2E: api_proxy — frontend API calls reach the backend through Vite proxy`

---

#### 7.2 — Interactive Lineage Graph Component

**What**: Build the React Flow-based lineage graph with custom dataset and job nodes, zoom/pan, depth controls, and column-level lineage toggle.

**Design**:

```typescript
// frontend/src/components/lineage/LineageGraph.tsx
interface LineageGraphProps {
  datasetId: string;
  direction: "upstream" | "downstream" | "both";
  maxDepth: number;
  showColumnLineage: boolean;
}

// Custom node types
interface DatasetNodeData {
  name: string;
  namespace: string;
  type: "DB_TABLE" | "STREAM" | "FILE";
  freshnessStatus: "OK" | "LATE" | "CRITICAL" | "UNKNOWN";
  rowCount: number | null;
  lastUpdated: string | null;
  isRoot: boolean;
}

interface JobNodeData {
  name: string;
  latestRunState: "COMPLETE" | "FAIL" | "RUNNING" | null;
}

// Layout: Dagre left-to-right for upstream, right-to-left for downstream
// Node colors: green=OK, yellow=LATE, red=CRITICAL, gray=UNKNOWN
// Edge styles: solid=active, dashed=stale (no recent runs)
```

**Testing**:
- `E2E: lineage_graph_renders — graph renders with correct number of nodes and edges`
- `E2E: lineage_node_click — clicking a dataset node navigates to dataset detail`
- `E2E: lineage_depth_control — changing depth slider updates visible graph`
- `E2E: lineage_freshness_colors — CRITICAL dataset shows red node, OK shows green`
- `E2E: lineage_direction_toggle — switching upstream/downstream reloads graph`

---

#### 7.3 — Dataset Browser & Detail Pages

**What**: Dataset list page with filtering/search and dataset detail page showing schema, lineage, monitors, and alerts.

**Design**:

```typescript
// frontend/src/pages/DatasetDetailPage.tsx
// Tabs: Overview | Schema | Lineage | Monitors | Alerts | History
// Overview: name, namespace, type, description, tags, facets, freshness status
// Schema: table of fields with name, type, nullable, PK, description
// Lineage: embedded LineageGraph component
// Monitors: list of monitors with status indicators
// Alerts: filtered alert feed for this dataset
// History: recent run history for jobs that write to this dataset
```

**Testing**:
- `E2E: dataset_list_renders — page shows paginated dataset list`
- `E2E: dataset_search — typing in search bar filters results`
- `E2E: dataset_detail_tabs — all tabs render without errors`
- `E2E: dataset_schema_table — schema tab shows correct columns`
- `E2E: dataset_lineage_tab — lineage tab renders graph`

---

#### 7.4 — Alert Dashboard & Monitor Overview

**What**: Real-time alert feed with severity filtering, alert detail view, and monitor health overview.

**Design**:

```typescript
// frontend/src/pages/AlertsPage.tsx
// Layout:
// - Top: summary cards (CRITICAL count, HIGH count, OPEN count)
// - Filters: severity, status, monitor_type, dataset, date range
// - Alert list: severity icon, title, dataset name, time ago, impact count badge
// - Clicking alert opens AlertDetailPanel:
//   - Full details, downstream impact list, resolution form
//   - Actions: Acknowledge, Resolve (with notes), Mark False Positive

// frontend/src/pages/MonitorsPage.tsx
// - Monitor list grouped by dataset
// - Status indicator (green/yellow/red) per monitor
// - Quick actions: enable/disable, evaluate now
// - Coverage summary: % of datasets with monitors
```

**Testing**:
- `E2E: alert_dashboard_renders — shows summary cards and alert list`
- `E2E: alert_filter_severity — filtering by CRITICAL shows only critical alerts`
- `E2E: alert_acknowledge — clicking acknowledge updates status`
- `E2E: alert_resolve — entering resolution notes and resolving works`
- `E2E: monitor_overview — shows monitor count and status distribution`

---

## Phase 8: Connectors — dbt, Airflow, Snowflake

### Purpose

Build connectors that actively pull metadata from the three most common data stack components. After this phase, the system can ingest lineage from dbt artifacts, Airflow DAG metadata, and Snowflake query history without requiring those tools to push OpenLineage events.

### Tasks

#### 8.1 — Connector Base Interface

**What**: Define the abstract connector interface and registration system.

**Design**:

```python
# src/lineage/connectors/base.py
from abc import ABC, abstractmethod

class ConnectorConfig(BaseModel):
    connector_type: str
    name: str
    enabled: bool = True
    schedule_seconds: int = 300  # polling interval
    credentials: dict = Field(default_factory=dict)
    settings: dict = Field(default_factory=dict)

class BaseConnector(ABC):
    def __init__(self, config: ConnectorConfig) -> None:
        self.config = config

    @abstractmethod
    async def test_connection(self) -> bool:
        """Verify credentials and connectivity."""

    @abstractmethod
    async def sync(self, session: AsyncSession) -> SyncResult:
        """
        Pull metadata and ingest as OpenLineage events.
        Returns SyncResult with counts of entities synced.
        """

    @abstractmethod
    def get_supported_facets(self) -> list[str]:
        """List of OpenLineage facet types this connector produces."""

@dataclass
class SyncResult:
    jobs_synced: int
    datasets_synced: int
    runs_synced: int
    lineage_edges_synced: int
    errors: list[str]
    duration_seconds: float
```

**Testing**:
- `Unit: connector_config_validation — invalid config raises ValidationError`
- `Unit: sync_result_fields — SyncResult has all required fields`

---

#### 8.2 — dbt Artifact Connector

**What**: Ingest lineage from dbt manifest.json, catalog.json, and run_results.json artifacts.

**Design**:

```python
# src/lineage/connectors/dbt.py
class DbtConnector(BaseConnector):
    """
    Reads dbt artifacts and converts to OpenLineage events:
    - manifest.json → jobs (models), datasets (sources + models), lineage edges (ref/source dependencies)
    - catalog.json → dataset fields (column names, types, descriptions)
    - run_results.json → runs with timing and status
    - Column-level lineage from manifest node column metadata if available

    Config:
    - credentials.artifact_path: local path or S3/GCS URI to dbt artifacts
    - credentials.api_token: dbt Cloud API token (optional, for Cloud artifact retrieval)
    - settings.project_name: dbt project name for namespace
    - settings.target: dbt target environment name
    """

    async def sync(self, session: AsyncSession) -> SyncResult:
        """
        1. Load manifest.json
        2. For each node in manifest.nodes:
           a. Create job (model name)
           b. Create output dataset (model's relation)
           c. For each depends_on.node → create input dataset + lineage edge
        3. Load catalog.json
        4. For each node in catalog.nodes:
           a. Sync dataset fields (column name, type, description)
        5. Load run_results.json
        6. For each result → create run with timing and status
        """
```

**Testing**:
- `Integration (fixture): dbt_manifest_sync — fixture manifest.json produces correct jobs and lineage`
- `Integration (fixture): dbt_catalog_sync — fixture catalog.json produces correct dataset fields`
- `Integration (fixture): dbt_run_results_sync — fixture run_results.json produces correct runs`
- `Integration (fixture): dbt_column_lineage — manifest with column metadata creates column edges`
- `Unit: dbt_ref_parsing — ref('model_name') correctly maps to dataset`
- `Unit: dbt_source_parsing — source('source', 'table') correctly maps to dataset`

---

#### 8.3 — Airflow Metadata Connector

**What**: Ingest DAG and task instance metadata from Airflow's REST API.

**Design**:

```python
# src/lineage/connectors/airflow.py
class AirflowConnector(BaseConnector):
    """
    Pulls metadata from Airflow REST API (v2):
    - /api/v1/dags → jobs
    - /api/v1/dags/{dag_id}/dagRuns → runs
    - /api/v1/dags/{dag_id}/tasks → task dependencies for lineage edges
    - /api/v1/eventLogs → OpenLineage events if airflow-openlineage provider is installed

    Config:
    - credentials.base_url: Airflow webserver URL
    - credentials.username / credentials.password: Basic auth
    - credentials.api_token: Alternative Bearer token auth
    - settings.dag_filter: regex pattern to filter which DAGs to sync
    """

    async def sync(self, session: AsyncSession) -> SyncResult:
        """
        1. List DAGs matching dag_filter
        2. For each DAG:
           a. Create job for the DAG
           b. Fetch recent DAG runs → create runs
           c. Fetch task instances for dependency mapping
        3. If OpenLineage events available in event logs, prefer those
        """
```

**Testing**:
- `Integration (mocked API): airflow_dag_sync — mocked /dags response creates correct jobs`
- `Integration (mocked API): airflow_run_sync — mocked /dagRuns response creates correct runs`
- `Integration (mocked API): airflow_auth — Basic auth header sent correctly`
- `Unit: dag_filter — regex filter excludes non-matching DAGs`

---

#### 8.4 — Snowflake Metadata Connector

**What**: Ingest table metadata and query-based lineage from Snowflake's INFORMATION_SCHEMA and ACCOUNT_USAGE views.

**Design**:

```python
# src/lineage/connectors/snowflake.py
class SnowflakeConnector(BaseConnector):
    """
    Pulls metadata from Snowflake:
    - INFORMATION_SCHEMA.TABLES → datasets
    - INFORMATION_SCHEMA.COLUMNS → dataset fields
    - ACCOUNT_USAGE.ACCESS_HISTORY → lineage edges (which queries read/wrote which tables)
    - ACCOUNT_USAGE.QUERY_HISTORY → runs

    Config:
    - credentials.account: Snowflake account identifier
    - credentials.user: Snowflake username
    - credentials.password / credentials.private_key_path: auth
    - settings.warehouse: Snowflake warehouse
    - settings.databases: list of databases to scan
    - settings.schemas: list of schemas to include (default: all)
    """

    async def sync(self, session: AsyncSession) -> SyncResult:
        """
        1. Query INFORMATION_SCHEMA.TABLES for each database/schema
        2. For each table → upsert dataset and dataset fields
        3. Query ACCESS_HISTORY for recent read/write operations
        4. For each access record → create lineage edge from sources to targets
        5. Query QUERY_HISTORY for recent queries → create runs
        """
```

**Testing**:
- `Integration (mocked): snowflake_table_sync — mocked INFORMATION_SCHEMA creates datasets`
- `Integration (mocked): snowflake_column_sync — mocked COLUMNS creates dataset fields`
- `Integration (mocked): snowflake_lineage — mocked ACCESS_HISTORY creates lineage edges`
- `Unit: snowflake_access_history_parsing — correctly extracts source/target tables`
- `Unit: snowflake_database_filter — only scans configured databases`

---

## Phase 9: AI-Augmented Features

### Purpose

Add the AI-native differentiators: natural-language lineage exploration, AI-generated impact narration, and automated freshness SLA recommendations. These features close the gap between raw lineage capture and actionable insight.

### Tasks

#### 9.1 — Natural-Language Lineage Explorer

**What**: LLM-powered endpoint that answers lineage questions in plain language, e.g., "Where does the revenue column in the exec dashboard come from?"

**Design**:

```python
# src/lineage/services/nl_explorer.py
from litellm import acompletion

class NLLineageExplorer:
    SYSTEM_PROMPT = """You are a data lineage expert. You have access to a lineage graph 
    and must answer questions about data origins, transformations, and dependencies.
    
    When answering:
    1. Trace the lineage path from the queried asset back to its sources
    2. Describe each transformation step in plain language
    3. Include dataset names, column names, and transformation types
    4. Mention any data quality issues or freshness concerns along the path
    
    Respond in structured markdown with the lineage path clearly shown."""

    async def explore(
        self, session: AsyncSession, question: str, namespace: str | None = None
    ) -> NLExplorationResult:
        """
        1. Parse question to identify target dataset/column
        2. Query lineage graph for relevant upstream/downstream data
        3. Format lineage context as structured text
        4. Call LLM with context + question
        5. Return annotated response with referenced dataset IDs
        """

    async def _build_context(
        self, session: AsyncSession, dataset_name: str, field_name: str | None
    ) -> str:
        """
        Build lineage context string for LLM:
        - Upstream datasets with transformation descriptions
        - Column-level lineage if field_name specified
        - Freshness status and recent anomalies along the path
        """

@dataclass
class NLExplorationResult:
    answer: str  # Markdown-formatted answer
    referenced_datasets: list[dict]  # [{id, name, namespace}]
    lineage_path: list[dict]  # [{from, to, transformation}]
    confidence: str  # "high", "medium", "low"

# src/lineage/api/v1/search.py
@router.post("/explore", response_model=NLExplorationResponse)
async def explore_lineage(
    query: NLExplorationRequest,
    user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_db),
) -> NLExplorationResponse: ...

class NLExplorationRequest(BaseModel):
    question: str = Field(..., max_length=1000)
    namespace: str | None = None
```

**Testing**:
- `Integration (mocked LLM): nl_simple_question — "Where does revenue come from?" returns traced path`
- `Integration (mocked LLM): nl_column_question — column-specific question returns column lineage`
- `Integration (mocked LLM): nl_no_match — question about nonexistent dataset returns helpful error`
- `Unit: context_builder — builds correct lineage context string from graph data`
- `Unit: dataset_name_extraction — parses dataset name from natural-language question`

---

#### 9.2 — AI Impact Narration

**What**: Generate plain-language descriptions of downstream impact when a schema change or anomaly is detected.

**Design**:

```python
# src/lineage/services/impact_narration.py
class ImpactNarrator:
    SYSTEM_PROMPT = """You are a data reliability expert. Given a data issue 
    (schema change, freshness delay, volume anomaly) and its downstream impact,
    generate a concise, actionable summary for the data team.
    
    Include:
    1. What happened (the root issue)
    2. What is affected (downstream datasets, dashboards, reports)
    3. Business impact (which teams/reports are at risk)
    4. Recommended action
    
    Be specific about dataset names and transformation paths.
    Keep it under 200 words."""

    async def narrate_impact(
        self, alert: AnomalyAlert, impact: ImpactReport
    ) -> str:
        """
        Build context from alert details and impact report,
        call LLM to generate plain-language narration.
        """

    async def narrate_schema_change(
        self, changes: list[SchemaChange], impact: ImpactReport
    ) -> str:
        """Generate narration specifically for schema changes."""
```

**Testing**:
- `Integration (mocked LLM): narrate_freshness — produces readable summary mentioning affected assets`
- `Integration (mocked LLM): narrate_schema_change — mentions specific columns changed and downstream dashboards`
- `Unit: context_formatting — impact report formatted correctly for LLM input`

---

#### 9.3 — Automated Freshness SLA Recommendations

**What**: Analyse historical pipeline behaviour to recommend freshness monitor thresholds per dataset.

**Design**:

```python
# src/lineage/services/sla_recommender.py
class FreshnessSLARecommender:
    async def recommend(
        self, session: AsyncSession, dataset_id: UUID, lookback_days: int = 30
    ) -> SLARecommendation:
        """
        1. Query run history for the dataset's producing jobs (last N days)
        2. Compute inter-arrival intervals
        3. Calculate p50, p95, p99 of intervals
        4. Recommend expected_interval = p50, tolerance = p95 - p50
        5. Flag if arrival pattern is irregular (high coefficient of variation)
        """

@dataclass
class SLARecommendation:
    dataset_id: UUID
    dataset_name: str
    recommended_interval_seconds: int  # p50 of historical intervals
    recommended_tolerance_seconds: int  # p95 - p50
    confidence: str  # "high" (>30 data points), "medium" (10-30), "low" (<10)
    data_points: int
    p50_seconds: float
    p95_seconds: float
    p99_seconds: float
    is_irregular: bool  # CV > 0.5
    notes: str

# src/lineage/api/v1/monitors.py
@router.get("/monitors/recommend/{dataset_id}", response_model=SLARecommendation)
async def recommend_freshness_sla(
    dataset_id: UUID, lookback_days: int = Query(30, ge=7, le=90), ...
) -> SLARecommendation: ...
```

**Testing**:
- `Unit: regular_schedule — hourly arrivals recommend ~3600s interval with low tolerance`
- `Unit: irregular_schedule — highly variable arrivals flag is_irregular=True`
- `Unit: insufficient_data — <10 data points return confidence="low"`
- `Unit: p95_tolerance — tolerance equals p95 minus p50`
- `Integration: recommend_from_db — computes recommendation from actual run history`

---

## Phase 10: Compliance Reporting

### Purpose

Implement automated compliance report generation for EU AI Act, DORA, and GDPR Article 30, using captured lineage metadata. After this phase, compliance officers can generate regulator-ready documentation directly from the platform.

### Tasks

#### 10.1 — GDPR Article 30 Records of Processing

**What**: Generate GDPR Records of Processing Activities from dataset metadata and lineage, tracking personal data flows across systems.

**Design**:

```python
# src/lineage/services/compliance_reporter.py
class GDPRReporter:
    async def generate_ropa(
        self, session: AsyncSession, namespace: str | None = None
    ) -> ROPAReport:
        """
        1. Find all datasets with properties containing pii_classification or personal_data tags
        2. For each PII dataset, trace downstream lineage to find all systems receiving personal data
        3. Generate Article 30-compliant records:
           - Processing purpose (from dataset properties)
           - Categories of data subjects and personal data
           - Recipients (downstream systems)
           - Transfers to third countries
           - Retention periods
        """

@dataclass
class ROPAReport:
    generated_at: datetime
    scope: str
    processing_activities: list[ProcessingActivity]
    total_personal_data_datasets: int
    total_downstream_flows: int

@dataclass
class ProcessingActivity:
    dataset_name: str
    namespace: str
    purpose: str
    legal_basis: str
    data_categories: list[str]
    recipients: list[str]  # downstream dataset namespaces
    retention_days: int | None
    cross_border_transfers: list[str]

# API endpoint
@router.get("/compliance/gdpr/ropa", response_model=ROPAReport)
async def generate_gdpr_ropa(
    namespace: str | None = None,
    format: str = Query("json", pattern="^(json|csv|pdf)$"),
    ...
) -> ROPAReport: ...
```

**Testing**:
- `Integration: ropa_with_pii_datasets — generates report with correct processing activities`
- `Integration: ropa_downstream_tracing — includes recipient systems from lineage`
- `Integration: ropa_no_pii — empty datasets return empty report`
- `Integration: ropa_csv_export — CSV format includes all required columns`

---

#### 10.2 — DORA Incident Reporting

**What**: Generate DORA-compliant incident reports from anomaly alerts, with tamper-evident hashing.

**Design**:

```python
class DORAReporter:
    async def create_incident_report(
        self, session: AsyncSession, alert_id: UUID
    ) -> DORAIncidentReport:
        """
        1. Load alert with full details
        2. Compute downstream impact
        3. Generate structured incident report per DORA Article 19
        4. Hash report content (SHA-256) for tamper evidence
        5. Store as compliance_record with type=DORA_INCIDENT
        """

    async def list_incidents(
        self, session: AsyncSession,
        since: datetime | None = None, until: datetime | None = None
    ) -> list[DORAIncidentReport]: ...

@dataclass
class DORAIncidentReport:
    incident_id: UUID
    incident_type: str
    severity: str
    description: str
    affected_datasets: list[str]
    root_cause_description: str
    detection_time: datetime
    resolution_time: datetime | None
    data_hash: str  # SHA-256 for tamper evidence
    reported_to_authority: bool
```

**Testing**:
- `Integration: dora_report_from_alert — creates incident report from CRITICAL alert`
- `Integration: dora_tamper_hash — report hash matches SHA-256 of content`
- `Integration: dora_list_incidents — lists incidents within date range`
- `Unit: dora_hash_verification — modified content produces different hash`

---

#### 10.3 — EU AI Act Documentation

**What**: Generate technical documentation required by the EU AI Act for high-risk AI systems, linking AI models to their training data lineage.

**Design**:

```python
class AIActReporter:
    async def generate_ai_system_doc(
        self, session: AsyncSession, system_name: str
    ) -> AIActDocumentation:
        """
        1. Find datasets tagged as AI training/validation data
        2. Trace full upstream lineage of training datasets
        3. Document data origins, transformations, and quality metrics
        4. Generate Articles 9-17 compliant documentation:
           - Data governance practices
           - Data preparation, labelling, and validation
           - Data quality measures
           - Bias examination and mitigation
        """

@dataclass
class AIActDocumentation:
    system_name: str
    risk_category: str
    training_datasets: list[DatasetLineageDoc]
    validation_datasets: list[DatasetLineageDoc]
    data_quality_summary: dict
    generated_at: datetime
    content_hash: str

@dataclass
class DatasetLineageDoc:
    dataset_name: str
    origin: str  # root source in lineage
    transformations: list[str]  # transformation descriptions along path
    quality_metrics: dict  # row count, null rates, freshness
```

**Testing**:
- `Integration: ai_act_doc — generates documentation for system with training datasets`
- `Integration: ai_act_lineage_trace — traces training data back to sources`
- `Integration: ai_act_quality_metrics — includes quality metrics for training data`

---

## Phase 11: Webhook Events & Notifications

### Purpose

Add outbound webhook event delivery and notification integrations (Slack, email). After this phase, external systems can subscribe to lineage and alert events, and teams receive proactive notifications.

### Tasks

#### 11.1 — Webhook Event System

**What**: Configurable webhook delivery for lineage events, anomaly alerts, and schema changes.

**Design**:

```python
# src/lineage/services/webhooks.py
class WebhookEventType(str, Enum):
    ALERT_CREATED = "alert.created"
    ALERT_RESOLVED = "alert.resolved"
    SCHEMA_CHANGED = "schema.changed"
    LINEAGE_UPDATED = "lineage.updated"
    MONITOR_FAILED = "monitor.failed"

class WebhookConfig(BaseModel):
    id: UUID
    url: str
    events: list[WebhookEventType]
    secret: str  # HMAC signing secret
    enabled: bool = True
    retry_count: int = 3

class WebhookService:
    async def deliver(
        self, event_type: WebhookEventType, payload: dict
    ) -> None:
        """
        1. Find all webhook configs subscribed to this event type
        2. For each: sign payload with HMAC-SHA256, POST to URL
        3. On failure: enqueue retry via Celery with exponential backoff
        """

    def sign_payload(self, payload: bytes, secret: str) -> str:
        """HMAC-SHA256 signature for webhook payload verification."""
```

**Testing**:
- `Integration (mocked HTTP): webhook_delivery — alert.created event POSTed to webhook URL`
- `Integration (mocked HTTP): webhook_signature — X-Signature header contains valid HMAC`
- `Integration (mocked HTTP): webhook_retry — failed delivery retried 3 times`
- `Unit: webhook_hmac — signature matches expected HMAC-SHA256`
- `Integration: webhook_filter — webhook subscribed to alert.created does not receive schema.changed`

---

#### 11.2 — Slack & Email Notifications

**What**: Send alert notifications to Slack channels and email recipients based on alert severity and dataset ownership.

**Design**:

```python
# src/lineage/services/notifications.py
class SlackNotifier:
    async def send_alert(self, alert: AnomalyAlert, channel: str) -> None:
        """
        Format alert as Slack Block Kit message:
        - Severity badge (emoji: red_circle, orange_circle, yellow_circle)
        - Alert title
        - Dataset name and namespace
        - Impact count
        - Link to alert detail in UI
        """

class EmailNotifier:
    async def send_alert(self, alert: AnomalyAlert, recipients: list[str]) -> None:
        """Send HTML-formatted alert email with impact summary."""

class NotificationRouter:
    async def route_alert(self, alert: AnomalyAlert) -> None:
        """
        Determine notification targets based on:
        1. Dataset ownership (notify dataset owner)
        2. Severity-based routing (CRITICAL → on-call channel)
        3. Notification preferences per user
        """
```

**Testing**:
- `Integration (mocked Slack): slack_alert — CRITICAL alert sends message with correct formatting`
- `Integration (mocked Email): email_alert — alert email sent to dataset owner`
- `Unit: notification_routing — CRITICAL routes to on-call, MEDIUM routes to team channel`
- `Unit: slack_block_format — message contains severity, title, and impact count`

---

## Phase 12: Visual Lineage Editor & Advanced Features

### Purpose

Add the visual no-code lineage editor for manual corrections, and remaining backlog features including full-text search and lineage gap detection. After this phase, the platform reaches feature parity with the v1.1 scope defined in features.md.

### Tasks

#### 12.1 — Visual No-Code Lineage Editor

**What**: Frontend component that allows users to manually add, edit, or remove lineage edges through a drag-and-drop interface.

**Design**:

```typescript
// frontend/src/components/lineage/LineageEditor.tsx
interface LineageEditorProps {
  datasetId: string;
  onSave: (changes: LineageEditBatch) => void;
}

interface LineageEditBatch {
  added_edges: Array<{
    source_dataset_id: string;
    target_dataset_id: string;
    edge_type: string;
    description: string;
  }>;
  removed_edges: Array<{ edge_id: string }>;
  updated_edges: Array<{
    edge_id: string;
    edge_type?: string;
    description?: string;
  }>;
}

// Backend API
// POST /api/v1/lineage/manual
// Accepts LineageEditBatch, creates/updates/deletes edges
// All manual edits tagged with properties: {"source": "manual", "edited_by": user_id}
```

**Testing**:
- `E2E: add_manual_edge — drag from dataset A to dataset B creates edge`
- `E2E: remove_manual_edge — selecting and deleting edge removes it`
- `E2E: manual_edge_tagged — manual edge has source="manual" in properties`
- `Integration: manual_edit_audit — manual edits create audit log entries`

---

#### 12.2 — Full-Text Search

**What**: Search across datasets, jobs, fields, and tags using PostgreSQL full-text search.

**Design**:

```python
# src/lineage/services/search.py
class SearchService:
    async def search(
        self, session: AsyncSession, query: str,
        entity_types: list[str] | None = None,
        namespace: str | None = None,
        limit: int = 50,
    ) -> list[SearchResult]:
        """
        PostgreSQL ts_vector search across:
        - datasets.name, datasets.description
        - jobs.name, jobs.description
        - dataset_fields.name, dataset_fields.description
        - tags (array contains)
        
        Results ranked by ts_rank with entity type boosting
        (datasets ranked higher than fields).
        """

@dataclass
class SearchResult:
    entity_type: str  # "dataset", "job", "field"
    entity_id: UUID
    name: str
    namespace: str
    description: str | None
    rank: float
    highlight: str  # search term highlighted in context

# API
@router.get("/search", response_model=list[SearchResult])
async def search(
    q: str = Query(..., min_length=2, max_length=200),
    type: str | None = None,
    namespace: str | None = None,
    limit: int = Query(50, ge=1, le=200),
    ...
) -> list[SearchResult]: ...
```

**Testing**:
- `Integration: search_by_name — "orders" matches datasets named "raw.orders" and "analytics.orders_daily"`
- `Integration: search_by_description — searching description text returns matching results`
- `Integration: search_type_filter — type=dataset excludes job results`
- `Integration: search_ranking — exact name match ranks higher than partial description match`
- `Integration: search_empty — no matches returns empty list`

---

#### 12.3 — Lineage Gap Detection

**What**: AI-powered analysis that identifies datasets and columns with missing lineage metadata and prioritises coverage gaps.

**Design**:

```python
# src/lineage/services/gap_detection.py
class LineageGapDetector:
    async def detect_gaps(
        self, session: AsyncSession, namespace: str | None = None
    ) -> GapReport:
        """
        1. Find datasets with no incoming lineage edges (potential source gaps)
        2. Find datasets with no outgoing lineage edges and high downstream usage (orphaned producers)
        3. Find fields with no column-level lineage but present in table-level lineage
        4. Rank gaps by downstream impact (more downstream dependents → higher priority)
        5. Generate prioritised gap report
        """

@dataclass
class LineageGap:
    dataset_id: UUID
    dataset_name: str
    gap_type: str  # "no_upstream", "no_downstream", "missing_column_lineage"
    priority: str  # "high", "medium", "low"
    downstream_count: int
    fields_without_lineage: list[str]

@dataclass
class GapReport:
    total_gaps: int
    high_priority: int
    gaps: list[LineageGap]
    coverage_percentage: float  # % of datasets with complete lineage
```

**Testing**:
- `Integration: detect_source_gap — dataset with no upstream flagged`
- `Integration: detect_column_gap — table with table-level but no column-level lineage flagged`
- `Integration: priority_ranking — dataset with 20 downstream ranked higher than dataset with 2`
- `Integration: coverage_calculation — correct percentage computed`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Skeleton            ─── required by everything
    │
Phase 2: OpenLineage Event Ingestion             ─── requires Phase 1
    │
Phase 3: Lineage Graph Queries & API             ─── requires Phase 2
    │
    ├── Phase 4: Observability Engine             ─── requires Phase 3
    │       │
    │       └── Phase 5: Impact Analysis          ─── requires Phase 4
    │
    ├── Phase 6: Authentication & Authorization   ─── requires Phase 3 (can parallel Phase 4)
    │
    ├── Phase 7: Frontend — Lineage Visualisation ─── requires Phase 3 (can parallel Phases 4-6)
    │
    └── Phase 8: Connectors                       ─── requires Phase 2 (can parallel Phases 3-7)
         │
Phase 9: AI-Augmented Features                    ─── requires Phases 3, 4, 5
    │
Phase 10: Compliance Reporting                    ─── requires Phases 3, 5, 6
    │
Phase 11: Webhook Events & Notifications          ─── requires Phases 4, 6
    │
Phase 12: Visual Lineage Editor & Advanced        ─── requires Phases 3, 7
```

**Parallelism opportunities:**
- Phases 4, 6, 7, 8 can all begin concurrently after Phase 3
- Phase 8 (Connectors) can begin after Phase 2, running in parallel with Phase 3
- Phase 7 (Frontend) can develop against mock API data while Phases 4-6 are in progress
- Phases 9, 10, 11, 12 depend on earlier phases but are independent of each other

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with production-quality code.
2. All unit tests pass with >90% line coverage for new code.
3. All integration tests pass against a real PostgreSQL database (test container).
4. Ruff linting passes with zero errors.
5. mypy strict mode passes with zero errors.
6. Docker build succeeds for all targets (api, worker).
7. `docker compose up` starts all services and the health endpoint returns 200.
8. Alembic migration applies cleanly on a fresh database and upgrades from the previous version.
9. All new API endpoints appear in the auto-generated OpenAPI 3.1 specification.
10. All new API endpoints have request/response Pydantic models with field descriptions.
11. All new configuration options have defaults and are documented in `.env.example`.
12. Phase functionality works end-to-end (e.g., ingest event → query lineage → see result).
13. No regressions in previously passing tests.
