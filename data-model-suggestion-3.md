# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Property Management System · Created: 2026-05-20

## Philosophy

This model uses relational tables for core entities with well-defined schemas (leases, payments, units) and JSONB columns for data that varies by jurisdiction, property type, or configuration. Instead of creating separate tables or nullable columns for every jurisdiction-specific rule, property-type-specific attribute, or custom field, this approach stores variable data in typed JSONB columns with PostgreSQL's native indexing and validation.

The hybrid approach is used by modern SaaS platforms that need to serve diverse customer segments without schema changes. Shopify stores product metafields as JSON, Stripe stores payment metadata as JSON, and Salesforce's custom fields are effectively a key-value store. For property management, where a California residential lease has different required fields than a Texas commercial lease, and where a Section 8 voucher property needs different tracking than a luxury condo, JSONB provides the flexibility to serve all segments from a single schema.

The key discipline is: structured, queryable, business-critical data stays in typed relational columns. Variable, segment-specific, or user-defined data goes in JSONB columns with GIN indexes. This avoids both the rigidity of full normalization and the chaos of pure document storage.

**Best for:** Teams building a multi-market platform that must support diverse property types, jurisdictions, and customer segments without per-customer schema migrations. Ideal for rapid MVP development where requirements are still evolving.

**Trade-offs:**
- (+) Fewer tables than fully normalized (~25 vs ~35+) — simpler to understand and maintain
- (+) Jurisdiction-specific and property-type-specific fields don't require schema migrations
- (+) Custom fields for tenants/owners can be added by end users without developer involvement
- (+) PostgreSQL JSONB operators and GIN indexes provide fast querying within JSON structures
- (+) Faster feature development — new optional fields are additive, not migratory
- (-) JSONB columns lack database-level type enforcement — validation must happen in application code or with CHECK constraints using jsonb_typeof
- (-) Complex JSONB queries can be harder to optimize than flat column queries
- (-) Reporting tools (Power BI, Metabase) handle JSONB less naturally than flat columns
- (-) Risk of "schema soup" if JSONB usage isn't governed — developers may put everything in JSON
- (-) Foreign key constraints cannot reference values inside JSONB columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RESO Data Dictionary 2.x | Core property fields (bedrooms, bathrooms, living_area) are relational columns; RESO-extended attributes stored in `reso_attributes JSONB` |
| OSCRE Industry Data Model | Entity decomposition follows OSCRE; variable OSCRE fields per use case stored in JSONB |
| IBPDI CDM | Building performance and sustainability metrics stored in `performance_data JSONB` on properties |
| ISO 3166-1/2 | Country/subdivision codes as relational columns; jurisdiction-specific rules in `rules JSONB` |
| ASC 842 / IFRS 16 | Classification criteria stored in `accounting_data JSONB` on leases — only populated for commercial/institutional leases |
| NACHA / ACH | Payment metadata (SEC codes, routing) in `processor_data JSONB` on payments |
| FCRA | Screening report details in `report_data JSONB` on screening_requests |
| HUD Section 8 | Voucher tracking fields in `subsidy_data JSONB` on leases |
| JSON Schema (draft 2020-12) | JSONB columns validated against JSON Schema definitions stored in `field_schemas` table |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    -- Organization-level settings and preferences in JSONB
    settings JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "late_fee_enabled": true,
    --   "default_late_fee_grace_days": 5,
    --   "auto_send_rent_reminders": true,
    --   "reminder_days_before": [3, 1],
    --   "branding": {"logo_url": "...", "primary_color": "#2563EB"},
    --   "enabled_modules": ["maintenance", "accounting", "screening", "ai_predictions"],
    --   "tax_reporting": {"jurisdiction": "US", "schedule_e": true, "mtd_uk": false}
    -- }
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
    role VARCHAR(50) NOT NULL DEFAULT 'viewer', -- admin, property_manager, accountant, maintenance_coordinator, viewer
    -- Role-specific permissions and preferences
    permissions JSONB NOT NULL DEFAULT '[]',
    -- Example: ["properties.read", "properties.write", "leases.read", "leases.write", "accounting.read"]
    preferences JSONB NOT NULL DEFAULT '{}',
    -- Example: {"dashboard_layout": "compact", "notifications": {"email": true, "sms": false, "push": true}}
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);
-- GIN index for permission queries: "find all users with accounting.approve permission"
CREATE INDEX idx_users_permissions ON users USING GIN (permissions);
```

---

## Properties & Units

```sql
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    property_type VARCHAR(50) NOT NULL, -- residential, commercial, mixed_use, hoa
    -- Core address fields (always present, always queryable)
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    -- Core property attributes (RESO-aligned, always queryable)
    year_built INTEGER,
    total_units INTEGER NOT NULL DEFAULT 1,
    -- Jurisdiction rules (variable by location)
    jurisdiction_rules JSONB NOT NULL DEFAULT '{}',
    -- Example for California residential: {
    --   "jurisdiction": "US-CA",
    --   "municipality": "Los Angeles",
    --   "rent_control": true,
    --   "max_annual_increase_pct": 5.0,
    --   "just_cause_eviction": true,
    --   "max_security_deposit_months": 1,
    --   "notice_to_vacate_days": 60,
    --   "late_fee_max_pct": 6.0,
    --   "required_disclosures": ["lead_paint", "mold", "bed_bugs", "flood_zone"]
    -- }
    -- Example for Texas commercial: {
    --   "jurisdiction": "US-TX",
    --   "rent_control": false,
    --   "cam_reconciliation_required": true,
    --   "property_tax_protest_deadline": "05-15"
    -- }
    -- Extended attributes (property-type-specific, IBPDI-aligned)
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for residential: {
    --   "amenities": ["pool", "gym", "laundry", "parking_garage"],
    --   "pet_policy": {"cats": true, "dogs": true, "weight_limit_lbs": 50, "monthly_pet_rent": 35},
    --   "parking_spaces": 120,
    --   "hoa_managed": false
    -- }
    -- Example for commercial: {
    --   "building_class": "A",
    --   "rentable_area_sqft": 45000,
    --   "common_area_factor": 1.15,
    --   "cam_per_sqft": 12.50,
    --   "operating_hours": {"weekday": "06:00-22:00", "weekend": "08:00-18:00"}
    -- }
    -- Financial data (acquisition, valuation)
    financial_data JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "acquisition_date": "2020-03-15",
    --   "acquisition_price": 2500000,
    --   "current_market_value": 3100000,
    --   "annual_insurance_premium": 12000,
    --   "annual_property_tax": 28000,
    --   "mortgage": {"lender": "Chase", "balance": 1800000, "monthly_payment": 9500, "rate": 4.25}
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_properties_org ON properties(organization_id);
CREATE INDEX idx_properties_type ON properties(organization_id, property_type);
CREATE INDEX idx_properties_location ON properties(country_code, state_province, city);
-- GIN index for querying inside jurisdiction rules
CREATE INDEX idx_properties_jurisdiction ON properties USING GIN (jurisdiction_rules);
-- GIN index for querying extended attributes (e.g., find all properties with pool)
CREATE INDEX idx_properties_extended ON properties USING GIN (extended_attributes);

CREATE TABLE units (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id),
    unit_number VARCHAR(50) NOT NULL,
    unit_type VARCHAR(50), -- apartment, townhouse, office, retail, storage
    -- Core RESO fields (always present for residential)
    bedrooms INTEGER,
    bathrooms_full INTEGER,
    bathrooms_half INTEGER,
    living_area_sqft NUMERIC(10, 2),
    floor_number INTEGER,
    market_rent NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'vacant',
    -- Unit-specific attributes (vary by property type)
    attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for residential: {
    --   "has_washer_dryer": true,
    --   "has_dishwasher": true,
    --   "flooring": "hardwood",
    --   "view": "city",
    --   "renovated_year": 2023,
    --   "parking_spot": "B-42"
    -- }
    -- Example for commercial: {
    --   "usable_area_sqft": 2200,
    --   "rentable_area_sqft": 2530,
    --   "load_factor": 1.15,
    --   "suite_number": "400A",
    --   "wired_for": ["ethernet", "fiber"],
    --   "ceiling_height_ft": 12
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, unit_number)
);
CREATE INDEX idx_units_property ON units(property_id);
CREATE INDEX idx_units_status ON units(status);
```

---

## Tenants & Leases

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(30),
    -- Extended tenant info in JSONB (varies by screening level, jurisdiction)
    profile JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "date_of_birth": "1990-05-15",
    --   "ssn_last_four": "4321",
    --   "emergency_contact": {"name": "Jane Doe", "phone": "555-0102", "relationship": "spouse"},
    --   "vehicles": [{"make": "Toyota", "model": "Camry", "year": 2022, "plate": "ABC1234", "state": "CA"}],
    --   "pets": [{"type": "dog", "breed": "Labrador", "weight_lbs": 65, "name": "Max"}],
    --   "employer": {"name": "Acme Corp", "phone": "555-0200", "monthly_income": 6500}
    -- }
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
    lease_type VARCHAR(30) NOT NULL, -- fixed_term, month_to_month, commercial_net, commercial_gross
    -- Core lease terms (always present, always queryable)
    start_date DATE NOT NULL,
    end_date DATE,
    monthly_rent NUMERIC(10, 2) NOT NULL,
    security_deposit NUMERIC(10, 2),
    rent_due_day INTEGER NOT NULL DEFAULT 1,
    lease_term_months INTEGER NOT NULL,
    -- Lease-specific terms (vary by type and jurisdiction)
    terms JSONB NOT NULL DEFAULT '{}',
    -- Example for residential fixed-term: {
    --   "late_fee_amount": 50.00,
    --   "late_fee_grace_days": 5,
    --   "auto_renew": true,
    --   "renewal_term_months": 12,
    --   "utilities_tenant_pays": ["electric", "gas", "internet"],
    --   "parking_included": true,
    --   "parking_spot": "B-42",
    --   "move_in_date": "2026-06-01",
    --   "move_out_date": null,
    --   "pet_deposit": 300,
    --   "recurring_charges": [
    --     {"type": "pet_rent", "amount": 35, "frequency": "monthly"},
    --     {"type": "storage", "amount": 75, "frequency": "monthly"}
    --   ]
    -- }
    -- Example for commercial NNN: {
    --   "base_rent_per_sqft": 28.00,
    --   "cam_per_sqft": 12.50,
    --   "insurance_per_sqft": 2.00,
    --   "tax_per_sqft": 8.50,
    --   "escalation_type": "cpi",
    --   "escalation_cap_pct": 3.0,
    --   "tenant_improvement_allowance": 45000,
    --   "free_rent_months": 2,
    --   "renewal_options": [
    --     {"term_years": 5, "notice_months": 6, "rent_adjustment": "market"}
    --   ]
    -- }
    -- Accounting / compliance data (only for institutional operators)
    accounting_data JSONB NOT NULL DEFAULT '{}',
    -- Example for ASC 842: {
    --   "asc842_classification": "operating",
    --   "transfer_of_ownership": false,
    --   "purchase_option": false,
    --   "discount_rate": 0.05,
    --   "right_of_use_asset": 210000,
    --   "lease_liability": 210000
    -- }
    -- Subsidy / voucher data (only for Section 8 / HCV)
    subsidy_data JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "program": "section_8_hcv",
    --   "voucher_number": "HCV-2026-00123",
    --   "housing_authority": "LA Housing Authority",
    --   "hap_amount": 1200,
    --   "tenant_portion": 650,
    --   "payment_standard": 1900,
    --   "contract_start": "2026-06-01",
    --   "contract_end": "2027-05-31",
    --   "last_hqs_inspection": "2026-04-15",
    --   "next_hqs_inspection_due": "2027-04-15"
    -- }
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_leases_unit ON leases(unit_id);
CREATE INDEX idx_leases_status ON leases(lease_status);
CREATE INDEX idx_leases_dates ON leases(start_date, end_date);
-- GIN index for querying lease terms (e.g., find all leases with auto_renew)
CREATE INDEX idx_leases_terms ON leases USING GIN (terms);
-- GIN index for Section 8 queries
CREATE INDEX idx_leases_subsidy ON leases USING GIN (subsidy_data);

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

## Maintenance & Vendors

```sql
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    company_name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(30),
    specialties VARCHAR(100)[] NOT NULL DEFAULT '{}', -- PostgreSQL array: {plumbing, hvac, electrical}
    hourly_rate NUMERIC(8, 2),
    -- Extended vendor data
    details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "insurance_expiry": "2027-01-15",
    --   "license_number": "C-39-12345",
    --   "license_state": "CA",
    --   "avg_rating": 4.7,
    --   "total_jobs": 142,
    --   "avg_response_hours": 2.3,
    --   "avg_completion_days": 1.8,
    --   "service_area_zip_codes": ["90001", "90002", "90003"],
    --   "preferred_contact_method": "sms",
    --   "w9_on_file": true
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);
CREATE INDEX idx_vendors_specialties ON vendors USING GIN (specialties);

CREATE TABLE maintenance_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    reported_by_tenant_id UUID REFERENCES tenants(id),
    reported_by_user_id UUID REFERENCES users(id),
    category VARCHAR(50) NOT NULL,
    priority VARCHAR(20) NOT NULL DEFAULT 'normal',
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'open',
    permission_to_enter BOOLEAN DEFAULT false,
    -- AI triage results
    ai_triage JSONB,
    -- Example: {
    --   "ai_priority": "urgent",
    --   "ai_category": "plumbing",
    --   "confidence": 0.94,
    --   "recommended_vendor_id": "uuid",
    --   "reasoning": "Water leak — potential structural damage risk",
    --   "model_version": "maint-triage-v2.3",
    --   "triaged_at": "2026-06-01T10:30:00Z"
    -- }
    -- Photos stored as JSONB array
    photos JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"url": "https://...", "filename": "leak_1.jpg", "uploaded_by": "tenant", "uploaded_at": "..."},
    --   {"url": "https://...", "filename": "leak_2.jpg", "uploaded_by": "tenant", "uploaded_at": "..."}
    -- ]
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maint_unit ON maintenance_requests(unit_id);
CREATE INDEX idx_maint_status ON maintenance_requests(status);
CREATE INDEX idx_maint_priority ON maintenance_requests(priority);

CREATE TABLE work_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    maintenance_request_id UUID NOT NULL REFERENCES maintenance_requests(id),
    vendor_id UUID REFERENCES vendors(id),
    assigned_by UUID REFERENCES users(id),
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    scheduled_date DATE,
    -- Work order details and costs
    details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "scheduled_time_start": "09:00",
    --   "scheduled_time_end": "12:00",
    --   "actual_start": "2026-06-02T09:15:00Z",
    --   "actual_end": "2026-06-02T11:30:00Z",
    --   "labor_cost": 250.00,
    --   "materials_cost": 85.00,
    --   "materials": [
    --     {"item": "P-trap assembly", "qty": 1, "cost": 35.00},
    --     {"item": "PVC pipe 2in", "qty": 3, "cost": 50.00}
    --   ],
    --   "total_cost": 335.00,
    --   "vendor_notes": "Replaced corroded P-trap and 3ft of drain pipe",
    --   "tenant_rating": 5,
    --   "tenant_feedback": "Fast and professional"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_maint ON work_orders(maintenance_request_id);
CREATE INDEX idx_wo_vendor ON work_orders(vendor_id);
CREATE INDEX idx_wo_status ON work_orders(status);
```

---

## Accounting & Payments

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    account_type VARCHAR(30) NOT NULL, -- asset, liability, equity, revenue, expense
    account_subtype VARCHAR(50),
    parent_account_id UUID REFERENCES accounts(id),
    is_system BOOLEAN NOT NULL DEFAULT false,
    is_active BOOLEAN NOT NULL DEFAULT true,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
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
    payment_date DATE NOT NULL,
    posted_date DATE,
    journal_entry_id UUID REFERENCES journal_entries(id),
    -- Processor-specific data (varies by payment processor)
    processor_data JSONB NOT NULL DEFAULT '{}',
    -- Example for Stripe ACH: {
    --   "processor": "stripe",
    --   "reference_id": "pi_abc123",
    --   "ach_sec_code": "WEB",
    --   "bank_name": "Chase",
    --   "last_four": "6789",
    --   "failure_reason": null,
    --   "fee_amount": 2.50
    -- }
    -- Example for Dwolla ACH: {
    --   "processor": "dwolla",
    --   "reference_id": "txn_xyz789",
    --   "ach_sec_code": "WEB",
    --   "funding_source_id": "fs_abc",
    --   "clearing_date": "2026-06-03"
    -- }
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_lease ON payments(lease_id);
CREATE INDEX idx_payments_tenant ON payments(tenant_id);
CREATE INDEX idx_payments_date ON payments(payment_date);
CREATE INDEX idx_payments_status ON payments(status);
```

---

## Owners & Distributions

```sql
CREATE TABLE owners (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    ownership_type VARCHAR(30), -- individual, llc, corporation, trust, partnership
    -- Tax and compliance info (varies by entity type and jurisdiction)
    tax_info JSONB NOT NULL DEFAULT '{}',
    -- Example for US individual: {
    --   "tax_id_type": "ssn",
    --   "tax_id_last_four": "4321",
    --   "1099_required": true,
    --   "1099_threshold": 600,
    --   "w9_on_file": true,
    --   "w9_received_date": "2026-01-15"
    -- }
    -- Example for UK company: {
    --   "tax_id_type": "company_registration",
    --   "company_number": "12345678",
    --   "vat_registered": true,
    --   "vat_number": "GB123456789",
    --   "mtd_enrolled": true
    -- }
    mailing_address TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owners_org ON owners(organization_id);

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

## Listings, Applications & Screening

```sql
CREATE TABLE listings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    listed_rent NUMERIC(10, 2) NOT NULL,
    available_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active',
    -- Listing details (vary by platform and property type)
    listing_data JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "syndicated_to": ["zillow", "apartments_com", "rentler"],
    --   "zillow_listing_id": "zpid_12345",
    --   "pet_policy": "cats_and_dogs",
    --   "pet_deposit": 300,
    --   "parking_included": true,
    --   "utilities_included": ["water", "trash"],
    --   "virtual_tour_url": "https://...",
    --   "photos": ["https://...", "https://..."],
    --   "showing_instructions": "Call 24 hours ahead"
    -- }
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listings_unit ON listings(unit_id);
CREATE INDEX idx_listings_status ON listings(status);

CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listings(id),
    unit_id UUID NOT NULL REFERENCES units(id),
    status VARCHAR(30) NOT NULL DEFAULT 'submitted',
    -- Applicant data (structured for screening)
    applicant_data JSONB NOT NULL,
    -- Example: {
    --   "first_name": "John",
    --   "last_name": "Smith",
    --   "email": "john@example.com",
    --   "phone": "555-0100",
    --   "desired_move_in": "2026-07-01",
    --   "current_address": "123 Main St, Apt 4, LA CA 90001",
    --   "current_landlord": {"name": "Bob Jones", "phone": "555-0200"},
    --   "employment": {"employer": "Acme Corp", "title": "Engineer", "monthly_income": 8500},
    --   "co_applicants": [
    --     {"first_name": "Jane", "last_name": "Smith", "relationship": "spouse", "income": 6000}
    --   ]
    -- }
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_applications_unit ON applications(unit_id);
CREATE INDEX idx_applications_status ON applications(status);

CREATE TABLE screening_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    tenant_id UUID REFERENCES tenants(id),
    application_id UUID REFERENCES applications(id),
    screening_provider VARCHAR(50) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    consent_given_at TIMESTAMPTZ NOT NULL,
    consent_method VARCHAR(30) NOT NULL,
    overall_recommendation VARCHAR(30),
    -- Full report data (provider-specific, sensitive)
    report_data JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "provider_reference_id": "scr_abc123",
    --   "credit_score": 720,
    --   "criminal_status": "clear",
    --   "eviction_status": "clear",
    --   "income_verified": true,
    --   "income_to_rent_ratio": 3.4,
    --   "adverse_action_sent": false,
    --   "adverse_action_sent_at": null,
    --   "report_expires_at": "2026-07-01T00:00:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_screening_tenant ON screening_requests(tenant_id);
```

---

## Communication & Notifications

```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    thread_id UUID,
    channel VARCHAR(20) NOT NULL, -- portal, email, sms
    direction VARCHAR(10) NOT NULL, -- inbound, outbound
    sender_type VARCHAR(20) NOT NULL,
    sender_id UUID,
    recipient_type VARCHAR(20) NOT NULL,
    recipient_id UUID,
    subject VARCHAR(255),
    body TEXT NOT NULL,
    -- Message metadata (delivery status, related entity, attachments)
    metadata JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "related_entity_type": "maintenance_request",
    --   "related_entity_id": "uuid",
    --   "delivery_status": "delivered",
    --   "read_at": "2026-06-01T14:30:00Z",
    --   "attachments": [{"filename": "photo.jpg", "url": "https://...", "size_bytes": 245000}]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_org_thread ON messages(organization_id, thread_id);
CREATE INDEX idx_messages_metadata ON messages USING GIN (metadata);
```

---

## AI Features & Predictions

```sql
CREATE TABLE ai_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    prediction_type VARCHAR(50) NOT NULL, -- delinquency, rent_pricing, lease_renewal, maintenance_triage
    entity_type VARCHAR(50) NOT NULL, -- tenant, unit, lease, maintenance_request
    entity_id UUID NOT NULL,
    prediction_date DATE NOT NULL,
    -- Prediction results (vary by type)
    results JSONB NOT NULL,
    -- Example for delinquency: {
    --   "risk_score": 0.73,
    --   "risk_level": "high",
    --   "contributing_factors": ["late_3_of_last_6", "income_decrease", "seasonal_q4"],
    --   "recommended_action": "outreach_call",
    --   "30_day_probability": 0.45,
    --   "60_day_probability": 0.73
    -- }
    -- Example for rent_pricing: {
    --   "current_rent": 1850,
    --   "recommended_rent": 1925,
    --   "market_average": 1960,
    --   "vacancy_rate_local": 0.042,
    --   "confidence": 0.88,
    --   "comparable_units": 23,
    --   "reasoning": "Below market by 5.9%; low vacancy supports increase"
    -- }
    model_version VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_entity ON ai_predictions(entity_type, entity_id, prediction_date);
CREATE INDEX idx_ai_type ON ai_predictions(prediction_type, prediction_date);
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
CREATE INDEX idx_audit_date ON audit_log(created_at);
```

---

## Field Schema Registry (Optional)

```sql
-- Defines expected JSONB structures per column per entity type
-- Used by application code for validation (or CHECK constraints)
CREATE TABLE field_schemas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    column_name VARCHAR(100) NOT NULL,
    entity_subtype VARCHAR(50), -- e.g. residential, commercial (NULL = applies to all)
    json_schema JSONB NOT NULL, -- JSON Schema (draft 2020-12) definition
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (table_name, column_name, entity_subtype)
);
```

---

## Example Queries

### Find all properties in rent-controlled jurisdictions

```sql
SELECT id, name, address_line1, city, state_province,
       jurisdiction_rules->>'max_annual_increase_pct' AS max_increase
FROM properties
WHERE organization_id = '{{org_id}}'
  AND jurisdiction_rules @> '{"rent_control": true}';
```

### Find all leases with Section 8 vouchers expiring in 90 days

```sql
SELECT l.id, u.unit_number, p.name AS property_name,
       l.subsidy_data->>'voucher_number' AS voucher,
       l.subsidy_data->>'hap_amount' AS hap_amount,
       (l.subsidy_data->>'contract_end')::date AS contract_end
FROM leases l
JOIN units u ON u.id = l.unit_id
JOIN properties p ON p.id = u.property_id
WHERE l.subsidy_data->>'program' = 'section_8_hcv'
  AND (l.subsidy_data->>'contract_end')::date <= CURRENT_DATE + INTERVAL '90 days'
  AND l.lease_status = 'active';
```

### Find vendors by specialty within a service area

```sql
SELECT id, company_name, hourly_rate, details->>'avg_rating' AS rating
FROM vendors
WHERE organization_id = '{{org_id}}'
  AND 'plumbing' = ANY(specialties)
  AND details->'service_area_zip_codes' ? '90001'
  AND is_active = true
ORDER BY (details->>'avg_rating')::numeric DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Users | 2 | organizations, users (RBAC via JSONB permissions) |
| Properties & Units | 2 | properties, units (jurisdiction rules and attributes in JSONB) |
| Tenants & Leases | 3 | tenants, leases, lease_tenants (terms, subsidy, accounting in JSONB) |
| Maintenance | 3 | vendors, maintenance_requests, work_orders (AI triage and costs in JSONB) |
| Accounting | 4 | accounts, journal_entries, journal_entry_lines, payments |
| Owners | 2 | owners, property_owners |
| Listings & Applications | 3 | listings, applications, screening_requests |
| Communication | 1 | messages (metadata in JSONB) |
| AI Predictions | 1 | ai_predictions (unified table with type-specific JSONB results) |
| Audit | 1 | audit_log |
| Schema Registry | 1 | field_schemas (optional) |
| **Total** | **~23 tables** | Significantly fewer than normalized model due to JSONB consolidation |

---

## Key Design Decisions

1. **JSONB for jurisdiction-specific rules** — instead of a separate `jurisdictions` table with fixed columns (that would need constant migration for new jurisdictions), each property carries its own `jurisdiction_rules` JSONB. This allows California rent control fields, Texas property tax deadlines, and UK council tax rules to coexist without schema changes.

2. **Unified AI predictions table** — rather than separate tables for delinquency, rent pricing, and maintenance triage predictions, a single `ai_predictions` table with `prediction_type` and JSONB `results` handles all AI output. New prediction types require zero schema changes.

3. **JSONB for lease terms with relational core** — the five fields every lease needs (start_date, end_date, monthly_rent, security_deposit, rent_due_day) are relational columns for indexing and constraint enforcement. Everything else (late fees, parking, pet deposits, CPI escalations, tenant improvement allowances) lives in `terms JSONB`, allowing residential and commercial leases to share the same table without nullable column sprawl.

4. **Payment processor abstraction via JSONB** — different payment processors (Stripe, Dwolla, Plaid) return different metadata. The `processor_data JSONB` column stores provider-specific fields without creating per-processor tables, making it easy to add new processors.

5. **Permissions as JSONB array on users** — for a simple RBAC model, storing permissions directly on the user record avoids the three-table (users/roles/role_permissions) join pattern. For organizations that need complex role hierarchies, this can be expanded later.

6. **GIN indexes on critical JSONB columns** — PostgreSQL GIN indexes on `jurisdiction_rules`, `terms`, `subsidy_data`, and `extended_attributes` enable fast containment queries (`@>` operator) without sacrificing the flexibility of schemaless storage.

7. **Field schema registry for governance** — the optional `field_schemas` table stores JSON Schema definitions for each JSONB column, enabling application-level validation and auto-generated documentation. This prevents the "anything goes" problem that plagues unstructured JSON usage.

8. **Photos as JSONB array** — maintenance request photos are stored as a JSONB array on the request itself rather than in a separate table. For most use cases (3-5 photos per request), this avoids an unnecessary JOIN and simplifies the API response.

9. **Subsidy data as JSONB on leases** — Section 8 / HCV voucher tracking is only relevant for a subset of leases. Storing it as `subsidy_data JSONB` avoids a separate table that would be NULL-heavy for non-subsidized leases.

10. **Tenant profile as JSONB** — emergency contacts, vehicles, pets, and employer info are not queried independently in most workflows. Storing them as a JSONB profile on the tenant record simplifies the schema without losing the ability to extract these fields for screening or lease generation.
