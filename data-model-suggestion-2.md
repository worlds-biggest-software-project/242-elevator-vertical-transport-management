# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Elevator & Vertical Transport Management · Created: 2026-05-22

## Philosophy

This model treats every change to every entity as an immutable event in a central event store. The current state of any elevator unit, inspection, or work order is derived by replaying its event stream. Materialised read models (CQRS pattern) serve the operational UI with denormalised, query-optimised projections that are rebuilt from events on demand.

This approach is inspired by financial ledger systems and regulatory audit platforms where the ability to answer "what was the state on date X?" is not optional — it is a compliance requirement. In the elevator domain, AHJs and insurance auditors need tamper-proof records showing when inspections occurred, when deficiencies were opened and closed, when permits changed status, and who made each change. An event-sourced architecture provides this intrinsically rather than as a bolt-on audit log.

The pattern is well-established in fintech (bank ledgers, trading systems), healthcare (electronic health record audit trails), and IoT platforms (sensor event streams). iFactory's blockchain-verified audit trail and BRYCER's regulatory decision trail both address the same underlying need — provable, ordered, tamper-evident history — but with proprietary implementations. Event sourcing delivers this at the architectural level.

**Best for:** Organisations where regulatory audit, temporal queries, and provable compliance history are the primary requirements — especially AHJ-facing deployments and contractors managing high-liability portfolios.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — no separate audit log needed
- (+) Temporal queries native: "show me the compliance state of unit X on 2025-03-15"
- (+) Event replay enables AI analytics on change patterns (deficiency resolution velocity, inspection pass rates over time)
- (+) Naturally supports regulatory change monitoring: code amendment events propagate through the system
- (+) Events are append-only — write performance scales linearly
- (-) Higher storage requirements: events accumulate forever
- (-) Read model complexity: every query hits a materialised projection, not the source of truth
- (-) Projection rebuild after schema changes can be slow for large event stores
- (-) More complex to implement than traditional CRUD; team must understand event sourcing patterns
- (-) Eventual consistency between event store and read models requires careful handling

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASME A17.1 / CSA B44 | Inspection events carry code edition, test category, and checklist results; compliance state projection maintains per-unit obligation timeline |
| ASME A17.3 | Existing-equipment compliance events distinguished by `code_standard` field in event payload |
| EN 81-20 / EN 81-50 | European inspection events use same event types with `standard_family: 'EN81'` in payload |
| ISO 8100 | Future global code events will use `standard_family: 'ISO8100'`; no schema change required — just new event payloads |
| BACnet (ASHRAE 135) | Sensor telemetry ingested as `TelemetryRecorded` events aligned with BACnet Lift/Escalator object property names |
| SEIS | Event payload vocabulary for equipment configuration and state changes informed by Standard Elevator Information Schema |
| NFPA 72 / 101 | Fire recall compliance tracked via `ComplianceObligationCreated` events with `requirement_type: 'fire_recall'` |
| ISO 8601 | All event timestamps are TIMESTAMPTZ in ISO 8601 format; event ordering uses monotonic sequence numbers |
| ISO 3166-1/2 | Jurisdiction identifiers in events use ISO 3166 codes for unambiguous cross-region analysis |

---

## Event Store (Source of Truth)

```sql
-- =============================================================
-- CORE EVENT STORE
-- =============================================================

-- The event store is the single source of truth. Every state change
-- in the system is recorded as an immutable event. Events are never
-- updated or deleted.

CREATE TABLE event_store (
    sequence_number     BIGSERIAL NOT NULL,          -- Global monotonic ordering
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_type         VARCHAR(100) NOT NULL,       -- Aggregate type: 'EquipmentUnit', 'Inspection', 'WorkOrder', 'ComplianceObligation', 'Permit', 'Incident', 'ServiceContract', 'Sensor'
    stream_id           UUID NOT NULL,               -- Aggregate ID (e.g., the unit ID, inspection ID)
    event_type          VARCHAR(100) NOT NULL,       -- See event type catalogue below
    event_version       INTEGER NOT NULL,            -- Version within this stream (optimistic concurrency)
    tenant_id           UUID NOT NULL,
    caused_by_user_id   UUID,                        -- NULL for system-generated events
    caused_by_event_id  UUID,                        -- Causal chain: which event triggered this one
    payload             JSONB NOT NULL,              -- Event-specific data
    metadata            JSONB NOT NULL DEFAULT '{}', -- IP address, user agent, correlation ID
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (sequence_number),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (recorded_at);

-- Partitioned monthly for performance
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at);
CREATE INDEX idx_event_tenant ON event_store(tenant_id, recorded_at);
CREATE INDEX idx_event_causal ON event_store(caused_by_event_id);
CREATE INDEX idx_event_payload ON event_store USING gin(payload);

-- =============================================================
-- EVENT TYPE CATALOGUE
-- =============================================================

-- Equipment Unit lifecycle:
--   UnitRegistered, UnitUpdated, UnitDecommissioned, UnitReactivated
--   ComponentInstalled, ComponentReplaced, ComponentFailed
--   FloorMappingSet, ElevatorGroupAssigned
--   EmergencyPhoneUpdated, CodeEditionChanged

-- Compliance lifecycle:
--   ComplianceObligationCreated, ObligationDueDateUpdated,
--   ObligationCompleted, ObligationOverdue, ObligationWaived
--   JurisdictionCodeAdopted, CodeAmendmentPublished

-- Inspection lifecycle:
--   InspectionScheduled, InspectionStarted, InspectionCompleted,
--   ChecklistItemRecorded, DeficiencyFound, DeficiencyResolved,
--   CertificateIssued, CertificateRevoked

-- Work Order lifecycle:
--   WorkOrderCreated, WorkOrderAssigned, WorkOrderStarted,
--   WorkOrderPartsUsed, WorkOrderLabourLogged,
--   WorkOrderCompleted, WorkOrderCancelled, WorkOrderReopened

-- Permit lifecycle:
--   PermitApplicationFiled, PermitIssued, PermitExpired,
--   PermitRevoked, PermitRenewed

-- Incident lifecycle:
--   IncidentReported, IncidentClassifiedByAI, IncidentInvestigated,
--   IncidentResolved, IncidentClosed

-- IoT Telemetry:
--   SensorRegistered, SensorCalibrated, SensorDeactivated,
--   TelemetryBatchRecorded, AnomalyDetected,
--   PredictiveAlertGenerated, AlertAcknowledged, AlertResolved

-- Contract lifecycle:
--   ContractCreated, ContractActivated, ContractUnitAdded,
--   ContractUnitRemoved, ContractRenewed, ContractExpired,
--   ContractTerminated

-- Fault Reporting:
--   PublicFaultReported, FaultTriaged, FaultLinkedToWorkOrder


-- =============================================================
-- EVENT SNAPSHOTS (Optimisation)
-- =============================================================

-- Snapshots avoid replaying the entire stream for long-lived aggregates.
-- A snapshot captures the computed state at a given event_version.

CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(100) NOT NULL,
    stream_id       UUID NOT NULL,
    at_version      INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, at_version)
);
CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, at_version DESC);
```

## Event Payload Examples

```sql
-- Example: UnitRegistered event payload
-- {
--   "unit_number": "E-3",
--   "building_id": "550e8400-e29b-41d4-a716-446655440000",
--   "equipment_type": "elevator",
--   "sub_type": "traction",
--   "manufacturer": "KONE",
--   "model": "MonoSpace 700",
--   "serial_number": "KN-2024-88421",
--   "installation_date": "2024-06-15",
--   "capacity_kg": 1600,
--   "capacity_persons": 21,
--   "speed_mps": 2.5,
--   "num_stops": 28,
--   "num_doors": 1,
--   "emergency_phone_number": "+1-800-555-0142",
--   "jurisdiction_code": "US-IL",
--   "applicable_code_edition": "ASME_A17_1_2025"
-- }

-- Example: InspectionCompleted event payload
-- {
--   "inspection_type": "cat_1",
--   "inspector_name": "J. Martinez",
--   "inspector_licence": "IL-EI-44892",
--   "result": "pass_with_deficiencies",
--   "code_edition": "ASME_A17_1_2025",
--   "certificate_number": "IL-2026-CAT1-88421-0522",
--   "certificate_hash": "a3f8b2c1d4e5...",
--   "checklist_summary": {
--     "total_items": 47,
--     "passed": 44,
--     "failed": 2,
--     "not_applicable": 1
--   }
-- }

-- Example: DeficiencyFound event payload
-- {
--   "inspection_id": "660e8400-e29b-41d4-a716-446655440099",
--   "component_type": "door_operator",
--   "deficiency_code": "DEF-DOOR-003",
--   "severity": "major",
--   "description": "Car door closing force exceeds 30 lbs limit per A17.1 §2.13.4",
--   "code_reference": "A17.1-2025 §2.13.4",
--   "due_date": "2026-06-22"
-- }

-- Example: AnomalyDetected event payload
-- {
--   "sensor_id": "770e8400-e29b-41d4-a716-446655440055",
--   "sensor_type": "vibration",
--   "location": "machine_room",
--   "alert_type": "anomaly",
--   "severity": "warning",
--   "current_value": 4.82,
--   "baseline_value": 2.1,
--   "threshold": 4.0,
--   "unit_of_measure": "mm_per_s",
--   "ai_diagnosis": "Possible bearing degradation in traction machine",
--   "ai_confidence": 0.87,
--   "recommended_action": "Schedule traction machine bearing inspection within 14 days"
-- }

-- Example: CodeAmendmentPublished event payload
-- {
--   "standard_family": "ASME_A17_1",
--   "amendment_id": "A17.1-2025/TIA-001",
--   "published_date": "2026-04-15",
--   "summary": "Revised door reopening device requirements for high-speed elevators",
--   "affected_sections": ["§2.13.1", "§2.13.4.1"],
--   "affected_equipment_types": ["elevator"],
--   "affected_sub_types": ["traction"],
--   "speed_threshold_mps": 3.0,
--   "impacted_unit_count": 42
-- }
```

## Materialised Read Models (CQRS Projections)

```sql
-- =============================================================
-- READ MODEL: Current Unit State
-- =============================================================
-- Rebuilt by replaying EquipmentUnit stream events

CREATE TABLE rm_unit_current (
    unit_id             UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    building_id         UUID NOT NULL,
    building_name       VARCHAR(255),
    portfolio_id        UUID,
    portfolio_name      VARCHAR(255),
    unit_number         VARCHAR(50) NOT NULL,
    equipment_type      VARCHAR(50) NOT NULL,
    sub_type            VARCHAR(50),
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    serial_number       VARCHAR(100),
    installation_date   DATE,
    capacity_kg         NUMERIC(10, 2),
    speed_mps           NUMERIC(6, 3),
    num_stops           INTEGER,
    emergency_phone_number VARCHAR(50),
    status              VARCHAR(50) NOT NULL,
    applicable_code_edition VARCHAR(100),
    jurisdiction_code   VARCHAR(6),
    component_count     INTEGER DEFAULT 0,
    ada_compliant       BOOLEAN,
    energy_rating       VARCHAR(10),
    last_inspection_date DATE,
    last_inspection_result VARCHAR(50),
    next_inspection_due DATE,
    open_deficiency_count INTEGER DEFAULT 0,
    open_work_order_count INTEGER DEFAULT 0,
    risk_score          NUMERIC(5, 2),        -- AI-computed portfolio risk score
    last_event_version  INTEGER NOT NULL,
    last_event_at       TIMESTAMPTZ NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_unit_tenant ON rm_unit_current(tenant_id);
CREATE INDEX idx_rm_unit_building ON rm_unit_current(building_id);
CREATE INDEX idx_rm_unit_status ON rm_unit_current(status);
CREATE INDEX idx_rm_unit_next_insp ON rm_unit_current(next_inspection_due);

-- =============================================================
-- READ MODEL: Compliance Dashboard
-- =============================================================
-- Rebuilt by replaying ComplianceObligation stream events

CREATE TABLE rm_compliance_dashboard (
    obligation_id       UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    unit_id             UUID NOT NULL,
    unit_number         VARCHAR(50),
    building_name       VARCHAR(255),
    portfolio_name      VARCHAR(255),
    requirement_type    VARCHAR(50) NOT NULL,  -- 'cat_1', 'cat_5', 'annual', 'fire_recall'
    standard_family     VARCHAR(50),
    code_edition        VARCHAR(100),
    jurisdiction_name   VARCHAR(255),
    next_due_date       DATE NOT NULL,
    last_completed_date DATE,
    status              VARCHAR(50) NOT NULL,
    days_until_due      INTEGER,               -- Computed daily
    advance_notice_days INTEGER,
    is_overdue          BOOLEAN NOT NULL DEFAULT false,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_compliance_tenant ON rm_compliance_dashboard(tenant_id);
CREATE INDEX idx_rm_compliance_status ON rm_compliance_dashboard(status);
CREATE INDEX idx_rm_compliance_due ON rm_compliance_dashboard(next_due_date);
CREATE INDEX idx_rm_compliance_overdue ON rm_compliance_dashboard(is_overdue) WHERE is_overdue = true;

-- =============================================================
-- READ MODEL: Work Order Board
-- =============================================================
-- Rebuilt by replaying WorkOrder stream events

CREATE TABLE rm_work_order_board (
    work_order_id       UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    wo_number           VARCHAR(50) NOT NULL,
    unit_id             UUID NOT NULL,
    unit_number         VARCHAR(50),
    building_name       VARCHAR(255),
    wo_type             VARCHAR(50) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    summary             VARCHAR(500),
    status              VARCHAR(50) NOT NULL,
    assigned_to_user_id UUID,
    assigned_to_name    VARCHAR(255),
    scheduled_date      DATE,
    is_entrapment       BOOLEAN NOT NULL DEFAULT false,
    source              VARCHAR(50),
    total_labour_minutes INTEGER DEFAULT 0,
    total_parts_cost    NUMERIC(12, 2) DEFAULT 0,
    sla_response_hours  INTEGER,
    sla_breach          BOOLEAN DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_wo_tenant ON rm_work_order_board(tenant_id);
CREATE INDEX idx_rm_wo_status ON rm_work_order_board(status);
CREATE INDEX idx_rm_wo_assigned ON rm_work_order_board(assigned_to_user_id);
CREATE INDEX idx_rm_wo_priority ON rm_work_order_board(priority);

-- =============================================================
-- READ MODEL: Deficiency Tracker
-- =============================================================

CREATE TABLE rm_deficiency_tracker (
    deficiency_id       UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    unit_id             UUID NOT NULL,
    unit_number         VARCHAR(50),
    building_name       VARCHAR(255),
    inspection_id       UUID NOT NULL,
    inspection_date     DATE,
    deficiency_code     VARCHAR(50),
    severity            VARCHAR(20) NOT NULL,
    description         TEXT,
    code_reference      VARCHAR(100),
    component_type      VARCHAR(100),
    status              VARCHAR(50) NOT NULL,
    due_date            DATE,
    days_open           INTEGER,
    resolved_date       DATE,
    resolved_by_name    VARCHAR(255),
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_def_tenant ON rm_deficiency_tracker(tenant_id);
CREATE INDEX idx_rm_def_status ON rm_deficiency_tracker(status);
CREATE INDEX idx_rm_def_severity ON rm_deficiency_tracker(severity);
CREATE INDEX idx_rm_def_unit ON rm_deficiency_tracker(unit_id);

-- =============================================================
-- READ MODEL: Telemetry Summary (Hourly Aggregates)
-- =============================================================

CREATE TABLE rm_telemetry_hourly (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    sensor_id           UUID NOT NULL,
    unit_id             UUID NOT NULL,
    sensor_type         VARCHAR(50) NOT NULL,
    hour_bucket         TIMESTAMPTZ NOT NULL,     -- Truncated to hour
    reading_count       INTEGER NOT NULL,
    min_value           NUMERIC(16, 6),
    max_value           NUMERIC(16, 6),
    avg_value           NUMERIC(16, 6),
    stddev_value        NUMERIC(16, 6),
    p95_value           NUMERIC(16, 6),
    anomaly_count       INTEGER DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, hour_bucket)
) PARTITION BY RANGE (hour_bucket);

CREATE INDEX idx_rm_telemetry_sensor ON rm_telemetry_hourly(sensor_id, hour_bucket DESC);
CREATE INDEX idx_rm_telemetry_unit ON rm_telemetry_hourly(unit_id, hour_bucket DESC);

-- =============================================================
-- READ MODEL: Portfolio Risk Scores
-- =============================================================

CREATE TABLE rm_portfolio_risk (
    unit_id             UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    building_id         UUID NOT NULL,
    building_name       VARCHAR(255),
    portfolio_id        UUID,
    portfolio_name      VARCHAR(255),
    unit_number         VARCHAR(50),
    equipment_type      VARCHAR(50),
    manufacturer        VARCHAR(100),
    age_years           NUMERIC(5, 1),
    total_door_cycles   BIGINT,
    open_deficiency_count INTEGER,
    critical_deficiency_count INTEGER,
    overdue_obligation_count INTEGER,
    last_inspection_result VARCHAR(50),
    mttr_hours          NUMERIC(8, 2),        -- Mean time to repair (rolling 12 months)
    callback_rate       NUMERIC(5, 4),        -- Callbacks / total WOs (rolling 12 months)
    entrapment_count_12m INTEGER,
    sensor_anomaly_rate NUMERIC(5, 4),
    compliance_score    NUMERIC(5, 2),         -- 0-100, derived from obligation completion
    overall_risk_score  NUMERIC(5, 2),         -- 0-100, AI-computed composite
    risk_tier           VARCHAR(20),           -- 'critical', 'high', 'medium', 'low'
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_risk_tenant ON rm_portfolio_risk(tenant_id);
CREATE INDEX idx_rm_risk_tier ON rm_portfolio_risk(risk_tier);
CREATE INDEX idx_rm_risk_score ON rm_portfolio_risk(overall_risk_score DESC);
```

## Reference Data Tables (Non-Event-Sourced)

```sql
-- =============================================================
-- REFERENCE DATA (Static/Slowly-Changing — not event-sourced)
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    parent_id       UUID REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE code_edition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    standard_family VARCHAR(50) NOT NULL,
    edition_year    INTEGER NOT NULL,
    edition_label   VARCHAR(100) NOT NULL,
    effective_date  DATE,
    superseded_by   UUID REFERENCES code_edition(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE component_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    category        VARCHAR(50) NOT NULL,
    is_safety_critical BOOLEAN NOT NULL DEFAULT false,
    typical_lifespan_years INTEGER,
    description     TEXT
);

-- RBAC tables remain non-event-sourced (same as Suggestion 1)
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

-- =============================================================
-- PROJECTION MANAGEMENT
-- =============================================================

CREATE TABLE projection_checkpoint (
    projection_name     VARCHAR(100) PRIMARY KEY,  -- 'rm_unit_current', 'rm_compliance_dashboard', etc.
    last_sequence_number BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              VARCHAR(20) NOT NULL DEFAULT 'running',  -- 'running', 'paused', 'rebuilding', 'error'
    error_message       TEXT
);
```

## Temporal Query Examples

```sql
-- =============================================
-- TEMPORAL QUERY: Unit state at a past date
-- =============================================
-- "What was the compliance status of unit X on 2025-12-01?"

-- Step 1: Get all events for the unit up to that date
SELECT event_type, payload, recorded_at
FROM event_store
WHERE stream_type = 'EquipmentUnit'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND recorded_at <= '2025-12-01T23:59:59Z'
ORDER BY event_version ASC;

-- The application replays these events to reconstruct past state.
-- For compliance obligations, replay the ComplianceObligation stream:

SELECT event_type, payload, recorded_at
FROM event_store
WHERE stream_type = 'ComplianceObligation'
  AND (payload->>'unit_id') = '550e8400-e29b-41d4-a716-446655440000'
  AND recorded_at <= '2025-12-01T23:59:59Z'
ORDER BY sequence_number ASC;

-- =============================================
-- CAUSAL CHAIN: What happened after a code amendment?
-- =============================================
-- "Which units were affected by TIA-001 and what actions were taken?"

WITH amendment_event AS (
    SELECT event_id, sequence_number
    FROM event_store
    WHERE event_type = 'CodeAmendmentPublished'
      AND payload->>'amendment_id' = 'A17.1-2025/TIA-001'
)
SELECT es.event_type, es.stream_id, es.payload, es.recorded_at
FROM event_store es
JOIN amendment_event ae ON es.caused_by_event_id = ae.event_id
ORDER BY es.sequence_number;

-- =============================================
-- ANALYTICS: Deficiency resolution velocity
-- =============================================
-- "Average time to resolve deficiencies by severity, last 12 months"

WITH found AS (
    SELECT stream_id,
           (payload->>'severity') AS severity,
           recorded_at AS found_at
    FROM event_store
    WHERE event_type = 'DeficiencyFound'
      AND recorded_at >= now() - INTERVAL '12 months'
),
resolved AS (
    SELECT stream_id,
           recorded_at AS resolved_at
    FROM event_store
    WHERE event_type = 'DeficiencyResolved'
)
SELECT f.severity,
       COUNT(*) AS deficiency_count,
       AVG(EXTRACT(EPOCH FROM (r.resolved_at - f.found_at)) / 86400) AS avg_days_to_resolve
FROM found f
LEFT JOIN resolved r ON f.stream_id = r.stream_id
GROUP BY f.severity
ORDER BY avg_days_to_resolve DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | Core event table (partitioned) + snapshots |
| Reference Data | 4 | Tenant, organisation, jurisdiction, code editions |
| User & RBAC | 5 | Users, roles, permissions (non-event-sourced) |
| Component Types | 1 | Static reference catalogue |
| Read Model: Units | 1 | Current unit state projection |
| Read Model: Compliance | 1 | Compliance dashboard projection |
| Read Model: Work Orders | 1 | Work order board projection |
| Read Model: Deficiencies | 1 | Deficiency tracker projection |
| Read Model: Telemetry | 1 | Hourly telemetry aggregates (partitioned) |
| Read Model: Risk | 1 | Portfolio risk score projection |
| Projection Management | 1 | Checkpoint tracking for each projection |
| **Total** | **~19 tables** | Plus partitions; read models are disposable/rebuildable |

---

## Key Design Decisions

1. **Single event store as source of truth.** All domain state changes — equipment registration, inspections, work orders, deficiencies, permits, incidents, telemetry alerts — flow through one `event_store` table. This eliminates the need for a separate audit log and guarantees that the audit trail is the system, not an afterthought.

2. **Stream-based aggregate organisation.** Events are grouped by `stream_type` (EquipmentUnit, Inspection, WorkOrder, etc.) and `stream_id`. Each stream represents one aggregate root. Optimistic concurrency is enforced via the `event_version` unique constraint per stream.

3. **Causal event chains.** The `caused_by_event_id` field creates a directed acyclic graph of causality. When a `CodeAmendmentPublished` event causes `ComplianceObligationCreated` events for affected units, the chain is preserved. This enables regulatory impact tracing: "show me everything that happened because of TIA-001."

4. **Snapshots for long-lived aggregates.** An elevator unit may accumulate thousands of events over a 20-year lifespan. Snapshots capture computed state at periodic intervals so that reconstruction only needs to replay events since the last snapshot.

5. **Disposable read models.** Every `rm_*` table can be dropped and rebuilt from the event store. This means read model schema changes do not require data migrations — just rebuild the projection. The `projection_checkpoint` table tracks where each projection has read up to.

6. **Partitioned event store and telemetry.** Both `event_store` and `rm_telemetry_hourly` use range partitioning by time. Old partitions can be archived to cold storage while keeping recent data on fast disks.

7. **JSONB event payloads with GIN index.** Event payloads are stored as JSONB with a GIN index, enabling containment queries like `payload @> '{"severity": "critical"}'` without needing to know the schema at query time. New event types can be added without schema migrations.

8. **Telemetry as events.** Raw sensor readings are ingested as `TelemetryBatchRecorded` events (batched for efficiency), then projected into the `rm_telemetry_hourly` aggregate table. This means sensor data participates in the same causal chain as everything else — an anomaly event can trace back to the readings that triggered it.

9. **AI classification stored as events.** When the NLP pipeline classifies an incident, it emits an `IncidentClassifiedByAI` event with confidence score, defect type, and recommended action. This is a separate event from `IncidentReported`, preserving the human-reported original and the AI interpretation as distinct, timestamped records.

10. **Regulatory change propagation.** A `CodeAmendmentPublished` event triggers a projection that identifies affected units (by equipment type, speed, jurisdiction) and auto-generates `ComplianceObligationCreated` events for each. The entire chain is traceable and auditable.
