# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Elevator & Vertical Transport Management · Created: 2026-05-22

## Philosophy

This model uses a relational backbone for core entities and relationships, but stores jurisdiction-specific, equipment-type-specific, and OEM-specific attributes in JSONB columns. The insight is that an elevator in New York City (governed by ASME A17.1 + NYC Appendix K) has different compliance fields than one in Frankfurt (governed by EN 81-20 + German TRbF), and a KONE traction elevator has different telemetry parameters than an Otis hydraulic unit. Trying to normalise every possible variant into dedicated columns creates an explosion of nullable columns or a sprawling EAV pattern. JSONB absorbs this variability while keeping the universal fields relational.

This pattern is used by modern SaaS platforms that serve diverse markets from a single schema: Salesforce (custom fields as JSON), Shopify (metafields), and many multi-tenant compliance tools. It is particularly appropriate for the elevator domain because: (a) different jurisdictions impose different data requirements, (b) different OEMs expose different telemetry parameters, (c) inspection checklist structures vary by code edition and equipment type, and (d) the platform must support rapid expansion to new countries without schema migrations.

The JSONB columns are not unstructured — each has a documented JSON Schema that the application layer enforces. PostgreSQL's JSONB operators and GIN indexes provide efficient querying into the flexible fields.

**Best for:** Rapid MVP development; platforms that must serve multiple jurisdictions, OEMs, and equipment types from day one; teams that need to iterate on field requirements without database migrations.

**Trade-offs:**
- (+) Core relationships remain referentially consistent (FK constraints)
- (+) Jurisdiction/OEM-specific fields absorbed without schema changes
- (+) New equipment types, code versions, or OEM adapters can be added at the application layer
- (+) GIN indexes on JSONB provide efficient containment and path queries
- (+) Lower table count than fully normalised model (~25 vs ~38)
- (-) JSONB fields bypass database-level constraints (NOT NULL, CHECK, FK)
- (-) Application layer must enforce JSON Schema validation — data integrity shifts from database to code
- (-) JSONB updates replace the entire document (no partial update in standard PostgreSQL)
- (-) Complex JSONB queries can be slower than indexed relational columns
- (-) Reporting/BI tools may struggle with JSONB fields without ETL extraction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASME A17.1 / CSA B44 | Core inspection types and categories are relational; A17.1-specific checklist items stored in JSONB `checklist_data` on inspection records |
| EN 81-20 / EN 81-50 | European inspection fields stored in same JSONB structure with different JSON Schema validation per `standard_family` |
| ISO 8100 | New global standard's unique fields will be supported via new JSON Schema version — no table changes |
| ISO 3166-1/2 | Jurisdiction identification uses ISO 3166 codes in relational columns |
| BACnet (ASHRAE 135) | Sensor configuration includes JSONB `protocol_config` with BACnet object IDs, property IDs, and polling intervals |
| SEIS | Equipment configuration JSONB structure uses SEIS-aligned property names where applicable |
| OPC UA / Modbus | Protocol-specific connection parameters stored in `protocol_config` JSONB on sensor records |
| ADA | Accessibility attributes stored in `equipment_unit.compliance_attributes` JSONB |
| ISO 25745 | Energy performance data stored in `equipment_unit.performance_attributes` JSONB |
| ISO 8601 | All temporal columns use TIMESTAMPTZ |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANT, ORG, USER, RBAC (Relational — same across all models)
-- =============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "default_currency": "USD",
    --   "default_timezone": "America/Chicago",
    --   "features_enabled": ["iot", "ai_classification", "ahj_portal"],
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- contact_info example: {
    --   "phone": "+1-312-555-0100",
    --   "email": "ops@acme-elevator.com",
    --   "address": {"line1": "...", "city": "Chicago", "state": "IL", "postal": "60601", "country": "US"},
    --   "licence_numbers": {"state_contractor": "IL-EC-2024-1234", "federal_ein": "12-3456789"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_tenant ON organisation(tenant_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {
    --   "language": "en",
    --   "timezone": "America/New_York",
    --   "notification_channels": ["email", "push"],
    --   "dashboard_layout": "compact"
    -- }
    certifications  JSONB NOT NULL DEFAULT '[]',
    -- certifications example: [
    --   {"type": "qei", "number": "QEI-2024-5567", "expiry": "2027-06-30", "jurisdiction": "US-NY"},
    --   {"type": "state_inspector", "number": "NY-EI-8812", "expiry": "2026-12-31", "jurisdiction": "US-NY"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- RBAC tables (same relational structure as Suggestion 1)
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["unit.read", "unit.write", "inspection.approve", "wo.assign", "wo.complete"]
    -- Using JSONB array instead of junction table for simpler role management
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_role_name ON role(tenant_id, name);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);
```

## Jurisdiction & Regulatory Framework

```sql
-- =============================================================
-- JURISDICTION & CODE VERSIONS
-- =============================================================

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    jurisdiction_type VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    parent_id       UUID REFERENCES jurisdiction(id),
    local_requirements JSONB NOT NULL DEFAULT '{}',
    -- local_requirements example (NYC):
    -- {
    --   "amendments": ["Appendix K", "Local Law 121"],
    --   "ahj_name": "NYC Department of Buildings",
    --   "ahj_filing_url": "https://www1.nyc.gov/dob",
    --   "additional_test_types": ["pressure_test_hydraulic"],
    --   "reporting_format": "DOB-ELV-1",
    --   "emergency_phone_registry_required": true,
    --   "inspection_witness_required": true,
    --   "special_rules": {
    --     "high_speed_threshold_mps": 3.048,
    --     "annual_brake_test_required": true,
    --     "five_year_load_test_required": true
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);
CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_id);

CREATE TABLE code_edition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    standard_family VARCHAR(50) NOT NULL,
    edition_year    INTEGER NOT NULL,
    edition_label   VARCHAR(100) NOT NULL,
    effective_date  DATE,
    superseded_by   UUID REFERENCES code_edition(id),
    requirements    JSONB NOT NULL DEFAULT '[]',
    -- requirements example:
    -- [
    --   {"code": "CAT1", "name": "Category 1 Periodic Test", "frequency_months": 12,
    --    "equipment_types": ["elevator"], "description": "Annual safety test..."},
    --   {"code": "CAT3", "name": "Category 3 Periodic Test", "frequency_months": 36,
    --    "equipment_types": ["elevator"], "sub_types": ["hydraulic"]},
    --   {"code": "CAT5", "name": "Category 5 Periodic Test", "frequency_months": 60,
    --    "equipment_types": ["elevator"]},
    --   {"code": "ANNUAL_INSPECTION", "name": "Annual Routine Inspection", "frequency_months": 12,
    --    "equipment_types": ["elevator", "escalator", "moving_walkway"]},
    --   {"code": "FIRE_RECALL", "name": "Fire Service Recall Test", "frequency_months": 12,
    --    "equipment_types": ["elevator"], "nfpa_reference": "NFPA 72 §21.3"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_code_family_year ON code_edition(standard_family, edition_year);
CREATE INDEX idx_code_requirements ON code_edition USING gin(requirements);

CREATE TABLE jurisdiction_code_adoption (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    code_edition_id UUID NOT NULL REFERENCES code_edition(id),
    adoption_date   DATE NOT NULL,
    enforcement_date DATE,
    local_overrides JSONB NOT NULL DEFAULT '{}',
    -- local_overrides example:
    -- {
    --   "CAT1": {"frequency_months": 6, "reason": "Local amendment requires semi-annual"},
    --   "additional_requirements": [
    --     {"code": "PRESSURE_TEST", "name": "Hydraulic Pressure Test", "frequency_months": 60}
    --   ]
    -- }
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jca_jurisdiction ON jurisdiction_code_adoption(jurisdiction_id);
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
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_portfolio_tenant ON portfolio(tenant_id);

CREATE TABLE building (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    portfolio_id    UUID REFERENCES portfolio(id),
    name            VARCHAR(255) NOT NULL,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    address         JSONB NOT NULL DEFAULT '{}',
    -- address example:
    -- {
    --   "line1": "233 S Wacker Dr",
    --   "city": "Chicago",
    --   "state": "IL",
    --   "postal_code": "60606",
    --   "country": "US",
    --   "latitude": 41.8789,
    --   "longitude": -87.6359
    -- }
    building_info   JSONB NOT NULL DEFAULT '{}',
    -- building_info example:
    -- {
    --   "type": "commercial",
    --   "total_floors": 110,
    --   "year_built": 1973,
    --   "bms_system": "Siemens Desigo CC",
    --   "fire_alarm_panel": "Notifier NFS2-3030",
    --   "building_management_contact": {"name": "...", "phone": "...", "email": "..."}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_building_tenant ON building(tenant_id);
CREATE INDEX idx_building_portfolio ON building(portfolio_id);
CREATE INDEX idx_building_jurisdiction ON building(jurisdiction_id);

-- =============================================================
-- EQUIPMENT UNITS
-- =============================================================

CREATE TABLE equipment_unit (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    building_id         UUID NOT NULL REFERENCES building(id),
    unit_number         VARCHAR(50) NOT NULL,
    equipment_type      VARCHAR(50) NOT NULL,      -- 'elevator', 'escalator', 'moving_walkway'
    sub_type            VARCHAR(50),               -- 'traction', 'hydraulic', 'mrl', 'freight'
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    serial_number       VARCHAR(100),
    installation_date   DATE,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    applicable_code_edition_id UUID REFERENCES code_edition(id),

    -- Universal numeric fields (relational for indexing and aggregation)
    capacity_kg         NUMERIC(10, 2),
    speed_mps           NUMERIC(6, 3),
    num_stops           INTEGER,
    travel_height_m     NUMERIC(8, 3),

    -- Emergency phone (ASME A17.1 requirement — relational for direct querying)
    emergency_phone_number VARCHAR(50),

    -- JSONB: Equipment-type-specific attributes
    equipment_attributes JSONB NOT NULL DEFAULT '{}',
    -- Elevator example:
    -- {
    --   "num_doors": 2,
    --   "door_type": "center_opening",
    --   "machine_room_location": "overhead",
    --   "controller_type": "microprocessor",
    --   "controller_manufacturer": "Elevator Controls Inc.",
    --   "roping": "2:1",
    --   "counterweight_ratio": 0.4,
    --   "car_dimensions": {"width_mm": 2100, "depth_mm": 1800, "height_mm": 2400},
    --   "door_dimensions": {"width_mm": 1200, "height_mm": 2100},
    --   "fire_service": {"phase1": true, "phase2": true, "designated_floor": "L"},
    --   "code_blue_enabled": true,
    --   "seismic_switch": true
    -- }
    -- Escalator example:
    -- {
    --   "inclination_degrees": 30,
    --   "step_width_mm": 1000,
    --   "rise_m": 4.5,
    --   "rated_speed_mps": 0.5,
    --   "handrail_type": "rubber",
    --   "comb_plate_type": "fixed"
    -- }

    -- JSONB: Compliance-related attributes that vary by jurisdiction
    compliance_attributes JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ada_compliant": true,
    --   "ada_last_audit": "2025-09-15",
    --   "energy_rating": "B",
    --   "energy_rating_standard": "ISO 25745",
    --   "permit_number": "IL-OP-2026-88421",
    --   "permit_expiry": "2027-06-30",
    --   "local_registration_number": "CHI-ELV-2024-3391"
    -- }

    -- JSONB: OEM-specific integration configuration
    oem_integration JSONB NOT NULL DEFAULT '{}',
    -- KONE example:
    -- {
    --   "oem_platform": "kone_247",
    --   "kone_equipment_id": "KN-NAM-2024-88421",
    --   "kone_site_id": "SITE-CHI-233WACKER",
    --   "monitoring_api": "site_monitoring_v2",
    --   "call_api_enabled": true
    -- }
    -- Otis example:
    -- {
    --   "oem_platform": "otis_one",
    --   "otis_unit_id": "OT-US-2024-55291",
    --   "bms_api_enabled": true,
    --   "robot_api_enabled": false
    -- }

    -- JSONB: Floor mapping (SEIS-aligned)
    floor_mapping JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"floor_number": -2, "label": "P2", "front_door": true, "rear_door": false},
    --   {"floor_number": -1, "label": "P1", "front_door": true, "rear_door": false},
    --   {"floor_number": 0,  "label": "L",  "front_door": true, "rear_door": true, "main_egress": true},
    --   {"floor_number": 1,  "label": "2",  "front_door": true, "rear_door": false},
    --   ...
    -- ]

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(building_id, unit_number)
);
CREATE INDEX idx_unit_tenant ON equipment_unit(tenant_id);
CREATE INDEX idx_unit_building ON equipment_unit(building_id);
CREATE INDEX idx_unit_type ON equipment_unit(equipment_type);
CREATE INDEX idx_unit_manufacturer ON equipment_unit(manufacturer);
CREATE INDEX idx_unit_status ON equipment_unit(status);
CREATE INDEX idx_unit_equip_attrs ON equipment_unit USING gin(equipment_attributes);
CREATE INDEX idx_unit_compliance ON equipment_unit USING gin(compliance_attributes);
CREATE INDEX idx_unit_oem ON equipment_unit USING gin(oem_integration);

CREATE TABLE elevator_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    building_id     UUID NOT NULL REFERENCES building(id),
    group_name      VARCHAR(100) NOT NULL,
    group_mode      VARCHAR(50) DEFAULT 'normal',
    unit_ids        UUID[] NOT NULL DEFAULT '{}',     -- Array of equipment_unit IDs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_group_building ON elevator_group(building_id);
```

## Component Tracking

```sql
-- =============================================================
-- COMPONENTS
-- =============================================================

CREATE TABLE component (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_type  VARCHAR(50) NOT NULL,     -- 'traction_machine', 'door_operator', 'governor', 'brake', 'controller', 'safety_device', 'rope_belt', 'sheave', 'buffer', 'guide_rails'
    category        VARCHAR(50) NOT NULL,     -- 'drive', 'safety', 'door', 'car', 'hoistway', 'controller', 'signalisation'
    is_safety_critical BOOLEAN NOT NULL DEFAULT false,
    manufacturer    VARCHAR(100),
    model           VARCHAR(100),
    serial_number   VARCHAR(100),
    install_date    DATE,
    warranty_expiry DATE,
    cycle_count     BIGINT DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'operational',
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Traction machine example:
    -- {
    --   "motor_type": "permanent_magnet",
    --   "power_kw": 22,
    --   "voltage": 480,
    --   "phase": 3,
    --   "rpm_rated": 100,
    --   "bearing_type": "sealed_ball",
    --   "last_vibration_analysis": "2026-03-15",
    --   "oil_type": "synthetic_iso_220",
    --   "oil_change_interval_months": 12
    -- }
    -- Door operator example:
    -- {
    --   "operator_type": "center_opening",
    --   "drive_type": "linear_motor",
    --   "door_weight_kg": 85,
    --   "opening_time_ms": 1800,
    --   "closing_force_n": 133,
    --   "nudging_force_n": 67
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_component_unit ON component(unit_id);
CREATE INDEX idx_component_type ON component(component_type);
CREATE INDEX idx_component_status ON component(status);
CREATE INDEX idx_component_attrs ON component USING gin(attributes);
```

## Compliance, Inspections & Work Orders

```sql
-- =============================================================
-- COMPLIANCE OBLIGATIONS
-- =============================================================

CREATE TABLE compliance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    requirement_code VARCHAR(50) NOT NULL,    -- 'CAT1', 'CAT5', 'ANNUAL_INSPECTION', 'FIRE_RECALL'
    standard_family VARCHAR(50) NOT NULL,     -- 'ASME_A17_1', 'EN81', 'ISO8100'
    code_edition_id UUID REFERENCES code_edition(id),
    next_due_date   DATE NOT NULL,
    last_completed  DATE,
    status          VARCHAR(50) NOT NULL DEFAULT 'upcoming',
    advance_notice_days INTEGER DEFAULT 60,
    jurisdiction_overrides JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "frequency_months": 6,
    --   "override_reason": "NYC Local Law 121 requires semi-annual Cat 1",
    --   "additional_checklist_items": ["pressure_relief_valve", "cylinder_inspection"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
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
    inspection_type VARCHAR(50) NOT NULL,
    inspector_user_id UUID REFERENCES app_user(id),
    inspection_date DATE NOT NULL,
    result          VARCHAR(50),
    code_edition_id UUID REFERENCES code_edition(id),
    certificate_number VARCHAR(100),
    certificate_hash VARCHAR(128),

    -- JSONB: Full checklist data (varies by inspection type and code edition)
    checklist_data  JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"item_code": "2.26.1", "section": "Safety devices",
    --    "description": "Governor tripping speed within tolerance",
    --    "result": "pass", "measurement": 3.12, "unit": "mps",
    --    "tolerance_min": 2.8, "tolerance_max": 3.3},
    --   {"item_code": "2.27.3", "section": "Fire service",
    --    "description": "Phase I recall operation functional",
    --    "result": "pass"},
    --   {"item_code": "2.13.4", "section": "Doors",
    --    "description": "Car door closing force",
    --    "result": "fail", "measurement": 142, "unit": "newtons",
    --    "tolerance_max": 133,
    --    "deficiency": {"severity": "major", "code": "DEF-DOOR-003"}}
    -- ]

    -- JSONB: Inspector details (for external inspectors)
    inspector_details JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "name": "J. Martinez",
    --   "licence_number": "IL-EI-44892",
    --   "licence_jurisdiction": "US-IL",
    --   "organisation": "Illinois State Inspectors",
    --   "phone": "+1-312-555-0199"
    -- }

    -- JSONB: Attachments
    attachments     JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "photo", "filename": "governor_nameplate.jpg", "storage_path": "s3://...", "size_bytes": 245000},
    --   {"type": "pdf", "filename": "cat1_report.pdf", "storage_path": "s3://...", "size_bytes": 1200000}
    -- ]

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inspection_unit ON inspection(unit_id);
CREATE INDEX idx_inspection_date ON inspection(inspection_date);
CREATE INDEX idx_inspection_result ON inspection(result);
CREATE INDEX idx_inspection_checklist ON inspection USING gin(checklist_data);

-- =============================================================
-- DEFICIENCIES
-- =============================================================

CREATE TABLE deficiency (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_id    UUID REFERENCES component(id),
    deficiency_code VARCHAR(50),
    severity        VARCHAR(20) NOT NULL,
    description     TEXT NOT NULL,
    code_reference  VARCHAR(100),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    due_date        DATE,
    resolved_date   DATE,
    resolved_by     UUID REFERENCES app_user(id),
    resolution_data JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "resolution_notes": "Replaced door operator motor and recalibrated closing force to 110N",
    --   "parts_used": [{"part": "Door motor assembly", "part_number": "KN-DM-220V", "qty": 1}],
    --   "work_order_id": "...",
    --   "retest_result": "pass",
    --   "retest_measurement": 110,
    --   "retest_date": "2026-06-05"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deficiency_unit ON deficiency(unit_id);
CREATE INDEX idx_deficiency_status ON deficiency(status);
CREATE INDEX idx_deficiency_severity ON deficiency(severity);

-- =============================================================
-- WORK ORDERS
-- =============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    wo_number       VARCHAR(50) NOT NULL,
    wo_type         VARCHAR(50) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    summary         VARCHAR(500) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    source          VARCHAR(50),
    is_entrapment   BOOLEAN NOT NULL DEFAULT false,
    scheduled_date  DATE,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,

    -- JSONB: Labour, parts, and cost tracking
    labour_entries  JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"technician_id": "...", "technician_name": "A. Chen", "clock_in": "2026-05-22T08:00:00Z",
    --    "clock_out": "2026-05-22T11:30:00Z", "travel_minutes": 25, "labour_minutes": 185,
    --    "notes": "Replaced door operator motor"},
    --   {"technician_id": "...", "technician_name": "B. Kowalski", "clock_in": "2026-05-22T08:00:00Z",
    --    "clock_out": "2026-05-22T10:00:00Z", "travel_minutes": 30, "labour_minutes": 90}
    -- ]

    parts_used      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"component_id": "...", "part_number": "KN-DM-220V", "description": "Door motor assembly",
    --    "quantity": 1, "unit_cost": 850.00, "currency": "USD"}
    -- ]

    attachments     JSONB NOT NULL DEFAULT '[]',

    -- JSONB: AI-assisted classification (for callback and incident-sourced WOs)
    ai_classification JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "defect_type": "door_operator_failure",
    --   "component": "door_operator",
    --   "severity": "major",
    --   "corrective_action": "Replace door operator motor and recalibrate closing force",
    --   "confidence": 0.91,
    --   "classified_at": "2026-05-22T07:45:00Z"
    -- }

    resolution      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, wo_number)
);
CREATE INDEX idx_wo_tenant ON work_order(tenant_id);
CREATE INDEX idx_wo_unit ON work_order(unit_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_priority ON work_order(priority);
```

## Contracts, IoT & Incidents

```sql
-- =============================================================
-- SERVICE CONTRACTS
-- =============================================================

CREATE TABLE service_contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contractor_org_id UUID NOT NULL REFERENCES organisation(id),
    client_org_id   UUID NOT NULL REFERENCES organisation(id),
    contract_number VARCHAR(100),
    contract_type   VARCHAR(50) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    unit_ids        UUID[] NOT NULL DEFAULT '{}',   -- Array of covered equipment_unit IDs
    terms           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "monthly_value": 12500.00,
    --   "currency": "USD",
    --   "auto_renew": true,
    --   "renewal_notice_days": 90,
    --   "sla": {
    --     "response_hours": 4,
    --     "resolution_hours": 24,
    --     "entrapment_response_minutes": 30,
    --     "pm_visits_per_year": 12,
    --     "penalties": {"late_response_per_hour": 150, "missed_pm_per_visit": 500}
    --   },
    --   "coverage": {
    --     "parts_included": true,
    --     "oil_included": true,
    --     "cosmetic_excluded": true,
    --     "vandalism_excluded": true
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contract_tenant ON service_contract(tenant_id);
CREATE INDEX idx_contract_status ON service_contract(status);

-- =============================================================
-- PM SCHEDULES
-- =============================================================

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    name            VARCHAR(255) NOT NULL,
    frequency_type  VARCHAR(20) NOT NULL,
    interval_days   INTEGER,
    meter_type      VARCHAR(50),
    meter_threshold BIGINT,
    next_due_date   DATE,
    last_completed  DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    checklist_template JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"step": 1, "task": "Check machine room temperature", "type": "measurement", "unit": "celsius", "acceptable_range": {"min": 10, "max": 40}},
    --   {"step": 2, "task": "Inspect rope condition", "type": "pass_fail"},
    --   {"step": 3, "task": "Measure door closing force", "type": "measurement", "unit": "newtons", "acceptable_range": {"max": 133}},
    --   {"step": 4, "task": "Lubricate guide rails", "type": "checkbox"},
    --   {"step": 5, "task": "Test emergency communication", "type": "pass_fail"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pm_unit ON pm_schedule(unit_id);
CREATE INDEX idx_pm_next_due ON pm_schedule(next_due_date);

-- =============================================================
-- IoT SENSORS & TELEMETRY
-- =============================================================

CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    component_id    UUID REFERENCES component(id),
    sensor_type     VARCHAR(50) NOT NULL,
    location        VARCHAR(100),
    unit_of_measure VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    protocol_config JSONB NOT NULL DEFAULT '{}',
    -- BACnet example:
    -- {
    --   "protocol": "bacnet",
    --   "device_id": 12345,
    --   "object_type": "analog-input",
    --   "object_instance": 1,
    --   "property": "present-value",
    --   "poll_interval_seconds": 60
    -- }
    -- Modbus TCP example:
    -- {
    --   "protocol": "modbus_tcp",
    --   "host": "192.168.1.100",
    --   "port": 502,
    --   "slave_id": 1,
    --   "register_address": 40001,
    --   "register_type": "holding",
    --   "data_type": "float32",
    --   "poll_interval_seconds": 30
    -- }
    -- OEM API example:
    -- {
    --   "protocol": "kone_api",
    --   "equipment_id": "KN-NAM-2024-88421",
    --   "metric_name": "door_cycle_time",
    --   "api_endpoint": "site_monitoring_v2"
    -- }
    thresholds      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "warning_low": 0.5,
    --   "warning_high": 4.0,
    --   "critical_low": null,
    --   "critical_high": 6.0,
    --   "anomaly_detection": {"enabled": true, "model": "isolation_forest", "sensitivity": 0.85}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_unit ON sensor(unit_id);
CREATE INDEX idx_sensor_type ON sensor(sensor_type);
CREATE INDEX idx_sensor_config ON sensor USING gin(protocol_config);

CREATE TABLE telemetry_reading (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL,
    unit_id         UUID NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16, 6) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good',
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_telemetry_sensor ON telemetry_reading(sensor_id, recorded_at DESC);
CREATE INDEX idx_telemetry_unit ON telemetry_reading(unit_id, recorded_at DESC);

-- =============================================================
-- INCIDENTS
-- =============================================================

CREATE TABLE incident (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    incident_number VARCHAR(50) NOT NULL,
    incident_type   VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    reported_at     TIMESTAMPTZ NOT NULL,
    reported_by     UUID REFERENCES app_user(id),
    description     TEXT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'reported',
    work_order_id   UUID REFERENCES work_order(id),
    ai_classification JSONB NOT NULL DEFAULT '{}',
    -- Same structure as work_order.ai_classification
    resolution_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, incident_number)
);
CREATE INDEX idx_incident_unit ON incident(unit_id);
CREATE INDEX idx_incident_status ON incident(status);

-- =============================================================
-- QR CODES & PUBLIC FAULT REPORTS
-- =============================================================

CREATE TABLE qr_code_registration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    qr_code_token   VARCHAR(100) NOT NULL UNIQUE,
    label           VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE public_fault_report (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    qr_code_id      UUID NOT NULL REFERENCES qr_code_registration(id),
    unit_id         UUID NOT NULL REFERENCES equipment_unit(id),
    reporter_info   JSONB NOT NULL DEFAULT '{}',
    -- {"name": "...", "phone": "...", "email": "..."}
    fault_description TEXT NOT NULL,
    attachments     JSONB NOT NULL DEFAULT '[]',
    status          VARCHAR(50) NOT NULL DEFAULT 'received',
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- {"field": "status", "old": "open", "new": "completed"}
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"ip": "...", "user_agent": "...", "correlation_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

## JSONB Query Examples

```sql
-- =============================================
-- Find all hydraulic elevators with seismic switches
-- =============================================
SELECT eu.unit_number, b.name AS building, eu.manufacturer
FROM equipment_unit eu
JOIN building b ON b.id = eu.building_id
WHERE eu.equipment_type = 'elevator'
  AND eu.sub_type = 'traction'
  AND eu.equipment_attributes @> '{"seismic_switch": true}';

-- =============================================
-- Find all units with overdue compliance in a specific jurisdiction
-- =============================================
SELECT eu.unit_number, b.name AS building, co.requirement_code, co.next_due_date
FROM compliance_obligation co
JOIN equipment_unit eu ON eu.id = co.unit_id
JOIN building b ON b.id = eu.building_id
JOIN jurisdiction j ON j.id = b.jurisdiction_id
WHERE j.subdivision_code = 'US-IL'
  AND co.status = 'overdue';

-- =============================================
-- Find all sensors using BACnet protocol
-- =============================================
SELECT s.id, eu.unit_number, s.sensor_type, s.protocol_config->>'device_id' AS bacnet_device
FROM sensor s
JOIN equipment_unit eu ON eu.id = s.unit_id
WHERE s.protocol_config @> '{"protocol": "bacnet"}';

-- =============================================
-- Aggregate inspection pass rates from JSONB checklist data
-- =============================================
SELECT
    eu.unit_number,
    i.inspection_date,
    jsonb_array_length(i.checklist_data) AS total_items,
    (SELECT COUNT(*) FROM jsonb_array_elements(i.checklist_data) elem
     WHERE elem->>'result' = 'pass') AS passed_items,
    (SELECT COUNT(*) FROM jsonb_array_elements(i.checklist_data) elem
     WHERE elem->>'result' = 'fail') AS failed_items
FROM inspection i
JOIN equipment_unit eu ON eu.id = i.unit_id
WHERE i.inspection_date >= '2026-01-01';

-- =============================================
-- Find units with KONE OEM integration enabled
-- =============================================
SELECT eu.unit_number, b.name AS building,
       eu.oem_integration->>'kone_equipment_id' AS kone_id
FROM equipment_unit eu
JOIN building b ON b.id = eu.building_id
WHERE eu.oem_integration @> '{"oem_platform": "kone_247"}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | Settings and contact info in JSONB |
| Users & RBAC | 3 | Permissions as JSONB array on role |
| Jurisdiction & Codes | 3 | Local requirements and code requirements in JSONB |
| Buildings & Portfolios | 2 | Address and building info in JSONB |
| Equipment Units | 2 | Equipment/compliance/OEM attributes in JSONB |
| Components | 1 | Type-specific attributes in JSONB |
| Compliance | 1 | Jurisdiction overrides in JSONB |
| Inspections | 1 | Full checklist data in JSONB |
| Deficiencies | 1 | Resolution data in JSONB |
| Work Orders | 1 | Labour, parts, attachments, AI classification in JSONB |
| Contracts | 1 | SLA terms and coverage in JSONB |
| PM Schedules | 1 | Checklist template in JSONB |
| Sensors & Telemetry | 2 | Protocol config and thresholds in JSONB; readings partitioned |
| Incidents | 1 | AI classification in JSONB |
| QR Codes & Fault Reports | 2 | Reporter info in JSONB |
| Audit Log | 1 | Changes and metadata in JSONB (partitioned) |
| **Total** | **~25** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for jurisdiction-specific variability.** Rather than creating tables for every possible local amendment (NYC Appendix K, Chicago amendments, California seismic requirements), the `jurisdiction.local_requirements` and `compliance_obligation.jurisdiction_overrides` JSONB columns absorb these variants. Adding a new jurisdiction is a data entry task, not a schema migration.

2. **JSONB for OEM integration configuration.** Each OEM (Otis, KONE, Schindler, TK Elevator) has different API parameters, equipment identifiers, and monitoring capabilities. The `equipment_unit.oem_integration` JSONB column holds OEM-specific configuration without requiring an OEM-specific table for each vendor.

3. **Inspection checklists as JSONB arrays.** Inspection checklists vary dramatically by inspection type (Cat 1 vs Cat 5), code edition, and equipment type. Storing the full checklist in a JSONB array on the `inspection` table avoids a separate `checklist_item` table with hundreds of thousands of rows. The GIN index enables efficient querying of results within checklists.

4. **Labour and parts embedded in work orders.** Rather than separate `work_order_labour` and `work_order_part` tables, these are stored as JSONB arrays on the work order. This simplifies the API (one POST to complete a work order with all its data) and reduces join complexity for the dispatch board.

5. **Permissions as JSONB array on role.** Instead of a separate `permission` + `role_permission` junction table, permissions are stored as a JSONB string array on the role. This trades referential integrity for simplicity — permission codes are validated at the application layer.

6. **Floor mapping as JSONB on equipment unit.** The floor-to-label mapping (including front/rear door availability per floor) is stored as a JSONB array aligned with the SEIS vocabulary. This avoids a `unit_floor_mapping` junction table while keeping the data structure rich and queryable.

7. **Contract terms as JSONB.** SLA parameters (response time, resolution time, penalties), coverage details, and billing terms vary significantly between contracts. JSONB absorbs this variability cleanly.

8. **Sensor protocol configuration as JSONB.** BACnet, Modbus, OPC-UA, and OEM APIs each require different connection parameters. The `sensor.protocol_config` JSONB column holds protocol-specific fields (BACnet object ID, Modbus register address, KONE equipment ID) without requiring protocol-specific tables.

9. **GIN indexes on all JSONB columns.** Every JSONB column used for querying has a GIN index, enabling efficient containment queries (`@>` operator) and path queries (`->>` operator). This is critical for performance when filtering by JSONB attributes.

10. **Relational core preserved where it matters.** Primary keys, foreign keys, tenant relationships, and universally-queried fields (status, equipment_type, manufacturer, due dates) remain relational columns with B-tree indexes. JSONB is used only for attributes that vary by type, jurisdiction, or OEM — not for core identity or relationships.
