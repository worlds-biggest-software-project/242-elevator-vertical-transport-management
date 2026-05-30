# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Elevator & Vertical Transport Management · Created: 2026-05-22

## Philosophy

This model follows classical relational database design with full normalisation (3NF). Every distinct concept in the elevator management domain — from jurisdictions and code versions to individual sensor readings — receives its own table with explicit foreign key relationships. The design prioritises data integrity, referential consistency, and the ability to answer complex cross-entity queries without ambiguity.

Real-world systems that use this pattern include ERP platforms like SAP and Microsoft Dynamics, as well as CMMS tools like UpKeep and Fiix. The approach aligns naturally with the ASME A17.1 compliance domain where regulators require precise, auditable records with clear provenance. Each elevator shaft is modelled as a first-class compliance entity — distinct from the building or the equipment — reflecting the regulatory reality that inspections, permits, and deficiency records attach to a specific shaft installation.

The trade-off is a higher table count and more complex joins, but this is offset by strong data integrity guarantees and the ability to evolve the schema incrementally as new code versions (ISO 8100 harmonisation, state-specific amendments) are adopted.

**Best for:** Organisations that prioritise regulatory compliance, audit integrity, and complex cross-entity reporting over rapid prototyping speed.

**Trade-offs:**
- (+) Maximum referential integrity — no orphaned records possible
- (+) Standard SQL tooling; well-understood by most engineering teams
- (+) Clean separation of concerns makes each domain area independently testable
- (+) Naturally aligns with ASME A17.1 / EN 81 inspection record structure
- (-) High table count (~55+) increases migration complexity
- (-) Schema changes require migrations; less flexible for jurisdiction-specific variants
- (-) Complex joins for cross-domain reports (e.g., "all units in portfolio X with overdue Cat 5 tests")
- (-) JSONB flexibility sacrificed for structural rigidity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASME A17.1 / CSA B44 | Inspection types (Cat 1/3/5), test categories, and deficiency severity codes modelled as reference tables; code_version tracked per jurisdiction |
| ASME A17.3 | Separate compliance rule set for existing installations; tracked via `code_standard` enum on `compliance_requirement` |
| EN 81-20 / EN 81-50 | European code editions tracked in same `code_edition` table with `standard_family = 'EN81'` |
| ISO 8100-1 / 8100-2 | Future global harmonisation standard; `code_edition` table supports `standard_family = 'ISO8100'` |
| ISO 3166-1/2 | Country and subdivision codes used in `jurisdiction` table for unambiguous location identification |
| BACnet (ASHRAE 135) | Lift, Escalator, and Elevator Group object types inform the equipment type hierarchy |
| SEIS (Standard Elevator Information Schema) | Entity structure for elevator configuration, door arrangements, and floor mappings informed by SEIS vocabulary |
| ADA | Accessibility compliance status tracked as a boolean + last-audit-date per unit |
| NFPA 72 / 101 | Fire recall compliance tracked as a compliance requirement type |
| ISO 25745 | Energy performance rating stored per unit |
| ISO 8601 | All timestamps use TIMESTAMPTZ; date fields stored as DATE in ISO 8601 format |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANT & ORGANISATION
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',  -- 'standard', 'enterprise', 'ahj'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,  -- 'contractor', 'building_owner', 'ahj', 'oem'
    tax_id          VARCHAR(100),
    website         VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_app_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_app_user_org ON app_user(organisation_id);

-- =============================================================
-- RBAC
-- =============================================================

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- 'admin', 'dispatcher', 'technician', 'inspector', 'building_manager', 'ahj_auditor'
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_role_tenant_name ON role(tenant_id, name);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- 'unit.read', 'unit.write', 'inspection.approve', 'wo.assign'
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
    scope_type      VARCHAR(50),        -- NULL = tenant-wide, 'building', 'portfolio'
    scope_id        UUID,               -- ID of the building or portfolio
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);
CREATE INDEX idx_user_role_user ON user_role(user_id);
```

## Jurisdiction & Regulatory Framework

```sql
-- =============================================================
-- JURISDICTION & CODE VERSIONS
-- =============================================================

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL,  -- 'country', 'state', 'province', 'city', 'county'
    country_code    CHAR(2) NOT NULL,        -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),             -- ISO 3166-2 (e.g., 'US-NY')
    parent_id       UUID REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);
CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_id);

CREATE TABLE code_edition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    standard_family VARCHAR(50) NOT NULL,    -- 'ASME_A17_1', 'ASME_A17_3', 'EN81', 'ISO8100'
    edition_year    INTEGER NOT NULL,        -- e.g., 2025
    edition_label   VARCHAR(100) NOT NULL,   -- 'ASME A17.1-2025', 'EN 81-20:2020'
    effective_date  DATE,
    superseded_by   UUID REFERENCES code_edition(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_code_edition_family_year ON code_edition(standard_family, edition_year);

CREATE TABLE jurisdiction_code_adoption (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    code_edition_id UUID NOT NULL REFERENCES code_edition(id),
    adoption_date   DATE NOT NULL,
    enforcement_date DATE,
    local_amendments TEXT,                   -- description of local modifications (e.g., 'NYC Appendix K')
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jca_jurisdiction ON jurisdiction_code_adoption(jurisdiction_id);
CREATE INDEX idx_jca_code ON jurisdiction_code_adoption(code_edition_id);

CREATE TABLE compliance_requirement_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_edition_id UUID NOT NULL REFERENCES code_edition(id),
    requirement_code VARCHAR(50) NOT NULL,   -- 'CAT1', 'CAT3', 'CAT5', 'ANNUAL_INSPECTION', 'FIRE_RECALL'
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    equipment_types VARCHAR(50)[] NOT NULL,  -- '{elevator,escalator}' or '{elevator}'
    frequency_months INTEGER,                -- 12 for annual, 60 for Cat 5
    applies_to_existing BOOLEAN NOT NULL DEFAULT true,  -- A17.1 vs A17.3 distinction
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crt_code_edition ON compliance_requirement_template(code_edition_id);
```

## Asset & Equipment Management

```sql
-- =============================================================
-- BUILDINGS & PORTFOLIOS
-- =============================================================

CREATE TABLE portfolio (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_portfolio_tenant ON portfolio(tenant_id);

CREATE TABLE building (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    portfolio_id    UUID REFERENCES portfolio(id),
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),                 -- ISO 3166-1
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    total_floors    INTEGER,
    building_type   VARCHAR(50),             -- 'commercial', 'residential', 'mixed', 'hospital', 'hotel'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_building_tenant ON building(tenant_id);
CREATE INDEX idx_building_portfolio ON building(portfolio_id);
CREATE INDEX idx_building_jurisdiction ON building(jurisdiction_id);

-- =============================================================
-- EQUIPMENT UNITS (Elevator Shaft as First-Class Entity)
-- =============================================================

CREATE TABLE equipment_unit (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    building_id         UUID NOT NULL REFERENCES building(id),
    unit_number         VARCHAR(50) NOT NULL,    -- Building-local identifier (e.g., 'E-1', 'ESC-2')
    equipment_type      VARCHAR(50) NOT NULL,    -- 'elevator', 'escalator', 'moving_walkway'
    sub_type            VARCHAR(50),             -- 'traction', 'hydraulic', 'mrl', 'freight', 'service'
    manufacturer        VARCHAR(100),            -- OEM brand (Otis, KONE, Schindler, ThyssenKrupp, etc.)
    model               VARCHAR(100),
    serial_number       VARCHAR(100),
    installation_date   DATE,
    capacity_kg         NUMERIC(10, 2),
    capacity_persons    INTEGER,
    speed_mps           NUMERIC(6, 3),           -- metres per second
    num_stops           INTEGER,
    num_doors           INTEGER DEFAULT 1,       -- front door, rear door
    travel_height_m     NUMERIC(8, 3),
    machine_room_location VARCHAR(50),           -- 'overhead', 'basement', 'machine_room_less'
    emergency_phone_number VARCHAR(50),          -- ASME A17.1 requirement
    emergency_phone_provider VARCHAR(100),
    ada_compliant       BOOLEAN,
    energy_rating       VARCHAR(10),             -- ISO 25745 classification (A-G)
    status              VARCHAR(50) NOT NULL DEFAULT 'active',  -- 'active', 'out_of_service', 'decommissioned', 'modernising'
    applicable_code_edition_id UUID REFERENCES code_edition(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(building_id, unit_number)
);
CREATE INDEX idx_unit_tenant ON equipment_unit(tenant_id);
CREATE INDEX idx_unit_building ON equipment_unit(building_id);
CREATE INDEX idx_unit_type ON equipment_unit(equipment_type);
CREATE INDEX idx_unit_manufacturer ON equipment_unit(manufacturer);
CREATE INDEX idx_unit_status ON equipment_unit(status);

CREATE TABLE unit_floor_mapping (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id) ON DELETE CASCADE,
    floor_number    INTEGER NOT NULL,            -- Universal floor number (0 = ground, -1 = basement, etc.)
    floor_label     VARCHAR(50) NOT NULL,        -- Display label ('L', 'G', '1', 'B1', 'P2', 'PH')
    has_front_door  BOOLEAN NOT NULL DEFAULT true,
    has_rear_door   BOOLEAN NOT NULL DEFAULT false,
    is_main_egress  BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL,
    UNIQUE(unit_id, floor_number)
);
CREATE INDEX idx_floor_mapping_unit ON unit_floor_mapping(unit_id);

CREATE TABLE elevator_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    building_id     UUID NOT NULL REFERENCES building(id),
    group_name      VARCHAR(100) NOT NULL,       -- 'High-Rise Bank', 'Low-Rise Bank', 'Service'
    group_mode      VARCHAR(50) DEFAULT 'normal', -- BACnet: 'normal', 'down_peak', 'two_way', 'four_way', 'emergency_power'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE elevator_group_member (
    group_id        UUID NOT NULL REFERENCES elevator_group(id) ON DELETE CASCADE,
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id) ON DELETE CASCADE,
    priority        INTEGER DEFAULT 0,
    PRIMARY KEY (group_id, unit_id)
);
```

## Component Tracking

```sql
-- =============================================================
-- COMPONENTS & PARTS
-- =============================================================

CREATE TABLE component_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,  -- 'traction_machine', 'door_operator', 'governor', 'brake', 'controller', 'car_frame', 'guide_rails', 'safety_device', 'buffer', 'rope_belt', 'sheave'
    category        VARCHAR(50) NOT NULL,          -- 'drive', 'safety', 'door', 'car', 'hoistway', 'controller', 'signalisation'
    is_safety_critical BOOLEAN NOT NULL DEFAULT false,
    typical_lifespan_years INTEGER,
    description     TEXT
);

CREATE TABLE component (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_type_id UUID NOT NULL REFERENCES component_type(id),
    manufacturer    VARCHAR(100),
    model           VARCHAR(100),
    serial_number   VARCHAR(100),
    install_date    DATE,
    warranty_expiry DATE,
    expected_replacement_date DATE,
    cycle_count     BIGINT DEFAULT 0,            -- door cycles, motor starts, etc.
    status          VARCHAR(50) NOT NULL DEFAULT 'operational',  -- 'operational', 'degraded', 'failed', 'replaced'
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_component_unit ON component(unit_id);
CREATE INDEX idx_component_type ON component(component_type_id);
CREATE INDEX idx_component_status ON component(status);
```

## Compliance & Inspection Management

```sql
-- =============================================================
-- COMPLIANCE OBLIGATIONS (Instance per Unit)
-- =============================================================

CREATE TABLE compliance_obligation (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id                 UUID NOT NULL REFERENCES equipment_unit(id),
    template_id             UUID NOT NULL REFERENCES compliance_requirement_template(id),
    jurisdiction_adoption_id UUID REFERENCES jurisdiction_code_adoption(id),
    next_due_date           DATE NOT NULL,
    last_completed_date     DATE,
    status                  VARCHAR(50) NOT NULL DEFAULT 'upcoming',  -- 'upcoming', 'due_soon', 'overdue', 'completed', 'waived'
    advance_notice_days     INTEGER NOT NULL DEFAULT 60,
    notes                   TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_obligation_unit ON compliance_obligation(unit_id);
CREATE INDEX idx_obligation_status ON compliance_obligation(status);
CREATE INDEX idx_obligation_due ON compliance_obligation(next_due_date);

-- =============================================================
-- INSPECTIONS
-- =============================================================

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    obligation_id   UUID REFERENCES compliance_obligation(id),
    inspection_type VARCHAR(50) NOT NULL,    -- 'cat_1', 'cat_3', 'cat_5', 'annual', 'periodic', 'witness', 'acceptance'
    inspector_user_id UUID REFERENCES app_user(id),
    inspector_name  VARCHAR(255),            -- For external inspectors not in the system
    inspector_licence_number VARCHAR(100),
    inspection_date DATE NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    result          VARCHAR(50),             -- 'pass', 'pass_with_deficiencies', 'fail', 'incomplete'
    code_edition_id UUID REFERENCES code_edition(id),
    certificate_number VARCHAR(100),
    certificate_hash VARCHAR(128),           -- SHA-512 hash for tamper-evidence
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inspection_unit ON inspection(unit_id);
CREATE INDEX idx_inspection_date ON inspection(inspection_date);
CREATE INDEX idx_inspection_obligation ON inspection(obligation_id);
CREATE INDEX idx_inspection_result ON inspection(result);

CREATE TABLE inspection_checklist_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    item_code       VARCHAR(50) NOT NULL,    -- Reference to specific A17.1 section
    description     TEXT NOT NULL,
    result          VARCHAR(50) NOT NULL,    -- 'pass', 'fail', 'not_applicable', 'not_tested'
    measurement     NUMERIC(12, 4),
    measurement_unit VARCHAR(20),
    photo_attachment_ids UUID[],
    notes           TEXT,
    sort_order      INTEGER NOT NULL
);
CREATE INDEX idx_checklist_inspection ON inspection_checklist_item(inspection_id);

-- =============================================================
-- DEFICIENCIES
-- =============================================================

CREATE TABLE deficiency (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_id    UUID REFERENCES component(id),
    deficiency_code VARCHAR(50),             -- Standardised deficiency code
    severity        VARCHAR(20) NOT NULL,    -- 'critical', 'major', 'minor', 'observation'
    description     TEXT NOT NULL,
    code_reference  VARCHAR(100),            -- e.g., 'A17.1-2025 §2.27.3.1'
    status          VARCHAR(50) NOT NULL DEFAULT 'open',  -- 'open', 'in_progress', 'resolved', 'waived'
    due_date        DATE,
    resolved_date   DATE,
    resolved_by     UUID REFERENCES app_user(id),
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deficiency_inspection ON deficiency(inspection_id);
CREATE INDEX idx_deficiency_unit ON deficiency(unit_id);
CREATE INDEX idx_deficiency_status ON deficiency(status);
CREATE INDEX idx_deficiency_severity ON deficiency(severity);

-- =============================================================
-- PERMITS
-- =============================================================

CREATE TABLE permit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    permit_type     VARCHAR(50) NOT NULL,    -- 'operating', 'installation', 'modernisation', 'temporary'
    permit_number   VARCHAR(100),
    issued_date     DATE,
    expiry_date     DATE,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'active', 'expired', 'revoked', 'renewed'
    filing_reference VARCHAR(200),
    issued_by       VARCHAR(255),            -- AHJ office name
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_permit_unit ON permit(unit_id);
CREATE INDEX idx_permit_status ON permit(status);
CREATE INDEX idx_permit_expiry ON permit(expiry_date);
```

## Work Order & Field Service

```sql
-- =============================================================
-- CONTRACTS (AMC)
-- =============================================================

CREATE TABLE service_contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contractor_org_id UUID NOT NULL REFERENCES organisation(id),
    client_org_id   UUID NOT NULL REFERENCES organisation(id),
    contract_number VARCHAR(100),
    contract_type   VARCHAR(50) NOT NULL,    -- 'full_maintenance', 'oil_and_grease', 'on_call', 'modernisation'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    auto_renew      BOOLEAN NOT NULL DEFAULT false,
    renewal_notice_days INTEGER DEFAULT 90,
    monthly_value   NUMERIC(12, 2),
    currency        CHAR(3) DEFAULT 'USD',
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- 'draft', 'active', 'expiring', 'expired', 'terminated'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contract_tenant ON service_contract(tenant_id);
CREATE INDEX idx_contract_status ON service_contract(status);
CREATE INDEX idx_contract_end ON service_contract(end_date);

CREATE TABLE contract_unit (
    contract_id     UUID NOT NULL REFERENCES service_contract(id) ON DELETE CASCADE,
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    sla_response_hours INTEGER DEFAULT 4,
    sla_resolution_hours INTEGER DEFAULT 24,
    PRIMARY KEY (contract_id, unit_id)
);

-- =============================================================
-- WORK ORDERS
-- =============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    contract_id     UUID REFERENCES service_contract(id),
    wo_number       VARCHAR(50) NOT NULL,
    wo_type         VARCHAR(50) NOT NULL,    -- 'corrective', 'preventive', 'inspection', 'callback', 'modernisation', 'emergency'
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',  -- 'emergency', 'high', 'normal', 'low'
    summary         VARCHAR(500) NOT NULL,
    description     TEXT,
    reported_by     UUID REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    scheduled_date  DATE,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',  -- 'open', 'assigned', 'in_progress', 'on_hold', 'completed', 'cancelled'
    resolution      TEXT,
    is_entrapment   BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(50),             -- 'manual', 'qr_code', 'iot_alert', 'pm_schedule', 'callback'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, wo_number)
);
CREATE INDEX idx_wo_tenant ON work_order(tenant_id);
CREATE INDEX idx_wo_unit ON work_order(unit_id);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_priority ON work_order(priority);
CREATE INDEX idx_wo_scheduled ON work_order(scheduled_date);

CREATE TABLE work_order_labour (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    technician_id   UUID NOT NULL REFERENCES app_user(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    travel_minutes  INTEGER,
    labour_minutes  INTEGER,
    notes           TEXT
);
CREATE INDEX idx_wo_labour_wo ON work_order_labour(work_order_id);

CREATE TABLE work_order_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    component_id    UUID REFERENCES component(id),
    part_number     VARCHAR(100),
    description     VARCHAR(255) NOT NULL,
    quantity        NUMERIC(10, 2) NOT NULL DEFAULT 1,
    unit_cost       NUMERIC(12, 2),
    currency        CHAR(3) DEFAULT 'USD'
);
CREATE INDEX idx_wo_part_wo ON work_order_part(work_order_id);

CREATE TABLE work_order_attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(50),             -- 'image/jpeg', 'application/pdf', etc.
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    uploaded_by     UUID REFERENCES app_user(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_attach_wo ON work_order_attachment(work_order_id);

-- =============================================================
-- PREVENTIVE MAINTENANCE SCHEDULES
-- =============================================================

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    frequency_type  VARCHAR(20) NOT NULL,    -- 'calendar', 'meter', 'both'
    interval_days   INTEGER,                 -- For calendar-based (e.g., 30 = monthly)
    meter_type      VARCHAR(50),             -- 'door_cycles', 'motor_starts', 'running_hours'
    meter_threshold BIGINT,                  -- Trigger when cycle count passes this
    checklist_template_id UUID,
    next_due_date   DATE,
    last_completed  DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pm_tenant ON pm_schedule(tenant_id);
CREATE INDEX idx_pm_unit ON pm_schedule(unit_id);
CREATE INDEX idx_pm_next_due ON pm_schedule(next_due_date);
```

## IoT Telemetry

```sql
-- =============================================================
-- SENSOR CONFIGURATION
-- =============================================================

CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_id    UUID REFERENCES component(id),
    sensor_type     VARCHAR(50) NOT NULL,    -- 'vibration', 'temperature', 'current', 'door_timing', 'speed', 'acceleration', 'humidity', 'noise'
    protocol        VARCHAR(50),             -- 'bacnet', 'modbus_tcp', 'opc_ua', 'mqtt', 'otis_api', 'kone_api'
    location        VARCHAR(100),            -- 'machine_room', 'car_top', 'pit', 'door_operator', 'governor'
    bacnet_object_id VARCHAR(100),           -- BACnet object identifier if applicable
    modbus_address  INTEGER,                 -- Modbus register address if applicable
    unit_of_measure VARCHAR(30),             -- 'celsius', 'amperes', 'mm_per_s', 'milliseconds', 'mps'
    threshold_low   NUMERIC(12, 4),
    threshold_high  NUMERIC(12, 4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_unit ON sensor(unit_id);
CREATE INDEX idx_sensor_component ON sensor(component_id);
CREATE INDEX idx_sensor_type ON sensor(sensor_type);

-- =============================================================
-- TELEMETRY READINGS (Partitioned by Month)
-- =============================================================

CREATE TABLE telemetry_reading (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    unit_id         UUID NOT NULL,           -- Denormalised for partition pruning
    recorded_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16, 6) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good',  -- 'good', 'uncertain', 'bad'
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for 2026)
-- CREATE TABLE telemetry_reading_2026_01 PARTITION OF telemetry_reading
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE INDEX idx_telemetry_sensor_time ON telemetry_reading(sensor_id, recorded_at DESC);
CREATE INDEX idx_telemetry_unit_time ON telemetry_reading(unit_id, recorded_at DESC);

-- =============================================================
-- IoT ALERTS
-- =============================================================

CREATE TABLE iot_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    alert_type      VARCHAR(50) NOT NULL,    -- 'threshold_breach', 'anomaly', 'predictive', 'communication_loss'
    severity        VARCHAR(20) NOT NULL,    -- 'critical', 'warning', 'info'
    value           NUMERIC(16, 6),
    threshold       NUMERIC(16, 6),
    message         TEXT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- 'active', 'acknowledged', 'resolved', 'auto_resolved'
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
    work_order_id   UUID REFERENCES work_order(id),        -- If an alert triggered a WO
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_iot_alert_unit ON iot_alert(unit_id);
CREATE INDEX idx_iot_alert_status ON iot_alert(status);
CREATE INDEX idx_iot_alert_created ON iot_alert(created_at DESC);
```

## Incident & Entrapment Tracking

```sql
-- =============================================================
-- INCIDENTS
-- =============================================================

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    incident_number VARCHAR(50) NOT NULL,
    incident_type   VARCHAR(50) NOT NULL,    -- 'entrapment', 'injury', 'near_miss', 'malfunction', 'vandalism'
    severity        VARCHAR(20) NOT NULL,    -- 'critical', 'major', 'minor'
    reported_at     TIMESTAMPTZ NOT NULL,
    reported_by     UUID REFERENCES app_user(id),
    description     TEXT NOT NULL,
    -- AI-classified fields (NLP-assisted)
    ai_defect_type  VARCHAR(100),
    ai_component    VARCHAR(100),
    ai_severity     VARCHAR(20),
    ai_corrective_action TEXT,
    ai_confidence   NUMERIC(4, 3),           -- 0.000 to 1.000
    status          VARCHAR(50) NOT NULL DEFAULT 'reported',  -- 'reported', 'investigating', 'resolved', 'closed'
    work_order_id   UUID REFERENCES work_order(id),
    resolution      TEXT,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, incident_number)
);
CREATE INDEX idx_incident_tenant ON incident(tenant_id);
CREATE INDEX idx_incident_unit ON incident(unit_id);
CREATE INDEX idx_incident_type ON incident(incident_type);
CREATE INDEX idx_incident_status ON incident(status);
```

## QR Code Fault Reporting

```sql
-- =============================================================
-- QR CODE FAULT REPORTING
-- =============================================================

CREATE TABLE qr_code_registration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    qr_code_token   VARCHAR(100) NOT NULL UNIQUE,
    label           VARCHAR(255),            -- e.g., 'Elevator 3 - Lobby Level'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_qr_unit ON qr_code_registration(unit_id);
CREATE INDEX idx_qr_token ON qr_code_registration(qr_code_token);

CREATE TABLE public_fault_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    qr_code_id      UUID NOT NULL REFERENCES qr_code_registration(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    reporter_name   VARCHAR(255),
    reporter_phone  VARCHAR(50),
    reporter_email  VARCHAR(255),
    fault_description TEXT NOT NULL,
    photo_urls      TEXT[],
    status          VARCHAR(50) NOT NULL DEFAULT 'received',  -- 'received', 'triaged', 'work_order_created'
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fault_report_unit ON public_fault_report(unit_id);
```

## Audit Trail

```sql
-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,    -- 'create', 'update', 'delete', 'approve', 'reject', 'login', 'export'
    entity_type     VARCHAR(100) NOT NULL,   -- 'inspection', 'work_order', 'deficiency', 'permit', etc.
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);

-- =============================================================
-- NOTIFICATIONS
-- =============================================================

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    channel         VARCHAR(20) NOT NULL,    -- 'in_app', 'email', 'sms', 'push'
    subject         VARCHAR(255) NOT NULL,
    body            TEXT,
    entity_type     VARCHAR(100),
    entity_id       UUID,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    sent_at         TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notification_user ON notification(user_id, is_read, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | Multi-tenant foundation |
| RBAC | 4 | Role-based access with scoped assignments |
| Jurisdiction & Codes | 4 | Code edition tracking with jurisdiction adoption |
| Buildings & Portfolios | 2 | Physical location hierarchy |
| Equipment Units | 4 | Shaft-level entities with floor mapping and groups |
| Components | 2 | Component type catalogue and installed instances |
| Compliance | 3 | Obligation instances, inspections, checklists |
| Deficiencies | 1 | Linked to inspections and components |
| Permits | 1 | Operating and installation permits |
| Contracts | 2 | AMC contracts with unit coverage |
| Work Orders | 4 | WOs with labour, parts, and attachments |
| PM Schedules | 1 | Calendar and meter-based triggers |
| IoT / Telemetry | 3 | Sensors, readings (partitioned), alerts |
| Incidents | 1 | With NLP-assisted classification fields |
| QR Code Reporting | 2 | Public-facing fault submission |
| Audit & Notifications | 2 | Partitioned audit log and notification queue |
| **Total** | **~38** | |

---

## Key Design Decisions

1. **Elevator shaft as first-class compliance entity.** The `equipment_unit` table represents a shaft installation, not a piece of equipment. Inspections, permits, deficiencies, and compliance obligations attach to this entity. This reflects the regulatory reality where ASME A17.1 and EN 81 requirements are per-installation, not per-building.

2. **Jurisdiction-aware code version tracking.** The `jurisdiction → jurisdiction_code_adoption → code_edition → compliance_requirement_template` chain enables the system to know exactly which code edition applies at any building location and auto-generate compliance calendars with the correct test types and intervals.

3. **Compliance obligation as template + instance.** `compliance_requirement_template` defines what a Cat 1 test is; `compliance_obligation` is the specific instance tied to a specific unit with a specific due date. This separation supports the AI deadline engine that generates individualised calendars.

4. **Partitioned telemetry and audit tables.** Both `telemetry_reading` and `audit_log` use PostgreSQL range partitioning by time. This is essential for performance: telemetry tables can grow to billions of rows, and partition pruning keeps queries fast.

5. **BACnet-aligned sensor model.** The `sensor` table includes `bacnet_object_id` and `modbus_address` fields to map directly to BACnet Lift/Escalator objects and Modbus register addresses, supporting OEM-agnostic integration.

6. **NLP classification fields on incidents.** The `incident` table includes `ai_defect_type`, `ai_component`, `ai_severity`, `ai_corrective_action`, and `ai_confidence` fields. These are populated by the AI classification pipeline but stored alongside human-reported data for comparison and model improvement.

7. **Scoped RBAC.** User roles can be scoped to the entire tenant, a specific building, or a portfolio. This supports scenarios where an AHJ auditor has read-only access to specific buildings within a contractor's portfolio.

8. **Emergency phone mapping.** `equipment_unit.emergency_phone_number` satisfies the ASME A17.1 requirement to map emergency phone numbers to specific elevator units — a gap identified across all surveyed competitors.

9. **Cryptographic certificate integrity.** `inspection.certificate_hash` stores a SHA-512 hash of the inspection certificate, providing tamper-evidence without requiring blockchain infrastructure.

10. **Multi-standard code support.** The `code_edition.standard_family` field supports ASME A17.1, ASME A17.3, EN 81, and ISO 8100 in the same table structure, enabling international multi-jurisdiction deployments from day one.
