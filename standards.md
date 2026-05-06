# Standards & API Reference

> Project: Property Management System · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Provides frameworks for consistent service delivery in property management operations, relevant when building a platform that helps PMCs demonstrate tenant satisfaction and process quality.

**ISO 41001:2018 — Facility Management System**
- URL: https://www.iso.org/standard/68021.html
- Directly applicable to the management of physical building assets, maintenance workflows, and vendor coordination within a property management platform.

**ISO 14001:2015 — Environmental Management Systems**
- URL: https://www.iso.org/standard/60857.html
- Increasingly required by institutional property owners for energy and waste management compliance; relevant for the reporting and compliance modules of a PMS.

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Baseline security standard for handling tenant PII, lease documents, payment data, and owner financial records — essential for any platform storing sensitive property and financial data.

**ISO 45001:2018 — Occupational Health and Safety**
- URL: https://www.iso.org/standard/63787.html
- Relevant to the maintenance and inspections module, particularly for documenting hazardous conditions (mould, lead, structural defects) and tracking remediation to meet workplace safety obligations.

**ISO/AWI TS 32214 — Data Model for Carbon Credit Markets**
- URL: https://www.iso.org/standard/87660.html
- Emerging standard; relevant for property platforms that need to track, report, and potentially trade carbon credits tied to building energy efficiency programmes.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines the core HTTP request methods (GET, POST, PUT, DELETE, PATCH) and status codes underpinning the REST API of any property management platform.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines hyperlinking standards for REST APIs; used for pagination and resource linking in property management API responses.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authentication and authorisation framework used by AppFolio (and recommended for any multi-tenant property management SaaS); enables third-party app access with scoped permissions.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard token format used in OAuth 2.0 flows and API authentication; relevant for stateless API session management in a property management platform.

**W3C WebAuthn (Web Authentication API)**
- URL: https://www.w3.org/TR/webauthn/
- Enables passwordless and MFA login for tenant and owner portals; increasingly expected in SaaS platforms handling financial transactions.

---

### Data Model & API Specifications

**RESO Web API (Real Estate Standards Organization)**
- URL: https://www.reso.org/reso-web-api/ and https://transport.reso.org
- De-facto standard for real estate data exchange; REST-style API using OData V4 over HTTP/JSON. Organises data into Resources (Property, Member, Office, Media), Fields, and Lookups. Over 75% of MLS providers have adopted it; relevant for listing syndication, property data exchange, and MLS integration in a property management system.

**RESO Data Dictionary 2.x**
- URL: https://www.reso.org/data-dictionary/
- Standardised field definitions for real estate data entities. The baseline many MLSs certify against as of 2025–2026. Use for canonical field naming in the property data model (e.g., ListingId, ListPrice, BedroomsTotal, BathroomsFullCount).

**OSCRE Industry Data Model (Open Standards Consortium for Real Estate)**
- URL: https://www.oscre.org/Industry-Data-Model/Introducing-the-Data-Model
- Free, open standard covering property management (leasing, maintenance, operational data), investment and finance, and BIM integration. Provides canonical data schemas for lease, maintenance order, and asset records — directly applicable to the core data model of a property management platform.

**IBPDI Common Data Model for Real Estate**
- URL: https://ibpdi.org/cdm-for-real-estate/ and https://github.com/ibpdi/cdm
- Open-source global data standard for real estate; defines schemas for Area Measurement, Building, Prices, Costs, Climate, and Performance. Enables portfolio benchmarking, ESG reporting, and data interoperability across enterprise property stacks. Available on GitHub under open licence.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Standard for describing REST API contracts; Buildium, DoorLoop, and Skywalk (AppFolio wrapper) publish OpenAPI-compatible documentation. Adopting OpenAPI 3.1 enables auto-generated SDKs, Postman collections, and developer-friendly documentation.

**JSON Schema (draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for validating JSON request and response payloads; essential for defining the property, lease, tenant, and maintenance data objects in a well-typed API.

**OData V4 (OASIS Standard)**
- URL: https://www.odata.org/documentation/
- Underlying protocol for the RESO Web API; provides querying, filtering, ordering, and pagination conventions over JSON. Relevant if building RESO-compatible listing feeds.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) — see W3C & IETF above**

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0; enables single sign-on (SSO) for tenant portals, owner portals, and staff dashboards. Required for enterprise property management firms with existing identity providers (Microsoft Entra, Okta).

**PCI DSS 4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Mandatory for any platform that processes, stores, or transmits credit or debit card data. Covers cardholder data protection, access control, encryption, and vulnerability management. A property management platform must comply (or use a compliant payment processor to reduce scope).

**NACHA Operating Rules — ACH Payments**
- URL: https://www.nacha.org/
- Governing rules for US ACH electronic payments; defines same-day ACH, consumer protection requirements, and (from March 2026) new PAYROLL description standards for compensation payments. ACH constitutes 64.8% of digital rent transactions in the US; compliance is mandatory for any rent collection feature.
- NACHA Afinis API Catalog: https://www.nacha.org/content/afinis-api-catalog

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/soc-2
- Industry-standard audit for cloud SaaS handling sensitive personal and financial data across all five Trust Services Criteria (Security, Availability, Processing Integrity, Confidentiality, Privacy). Expected by enterprise property management firms evaluating software vendors.

**OWASP Application Security Verification Standard (ASVS)**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Provides a framework for securing web application features handling tenant PII, lease documents, and payment flows. Relevant to authentication, session management, input validation, and API security layers.

**FCRA (Fair Credit Reporting Act)**
- URL: https://www.ftc.gov/legal-library/browse/statutes/fair-credit-reporting-act
- US federal regulation governing tenant screening and credit reporting practices; any platform integrating with TransUnion, Experian, or Equifax for screening must comply with adverse action notice requirements and permissible purpose rules.

**GDPR (EU) / UK GDPR / CCPA (California)**
- URL: https://gdpr.eu/ | https://oag.ca.gov/privacy/ccpa
- Data privacy regulations applicable when storing tenant and owner PII; relevant for consent management, data retention policies, and right-to-erasure workflows in the platform.

---

### Compliance Frameworks

**ASC 842 / IFRS 16 — Lease Accounting Standards**
- ASC 842: https://fasb.org/standards/accounting-standards/topic-842
- IFRS 16: https://www.ifrs.org/issued-standards/list-of-standards/ifrs-16-leases/
- US GAAP (ASC 842) and international (IFRS 16) standards mandate operating vs. finance lease classification on balance sheets; relevant for any accounting module serving institutional or commercial property operators.

**HUD Section 8 / Housing Choice Voucher (HCV) Programme**
- URL: https://www.hud.gov/program_offices/public_indian_housing/programs/hcv
- Federal guidelines for subsidised housing management; platforms serving US affordable housing operators must support HCV payment tracking, Housing Assistance Payments (HAP) contracts, and inspection scheduling under HQS standards.

**Making Tax Digital (MTD) — HMRC UK**
- URL: https://www.gov.uk/government/collections/making-tax-digital
- UK requirement for digital quarterly income and expense submissions; Landlord Studio achieved HMRC-certified MTD compatibility in March 2026. A UK-facing property management platform must support direct MTD submission via the HMRC API.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Open standard introduced by Anthropic (November 2024) for connecting AI models to external data sources and tools via standardised server interfaces. Adopted by OpenAI, Google Gemini, Cursor, and major cloud providers. A property management platform exposing an MCP server would enable AI agents to query properties, leases, maintenance queues, and financial data without custom integrations — directly enabling the AI-native value proposition of this project.
- MCP GitHub organisation: https://github.com/modelcontextprotocol

---

## Similar Products — Developer Documentation & APIs

### Buildium
- **Description:** Mid-market property management SaaS for 50–1,000 unit portfolios with full accounting, maintenance, and leasing features.
- **API Documentation:** https://developer.buildium.com/
- **SDKs/Libraries:** Auto-generated Ruby SDK (https://github.com/LeadSimple/buildium); community connectors via Nango (https://nango.dev/docs/integrations/all/buildium)
- **Developer Guide:** https://www.buildium.com/features/open-api/
- **Standards:** REST/JSON; API key authentication via custom headers (x-buildium-client-id / x-buildium-client-secret)
- **Authentication:** API Key (client ID + secret pair)

### AppFolio
- **Description:** Professional property management SaaS for portfolios of 50+ units; AI leasing assistant and Stack Marketplace ecosystem.
- **API Documentation:** https://developer.appfolio.com/ and https://www.appfolio.com/stack/partners/api/
- **SDKs/Libraries:** Third-party Skywalk API wrapper (https://skywalkapi.com/); AppFolio GitHub (https://github.com/appfolio)
- **Developer Guide:** https://developer.appfolio.com/
- **Standards:** REST/JSON; OpenAPI-compatible documentation available
- **Authentication:** OAuth 2.0 (client_id + client_secret)

### DoorLoop
- **Description:** Cloud-based property management software with AI-assisted maintenance and inspections, targeting small-to-mid portfolios.
- **API Documentation:** https://api.doorloop.com/
- **SDKs/Libraries:** Community Ruby gem (https://github.com/mezbahalam/doorloop); Zapier integration for no-code automation; Fivetran connector (https://fivetran.com/docs/connectors/applications/doorloop)
- **Developer Guide:** https://support.doorloop.com/en/articles/7902913-api-documentation-and-where-to-find-help
- **Standards:** REST/JSON; resource-oriented URLs; standard HTTP status codes
- **Authentication:** API key

### TenantCloud
- **Description:** End-to-end property management tool for independent landlords with integrated accounting and QuickBooks sync.
- **API Documentation:** https://docs.api.tenantcloud.com/
- **SDKs/Libraries:** Unofficial .NET client library (https://github.com/yllibed/TenantCloudClient); TenantCloud GitHub (https://github.com/tenantcloud)
- **Developer Guide:** https://docs.api.tenantcloud.com/
- **Standards:** REST/JSON
- **Authentication:** Personal Access Token (PAT)

### Rentvine
- **Description:** Modern professional property management SaaS with trust accounting, open API at no extra cost, and voice-enabled AI assistant.
- **API Documentation:** https://www.rentvine.com/open-api
- **SDKs/Libraries:** Community Ruby SDK (https://github.com/Launch-Engine/rentvine)
- **Developer Guide:** https://www.rentvine.com/blog/open-api-integration
- **Standards:** REST/JSON; fully RESTful resource-oriented design
- **Authentication:** API key + API secret

### Yardi Breeze / Yardi Voyager
- **Description:** Yardi's family of property management platforms; Breeze for small-to-mid portfolios, Voyager for large institutional operators.
- **API Documentation:** Enterprise partner programme (not publicly documented for Breeze); Voyager exposes ERP-level integrations
- **SDKs/Libraries:** Enterprise connectors to SAP, Oracle, Power BI, and Microsoft Excel
- **Developer Guide:** Available via Yardi partner programme only
- **Standards:** REST (Breeze); proprietary enterprise integration layer (Voyager); Power BI and Excel connectors
- **Authentication:** Enterprise SSO / partner-issued credentials

### RESO Web API (Reference Implementation)
- **Description:** The industry standard for real estate data exchange, underpinning MLS integrations and listing syndication across the US property market.
- **API Documentation:** https://transport.reso.org
- **SDKs/Libraries:** RESO Standards GitHub (https://github.com/RESOStandards); OData V4 client libraries in multiple languages
- **Developer Guide:** https://www.reso.org/reso-web-api/
- **Standards:** OData V4 over REST/JSON; RESO Data Dictionary 2.x
- **Authentication:** OAuth 2.0 Bearer Token (required for RESO certification)

### Baselane (Property Banking & Accounting)
- **Description:** Landlord banking and financial management platform combining rent collection, bookkeeping, and business banking in one product.
- **API Documentation:** Not publicly documented; Baselane focuses on landlord-facing UX rather than developer ecosystem
- **SDKs/Libraries:** None publicly documented
- **Developer Guide:** Not available
- **Standards:** ACH/NACHA-compliant payment processing; FDIC-insured banking integration
- **Authentication:** Proprietary

### NACHA Afinis API Catalog
- **Description:** NACHA's curated catalogue of ACH-related APIs for payment initiation, status tracking, and compliance verification.
- **API Documentation:** https://www.nacha.org/content/afinis-api-catalog
- **SDKs/Libraries:** Varies by API provider; NACHA documents standards, not SDKs
- **Developer Guide:** https://www.nacha.org/resource-landing/apis-resource-center
- **Standards:** ACH / NACHA Operating Rules; Same-Day ACH; WEB, TEL, PPD, CCD SEC codes
- **Authentication:** Provider-specific (typically OAuth 2.0 or API key)

---

## Notes

**RESO vs. RETS:** As of 2025–2026, RETS (Real Estate Transaction Standard) feeds are being retired across the MLS landscape. RESO Web API (OData V4) is the successor and should be the default integration target for any listing syndication or MLS data feed.

**OSCRE and IBPDI convergence:** Both OSCRE and IBPDI are developing open data models for real estate with overlapping scope. IBPDI's CDM is available on GitHub under an open licence and is the more active project as of 2026. A new AI-native platform should evaluate adopting the IBPDI CDM as the canonical internal data model to ensure future interoperability.

**MCP opportunity:** No existing property management platform has published an MCP server as of May 2026. Building a first-class MCP server alongside the REST API would be a meaningful differentiator, enabling AI assistant workflows without bespoke integration work.

**ACH compliance update (March 2026):** NACHA's new PAYROLL description requirement (effective March 20, 2026) applies to ACH payments sent to consumer bank accounts for compensation. Platforms processing owner distributions must update their ACH SEC code descriptions to remain compliant.

**PCI DSS scope reduction:** Most platforms reduce PCI scope by delegating card processing to a compliant payment processor (Stripe, Dwolla, PaymentRails) rather than handling card data directly. This pattern is strongly recommended for any new property management platform.
