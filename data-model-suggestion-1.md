# Data Model Suggestion 1: Entity-Centric Normalized Relational (ISA-95 Aligned)

> Project: Manufacturing ERP · Created: 2026-05-12

## Philosophy

This model follows the classic normalized relational approach, mapping every ISA-95 object model entity to a dedicated PostgreSQL table with strict foreign key constraints and referential integrity. The design directly mirrors the four ISA-95 resource categories — Personnel, Equipment, Material, and Physical Assets — and implements the four ISA-95 operational domains (Production Operations, Quality Operations, Maintenance Operations, and Inventory Operations) as separate but cross-referenced table groups.

The normalized structure ensures that each fact is stored exactly once, eliminating update anomalies and making the schema a faithful representation of real-world manufacturing relationships. This is the approach used by SAP S/4HANA's underlying HANA tables (STKO/STPO for BOM, PLKO/PLPO for routings, AFKO/AFPO for production orders) and is the most battle-tested architecture for manufacturing ERP systems. Every major commercial manufacturing ERP — Epicor Kinetic, Infor SyteLine, SYSPRO — uses a normalized relational core.

The trade-off is schema rigidity: adding a new field requires a migration, multi-level BOM queries require recursive CTEs, and jurisdiction-specific variations need additional columns or extension tables. However, for a system where data integrity, regulatory compliance, and predictable query performance are paramount, full normalization is the gold standard.

**Best for:** Manufacturers requiring ISO 9001/AS9100/IATF 16949 compliance, where data integrity and audit trail completeness are non-negotiable requirements.

**Trade-offs:**
- (+) Maximum data integrity through foreign key constraints and normalized storage
- (+) Well-understood by PostgreSQL DBAs and ERP developers worldwide
- (+) Predictable query performance with proper indexing
- (+) Direct alignment with ISA-95 object models eases MES integration via B2MML
- (-) Schema changes require database migrations for every new field
- (-) Multi-level BOM explosion requires recursive CTEs (can be slow for 10+ levels)
- (-) Jurisdiction-specific or customer-specific fields need extension tables
- (-) High table count increases JOIN complexity for cross-domain queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | Four resource categories (Personnel, Equipment, Material, Physical Asset) map directly to table groups; four operational domains (Production, Quality, Maintenance, Inventory) structure the schema sections |
| ISO 9001:2015 | Quality tables implement NCR tracking, CAR/CAPA workflow, inspection records with full lot traceability |
| AS9100 Rev D | First-article inspection (FAI) table and configuration management fields support aerospace supplier requirements |
| IATF 16949 | FMEA, PPAP, and SPC tables support automotive quality methodology requirements |
| GS1 / GTIN / UDI | Material definition includes GTIN fields; lot/serial tables carry GS1-compliant identifiers |
| ISO 55000 | Physical asset and maintenance tables align with asset management lifecycle concepts |
| ISO 15531 (MANDATE) | BOM and routing data models reference MANDATE manufacturing management data structures |
| B2MML v0700 | Table structures map cleanly to B2MML XML schema elements for MES data exchange |
| MTConnect / OPC-UA | Equipment telemetry tables store machine data using MTConnect device/component hierarchy |

---

## Core Infrastructure Tables

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
    settings        JSONB NOT NULL DEFAULT '{}',
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
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,          -- e.g. 'plant_manager', 'operator', 'quality_inspector'
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES "user"(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES "user"(id),
    PRIMARY KEY (user_id, role_id)
);

-- Row-Level Security policy for multi-tenancy
ALTER TABLE "user" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "user"
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

---

## ISA-95 Resource Models

### Personnel

```sql
-- ============================================================
-- PERSONNEL (ISA-95 Personnel Model)
-- ============================================================

CREATE TABLE personnel_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,          -- e.g. 'CNC Operator', 'Quality Inspector', 'Welder'
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"certifications_required": ["AWS D1.1"], "license_type": "Welding"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    employee_number TEXT NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    hire_date       DATE,
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'on_leave', 'terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, employee_number)
);

CREATE TABLE person_qualification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id),
    personnel_class_id UUID NOT NULL REFERENCES personnel_class(id),
    qualification_name TEXT NOT NULL,       -- e.g. 'AWS D1.1 Certified Welder'
    qualification_level TEXT,               -- e.g. 'Level 3'
    issued_date     DATE,
    expiry_date     DATE,
    issuing_body    TEXT,
    certificate_ref TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_qualification_expiry ON person_qualification(expiry_date);
```

### Equipment

```sql
-- ============================================================
-- EQUIPMENT (ISA-95 Equipment Model)
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,          -- e.g. 'CNC Lathe', '3-Axis Mill', 'Hydraulic Press'
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- e.g. {"max_rpm": 12000, "axis_count": 3, "table_size_mm": "500x300"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE work_center (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    capacity_units  TEXT NOT NULL DEFAULT 'hours',
    available_hours_per_day NUMERIC(5,2) NOT NULL DEFAULT 8.0,
    cost_rate       NUMERIC(12,4),         -- cost per capacity unit
    overhead_rate   NUMERIC(12,4),
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'maintenance')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_class_id UUID REFERENCES equipment_class(id),
    work_center_id  UUID REFERENCES work_center(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    serial_number   TEXT,
    manufacturer    TEXT,
    model           TEXT,
    status          TEXT NOT NULL DEFAULT 'operational'
        CHECK (status IN ('operational', 'maintenance', 'breakdown', 'decommissioned')),
    commissioned_date DATE,
    -- MTConnect / OPC-UA connectivity
    mtconnect_agent_url TEXT,              -- e.g. 'http://192.168.1.100:5000'
    opcua_endpoint_url  TEXT,              -- e.g. 'opc.tcp://192.168.1.100:4840'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_equipment_work_center ON equipment(work_center_id);
CREATE INDEX idx_equipment_status ON equipment(status);
```

### Physical Assets

```sql
-- ============================================================
-- PHYSICAL ASSETS (ISA-95 Physical Asset Model)
-- ============================================================

CREATE TABLE physical_asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID REFERENCES equipment(id),  -- links asset to equipment role
    asset_tag       TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    asset_type      TEXT NOT NULL,          -- 'machine', 'tooling', 'fixture', 'gauge', 'building'
    location        TEXT,
    purchase_date   DATE,
    purchase_cost   NUMERIC(14,2),
    replacement_cost NUMERIC(14,2),
    expected_life_years NUMERIC(5,1),
    status          TEXT NOT NULL DEFAULT 'in_service'
        CHECK (status IN ('in_service', 'spare', 'repair', 'disposed')),
    -- ISO 55000 asset management fields
    criticality     TEXT DEFAULT 'medium'
        CHECK (criticality IN ('critical', 'high', 'medium', 'low')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, asset_tag)
);
```

### Material

```sql
-- ============================================================
-- MATERIAL (ISA-95 Material Model)
-- ============================================================

CREATE TABLE material_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,          -- e.g. 'Raw Steel', 'Fastener', 'Electronic Component'
    description     TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    material_class_id UUID REFERENCES material_class(id),
    item_number     TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    item_type       TEXT NOT NULL
        CHECK (item_type IN ('raw_material', 'component', 'sub_assembly', 'finished_good', 'consumable', 'tooling')),
    uom             TEXT NOT NULL,          -- unit of measure: 'EA', 'KG', 'M', 'L'
    -- GS1 / GTIN identification
    gtin            TEXT,                   -- Global Trade Item Number (14-digit)
    udi             TEXT,                   -- Unique Device Identifier (if applicable)
    -- Costing
    standard_cost   NUMERIC(14,4),
    last_cost       NUMERIC(14,4),
    -- Planning
    lead_time_days  INTEGER DEFAULT 0,
    safety_stock    NUMERIC(14,4) DEFAULT 0,
    reorder_point   NUMERIC(14,4) DEFAULT 0,
    lot_size_min    NUMERIC(14,4) DEFAULT 1,
    lot_size_max    NUMERIC(14,4),
    lot_size_multiple NUMERIC(14,4) DEFAULT 1,
    -- Tracking
    lot_tracked     BOOLEAN NOT NULL DEFAULT false,
    serial_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_number)
);

CREATE INDEX idx_item_type ON item(item_type);
CREATE INDEX idx_item_gtin ON item(gtin) WHERE gtin IS NOT NULL;

CREATE TABLE item_revision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES item(id),
    revision        TEXT NOT NULL,          -- e.g. 'A', 'B', 'C' or '1.0', '1.1'
    description     TEXT,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'superseded', 'obsolete')),
    approved_by     UUID REFERENCES "user"(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (item_id, revision)
);
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
    item_id         UUID NOT NULL REFERENCES item(id),      -- parent item
    revision_id     UUID REFERENCES item_revision(id),
    bom_type        TEXT NOT NULL DEFAULT 'manufacturing'
        CHECK (bom_type IN ('manufacturing', 'engineering', 'planning', 'costing')),
    name            TEXT NOT NULL,
    description     TEXT,
    quantity         NUMERIC(14,4) NOT NULL DEFAULT 1,      -- output quantity
    uom             TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'active', 'superseded', 'obsolete')),
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    created_by      UUID REFERENCES "user"(id),
    approved_by     UUID REFERENCES "user"(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bom_item ON bom(item_id);
CREATE INDEX idx_bom_status ON bom(status);

CREATE TABLE bom_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bom_id          UUID NOT NULL REFERENCES bom(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    component_item_id UUID NOT NULL REFERENCES item(id),    -- child item
    quantity        NUMERIC(14,6) NOT NULL,
    uom             TEXT NOT NULL,
    scrap_factor    NUMERIC(5,4) DEFAULT 0,                 -- e.g. 0.02 = 2% scrap
    is_phantom      BOOLEAN NOT NULL DEFAULT false,         -- phantom/sub-assembly pass-through
    reference_designator TEXT,                               -- e.g. 'R1, R2, R3' for PCB components
    operation_seq   INTEGER,                                 -- links to routing operation
    effective_from  DATE,
    effective_to    DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (bom_id, line_number)
);

CREATE INDEX idx_bom_line_component ON bom_line(component_item_id);

-- Engineering Change Order for BOM revision management
CREATE TABLE engineering_change_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    eco_number      TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    priority        TEXT NOT NULL DEFAULT 'normal'
        CHECK (priority IN ('critical', 'high', 'normal', 'low')),
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'submitted', 'approved', 'rejected', 'implemented', 'closed')),
    requested_by    UUID REFERENCES "user"(id),
    approved_by     UUID REFERENCES "user"(id),
    requested_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    target_date     DATE,
    implemented_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, eco_number)
);

CREATE TABLE eco_affected_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    eco_id          UUID NOT NULL REFERENCES engineering_change_order(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    old_revision_id UUID REFERENCES item_revision(id),
    new_revision_id UUID REFERENCES item_revision(id),
    change_type     TEXT NOT NULL
        CHECK (change_type IN ('add', 'modify', 'remove', 'replace')),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Multi-Level BOM Explosion Query

```sql
-- Recursive CTE for multi-level BOM explosion
WITH RECURSIVE bom_explosion AS (
    -- Anchor: top-level item
    SELECT
        bl.component_item_id,
        i.item_number,
        i.name,
        bl.quantity,
        bl.uom,
        bl.scrap_factor,
        bl.is_phantom,
        1 AS level,
        ARRAY[b.item_id] AS path
    FROM bom b
    JOIN bom_line bl ON bl.bom_id = b.id
    JOIN item i ON i.id = bl.component_item_id
    WHERE b.item_id = :parent_item_id
      AND b.status = 'active'
      AND (b.effective_to IS NULL OR b.effective_to > CURRENT_DATE)

    UNION ALL

    -- Recursive: child BOMs
    SELECT
        bl.component_item_id,
        i.item_number,
        i.name,
        bl.quantity * be.quantity AS quantity,  -- accumulated quantity
        bl.uom,
        bl.scrap_factor,
        bl.is_phantom,
        be.level + 1,
        be.path || b.item_id
    FROM bom_explosion be
    JOIN bom b ON b.item_id = be.component_item_id AND b.status = 'active'
    JOIN bom_line bl ON bl.bom_id = b.id
    JOIN item i ON i.id = bl.component_item_id
    WHERE NOT (b.item_id = ANY(be.path))  -- prevent cycles
      AND be.level < 20                    -- safety limit
)
SELECT * FROM bom_explosion ORDER BY level, item_number;
```

---

## Routing & Operations

```sql
-- ============================================================
-- ROUTING (Production Process Definition)
-- ============================================================

CREATE TABLE routing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    revision        TEXT NOT NULL DEFAULT '1',
    name            TEXT NOT NULL,
    description     TEXT,
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
    sequence        INTEGER NOT NULL,       -- operation sequence number (10, 20, 30...)
    name            TEXT NOT NULL,          -- e.g. 'Rough Turning', 'Finish Milling', 'Deburr'
    description     TEXT,
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    equipment_class_id UUID REFERENCES equipment_class(id),
    personnel_class_id UUID REFERENCES personnel_class(id),
    -- Time standards
    setup_time_mins NUMERIC(10,2) NOT NULL DEFAULT 0,
    run_time_mins   NUMERIC(10,2) NOT NULL DEFAULT 0,   -- per unit
    teardown_time_mins NUMERIC(10,2) NOT NULL DEFAULT 0,
    overlap_percent NUMERIC(5,2) DEFAULT 0,              -- % overlap with next operation
    -- Costing
    labor_rate      NUMERIC(12,4),
    overhead_rate   NUMERIC(12,4),
    -- Quality
    inspection_required BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (routing_id, sequence)
);

CREATE INDEX idx_routing_operation_wc ON routing_operation(work_center_id);
```

---

## Production Operations (MRP & Work Orders)

```sql
-- ============================================================
-- MRP & PRODUCTION PLANNING
-- ============================================================

CREATE TABLE production_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    plan_number     TEXT NOT NULL,
    name            TEXT NOT NULL,
    plan_type       TEXT NOT NULL
        CHECK (plan_type IN ('mrp', 'mps', 'manual')),
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'running', 'completed', 'cancelled')),
    planned_start   DATE NOT NULL,
    planned_end     DATE NOT NULL,
    run_at          TIMESTAMPTZ,
    run_by          UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, plan_number)
);

CREATE TABLE planned_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    production_plan_id UUID NOT NULL REFERENCES production_plan(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    order_type      TEXT NOT NULL
        CHECK (order_type IN ('production', 'purchase', 'transfer')),
    quantity        NUMERIC(14,4) NOT NULL,
    uom             TEXT NOT NULL,
    planned_start   DATE NOT NULL,
    planned_end     DATE NOT NULL,
    priority        INTEGER DEFAULT 500,
    source          TEXT NOT NULL DEFAULT 'mrp'
        CHECK (source IN ('mrp', 'manual', 'ai_scheduler')),
    firmed          BOOLEAN NOT NULL DEFAULT false,
    pegging_demand_id UUID,                -- links to sales order or parent work order
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WORK ORDERS (Production Execution)
-- ============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    wo_number       TEXT NOT NULL,
    item_id         UUID NOT NULL REFERENCES item(id),
    bom_id          UUID NOT NULL REFERENCES bom(id),
    routing_id      UUID REFERENCES routing(id),
    planned_order_id UUID REFERENCES planned_order(id),
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
    -- Lot tracking
    lot_number      TEXT,
    -- Costing
    estimated_cost  NUMERIC(14,4),
    actual_cost     NUMERIC(14,4),
    -- Subcontracting
    is_subcontracted BOOLEAN NOT NULL DEFAULT false,
    supplier_id     UUID,                   -- references supplier table (not shown)
    created_by      UUID REFERENCES "user"(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, wo_number)
);

CREATE INDEX idx_wo_item ON work_order(item_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_planned_start ON work_order(planned_start);

CREATE TABLE work_order_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    bom_line_id     UUID REFERENCES bom_line(id),
    quantity_required NUMERIC(14,6) NOT NULL,
    quantity_issued NUMERIC(14,6) NOT NULL DEFAULT 0,
    uom             TEXT NOT NULL,
    warehouse_id    UUID,                   -- references warehouse/location
    lot_number      TEXT,
    serial_number   TEXT,
    backflush       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- JOB CARDS (Operator-Level Execution)
-- ============================================================

CREATE TABLE job_card (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    operation_id    UUID NOT NULL REFERENCES routing_operation(id),
    jc_number       TEXT NOT NULL,
    work_center_id  UUID NOT NULL REFERENCES work_center(id),
    equipment_id    UUID REFERENCES equipment(id),
    operator_id     UUID REFERENCES person(id),
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
    -- Time capture
    setup_time_mins NUMERIC(10,2) DEFAULT 0,
    run_time_mins   NUMERIC(10,2) DEFAULT 0,
    idle_time_mins  NUMERIC(10,2) DEFAULT 0,
    -- Operator notes (natural language interface captures these)
    operator_notes  TEXT,
    scrap_reason    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, jc_number)
);

CREATE INDEX idx_jc_wo ON job_card(work_order_id);
CREATE INDEX idx_jc_operator ON job_card(operator_id);
CREATE INDEX idx_jc_status ON job_card(status);
```

---

## Inventory Management

```sql
-- ============================================================
-- INVENTORY (ISA-95 Inventory Operations)
-- ============================================================

CREATE TABLE warehouse (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    address         TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE storage_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    code            TEXT NOT NULL,          -- e.g. 'A-01-03' (aisle-rack-bin)
    name            TEXT,
    location_type   TEXT DEFAULT 'bin'
        CHECK (location_type IN ('bin', 'rack', 'floor', 'staging', 'quarantine', 'shipping')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (warehouse_id, code)
);

CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    lot_number      TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'available'
        CHECK (status IN ('available', 'quarantine', 'hold', 'expired', 'consumed')),
    manufactured_date DATE,
    expiry_date     DATE,
    supplier_lot    TEXT,                   -- supplier's lot number
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_id, lot_number)
);

CREATE TABLE serial_number (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES item(id),
    lot_id          UUID REFERENCES lot(id),
    serial          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'available'
        CHECK (status IN ('available', 'allocated', 'shipped', 'returned', 'scrapped')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (item_id, serial)
);

CREATE TABLE inventory_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    warehouse_id    UUID NOT NULL REFERENCES warehouse(id),
    location_id     UUID REFERENCES storage_location(id),
    lot_id          UUID REFERENCES lot(id),
    quantity_on_hand NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(14,4) GENERATED ALWAYS AS
        (quantity_on_hand - quantity_allocated) STORED,
    uom             TEXT NOT NULL,
    last_count_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_id, warehouse_id, COALESCE(location_id, '00000000-0000-0000-0000-000000000000'::UUID), COALESCE(lot_id, '00000000-0000-0000-0000-000000000000'::UUID))
);

CREATE INDEX idx_inv_balance_item ON inventory_balance(item_id);
CREATE INDEX idx_inv_balance_warehouse ON inventory_balance(warehouse_id);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    transaction_type TEXT NOT NULL
        CHECK (transaction_type IN ('receipt', 'issue', 'transfer', 'adjustment',
                                     'scrap', 'return', 'backflush', 'count')),
    quantity        NUMERIC(14,4) NOT NULL,  -- positive = in, negative = out
    uom             TEXT NOT NULL,
    from_warehouse_id UUID REFERENCES warehouse(id),
    from_location_id  UUID REFERENCES storage_location(id),
    to_warehouse_id   UUID REFERENCES warehouse(id),
    to_location_id    UUID REFERENCES storage_location(id),
    lot_id          UUID REFERENCES lot(id),
    serial_id       UUID REFERENCES serial_number(id),
    work_order_id   UUID REFERENCES work_order(id),
    reference_type  TEXT,                   -- 'purchase_order', 'work_order', 'adjustment'
    reference_id    UUID,
    cost            NUMERIC(14,4),
    transacted_by   UUID REFERENCES "user"(id),
    transacted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_item ON inventory_transaction(item_id);
CREATE INDEX idx_inv_txn_type ON inventory_transaction(transaction_type);
CREATE INDEX idx_inv_txn_date ON inventory_transaction(transacted_at);
CREATE INDEX idx_inv_txn_wo ON inventory_transaction(work_order_id);
```

---

## Quality Operations

```sql
-- ============================================================
-- QUALITY MANAGEMENT (ISO 9001 / AS9100 / IATF 16949)
-- ============================================================

CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    inspection_type TEXT NOT NULL
        CHECK (inspection_type IN ('incoming', 'in_process', 'final', 'first_article')),
    item_id         UUID REFERENCES item(id),              -- specific item, or NULL for generic
    operation_seq   INTEGER,                                -- specific operation, or NULL
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_template_check (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES inspection_template(id) ON DELETE CASCADE,
    sequence        INTEGER NOT NULL,
    check_name      TEXT NOT NULL,          -- e.g. 'Diameter', 'Surface Roughness', 'Torque'
    check_type      TEXT NOT NULL
        CHECK (check_type IN ('numeric', 'pass_fail', 'text', 'visual')),
    uom             TEXT,
    nominal_value   NUMERIC(14,6),
    tolerance_upper NUMERIC(14,6),
    tolerance_lower NUMERIC(14,6),
    gauge_id        UUID REFERENCES equipment(id),         -- measurement instrument
    instructions    TEXT,
    UNIQUE (template_id, sequence)
);

CREATE TABLE inspection_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    template_id     UUID NOT NULL REFERENCES inspection_template(id),
    inspection_number TEXT NOT NULL,
    work_order_id   UUID REFERENCES work_order(id),
    job_card_id     UUID REFERENCES job_card(id),
    lot_id          UUID REFERENCES lot(id),
    item_id         UUID NOT NULL REFERENCES item(id),
    inspection_type TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'in_progress', 'passed', 'failed',
                          'conditional_release', 'cancelled')),
    inspector_id    UUID REFERENCES person(id),
    inspected_at    TIMESTAMPTZ,
    quantity_inspected NUMERIC(14,4),
    quantity_accepted  NUMERIC(14,4),
    quantity_rejected  NUMERIC(14,4),
    disposition     TEXT,                   -- what to do with rejected items
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, inspection_number)
);

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection_record(id),
    check_id        UUID NOT NULL REFERENCES inspection_template_check(id),
    measured_value  TEXT,                   -- stored as text for flexibility; parsed by check_type
    numeric_value   NUMERIC(14,6),         -- populated for numeric checks
    result          TEXT NOT NULL
        CHECK (result IN ('pass', 'fail', 'na')),
    notes           TEXT,
    measured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Non-Conformance Report (NCR) — ISO 9001 requirement
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
    source          TEXT NOT NULL
        CHECK (source IN ('inspection', 'customer_complaint', 'internal_audit',
                          'supplier', 'process', 'ai_predicted')),
    -- Linked records
    inspection_id   UUID REFERENCES inspection_record(id),
    work_order_id   UUID REFERENCES work_order(id),
    item_id         UUID REFERENCES item(id),
    lot_id          UUID REFERENCES lot(id),
    -- Disposition
    disposition     TEXT
        CHECK (disposition IN ('use_as_is', 'rework', 'repair', 'scrap',
                                'return_to_supplier', 'concession')),
    quantity_affected NUMERIC(14,4),
    -- Responsibility
    reported_by     UUID REFERENCES "user"(id),
    assigned_to     UUID REFERENCES "user"(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ncr_number)
);

-- Corrective Action Request (CAR/CAPA) — ISO 9001 requirement
CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    car_number      TEXT NOT NULL,
    ncr_id          UUID NOT NULL REFERENCES non_conformance(id),
    action_type     TEXT NOT NULL
        CHECK (action_type IN ('corrective', 'preventive', 'containment')),
    root_cause      TEXT,
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES "user"(id),
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open', 'in_progress', 'implemented', 'verified', 'closed')),
    verified_by     UUID REFERENCES "user"(id),
    verified_at     TIMESTAMPTZ,
    effectiveness   TEXT
        CHECK (effectiveness IN ('effective', 'partially_effective', 'ineffective')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, car_number)
);
```

---

## Maintenance Operations

```sql
-- ============================================================
-- MAINTENANCE / CMMS (ISO 55000)
-- ============================================================

CREATE TABLE maintenance_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    name            TEXT NOT NULL,
    maintenance_type TEXT NOT NULL
        CHECK (maintenance_type IN ('preventive', 'predictive', 'condition_based')),
    frequency_value INTEGER NOT NULL,
    frequency_unit  TEXT NOT NULL
        CHECK (frequency_unit IN ('hours', 'days', 'weeks', 'months', 'cycles')),
    instructions    TEXT,
    estimated_duration_mins INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_performed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE maintenance_work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    mwo_number      TEXT NOT NULL,
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    maintenance_plan_id UUID REFERENCES maintenance_plan(id),
    maintenance_type TEXT NOT NULL
        CHECK (maintenance_type IN ('preventive', 'predictive', 'corrective', 'breakdown')),
    priority        TEXT NOT NULL DEFAULT 'normal'
        CHECK (priority IN ('emergency', 'high', 'normal', 'low')),
    status          TEXT NOT NULL DEFAULT 'requested'
        CHECK (status IN ('requested', 'approved', 'scheduled', 'in_progress',
                          'completed', 'cancelled')),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES person(id),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    downtime_mins   INTEGER,
    parts_cost      NUMERIC(14,4),
    labor_cost      NUMERIC(14,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, mwo_number)
);
```

---

## Equipment Telemetry (IoT Integration)

```sql
-- ============================================================
-- EQUIPMENT TELEMETRY (OPC-UA / MTConnect)
-- ============================================================

CREATE TABLE equipment_telemetry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    source_protocol TEXT NOT NULL
        CHECK (source_protocol IN ('mtconnect', 'opcua', 'manual', 'api')),
    -- MTConnect data items
    execution_state TEXT,                   -- 'ACTIVE', 'READY', 'STOPPED', 'INTERRUPTED'
    mode            TEXT,                   -- 'AUTOMATIC', 'MANUAL', 'SEMI_AUTOMATIC'
    program_name    TEXT,
    -- Machine parameters (numeric samples)
    spindle_speed   NUMERIC(10,2),         -- RPM
    feed_rate       NUMERIC(10,4),         -- mm/min
    spindle_load    NUMERIC(5,2),          -- % load
    axis_position_x NUMERIC(12,6),
    axis_position_y NUMERIC(12,6),
    axis_position_z NUMERIC(12,6),
    coolant_temp    NUMERIC(6,2),
    vibration       NUMERIC(8,4),
    power_consumption NUMERIC(10,2),       -- kW
    -- Cycle tracking
    cycle_count     BIGINT,
    parts_count     BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

-- Partition by month for time-series performance
CREATE TABLE equipment_telemetry_2026_01 PARTITION OF equipment_telemetry
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions created by maintenance job

CREATE INDEX idx_telemetry_equip_time ON equipment_telemetry(equipment_id, timestamp DESC);
CREATE INDEX idx_telemetry_state ON equipment_telemetry(execution_state);
```

---

## Costing

```sql
-- ============================================================
-- JOB COSTING
-- ============================================================

CREATE TABLE cost_element (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    cost_type       TEXT NOT NULL
        CHECK (cost_type IN ('material', 'labor', 'overhead', 'subcontract', 'other')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE work_order_cost (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    cost_element_id UUID NOT NULL REFERENCES cost_element(id),
    cost_type       TEXT NOT NULL,
    planned_cost    NUMERIC(14,4),
    actual_cost     NUMERIC(14,4) NOT NULL DEFAULT 0,
    variance        NUMERIC(14,4) GENERATED ALWAYS AS (actual_cost - COALESCE(planned_cost, 0)) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_cost_wo ON work_order_cost(work_order_id);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG (Cross-cutting)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      TEXT NOT NULL,
    record_id       UUID NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by      UUID,                   -- user who made the change
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT
) PARTITION BY RANGE (changed_at);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_log(changed_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Identity | 4 | tenant, user, role, user_role |
| Personnel (ISA-95) | 3 | personnel_class, person, person_qualification |
| Equipment (ISA-95) | 3 | equipment_class, work_center, equipment |
| Physical Assets (ISA-95) | 1 | physical_asset |
| Material (ISA-95) | 3 | material_class, item, item_revision |
| Bill of Materials | 4 | bom, bom_line, engineering_change_order, eco_affected_item |
| Routing & Operations | 2 | routing, routing_operation |
| Production Planning | 2 | production_plan, planned_order |
| Work Orders & Job Cards | 3 | work_order, work_order_material, job_card |
| Inventory | 6 | warehouse, storage_location, lot, serial_number, inventory_balance, inventory_transaction |
| Quality Management | 6 | inspection_template, inspection_template_check, inspection_record, inspection_result, non_conformance, corrective_action |
| Maintenance / CMMS | 2 | maintenance_plan, maintenance_work_order |
| Equipment Telemetry | 1 | equipment_telemetry (partitioned) |
| Costing | 2 | cost_element, work_order_cost |
| Audit Trail | 1 | audit_log (partitioned) |
| **Total** | **43** | |

---

## Key Design Decisions

1. **Every ISA-95 resource category has its own table group** — Personnel, Equipment, Material, and Physical Assets are modeled as separate concerns with explicit relationships between them, directly mirroring the ISA-95 object model and enabling clean B2MML data exchange.

2. **BOM and Routing are separate entities linked through work orders** — Following the SAP production version pattern, BOMs define material structure and routings define process structure; they are independent so multiple routings can apply to the same BOM.

3. **Multi-level BOM explosion uses recursive CTEs** — Rather than denormalizing the BOM hierarchy, the normalized parent-child structure is traversed at query time using PostgreSQL recursive CTEs, preserving data integrity at the cost of query complexity.

4. **Inventory uses double-entry transaction logging** — Every stock movement creates an `inventory_transaction` record; balances in `inventory_balance` are the running total. This provides full traceability for ISO 9001 lot tracking.

5. **Equipment telemetry is time-partitioned** — The `equipment_telemetry` table uses PostgreSQL range partitioning by month, allowing old telemetry data to be archived or dropped without affecting operational tables.

6. **Multi-tenancy through tenant_id + RLS** — Every operational table carries a `tenant_id` foreign key with PostgreSQL Row-Level Security policies, enabling shared-schema multi-tenancy with strong isolation.

7. **Quality management implements the full ISO 9001 NCR/CAR workflow** — Inspection templates gate stock movement; non-conformances link to inspections, work orders, and lots; corrective actions track root cause and effectiveness verification.

8. **JSONB is used sparingly for properties** — Equipment classes, personnel classes, and material classes use JSONB for variable properties (e.g., machine specifications), but all operational fields are typed columns. This balances flexibility with data integrity.

9. **Audit log captures before/after state** — A trigger-driven audit log stores old and new JSONB values for every INSERT, UPDATE, and DELETE across regulated tables, satisfying ISO 9001 audit trail requirements.

10. **MTConnect and OPC-UA connectivity is modeled per equipment** — Each equipment record stores its MTConnect agent URL and OPC-UA endpoint, enabling the IoT integration layer to discover and connect to machines dynamically.
