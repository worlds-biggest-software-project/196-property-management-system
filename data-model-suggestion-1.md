# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Property Management System · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form (3NF) relational design: every real-world concept gets its own table, relationships are expressed through foreign keys and junction tables, and reference data (property types, lease statuses, maintenance priorities) lives in dedicated lookup tables. The schema is designed so that a single SQL query can answer any business question without denormalization or reconstruction from event logs.

The approach mirrors how mature property management platforms like Yardi Voyager and Buildium structure their internal data. It aligns naturally with the OSCRE Industry Data Model's entity decomposition (separate entities for Property, Lease, Space, Work Order, Party) and the RESO Data Dictionary's field-level standardization. Double-entry trust accounting tables follow GAAP conventions with explicit debit/credit journal entries, supporting ASC 842 / IFRS 16 lease classification.

This is the "safe default" for teams with strong SQL skills who need complex cross-entity reporting (owner statements, portfolio P&L, compliance dashboards) and who prefer schema-level data integrity over application-level validation.

**Best for:** Teams building a full-featured platform with trust accounting, regulatory compliance, and complex multi-entity reporting where data integrity is paramount.

**Trade-offs:**
- (+) Strong referential integrity enforced at the database level
- (+) Complex ad-hoc queries are straightforward with standard JOINs
- (+) Well-understood by most developers and DBAs
- (+) Natural fit for double-entry accounting with balanced journal entries
- (-) Schema migrations required for every new field or entity
- (-) High table count (~55-65 tables) increases cognitive overhead
- (-) Junction tables for many-to-many relationships add write complexity
- (-) Jurisdiction-specific fields require either nullable columns or separate tables per jurisdiction
- (-) Audit trails require explicit trigger-based or application-level history tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RESO Data Dictionary 2.x | Field naming for property attributes (ListPrice, BedroomsTotal, BathroomsFullCount, LivingArea, LotSizeAcres) |
| OSCRE Industry Data Model | Entity decomposition for Lease, Space, Work Order, Party follows OSCRE's 130+ use-case structure |
| IBPDI CDM | Building and Area Measurement entity schemas inform the `properties` and `units` table design |
| ISO 3166-1/2 | Country and subdivision codes for jurisdiction modeling in `jurisdictions` table |
| ASC 842 / IFRS 16 | Lease classification fields (lease_type, transfer_of_ownership, purchase_option, etc.) on `leases` table |
| NACHA / ACH | Payment SEC codes and routing metadata on `payments` table for rent collection compliance |
| FCRA | Screening consent and adverse action tracking fields on `screening_requests` table |
| ISO 4217 | Currency codes on all monetary fields for multi-currency support |
| PCI DSS 4.0 | No card data stored; `payment_methods` references tokenized processor IDs only |

---

## Core Identity & Multi-Tenancy

```sql
-- Organization (top-level tenant in multi-tenant SaaS)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan_tier VARCHAR(50) NOT NULL DEFAULT 'free', -- free, growth, professional, enterprise
    timezone VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users (staff, landlords, property managers)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(30),
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);

-- Roles and permissions (RBAC)
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(100) NOT NULL, -- admin, property_manager, accountant, maintenance_coordinator, viewer
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) UNIQUE NOT NULL, -- e.g. properties.read, leases.write, accounting.approve
    description TEXT
);

CREATE TABLE role_permissions (
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

---

## Property & Unit Management

```sql
-- Properties (buildings, houses, complexes)
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    property_type VARCHAR(50) NOT NULL, -- residential, commercial, mixed_use, hoa
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL DEFAULT 'US', -- ISO 3166-1 alpha-2
    jurisdiction_id UUID REFERENCES jurisdictions(id),
    latitude NUMERIC(10, 7),
    longitude NUMERIC(10, 7),
    year_built INTEGER,
    lot_size_acres NUMERIC(10, 4), -- RESO: LotSizeAcres
    total_units INTEGER NOT NULL DEFAULT 1,
    acquisition_date DATE,
    acquisition_price NUMERIC(14, 2),
    current_market_value NUMERIC(14, 2),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_properties_org ON properties(organization_id);
CREATE INDEX idx_properties_type ON properties(organization_id, property_type);

-- Units within a property
CREATE TABLE units (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id),
    unit_number VARCHAR(50) NOT NULL,
    unit_type VARCHAR(50), -- apartment, townhouse, office, retail, storage
    bedrooms INTEGER, -- RESO: BedroomsTotal
    bathrooms_full INTEGER, -- RESO: BathroomsFullCount
    bathrooms_half INTEGER,
    living_area_sqft NUMERIC(10, 2), -- RESO: LivingArea
    floor_number INTEGER,
    market_rent NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'vacant', -- vacant, occupied, maintenance, offline
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, unit_number)
);
CREATE INDEX idx_units_property ON units(property_id);
CREATE INDEX idx_units_status ON units(status);

-- Jurisdictions (for compliance rules)
CREATE TABLE jurisdictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code CHAR(2) NOT NULL, -- ISO 3166-1
    subdivision_code VARCHAR(10), -- ISO 3166-2 (e.g. US-CA, US-NY)
    municipality VARCHAR(255),
    rent_control_active BOOLEAN NOT NULL DEFAULT false,
    max_security_deposit_months NUMERIC(4, 2),
    notice_to_vacate_days INTEGER,
    late_fee_max_percent NUMERIC(5, 2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdictions_country ON jurisdictions(country_code, subdivision_code);
```

---

## Tenant & Lease Management

```sql
-- Tenants (people who rent units)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(30),
    date_of_birth DATE,
    ssn_last_four CHAR(4), -- stored encrypted; only last 4 for display
    emergency_contact_name VARCHAR(200),
    emergency_contact_phone VARCHAR(30),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tenants_org ON tenants(organization_id);
CREATE INDEX idx_tenants_name ON tenants(organization_id, last_name, first_name);

-- Leases
CREATE TABLE leases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    lease_status VARCHAR(30) NOT NULL DEFAULT 'draft', -- draft, active, expired, terminated, renewed
    lease_type VARCHAR(30) NOT NULL, -- fixed_term, month_to_month, commercial_net, commercial_gross
    -- ASC 842 classification fields
    asc842_classification VARCHAR(20), -- operating, finance (NULL if not applicable)
    transfer_of_ownership BOOLEAN DEFAULT false,
    purchase_option BOOLEAN DEFAULT false,
    lease_term_months INTEGER NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    monthly_rent NUMERIC(10, 2) NOT NULL,
    security_deposit NUMERIC(10, 2),
    late_fee_amount NUMERIC(10, 2),
    late_fee_grace_days INTEGER DEFAULT 5,
    rent_due_day INTEGER NOT NULL DEFAULT 1, -- day of month rent is due
    auto_renew BOOLEAN NOT NULL DEFAULT false,
    renewal_term_months INTEGER,
    move_in_date DATE,
    move_out_date DATE,
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_leases_unit ON leases(unit_id);
CREATE INDEX idx_leases_status ON leases(lease_status);
CREATE INDEX idx_leases_dates ON leases(start_date, end_date);

-- Lease-tenant junction (multiple tenants per lease)
CREATE TABLE lease_tenants (
    lease_id UUID NOT NULL REFERENCES leases(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    is_primary BOOLEAN NOT NULL DEFAULT false,
    move_in_date DATE,
    move_out_date DATE,
    PRIMARY KEY (lease_id, tenant_id)
);

-- Lease documents (signed leases, addenda, disclosures)
CREATE TABLE lease_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    document_type VARCHAR(50) NOT NULL, -- lease_agreement, addendum, disclosure, notice
    file_name VARCHAR(255) NOT NULL,
    file_url TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type VARCHAR(100),
    esignature_status VARCHAR(30), -- pending, signed, declined, expired
    esignature_completed_at TIMESTAMPTZ,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lease_docs_lease ON lease_documents(lease_id);

-- Lease recurring charges (beyond base rent)
CREATE TABLE lease_charges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    charge_type VARCHAR(50) NOT NULL, -- pet_rent, parking, storage, utility, cam
    description VARCHAR(255),
    amount NUMERIC(10, 2) NOT NULL,
    frequency VARCHAR(20) NOT NULL DEFAULT 'monthly', -- monthly, quarterly, annually, one_time
    start_date DATE NOT NULL,
    end_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lease_charges_lease ON lease_charges(lease_id);
```

---

## Tenant Screening

```sql
-- Screening requests (FCRA-compliant)
CREATE TABLE screening_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    screening_provider VARCHAR(50) NOT NULL, -- transunion, experian
    provider_reference_id VARCHAR(255),
    consent_given_at TIMESTAMPTZ NOT NULL, -- FCRA: must record consent
    consent_method VARCHAR(30) NOT NULL, -- electronic, paper
    credit_score INTEGER,
    criminal_check_status VARCHAR(30), -- clear, flagged, pending
    eviction_check_status VARCHAR(30), -- clear, flagged, pending
    income_verification_status VARCHAR(30), -- verified, unverified, pending
    overall_recommendation VARCHAR(30), -- approve, conditional, deny
    adverse_action_sent BOOLEAN DEFAULT false, -- FCRA: adverse action notice
    adverse_action_sent_at TIMESTAMPTZ,
    report_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_screening_tenant ON screening_requests(tenant_id);
```

---

## Maintenance & Work Orders

```sql
-- Maintenance requests (submitted by tenants or staff)
CREATE TABLE maintenance_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    reported_by_tenant_id UUID REFERENCES tenants(id),
    reported_by_user_id UUID REFERENCES users(id),
    category VARCHAR(50) NOT NULL, -- plumbing, electrical, hvac, appliance, structural, pest, general
    priority VARCHAR(20) NOT NULL DEFAULT 'normal', -- emergency, urgent, normal, low
    ai_priority VARCHAR(20), -- AI-assigned priority for comparison
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    permission_to_enter BOOLEAN DEFAULT false,
    status VARCHAR(30) NOT NULL DEFAULT 'open', -- open, assigned, in_progress, completed, cancelled
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maintenance_unit ON maintenance_requests(unit_id);
CREATE INDEX idx_maintenance_status ON maintenance_requests(status);
CREATE INDEX idx_maintenance_priority ON maintenance_requests(priority);

-- Maintenance photos
CREATE TABLE maintenance_photos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    maintenance_request_id UUID NOT NULL REFERENCES maintenance_requests(id) ON DELETE CASCADE,
    file_url TEXT NOT NULL,
    file_name VARCHAR(255),
    description TEXT,
    uploaded_by_type VARCHAR(20) NOT NULL, -- tenant, staff, vendor
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Vendors (contractors, service providers)
CREATE TABLE vendors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    company_name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(30),
    specialty VARCHAR(100), -- plumbing, electrical, hvac, general, cleaning
    hourly_rate NUMERIC(8, 2),
    insurance_expiry DATE,
    license_number VARCHAR(100),
    avg_rating NUMERIC(3, 2), -- 1.00 to 5.00
    total_jobs_completed INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);
CREATE INDEX idx_vendors_specialty ON vendors(organization_id, specialty);

-- Work orders (assigned to vendors from maintenance requests)
CREATE TABLE work_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    maintenance_request_id UUID NOT NULL REFERENCES maintenance_requests(id),
    vendor_id UUID REFERENCES vendors(id),
    assigned_by UUID REFERENCES users(id),
    status VARCHAR(30) NOT NULL DEFAULT 'pending', -- pending, accepted, scheduled, in_progress, completed, cancelled
    scheduled_date DATE,
    scheduled_time_start TIME,
    scheduled_time_end TIME,
    actual_start_at TIMESTAMPTZ,
    actual_end_at TIMESTAMPTZ,
    labor_cost NUMERIC(10, 2),
    materials_cost NUMERIC(10, 2),
    total_cost NUMERIC(10, 2),
    vendor_notes TEXT,
    tenant_rating INTEGER CHECK (tenant_rating BETWEEN 1 AND 5),
    tenant_feedback TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_work_orders_maintenance ON work_orders(maintenance_request_id);
CREATE INDEX idx_work_orders_vendor ON work_orders(vendor_id);
CREATE INDEX idx_work_orders_status ON work_orders(status);
```

---

## Accounting & Payments

```sql
-- Chart of accounts
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    account_type VARCHAR(30) NOT NULL, -- asset, liability, equity, revenue, expense
    account_subtype VARCHAR(50), -- operating_bank, trust_bank, accounts_receivable, rent_income, etc.
    parent_account_id UUID REFERENCES accounts(id),
    is_system BOOLEAN NOT NULL DEFAULT false, -- system-generated accounts cannot be deleted
    is_active BOOLEAN NOT NULL DEFAULT true,
    currency CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, account_code)
);

-- Journal entries (double-entry bookkeeping)
CREATE TABLE journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entry_date DATE NOT NULL,
    description TEXT NOT NULL,
    reference_type VARCHAR(50), -- payment, invoice, adjustment, deposit, refund
    reference_id UUID, -- FK to the source record
    property_id UUID REFERENCES properties(id), -- for property-level P&L
    is_posted BOOLEAN NOT NULL DEFAULT false,
    posted_by UUID REFERENCES users(id),
    posted_at TIMESTAMPTZ,
    voided BOOLEAN NOT NULL DEFAULT false,
    voided_by UUID REFERENCES users(id),
    voided_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_journal_entries_org_date ON journal_entries(organization_id, entry_date);
CREATE INDEX idx_journal_entries_property ON journal_entries(property_id);

-- Journal entry lines (debit/credit pairs)
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

-- Payments (rent payments, owner distributions)
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    lease_id UUID REFERENCES leases(id),
    tenant_id UUID REFERENCES tenants(id),
    payment_type VARCHAR(30) NOT NULL, -- rent, security_deposit, late_fee, utility, refund
    payment_method VARCHAR(30) NOT NULL, -- ach, credit_card, debit_card, check, cash
    amount NUMERIC(10, 2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(30) NOT NULL DEFAULT 'pending', -- pending, processing, completed, failed, refunded
    -- ACH / NACHA fields
    ach_sec_code VARCHAR(10), -- WEB, TEL, PPD, CCD
    processor_reference_id VARCHAR(255), -- Stripe/Dwolla transaction ID
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
CREATE INDEX idx_payments_status ON payments(status);

-- Owner distributions
CREATE TABLE owner_distributions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    owner_id UUID NOT NULL REFERENCES owners(id),
    property_id UUID NOT NULL REFERENCES properties(id),
    distribution_date DATE NOT NULL,
    gross_income NUMERIC(14, 2) NOT NULL,
    total_expenses NUMERIC(14, 2) NOT NULL,
    management_fee NUMERIC(14, 2) NOT NULL,
    net_distribution NUMERIC(14, 2) NOT NULL,
    payment_method VARCHAR(30),
    status VARCHAR(30) NOT NULL DEFAULT 'pending', -- pending, paid, held
    journal_entry_id UUID REFERENCES journal_entries(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Property owners
CREATE TABLE owners (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(30),
    tax_id_last_four CHAR(4), -- EIN or SSN last 4
    mailing_address TEXT,
    ownership_type VARCHAR(30), -- individual, llc, corporation, trust, partnership
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owners_org ON owners(organization_id);

-- Property ownership (many-to-many with ownership percentage)
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

## Listings & Applications

```sql
-- Vacancy listings
CREATE TABLE listings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    listed_rent NUMERIC(10, 2) NOT NULL,
    available_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'active', -- active, paused, filled, expired
    -- RESO-aligned fields
    listing_id_external VARCHAR(100), -- RESO: ListingId
    syndicated_to TEXT[], -- array of platform names: zillow, apartments_com, rentler
    pet_policy VARCHAR(50), -- no_pets, cats_only, dogs_only, cats_and_dogs
    parking_included BOOLEAN DEFAULT false,
    utilities_included TEXT[], -- water, electric, gas, internet, trash
    virtual_tour_url TEXT,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listings_unit ON listings(unit_id);
CREATE INDEX idx_listings_status ON listings(status);

-- Rental applications
CREATE TABLE applications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listings(id),
    unit_id UUID NOT NULL REFERENCES units(id),
    applicant_first_name VARCHAR(100) NOT NULL,
    applicant_last_name VARCHAR(100) NOT NULL,
    applicant_email VARCHAR(255) NOT NULL,
    applicant_phone VARCHAR(30),
    desired_move_in DATE,
    current_address TEXT,
    current_landlord_name VARCHAR(200),
    current_landlord_phone VARCHAR(30),
    employer_name VARCHAR(255),
    monthly_income NUMERIC(10, 2),
    status VARCHAR(30) NOT NULL DEFAULT 'submitted', -- submitted, under_review, approved, denied, withdrawn
    screening_request_id UUID REFERENCES screening_requests(id),
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_applications_unit ON applications(unit_id);
CREATE INDEX idx_applications_status ON applications(status);
```

---

## Communication & Notifications

```sql
-- Messages (unified inbox: SMS, email, portal)
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    thread_id UUID, -- groups related messages
    channel VARCHAR(20) NOT NULL, -- portal, email, sms
    direction VARCHAR(10) NOT NULL, -- inbound, outbound
    sender_type VARCHAR(20) NOT NULL, -- user, tenant, owner, vendor, system
    sender_id UUID, -- polymorphic: user_id, tenant_id, owner_id, vendor_id
    recipient_type VARCHAR(20) NOT NULL,
    recipient_id UUID,
    subject VARCHAR(255),
    body TEXT NOT NULL,
    related_entity_type VARCHAR(50), -- property, unit, lease, maintenance_request, work_order
    related_entity_id UUID,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_org_thread ON messages(organization_id, thread_id);
CREATE INDEX idx_messages_related ON messages(related_entity_type, related_entity_id);

-- Notifications
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    owner_id UUID REFERENCES owners(id),
    notification_type VARCHAR(50) NOT NULL, -- rent_due, rent_overdue, maintenance_update, lease_expiring, payment_received
    title VARCHAR(255) NOT NULL,
    body TEXT,
    related_entity_type VARCHAR(50),
    related_entity_id UUID,
    channel VARCHAR(20) NOT NULL DEFAULT 'in_app', -- in_app, email, sms, push
    sent_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_tenant ON notifications(tenant_id);
```

---

## Inspections & Compliance

```sql
-- Property inspections
CREATE TABLE inspections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id),
    unit_id UUID REFERENCES units(id),
    inspection_type VARCHAR(50) NOT NULL, -- move_in, move_out, annual, hqs, safety
    inspector_user_id UUID REFERENCES users(id),
    inspector_vendor_id UUID REFERENCES vendors(id),
    scheduled_date DATE,
    completed_date DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'scheduled', -- scheduled, in_progress, completed, cancelled
    overall_condition VARCHAR(30), -- excellent, good, fair, poor
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inspections_property ON inspections(property_id);

-- Inspection items (room-by-room condition)
CREATE TABLE inspection_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    room_area VARCHAR(100) NOT NULL, -- kitchen, bathroom, bedroom_1, living_room, exterior
    item_name VARCHAR(100) NOT NULL, -- walls, flooring, fixtures, appliances, windows
    condition VARCHAR(30) NOT NULL, -- excellent, good, fair, poor, damaged
    notes TEXT,
    photo_url TEXT,
    requires_action BOOLEAN NOT NULL DEFAULT false,
    maintenance_request_id UUID REFERENCES maintenance_requests(id)
);

-- Section 8 / HCV tracking (HUD compliance)
CREATE TABLE hcv_vouchers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lease_id UUID NOT NULL REFERENCES leases(id),
    voucher_number VARCHAR(100) NOT NULL,
    housing_authority_name VARCHAR(255) NOT NULL,
    hap_amount NUMERIC(10, 2) NOT NULL, -- Housing Assistance Payment
    tenant_portion NUMERIC(10, 2) NOT NULL,
    payment_standard NUMERIC(10, 2),
    contract_start_date DATE NOT NULL,
    contract_end_date DATE,
    last_hqs_inspection_date DATE,
    next_hqs_inspection_due DATE,
    status VARCHAR(30) NOT NULL DEFAULT 'active', -- active, suspended, terminated
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hcv_lease ON hcv_vouchers(lease_id);
```

---

## AI Features Support

```sql
-- Delinquency risk scores (AI-generated)
CREATE TABLE delinquency_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    lease_id UUID NOT NULL REFERENCES leases(id),
    prediction_date DATE NOT NULL,
    risk_score NUMERIC(5, 4) NOT NULL, -- 0.0000 to 1.0000
    risk_level VARCHAR(20) NOT NULL, -- low, moderate, high, critical
    contributing_factors TEXT[], -- late_history, income_change, seasonal, macro
    recommended_action VARCHAR(100), -- no_action, send_reminder, outreach_call, payment_plan
    model_version VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_delinquency_tenant ON delinquency_predictions(tenant_id);
CREATE INDEX idx_delinquency_date ON delinquency_predictions(prediction_date);

-- Rent pricing recommendations (AI-generated)
CREATE TABLE rent_recommendations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id UUID NOT NULL REFERENCES units(id),
    recommendation_date DATE NOT NULL,
    current_rent NUMERIC(10, 2) NOT NULL,
    recommended_rent NUMERIC(10, 2) NOT NULL,
    market_average NUMERIC(10, 2),
    vacancy_rate_local NUMERIC(5, 4),
    confidence_score NUMERIC(5, 4),
    reasoning TEXT,
    model_version VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rent_rec_unit ON rent_recommendations(unit_id);
```

---

## Audit History

```sql
-- Generic audit log (trigger-populated)
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL, -- INSERT, UPDATE, DELETE
    changed_by UUID, -- user who made the change
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

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & RBAC | 5 | organizations, users, roles, permissions, role_permissions, user_roles |
| Properties & Units | 3 | properties, units, jurisdictions |
| Tenants & Leases | 5 | tenants, leases, lease_tenants, lease_documents, lease_charges |
| Screening | 1 | screening_requests |
| Maintenance | 4 | maintenance_requests, maintenance_photos, vendors, work_orders |
| Accounting | 6 | accounts, journal_entries, journal_entry_lines, payments, owner_distributions, owners, property_owners |
| Listings & Applications | 2 | listings, applications |
| Communication | 2 | messages, notifications |
| Inspections & Compliance | 3 | inspections, inspection_items, hcv_vouchers |
| AI Features | 2 | delinquency_predictions, rent_recommendations |
| Audit | 1 | audit_log |
| **Total** | **~34 core tables** | Expandable with tax reporting, 1099 tracking, and HOA tables |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployment, and avoids sequential ID enumeration attacks on tenant-facing APIs.

2. **Organization-scoped multi-tenancy with shared tables** — all major entities carry `organization_id` for row-level filtering. Simpler than schema-per-tenant but requires disciplined query scoping (ideally via Row Level Security policies).

3. **Double-entry journal system for accounting** — every financial transaction produces a balanced journal entry with explicit debit/credit lines. This supports trust accounting (separating owner funds from management fees), owner statements, and audit compliance.

4. **Jurisdiction table for compliance variability** — rather than hard-coding rent control rules or late fee limits, jurisdiction-specific rules are stored in a lookup table and referenced by properties. This avoids the nullable-column sprawl that would otherwise be needed.

5. **Separate tenants and users tables** — tenants are not system users by default. A tenant becomes a portal user through a separate invitation flow, keeping the authentication boundary clean.

6. **RESO field naming on property/unit attributes** — using RESO Data Dictionary field names (BedroomsTotal, BathroomsFullCount, LivingArea) ensures compatibility with MLS feeds and listing syndication APIs.

7. **ASC 842 classification fields on leases** — the five classification criteria are stored directly on the lease record, enabling automated operating vs. finance lease classification for institutional operators.

8. **Polymorphic messaging with explicit type columns** — the messages table uses sender_type/sender_id rather than separate FK columns for each party type, keeping the schema flat while supporting tenant, owner, vendor, and staff communication in a unified inbox.

9. **AI prediction tables as append-only time series** — delinquency scores and rent recommendations are stored with timestamps and model versions, creating a historical record that can be used to evaluate model accuracy over time.

10. **Trigger-based audit log with JSONB diffs** — a single audit_log table captures all changes across the system. Using JSONB for old/new values avoids creating per-table history tables while still recording field-level changes.
