# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Property Management System · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable domain event stored in an append-only event store. The current state of any entity (lease, payment, maintenance request) is derived by replaying its event stream. Separate materialized read models (projections) are maintained for fast queries: portfolio dashboards, tenant portals, accounting reports. The write side validates and persists events; the read side projects them into query-optimized views.

Event sourcing is used in financial systems (banking ledgers, payment processors), compliance-heavy domains (healthcare EHRs, regulatory platforms), and anywhere full audit trails are a hard requirement. For property management, this architecture naturally answers questions like "what was the rent on unit 4B on March 15th?", "who changed the lease terms and when?", and "reconstruct the complete payment history for this tenant across all their leases." It also provides the historical data foundation that AI models need for delinquency prediction and rent pricing.

The CQRS (Command Query Responsibility Segregation) pattern separates the write model (event store) from read models (projections). Commands produce events; queries read projections. This enables independent scaling of read-heavy operations (tenant portals, dashboards) and write-heavy operations (rent processing, maintenance intake).

**Best for:** Organizations requiring complete audit trails, regulatory compliance (HUD Section 8 inspections, ASC 842 lease accounting), and AI/ML features that need rich historical data for training and inference.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is recorded with who, what, when, and why
- (+) Temporal queries are trivial — reconstruct state at any point in time by replaying events up to that timestamp
- (+) Rich historical data for AI model training (payment patterns, maintenance trends, occupancy cycles)
- (+) Event replay enables bug fixes — correct a projection bug and rebuild from the event store
- (+) Natural fit for real-time notifications — events can trigger push notifications, emails, and SMS
- (-) Higher implementation complexity — developers must think in events rather than CRUD
- (-) Event schema evolution requires careful versioning (upcasting old events to new schemas)
- (-) Eventual consistency between event store and projections can confuse users ("I just paid but the balance hasn't updated")
- (-) Projections must be rebuilt when their schema changes, which takes time for large datasets
- (-) Debugging requires understanding both the event stream and the projection logic
- (-) More infrastructure: event store + projection database + message bus

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RESO Data Dictionary 2.x | Property and listing event payloads use RESO field names for interoperability |
| OSCRE Industry Data Model | Event types map to OSCRE use cases (Lease Execution, Work Order Creation, Space Change) |
| ASC 842 / IFRS 16 | Lease classification events capture the five criteria at point of classification; historical replay shows classification changes over time |
| NACHA / ACH | Payment events include SEC codes and processor references; payment lifecycle (initiated → processing → settled → failed) maps naturally to event sequences |
| FCRA | Screening events record consent, report receipt, and adverse action as discrete, timestamped, immutable events — ideal for FCRA compliance audits |
| CloudEvents 1.0 | Event envelope format follows CloudEvents specification (id, source, type, time, data) for interoperability |
| ISO 8601 | All timestamps in events use ISO 8601 with timezone (TIMESTAMPTZ) |

---

## Event Store

```sql
-- The single source of truth: an append-only event store
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- CloudEvents-aligned envelope
    event_type VARCHAR(200) NOT NULL, -- e.g. lease.created, payment.received, maintenance.assigned
    aggregate_type VARCHAR(100) NOT NULL, -- e.g. lease, payment, maintenance_request, property
    aggregate_id UUID NOT NULL, -- the entity this event belongs to
    organization_id UUID NOT NULL, -- tenant isolation
    sequence_number BIGINT NOT NULL, -- ordering within an aggregate
    -- Event metadata
    caused_by UUID, -- user or system that triggered the event
    caused_by_type VARCHAR(20) NOT NULL DEFAULT 'user', -- user, system, ai, webhook
    correlation_id UUID, -- links related events across aggregates
    causation_id UUID, -- the event that caused this event
    -- Event payload
    data JSONB NOT NULL, -- the event-specific payload
    metadata JSONB, -- additional context (IP address, user agent, AI model version)
    -- Timestamps
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE (aggregate_type, aggregate_id, sequence_number)
);

-- Primary query pattern: replay events for an aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);
-- Organization-scoped queries
CREATE INDEX idx_events_org_type ON events(organization_id, event_type, occurred_at);
-- Correlation tracking
CREATE INDEX idx_events_correlation ON events(correlation_id);
-- Time-range queries for projections and analytics
CREATE INDEX idx_events_occurred ON events(occurred_at);
-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Event type registry (documents all known event types and their schemas)
CREATE TABLE event_types (
    event_type VARCHAR(200) PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    schema_version INTEGER NOT NULL DEFAULT 1,
    json_schema JSONB NOT NULL, -- JSON Schema defining the event's data payload
    deprecated BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshots (optimization: avoid replaying entire event streams)
CREATE TABLE snapshots (
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL, -- snapshot is valid up to this event
    state JSONB NOT NULL, -- serialized aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

### Example Event Types and Payloads

```sql
-- Example: lease.created event
-- data: {
--   "unit_id": "uuid",
--   "lease_type": "fixed_term",
--   "start_date": "2026-06-01",
--   "end_date": "2027-05-31",
--   "monthly_rent": 1850.00,
--   "security_deposit": 1850.00,
--   "tenant_ids": ["uuid1", "uuid2"],
--   "rent_due_day": 1,
--   "late_fee_amount": 50.00,
--   "late_fee_grace_days": 5
-- }

-- Example: payment.received event
-- data: {
--   "lease_id": "uuid",
--   "tenant_id": "uuid",
--   "amount": 1850.00,
--   "currency": "USD",
--   "payment_method": "ach",
--   "ach_sec_code": "WEB",
--   "processor_reference_id": "pi_abc123",
--   "payment_date": "2026-06-01",
--   "covers_period_start": "2026-06-01",
--   "covers_period_end": "2026-06-30"
-- }

-- Example: maintenance.triage_completed event (AI-generated)
-- data: {
--   "request_id": "uuid",
--   "original_priority": "normal",
--   "ai_priority": "urgent",
--   "ai_category": "plumbing",
--   "ai_confidence": 0.92,
--   "recommended_vendor_id": "uuid",
--   "reasoning": "Water leak reported in bathroom ceiling — risk of structural damage if not addressed within 24 hours",
--   "model_version": "maint-triage-v2.3"
-- }

-- Example: lease.rent_adjusted event
-- data: {
--   "previous_rent": 1850.00,
--   "new_rent": 1925.00,
--   "effective_date": "2027-06-01",
--   "reason": "annual_increase",
--   "ai_recommended_rent": 1940.00,
--   "market_average": 1960.00
-- }
```

---

## Command Handlers

```sql
-- Commands are validated and produce events. This table tracks command execution
-- for idempotency and debugging (optional — can be application-level only).
CREATE TABLE commands (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type VARCHAR(200) NOT NULL, -- e.g. CreateLease, RecordPayment, AssignVendor
    organization_id UUID NOT NULL,
    issued_by UUID,
    issued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, accepted, rejected
    rejection_reason TEXT,
    resulting_event_ids UUID[], -- events produced by this command
    idempotency_key VARCHAR(255), -- prevents duplicate processing
    UNIQUE (idempotency_key)
);
CREATE INDEX idx_commands_org ON commands(organization_id, issued_at);
```

---

## Read Model Projections

These tables are derived from events and can be rebuilt at any time. They represent the "current state" views.

```sql
-- ============================================================
-- PROJECTION: Properties (current state)
-- Rebuilt from: property.created, property.updated, property.deactivated
-- ============================================================
CREATE TABLE proj_properties (
    id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    property_type VARCHAR(50) NOT NULL,
    address_line1 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    year_built INTEGER,
    total_units INTEGER,
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_properties_org ON proj_properties(organization_id);

-- ============================================================
-- PROJECTION: Units (current state)
-- Rebuilt from: unit.created, unit.updated, unit.status_changed
-- ============================================================
CREATE TABLE proj_units (
    id UUID PRIMARY KEY,
    property_id UUID NOT NULL,
    unit_number VARCHAR(50),
    bedrooms INTEGER,
    bathrooms_full INTEGER,
    living_area_sqft NUMERIC(10, 2),
    market_rent NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'vacant',
    current_lease_id UUID,
    current_tenant_names TEXT, -- denormalized for display
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_units_property ON proj_units(property_id);
CREATE INDEX idx_proj_units_status ON proj_units(status);

-- ============================================================
-- PROJECTION: Leases (current state with denormalized tenant info)
-- Rebuilt from: lease.* events
-- ============================================================
CREATE TABLE proj_leases (
    id UUID PRIMARY KEY,
    unit_id UUID NOT NULL,
    property_id UUID NOT NULL, -- denormalized for dashboard queries
    property_name VARCHAR(255),
    unit_number VARCHAR(50),
    lease_status VARCHAR(30) NOT NULL,
    lease_type VARCHAR(30) NOT NULL,
    asc842_classification VARCHAR(20),
    start_date DATE NOT NULL,
    end_date DATE,
    monthly_rent NUMERIC(10, 2) NOT NULL,
    security_deposit NUMERIC(10, 2),
    primary_tenant_name VARCHAR(200), -- denormalized
    tenant_count INTEGER,
    days_until_expiry INTEGER, -- computed and refreshed
    balance_due NUMERIC(10, 2) DEFAULT 0, -- running balance from payment events
    last_payment_date DATE,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_leases_unit ON proj_leases(unit_id);
CREATE INDEX idx_proj_leases_status ON proj_leases(lease_status);
CREATE INDEX idx_proj_leases_expiry ON proj_leases(end_date);

-- ============================================================
-- PROJECTION: Tenant ledger (running balance per tenant per lease)
-- Rebuilt from: payment.*, charge.*, late_fee.* events
-- ============================================================
CREATE TABLE proj_tenant_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    lease_id UUID NOT NULL,
    entry_date DATE NOT NULL,
    entry_type VARCHAR(30) NOT NULL, -- charge, payment, credit, late_fee, refund
    description VARCHAR(255),
    amount NUMERIC(10, 2) NOT NULL, -- positive = charge/debit, negative = payment/credit
    running_balance NUMERIC(10, 2) NOT NULL,
    source_event_id UUID NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_ledger_tenant_lease ON proj_tenant_ledger(tenant_id, lease_id, entry_date);

-- ============================================================
-- PROJECTION: Maintenance dashboard
-- Rebuilt from: maintenance.* and work_order.* events
-- ============================================================
CREATE TABLE proj_maintenance_requests (
    id UUID PRIMARY KEY,
    unit_id UUID NOT NULL,
    property_id UUID NOT NULL,
    property_name VARCHAR(255),
    unit_number VARCHAR(50),
    tenant_name VARCHAR(200),
    category VARCHAR(50),
    priority VARCHAR(20),
    ai_priority VARCHAR(20),
    title VARCHAR(255),
    status VARCHAR(30),
    vendor_name VARCHAR(255),
    vendor_id UUID,
    work_order_status VARCHAR(30),
    total_cost NUMERIC(10, 2),
    days_open INTEGER, -- computed
    created_at TIMESTAMPTZ,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_maint_property ON proj_maintenance_requests(property_id);
CREATE INDEX idx_proj_maint_status ON proj_maintenance_requests(status);
CREATE INDEX idx_proj_maint_priority ON proj_maintenance_requests(priority);

-- ============================================================
-- PROJECTION: Accounting journal (double-entry from payment events)
-- Rebuilt from: payment.*, distribution.*, expense.* events
-- ============================================================
CREATE TABLE proj_journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    entry_date DATE NOT NULL,
    description TEXT,
    property_id UUID,
    property_name VARCHAR(255),
    source_event_id UUID NOT NULL,
    source_event_type VARCHAR(200),
    is_posted BOOLEAN NOT NULL DEFAULT true,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_journal_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES proj_journal_entries(id),
    account_code VARCHAR(20) NOT NULL,
    account_name VARCHAR(255),
    debit NUMERIC(14, 2) NOT NULL DEFAULT 0,
    credit NUMERIC(14, 2) NOT NULL DEFAULT 0,
    memo TEXT
);
CREATE INDEX idx_proj_jlines_entry ON proj_journal_lines(journal_entry_id);
CREATE INDEX idx_proj_jlines_account ON proj_journal_lines(account_code);

-- ============================================================
-- PROJECTION: AI prediction history
-- Rebuilt from: ai.delinquency_predicted, ai.rent_recommended events
-- ============================================================
CREATE TABLE proj_delinquency_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    lease_id UUID NOT NULL,
    prediction_date DATE NOT NULL,
    risk_score NUMERIC(5, 4),
    risk_level VARCHAR(20),
    recommended_action VARCHAR(100),
    model_version VARCHAR(50),
    source_event_id UUID NOT NULL
);
CREATE INDEX idx_proj_delinquency_tenant ON proj_delinquency_scores(tenant_id, prediction_date);

-- ============================================================
-- PROJECTION: Portfolio summary (materialized for dashboard)
-- Rebuilt from: all property, unit, lease, payment events
-- ============================================================
CREATE TABLE proj_portfolio_summary (
    organization_id UUID PRIMARY KEY,
    total_properties INTEGER NOT NULL DEFAULT 0,
    total_units INTEGER NOT NULL DEFAULT 0,
    occupied_units INTEGER NOT NULL DEFAULT 0,
    vacancy_rate NUMERIC(5, 4) NOT NULL DEFAULT 0,
    total_monthly_rent NUMERIC(14, 2) NOT NULL DEFAULT 0,
    outstanding_balance NUMERIC(14, 2) NOT NULL DEFAULT 0,
    open_maintenance_requests INTEGER NOT NULL DEFAULT 0,
    leases_expiring_30d INTEGER NOT NULL DEFAULT 0,
    leases_expiring_60d INTEGER NOT NULL DEFAULT 0,
    high_risk_tenants INTEGER NOT NULL DEFAULT 0,
    last_event_at TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projection Tracking

```sql
-- Tracks which projections have processed which events
-- Enables incremental projection updates and full rebuilds
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID,
    last_event_occurred_at TIMESTAMPTZ,
    last_projected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- active, rebuilding, paused, error
    error_message TEXT
);
```

---

## Reference Data (Shared Across Write and Read Sides)

```sql
-- These tables are NOT event-sourced; they are stable reference data.

CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    account_type VARCHAR(30) NOT NULL,
    account_subtype VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT true,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    UNIQUE (organization_id, account_code)
);

CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    subdivision_code VARCHAR(10),
    municipality VARCHAR(255),
    rent_control_active BOOLEAN NOT NULL DEFAULT false,
    max_security_deposit_months NUMERIC(4, 2),
    notice_to_vacate_days INTEGER,
    late_fee_max_percent NUMERIC(5, 2)
);
```

---

## Example Queries

### Temporal query: What was the rent on a unit on a specific date?

```sql
-- Reconstruct lease state at a point in time by replaying events
SELECT data->>'monthly_rent' AS rent_at_date
FROM events
WHERE aggregate_type = 'lease'
  AND aggregate_id = '{{lease_id}}'
  AND event_type IN ('lease.created', 'lease.rent_adjusted')
  AND occurred_at <= '2026-09-15T00:00:00Z'
ORDER BY sequence_number DESC
LIMIT 1;
```

### Audit query: Complete change history for a lease

```sql
SELECT event_type, 
       occurred_at,
       caused_by,
       data
FROM events
WHERE aggregate_type = 'lease'
  AND aggregate_id = '{{lease_id}}'
ORDER BY sequence_number ASC;
```

### Analytics: Payment patterns for delinquency model training

```sql
SELECT 
    data->>'tenant_id' AS tenant_id,
    data->>'lease_id' AS lease_id,
    data->>'amount' AS amount,
    data->>'payment_date' AS payment_date,
    occurred_at,
    EXTRACT(DAY FROM occurred_at - (data->>'payment_due_date')::date) AS days_late
FROM events
WHERE event_type = 'payment.received'
  AND organization_id = '{{org_id}}'
  AND occurred_at >= now() - INTERVAL '2 years'
ORDER BY occurred_at;
```

### Rebuilding a projection from scratch

```sql
-- Step 1: Mark projection as rebuilding
UPDATE projection_checkpoints 
SET status = 'rebuilding' 
WHERE projection_name = 'proj_leases';

-- Step 2: Truncate the projection table
TRUNCATE TABLE proj_leases;

-- Step 3: Application code replays all lease.* events in sequence_number order
-- and inserts/updates proj_leases rows

-- Step 4: Update checkpoint
UPDATE projection_checkpoints 
SET status = 'active',
    last_event_id = '{{latest_event_id}}',
    last_projected_at = now()
WHERE projection_name = 'proj_leases';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events, event_types, snapshots |
| Commands | 1 | commands (optional, for idempotency) |
| Reference Data | 4 | organizations, users, accounts, jurisdictions |
| Projections — Properties | 2 | proj_properties, proj_units |
| Projections — Leases | 2 | proj_leases, proj_tenant_ledger |
| Projections — Maintenance | 1 | proj_maintenance_requests |
| Projections — Accounting | 2 | proj_journal_entries, proj_journal_lines |
| Projections — AI | 1 | proj_delinquency_scores |
| Projections — Dashboard | 1 | proj_portfolio_summary |
| Projection Tracking | 1 | projection_checkpoints |
| **Total** | **~18 tables** | Plus the events table which stores everything; projections are rebuildable |

---

## Key Design Decisions

1. **Single events table as source of truth** — all domain state lives in one append-only table. This dramatically simplifies backup, replication, and compliance (all changes are in one place). The table is partitioned by month for performance.

2. **CloudEvents-compatible envelope** — event metadata (id, type, source, time) follows the CloudEvents 1.0 specification, enabling interoperability with external event-driven systems and message brokers.

3. **Aggregate-scoped sequence numbers** — events within an aggregate are strictly ordered by sequence_number, enabling optimistic concurrency control. Two concurrent writers to the same aggregate will fail on the UNIQUE constraint, preventing lost updates.

4. **Correlation and causation IDs** — every event can be traced to its cause (which event or command triggered it) and correlated with related events across aggregates. This is essential for debugging multi-step workflows (e.g., lease creation triggers unit status change and accounting entries).

5. **Projections are disposable** — every proj_* table can be truncated and rebuilt from the event store. This means read model schema changes are safe — add a column, rebuild, done. No data migration needed.

6. **Denormalized projections for performance** — proj_leases includes property_name and unit_number to avoid JOINs in dashboard queries. This is intentional denormalization on the read side, trading storage for query speed.

7. **Reference data tables are not event-sourced** — organizations, users, and chart of accounts are traditional CRUD tables. Not everything benefits from event sourcing; these entities change rarely and don't need temporal queries.

8. **Snapshots for replay optimization** — for aggregates with many events (a lease active for 5 years might have hundreds of events), snapshots cache the current state at a point in time so replays start from the snapshot rather than from event zero.

9. **AI events are first-class citizens** — AI predictions (delinquency scores, rent recommendations, maintenance triage decisions) are recorded as events alongside human actions. This creates an auditable record of AI decisions and enables model performance evaluation over time.

10. **Event schema versioning via event_types table** — each event type has a JSON Schema definition and version number. When event schemas evolve, old events can be upcasted to the new schema during replay, maintaining backward compatibility without rewriting history.
