# Elevator & Vertical Transport Management — Phased Development Plan

> Project: `242-elevator-vertical-transport-management` · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.
> Source artefacts: `README.md`, `research.md`, `features.md`, `standards.md`, `data-model-suggestion-1.md` (chosen baseline), `data-model-suggestion-2.md`, `data-model-suggestion-3.md`, `data-model-suggestion-4.md`.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | Python 3.12 | Domain demands strong libraries for IoT protocol bridges (BACnet, Modbus, OPC-UA), ML/anomaly detection on telemetry, NLP incident classification, and PDF certificate generation. Python has mature, well-maintained libraries for every one of these (`bacpypes3`, `pymodbus`, `asyncua`, `scikit-learn`, `transformers`, `weasyprint`). |
| API framework | FastAPI 0.110+ | Native async (required for WebSocket telemetry + long-running OEM API polls), automatic OpenAPI 3.1 generation (required per standards.md), Pydantic v2 validation, first-class dependency injection for multi-tenant scoping. |
| ASGI server | Uvicorn behind Gunicorn (UvicornWorker) | Production-grade async serving; horizontal scaling per container. |
| Database | PostgreSQL 16 | Required by data-model-suggestion-1: native range partitioning (telemetry, audit_log), JSONB (audit values), GIN indexes, recursive CTEs (jurisdiction hierarchy), `gen_random_uuid()`. ARRAY types used in schema. |
| Time-series storage | TimescaleDB extension on PostgreSQL 16 | Telemetry table is partitioned by month in the chosen data model; TimescaleDB hypertables give automatic partition management, continuous aggregates (1-min/1-hour/1-day rollups for dashboards), and compression for old chunks — keeps the operational footprint single-database. |
| ORM / migrations | SQLAlchemy 2.x (async) + Alembic | Async ORM matches FastAPI async; Alembic migrations are auditable and roll forward/back cleanly. |
| Validation / schemas | Pydantic v2 | Same model serves request validation, response shaping, and JSONB document schemas. |
| Task queue | Arq (Redis-backed async task queue) | Lightweight, asyncio-native, fits FastAPI ecosystem; needed for OEM webhook ingestion, regulatory change monitoring, scheduled compliance recompute, AI inference jobs, and notification dispatch. |
| Cache & pub/sub | Redis 7 | Doubles as Arq broker, JWT denylist, idempotency-key store, and pub/sub fan-out for live dashboards. |
| Object storage | S3-compatible (MinIO for self-hosted, AWS S3 for cloud) | Inspection photos, signed PDF certificates, work order attachments. |
| Authentication | OAuth 2.1 + OIDC (Authlib) with JWT bearer tokens; password hashing via Argon2id | OWASP API Security Top 10 compliant; integrates with enterprise IdPs (Azure AD, Okta) and AHJ federation. |
| Authorisation | Casbin (PostgreSQL adapter) | Scoped RBAC matches the `user_role.scope_type`/`scope_id` pattern in the data model; rules are auditable and tenant-isolated. |
| IoT protocol adapters | `bacpypes3` (BACnet/IP), `pymodbus` (Modbus TCP), `asyncua` (OPC UA), `paho-mqtt` (MQTT bridge) | Required by features.md "Should-have" IoT ingestion; all open-source, async-friendly. |
| AI/LLM | Anthropic Claude (Opus + Sonnet) via official SDK; locally hostable fallback (Ollama with `llama3.1:8b` for air-gapped sites) | Used for NLP incident classification, regulatory change impact assessment, and inspection narrative summarisation. |
| ML (anomaly detection) | `scikit-learn` (Isolation Forest + STL decomposition baseline), `river` (online learning for sensor drift) | Required for predictive maintenance scoring. No GPU dependency; runs in CPU containers. |
| PDF generation | WeasyPrint + Jinja2 | Inspection certificates, AHJ filing packets. Produces tamper-evident PDFs (hash stored per `inspection.certificate_hash`). |
| Frontend | Next.js 15 (App Router) + TypeScript 5 + React 19 | Required for: contractor dashboard, AHJ read-only portal, building-manager portal. App Router supports server components for fast initial loads on slower jurisdiction-office connections. |
| UI components | Tailwind CSS 4 + shadcn/ui + TanStack Table v8 | Dense compliance-calendar grids and unit registers benefit from headless tables; shadcn/ui gives consistent, accessible components without licence risk. |
| Mobile (technicians) | Capacitor 6 wrapping a React (Vite) PWA | Single TypeScript codebase deploys as iOS, Android, and web. Offline-capable via Service Worker + IndexedDB (Dexie). Camera, GPS, QR-scan via Capacitor plugins. Matches features.md "offline-capable mobile app" requirement. |
| Public fault-report page | Static Next.js page served at `/fr/<qr_token>` | No app install required (per ElevatorPlus pattern). |
| Map / geo | MapLibre GL (open-source, OpenMapTiles styles) | Portfolio map view; no Mapbox token / pricing dependency. |
| Containerisation | Docker + docker-compose (dev), Helm chart (k8s prod) | Self-hosted deployment is a stated mode. |
| Observability | OpenTelemetry SDK + Prometheus + Grafana + Loki | Required for IoT-heavy system; OTel auto-instruments FastAPI, SQLAlchemy, Redis, Arq. |
| Testing | pytest + pytest-asyncio + httpx; Playwright for e2e; Vitest for frontend unit | Standard for the chosen stack. |
| Linting / formatting | Ruff (lint + format), Mypy (strict), Black-style via Ruff | Single tool replaces flake8/isort/black; Mypy enforces type discipline. |
| Frontend tooling | ESLint, TypeScript strict, Prettier | Standard. |
| Package manager | `uv` (Python), `pnpm` (JavaScript/TypeScript) | `uv` is dramatically faster than pip and produces reproducible lockfiles; `pnpm` matches Next.js best practice. |
| Secrets | `.env` for dev; `SOPS` + age for committed encrypted secrets; AWS Secrets Manager / Vault for prod | Standard zero-trust pattern. |
| CI/CD | GitHub Actions: lint → typecheck → unit → integration → docker build → e2e → push image | Standard. Production deploy via Helm; preview environments via docker-compose-on-PR. |
| Licence | Apache 2.0 | Permissive; allows enterprise adoption while keeping derivative integrations open. |

### Chosen Data Model

Adopt **Data Model Suggestion 1 (Entity-Centric Normalised Relational)** as the baseline schema. Rationale:

- Regulatory audit demands (ASME A17.1, EN 81, ADA, NFPA 72) are best served by strong referential integrity.
- The 38-table footprint is large but tractable with Alembic and is the same shape used by analogous CMMS platforms.
- We borrow two ideas from other suggestions:
  - From Suggestion 2 (Event Sourcing): the `audit_log` table is treated as an append-only event log with `old_values`/`new_values` JSONB, satisfying tamper-evident audit requirements without a parallel event store.
  - From Suggestion 3 (Hybrid JSONB): three JSONB extension columns are added on `equipment_unit`, `inspection_checklist_item`, and `incident` (`oem_metadata`, `code_specific_fields`, `ai_extracted_data`) to absorb jurisdiction- and OEM-specific variability without monthly migrations.
- Suggestion 4 (Graph) capabilities are deferred — PostgreSQL recursive CTEs over the existing FK structure cover the jurisdiction hierarchy and portfolio→building→unit traversal needed for v1.

### Project Structure

```
elevator-vt-platform/
├── README.md
├── LICENSE                              # Apache 2.0
├── pyproject.toml                       # uv-managed, ruff/mypy/pytest config
├── uv.lock
├── .env.example
├── .pre-commit-config.yaml
├── docker-compose.yml                   # Postgres+Timescale, Redis, MinIO, services
├── docker-compose.test.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.iot-gateway
├── helm/
│   └── elevator-vt/                     # Helm chart for k8s deployment
├── docs/
│   ├── architecture.md
│   ├── adr/                             # Architecture Decision Records
│   ├── compliance/
│   │   ├── asme-a17-1-mapping.md
│   │   ├── en-81-mapping.md
│   │   └── nfpa-72-mapping.md
│   └── api/                             # generated OpenAPI YAML
├── alembic/
│   ├── env.py
│   └── versions/                        # one migration per phase
├── src/
│   └── evt/                             # "evt" = elevator vertical transport
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory
│       ├── settings.py                  # Pydantic Settings, env-driven
│       ├── deps.py                      # FastAPI dependency providers (db, current_user, tenant_ctx)
│       ├── db/
│       │   ├── base.py                  # SQLAlchemy declarative base, async session
│       │   ├── models/                  # ORM models (one file per domain area)
│       │   │   ├── tenant.py
│       │   │   ├── rbac.py
│       │   │   ├── jurisdiction.py
│       │   │   ├── building.py
│       │   │   ├── unit.py
│       │   │   ├── component.py
│       │   │   ├── compliance.py
│       │   │   ├── inspection.py
│       │   │   ├── deficiency.py
│       │   │   ├── permit.py
│       │   │   ├── contract.py
│       │   │   ├── work_order.py
│       │   │   ├── pm_schedule.py
│       │   │   ├── sensor.py
│       │   │   ├── telemetry.py         # hypertable + alert
│       │   │   ├── incident.py
│       │   │   ├── qr_fault.py
│       │   │   ├── audit.py
│       │   │   └── notification.py
│       │   ├── seeds/                   # ASME A17.1, EN 81 reference data
│       │   │   ├── code_editions.json
│       │   │   ├── compliance_templates.json
│       │   │   ├── jurisdictions_us.json
│       │   │   ├── jurisdictions_eu.json
│       │   │   └── component_types.json
│       │   └── repositories/            # data access layer
│       ├── schemas/                     # Pydantic v2 request/response models
│       │   ├── tenant.py
│       │   ├── unit.py
│       │   ├── inspection.py
│       │   └── ...
│       ├── api/
│       │   ├── v1/
│       │   │   ├── router.py            # mounts all routers
│       │   │   ├── auth.py
│       │   │   ├── tenants.py
│       │   │   ├── buildings.py
│       │   │   ├── units.py
│       │   │   ├── components.py
│       │   │   ├── compliance.py
│       │   │   ├── inspections.py
│       │   │   ├── deficiencies.py
│       │   │   ├── permits.py
│       │   │   ├── work_orders.py
│       │   │   ├── pm_schedules.py
│       │   │   ├── contracts.py
│       │   │   ├── sensors.py
│       │   │   ├── telemetry.py
│       │   │   ├── alerts.py
│       │   │   ├── incidents.py
│       │   │   ├── qr_fault.py
│       │   │   ├── reports.py
│       │   │   ├── ahj_portal.py        # read-only AHJ endpoints
│       │   │   └── public_fault.py      # unauthenticated QR submission
│       │   └── webhooks/
│       │       ├── otis.py
│       │       ├── kone.py
│       │       └── tk.py
│       ├── services/                    # business logic, no FastAPI imports
│       │   ├── compliance_engine.py     # calendar generation
│       │   ├── permit_filer.py
│       │   ├── certificate.py           # PDF + hash
│       │   ├── work_order_dispatcher.py
│       │   ├── pm_runner.py
│       │   ├── notification_dispatcher.py
│       │   └── risk_scorer.py
│       ├── adapters/                    # external integrations
│       │   ├── oem/
│       │   │   ├── base.py              # AbstractOEMAdapter
│       │   │   ├── otis_one.py
│       │   │   ├── kone_247.py
│       │   │   ├── tk_max.py
│       │   │   └── schindler_ahead.py
│       │   ├── iot/
│       │   │   ├── bacnet.py
│       │   │   ├── modbus.py
│       │   │   ├── opcua.py
│       │   │   └── mqtt.py
│       │   ├── ahj/
│       │   │   └── filing.py            # state/city permit-filing connectors
│       │   ├── notifications/
│       │   │   ├── email_ses.py
│       │   │   ├── sms_twilio.py
│       │   │   └── push_fcm.py
│       │   └── storage/
│       │       └── s3.py
│       ├── ai/
│       │   ├── llm.py                   # provider-agnostic LLM client
│       │   ├── prompts/                 # versioned prompt templates
│       │   │   ├── incident_classify.txt
│       │   │   ├── regulatory_impact.txt
│       │   │   └── inspection_summary.txt
│       │   ├── classifier.py            # NLP incident classifier
│       │   ├── anomaly.py               # Isolation Forest + STL
│       │   ├── risk.py                  # portfolio risk scoring
│       │   └── regwatch.py              # regulatory change monitor
│       ├── workers/
│       │   ├── arq_settings.py
│       │   ├── tasks/
│       │   │   ├── compliance.py        # nightly recompute
│       │   │   ├── pm.py                # PM trigger sweep
│       │   │   ├── notifications.py
│       │   │   ├── iot_ingest.py
│       │   │   ├── anomaly.py
│       │   │   ├── permit_filing.py
│       │   │   └── regwatch.py
│       │   └── cron.py                  # registered schedules
│       ├── iot_gateway/                 # standalone process, runs on-prem
│       │   ├── __main__.py
│       │   ├── config.py
│       │   └── relay.py                 # bridges BACnet/Modbus/OPCUA → cloud HTTPS
│       ├── security/
│       │   ├── jwt.py
│       │   ├── password.py              # Argon2id
│       │   ├── casbin_model.conf
│       │   └── tenancy.py               # row-level tenant guard
│       ├── audit/
│       │   ├── recorder.py              # SQLAlchemy event listeners → audit_log
│       │   └── hashchain.py             # tamper-evident chain hash per tenant
│       ├── reporting/
│       │   ├── certificate_pdf.py
│       │   ├── ahj_packet_pdf.py
│       │   └── analytics_queries.py     # MTTR, PM completion, etc.
│       └── utils/
│           ├── time.py                  # ISO 8601 helpers
│           ├── idempotency.py
│           └── pagination.py
├── tests/
│   ├── conftest.py                      # test DB, factories
│   ├── factories/                       # factory_boy
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                        # SARIF-style golden files: sample inspections, sensor traces
├── web/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── app/
│   │   │   ├── (auth)/sign-in/page.tsx
│   │   │   ├── (dashboard)/portfolio/page.tsx
│   │   │   ├── (dashboard)/buildings/[id]/page.tsx
│   │   │   ├── (dashboard)/units/[id]/page.tsx
│   │   │   ├── (dashboard)/compliance/calendar/page.tsx
│   │   │   ├── (dashboard)/work-orders/page.tsx
│   │   │   ├── (dashboard)/inspections/page.tsx
│   │   │   ├── (dashboard)/sensors/page.tsx
│   │   │   ├── (dashboard)/alerts/page.tsx
│   │   │   ├── (ahj)/portal/page.tsx     # AHJ read-only portal
│   │   │   ├── (public)/fr/[token]/page.tsx  # public QR fault page
│   │   │   └── api/                     # BFF route handlers if needed
│   │   ├── components/
│   │   ├── lib/
│   │   │   ├── api-client.ts            # generated from OpenAPI
│   │   │   └── auth.ts
│   │   └── hooks/
│   └── playwright/                      # e2e specs
├── mobile/
│   ├── package.json
│   ├── capacitor.config.ts
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Login.tsx
│   │   │   ├── WorkOrderList.tsx
│   │   │   ├── WorkOrderDetail.tsx
│   │   │   ├── InspectionForm.tsx
│   │   │   ├── QrScan.tsx
│   │   │   └── OfflineQueue.tsx
│   │   ├── db/                          # Dexie schema
│   │   └── sync/                        # background sync engine
│   └── ios/  android/                   # Capacitor platform projects
└── scripts/
    ├── seed_dev.py                       # demo tenant, building, units, inspections
    ├── regenerate_openapi.py
    └── load_jurisdiction_data.py
```

---

## Phase 1: Foundation, Multi-Tenancy & Auth

### Purpose
Establish the repo, tooling, Docker dev environment, PostgreSQL+TimescaleDB schema for tenancy and RBAC, and the authentication system. After this phase a developer can `docker compose up`, sign in, receive a JWT, and call a protected `/healthz` endpoint scoped to their tenant. Nothing domain-specific exists yet — this is the chassis.

### Tasks

#### 1.1 — Repository & Tooling Bootstrap

**What**: Initialise the monorepo with `uv`-managed Python project, `pnpm` workspaces for `web` and `mobile`, pre-commit hooks, Ruff/Mypy/pytest configuration, and CI.

**Design**:
- `pyproject.toml` declares Python 3.12, dependencies (`fastapi[standard]`, `sqlalchemy[asyncio]`, `alembic`, `asyncpg`, `pydantic[email]`, `pydantic-settings`, `argon2-cffi`, `python-jose[cryptography]`, `authlib`, `casbin`, `casbin_async_sqlalchemy_adapter`, `arq`, `redis`, `httpx`, `opentelemetry-distro`).
- Ruff config: `line-length=100`, `target-version="py312"`, enable `E,F,I,N,UP,B,SIM,RUF`.
- Mypy: `strict = true`; per-file ignores only with documented justification.
- Pre-commit: ruff, mypy, eslint, prettier, yamllint, shellcheck.
- GitHub Actions workflow `.github/workflows/ci.yml`:
  - Job `lint-py`: `uv run ruff check . && uv run ruff format --check .`
  - Job `type-py`: `uv run mypy src/`
  - Job `test-py`: postgres+redis services, `uv run pytest -q`
  - Job `lint-web`: `pnpm -C web lint && pnpm -C web typecheck`
  - Job `docker`: `docker build` for each Dockerfile
- `docker-compose.yml` services: `postgres` (`timescale/timescaledb:2-pg16`), `redis:7`, `minio`, `api`, `worker`, optional `iot_gateway`.
- `.env.example` enumerates every env var the app reads.

**Testing**:
- Unit: `pytest --collect-only` returns 0 errors on empty repo.
- CI: dry-run `act` execution passes on every Ruff/Mypy/test job.
- Integration: `docker compose up -d postgres redis minio` succeeds; `docker compose down -v` cleans state.

#### 1.2 — Configuration & Settings

**What**: Implement `evt.settings.Settings` (Pydantic Settings) reading from environment with strong typing and validators.

**Design**:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="EVT_", env_file=".env")

    environment: Literal["dev", "test", "staging", "prod"] = "dev"
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"

    database_url: PostgresDsn
    database_pool_size: int = 10
    database_max_overflow: int = 20

    redis_url: RedisDsn

    s3_endpoint: str | None = None
    s3_bucket: str = "evt-attachments"
    s3_access_key: SecretStr
    s3_secret_key: SecretStr
    s3_region: str = "us-east-1"

    jwt_secret: SecretStr
    jwt_alg: Literal["HS256", "RS256"] = "HS256"
    jwt_access_ttl_seconds: int = 900
    jwt_refresh_ttl_seconds: int = 60 * 60 * 24 * 14

    argon2_time_cost: int = 3
    argon2_memory_cost: int = 65536
    argon2_parallelism: int = 4

    cors_allow_origins: list[AnyHttpUrl] = []

    feature_iot_ingest: bool = False
    feature_ai_classify: bool = False
    feature_permit_filing: bool = False

    llm_provider: Literal["anthropic", "ollama"] = "anthropic"
    llm_model: str = "claude-opus-4-7"
    llm_api_key: SecretStr | None = None
```

**Testing**:
- Unit: missing required env var → `ValidationError` mentioning the field.
- Unit: `EVT_JWT_ALG=invalid` → `ValidationError`.
- Unit: defaults applied when optional vars absent.

#### 1.3 — Database Foundation & Alembic

**What**: Configure async SQLAlchemy engine, declarative base, and Alembic with auto-generation.

**Design**:
- `evt.db.base.Base` extends `DeclarativeBase` with naming convention for FK/index/unique constraints (Alembic-stable).
- All models inherit a `TimestampMixin` providing `created_at`, `updated_at` (server defaults, `onupdate=now()`).
- `evt.db.session.async_session_factory` returns `AsyncSession`.
- Alembic `env.py` uses async engine and imports all model modules from `evt.db.models`.
- First migration `0001_init.py`: enables `pgcrypto` (for `gen_random_uuid()`) and the `timescaledb` extension.

**Testing**:
- Integration (real Postgres): apply `0001_init`, then downgrade, then re-apply — exit 0 each time.
- Unit: model with no `__tablename__` raises an assertion at registry validation time.

#### 1.4 — Tenant, Organisation, User, RBAC Models

**What**: Implement the eight tables from the "Core Identity & Multi-Tenancy" and "RBAC" sections of data-model-suggestion-1 as SQLAlchemy models and Alembic migration `0002_identity_rbac.py`.

**Design**:
- Models exactly mirror the DDL in `data-model-suggestion-1.md` (tenant, organisation, app_user, role, permission, role_permission, user_role).
- The composite PK on `user_role` uses `COALESCE(scope_id, '00000000-0000-0000-0000-000000000000')` — implemented as a `text("...")` expression in the migration.
- Seed five system roles per tenant: `admin`, `dispatcher`, `technician`, `inspector`, `building_manager`, `ahj_auditor` — seeded on tenant creation via a service.
- Define a fixed permission catalogue (≈40 permissions) loaded from `evt/db/seeds/permissions.json`.

**Testing**:
- Unit: creating a tenant via service auto-seeds six roles.
- Unit: `UserRole.grant(user, role, scope=Building(id=...))` writes scope_type='building'.
- Integration: query "all admins of tenant X" returns expected users.

#### 1.5 — Password Hashing & JWT

**What**: Implement Argon2id password hashing and JWT issuance/verification.

**Design**:
- `evt.security.password`: `hash_password(plain) -> str`, `verify_password(plain, hash) -> bool`. Uses `argon2-cffi` with parameters from `Settings`. Includes pepper concatenation (`jwt_secret` reused as pepper).
- `evt.security.jwt`:
  ```python
  class TokenClaims(BaseModel):
      sub: UUID          # user_id
      tid: UUID          # tenant_id
      oid: UUID | None   # organisation_id
      roles: list[str]
      scopes: list[ScopeEntry]
      exp: int
      iat: int
      jti: UUID
      typ: Literal["access", "refresh"]
  ```
- Refresh tokens are stored in Redis `denylist:{jti}` on rotation/logout.

**Testing**:
- Unit: hash → verify roundtrip.
- Unit: tampered hash fails verification.
- Unit: expired token raises `TokenExpiredError`.
- Unit: `jti` in denylist raises `TokenRevokedError`.
- Unit: tokens are RFC 7519 JWS-compliant (decode with `python-jose` returns expected claims).

#### 1.6 — Authentication API

**What**: Endpoints for sign-up, sign-in, refresh, sign-out, and OIDC code-exchange callback.

**Design**:
- `POST /v1/auth/sign-in` body `{email, password}` → `{access_token, refresh_token, expires_in}`.
- `POST /v1/auth/refresh` body `{refresh_token}` → new pair; old refresh is denylisted.
- `POST /v1/auth/sign-out` (auth required) → denylists current refresh.
- `GET /v1/auth/oidc/{provider}/login` redirects to IdP authorise URL.
- `GET /v1/auth/oidc/{provider}/callback` handles code exchange, provisions `app_user` if first sign-in.
- All endpoints subject to per-IP rate limit (5/min sign-in, 30/min refresh) via Redis sliding window.

**Testing**:
- Integration: valid credentials → 200 with tokens; invalid password → 401 after consistent-time hash compare.
- Integration: refresh with denylisted token → 401.
- Integration (mocked OIDC server): callback creates user with tenant scoping = tenant tied to email domain or invitation token.
- E2E: 5 failed sign-ins from one IP → 6th returns 429.

#### 1.7 — Tenant Context Middleware & Dependency

**What**: Every authenticated request resolves a `TenantContext(tenant_id, user_id, roles, scopes)` available via FastAPI `Depends`.

**Design**:
- Middleware extracts JWT from `Authorization: Bearer …`, verifies, builds `TenantContext`, stores on `request.state.ctx`.
- `Depends(get_current_ctx)` returns it; `Depends(require_perm("unit.read"))` validates against Casbin policy.
- All SQLAlchemy queries pass through a `tenant_filter(stmt, ctx)` helper that injects `WHERE tenant_id = :tid` automatically; a `@no_tenant_filter` marker is required to opt out.

**Testing**:
- Unit: missing Authorization → 401.
- Unit: tampered JWT signature → 401.
- Unit: cross-tenant query attempt (tenant A user querying tenant B unit) returns 0 rows.
- Integration: superadmin marker bypasses tenant filter only when explicitly opted-in.

#### 1.8 — Audit Log Infrastructure

**What**: Implement append-only audit log with partitioning and SQLAlchemy ORM-level capture.

**Design**:
- Migration creates `audit_log` partitioned by `created_at` (monthly), with default partition.
- `evt.audit.recorder` registers SQLAlchemy `after_insert`, `after_update`, `after_delete` listeners on all `Base` subclasses except `AuditLog`, `TelemetryReading`, `Notification`.
- Diff is computed via `inspect(target).attrs` and stored as `old_values`/`new_values` JSONB.
- A monthly Arq cron job creates the next month's partition 7 days in advance.
- `evt.audit.hashchain`: each audit row stores `prev_hash` and `row_hash = sha256(prev_hash || canonical_json(row))`, providing tamper detection per tenant chain.

**Testing**:
- Unit: updating a `Unit.name` produces one audit_log row with both old and new value.
- Unit: deleting an `Inspection` produces an audit row before the DB row is removed.
- Integration: tampering with one audit row breaks the chain hash for all subsequent rows in that tenant.
- Integration: the cron creates next-month partition on the 25th of the current month.

---

## Phase 2: Asset Register & Compliance Reference Data

### Purpose
Make the platform usable for cataloguing real elevator portfolios. Implements buildings, equipment units (the shaft-as-entity model), floors, groups, components, and seeds the global jurisdiction + ASME A17.1 / EN 81 / ISO 8100 reference data. After this phase a tenant can model their entire portfolio and components and a compliance calendar can be auto-generated in Phase 3.

### Tasks

#### 2.1 — Jurisdiction, Code Edition & Compliance Templates

**What**: Implement the four tables under "Jurisdiction & Regulatory Framework" plus seed data for: ISO 3166-1 countries; US states + 25 major US cities; EU member states; ASME A17.1 editions 2016/2019/2022/2025; ASME A17.3-2023; EN 81-20:2020, EN 81-50:2020; ISO 8100-1:2019, 8100-2:2019.

**Design**:
- Migration `0003_jurisdiction.py` creates the four tables.
- `evt/db/seeds/code_editions.json`: array of `{standard_family, edition_year, edition_label, effective_date}`.
- `evt/db/seeds/jurisdictions_us.json`, `jurisdictions_eu.json`: hierarchical `[{name, type, country_code, subdivision_code, parent_slug}]`.
- `evt/db/seeds/compliance_templates.json` — example entry:
  ```json
  {
    "code_edition": {"family": "ASME_A17_1", "year": 2025},
    "requirement_code": "CAT1",
    "name": "Category 1 Annual Test",
    "description": "Annual no-load test per ASME A17.1 Part 8.6.4.20",
    "equipment_types": ["elevator"],
    "frequency_months": 12,
    "applies_to_existing": true
  }
  ```
- `scripts/load_jurisdiction_data.py` reads seeds and upserts on `(standard_family, edition_year)` and (jurisdiction natural key).
- A `POST /v1/admin/seeds/refresh` superadmin endpoint reruns the loader.

**Testing**:
- Unit: `Jurisdiction.find_by_iso("US-NY")` returns the New York state row.
- Integration: running seed loader twice is idempotent (no duplicate rows).
- Integration: `CodeEdition.superseded_by` chain — A17.1-2022 → A17.1-2025 resolves correctly.
- Fixture-based: golden file containing every state's adopted A17.1 edition as of 2026-01-01 round-trips through the loader.

#### 2.2 — Portfolio & Building Models

**What**: `portfolio` and `building` tables plus CRUD API.

**Design**:
- Models per `data-model-suggestion-1.md`.
- Pydantic schemas: `BuildingCreate`, `BuildingUpdate`, `BuildingRead`.
- API:
  - `GET /v1/portfolios`
  - `POST /v1/portfolios`
  - `GET /v1/buildings?portfolio_id=&jurisdiction_id=`
  - `POST /v1/buildings`
  - `PATCH /v1/buildings/{id}`
  - `DELETE /v1/buildings/{id}` (soft delete via `status='archived'`; full hard delete requires `admin.purge`)
- Address normalisation via libpostal optional integration (deferred — keep raw fields for v1).
- On create, if `jurisdiction_id` is null and address has `country_code` + `state_province` + `city`, attempt auto-resolve from jurisdiction table.

**Testing**:
- Unit: portfolio-less building allowed (`portfolio_id` nullable).
- Integration: building with jurisdiction_id of EU jurisdiction stores correctly.
- Integration: cross-tenant building access returns 404.
- E2E: create portfolio → create two buildings under it → list buildings filtered by portfolio.

#### 2.3 — Equipment Unit (Shaft) Model

**What**: `equipment_unit`, `unit_floor_mapping`, `elevator_group`, `elevator_group_member` tables and CRUD API; absorb OEM-specific extension fields via JSONB.

**Design**:
- Model adds `oem_metadata JSONB NOT NULL DEFAULT '{}'::jsonb` and `code_specific_fields JSONB NOT NULL DEFAULT '{}'::jsonb` (deviation from suggestion-1 in line with suggestion-3).
- `applicable_code_edition_id` defaults via service to "current adoption for the building's jurisdiction" at create time.
- Pydantic schema `UnitCreate` requires `building_id`, `unit_number`, `equipment_type`; everything else optional.
- API:
  - `GET /v1/buildings/{id}/units`
  - `POST /v1/buildings/{id}/units`
  - `GET /v1/units/{id}`
  - `PATCH /v1/units/{id}`
  - `POST /v1/units/{id}/floors` (bulk replace mapping)
  - `POST /v1/buildings/{id}/groups`
  - `PATCH /v1/groups/{id}/members` (set membership)
- Constraint enforced in service: `equipment_type='escalator'` units cannot have `unit_floor_mapping` (use `num_stops=2` instead).

**Testing**:
- Unit: creating a unit without `unit_number` → 422.
- Unit: duplicate `(building_id, unit_number)` → 409.
- Unit: hydraulic elevator sub_type implies `machine_room_location='basement'` default.
- Integration: assigning a unit from building A to a group in building B → 400.
- Fixture: load a 30-unit "Class A Tower" sample portfolio; verify counts.

#### 2.4 — Component Catalogue & Instances

**What**: `component_type` reference table seeded with 18 standard component types from features.md / standards.md; `component` table for per-unit instances; CRUD endpoints.

**Design**:
- Seed component types: traction_machine, hydraulic_pump, door_operator, governor, brake, controller, drive_inverter, car_frame, car_door, hoistway_door, guide_rails, safety_device, buffer, suspension_rope, suspension_belt, sheave, counterweight, emergency_phone.
- Each typed with category, `is_safety_critical`, typical lifespan.
- API:
  - `GET /v1/units/{id}/components`
  - `POST /v1/units/{id}/components`
  - `PATCH /v1/components/{id}`
  - `POST /v1/components/{id}/cycle-count` (increment / set)
- Cycle count increments bound to source: `manual`, `iot_sensor`, `inspection` — recorded in audit.

**Testing**:
- Unit: seeded component_type rows exist after `load_jurisdiction_data.py`.
- Integration: setting `component.status='failed'` triggers a notification to the dispatcher (covered in Phase 9).
- Integration: cycle-count increment from IoT source updates `updated_at` and audit log.

---

## Phase 3: Compliance Engine & Inspection Lifecycle

### Purpose
Deliver the heart of the product: jurisdiction-aware compliance calendar generation, inspection capture, deficiency tracking, and permit lifecycle. After this phase a contractor sees every upcoming Cat 1 / Cat 5 / annual / fire-recall test across their entire portfolio with correct due dates derived from each unit's specific jurisdiction adoption and code edition.

### Tasks

#### 3.1 — Compliance Obligation Generation Service

**What**: `evt.services.compliance_engine` produces and maintains `compliance_obligation` rows for every unit based on its jurisdiction's adopted code edition.

**Design**:
- Pure function `compute_obligations(unit, jurisdiction_adoption, last_completed: dict[str, date]) -> list[ComputedObligation]`:
  1. Resolve all `compliance_requirement_template`s for the `code_edition_id`.
  2. Filter by `equipment_types ∋ unit.equipment_type` and `applies_to_existing` vs install date.
  3. For each template, derive `next_due_date = max(unit.installation_date, last_completed[code] or installation_date) + frequency_months`.
  4. Status: `upcoming` if `next_due_date > today + advance_notice_days`, `due_soon` if within window, `overdue` if `< today`.
- Service `ComplianceEngine.recompute_for_unit(unit_id)` upserts obligations (PK candidate = `(unit_id, template_id)`).
- `ComplianceEngine.recompute_tenant(tenant_id)` iterates units; called nightly by Arq cron at 02:00 tenant-local.
- Triggered on: unit create/update, building jurisdiction change, jurisdiction code adoption change.

**Testing**:
- Unit: NYC building, A17.1-2025 unit installed 2024-06-01, no Cat 1 done → next_due = 2025-06-01, status `overdue` on 2026-05-29.
- Unit: Munich building, EN 81-20 unit → only EN 81 templates resolve; no A17.1 obligations created.
- Unit: changing jurisdiction adoption from A17.1-2022 to A17.1-2025 → obligations regenerated with new template set.
- Unit: existing-equipment-only template (A17.3) skipped for new install.
- Integration: cron job recomputes a 500-unit tenant in <5s.

#### 3.2 — Compliance Calendar API

**What**: Endpoints to query upcoming/overdue obligations across a portfolio.

**Design**:
- `GET /v1/compliance/calendar` query params:
  - `from`, `to` (ISO 8601 dates; default = today, today+90)
  - `building_id`, `portfolio_id`, `jurisdiction_id` (filters)
  - `status` (comma-separated)
  - `requirement_code`
  - `group_by=building|unit|requirement` (default `unit`)
- Response is paginated, ordered by `next_due_date ASC`.
- `GET /v1/units/{id}/obligations` returns per-unit calendar.
- `POST /v1/obligations/{id}/waive` body `{reason, until_date}` → status `waived`; audit-logged.

**Testing**:
- Integration: calendar returns expected obligations after Phase 3.1 seeding.
- Integration: waive cannot be performed without `compliance.waive` permission.
- Performance: calendar over 5,000-unit portfolio for 365-day window returns in <500ms with proper indexes.

#### 3.3 — Inspection Capture

**What**: `inspection`, `inspection_checklist_item`, `deficiency` tables + capture API + tamper-evident certificate hash.

**Design**:
- `inspection_checklist_item.ai_extracted_data JSONB` added (extension from suggestion-3) for AI-augmented capture later.
- Inspection state machine: `draft → in_progress → completed → certified` (with `cancelled` terminal); enforced in service.
- API:
  - `POST /v1/units/{id}/inspections` body includes inspection_type, inspector, obligation_id (optional).
  - `POST /v1/inspections/{id}/checklist` bulk upsert items.
  - `POST /v1/inspections/{id}/deficiencies` create deficiency rows.
  - `POST /v1/inspections/{id}/complete` transitions to completed; computes `certificate_hash` over canonicalised JSON of all inspection fields + checklist items + deficiencies (SHA-512).
  - `POST /v1/inspections/{id}/certificate` generates PDF (Phase 3.5) and writes `certificate_number`.
- Upon completion: link to obligation and update `compliance_obligation.last_completed_date`; the next obligation recomputation runs.

**Testing**:
- Unit: state-machine guard rejects `completed → draft`.
- Unit: completing without obligation linkage allowed; with obligation, obligation updates.
- Integration: completing an inspection with mismatched canonical hash (simulated by manual SQL update of one checklist item) is detected by `verify_certificate_hash()`.
- Fixture: golden inspection JSON produces identical hash on every CI run.

#### 3.4 — Deficiency & Permit Lifecycle

**What**: Deficiency tracking and permit (operating, installation) lifecycle.

**Design**:
- Deficiency state machine: `open → in_progress → resolved` or `→ waived`.
- Auto-create corrective work order (Phase 4) when deficiency severity = `critical` and `auto_dispatch_critical=true` in tenant settings.
- Permit API:
  - `POST /v1/units/{id}/permits`
  - `PATCH /v1/permits/{id}` (status transitions)
  - `GET /v1/permits/expiring?days=30`
- Permit auto-expiry sweep: Arq cron daily updates `status='expired'` where `expiry_date < today` and `status='active'`.

**Testing**:
- Unit: resolving a deficiency without `resolution_notes` → 422.
- Integration: critical deficiency auto-creates WO with priority=emergency.
- Integration: cron flips an `active` permit with expiry yesterday to `expired`.

#### 3.5 — Inspection Certificate PDF

**What**: WeasyPrint-rendered PDF certificates with embedded hash and QR verification.

**Design**:
- `evt.reporting.certificate_pdf.render(inspection_id) -> bytes` uses Jinja2 template at `evt/reporting/templates/certificate_<code_family>.html`.
- Template includes: tenant logo, unit identification, code edition reference, inspector details, signed-off checklist summary, deficiencies, QR code linking to `/v1/public/verify/{hash_prefix}`.
- Certificate is stored in S3 under `tenants/{tid}/certificates/{inspection_id}.pdf`.
- `GET /v1/public/verify/{hash_prefix}` returns inspection's hash and status (no PII) for AHJ verification.

**Testing**:
- Fixture-based: render certificate from canned inspection; pixel-stable on 3 consecutive runs (allow weasy timestamp injection to be deterministic via override).
- Integration: hash on PDF QR matches stored `certificate_hash`.
- Integration: verify endpoint returns 404 for unknown prefix, 200 with status for known one.

---

## Phase 4: Work Orders, AMC Contracts & PM Schedules

### Purpose
Operationalise day-to-day field service: work order management, AMC contract coverage, preventive maintenance schedules (calendar + meter-driven), and parts/labour tracking. After this phase a dispatcher can create, assign, and close work orders, and PM schedules generate WOs automatically.

### Tasks

#### 4.1 — Work Order Model & API

**What**: `work_order`, `work_order_labour`, `work_order_part`, `work_order_attachment` tables + full CRUD/lifecycle API.

**Design**:
- WO state machine: `open → assigned → in_progress → on_hold → completed` (or `cancelled` terminal).
- `wo_number` generated as `WO-{tenant_short}-{seq:08d}` via Postgres sequence per tenant.
- API:
  - `POST /v1/work-orders`
  - `GET /v1/work-orders?status=&assigned_to=&unit_id=&building_id=&priority=&from=&to=`
  - `PATCH /v1/work-orders/{id}/assign` body `{user_id}`
  - `PATCH /v1/work-orders/{id}/status` body `{status, reason?}`
  - `POST /v1/work-orders/{id}/labour` clock in/out
  - `POST /v1/work-orders/{id}/parts`
  - `POST /v1/work-orders/{id}/attachments` (multipart; S3 upload)
  - `POST /v1/work-orders/{id}/complete` body includes resolution text.
- Idempotency: every mutating endpoint accepts `Idempotency-Key` header → Redis cache for 24h.

**Testing**:
- Unit: invalid transition `completed → in_progress` rejected.
- Integration: attachment upload to MinIO produces presigned download URL.
- Integration: SLA breach detection — WO with priority=emergency and contract sla_response_hours=4 with no assignment after 4h → SLA breach alert.

#### 4.2 — AMC Contract Management

**What**: `service_contract` and `contract_unit` tables + CRUD + auto-renewal reminders.

**Design**:
- API:
  - `POST /v1/contracts`
  - `GET /v1/contracts?status=&client_org_id=`
  - `POST /v1/contracts/{id}/units` (bulk add/remove)
  - `PATCH /v1/contracts/{id}` (status transitions)
- Renewal sweep: Arq daily cron flips `active → expiring` when `end_date <= today + renewal_notice_days`; queues notification to contract owner.
- Auto-renew: when `auto_renew=true` and `today >= end_date - 7 days`, create successor contract with new end_date = old end + 12 months.

**Testing**:
- Unit: cannot add a unit to a contract whose tenant differs from the unit's tenant.
- Integration: cron expiring sweep updates exactly contracts in window.
- Integration: auto-renew creates successor with linked `parent_contract_id` (add this column).

#### 4.3 — Preventive Maintenance Engine

**What**: `pm_schedule` table + nightly PM sweep that creates work orders when triggered.

**Design**:
- `pm_runner.sweep(tenant_id)` runs nightly per tenant:
  - For calendar schedules: `next_due_date <= today` → create WO type `preventive`, scheduled_date = next_due_date.
  - For meter schedules: latest `component.cycle_count >= meter_threshold` → create WO, increment `meter_threshold` by interval.
  - Update `pm_schedule.last_completed` only after WO is completed (handled in WO completion hook).
- `checklist_template_id` references a stored JSON template (`pm_checklist_template` mini-table) so PMs and inspections share format.
- API:
  - `POST /v1/units/{id}/pm-schedules`
  - `GET /v1/pm-schedules?upcoming_days=30`
  - `POST /v1/pm-schedules/{id}/disable`

**Testing**:
- Unit: meter PM creates WO once per crossing, not repeatedly until completed.
- Integration: nightly sweep on tenant with mixed calendar/meter schedules creates correct WO counts.

---

## Phase 5: Web Dashboard (Contractor & Building-Manager UI)

### Purpose
Ship the primary web UI consumed by dispatchers, building managers, and inspectors. Covers authentication, portfolio overview, compliance calendar, work-order board, unit detail with inspection history, and an admin area.

### Tasks

#### 5.1 — Next.js App Foundation & API Client

**What**: Bootstrap the Next.js 15 app with TypeScript strict, Tailwind 4, shadcn/ui, TanStack Query, and an OpenAPI-generated API client.

**Design**:
- `scripts/regenerate_openapi.py` exports `docs/api/openapi.yaml` from FastAPI.
- `web/scripts/codegen.ts` uses `openapi-typescript` to generate `web/src/lib/api-types.ts` and a thin `fetchClient` wrapper.
- Auth handled via httpOnly refresh cookie + in-memory access token; refresh-on-401 interceptor.
- Layouts: `(auth)`, `(dashboard)`, `(ahj)`, `(public)` segments with appropriate guards.

**Testing**:
- Unit (Vitest): `apiClient` retries on 401 with refresh, fails after 2 attempts.
- E2E (Playwright): sign-in via UI → lands on `/portfolio`.

#### 5.2 — Portfolio & Building Pages

**What**: List/grid views for portfolios and buildings, with map and key counts (units, open WOs, overdue obligations).

**Design**:
- `/portfolio` page: cards per portfolio with totals; MapLibre embed showing buildings.
- `/buildings/[id]` page: tabs `Units`, `Compliance`, `Work Orders`, `Contracts`.
- Use TanStack Table for filterable, sortable unit lists with column visibility persistence.

**Testing**:
- E2E: click portfolio card → see buildings; click building → see units; deep-link survives refresh.
- Accessibility (axe-core): pages pass WCAG AA contrast checks.

#### 5.3 — Compliance Calendar View

**What**: Calendar grid + table dual-view for upcoming obligations.

**Design**:
- Month grid (`react-day-picker` headless) coloured by status; tooltips show requirement and unit.
- "List" toggle uses TanStack Table grouped by jurisdiction → building → unit.
- Bulk "schedule inspection" action selecting N obligations → opens WO-create modal pre-filled.

**Testing**:
- E2E: change jurisdiction filter → calendar reflects only that jurisdiction's units.
- E2E: bulk select 3 obligations → modal pre-fills 3 work orders; submit → 3 WOs created.

#### 5.4 — Work Order Board

**What**: Kanban board (open / assigned / in_progress / on_hold / completed) with drag-to-transition and dispatcher overrides.

**Design**:
- DnD via `@dnd-kit`.
- Quick-assign popover with technician availability snapshot.
- Real-time updates via Redis pub/sub → SSE endpoint `/v1/work-orders/stream`.

**Testing**:
- E2E: dragging WO column updates DB.
- E2E: two browsers open — change in browser A appears in browser B within 2s.

#### 5.5 — Unit Detail Page

**What**: Single source of truth for one elevator unit: spec, components, inspection history, deficiencies, permits, sensors (placeholder until Phase 7), WO history.

**Design**:
- Header card: emergency phone, manufacturer, install date, ADA, energy rating.
- Timeline component showing inspections, WOs, deficiencies chronologically.
- Action menu: schedule inspection, create WO, generate certificate, decommission.

**Testing**:
- E2E: full unit lifecycle navigation works (create unit → add component → schedule inspection → open WO → close).

---

## Phase 6: Mobile Technician App

### Purpose
Equip field technicians with a mobile PWA that works offline in machine rooms and elevator pits. Built once with Capacitor; deployed as iOS, Android, and web. Covers task list, inspection capture, deficiency reporting with photos, time clock, and parts logging — all with full offline support and conflict-aware sync.

### Tasks

#### 6.1 — Capacitor PWA Shell & Offline Sync Engine

**What**: Vite + React shell wrapped by Capacitor 6; offline queue and sync engine.

**Design**:
- Dexie (IndexedDB) tables mirror server entities the tech needs: `work_orders`, `units`, `components`, `inspection_drafts`, `outbox`.
- Sync engine `sync.ts`:
  - Pull: `GET /v1/sync/since?cursor=<server_lsn>` returns changed rows since last sync.
  - Push: outbox entries POSTed with `Idempotency-Key=outbox_id`; on success, mark synced.
  - Conflict policy: server is authoritative for status transitions; client wins for text fields edited offline (with audit note).
- Background sync via Capacitor Background Task plugin and Service Worker Periodic Background Sync where supported.

**Testing**:
- Unit (Vitest): sync engine deduplicates outbox entries with same Idempotency-Key.
- Integration: simulated offline → 3 inspection drafts saved → reconnect → all 3 sync within 30s.
- Playwright (web PWA mode): install service worker, go offline, open WO, mark complete, go online → server reflects completion.

#### 6.2 — Work Order List & Detail (Mobile)

**What**: Touch-optimised list with status filters; detail with one-tap status updates and labour clock.

**Design**:
- Status filter chips: My Open, Assigned Today, In Progress, Emergencies.
- Detail: tap to clock in (writes labour row), tap parts (search + add), tap photo (Capacitor Camera) → S3 upload via signed URL.
- Resolution screen with voice-to-text (Capacitor SpeechRecognition) feeding the resolution textarea.

**Testing**:
- E2E (mobile-viewport Playwright): clock in → state changes; second clock-in same WO blocked.
- Manual smoke on iOS simulator: camera, GPS, scan flows work.

#### 6.3 — Inspection Capture (Mobile)

**What**: Step-by-step inspection wizard following the checklist template, capturing measurements, photos, and defects per item.

**Design**:
- Each checklist item rendered as a card with: result picker, measurement input (when applicable), photo button (multi-photo), notes field.
- Items appear in `sort_order` with progress bar and "Save & Continue" persistence to Dexie after each item.
- Final review screen shows summary; tap "Complete Inspection" creates the certificate after sync.

**Testing**:
- Integration: completing all checklist items offline, then syncing, results in a `completed` inspection on the server with all checklist items.
- E2E: photo capture compresses to ≤1.5MB JPEG before queuing.

#### 6.4 — QR-Scan Fault Lookup

**What**: Scan a unit's installed QR code to jump straight to its WO list (technician variant of the public fault page).

**Design**:
- Capacitor BarcodeScanner reads token; client calls `GET /v1/units/by-qr/{token}` → unit_id → navigate to WO list filtered by that unit.

**Testing**:
- Unit: invalid QR token → user-friendly error.
- E2E (simulated QR): scan → land on unit's WO list.

---

## Phase 7: IoT Sensor Ingestion & Anomaly Alerting

### Purpose
Bridge OEM-agnostic IoT data into the platform via BACnet, Modbus, OPC-UA, and MQTT. Store telemetry in TimescaleDB, run threshold and statistical anomaly detection, and auto-generate IoT alerts that may escalate to work orders. This is the v1.1 differentiator versus iFactory and the multi-OEM gap versus Otis ONE / KONE 24/7.

### Tasks

#### 7.1 — Sensor Registry & Telemetry Hypertable

**What**: `sensor`, `telemetry_reading` (TimescaleDB hypertable), `iot_alert` tables + sensor CRUD API.

**Design**:
- Migration converts `telemetry_reading` to a hypertable: `SELECT create_hypertable('telemetry_reading','recorded_at', chunk_time_interval => INTERVAL '7 days');`
- Continuous aggregates:
  - `telemetry_1min` (mean/min/max/stddev per sensor per minute, retained 90 days)
  - `telemetry_1hour` (retained 2 years)
  - `telemetry_1day` (retained indefinitely)
- API:
  - `POST /v1/units/{id}/sensors`
  - `PATCH /v1/sensors/{id}/thresholds`
  - `GET /v1/telemetry?sensor_id=&from=&to=&agg=raw|1min|1hour|1day`
  - `GET /v1/alerts?status=&unit_id=`
  - `POST /v1/alerts/{id}/acknowledge`

**Testing**:
- Integration: inserting 100k readings then querying with `agg=1hour` returns rolled-up rows.
- Integration: chunk compression applied after 30 days reduces disk by >70% on synthetic data.

#### 7.2 — IoT Gateway Process

**What**: Standalone container `evt-iot-gateway` deployed on-premise that bridges BACnet/Modbus/OPC-UA to the platform's HTTPS ingestion endpoint.

**Design**:
- Config in YAML (`/etc/evt/gateway.yaml`):
  ```yaml
  api_base: https://platform.example.com
  api_token: <gateway_jwt>
  poll_interval_s: 5
  bacnet:
    enabled: true
    device_id: 1234
    objects:
      - object_id: analogInput:1
        sensor_uuid: <uuid>
        unit_of_measure: amperes
  modbus:
    enabled: true
    targets:
      - host: 10.0.0.50
        port: 502
        registers:
          - address: 40001
            sensor_uuid: <uuid>
  opcua:
    enabled: false
  ```
- Adapter pattern: `IoTAdapter` interface with `poll() -> list[Reading]` and `subscribe(callback)`.
- Batches readings (max 1000 per call or 5s) → `POST /v1/ingest/telemetry` (token authenticated, idempotent via batch UUID).
- Local buffering to SQLite on connectivity loss; replay on reconnect.

**Testing**:
- Unit: BACnet adapter mocks `bacpypes3` → poll returns synthetic readings.
- Unit: Modbus adapter handles slave timeout gracefully.
- Integration (docker compose with a `bacnet-simulator` service): gateway → API → DB end-to-end.
- Integration: gateway buffers 5k readings during simulated outage and replays them within 30s of reconnect.

#### 7.3 — Threshold & Anomaly Detection

**What**: Real-time threshold breach detection + scheduled Isolation Forest anomaly detection per sensor.

**Design**:
- Threshold: Arq worker subscribes to `iot_ingest` queue; on each batch, checks against sensor.threshold_low/high; creates `iot_alert` with type `threshold_breach`.
- Anomaly: nightly per-sensor job pulls last 30 days of 1-min aggregates; fits `IsolationForest(contamination=0.01)`; scores last 24h; alerts where score < -0.55 (configurable).
- Predictive: simple linear/MA forecast on motor current trend → alert if predicted threshold breach within 14 days; type `predictive`.
- Escalation: critical alert auto-creates WO type `corrective` with priority `high` (configurable per tenant).

**Testing**:
- Unit: threshold of 12.5 amps; reading 13.0 → alert created; reading 12.4 → no alert.
- Fixture: 30-day synthetic sensor stream with injected anomalies — detector flags ≥85% of injected anomalies.
- Integration: critical alert auto-creates exactly one WO (not duplicated on repeated breaches within 1h).

#### 7.4 — OEM Adapter Framework (Otis, KONE, TK, Schindler)

**What**: Abstract adapter interface + initial KONE adapter (publicly documented APIs at dev.kone.com).

**Design**:
- `AbstractOEMAdapter` interface:
  ```python
  class OEMAdapter(Protocol):
      brand: str
      async def authenticate(self, credentials: dict) -> None: ...
      async def list_devices(self) -> list[OEMDevice]: ...
      async def stream_status(self, device_id: str) -> AsyncIterator[OEMStatus]: ...
      async def fetch_service_orders(self, since: datetime) -> list[OEMServiceOrder]: ...
  ```
- KONE adapter: OAuth2 client credentials → WebSocket subscribe to `/site-monitoring` → emit normalised readings to `sensor` table (sensor auto-provisioned with `protocol='kone_api'`).
- Mapping layer: KONE event types → internal `sensor_type` (e.g., `lift_call_state` → ignored; `door_state` → boolean sensor; `deck_position` → numeric).
- Tenant config table `oem_credential` (encrypted at rest using application-level KMS).

**Testing**:
- Unit: KONE adapter normalises mock WebSocket frame to canonical Reading shape.
- Integration (KONE sandbox or recorded fixture replay): subscribe → 100 messages → 100 readings persisted.

---

## Phase 8: AHJ Portal, Public QR Fault Reporting & Permit Filing

### Purpose
Open the platform outward: a read-only portal for Authorities Having Jurisdiction, a tokenised QR-code page for any building occupant to report a fault without an app, and a workflow for filing operating permits to participating AHJs. Closes the AHJ-facing gap identified in features.md.

### Tasks

#### 8.1 — AHJ Read-Only Portal

**What**: Separate Next.js route group `(ahj)` with a constrained UI for AHJ auditor users.

**Design**:
- AHJ users provisioned with role `ahj_auditor`, scoped to one or more jurisdictions (`user_role.scope_type='jurisdiction'`).
- Read-only routes:
  - `/ahj/portal` → buildings in scoped jurisdiction(s).
  - `/ahj/buildings/[id]` → units, compliance status, certificates, open deficiencies.
  - `/ahj/certificates/[hash_prefix]` → public verification.
- Backend enforces: GET-only, no PII for occupants, every read audited.
- AHJ logo and jurisdictions configurable per tenant for branding.

**Testing**:
- Unit: AHJ user attempting `POST /v1/work-orders` → 403.
- Integration: AHJ user scoped to NY sees NY units but not CA units.
- E2E: AHJ flow: sign in → list buildings → drill to unit → view certificate PDF.

#### 8.2 — Public QR Fault Page

**What**: `qr_code_registration` and `public_fault_report` tables + unauthenticated submission page.

**Design**:
- QR tokens generated via `secrets.token_urlsafe(16)` on unit create or via `POST /v1/units/{id}/qr` (re-issue).
- Public page route: `web/src/app/(public)/fr/[token]/page.tsx` — minimal Next.js static SSR.
- Form: name (optional), phone/email (one required), description (required, max 1000 chars), up to 3 photos (uploaded directly to S3 via presigned POST).
- API: `POST /v1/public/fault-reports/{token}` rate-limited 5/hour per IP+token; CAPTCHA via hCaptcha after 2 submissions.
- On submission: triage queue notification to dispatcher; auto-WO creation optional per tenant policy.

**Testing**:
- Integration: invalid token → 404.
- Integration: 6 submissions from one IP+token in 1h → 6th returns 429.
- E2E: visit `/fr/{token}` → fill → submit → dispatcher sees report in triage queue.

#### 8.3 — Permit Filing Workflow

**What**: Pluggable AHJ permit filing connectors + workflow engine.

**Design**:
- `evt.adapters.ahj.filing` defines `PermitFiler` interface; v1 ships:
  - `EmailFiler`: produces signed PDF packet + emails to configured AHJ address.
  - `NYCDOB_StubFiler`: scaffolding for NYC Department of Buildings (manual file generation; future automation).
- Workflow: `POST /v1/permits/{id}/file` triggers filing job → packet PDF generated → sent → external reference recorded.
- Filing packet template includes: latest certified inspection, unit identification, all open/closed deficiencies, contractor licence info.

**Testing**:
- Unit: filing packet PDF round-trips through hash verification.
- Integration (mocked SMTP): EmailFiler successfully sends to a test inbox via MailHog.
- Integration: filing without latest certified inspection → 422 with helpful message.

---

## Phase 9: AI-Native Features (Incident Classification, Regwatch, Risk Scoring)

### Purpose
Deliver the AI-native differentiators that distinguish this platform from incumbents: NLP classification of technician incident notes, regulatory change monitoring with fleet impact assessment, and portfolio-level risk scoring. These are the features highlighted in the README's "AI-Native Advantage" section.

### Tasks

#### 9.1 — LLM Provider Abstraction

**What**: Provider-agnostic LLM client supporting Anthropic Claude (production) and Ollama (air-gapped fallback).

**Design**:
- `evt.ai.llm.LLMClient`:
  ```python
  class LLMClient(Protocol):
      async def complete(
          self,
          system: str,
          user: str,
          *,
          max_tokens: int = 2048,
          response_format: Literal["text", "json"] = "text",
      ) -> LLMResponse: ...
  ```
- Implementations: `AnthropicLLM` (uses `anthropic` SDK; supports prompt caching for stable system prompts), `OllamaLLM`.
- Prompts stored under `evt/ai/prompts/*.txt` with frontmatter for version and model hints.
- Every call recorded in `ai_invocation` (new mini-table): prompt_name, version, model, tokens_in/out, cost_usd, latency_ms, success.

**Testing**:
- Unit: provider switches via env var.
- Unit: prompt template loader fails fast on missing variable.
- Integration: anthropic call with cached system prompt — second call within 5 minutes has `cache_read_input_tokens > 0` in response.

#### 9.2 — NLP Incident Classification

**What**: Classify free-text incident reports into defect type, affected component, severity, and recommended corrective action.

**Design**:
- Prompt template `incident_classify.txt`:
  ```
  SYSTEM:
  You are an elevator incident triage assistant. Output JSON only with fields:
  defect_type, component, severity (critical|major|minor), corrective_action, confidence (0..1).
  Use the controlled vocabularies provided.

  USER:
  Vocabulary - defect_type: {{defect_vocab}}
  Vocabulary - component: {{component_vocab}}
  Unit context: {{unit_summary}}
  Report text: """{{report_text}}"""
  ```
- `evt.ai.classifier.classify_incident(incident_id)`:
  1. Load incident + unit + recent components.
  2. Render prompt, call LLM with `response_format=json`.
  3. Validate against `IncidentClassification` Pydantic model.
  4. Update `incident.ai_*` fields and `ai_extracted_data` JSONB.
- Triggered on incident create + via re-classify button.

**Testing**:
- Unit (mocked LLM): classifier persists fields and confidence.
- Integration with real LLM (marked `slow`): three canonical reports produce expected labels within tolerance.
- Eval harness: 50-report golden set; classifier accuracy ≥85% on component, ≥90% on severity.

#### 9.3 — Regulatory Change Monitor

**What**: Periodic crawl of ASME, ANSI, CEN, ISO change pages → LLM impact assessment → notification + obligation regeneration where needed.

**Design**:
- Source list in `evt/db/seeds/regwatch_sources.json`: URLs, parse strategy (RSS / HTML selector), standard_family mapping.
- Arq cron weekly: fetch each source, diff against last snapshot stored in `regwatch_snapshot` table, identify new entries.
- For each new entry: LLM prompt `regulatory_impact.txt`:
  ```
  SYSTEM:
  You are a regulatory analyst for vertical-transport safety codes. Given a code amendment,
  describe: standard_family, edition_or_amendment, effective_date, change_summary,
  unit_impact_criteria (filter conditions to identify affected units), recommended_actions.
  ```
- Output validated; if `unit_impact_criteria` provided, system queries matching units and creates a `regwatch_alert` per tenant with affected unit counts.

**Testing**:
- Unit: source diff detects exactly the new entries (fixture: before/after HTML).
- Integration (mocked LLM): new ASME A17.1-2028 entry → alert created with correct unit count.
- E2E: dispatcher sees regwatch banner; clicking shows affected units.

#### 9.4 — Portfolio Risk Scoring

**What**: Composite risk score per unit, aggregated to building and portfolio.

**Design**:
- Score components (each normalised 0-1, weighted):
  - Age factor (installation_date)
  - Cycle factor (max component cycle / typical lifespan)
  - Compliance factor (overdue obligations count, weighted by severity)
  - Deficiency factor (open critical deficiencies in last 12 months)
  - Telemetry factor (anomaly count in last 90 days)
  - OEM-specific recall factor (cross-ref regwatch alerts)
- Weights configurable per tenant (`risk_weights` table, defaults shipped).
- Nightly Arq job recomputes per unit; results stored in `unit_risk_score` (date-partitioned for trend).
- API:
  - `GET /v1/risk/portfolio?portfolio_id=`
  - `GET /v1/units/{id}/risk/history`

**Testing**:
- Unit: score deterministic given fixed inputs.
- Integration: changing a weight regenerates scores on demand via `POST /v1/risk/recompute`.
- E2E: portfolio risk heatmap renders top-N highest-risk units.

---

## Phase 10: Reporting, Observability, Deployment & Hardening

### Purpose
Final phase before v1.0: analytics reports, observability stack, production deployment artefacts, security hardening, and documentation.

### Tasks

#### 10.1 — Operational Analytics

**What**: Pre-built reports for MTTR, PM completion rate, downtime, AMC coverage, first-time-fix rate.

**Design**:
- `evt.reporting.analytics_queries` — parameterised SQL templates returning Pandas-friendly result sets.
- API:
  - `GET /v1/reports/mttr?from=&to=&building_id=`
  - `GET /v1/reports/pm-completion`
  - `GET /v1/reports/downtime`
  - `GET /v1/reports/amc-coverage`
- Each endpoint supports `format=json|csv|xlsx`.
- Web dashboard `/reports` page with chart cards (Recharts).

**Testing**:
- Unit: MTTR computed correctly from canned WO history.
- Integration: CSV export reproduces JSON values row-for-row.

#### 10.2 — Observability Stack

**What**: OpenTelemetry instrumentation, Prometheus scrape, Grafana dashboards, Loki log aggregation.

**Design**:
- OTel SDK initialised at app startup; auto-instrumentation for FastAPI, SQLAlchemy, Redis, httpx, Arq.
- Metrics exported on `/metrics` (Prometheus format).
- Dashboards committed under `helm/elevator-vt/dashboards/`:
  - API: RPS, latency P50/P95/P99, error rate, by route.
  - Worker: queue depth per queue, task duration distribution, failure rate.
  - DB: connection pool usage, slow query log count.
  - IoT: ingestion lag, sensor freshness per tenant.
- Loki pipeline parses structured JSON logs; trace_id propagated for cross-service correlation.

**Testing**:
- Integration: `/metrics` returns expected metric names.
- Integration: a 500 error appears in trace with full DB span context.

#### 10.3 — Security Hardening

**What**: Apply OWASP API Security Top 10 controls; perform self-audit using OWASP ZAP and Bandit.

**Design**:
- Rate limiting on every public endpoint (Redis token bucket).
- All PII fields (technician phone, fault reporter contact) flagged in `pii_fields.yaml`; redaction filter in logs.
- Per-tenant data export and deletion endpoints (GDPR Article 15/17 support).
- TLS 1.3 enforced via reverse proxy (Caddy in dev compose; Nginx in Helm).
- Secrets via SOPS+age committed encrypted; CI decrypts using deploy key.
- CSP headers on web app; CSRF token on mutating BFF routes.
- Dependency scanning: `pip-audit`, `pnpm audit`, Dependabot.
- Container scanning: Trivy in CI.

**Testing**:
- Integration: ZAP baseline scan on staging — 0 high-severity findings.
- Integration: Bandit scan — 0 high severity.
- Unit: PII redaction filter masks email/phone in log lines.
- E2E: GDPR export endpoint produces a complete tenant snapshot ZIP.

#### 10.4 — Helm Chart & Production Deployment

**What**: Production-ready Kubernetes chart, runbook, and example values for AWS EKS deployment.

**Design**:
- Sub-charts for: api, worker, web, iot-gateway (optional), postgres (or pointer to external), redis (or pointer).
- Pod disruption budgets, HPA on api/worker, resource requests/limits documented.
- Migration job pattern: pre-install hook running `alembic upgrade head`.
- Backup CronJob: nightly `pg_dump` to S3 + restore-test job weekly.
- Runbook in `docs/operations/runbook.md`: deploys, rollbacks, common incidents, on-call queries.

**Testing**:
- Integration: `helm install` into a kind cluster succeeds; smoke test hits `/healthz`.
- Integration: rollback to previous chart version preserves data.

#### 10.5 — Documentation & Developer Experience

**What**: Architecture docs, API reference, contributor guide, compliance mapping docs.

**Design**:
- `docs/architecture.md` overview with C4 diagrams (mermaid).
- `docs/adr/`: at least one ADR per major decision (chosen data model, queue choice, LLM provider strategy, OEM adapter pattern, AHJ data isolation).
- `docs/api/openapi.yaml` published; rendered to HTML at `/docs` via Redoc.
- `docs/compliance/asme-a17-1-mapping.md` — maps each MVP feature to specific ASME A17.1-2025 sections.
- `docs/compliance/en-81-mapping.md` — same for EN 81-20/-50.
- `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, issue templates, PR template.

**Testing**:
- CI: `markdownlint` on all docs.
- CI: link checker passes (no 404 internal links).
- CI: OpenAPI schema regenerated and committed (fail if drift).

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Auth                          [required by everything]
   │
Phase 2: Asset Register & Reference Data            [requires Phase 1]
   │
Phase 3: Compliance Engine & Inspections            [requires Phase 2]
   │
   ├── Phase 4: Work Orders, AMC, PM                [requires Phase 3]
   │      │
   │      └── Phase 6: Mobile Technician App        [requires Phase 4 + Phase 5.1]
   │
   └── Phase 5: Web Dashboard                       [requires Phase 3; can run parallel to Phase 4]
          │
          └── Phase 8: AHJ Portal & QR & Permits    [requires Phase 5 + Phase 4]
   │
   └── Phase 7: IoT & Anomaly                       [requires Phase 2; can run parallel to Phases 3-5]
          │
          └── Phase 9: AI Features                  [requires Phase 3 + Phase 4 + Phase 7]
                 │
                 └── Phase 10: Reporting, Obs, Hardening, Docs    [requires all prior]
```

Parallelism opportunities:

- After Phase 3 completes, Phase 4 and Phase 5 can be developed concurrently by separate streams (backend vs frontend).
- Phase 7 (IoT pipeline) shares only Phase 2 dependencies and can run in parallel with Phases 3-5.
- Phase 9.1 (LLM client) can be scaffolded immediately after Phase 1 since it has no domain dependency — the actual classifiers depend on Phase 3 / Phase 7 data.

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before it is considered complete:

1. All tasks in the phase implemented per design specification.
2. All new unit, integration, and e2e tests pass; coverage for new modules ≥85%.
3. `ruff check . && ruff format --check .` exits clean.
4. `mypy src/` exits clean (no new ignores without ADR justification).
5. Frontend: `pnpm -C web lint && pnpm -C web typecheck && pnpm -C web build` exits clean.
6. `docker compose build` succeeds for every Dockerfile and `docker compose up` starts cleanly.
7. Alembic migrations apply forward and backward without manual SQL.
8. Every new API endpoint appears in the auto-generated OpenAPI schema with examples.
9. Every new config option documented in `.env.example` and `docs/configuration.md`.
10. Every new permission added to `permissions.json` and seeded role mapping updated.
11. Audit log records all mutating operations for new entities (verify via integration test).
12. Tenant isolation enforced and tested for any cross-tenant access path.
13. Performance budget met: per-request P95 < 300ms for read endpoints, < 800ms for write endpoints on a 1000-tenant test database.
14. At least one ADR written for any non-obvious decision made during the phase.
15. README updated with new capabilities; CHANGELOG entry added under "Unreleased".
