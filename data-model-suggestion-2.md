# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Manufacturing ERP · Created: 2026-05-12

## Philosophy

This model treats every state change in the manufacturing process as an immutable event stored in an append-only event store. The event store is the single source of truth; all queryable state (work order status, inventory balances, quality records) is derived by replaying events into materialised read models (projections). This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from optimised projections.

Event sourcing is particularly well-suited to manufacturing because manufacturing IS a sequence of events: material received, work order released, operation started, operation completed, scrap reported, inspection passed, lot shipped. Every manufacturing audit trail is fundamentally an event log. ISO 9001 requires that organisations maintain records of quality events and be able to reconstruct the history of any lot or product — event sourcing delivers this by construction rather than by bolt-on audit logging.

The approach is inspired by financial ledger systems (where double-entry bookkeeping is inherently event-sourced), by the OCSF (Open Cybersecurity Schema Framework) structured event format, and by modern event-driven manufacturing execution systems. The trade-off is increased complexity in the write path (events must be designed as a complete domain language) and eventual consistency in read models (projections lag behind the event store by milliseconds to seconds).

**Best for:** Manufacturers requiring complete audit trails, temporal queries ("what was the BOM on January 15th?"), AI-driven analytics on production patterns, and regulatory environments where immutable records are mandatory (AS9100 aerospace, FDA 21 CFR Part 11).

**Trade-offs:**
- (+) Complete, immutable audit trail by construction — no separate audit log needed
- (+) Temporal queries are trivial: replay events to any point in time
- (+) AI scheduling and quality prediction agents can consume the event stream in real time
- (+) New read models can be built retroactively from the full event history
- (+) Natural fit for OPC-UA PubSub and MTConnect data streams (both are event-based)
- (-) Increased write-path complexity: every business action must be expressed as events
- (-) Read models are eventually consistent (milliseconds to seconds of lag)
- (-) Event schema evolution requires careful versioning (upcasting old events)
- (-) Debugging requires event replay tooling; cannot simply "look at the database"
- (-) Higher storage requirements: events are never deleted, only archived

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISA-95 / IEC 62264 | ISA-95 operational domains map to event stream categories: production events, quality events, maintenance events, inventory events |
| ISO 9001:2015 | Immutable event store satisfies audit trail requirements by construction; every quality event is recorded with full context |
| AS9100 Rev D | Configuration management and first-article inspection histories are fully reconstructable from the event stream |
| GS1 EPCIS | GS1 Electronic Product Code Information Services defines traceability events (ObjectEvent, AggregationEvent, TransactionEvent) — the event store adopts this vocabulary for lot/serial lifecycle events |
| B2MML v0700 | Production schedule and work order events map to B2MML ProductionPerformance and ProductionSchedule schemas |
| MTConnect | Machine telemetry events from MTConnect agents flow directly into the event store as equipment_telemetry events |
| OPC-UA PubSub | OPC-UA PubSub messages are native events that can be stored directly without transformation |
| OCSF | Event envelope structure (id, timestamp, category, type, severity, actor, target) draws from OCSF structured event format |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE — Single Source of Truth
-- ============================================================

CREATE TABLE event_store (
    -- Event identity
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number BIGSERIAL,             -- global ordering for replay
    
    -- Event metadata
    stream_id       UUID NOT NULL,         -- aggregate root ID (e.g., work order ID)
    stream_type     TEXT NOT NULL,          -- aggregate type: 'work_order', 'item', 'lot', etc.
    stream_version  INTEGER NOT NULL,      -- per-stream version for optimistic concurrency
    
    -- Event classification
    event_type      TEXT NOT NULL,          -- e.g. 'WorkOrderReleased', 'OperationCompleted'
    event_category  TEXT NOT NULL,          -- ISA-95 domain: 'production', 'quality', 'maintenance', 'inventory'
    
    -- Multi-tenancy
    tenant_id       UUID NOT NULL,
    
    -- Event payload
    data            JSONB NOT NULL,         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "actor_id": "uuid",
    --   "actor_type": "user|ai_agent|system|operator",
    --   "source": "web_ui|mobile|voice|api|mtconnect|opcua",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",
    --   "ip_address": "192.168.1.100",
    --   "ai_confidence": 0.95
    -- }
    
    -- Timestamps
    occurred_at     TIMESTAMPTZ NOT NULL,  -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when it was stored
    
    -- Concurrency control
    UNIQUE (stream_id, stream_version)
) PARTITION BY RANGE (recorded_at);

-- Partitioned by month for scalable storage
CREATE TABLE event_store_2026_01 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions created automatically

-- Primary query patterns
CREATE INDEX idx_event_stream ON event_store(stream_id, stream_version);
CREATE INDEX idx_event_type ON event_store(event_type, recorded_at);
CREATE INDEX idx_event_category ON event_store(event_category, recorded_at);
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, recorded_at);
CREATE INDEX idx_event_sequence ON event_store(sequence_number);
CREATE INDEX idx_event_correlation ON event_store((metadata->>'correlation_id'));
```

---

## Event Type Catalog

Manufacturing domain events organised by ISA-95 operational domain:

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (Reference Data)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    category        TEXT NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    payload_schema  JSONB NOT NULL,        -- JSON Schema for the data field
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Production Operations Events
INSERT INTO event_type_registry (event_type, category, description, schema_version, payload_schema) VALUES
('ItemCreated',              'master_data', 'New item/part defined', 1, '{}'),
('ItemRevisionPublished',    'master_data', 'New item revision activated', 1, '{}'),
('BomCreated',               'master_data', 'Bill of Materials created', 1, '{}'),
('BomLineAdded',             'master_data', 'Component added to BOM', 1, '{}'),
('BomLineRemoved',           'master_data', 'Component removed from BOM', 1, '{}'),
('BomActivated',             'master_data', 'BOM status set to active', 1, '{}'),
('BomSuperseded',            'master_data', 'BOM replaced by new version', 1, '{}'),
('RoutingCreated',           'master_data', 'Routing created for item', 1, '{}'),
('RoutingOperationAdded',    'master_data', 'Operation added to routing', 1, '{}'),
('EcoSubmitted',             'master_data', 'Engineering change order submitted', 1, '{}'),
('EcoApproved',              'master_data', 'Engineering change order approved', 1, '{}'),
('EcoImplemented',           'master_data', 'Engineering change order implemented', 1, '{}'),

('PlannedOrderCreated',      'production', 'MRP generated a planned order', 1, '{}'),
('PlannedOrderFirmed',       'production', 'Planned order firmed by planner', 1, '{}'),
('WorkOrderCreated',         'production', 'Work order created from plan or manual', 1, '{}'),
('WorkOrderReleased',        'production', 'Work order released to shop floor', 1, '{}'),
('WorkOrderStarted',         'production', 'First operation started', 1, '{}'),
('WorkOrderHeld',            'production', 'Work order placed on hold', 1, '{}'),
('WorkOrderResumed',         'production', 'Work order resumed from hold', 1, '{}'),
('WorkOrderCompleted',       'production', 'All operations completed', 1, '{}'),
('WorkOrderClosed',          'production', 'Work order financially closed', 1, '{}'),
('WorkOrderCancelled',       'production', 'Work order cancelled', 1, '{}'),
('WorkOrderRescheduled',     'production', 'Work order dates changed by AI scheduler', 1, '{}'),

('JobCardCreated',           'production', 'Job card created for operation', 1, '{}'),
('OperationStarted',         'production', 'Operator began work on operation', 1, '{}'),
('OperationPaused',          'production', 'Operation paused', 1, '{}'),
('OperationResumed',         'production', 'Operation resumed', 1, '{}'),
('OperationCompleted',       'production', 'Operation completed with quantities', 1, '{}'),
('ScrapReported',            'production', 'Scrap quantity recorded with reason', 1, '{}'),
('MaterialIssued',           'production', 'Material issued to work order', 1, '{}'),
('MaterialBackflushed',      'production', 'Material auto-consumed on completion', 1, '{}'),

-- Quality Operations Events
('InspectionRequested',      'quality', 'Inspection triggered for lot/operation', 1, '{}'),
('InspectionStarted',        'quality', 'Inspector began inspection', 1, '{}'),
('InspectionResultRecorded', 'quality', 'Individual check result recorded', 1, '{}'),
('InspectionPassed',         'quality', 'Inspection passed — lot released', 1, '{}'),
('InspectionFailed',         'quality', 'Inspection failed — NCR initiated', 1, '{}'),
('NcrCreated',               'quality', 'Non-conformance report opened', 1, '{}'),
('NcrDispositioned',         'quality', 'NCR disposition decided', 1, '{}'),
('NcrClosed',                'quality', 'NCR closed after verification', 1, '{}'),
('CarCreated',               'quality', 'Corrective action request opened', 1, '{}'),
('CarImplemented',           'quality', 'Corrective action implemented', 1, '{}'),
('CarVerified',              'quality', 'Corrective action effectiveness verified', 1, '{}'),
('QualityPredictionAlert',   'quality', 'AI predicted quality failure from telemetry', 1, '{}'),

-- Inventory Operations Events
('MaterialReceived',         'inventory', 'Material received from supplier', 1, '{}'),
('LotCreated',               'inventory', 'New lot created', 1, '{}'),
('LotQuarantined',           'inventory', 'Lot placed in quarantine', 1, '{}'),
('LotReleased',              'inventory', 'Lot released from quarantine', 1, '{}'),
('StockTransferred',         'inventory', 'Stock moved between locations', 1, '{}'),
('StockAdjusted',            'inventory', 'Inventory adjustment recorded', 1, '{}'),
('CycleCountRecorded',       'inventory', 'Cycle count result recorded', 1, '{}'),
('MaterialScrapped',         'inventory', 'Material scrapped and removed from inventory', 1, '{}'),

-- Maintenance Operations Events
('MaintenanceScheduled',     'maintenance', 'Preventive maintenance scheduled', 1, '{}'),
('MaintenanceStarted',       'maintenance', 'Maintenance work begun', 1, '{}'),
('MaintenanceCompleted',     'maintenance', 'Maintenance work completed', 1, '{}'),
('EquipmentBreakdown',       'maintenance', 'Unplanned equipment failure', 1, '{}'),
('EquipmentRestored',        'maintenance', 'Equipment returned to service', 1, '{}'),
('PredictiveMaintenanceAlert','maintenance', 'AI predicted maintenance need from telemetry', 1, '{}'),

-- Equipment Telemetry Events
('MachineStateChanged',      'telemetry', 'Machine execution state changed', 1, '{}'),
('MachineCycleCompleted',    'telemetry', 'Machine completed one production cycle', 1, '{}'),
('SensorReadingRecorded',    'telemetry', 'OPC-UA/MTConnect sensor data captured', 1, '{}'),
('AlarmTriggered',           'telemetry', 'Machine alarm condition triggered', 1, '{}'),
('AlarmCleared',             'telemetry', 'Machine alarm condition cleared', 1, '{}');
```

### Event Payload Examples

```sql
-- Example: WorkOrderReleased event
-- {
--   "wo_number": "WO-2026-00142",
--   "item_id": "uuid",
--   "item_number": "PART-A100",
--   "bom_id": "uuid",
--   "routing_id": "uuid",
--   "quantity_planned": 500,
--   "uom": "EA",
--   "planned_start": "2026-05-15T06:00:00Z",
--   "planned_end": "2026-05-17T18:00:00Z",
--   "lot_number": "L2026-0512-001",
--   "priority": 800
-- }

-- Example: OperationCompleted event
-- {
--   "job_card_id": "uuid",
--   "work_order_id": "uuid",
--   "operation_sequence": 20,
--   "operation_name": "CNC Turning",
--   "work_center_code": "WC-LATHE-01",
--   "equipment_code": "CNC-L-003",
--   "operator_id": "uuid",
--   "operator_name": "John Smith",
--   "quantity_completed": 48,
--   "quantity_scrapped": 2,
--   "scrap_reason": "surface_finish_oor",
--   "setup_time_mins": 25,
--   "run_time_mins": 185,
--   "actual_start": "2026-05-15T06:15:00Z",
--   "actual_end": "2026-05-15T09:45:00Z"
-- }

-- Example: QualityPredictionAlert event (from AI agent)
-- {
--   "equipment_id": "uuid",
--   "equipment_code": "CNC-L-003",
--   "work_order_id": "uuid",
--   "prediction_type": "surface_finish_degradation",
--   "confidence": 0.92,
--   "contributing_factors": [
--     {"factor": "spindle_vibration", "value": 0.45, "threshold": 0.35, "trend": "increasing"},
--     {"factor": "tool_wear_cycles", "value": 4200, "threshold": 5000, "trend": "approaching"}
--   ],
--   "recommended_action": "inspect_tool_insert",
--   "affected_items_in_progress": ["WO-2026-00142"]
-- }

-- Example: SensorReadingRecorded event (from MTConnect/OPC-UA)
-- {
--   "equipment_id": "uuid",
--   "source_protocol": "mtconnect",
--   "device_uuid": "Mazak-QTN-250M-001",
--   "readings": {
--     "execution": "ACTIVE",
--     "mode": "AUTOMATIC",
--     "spindle_speed_rpm": 2800,
--     "feed_rate_mmpm": 150.5,
--     "spindle_load_pct": 42.3,
--     "x_position": 125.450,
--     "y_position": 0.000,
--     "z_position": -35.200,
--     "coolant_temp_c": 22.5,
--     "power_kw": 4.8
--   }
-- }
```

---

## Read Models (Projections — Query Side)

The following materialised views are rebuilt from the event stream. Each projection is optimised for a specific query pattern.

```sql
-- ============================================================
-- PROJECTION: Current Item State
-- ============================================================

CREATE TABLE proj_item (
    item_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    item_number     TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    item_type       TEXT NOT NULL,
    uom             TEXT NOT NULL,
    gtin            TEXT,
    standard_cost   NUMERIC(14,4),
    current_revision TEXT,
    current_bom_id  UUID,
    lot_tracked     BOOLEAN NOT NULL DEFAULT false,
    serial_tracked  BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_id   UUID NOT NULL,         -- tracks projection currency
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_item_tenant ON proj_item(tenant_id);
CREATE INDEX idx_proj_item_number ON proj_item(tenant_id, item_number);

-- ============================================================
-- PROJECTION: Current Work Order State
-- ============================================================

CREATE TABLE proj_work_order (
    work_order_id   UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    wo_number       TEXT NOT NULL,
    item_id         UUID NOT NULL,
    item_number     TEXT NOT NULL,
    item_name       TEXT NOT NULL,
    bom_id          UUID,
    routing_id      UUID,
    quantity_planned NUMERIC(14,4) NOT NULL,
    quantity_completed NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_scrapped NUMERIC(14,4) NOT NULL DEFAULT 0,
    uom             TEXT NOT NULL,
    status          TEXT NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 500,
    lot_number      TEXT,
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    estimated_cost  NUMERIC(14,4),
    actual_cost     NUMERIC(14,4),
    -- Denormalized for dashboard queries
    operations_total   INTEGER DEFAULT 0,
    operations_done    INTEGER DEFAULT 0,
    current_operation  TEXT,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_wo_tenant ON proj_work_order(tenant_id);
CREATE INDEX idx_proj_wo_status ON proj_work_order(tenant_id, status);
CREATE INDEX idx_proj_wo_dates ON proj_work_order(planned_start, planned_end);

-- ============================================================
-- PROJECTION: Current Inventory Balances
-- ============================================================

CREATE TABLE proj_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    item_id         UUID NOT NULL,
    item_number     TEXT NOT NULL,
    warehouse_code  TEXT NOT NULL,
    location_code   TEXT,
    lot_number      TEXT,
    quantity_on_hand NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_allocated NUMERIC(14,4) NOT NULL DEFAULT 0,
    quantity_available NUMERIC(14,4) NOT NULL DEFAULT 0,
    uom             TEXT NOT NULL,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_id, warehouse_code, COALESCE(location_code, ''), COALESCE(lot_number, ''))
);

CREATE INDEX idx_proj_inv_item ON proj_inventory(tenant_id, item_id);

-- ============================================================
-- PROJECTION: Current Equipment Status
-- ============================================================

CREATE TABLE proj_equipment_status (
    equipment_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    equipment_code  TEXT NOT NULL,
    equipment_name  TEXT NOT NULL,
    work_center_code TEXT,
    status          TEXT NOT NULL,          -- 'operational', 'running', 'idle', 'maintenance', 'breakdown'
    current_work_order TEXT,
    current_operation TEXT,
    current_operator TEXT,
    execution_state TEXT,                   -- MTConnect: 'ACTIVE', 'READY', 'STOPPED'
    last_telemetry  JSONB,                 -- latest sensor readings
    utilization_pct NUMERIC(5,2),          -- rolling 24h utilization
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Quality Dashboard
-- ============================================================

CREATE TABLE proj_quality_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    item_id         UUID NOT NULL,
    item_number     TEXT NOT NULL,
    period          DATE NOT NULL,         -- daily aggregation
    inspections_total INTEGER DEFAULT 0,
    inspections_passed INTEGER DEFAULT 0,
    inspections_failed INTEGER DEFAULT 0,
    first_pass_yield NUMERIC(5,4),         -- passed / total
    ncrs_opened     INTEGER DEFAULT 0,
    ncrs_closed     INTEGER DEFAULT 0,
    scrap_quantity  NUMERIC(14,4) DEFAULT 0,
    ai_predictions  INTEGER DEFAULT 0,
    ai_predictions_confirmed INTEGER DEFAULT 0,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, item_id, period)
);

-- ============================================================
-- PROJECTION: Lot Traceability (GS1 EPCIS-aligned)
-- ============================================================

CREATE TABLE proj_lot_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    lot_number      TEXT NOT NULL,
    item_id         UUID NOT NULL,
    item_number     TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    event_data      JSONB NOT NULL,
    -- Forward/backward trace links
    source_lots     TEXT[],                -- lot numbers consumed to make this lot
    destination_lots TEXT[],               -- lot numbers produced from this lot
    work_order_number TEXT,
    location        TEXT,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_lot_number ON proj_lot_history(tenant_id, lot_number);
CREATE INDEX idx_proj_lot_item ON proj_lot_history(tenant_id, item_id);
```

---

## Projection Processor

```sql
-- ============================================================
-- PROJECTION CHECKPOINT (Tracks replay position per projection)
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    last_event_id   UUID,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'rebuilding', 'paused', 'error')),
    error_message   TEXT
);

INSERT INTO projection_checkpoint (projection_name) VALUES
('proj_item'),
('proj_work_order'),
('proj_inventory'),
('proj_equipment_status'),
('proj_quality_summary'),
('proj_lot_history');
```

---

## Snapshot Store (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOT STORE (Avoids full replay for long-lived aggregates)
-- ============================================================

CREATE TABLE snapshot_store (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,        -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, version)
);

-- Snapshots are taken every N events (e.g., every 100 events for an aggregate)
-- to avoid replaying thousands of events when loading an aggregate
CREATE INDEX idx_snapshot_latest ON snapshot_store(stream_id, version DESC);
```

---

## Temporal Query Examples

```sql
-- What was the state of work order WO-2026-00142 at 2pm on May 15th?
SELECT *
FROM event_store
WHERE stream_id = :work_order_id
  AND stream_type = 'work_order'
  AND occurred_at <= '2026-05-15T14:00:00Z'
ORDER BY stream_version ASC;
-- Application replays these events to reconstruct state at that moment

-- What happened to lot L2026-0512-001 throughout its lifecycle?
SELECT event_type, occurred_at, data, metadata
FROM event_store
WHERE data->>'lot_number' = 'L2026-0512-001'
ORDER BY occurred_at ASC;

-- Which AI quality predictions were confirmed by subsequent inspection failures?
SELECT
    p.event_id AS prediction_event,
    p.occurred_at AS predicted_at,
    p.data->>'prediction_type' AS prediction_type,
    (p.data->>'confidence')::NUMERIC AS confidence,
    f.event_id AS failure_event,
    f.occurred_at AS failed_at
FROM event_store p
JOIN event_store f ON f.data->>'equipment_id' = p.data->>'equipment_id'
WHERE p.event_type = 'QualityPredictionAlert'
  AND f.event_type = 'InspectionFailed'
  AND f.occurred_at > p.occurred_at
  AND f.occurred_at < p.occurred_at + INTERVAL '4 hours';

-- Production throughput trend by work center (last 30 days)
SELECT
    data->>'work_center_code' AS work_center,
    DATE(occurred_at) AS production_date,
    COUNT(*) AS operations_completed,
    SUM((data->>'quantity_completed')::NUMERIC) AS total_output,
    SUM((data->>'quantity_scrapped')::NUMERIC) AS total_scrap,
    AVG((data->>'run_time_mins')::NUMERIC) AS avg_run_time
FROM event_store
WHERE event_type = 'OperationCompleted'
  AND occurred_at > now() - INTERVAL '30 days'
GROUP BY data->>'work_center_code', DATE(occurred_at)
ORDER BY work_center, production_date;
```

---

## Reference Data Tables (Non-Event-Sourced)

Some rarely-changing reference data is stored relationally for simplicity:

```sql
-- ============================================================
-- REFERENCE DATA (Relational — not event-sourced)
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE uom (
    code            TEXT PRIMARY KEY,      -- 'EA', 'KG', 'M', 'L', 'HR'
    name            TEXT NOT NULL,
    uom_type        TEXT NOT NULL
        CHECK (uom_type IN ('quantity', 'weight', 'length', 'volume', 'time', 'area')),
    base_uom        TEXT REFERENCES uom(code),
    conversion_factor NUMERIC(14,8)        -- multiplier to convert to base_uom
);

CREATE TABLE scrap_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    category        TEXT,
    UNIQUE (tenant_id, code)
);

CREATE TABLE hold_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    UNIQUE (tenant_id, code)
);
```

---

## AI Agent Event Stream Integration

```sql
-- ============================================================
-- AI AGENT SUBSCRIPTION (Event consumption for AI agents)
-- ============================================================

CREATE TABLE event_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_name TEXT NOT NULL UNIQUE,   -- 'ai_scheduler', 'quality_predictor', 'bom_assistant'
    filter_categories TEXT[] NOT NULL,       -- ['production', 'telemetry']
    filter_event_types TEXT[],              -- optional: specific event types
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'paused', 'error')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI scheduler subscribes to production + telemetry events
INSERT INTO event_subscription (subscriber_name, filter_categories) VALUES
('ai_dynamic_scheduler', ARRAY['production', 'telemetry', 'maintenance']),
('ai_quality_predictor', ARRAY['quality', 'telemetry']),
('ai_bom_assistant', ARRAY['master_data']);

-- Query: Get next batch of events for AI scheduler
-- SELECT * FROM event_store
-- WHERE event_category = ANY(:filter_categories)
--   AND sequence_number > :last_sequence
-- ORDER BY sequence_number
-- LIMIT 1000;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | event_store (partitioned by month) |
| Event Type Registry | 1 | event_type_registry (reference data) |
| Projections | 6 | proj_item, proj_work_order, proj_inventory, proj_equipment_status, proj_quality_summary, proj_lot_history |
| Projection Infrastructure | 1 | projection_checkpoint |
| Snapshot Store | 1 | snapshot_store |
| Reference Data | 5 | tenant, user, uom, scrap_reason, hold_reason |
| AI Agent Integration | 1 | event_subscription |
| **Total** | **16** | Much lower table count than normalized; complexity is in event design |

---

## Key Design Decisions

1. **Single event_store table as the canonical source of truth** — All manufacturing state changes, from BOM edits to machine telemetry, are stored as immutable events in one partitioned table. This eliminates the need for a separate audit log and guarantees a complete history.

2. **Events carry rich context, not just deltas** — Each event includes enough data to be self-describing. An `OperationCompleted` event includes the operator name, work center code, quantities, and times — not just foreign keys. This makes event streams consumable by AI agents without expensive joins.

3. **Projections are disposable read models** — Every projection table can be dropped and rebuilt from the event store. This means new query patterns can be added retroactively, and projection bugs can be fixed by replaying.

4. **Optimistic concurrency via stream_version** — The UNIQUE constraint on (stream_id, stream_version) prevents concurrent writes to the same aggregate, ensuring event ordering consistency without pessimistic locks.

5. **ISA-95 operational domains as event categories** — Events are categorised into production, quality, maintenance, inventory, telemetry, and master_data — directly mapping to ISA-95's four operational domains plus supporting categories. This enables domain-scoped event subscriptions.

6. **AI agents consume events as first-class subscribers** — The event_subscription table lets AI scheduling, quality prediction, and BOM assistant agents subscribe to filtered event streams, processing them asynchronously. The AI scheduler can emit its own events (WorkOrderRescheduled, QualityPredictionAlert) back into the store.

7. **Temporal queries are answered by event replay** — "What was the BOM on date X?" or "What was inventory at time Y?" are answered by replaying events up to that timestamp. No bi-temporal columns or slowly changing dimensions needed.

8. **Equipment telemetry events flow into the same store** — MTConnect and OPC-UA data points are stored as events alongside business events. This enables the AI quality predictor to correlate machine parameters with quality outcomes in a single query.

9. **Snapshot store prevents unbounded replay** — Long-lived aggregates (items, equipment) get periodic snapshots so loading current state doesn't require replaying thousands of events.

10. **Event schema evolution uses versioning** — The event_type_registry tracks schema versions. Old events are "upcasted" (transformed to the latest schema version) during replay, enabling backward-compatible event evolution.
