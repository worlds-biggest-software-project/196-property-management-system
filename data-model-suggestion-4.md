# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Property Management System · Created: 2026-05-20

## Philosophy

This model combines a traditional relational core for operational CRUD (leases, payments, maintenance) with a property graph layer for relationship-heavy queries. The graph layer models ownership chains, tenant-property histories, vendor assignment networks, and lease succession as first-class edges between nodes, enabling traversal queries that are awkward or slow in pure relational models.

Property management is inherently relationship-rich: owners hold fractional interests in properties, properties contain units, units have sequential leases, leases link to multiple tenants, tenants have payment histories across leases, vendors serve multiple properties with varying performance, and maintenance requests chain through assignment, escalation, and resolution workflows. A graph model makes these relationships explicit and traversable, while relational tables handle the structured, transactional data (accounting, payments, screening) where referential integrity matters most.

This approach is used by platforms that need relationship discovery (LinkedIn's social graph, financial fraud detection systems, supply chain networks) and is implemented here using PostgreSQL's `ltree` extension for hierarchies and dedicated `graph_nodes` / `graph_edges` tables for arbitrary relationship traversal. For teams comfortable with Neo4j or Amazon Neptune, the graph layer could be externalized to a dedicated graph database.

**Best for:** Organizations managing complex ownership structures (joint ventures, LP/GP funds, trusts), multi-property portfolios with cross-property vendor networks, and platforms that need to answer relationship questions like "show me all properties owned by entities related to this person" or "which vendors have the best track record across similar property types?"

**Trade-offs:**
- (+) Ownership chains, tenant histories, and vendor networks are first-class queryable relationships
- (+) Fraud and conflict-of-interest detection through graph traversal (related party transactions, vendor kickbacks)
- (+) Temporal relationships — edges have start/end dates, enabling historical queries ("who owned what, when?")
- (+) New relationship types can be added without schema changes (just add a new edge type)
- (+) Cross-entity analytics are natural (tenant churn paths, vendor performance networks)
- (-) Higher complexity — developers must understand both relational and graph query patterns
- (-) Graph queries in PostgreSQL (recursive CTEs or ltree) are less ergonomic than native graph databases
- (-) Dual storage (relational + graph) creates data synchronization requirements
- (-) Reporting tools have limited support for graph traversal; dashboards still need relational projections
- (-) More tables than the JSONB hybrid approach (~30+ vs ~23)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RESO Data Dictionary 2.x | Property and unit attributes use RESO field names; listing syndication uses RESO resource types |
| OSCRE Industry Data Model | Entity decomposition follows OSCRE; graph edges map to OSCRE relationship types (owns, manages, occupies, maintains) |
| IBPDI CDM | Building entities and area measurements inform property node attributes |
| ISO 3166-1/2 | Jurisdiction nodes in the graph carry ISO country and subdivision codes |
| ASC 842 / IFRS 16 | Lease classification on relational lease table; ownership graph enables portfolio-level compliance rollup |
| NACHA / ACH | Payment processing metadata on relational payments table |
| FCRA | Screening data on relational screening table with graph edges linking applicant to screening events |
| ISO 17442 (LEI) | Legal Entity Identifier field on owner nodes for institutional operators |
| W3C RDF / Property Graph Model | Graph layer follows the labeled property graph model (nodes, edges, properties) |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH NODES: Every entity that participates in relationships
-- ============================================================
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL, 
    -- Node types: person, organization_entity, property, unit, lease, 
    --             vendor, maintenance_request, bank_account, jurisdiction
    entity_id UUID NOT NULL, -- FK to the relational table for this entity
    label VARCHAR(255) NOT NULL, -- human-readable label for display
    properties JSONB NOT NULL DEFAULT '{}', -- graph-queryable attributes
    -- Example for a person node:
    -- {"email": "john@example.com", "roles": ["tenant", "owner"], "risk_score": 0.23}
    -- Example for a property node:
    -- {"address": "123 Main St", "type": "residential", "units": 12, "value": 3100000}
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gnodes_org_type ON graph_nodes(organization_id, node_type);
CREATE INDEX idx_gnodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_gnodes_properties ON graph_nodes USING GIN (properties);

-- ============================================================
-- GRAPH EDGES: Typed, directed relationships between nodes
-- ============================================================
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(50) NOT NULL,
    -- Edge types:
    -- OWNS (person/org → property, with ownership_pct)
    -- MANAGES (user → property)
    -- OCCUPIES (tenant → unit, via lease)
    -- LEASED_UNDER (tenant → lease)
    -- CONTAINS (property → unit)
    -- ASSIGNED_TO (maintenance_request → vendor)
    -- SERVES (vendor → property)
    -- SUCCEEDED_BY (lease → lease, for renewal chains)
    -- REPORTED_BY (maintenance_request → tenant)
    -- PAID_BY (payment → tenant)
    -- LOCATED_IN (property → jurisdiction)
    -- RELATED_TO (person → person, for conflict-of-interest)
    properties JSONB NOT NULL DEFAULT '{}',
    -- Example for OWNS: {"ownership_pct": 33.33, "ownership_type": "llc_member"}
    -- Example for OCCUPIES: {"lease_id": "uuid", "is_primary": true}
    -- Example for SERVES: {"avg_rating": 4.8, "total_jobs": 42, "specialties": ["plumbing"]}
    -- Example for SUCCEEDED_BY: {"renewal_type": "annual", "rent_change_pct": 3.5}
    -- Temporal: when was this relationship active?
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE, -- NULL = currently active
    weight NUMERIC(10, 4) DEFAULT 1.0, -- for ranking/scoring algorithms
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gedges_org ON graph_edges(organization_id);
CREATE INDEX idx_gedges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_gedges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_gedges_type ON graph_edges(edge_type);
CREATE INDEX idx_gedges_temporal ON graph_edges(effective_from, effective_to);
CREATE INDEX idx_gedges_properties ON graph_edges USING GIN (properties);

-- ============================================================
-- OWNERSHIP HIERARCHY (using PostgreSQL ltree for fast path queries)
-- ============================================================
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE ownership_paths (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    node_id UUID NOT NULL REFERENCES graph_nodes(id),
    path ltree NOT NULL, -- e.g. 'fund_a.spv_1.property_123'
    depth INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ownership_path ON ownership_paths USING GIST (path);
CREATE INDEX idx_ownership_node ON ownership_paths(node_id);
```

---

## Relational Core: Properties & Units

```sql
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
    phone VARCHAR(30),
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);

CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    property_type VARCHAR(50) NOT NULL,
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    year_built INTEGER,
    total_units INTEGER NOT NULL DEFAULT 1,
    acquisition_date DATE,
    acquisition_price NUMERIC(14, 2),
    current_market_value NUMERIC(14, 2),
    -- Graph node reference (for quick joins between relational and graph)
    graph_node_id UUID REFERENCES graph_nodes(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_properties_org ON properties(organization_id);
CREATE INDEX idx_properties_graph ON properties(graph_node_id);

CREATE TABLE units (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id),
    unit_number VARCHAR(50) NOT NULL,
    unit_type VARCHAR(50),
    bedrooms INTEGER,
    bathrooms_full INTEGER,
    bathrooms_half INTEGER,
    living_area_sqft NUMERIC(10, 2),
    floor_number INTEGER,
    market_rent NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'vacant',
    graph_node_id UUID REFERENCES graph_nodes(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, unit_number)
);
CREATE INDEX idx_units_property ON units(property_id);
CREATE INDEX idx_units_status ON units(status);

CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL,
    subdivision_code VARCHAR(10),
    municipality VARCHAR(255),
    rent_control_active BOOLEAN NOT NULL DEFAULT false,
    max_security_deposit_months NUMERIC(4, 2),
    notice_to_vacate_days INTEGER,
    late_fee_max_percent NUMERIC(5, 2),
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Relational Core: Owners (with Graph Integration)

```sql
CREATE TABLE owners (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    owner_type VARCHAR(30) NOT NULL, -- individual, llc, corporation, trust, partnership, fund, spv
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    entity_name VARCHAR(255), -- for non-individual owners
    email VARCHAR(255),
    phone VARCHAR(30),
    lei VARCHAR(20), -- ISO 17442 Legal Entity Identifier (for institutional operators)
    tax_id_last_four CHAR(4),
    mailing_address TEXT,
    graph_node_id UUID REFERENCES graph_nodes(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owners_org ON owners(organization_id);
CREATE INDEX idx_owners_lei ON owners(lei) WHERE lei IS NOT NULL;

-- Note: ownership relationships are primarily modeled in graph_edges (OWNS edge type)
-- This relational table provides a simplified view for basic queries
CREATE TABLE property_owners (
    property_id UUID NOT NULL REFERENCES properties(id),
    owner_id UUID NOT NULL REFERENCES owners(id),
    ownership_percentage NUMERIC(5, 2) NOT NULL DEFAULT 100.00,
    effective_date DATE NOT NULL,
    end_date DATE,
    PRIMARY KEY (property_id, owner_id, effective_date)
);
```

---

## Relational Core: Tenants & Leases

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(30),
    date_of_birth DATE,
    emergency_contact_name VARCHAR(200),
    emergency_contact_phone VARCHAR(30),
    graph_node_id UUID REFERENCES graph_nodes(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tenants_org ON tenants(organization_id);
CREATE INDEX idx_tenants_name ON tenants(organization_id, last_name, first_name);

CREATE TABLE leases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    lease_status VARCHAR(30) NOT NULL DEFAULT 'draft',
    lease_type VARCHAR(30) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    monthly_rent NUMERIC(10, 2) NOT NULL,
    security_deposit NUMERIC(10, 2),
    rent_due_day INTEGER NOT NULL DEFAULT 1,
    lease_term_months INTEGER NOT NULL,
    late_fee_amount NUMERIC(10, 2),
    late_fee_grace_days INTEGER DEFAULT 5,
    auto_renew BOOLEAN NOT NULL DEFAULT false,
    asc842_classification VARCHAR(20),
    -- Graph references
    graph_node_id UUID REFERENCES graph_nodes(id),
    previous_lease_id UUID REFERENCES leases(id), -- relational renewal chain
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_leases_unit ON leases(unit_id);
CREATE INDEX idx_leases_status ON leases(lease_status);
CREATE INDEX idx_leases_dates ON leases(start_date, end_date);
CREATE INDEX idx_leases_previous ON leases(previous_lease_id) WHERE previous_lease_id IS NOT NULL;

CREATE TABLE lease_tenants (
    lease_id UUID NOT NULL REFERENCES leases(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    is_primary BOOLEAN NOT NULL DEFAULT false,
    move_in_date DATE,
    move_out_date DATE,
    PRIMARY KEY (lease_id, tenant_id)
);
```

---

## Relational Core: Maintenance & Vendors

```sql
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    company_name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(30),
    specialty VARCHAR(100),
    hourly_rate NUMERIC(8, 2),
    insurance_expiry DATE,
    license_number VARCHAR(100),
    graph_node_id UUID REFERENCES graph_nodes(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);

CREATE TABLE maintenance_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    reported_by_tenant_id UUID REFERENCES tenants(id),
    reported_by_user_id UUID REFERENCES users(id),
    category VARCHAR(50) NOT NULL,
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    ai_priority VARCHAR(20),
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    permission_to_enter BOOLEAN DEFAULT false,
    status VARCHAR(30) NOT NULL DEFAULT 'open',
    graph_node_id UUID REFERENCES graph_nodes(id),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maint_unit ON maintenance_requests(unit_id);
CREATE INDEX idx_maint_status ON maintenance_requests(status);

CREATE TABLE work_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    maintenance_request_id UUID NOT NULL REFERENCES maintenance_requests(id),
    vendor_id UUID REFERENCES vendors(id),
    assigned_by UUID REFERENCES users(id),
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    scheduled_date DATE,
    labor_cost NUMERIC(10, 2),
    materials_cost NUMERIC(10, 2),
    total_cost NUMERIC(10, 2),
    vendor_notes TEXT,
    tenant_rating INTEGER CHECK (tenant_rating BETWEEN 1 AND 5),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_maint ON work_orders(maintenance_request_id);
CREATE INDEX idx_wo_vendor ON work_orders(vendor_id);
```

---

## Relational Core: Accounting & Payments

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    account_type VARCHAR(30) NOT NULL,
    account_subtype VARCHAR(50),
    parent_account_id UUID REFERENCES accounts(id),
    is_system BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    UNIQUE (organization_id, account_code)
);

CREATE TABLE journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entry_date DATE NOT NULL,
    description TEXT NOT NULL,
    reference_type VARCHAR(50),
    reference_id UUID,
    property_id UUID REFERENCES properties(id),
    is_posted BOOLEAN NOT NULL DEFAULT false,
    posted_by UUID REFERENCES users(id),
    posted_at TIMESTAMPTZ,
    voided BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_je_org_date ON journal_entries(organization_id, entry_date);

CREATE TABLE journal_entry_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entries(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id),
    debit NUMERIC(14, 2) NOT NULL DEFAULT 0,
    credit NUMERIC(14, 2) NOT NULL DEFAULT 0,
    memo TEXT,
    CONSTRAINT chk_debit_or_credit CHECK (
        (debit > 0 AND credit = 0) OR (debit = 0 AND credit > 0)
    )
);
CREATE INDEX idx_jel_entry ON journal_entry_lines(journal_entry_id);
CREATE INDEX idx_jel_account ON journal_entry_lines(account_id);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    lease_id UUID REFERENCES leases(id),
    tenant_id UUID REFERENCES tenants(id),
    payment_type VARCHAR(30) NOT NULL,
    payment_method VARCHAR(30) NOT NULL,
    amount NUMERIC(10, 2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    ach_sec_code VARCHAR(10),
    processor_reference_id VARCHAR(255),
    payment_date DATE NOT NULL,
    posted_date DATE,
    journal_entry_id UUID REFERENCES journal_entries(id),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_lease ON payments(lease_id);
CREATE INDEX idx_payments_tenant ON payments(tenant_id);
CREATE INDEX idx_payments_date ON payments(payment_date);
```

---

## Screening & Listings

```sql
CREATE TABLE screening_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    screening_provider VARCHAR(50) NOT NULL,
    provider_reference_id VARCHAR(255),
    consent_given_at TIMESTAMPTZ NOT NULL,
    consent_method VARCHAR(30) NOT NULL,
    credit_score INTEGER,
    criminal_check_status VARCHAR(30),
    eviction_check_status VARCHAR(30),
    overall_recommendation VARCHAR(30),
    adverse_action_sent BOOLEAN DEFAULT false,
    adverse_action_sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE listings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    listed_rent NUMERIC(10, 2) NOT NULL,
    available_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listings_unit ON listings(unit_id);

CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listings(id),
    unit_id UUID NOT NULL REFERENCES units(id),
    applicant_first_name VARCHAR(100) NOT NULL,
    applicant_last_name VARCHAR(100) NOT NULL,
    applicant_email VARCHAR(255) NOT NULL,
    applicant_phone VARCHAR(30),
    desired_move_in DATE,
    monthly_income NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'submitted',
    screening_request_id UUID REFERENCES screening_requests(id),
    reviewed_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_applications_unit ON applications(unit_id);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL,
    changed_by UUID,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_table ON audit_log(organization_id, table_name);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id);
```

---

## Example Graph Queries

### Find all properties owned (directly or indirectly) by a person

```sql
-- Using graph edges with recursive CTE
WITH RECURSIVE ownership_chain AS (
    -- Start from the person's node
    SELECT ge.target_node_id AS node_id,
           gn.node_type,
           gn.entity_id,
           gn.label,
           (ge.properties->>'ownership_pct')::numeric AS ownership_pct,
           1 AS depth,
           ARRAY[ge.source_node_id] AS path
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.target_node_id
    WHERE ge.source_node_id = '{{person_node_id}}'
      AND ge.edge_type = 'OWNS'
      AND (ge.effective_to IS NULL OR ge.effective_to > CURRENT_DATE)
    
    UNION ALL
    
    -- Follow ownership chains (person → LLC → property)
    SELECT ge.target_node_id,
           gn.node_type,
           gn.entity_id,
           gn.label,
           oc.ownership_pct * (ge.properties->>'ownership_pct')::numeric / 100 AS ownership_pct,
           oc.depth + 1,
           oc.path || ge.source_node_id
    FROM ownership_chain oc
    JOIN graph_edges ge ON ge.source_node_id = oc.node_id
    JOIN graph_nodes gn ON gn.id = ge.target_node_id
    WHERE ge.edge_type = 'OWNS'
      AND (ge.effective_to IS NULL OR ge.effective_to > CURRENT_DATE)
      AND oc.depth < 5 -- prevent infinite loops
      AND NOT ge.source_node_id = ANY(oc.path) -- cycle detection
)
SELECT entity_id AS property_id, label AS property_name, 
       ownership_pct AS effective_ownership_pct, depth
FROM ownership_chain
WHERE node_type = 'property'
ORDER BY effective_ownership_pct DESC;
```

### Find a tenant's complete occupancy history across all units

```sql
SELECT gn_unit.label AS unit,
       gn_prop.label AS property,
       ge.effective_from AS moved_in,
       ge.effective_to AS moved_out,
       ge.properties->>'lease_id' AS lease_id,
       ge.properties->>'monthly_rent' AS rent
FROM graph_edges ge
JOIN graph_nodes gn_unit ON gn_unit.id = ge.target_node_id
JOIN graph_edges ge_contains ON ge_contains.target_node_id = gn_unit.id 
    AND ge_contains.edge_type = 'CONTAINS'
JOIN graph_nodes gn_prop ON gn_prop.id = ge_contains.source_node_id
WHERE ge.source_node_id = '{{tenant_node_id}}'
  AND ge.edge_type = 'OCCUPIES'
ORDER BY ge.effective_from DESC;
```

### Find the best vendor for a plumbing job at a specific property

```sql
-- Vendors who have served this property or similar properties, ranked by performance
SELECT gn_vendor.label AS vendor_name,
       gn_vendor.entity_id AS vendor_id,
       ge.properties->>'avg_rating' AS rating,
       ge.properties->>'total_jobs' AS jobs_completed,
       CASE WHEN ge.target_node_id = '{{property_node_id}}' 
            THEN 'served_this_property' 
            ELSE 'served_similar' END AS relationship
FROM graph_edges ge
JOIN graph_nodes gn_vendor ON gn_vendor.id = ge.source_node_id
WHERE ge.edge_type = 'SERVES'
  AND ge.properties @> '{"specialties": ["plumbing"]}'::jsonb
  AND (ge.effective_to IS NULL OR ge.effective_to > CURRENT_DATE)
ORDER BY 
    (ge.target_node_id = '{{property_node_id}}') DESC, -- prefer vendors who know this property
    (ge.properties->>'avg_rating')::numeric DESC
LIMIT 5;
```

### Lease renewal chain for a unit

```sql
-- Follow the SUCCEEDED_BY edges to trace lease history
WITH RECURSIVE lease_chain AS (
    SELECT gn.entity_id AS lease_id,
           gn.label,
           ge.properties->>'rent_change_pct' AS rent_change,
           1 AS position
    FROM graph_nodes gn
    LEFT JOIN graph_edges ge ON ge.target_node_id = gn.id AND ge.edge_type = 'SUCCEEDED_BY'
    WHERE gn.entity_id = '{{first_lease_id}}'
      AND gn.node_type = 'lease'
    
    UNION ALL
    
    SELECT gn.entity_id,
           gn.label,
           ge_next.properties->>'rent_change_pct',
           lc.position + 1
    FROM lease_chain lc
    JOIN graph_nodes gn_current ON gn_current.entity_id = lc.lease_id AND gn_current.node_type = 'lease'
    JOIN graph_edges ge_next ON ge_next.source_node_id = gn_current.id AND ge_next.edge_type = 'SUCCEEDED_BY'
    JOIN graph_nodes gn ON gn.id = ge_next.target_node_id
    WHERE lc.position < 20
)
SELECT * FROM lease_chain ORDER BY position;
```

### Conflict of interest detection

```sql
-- Find if a vendor is related to any tenant or owner in the same property
SELECT gn_vendor.label AS vendor,
       ge_related.edge_type AS relationship_type,
       gn_person.label AS related_person,
       gn_person.properties->>'roles' AS person_roles
FROM graph_edges ge_serves
JOIN graph_nodes gn_vendor ON gn_vendor.id = ge_serves.source_node_id
JOIN graph_edges ge_related ON (
    ge_related.source_node_id = gn_vendor.id OR ge_related.target_node_id = gn_vendor.id
) AND ge_related.edge_type = 'RELATED_TO'
JOIN graph_nodes gn_person ON gn_person.id = CASE 
    WHEN ge_related.source_node_id = gn_vendor.id THEN ge_related.target_node_id
    ELSE ge_related.source_node_id END
WHERE ge_serves.target_node_id = '{{property_node_id}}'
  AND ge_serves.edge_type = 'SERVES';
```

---

## Using ltree for Ownership Hierarchies

```sql
-- Query: Find all entities under a specific fund
SELECT gn.label, gn.node_type, gn.entity_id, op.path, op.depth
FROM ownership_paths op
JOIN graph_nodes gn ON gn.id = op.node_id
WHERE op.path <@ 'fund_alpha'  -- all descendants of fund_alpha
ORDER BY op.path;

-- Query: Find the ownership path for a specific property
SELECT op.path, op.depth
FROM ownership_paths op
JOIN graph_nodes gn ON gn.id = op.node_id
WHERE gn.entity_id = '{{property_id}}'
  AND gn.node_type = 'property';
-- Returns: 'fund_alpha.spv_west_coast.property_sunset_apts'
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_nodes, graph_edges, ownership_paths |
| Identity & Users | 2 | organizations, users |
| Properties & Units | 3 | properties, units, jurisdictions |
| Owners | 2 | owners, property_owners |
| Tenants & Leases | 3 | tenants, leases, lease_tenants |
| Maintenance | 3 | vendors, maintenance_requests, work_orders |
| Accounting | 4 | accounts, journal_entries, journal_entry_lines, payments |
| Listings & Applications | 3 | listings, applications, screening_requests |
| Audit | 1 | audit_log |
| **Total** | **~24 relational + 3 graph = ~27 tables** | Graph layer adds 3 tables but replaces some junction tables |

---

## Key Design Decisions

1. **Graph layer alongside relational, not instead of** — the graph_nodes and graph_edges tables augment the relational core. Operational CRUD (creating a lease, recording a payment) uses relational tables. Relationship discovery, traversal, and analytics use the graph layer. This avoids forcing developers to use graph patterns for simple operations.

2. **graph_node_id on relational tables** — every entity that participates in relationships carries a `graph_node_id` FK, enabling quick bidirectional joins between relational and graph queries without full table scans.

3. **Temporal edges with effective_from/effective_to** — all graph edges have date ranges, making historical queries natural. "Who owned this property in 2024?" is a simple date filter on the OWNS edges rather than a complex temporal join on relational tables.

4. **ltree for deep ownership hierarchies** — PostgreSQL's ltree extension enables fast ancestor/descendant queries on ownership structures (fund → SPV → property → unit). This is more efficient than recursive CTEs for deep, stable hierarchies.

5. **Edge properties as JSONB** — each edge type carries type-specific metadata (ownership_pct for OWNS, avg_rating for SERVES, rent_change_pct for SUCCEEDED_BY). This avoids creating per-edge-type junction tables while keeping the graph schema uniform.

6. **Lease succession as graph edges** — rather than just a `previous_lease_id` column, lease renewals are modeled as SUCCEEDED_BY edges in the graph. This enables chain traversal queries ("show me the complete rental history of unit 4B from 2018 to today") that would require recursive CTEs on a relational self-join.

7. **Legal Entity Identifier (LEI) on owners** — for institutional operators managing fund structures, the ISO 17442 LEI provides a globally unique identifier for legal entities, enabling cross-platform entity resolution.

8. **Conflict-of-interest detection via RELATED_TO edges** — by modeling personal relationships (family, business partners) as graph edges, the system can automatically flag potential conflicts when a vendor related to an owner or tenant is assigned to work on that property.

9. **Vendor performance as edge properties** — rather than aggregating vendor ratings from work order history on every query, the SERVES edge between a vendor and property carries pre-computed performance metrics (avg_rating, total_jobs, avg_response_hours). These are updated when work orders complete.

10. **Graph layer is optional and additive** — the relational core functions independently. The graph layer can be populated lazily (backfilled from relational data) and queried only when relationship-heavy features (ownership analysis, vendor matching, tenant history) are needed. This allows incremental adoption.
