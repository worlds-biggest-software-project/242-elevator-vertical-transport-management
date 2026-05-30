# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Elevator & Vertical Transport Management · Created: 2026-05-22

## Philosophy

This model uses a property graph layer for relationship-heavy queries alongside relational tables for transactional CRUD. The elevator management domain has surprisingly complex relationship patterns: buildings contain elevator groups which contain units which contain components; jurisdictions form hierarchies (country > state > city) with code adoptions that cascade down; technicians are assigned to work orders on units covered by contracts held by organisations within portfolios; deficiencies found during inspections trace to specific components and link to corrective work orders. When you add compliance obligation chains, causal incident analysis, and multi-OEM telemetry routing, the relationship graph becomes the most valuable asset in the data model.

The graph layer is implemented as a pair of generic tables (`graph_node` and `graph_edge`) within PostgreSQL, using JSONB for node/edge properties. This avoids the operational complexity of a separate graph database (Neo4j, Amazon Neptune) while enabling recursive relationship traversal via SQL CTEs. For deployments that need dedicated graph query performance, the same data can be synchronised to a graph database.

This approach is inspired by knowledge graph systems used in asset management (IBM Maximo's asset relationship model), supply chain traceability (GS1 Digital Link), and compliance network analysis (financial crime detection graphs). In the elevator domain, it enables queries that are painful in pure relational models: "find all units whose components share a common deficiency pattern," "trace the compliance impact of a code amendment across all jurisdictions that adopted it," or "which technician has the best resolution rate for door operator failures on KONE equipment?"

**Best for:** Large multi-building portfolios where relationship traversal, impact analysis, compliance dependency tracing, and cross-entity pattern detection are primary use cases — especially for AI-driven risk scoring and predictive analytics.

**Trade-offs:**
- (+) Relationship traversal is first-class: multi-hop queries are natural
- (+) Impact analysis and dependency tracing across equipment, compliance, and organisational boundaries
- (+) Flexible relationship types: new edge types added without schema changes
- (+) Enables AI-powered conflict detection, risk propagation, and pattern analysis on the graph
- (+) PostgreSQL-native: no additional database infrastructure required
- (-) Graph queries via recursive CTEs are less performant than native graph databases for deep traversals (>5 hops)
- (-) Generic node/edge tables lose the self-documenting clarity of named relational tables
- (-) Application layer must enforce relationship constraints that would be implicit in a relational FK model
- (-) More complex to understand for developers unfamiliar with graph patterns
- (-) Reporting and BI tools typically expect relational tables, not graph structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASME A17.1 / CSA B44 | Code editions and their requirements modelled as graph nodes; adoption relationships between jurisdictions and code editions are graph edges with temporal properties |
| EN 81 / ISO 8100 | Additional standard family nodes in the same graph; edges connect overlapping requirements across standards |
| BACnet (ASHRAE 135) | BACnet Elevator Group, Lift, and Escalator objects mapped to graph nodes with CONTAINS_UNIT edges mirroring BACnet group membership |
| SEIS | Equipment configuration properties on graph nodes aligned with SEIS vocabulary |
| ISO 3166 | Jurisdiction hierarchy modelled as a graph with PARENT_OF edges; ISO 3166 codes stored as node properties |
| NFPA 72 / 101 | Fire recall compliance modelled as edges between units and NFPA requirement nodes |
| ISO 25745 | Energy performance rating stored as a property on equipment unit nodes |

---

## Graph Layer (Generic Node/Edge Tables)

```sql
-- =============================================================
-- GRAPH LAYER
-- =============================================================
-- Generic property graph implemented in PostgreSQL.
-- Nodes represent entities; edges represent typed relationships.
-- Both carry JSONB properties for flexible attribute storage.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(100) NOT NULL,
    -- Node types:
    --   'Portfolio', 'Building', 'EquipmentUnit', 'ElevatorGroup',
    --   'Component', 'ComponentType',
    --   'Jurisdiction', 'CodeEdition', 'ComplianceRequirement',
    --   'Inspection', 'Deficiency', 'Permit',
    --   'WorkOrder', 'ServiceContract',
    --   'Organisation', 'User', 'Technician',
    --   'Sensor', 'IoTAlert',
    --   'Incident', 'QRCode'
    label           VARCHAR(255) NOT NULL,     -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(50) DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gn_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_status ON graph_node(status);
CREATE INDEX idx_gn_properties ON graph_node USING gin(properties);
CREATE INDEX idx_gn_label ON graph_node(label);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    --   Structural: 'CONTAINS', 'BELONGS_TO', 'MEMBER_OF', 'PARENT_OF'
    --   Asset: 'INSTALLED_IN', 'REPLACED_BY', 'MANUFACTURED_BY'
    --   Compliance: 'ADOPTS_CODE', 'REQUIRES_TEST', 'GOVERNS', 'SATISFIES'
    --   Inspection: 'INSPECTED_BY', 'FOUND_DEFICIENCY', 'DEFICIENCY_ON_COMPONENT'
    --   Work: 'ASSIGNED_TO', 'RESOLVES', 'COVERS_UNIT', 'CREATED_FROM'
    --   Telemetry: 'MONITORS', 'TRIGGERED_ALERT', 'ALERT_GENERATED_WO'
    --   Causal: 'CAUSED_BY', 'LED_TO', 'RELATED_TO'
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge properties examples:
    --   ADOPTS_CODE: {"adoption_date": "2024-01-15", "enforcement_date": "2024-07-01", "local_amendments": ["Appendix K"]}
    --   ASSIGNED_TO: {"assigned_at": "2026-05-22T08:00:00Z", "priority": "high"}
    --   INSTALLED_IN: {"install_date": "2024-06-15", "position": "machine_room"}
    --   COVERS_UNIT: {"sla_response_hours": 4, "sla_resolution_hours": 24}
    valid_from      TIMESTAMPTZ DEFAULT now(),  -- Temporal validity
    valid_to        TIMESTAMPTZ,                -- NULL = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_properties ON graph_edge USING gin(properties);
CREATE INDEX idx_ge_validity ON graph_edge(valid_from, valid_to);
-- Composite index for common traversal pattern
CREATE INDEX idx_ge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_id, edge_type);
```

## Graph Node Property Examples

```sql
-- =============================================
-- Equipment Unit Node
-- =============================================
-- node_type: 'EquipmentUnit'
-- label: 'E-3 (233 S Wacker Dr)'
-- properties:
-- {
--   "unit_number": "E-3",
--   "equipment_type": "elevator",
--   "sub_type": "traction",
--   "manufacturer": "KONE",
--   "model": "MonoSpace 700",
--   "serial_number": "KN-2024-88421",
--   "installation_date": "2024-06-15",
--   "capacity_kg": 1600,
--   "speed_mps": 2.5,
--   "num_stops": 28,
--   "emergency_phone_number": "+1-800-555-0142",
--   "ada_compliant": true,
--   "energy_rating": "B",
--   "floor_mapping": [
--     {"floor": -2, "label": "P2", "doors": ["front"]},
--     {"floor": 0, "label": "L", "doors": ["front", "rear"], "main_egress": true},
--     {"floor": 27, "label": "28", "doors": ["front"]}
--   ],
--   "oem_integration": {
--     "platform": "kone_247",
--     "equipment_id": "KN-NAM-2024-88421"
--   },
--   "risk_score": 34.5
-- }

-- =============================================
-- Jurisdiction Node
-- =============================================
-- node_type: 'Jurisdiction'
-- label: 'City of Chicago'
-- properties:
-- {
--   "jurisdiction_type": "city",
--   "country_code": "US",
--   "subdivision_code": "US-IL",
--   "city_name": "Chicago",
--   "ahj_name": "City of Chicago Department of Buildings",
--   "ahj_filing_url": "https://www.chicago.gov/dob",
--   "local_amendments": ["Chicago Municipal Code 13-76"],
--   "emergency_phone_registry_required": true
-- }

-- =============================================
-- Code Edition Node
-- =============================================
-- node_type: 'CodeEdition'
-- label: 'ASME A17.1-2025'
-- properties:
-- {
--   "standard_family": "ASME_A17_1",
--   "edition_year": 2025,
--   "effective_date": "2025-01-01",
--   "requirements": [
--     {"code": "CAT1", "name": "Category 1 Periodic Test", "frequency_months": 12, "equipment_types": ["elevator"]},
--     {"code": "CAT5", "name": "Category 5 Periodic Test", "frequency_months": 60, "equipment_types": ["elevator"]},
--     {"code": "FIRE_RECALL", "name": "Fire Recall Test", "frequency_months": 12, "equipment_types": ["elevator"]}
--   ]
-- }

-- =============================================
-- Component Node
-- =============================================
-- node_type: 'Component'
-- label: 'Door Operator (E-3, Front)'
-- properties:
-- {
--   "component_type": "door_operator",
--   "category": "door",
--   "is_safety_critical": true,
--   "manufacturer": "KONE",
--   "model": "DM-220-CO",
--   "serial_number": "KN-DM-2024-00412",
--   "install_date": "2024-06-15",
--   "warranty_expiry": "2027-06-14",
--   "cycle_count": 284500,
--   "attributes": {
--     "operator_type": "center_opening",
--     "drive_type": "linear_motor",
--     "closing_force_n": 120,
--     "opening_time_ms": 1800
--   }
-- }

-- =============================================
-- Inspection Node
-- =============================================
-- node_type: 'Inspection'
-- label: 'Cat 1 - E-3 - 2026-05-22'
-- properties:
-- {
--   "inspection_type": "cat_1",
--   "inspection_date": "2026-05-22",
--   "result": "pass_with_deficiencies",
--   "inspector_name": "J. Martinez",
--   "inspector_licence": "IL-EI-44892",
--   "code_edition": "ASME_A17_1_2025",
--   "certificate_number": "IL-2026-CAT1-88421-0522",
--   "certificate_hash": "a3f8b2c1d4e5...",
--   "checklist_summary": {"total": 47, "passed": 44, "failed": 2, "na": 1}
-- }
```

## Relational Tables (Transactional CRUD)

```sql
-- =============================================================
-- OPERATIONAL RELATIONAL TABLES
-- =============================================================
-- These tables serve transactional workloads where relational
-- constraints, indexing, and standard SQL patterns are essential.
-- They are synchronised with graph nodes via application-layer
-- event handlers.

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- RBAC via graph edges (User -[HAS_ROLE]-> Role -[HAS_PERMISSION]-> Permission)
-- but also maintained relationally for fast auth checks:
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    scope_type      VARCHAR(50),
    scope_id        UUID,
    PRIMARY KEY (user_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

-- =============================================================
-- WORK ORDERS (High-frequency transactional table)
-- =============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID REFERENCES graph_node(id),  -- Link to graph layer
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    unit_id         UUID NOT NULL,                     -- Equipment unit ID (also a graph node)
    wo_number       VARCHAR(50) NOT NULL,
    wo_type         VARCHAR(50) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    summary         VARCHAR(500) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    is_entrapment   BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(50),
    scheduled_date  DATE,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    resolution      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, wo_number)
);
CREATE INDEX idx_wo_tenant ON work_order(tenant_id);
CREATE INDEX idx_wo_unit ON work_order(unit_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_graph ON work_order(graph_node_id);

-- =============================================================
-- TELEMETRY (High-volume time-series — relational for performance)
-- =============================================================

CREATE TABLE telemetry_reading (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    sensor_node_id  UUID NOT NULL,             -- Graph node ID for the sensor
    unit_node_id    UUID NOT NULL,             -- Graph node ID for the unit
    recorded_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16, 6) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good',
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_telemetry_sensor ON telemetry_reading(sensor_node_id, recorded_at DESC);
CREATE INDEX idx_telemetry_unit ON telemetry_reading(unit_node_id, recorded_at DESC);

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
    graph_node_id   UUID,                      -- Graph node reference if applicable
    changes         JSONB NOT NULL DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

## Graph Traversal Queries

```sql
-- =============================================
-- QUERY 1: Full asset hierarchy for a building
-- =============================================
-- Building -> Elevator Groups -> Units -> Components

WITH RECURSIVE asset_tree AS (
    -- Start from a building node
    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.id = '550e8400-e29b-41d4-a716-446655440000'  -- Building ID
      AND gn.node_type = 'Building'

    UNION ALL

    -- Traverse CONTAINS edges downward
    SELECT
        child.id,
        child.node_type,
        child.label,
        child.properties,
        at.depth + 1,
        at.path || child.id
    FROM asset_tree at
    JOIN graph_edge ge ON ge.source_id = at.id
        AND ge.edge_type = 'CONTAINS'
        AND (ge.valid_to IS NULL OR ge.valid_to > now())
    JOIN graph_node child ON child.id = ge.target_id
    WHERE at.depth < 5  -- Prevent infinite loops
)
SELECT depth, node_type, label,
       properties->>'unit_number' AS unit_number,
       properties->>'component_type' AS component_type,
       properties->>'status' AS status
FROM asset_tree
ORDER BY depth, node_type, label;

-- =============================================
-- QUERY 2: Compliance impact of a code amendment
-- =============================================
-- CodeEdition -> (ADOPTS_CODE) -> Jurisdictions -> (GOVERNS) -> Buildings -> (CONTAINS) -> Units

WITH code_amendment AS (
    SELECT id FROM graph_node
    WHERE node_type = 'CodeEdition'
      AND properties->>'edition_label' = 'ASME A17.1-2025'
),
affected_jurisdictions AS (
    SELECT ge.target_id AS jurisdiction_id, ge.properties AS adoption_info
    FROM graph_edge ge
    JOIN code_amendment ca ON ge.source_id = ca.id
    WHERE ge.edge_type = 'ADOPTS_CODE'
      AND (ge.valid_to IS NULL OR ge.valid_to > now())
),
affected_buildings AS (
    SELECT ge2.target_id AS building_id, aj.jurisdiction_id
    FROM affected_jurisdictions aj
    JOIN graph_edge ge2 ON ge2.source_id = aj.jurisdiction_id
    WHERE ge2.edge_type = 'GOVERNS'
),
affected_units AS (
    SELECT ge3.target_id AS unit_id, ab.building_id
    FROM affected_buildings ab
    JOIN graph_edge ge3 ON ge3.source_id = ab.building_id
    WHERE ge3.edge_type = 'CONTAINS'
    AND (SELECT node_type FROM graph_node WHERE id = ge3.target_id) = 'EquipmentUnit'
)
SELECT
    gn_unit.label AS unit,
    gn_unit.properties->>'equipment_type' AS type,
    gn_unit.properties->>'manufacturer' AS manufacturer,
    gn_bldg.label AS building,
    gn_jur.label AS jurisdiction
FROM affected_units au
JOIN graph_node gn_unit ON gn_unit.id = au.unit_id
JOIN graph_node gn_bldg ON gn_bldg.id = au.building_id
JOIN graph_node gn_jur ON gn_jur.id = (
    SELECT aj2.jurisdiction_id FROM affected_jurisdictions aj2
    JOIN affected_buildings ab2 ON ab2.jurisdiction_id = aj2.jurisdiction_id
    WHERE ab2.building_id = au.building_id
    LIMIT 1
);

-- =============================================
-- QUERY 3: Deficiency pattern analysis
-- =============================================
-- Find components that share similar deficiency patterns across units

SELECT
    comp_node.properties->>'component_type' AS component_type,
    comp_node.properties->>'manufacturer' AS manufacturer,
    def_node.properties->>'deficiency_code' AS deficiency_code,
    def_node.properties->>'severity' AS severity,
    COUNT(DISTINCT unit_edge.source_id) AS affected_unit_count,
    array_agg(DISTINCT
        (SELECT label FROM graph_node WHERE id = unit_edge.source_id)
    ) AS affected_units
FROM graph_node def_node
JOIN graph_edge comp_edge ON comp_edge.source_id = def_node.id
    AND comp_edge.edge_type = 'DEFICIENCY_ON_COMPONENT'
JOIN graph_node comp_node ON comp_node.id = comp_edge.target_id
JOIN graph_edge unit_edge ON unit_edge.target_id = comp_node.id
    AND unit_edge.edge_type = 'CONTAINS'
WHERE def_node.node_type = 'Deficiency'
  AND def_node.properties->>'status' != 'resolved'
GROUP BY comp_node.properties->>'component_type',
         comp_node.properties->>'manufacturer',
         def_node.properties->>'deficiency_code',
         def_node.properties->>'severity'
HAVING COUNT(DISTINCT unit_edge.source_id) > 1
ORDER BY affected_unit_count DESC;

-- =============================================
-- QUERY 4: Technician expertise graph
-- =============================================
-- Find the best technician for a specific component failure type
-- by traversing: Technician -[ASSIGNED_TO]-> WorkOrder -[RESOLVES]-> Deficiency -[DEFICIENCY_ON_COMPONENT]-> Component

SELECT
    tech.label AS technician,
    comp_node.properties->>'component_type' AS component_type,
    COUNT(*) AS resolved_count,
    AVG(EXTRACT(EPOCH FROM (
        (wo.properties->>'actual_end')::TIMESTAMPTZ -
        (wo.properties->>'actual_start')::TIMESTAMPTZ
    )) / 3600) AS avg_resolution_hours
FROM graph_node tech
JOIN graph_edge assign ON assign.target_id = tech.id
    AND assign.edge_type = 'ASSIGNED_TO'
JOIN graph_node wo ON wo.id = assign.source_id
    AND wo.node_type = 'WorkOrder'
    AND wo.properties->>'status' = 'completed'
JOIN graph_edge resolves ON resolves.source_id = wo.id
    AND resolves.edge_type = 'RESOLVES'
JOIN graph_node def ON def.id = resolves.target_id
    AND def.node_type = 'Deficiency'
JOIN graph_edge on_comp ON on_comp.source_id = def.id
    AND on_comp.edge_type = 'DEFICIENCY_ON_COMPONENT'
JOIN graph_node comp_node ON comp_node.id = on_comp.target_id
WHERE tech.node_type = 'User'
  AND wo.properties->>'actual_end' IS NOT NULL
GROUP BY tech.label, comp_node.properties->>'component_type'
ORDER BY resolved_count DESC;

-- =============================================
-- QUERY 5: Jurisdiction hierarchy traversal
-- =============================================
-- Get the full jurisdiction chain for a building (city -> state -> country)
-- and collect all applicable code editions at each level

WITH RECURSIVE jurisdiction_chain AS (
    -- Start from the building's jurisdiction
    SELECT
        gn.id AS jurisdiction_id,
        gn.label AS jurisdiction_name,
        gn.properties->>'jurisdiction_type' AS jurisdiction_type,
        0 AS depth
    FROM graph_node gn
    JOIN graph_edge ge ON ge.target_id = gn.id
        AND ge.edge_type = 'GOVERNS'
        AND ge.source_id = '550e8400-e29b-41d4-a716-446655440000'  -- Building ID
    WHERE gn.node_type = 'Jurisdiction'

    UNION ALL

    -- Traverse PARENT_OF edges upward
    SELECT
        parent.id,
        parent.label,
        parent.properties->>'jurisdiction_type',
        jc.depth + 1
    FROM jurisdiction_chain jc
    JOIN graph_edge ge ON ge.target_id = jc.jurisdiction_id
        AND ge.edge_type = 'PARENT_OF'
    JOIN graph_node parent ON parent.id = ge.source_id
    WHERE jc.depth < 5
)
SELECT
    jc.jurisdiction_name,
    jc.jurisdiction_type,
    jc.depth,
    code.label AS adopted_code,
    adopt.properties->>'adoption_date' AS adoption_date,
    adopt.properties->'local_amendments' AS local_amendments
FROM jurisdiction_chain jc
LEFT JOIN graph_edge adopt ON adopt.target_id = jc.jurisdiction_id
    AND adopt.edge_type = 'ADOPTS_CODE'
    AND (adopt.valid_to IS NULL OR adopt.valid_to > now())
LEFT JOIN graph_node code ON code.id = adopt.source_id
ORDER BY jc.depth ASC;

-- =============================================
-- QUERY 6: Risk propagation — units at risk from a component recall
-- =============================================
-- Given a component type and manufacturer with a known defect,
-- find all units with that component installed

SELECT
    unit_node.label AS unit,
    unit_node.properties->>'equipment_type' AS type,
    bldg_node.label AS building,
    comp_node.properties->>'serial_number' AS component_serial,
    comp_node.properties->>'install_date' AS installed,
    comp_node.properties->>'cycle_count' AS cycles
FROM graph_node comp_node
JOIN graph_edge installed ON installed.target_id = comp_node.id
    AND installed.edge_type = 'CONTAINS'
JOIN graph_node unit_node ON unit_node.id = installed.source_id
    AND unit_node.node_type = 'EquipmentUnit'
JOIN graph_edge in_building ON in_building.target_id = unit_node.id
    AND in_building.edge_type = 'CONTAINS'
JOIN graph_node bldg_node ON bldg_node.id = in_building.source_id
    AND bldg_node.node_type = 'Building'
WHERE comp_node.node_type = 'Component'
  AND comp_node.properties->>'component_type' = 'governor'
  AND comp_node.properties->>'manufacturer' = 'Hollister-Whitney'
  AND comp_node.properties->>'model' = 'GV-300'
ORDER BY (comp_node.properties->>'cycle_count')::BIGINT DESC;
```

## Graph Edge Relationship Catalogue

```
STRUCTURAL RELATIONSHIPS:
  Portfolio    --[CONTAINS]-->     Building
  Building     --[CONTAINS]-->     ElevatorGroup
  Building     --[CONTAINS]-->     EquipmentUnit
  ElevatorGroup--[CONTAINS]-->     EquipmentUnit
  EquipmentUnit--[CONTAINS]-->     Component
  Jurisdiction --[PARENT_OF]-->    Jurisdiction    (country > state > city)
  Jurisdiction --[GOVERNS]-->      Building

COMPLIANCE RELATIONSHIPS:
  CodeEdition  --[ADOPTS_CODE]-->  Jurisdiction    (with adoption_date, local_amendments)
  CodeEdition  --[REQUIRES_TEST]--> ComplianceRequirement
  CodeEdition  --[SUPERSEDES]-->   CodeEdition
  EquipmentUnit--[SUBJECT_TO]-->   ComplianceRequirement
  Inspection   --[SATISFIES]-->    ComplianceRequirement
  Inspection   --[INSPECTED]-->    EquipmentUnit
  Permit       --[PERMITS]-->      EquipmentUnit
  Permit       --[ISSUED_BY]-->    Jurisdiction

INSPECTION & DEFICIENCY:
  Inspection   --[FOUND_DEFICIENCY]--> Deficiency
  Deficiency   --[DEFICIENCY_ON_COMPONENT]--> Component
  WorkOrder    --[RESOLVES]-->     Deficiency

WORK & ASSIGNMENT:
  WorkOrder    --[ON_UNIT]-->      EquipmentUnit
  WorkOrder    --[ASSIGNED_TO]-->  User
  WorkOrder    --[CREATED_FROM]--> Incident
  WorkOrder    --[CREATED_FROM]--> IoTAlert
  WorkOrder    --[CREATED_FROM]--> PublicFaultReport

CONTRACT:
  ServiceContract --[COVERS_UNIT]--> EquipmentUnit (with SLA properties)
  ServiceContract --[BETWEEN]-->   Organisation   (contractor side)
  ServiceContract --[FOR]-->       Organisation   (client side)
  Organisation    --[EMPLOYS]-->   User

ORGANISATIONAL:
  User         --[HAS_ROLE]-->     Role           (with scope_type, scope_id)
  User         --[MEMBER_OF]-->    Organisation
  Organisation --[OWNS]-->         Portfolio

IoT & TELEMETRY:
  Sensor       --[MONITORS]-->     Component
  Sensor       --[INSTALLED_IN]--> EquipmentUnit
  IoTAlert     --[TRIGGERED_BY]--> Sensor
  IoTAlert     --[ALERT_ON]-->     EquipmentUnit

CAUSAL / ANALYTICAL:
  Incident     --[OCCURRED_ON]-->  EquipmentUnit
  Incident     --[INVOLVED_COMPONENT]--> Component
  Deficiency   --[RELATED_TO]-->   Deficiency     (cross-unit pattern links)
  Component    --[SAME_TYPE_AS]--> Component      (for fleet-wide analysis)
  CodeEdition  --[AMENDMENT_OF]--> CodeEdition     (code change tracking)
```

## Temporal Edge Management

```sql
-- =============================================
-- Temporal edges allow historical relationship queries
-- =============================================

-- When a component is replaced, the old CONTAINS edge is ended
-- and a new one is created:

-- End old relationship
UPDATE graph_edge
SET valid_to = now()
WHERE source_id = 'unit-uuid'
  AND target_id = 'old-component-uuid'
  AND edge_type = 'CONTAINS'
  AND valid_to IS NULL;

-- Create new relationship
INSERT INTO graph_edge (tenant_id, edge_type, source_id, target_id, properties, valid_from)
VALUES ('tenant-uuid', 'CONTAINS', 'unit-uuid', 'new-component-uuid',
        '{"install_date": "2026-05-22", "position": "machine_room"}', now());

-- Create replacement chain
INSERT INTO graph_edge (tenant_id, edge_type, source_id, target_id, properties)
VALUES ('tenant-uuid', 'REPLACED_BY', 'old-component-uuid', 'new-component-uuid',
        '{"replacement_date": "2026-05-22", "reason": "bearing_failure", "work_order_id": "wo-uuid"}');

-- =============================================
-- Query: What components were installed on a specific date?
-- =============================================
SELECT comp.label, comp.properties->>'component_type', ge.properties->>'install_date'
FROM graph_edge ge
JOIN graph_node comp ON comp.id = ge.target_id
WHERE ge.source_id = 'unit-uuid'
  AND ge.edge_type = 'CONTAINS'
  AND ge.valid_from <= '2025-06-01'
  AND (ge.valid_to IS NULL OR ge.valid_to > '2025-06-01');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | Generic node and edge tables |
| Tenant & Users | 2 | Relational identity tables |
| RBAC | 2 | Roles and user-role assignments (also modelled as graph edges) |
| Work Orders | 1 | High-frequency transactional table with graph_node_id FK |
| Telemetry | 1 | Partitioned time-series readings |
| Audit Log | 1 | Partitioned audit trail |
| **Total** | **~9 tables** | Plus graph nodes/edges that represent all other entities |

---

## Key Design Decisions

1. **Two-table graph layer.** All entities (buildings, units, components, inspections, deficiencies, permits, contracts, organisations, users) are represented as nodes in `graph_node`. All relationships are edges in `graph_edge`. This enables arbitrary relationship traversal without schema changes — adding a new relationship type (e.g., `ROBOT_INTERFACES_WITH` for autonomous vehicle integration) is a data operation, not a DDL change.

2. **Temporal edges for compliance auditing.** Every edge has `valid_from` and `valid_to` timestamps. When a component is replaced, the old `CONTAINS` edge is ended and a new one created. When a jurisdiction adopts a new code edition, the old `ADOPTS_CODE` edge is ended. This enables point-in-time queries: "what was installed in this unit on 2025-01-01?"

3. **Relational work order table for transactional performance.** Work orders are high-frequency transactional records (create, update status, assign, complete). A relational table with proper indexes outperforms graph queries for CRUD operations. The `graph_node_id` FK links the relational record to its graph representation for relationship queries.

4. **Relational telemetry for time-series performance.** Sensor readings at 200+ data points per car per hour cannot be efficiently stored as graph nodes. They remain in a partitioned relational table, with sensor graph nodes providing the relationship context (which sensor monitors which component on which unit).

5. **Causal edge chains.** The `CAUSED_BY`, `LED_TO`, and `RELATED_TO` edge types enable causal analysis: an `IoTAlert` node connected via `TRIGGERED_BY` to a `Sensor` node, then via `ALERT_GENERATED_WO` to a `WorkOrder` node, which `RESOLVES` a `Deficiency` node. This chain is trivially traversable in the graph but would require multiple joins across 4+ tables in a normalized relational model.

6. **Deficiency pattern detection across the fleet.** The `DEFICIENCY_ON_COMPONENT` and `SAME_TYPE_AS` edges enable fleet-wide analysis: "across all KONE MonoSpace 700 units, which component type has the highest deficiency rate?" This query traverses the graph rather than joining normalised tables.

7. **Jurisdiction hierarchy as graph.** The PARENT_OF edges between jurisdiction nodes (City of Chicago -> State of Illinois -> United States) enable recursive code applicability queries. Combined with ADOPTS_CODE edges, the system can determine which code edition applies at any level and whether local amendments override parent jurisdiction requirements.

8. **Dual representation for auth-critical data.** RBAC is maintained both as graph edges (User -[HAS_ROLE]-> Role) for relationship queries and as relational tables (user_role) for fast authentication checks. The application layer keeps these in sync.

9. **GIN indexes on all JSONB properties.** Both `graph_node.properties` and `graph_edge.properties` have GIN indexes, enabling efficient containment queries (e.g., find all nodes where `properties @> '{"manufacturer": "KONE"}'`).

10. **Compatible with graph database migration.** If the PostgreSQL recursive CTE approach proves insufficient for deep traversals (e.g., multi-hop risk propagation across thousands of nodes), the `graph_node` and `graph_edge` tables can be exported to Neo4j or Amazon Neptune. The schema is designed to be a direct mapping to a labeled property graph model.
