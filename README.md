# Property Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source property management platform for tenant management, lease tracking, maintenance, and rent collection.

Property Management System is a candidate project to build an open alternative to entrenched SaaS platforms like Buildium, AppFolio, and Yardi. It targets independent landlords, mid-market property managers, and institutional operators who today must choose between low-cost tools that lack accounting, mid-tier SaaS with opaque per-transaction fees, or expensive enterprise suites that take months to deploy.

---

## Why Property Management System?

- Incumbents are universally proprietary SaaS — none of the ten leading tools surveyed (Buildium, AppFolio, Yardi Breeze, Yardi Voyager, TurboTenant, TenantCloud, RentRedi, Landlord Studio, DoorLoop, Rentvine) ship open-source components.
- The $50–$200/month gap between free DIY tools (TurboTenant) and professional suites (Buildium from $62/month, AppFolio with a 50-unit minimum) leaves growing portfolios underserved.
- Predictive delinquency alerting, plain-language compliance summaries, and unified accounting without paid add-ons are gaps no incumbent fully addresses.
- Per-transaction fees for ePay, screening, and eSignature stack on top of subscription costs across most platforms, making total cost of ownership unpredictable.
- Cross-platform data portability is poor; switching providers requires manual migration with no standard export format, locking customers in.

---

## Key Features

### Rent Collection & Accounting

- Online rent collection via ACH, debit, and credit with autopay and automated reminders
- Income and expense tracking with bank reconciliation and owner statements
- 1099 e-filing and tax-ready reports (US Schedule E; UK MTD as a stretch goal)
- Automated late fee calculation and recurring task reminders

### Leasing & Tenant Lifecycle

- Lease creation and management with eSignature and state-aware templates
- Tenant screening (credit, criminal, eviction history) via TransUnion or Experian
- Listing syndication to Zillow, Apartments.com, Rentler, and other major platforms
- Online applications with lead management and showing scheduling

### Maintenance & Operations

- Maintenance request portal with photo uploads, vendor assignment, and work order tracking
- AI-powered maintenance triage that classifies request urgency and routes vendors automatically
- AI-generated inspection reports from photos using computer vision
- Vendor performance tracking across turnaround time, tenant ratings, and cost benchmarks

### Portals & Communication

- Tenant and owner portals (web and mobile) for payments, requests, and document access
- Role-based access controls for staff, owners, and vendors
- Centralised communication threads spanning SMS, email, and portal messages

### AI-Driven Intelligence

- Predictive delinquency alerting 30–60 days before a missed payment
- Dynamic rent pricing recommendations informed by local vacancy and seasonal demand
- Lease renewal optimisation with personalised retention offers based on tenure and payment history
- Plain-language summaries of local landlord-tenant law changes translated into action items

---

## AI-Native Advantage

Incumbents are only beginning to ship AI features (AppFolio's leasing assistant, DoorLoop's AI inspections, Rentvine's voice assistant), and these capabilities sit behind proprietary paywalls. An open AI-native platform can ship proactive delinquency prediction, automated maintenance triage, dynamic rent pricing, lease renewal optimisation, and compliance summarisation as first-class features rather than upsells. Receipt OCR and smart expense categorisation map transactions to Schedule E or MTD codes without manual entry.

---

## Tech Stack & Deployment

The project is expected to support both self-hosted and cloud deployment so independent landlords and institutional operators can pick the right operating model. Integration aligns with openly published standards: the RESO Web API for real estate data exchange, NACHA / ACH for US payments, FCRA-compliant tenant screening, ASC 842 / IFRS 16 lease accounting, and HUD Section 8 / HCV compliance for subsidised housing. A public REST API with full CRUD access to core resources is in scope for v1.1, with an MCP server on the backlog so AI assistants can query properties, leases, and maintenance queues directly.

---

## Market Context

The property management software market is estimated at $3.61B (Grand View Research, 2025) on a narrow definition and up to $26.55B on broader definitions, growing at 6.4–10.1% CAGR through 2032–2033. Incumbent pricing spans free tiers (TurboTenant, Landlord Studio), mid-market SaaS at $50–$300/month (Buildium from $62/month, Yardi Breeze at $1/unit/month), and enterprise contracts for Yardi Voyager and AppFolio. Primary buyers are independent landlords with 1–10 units, mid-size firms managing 50–500 units, and large institutional operators requiring compliance reporting and ERP integrations.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
