# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Manufacturing ERP · Created: 2026-05-12

## Philosophy

This model takes a pragmatic middle ground: core manufacturing entities (items, BOMs, work orders, inventory) use typed relational columns with full foreign key constraints, while variable, tenant-specific, and jurisdiction-dependent data is stored in JSONB columns on each table. The result is a schema that is structured enough for referential integrity and SQL joins, but flexible enough to accommodate the enormous variation in real-world manufacturing without requiring a migration for every new field.

Manufacturing is inherently variable. A CNC machine shop tracking tool offsets and fixture assignments needs different work order fields than a PCB assembly line tracking solder paste lot numbers and reflow profiles. An aerospace tier-2 supplier needs AS9100 first-article inspection fields that a furniture manufacturer does not. A European manufacturer must track REACH/RoHS compliance data that a US job shop ignores. The hybrid model handles all these variations through JSONB extension columns (`properties`, `custom_fields`, `compliance_data`) rather than forcing every possible field into the relational schema or requiring custom tables per tenant.

This approach draws from PostgreSQL's strength as the industry's leading hybrid relational-document database. ERPNext's DocType system and Odoo's dynamic field model are conceptual predecessors — both allow adding fields without schema migration. The JSONB approach achieves similar flexibility at the database level while preserving the relational core that makes manufacturing queries (BOM explosion, MRP netting, lot traceability) performant and correct.

**Best for:** Rapid MVP development, multi-market deployment where jurisdictions require different compliance fields, and teams that want to iterate on the schema quickly without heavyweight migrations.

**Trade-offs:**
- (+) Core data integrity preserved through relational columns and foreign keys
- (+) New fields can be added to any entity without database migration
- (+) Jurisdiction-specific compliance data (AS9100, IATF 16949, REACH/RoHS) lives in JSONB without polluting the core schema
- (+) JSONB GIN indexes enable fast containment queries on flexible fields
- (+) Lower table count than full normalization — simpler to understand and deploy
- (+) PostgreSQL JSONB operators are mature, well-documented, and widely understood
- (-) No database-level type checking or NOT NULL constraints on JSONB fields
- (-) Application layer must validate JSONB structure (JSON Schema recommended)
- (-) JSONB fields are harder to report on with standard BI tools
- (-) Migration of data within JSONB columns must be handled in application code
- (-) Cannot create foreign keys on values inside JSONB columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Core ISA-95 resource entities (personnel, equipment, material) are relational; ISA-95 property bags (variable per equipment class, material class) map to JSONB `properties` columns |
| ISO 9001:2015 | Quality workflow tables are fully relational; compliance-specific fields (certification bodies, accreditation details) are JSONB |
| AS9100 Rev D | Aerospace-specific fields (FAI data, PPAP data, airworthiness references) stored in `compliance_data` JSONB on inspection and item records |
| IATF 16949 | Automotive-specific fields (FMEA references, PPAP status, customer-specific requirements) stored in `compliance_data` JSONB |
| GS1 / GTIN / UDI | GTIN is a typed relational column; UDI and extended GS1 attributes in `properties` JSONB |
| MTConnect / OPC-UA | Equipment connectivity config and device-specific attributes stored in `properties` JSONB per equipment record |
| REACH / RoHS | Chemical compliance data stored in `compliance_data` JSONB on material items — varies widely by jurisdiction |

---

## Core Tables

```sql
-- ============================================================
-- MULTI-TENANCY & IDENTITY
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    locale          TEXT NOT NULL DEFAULT 'en-US',
    -- Tenant-level configuration in JSONB
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "fiscal_year_start": "01-01",
    --   "numbering_series": {
    --     "work_order": "WO-{YYYY}-{SEQ:5}",
    --     "ncr": "NCR-{YYYY}-{SEQ:4}"
    --   },
    --   "quality_standards": ["ISO9001", "AS9100"],
    --   "compliance_requirements": ["ITAR", "REACH"],
    --   "shop_floor_config": {
    --     "auto_backflush": true,
    --     "require_operator_login": true,
    --     "telemetry_interval_secs": 5
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    roles           JSONB NOT NULL DEFAULT '[]',
    -- roles example: ["plant_manager", "quality_reviewer"]
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {"dashboard": "production", "theme": "dark", "locale": "de-DE"}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- Row-Level Security
ALTER TABLE "user" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "user"
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

---

## Items & Materials

```sql
-- ============================================================
-- ITEMS & MATERIALS
-- ============================================================

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_number     TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    item_type       TEXT NOT NULL
        CHECK (item_type IN ('raw_material', 'component', 'sub_assembly',
                              'finished_good', 'consumable', 'tooling')),
    category        TEXT,                   -- user-defined category
    uom             TEXT NOT NULL,
    -- GS1 identification
    gtin            TEXT,
    -- Costing (relational — queried frequently)
    standard_cost   NUMERIC(14,4),
    last_cost       NUMERIC(14,4),
    -- Planning parameters (relational — used by MRP engine)
    lead_time_days  INTEGER DEFAULT 0,
    safety_stock    NUMERIC(14,4) DEFAULT 0,
    reorder_point   NUMERIC(14,4) DEFAULT 0,
    lot_size_min    NUMERIC(14,4) DEFAULT 1,
    lot_size_max    NUMERIC(14,4),
    lot_size_multiple NUMERIC(14,4) DEFAULT 1,
    -- Tracking method
    lot_tracked     BOOLEAN NOT NULL DEFAULT false,
    serial_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- ===== JSONB EXTENSION COLUMNS =====

    -- Variable item properties (varies by item_type and industry)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples by item_type:
    -- Raw material: {"material_spec": "ASTM A36", "grade": "Grade 50",
    --                "density_kg_m3": 7850, "hardness_hrc": 22}
    -- Electronic component: {"package": "0805", "voltage_rating": "50V",
    --                         "tolerance": "5%", "manufacturer_pn": "RC0805FR-07100KL"}
    -- Finished good: {"weight_kg": 12.5, "dimensions_mm": {"l": 300, "w": 200, "h": 150},
    --                  "color": "RAL 7035", "finish": "powder_coat"}

    -- Compliance data (varies by jurisdiction and quality standard)
    compliance_data JSONB NOT NULL DEFAULT '{}',
    -- compliance_data examples:
    -- Aerospace: {"cage_code": "1ABC2", "nsn": "5305-01-123-4567",
    --             "itar_controlled": true, "eccn": "EAR99"}
    -- Automotive: {"ppap_level": 3, "ppap_status": "approved",
    --              "customer_part_number": "GM-12345-A"}
    -- EU: {"reach_compliant": true, "rohs_compliant": true,
    --       "reach_substances": [{"name": "Lead", "cas": "7439-92-1", "ppm": 50}],
    --       "ce_marking": true}

    -- Custom fields defined by the tenant
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields example: {"internal_class": "A", "buyer_code": "JD",
    --                          "tariff_code": "8482.10.50"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_number)
);

CREATE INDEX idx_item_type ON item(item_type);
CREATE INDEX idx_item_gtin ON item(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_item_category ON item(tenant_id, category);
-- GIN index for JSONB containment queries
CREATE INDEX idx_item_properties ON item USING GIN (properties);
CREATE INDEX idx_item_compliance ON item USING GIN (compliance_data);
CREATE INDEX idx_item_custom ON item USING GIN (custom_fields);
```

---

## Bill of Materials

```sql
-- ============================================================
-- BILL OF MATERIALS
-- ============================================================

CREATE TABLE bom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    revision        TEXT NOT NULL DEFAULT 'A',
    name            TEXT NOT NULL,
    description     TEXT,
    bom_type        TEXT NOT NULL DEFAULT 'manufacturing'
        CHECK (bom_type IN ('manufacturing', 'engineering', 'planning', 'costing')),
    output_quantity NUMERIC(14,4) NOT NULL DEFAULT 1,
    uom             TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'superseded', 'obsolete')),
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    -- JSONB for revision metadata and ECO references
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "eco_number": "ECO-2026-042",
    --   "change_reason": "Customer requested material substitution",
    --   "approved_by": "Jane Smith",
    --   "approved_at": "2026-05-10T14:30:00Z",
    --   "previous_revision": "uuid-of-rev-A"
    -- }
    created_by      UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_id, revision)
);

CREATE TABLE bom_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bom_id          UUID NOT NULL REFERENCES bom(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    component_item_id UUID NOT NULL REFERENCES item(id),
    quantity        NUMERIC(14,6) NOT NULL,
    uom             TEXT NOT NULL,
    scrap_factor    NUMERIC(5,4) DEFAULT 0,
    is_phantom      BOOLEAN NOT NULL DEFAULT false,
    operation_seq   INTEGER,
    effective_from  DATE,
    effective_to    DATE,
    -- JSONB for line-specific notes and alternates
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "reference_designators": ["R1", "R2", "R3"],
    --   "alternate_items": [
    --     {"item_number": "RES-100K-0805-ALT", "priority": 1, "notes": "Yageo equivalent"}
    --   ],
    --   "manufacturing_notes": "Apply threadlocker to fastener",
    --   "critical_to_quality": true,
    --   "supply_type": "stock"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (bom_id, line_number)
);

CREATE INDEX idx_bom_item ON bom(item_id);
CREATE INDEX idx_bom_line_component ON bom_line(component_item_id);
```

---

## Work Centers & Equipment

```sql
-- ============================================================
-- WORK CENTERS & EQUIPMENT
-- ============================================================

CREATE TABLE work_center (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    capacity_units  TEXT NOT NULL DEFAULT 'hours',
    available_hours_per_day NUMERIC(5,2) NOT NULL DEFAULT 8.0,
    cost_rate       NUMERIC(12,4),
    overhead_rate   NUMERIC(12,4),
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'maintenance')),
    -- JSONB for schedule and capability details
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "shift_schedule": [
    --     {"shift": "day", "start": "06:00", "end": "14:00", "days": [1,2,3,4,5]},
    --     {"shift": "evening", "start": "14:00", "end": "22:00", "days": [1,2,3,4,5]}
    --   ],
    --   "capabilities": ["turning", "boring", "threading"],
    --   "max_weight_kg": 500,
    --   "floor_area_sqm": 25
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_center_id  UUID REFERENCES work_center(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    serial_number   TEXT,
    manufacturer    TEXT,
    model           TEXT,
    status          TEXT NOT NULL DEFAULT 'operational'
        CHECK (status IN ('operational', 'maintenance', 'breakdown', 'decommissioned')),
    commissioned_date DATE,
    -- JSONB for machine-specific attributes and IoT config
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (CNC lathe):
    -- {
    --   "type": "CNC Lathe",
    --   "axes": 2,
    --   "max_rpm": 6000,
    --   "max_bar_diameter_mm": 65,
    --   "chuck_size_mm": 254,
    --   "turret_stations": 12,
    --   "controller": "Fanuc 0i-TF Plus",
    --   "connectivity": {
    --     "mtconnect_url": "http://192.168.1.100:5000/current",
    --     "opcua_endpoint": "opc.tcp://192.168.1.100:4840",
    --     "protocol_priority": "mtconnect"
    --   },
    --   "maintenance": {
    --     "last_service_date": "2026-04-15",
    --     "next_service_date": "2026-07-15",
    --     "service_interval_hours": 2000,
    --     "current_hours": 14523
    --   },
    --   "calibration": {
    --     "last_calibrated": "2026-03-01",
    --     "next_calibration": "2026-06-01",
    --     "calibration_standard": "ISO 230-2"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_equipment_wc ON equipment(work_center_id);
CREATE INDEX idx_equipment_status ON equipment(status);
CREATE INDEX idx_equipment_props ON equipment USING GIN (properties);
```

---

## Routing

```sql
-- ============================================================
-- ROUTING
-- ============================================================

CREATE TABLE routing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    revision        TEXT NOT NULL DEFAULT '1',
    name            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'superseded', 'obsolete')),
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routing_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    routing_id      UUID NOT NULL REFERENCES routing(id) ON DELETE CASCADE,
    sequence        INTEGER NOT NULL,
    name            TEXT NOT NULL,
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    setup_time_mins NUMERIC(10,2) NOT NULL DEFAULT 0,
    run_time_mins   NUMERIC(10,2) NOT NULL DEFAULT 0,
    teardown_time_mins NUMERIC(10,2) NOT NULL DEFAULT 0,
    overlap_percent NUMERIC(5,2) DEFAULT 0,
    inspection_required BOOLEAN NOT NULL DEFAULT false,
    -- JSONB for operation-specific details
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "required_skills": ["cnc_programming", "gd_t_reading"],
    --   "required_tooling": ["T01-CNMG-120408", "T02-DNMG-150412"],
    --   "required_fixtures": ["FIX-V-BLOCK-006"],
    --   "setup_instructions": "Load program O1234. Set work offset G54 to datum A.",
    --   "quality_checks": [
    --     {"dimension": "OD", "nominal": 50.000, "tol_upper": 0.025, "tol_lower": -0.025, "gauge": "Micrometer 0-75mm"}
    --   ],
    --   "labor_rate_override": 85.00,
    --   "subcontract": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (routing_id, sequence)
);
```

---

## Production & Work Orders

```sql
-- ============================================================
-- WORK ORDERS
-- ============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    wo_number       TEXT NOT NULL,
    item_id         UUID NOT NULL REFERENCES item(id),
    bom_id          UUID NOT NULL REFERENCES bom(id),
    routing_id      UUID REFERENCES routing(id),
    quantity_planned NUMERIC(14,4) NOT NULL,
    quantity_completed NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_scrapped NUMERIC(14,4) NOT NULL DEFAULT 0,
    uom             TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'released', 'in_progress', 'on_hold',
                          'completed', 'closed', 'cancelled')),
    priority        INTEGER NOT NULL DEFAULT 500,
    planned_start   TIMESTAMPTZ NOT NULL,
    planned_end     TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    lot_number      TEXT,
    -- Costing
    estimated_cost  NUMERIC(14,4),
    actual_material_cost NUMERIC(14,4) DEFAULT 0,
    actual_labor_cost    NUMERIC(14,4) DEFAULT 0,
    actual_overhead_cost NUMERIC(14,4) DEFAULT 0,
    -- JSONB for variable work order data
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "customer_po": "PO-ACME-2026-0891",
    --   "customer_name": "ACME Industries",
    --   "ship_date": "2026-05-25",
    --   "sales_order_ref": "SO-2026-00234",
    --   "scheduling": {
    --     "scheduled_by": "ai_scheduler",
    --     "schedule_confidence": 0.87,
    --     "constraint_notes": "Bottleneck at WC-GRIND-01; shifted +4h"
    --   },
    --   "subcontract": {
    --     "supplier": "Accu-Finish Plating",
    --     "operation": "Nickel Plating",
    --     "po_number": "PO-2026-00567"
    --   },
    --   "special_instructions": "Customer requires 100% inspection per AS9102"
    -- }
    created_by      UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, wo_number)
);

CREATE INDEX idx_wo_item ON work_order(item_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_dates ON work_order(planned_start, planned_end);
CREATE INDEX idx_wo_props ON work_order USING GIN (properties);

CREATE TABLE work_order_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    quantity_required NUMERIC(14,6) NOT NULL,
    quantity_issued NUMERIC(14,6) NOT NULL DEFAULT 0,
    uom             TEXT NOT NULL,
    lot_number      TEXT,
    serial_number   TEXT,
    backflush       BOOLEAN NOT NULL DEFAULT false,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- JOB CARDS
-- ============================================================

CREATE TABLE job_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jc_number       TEXT NOT NULL,
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    operation_id    UUID NOT NULL REFERENCES routing_operation(id),
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    equipment_id    UUID REFERENCES equipment(id),
    operator_id     UUID REFERENCES "user"(id),
    status          TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'ready', 'in_progress', 'paused',
                          'completed', 'cancelled')),
    quantity_planned NUMERIC(14,4) NOT NULL,
    quantity_completed NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_scrapped NUMERIC(14,4) NOT NULL DEFAULT 0,
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    setup_time_mins NUMERIC(10,2) DEFAULT 0,
    run_time_mins   NUMERIC(10,2) DEFAULT 0,
    -- JSONB for operator-reported data and AI input
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "operator_notes": "Tool insert chipped at piece 42, replaced T01",
    --   "scrap_details": [
    --     {"quantity": 1, "reason": "surface_finish", "disposition": "scrap"},
    --     {"quantity": 1, "reason": "diameter_oor", "disposition": "rework"}
    --   ],
    --   "tool_changes": [
    --     {"tool": "T01-CNMG-120408", "changed_at_piece": 42, "reason": "chipped"}
    --   ],
    --   "machine_readings_at_completion": {
    --     "spindle_hours": 14528,
    --     "cycle_count": 48,
    --     "avg_cycle_time_sec": 145
    --   },
    --   "nl_input_transcript": "Finished 48 pieces on the Mazak, scrapped 2 — one surface finish and one OD was out"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, jc_number)
);

CREATE INDEX idx_jc_wo ON job_card(work_order_id);
CREATE INDEX idx_jc_status ON job_card(status);
```

---

## Inventory

```sql
-- ============================================================
-- INVENTORY
-- ============================================================

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"address": "123 Factory Rd", "type": "main",
    --                       "bin_naming": "AISLE-RACK-BIN"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE inventory_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_code   TEXT,                   -- bin/rack/aisle code
    lot_number      TEXT,
    quantity_on_hand NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(14,4) GENERATED ALWAYS AS
        (quantity_on_hand - quantity_allocated) STORED,
    uom             TEXT NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"supplier_lot": "SUP-LOT-2026-001",
    --                       "received_date": "2026-05-01",
    --                       "expiry_date": "2027-05-01",
    --                       "certificate_of_conformance": true}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_item ON inventory_balance(tenant_id, item_id);
CREATE INDEX idx_inv_warehouse ON inventory_balance(warehouse_id);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    transaction_type TEXT NOT NULL
        CHECK (transaction_type IN ('receipt', 'issue', 'transfer', 'adjustment',
                                     'scrap', 'return', 'backflush', 'count')),
    quantity        NUMERIC(14,4) NOT NULL,
    uom             TEXT NOT NULL,
    warehouse_id    UUID REFERENCES warehouse(id),
    location_code   TEXT,
    lot_number      TEXT,
    serial_number   TEXT,
    work_order_id   UUID REFERENCES work_order(id),
    reference_type  TEXT,
    reference_id    UUID,
    cost            NUMERIC(14,4),
    transacted_by   UUID REFERENCES "user"(id),
    transacted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for transaction context
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example: {"purchase_order": "PO-2026-00123",
    --                       "supplier": "Steel Service Center Inc",
    --                       "receiving_inspection": "pending",
    --                       "bol_number": "BOL-12345"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_item ON inventory_transaction(item_id);
CREATE INDEX idx_inv_txn_date ON inventory_transaction(transacted_at);
CREATE INDEX idx_inv_txn_wo ON inventory_transaction(work_order_id);
```

---

## Quality Management

```sql
-- ============================================================
-- QUALITY MANAGEMENT
-- ============================================================

CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    inspection_type TEXT NOT NULL
        CHECK (inspection_type IN ('incoming', 'in_process', 'final', 'first_article')),
    item_id         UUID REFERENCES item(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Checks defined as a JSONB array for maximum flexibility
    checks          JSONB NOT NULL DEFAULT '[]',
    -- checks example:
    -- [
    --   {
    --     "seq": 1,
    --     "name": "Outside Diameter",
    --     "type": "numeric",
    --     "uom": "mm",
    --     "nominal": 50.000,
    --     "tol_upper": 0.025,
    --     "tol_lower": -0.025,
    --     "gauge": "Micrometer 0-75mm",
    --     "sample_size": 5,
    --     "frequency": "every_10_parts"
    --   },
    --   {
    --     "seq": 2,
    --     "name": "Surface Finish",
    --     "type": "numeric",
    --     "uom": "Ra µm",
    --     "max_value": 1.6,
    --     "gauge": "Surface Profilometer"
    --   },
    --   {
    --     "seq": 3,
    --     "name": "Visual — No Burrs",
    --     "type": "pass_fail",
    --     "instructions": "Inspect all edges under 10x magnification"
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    inspection_number TEXT NOT NULL,
    template_id     UUID NOT NULL REFERENCES inspection_template(id),
    work_order_id   UUID REFERENCES work_order(id),
    job_card_id     UUID REFERENCES job_card(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    lot_number      TEXT,
    inspection_type TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'in_progress', 'passed', 'failed',
                          'conditional_release', 'cancelled')),
    inspector_id    UUID REFERENCES "user"(id),
    inspected_at    TIMESTAMPTZ,
    quantity_inspected NUMERIC(14,4),
    quantity_accepted  NUMERIC(14,4),
    quantity_rejected  NUMERIC(14,4),
    -- Results stored as JSONB array matching template checks
    results         JSONB NOT NULL DEFAULT '[]',
    -- results example:
    -- [
    --   {"check_seq": 1, "measured": [50.012, 50.008, 49.995, 50.018, 50.003],
    --    "result": "pass", "avg": 50.007, "range": 0.023},
    --   {"check_seq": 2, "measured": [1.2], "result": "pass"},
    --   {"check_seq": 3, "result": "pass", "notes": "Clean edges, no burrs detected"}
    -- ]
    -- Compliance-specific inspection data
    compliance_data JSONB NOT NULL DEFAULT '{}',
    -- compliance_data example (AS9100 First Article):
    -- {
    --   "fai_form": "AS9102",
    --   "fai_type": "full",
    --   "balloon_drawing_ref": "DWG-A100-REV-C",
    --   "characteristic_accountability": [
    --     {"char_num": 1, "drawing_req": "50.000 ±0.025", "measured": 50.012, "result": "conforming"}
    --   ]
    -- }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, inspection_number)
);

CREATE INDEX idx_inspection_wo ON inspection_record(work_order_id);
CREATE INDEX idx_inspection_status ON inspection_record(status);

CREATE TABLE non_conformance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    ncr_number      TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    severity        TEXT NOT NULL
        CHECK (severity IN ('critical', 'major', 'minor', 'observation')),
    status          TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open', 'investigating', 'corrective_action',
                          'verification', 'closed')),
    source          TEXT NOT NULL,
    disposition     TEXT,
    -- Linked records
    inspection_id   UUID REFERENCES inspection_record(id),
    work_order_id   UUID REFERENCES work_order(id),
    item_id         UUID REFERENCES item(id),
    lot_number      TEXT,
    quantity_affected NUMERIC(14,4),
    -- Responsibility
    reported_by     UUID REFERENCES "user"(id),
    assigned_to     UUID REFERENCES "user"(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at       TIMESTAMPTZ,
    -- JSONB for root cause analysis and corrective actions
    analysis        JSONB NOT NULL DEFAULT '{}',
    -- analysis example:
    -- {
    --   "root_cause_method": "5_why",
    --   "root_cause": "Tool insert worn beyond replacement threshold",
    --   "contributing_factors": ["No tool life monitoring", "Skip in PM schedule"],
    --   "corrective_actions": [
    --     {"action": "Implement tool life counter on CNC program",
    --      "assigned_to": "uuid", "due_date": "2026-06-01", "status": "open"},
    --     {"action": "Add tool inspection to PM checklist",
    --      "assigned_to": "uuid", "due_date": "2026-05-20", "status": "completed"}
    --   ],
    --   "effectiveness_check": {
    --     "method": "Monitor scrap rate for 30 days",
    --     "target": "< 1% scrap on OD dimension",
    --     "check_date": "2026-07-01"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ncr_number)
);
```

---

## Equipment Telemetry

```sql
-- ============================================================
-- EQUIPMENT TELEMETRY (Time-Series)
-- ============================================================

CREATE TABLE equipment_telemetry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    source          TEXT NOT NULL,          -- 'mtconnect', 'opcua', 'manual'
    -- Core metrics as typed columns (queried by AI quality predictor)
    execution_state TEXT,
    spindle_speed   NUMERIC(10,2),
    feed_rate       NUMERIC(10,4),
    spindle_load    NUMERIC(5,2),
    power_kw        NUMERIC(10,2),
    vibration       NUMERIC(8,4),
    coolant_temp    NUMERIC(6,2),
    -- All other readings in JSONB (varies by machine type)
    readings        JSONB NOT NULL DEFAULT '{}',
    -- readings example (5-axis mill):
    -- {
    --   "axes": {"x": 125.450, "y": 88.200, "z": -35.200, "a": 45.0, "b": 0.0},
    --   "tool_number": 3,
    --   "tool_life_remaining_pct": 62,
    --   "program": "O1234",
    --   "block": "N0450",
    --   "cycle_time_sec": 145,
    --   "parts_count": 48,
    --   "alarms": []
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_telemetry_equip_time ON equipment_telemetry(equipment_id, timestamp DESC);
```

---

## Maintenance

```sql
-- ============================================================
-- MAINTENANCE
-- ============================================================

CREATE TABLE maintenance_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    record_number   TEXT NOT NULL,
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    maintenance_type TEXT NOT NULL
        CHECK (maintenance_type IN ('preventive', 'predictive', 'corrective', 'breakdown')),
    priority        TEXT NOT NULL DEFAULT 'normal'
        CHECK (priority IN ('emergency', 'high', 'normal', 'low')),
    status          TEXT NOT NULL DEFAULT 'requested'
        CHECK (status IN ('requested', 'scheduled', 'in_progress', 'completed', 'cancelled')),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES "user"(id),
    scheduled_start TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    downtime_mins   INTEGER,
    -- JSONB for parts used, checklists, and findings
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "parts_used": [
    --     {"item_number": "BELT-V-A68", "quantity": 1, "cost": 45.00},
    --     {"item_number": "FILTER-OIL-CNC", "quantity": 2, "cost": 12.50}
    --   ],
    --   "checklist": [
    --     {"task": "Check spindle runout", "result": "0.003mm — within spec"},
    --     {"task": "Inspect way covers", "result": "Minor tear on X-axis cover, replaced"},
    --     {"task": "Oil level check", "result": "Topped up 2L"}
    --   ],
    --   "labor_hours": 3.5,
    --   "findings": "X-axis way cover showing wear. Schedule replacement in next PM.",
    --   "ai_prediction_ref": "event-uuid-12345"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, record_number)
);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      TEXT NOT NULL,
    record_id       UUID NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by      UUID,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values      JSONB,
    new_values      JSONB,
    -- JSONB captures changes to both relational and JSONB columns
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"source": "web_ui", "ip": "10.0.1.50", "session": "uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (changed_at);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_date ON audit_log(changed_at);
```

---

## JSONB Query Examples

```sql
-- Find all ITAR-controlled items
SELECT item_number, name, compliance_data
FROM item
WHERE compliance_data @> '{"itar_controlled": true}';

-- Find all items with REACH substance declarations above threshold
SELECT item_number, name,
       jsonb_array_elements(compliance_data->'reach_substances') AS substance
FROM item
WHERE compliance_data ? 'reach_substances';

-- Find all equipment with MTConnect connectivity configured
SELECT code, name, properties->'connectivity'->>'mtconnect_url' AS mtconnect_url
FROM equipment
WHERE properties->'connectivity' ? 'mtconnect_url';

-- Find all work orders for a specific customer PO
SELECT wo_number, status, planned_start, planned_end
FROM work_order
WHERE properties->>'customer_po' = 'PO-ACME-2026-0891';

-- Aggregate scrap reasons from job card JSONB across a date range
SELECT
    detail->>'reason' AS scrap_reason,
    SUM((detail->>'quantity')::NUMERIC) AS total_scrapped,
    COUNT(*) AS occurrences
FROM job_card,
     jsonb_array_elements(properties->'scrap_details') AS detail
WHERE actual_end BETWEEN '2026-05-01' AND '2026-05-31'
GROUP BY detail->>'reason'
ORDER BY total_scrapped DESC;

-- Work orders scheduled by AI with their confidence scores
SELECT wo_number, status,
       properties->'scheduling'->>'scheduled_by' AS scheduler,
       (properties->'scheduling'->>'schedule_confidence')::NUMERIC AS confidence
FROM work_order
WHERE properties->'scheduling'->>'scheduled_by' = 'ai_scheduler'
ORDER BY confidence ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 2 | tenant, user (roles in JSONB) |
| Items & Materials | 1 | item (with properties, compliance_data, custom_fields JSONB) |
| Bill of Materials | 2 | bom, bom_line |
| Work Centers & Equipment | 2 | work_center, equipment (with rich properties JSONB) |
| Routing | 2 | routing, routing_operation |
| Work Orders & Job Cards | 3 | work_order, work_order_material, job_card |
| Inventory | 3 | warehouse, inventory_balance, inventory_transaction |
| Quality Management | 3 | inspection_template (checks in JSONB), inspection_record (results in JSONB), non_conformance (analysis in JSONB) |
| Equipment Telemetry | 1 | equipment_telemetry (partitioned; extra readings in JSONB) |
| Maintenance | 1 | maintenance_record (details in JSONB) |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **21** | Roughly half the table count of the normalized model |

---

## Key Design Decisions

1. **Every table has a `properties` or equivalent JSONB column** — This is the escape hatch for tenant-specific, industry-specific, and machine-specific data. New fields never require a migration; they are added to the JSONB column with application-level JSON Schema validation.

2. **Core manufacturing fields remain relational** — Item number, quantities, dates, statuses, and foreign keys are always typed columns. The MRP engine, scheduler, and inventory balance calculations operate on relational data, not JSONB. JSONB is for context, not computation.

3. **Compliance data is a dedicated JSONB column** — Separating `compliance_data` from `properties` makes it clear which JSONB fields are regulatory (AS9100, IATF 16949, REACH/RoHS) versus operational. This allows compliance-specific validation schemas to be applied independently.

4. **Inspection checks and results live in JSONB arrays** — Rather than the normalized approach of separate `inspection_template_check` and `inspection_result` tables, the hybrid model stores checks as a JSONB array on the template and results as a JSONB array on the record. This eliminates two tables and makes inspection data self-contained.

5. **Roles stored as JSONB array on user** — Instead of separate role and user_role junction tables, roles are a simple JSONB array. This works well for manufacturing where role assignments change infrequently and the set of roles is small (< 20 per tenant).

6. **Equipment IoT configuration lives in JSONB** — MTConnect URLs, OPC-UA endpoints, controller models, and calibration data are stored in the equipment `properties` JSONB. This accommodates the wide variation in machine types without a separate table per machine category.

7. **NCR analysis workflow is JSONB** — Root cause analysis, corrective actions, and effectiveness checks are stored as structured JSONB within the non_conformance record. This keeps the full NCR lifecycle in a single row while allowing flexible analysis methods (5 Why, Ishikawa, etc.).

8. **GIN indexes on all JSONB columns** — Every JSONB column has a GIN index enabling fast containment queries (`@>`) and existence checks (`?`). This makes queries like "find all ITAR items" or "find all equipment with MTConnect" performant.

9. **Table count is roughly half the normalized model** — 21 tables vs. 43 in the normalized model. The reduction comes from absorbing extension tables (roles, inspection checks, inspection results, corrective actions, ECOs) into JSONB. Fewer tables means simpler deployments and fewer JOINs.

10. **Natural language interface transcripts stored in job card JSONB** — When operators report production events via the voice/chat interface, the raw transcript is stored in the job card `properties` alongside the parsed structured data, enabling AI model training and audit trail for voice-reported events.
