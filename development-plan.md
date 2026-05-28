# Property Management System — Phased Development Plan

> Project: 196-property-management-system · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js) | Full-stack language sharing types between API and frontend; strong ecosystem for REST APIs, async payment webhooks, and real-time notifications; first-class PostgreSQL support via Drizzle ORM |
| Language (AI services) | Python 3.12 | Required for ML model training/inference (delinquency prediction, rent pricing); scikit-learn, pandas, and LangChain are Python-native; deployed as separate microservice behind internal API |
| API framework | Fastify 5 | 2-3x faster than Express; built-in JSON Schema validation (aligns with JSON Schema draft 2020-12 from standards.md); native OpenAPI 3.1 generation via @fastify/swagger |
| Database | PostgreSQL 16 | JSONB with GIN indexes for jurisdiction-specific data; ltree extension available for ownership hierarchies; Row Level Security for multi-tenant isolation; double-entry accounting requires ACID guarantees; adopted by data-model-suggestion-3 (hybrid relational + JSONB) |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero overhead; generates migrations automatically; supports PostgreSQL JSONB operators and custom types; lighter than Prisma with better raw SQL escape hatches |
| Cache / queue | Redis 7 (via BullMQ) | Task queue for async jobs (payment processing, screening requests, AI inference, notification dispatch); caching for dashboard aggregates and session storage |
| Frontend | Next.js 15 (App Router) | React Server Components reduce client bundle size for data-heavy dashboards; built-in API routes for BFF pattern; Vercel deployment path for SaaS; shadcn/ui for accessible component library |
| Mobile | React Native (Expo) | Shared TypeScript types with backend and web frontend; tenant and owner portal apps for iOS and Android; Expo handles OTA updates for fast iteration |
| Authentication | NextAuth.js 5 + OAuth 2.0 / OpenID Connect | Supports email/password, Google, Microsoft Entra SSO; JWT sessions (RFC 7519); WebAuthn for passwordless login on tenant portal |
| Payment processor | Stripe Connect + Dwolla (ACH) | Stripe handles card payments with PCI DSS scope reduction; Dwolla specializes in ACH (NACHA-compliant) with lower per-transaction fees for rent collection; both provide webhook-driven status updates |
| File storage | AWS S3 (or MinIO for self-hosted) | Lease documents, maintenance photos, inspection reports; pre-signed URLs for secure tenant/owner access; MinIO API-compatible for self-hosted deployments |
| Email / SMS | Resend (email) + Twilio (SMS) | Resend for transactional email (rent reminders, maintenance updates); Twilio for SMS notifications and two-factor authentication |
| Search | PostgreSQL full-text search (pg_trgm) | Sufficient for tenant/property/vendor search at MVP scale; avoids Elasticsearch operational overhead; upgrade path to Meilisearch if needed |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest is fastest for TypeScript with native ESM support; Playwright for cross-browser E2E testing of tenant/owner portals |
| Code quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with 10-100x faster performance; strict TypeScript for type checking |
| Containerisation | Docker + Docker Compose | Multi-container setup (API, web, worker, postgres, redis); self-hosted deployment target |
| CI/CD | GitHub Actions | Lint, type-check, test, build Docker images, deploy to staging on PR merge |
| API documentation | OpenAPI 3.1 (auto-generated) | @fastify/swagger generates spec from route schemas; Scalar for interactive API docs UI |
| Monorepo | pnpm workspaces + Turborepo | Shared TypeScript types between packages; parallel builds; single lockfile |

### Data Model Selection

**Adopted: Data Model Suggestion 3 (Hybrid Relational + JSONB)** with selective elements from Suggestion 1 (normalized accounting tables) and Suggestion 4 (graph edges for ownership).

Rationale:
- ~23 core tables vs. ~34 for full normalization keeps the schema manageable
- JSONB columns handle jurisdiction variability (California rent control vs. Texas commercial) without schema migrations
- Relational columns for all queryable, business-critical fields (rent amounts, dates, statuses)
- Double-entry accounting tables remain fully normalized (from Suggestion 1) because financial data requires strict referential integrity
- Ownership relationships use a simple `property_owners` junction table for MVP; graph layer (Suggestion 4) is a backlog item for institutional operators with complex fund structures
- GIN indexes on JSONB columns enable fast containment queries for compliance features

### Project Structure

```
property-management-system/
├── README.md
├── LICENSE
├── docker-compose.yml
├── docker-compose.prod.yml
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       └── deploy-production.yml
├── packages/
│   ├── shared/                          # Shared types and utilities
│   │   ├── package.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── property.ts
│   │       │   ├── unit.ts
│   │       │   ├── tenant.ts
│   │       │   ├── lease.ts
│   │       │   ├── payment.ts
│   │       │   ├── maintenance.ts
│   │       │   ├── accounting.ts
│   │       │   ├── listing.ts
│   │       │   ├── screening.ts
│   │       │   ├── notification.ts
│   │       │   └── index.ts
│   │       ├── validators/
│   │       │   └── schemas.ts           # Zod schemas matching DB types
│   │       ├── constants/
│   │       │   ├── lease-statuses.ts
│   │       │   ├── payment-statuses.ts
│   │       │   └── maintenance-priorities.ts
│   │       └── utils/
│   │           ├── money.ts             # Currency arithmetic helpers
│   │           └── dates.ts             # Lease date calculations
│   ├── db/                              # Database package
│   │   ├── package.json
│   │   ├── drizzle.config.ts
│   │   └── src/
│   │       ├── schema/
│   │       │   ├── organizations.ts
│   │       │   ├── users.ts
│   │       │   ├── properties.ts
│   │       │   ├── units.ts
│   │       │   ├── tenants.ts
│   │       │   ├── leases.ts
│   │       │   ├── payments.ts
│   │       │   ├── accounting.ts
│   │       │   ├── maintenance.ts
│   │       │   ├── vendors.ts
│   │       │   ├── listings.ts
│   │       │   ├── screening.ts
│   │       │   ├── messages.ts
│   │       │   ├── ai-predictions.ts
│   │       │   ├── audit-log.ts
│   │       │   └── index.ts
│   │       ├── migrations/
│   │       ├── seeds/
│   │       │   ├── demo-data.ts
│   │       │   └── jurisdictions.ts
│   │       └── client.ts                # Drizzle client with RLS
│   └── email-templates/                 # React Email templates
│       ├── package.json
│       └── src/
│           ├── rent-reminder.tsx
│           ├── payment-confirmation.tsx
│           ├── maintenance-update.tsx
│           └── lease-expiry-notice.tsx
├── apps/
│   ├── api/                             # Fastify API server
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── server.ts
│   │       ├── plugins/
│   │       │   ├── auth.ts
│   │       │   ├── multi-tenant.ts
│   │       │   ├── rate-limit.ts
│   │       │   └── audit-log.ts
│   │       ├── routes/
│   │       │   ├── v1/
│   │       │   │   ├── properties.ts
│   │       │   │   ├── units.ts
│   │       │   │   ├── tenants.ts
│   │       │   │   ├── leases.ts
│   │       │   │   ├── payments.ts
│   │       │   │   ├── maintenance.ts
│   │       │   │   ├── vendors.ts
│   │       │   │   ├── listings.ts
│   │       │   │   ├── applications.ts
│   │       │   │   ├── screening.ts
│   │       │   │   ├── accounting.ts
│   │       │   │   ├── owners.ts
│   │       │   │   ├── messages.ts
│   │       │   │   ├── notifications.ts
│   │       │   │   ├── reports.ts
│   │       │   │   └── ai.ts
│   │       │   └── webhooks/
│   │       │       ├── stripe.ts
│   │       │       ├── dwolla.ts
│   │       │       └── screening.ts
│   │       ├── services/
│   │       │   ├── property.service.ts
│   │       │   ├── lease.service.ts
│   │       │   ├── payment.service.ts
│   │       │   ├── maintenance.service.ts
│   │       │   ├── accounting.service.ts
│   │       │   ├── screening.service.ts
│   │       │   ├── listing.service.ts
│   │       │   ├── notification.service.ts
│   │       │   └── report.service.ts
│   │       ├── jobs/
│   │       │   ├── rent-reminder.job.ts
│   │       │   ├── late-fee.job.ts
│   │       │   ├── lease-expiry-check.job.ts
│   │       │   ├── payment-reconciliation.job.ts
│   │       │   └── ai-prediction.job.ts
│   │       └── middleware/
│   │           ├── org-context.ts
│   │           └── permission-check.ts
│   ├── web/                             # Next.js web application
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── app/
│   │       │   ├── (auth)/
│   │       │   │   ├── login/
│   │       │   │   └── register/
│   │       │   ├── (dashboard)/
│   │       │   │   ├── properties/
│   │       │   │   ├── units/
│   │       │   │   ├── tenants/
│   │       │   │   ├── leases/
│   │       │   │   ├── maintenance/
│   │       │   │   ├── accounting/
│   │       │   │   ├── reports/
│   │       │   │   └── settings/
│   │       │   ├── tenant-portal/
│   │       │   └── owner-portal/
│   │       ├── components/
│   │       │   ├── ui/                  # shadcn/ui components
│   │       │   ├── forms/
│   │       │   ├── tables/
│   │       │   ├── charts/
│   │       │   └── layout/
│   │       ├── hooks/
│   │       ├── lib/
│   │       │   ├── api-client.ts
│   │       │   └── auth.ts
│   │       └── styles/
│   ├── worker/                          # BullMQ job processor
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── worker.ts
│   │       └── processors/
│   │           ├── rent-reminder.processor.ts
│   │           ├── late-fee.processor.ts
│   │           ├── payment.processor.ts
│   │           ├── notification.processor.ts
│   │           └── ai-prediction.processor.ts
│   └── ai-service/                      # Python ML microservice
│       ├── pyproject.toml
│       ├── Dockerfile
│       └── src/
│           ├── main.py                  # FastAPI server
│           ├── models/
│           │   ├── delinquency.py
│           │   ├── rent_pricing.py
│           │   └── maintenance_triage.py
│           ├── routes/
│           │   ├── predict.py
│           │   └── health.py
│           └── training/
│               ├── delinquency_trainer.py
│               └── rent_pricing_trainer.py
└── tests/
    ├── api/                             # API integration tests
    ├── web/                             # Playwright E2E tests
    └── fixtures/
        ├── properties.json
        ├── leases.json
        ├── tenants.json
        └── payments.json
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo structure, database schema, authentication, and multi-tenant isolation. After this phase, a developer can start the stack, authenticate, create an organization, and see an empty dashboard. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Monorepo Setup & Toolchain

**What**: Initialize pnpm workspace with Turborepo, configure Biome, TypeScript, and Docker Compose.

**Design**:

Root `package.json`:
```json
{
  "name": "property-management-system",
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "biome check .",
    "format": "biome format --write .",
    "typecheck": "turbo typecheck",
    "test": "turbo test",
    "db:migrate": "pnpm --filter @pms/db migrate",
    "db:seed": "pnpm --filter @pms/db seed"
  },
  "devDependencies": {
    "@biomejs/biome": "^1.9",
    "turbo": "^2.3",
    "typescript": "^5.6"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "typecheck": { "dependsOn": ["^build"] },
    "lint": {}
  }
}
```

Root `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pms_dev
      POSTGRES_USER: pms
      POSTGRES_PASSWORD: pms_dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./apps/api
    depends_on: [postgres, redis]
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgresql://pms:pms_dev_password@postgres:5432/pms_dev
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-secret-change-in-production
    volumes:
      - ./apps/api/src:/app/src

  web:
    build: ./apps/web
    depends_on: [api]
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:3001
    volumes:
      - ./apps/web/src:/app/src

  worker:
    build: ./apps/worker
    depends_on: [postgres, redis]
    environment:
      DATABASE_URL: postgresql://pms:pms_dev_password@postgres:5432/pms_dev
      REDIS_URL: redis://redis:6379

volumes:
  pgdata:
```

**Testing**:
- `Unit: pnpm install succeeds with no peer dependency warnings`
- `Unit: turbo build completes for all packages without errors`
- `Unit: biome check . passes with zero diagnostics`
- `Unit: tsc --noEmit passes for all packages`
- `Integration: docker-compose up starts postgres, redis, api, web, and worker containers`
- `Integration: postgres accepts connections on port 5432`
- `Integration: redis accepts connections on port 6379`

---

#### 1.2 — Database Schema & Migrations

**What**: Implement the hybrid relational + JSONB schema using Drizzle ORM, covering organizations, users, properties, units, tenants, leases, and accounting tables.

**Design**:

Core Drizzle schema types (from data-model-suggestion-3):

```typescript
// packages/db/src/schema/organizations.ts
import { pgTable, uuid, varchar, text, char, boolean, timestamp, jsonb } from "drizzle-orm/pg-core";

export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).unique().notNull(),
  planTier: varchar("plan_tier", { length: 50 }).notNull().default("free"),
  timezone: varchar("timezone", { length: 50 }).notNull().default("America/New_York"),
  defaultCurrency: char("default_currency", { length: 3 }).notNull().default("USD"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/users.ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id),
  email: varchar("email", { length: 255 }).notNull(),
  passwordHash: varchar("password_hash", { length: 255 }),
  firstName: varchar("first_name", { length: 100 }).notNull(),
  lastName: varchar("last_name", { length: 100 }).notNull(),
  phone: varchar("phone", { length: 30 }),
  role: varchar("role", { length: 50 }).notNull().default("viewer"),
  permissions: jsonb("permissions").notNull().default([]),
  preferences: jsonb("preferences").notNull().default({}),
  isActive: boolean("is_active").notNull().default(true),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
// UNIQUE constraint: (organization_id, email)

// packages/db/src/schema/properties.ts
export const properties = pgTable("properties", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id),
  name: varchar("name", { length: 255 }).notNull(),
  propertyType: varchar("property_type", { length: 50 }).notNull(),
  addressLine1: varchar("address_line1", { length: 255 }).notNull(),
  addressLine2: varchar("address_line2", { length: 255 }),
  city: varchar("city", { length: 100 }).notNull(),
  stateProvince: varchar("state_province", { length: 100 }).notNull(),
  postalCode: varchar("postal_code", { length: 20 }).notNull(),
  countryCode: char("country_code", { length: 2 }).notNull().default("US"),
  latitude: numeric("latitude", { precision: 10, scale: 7 }),
  longitude: numeric("longitude", { precision: 10, scale: 7 }),
  yearBuilt: integer("year_built"),
  totalUnits: integer("total_units").notNull().default(1),
  jurisdictionRules: jsonb("jurisdiction_rules").notNull().default({}),
  extendedAttributes: jsonb("extended_attributes").notNull().default({}),
  financialData: jsonb("financial_data").notNull().default({}),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Shared Zod validators (used by both API and frontend):

```typescript
// packages/shared/src/validators/schemas.ts
import { z } from "zod";

export const CreateOrganizationSchema = z.object({
  name: z.string().min(1).max(255),
  slug: z.string().min(3).max(100).regex(/^[a-z0-9-]+$/),
  timezone: z.string().default("America/New_York"),
  defaultCurrency: z.string().length(3).default("USD"),
});

export const CreatePropertySchema = z.object({
  name: z.string().min(1).max(255),
  propertyType: z.enum(["residential", "commercial", "mixed_use", "hoa"]),
  addressLine1: z.string().min(1).max(255),
  addressLine2: z.string().max(255).optional(),
  city: z.string().min(1).max(100),
  stateProvince: z.string().min(1).max(100),
  postalCode: z.string().min(1).max(20),
  countryCode: z.string().length(2).default("US"),
  yearBuilt: z.number().int().min(1800).max(2100).optional(),
  totalUnits: z.number().int().min(1).default(1),
  jurisdictionRules: z.record(z.unknown()).optional(),
  extendedAttributes: z.record(z.unknown()).optional(),
});

export const CreateUnitSchema = z.object({
  propertyId: z.string().uuid(),
  unitNumber: z.string().min(1).max(50),
  unitType: z.enum(["apartment", "townhouse", "office", "retail", "storage"]).optional(),
  bedrooms: z.number().int().min(0).optional(),
  bathroomsFull: z.number().int().min(0).optional(),
  bathroomsHalf: z.number().int().min(0).optional(),
  livingAreaSqft: z.number().positive().optional(),
  floorNumber: z.number().int().optional(),
  marketRent: z.number().positive().optional(),
  attributes: z.record(z.unknown()).optional(),
});

export const CreateLeaseSchema = z.object({
  unitId: z.string().uuid(),
  leaseType: z.enum(["fixed_term", "month_to_month", "commercial_net", "commercial_gross"]),
  startDate: z.string().date(),
  endDate: z.string().date().optional(),
  monthlyRent: z.number().positive(),
  securityDeposit: z.number().min(0).optional(),
  rentDueDay: z.number().int().min(1).max(28).default(1),
  leaseTermMonths: z.number().int().min(1),
  tenantIds: z.array(z.string().uuid()).min(1),
  primaryTenantId: z.string().uuid(),
  terms: z.record(z.unknown()).optional(),
  accountingData: z.record(z.unknown()).optional(),
  subsidyData: z.record(z.unknown()).optional(),
});
```

Row Level Security policies (applied via migration):

```sql
-- Enable RLS on all organization-scoped tables
ALTER TABLE properties ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation_properties ON properties
  USING (organization_id = current_setting('app.current_org_id')::uuid);
-- Repeat for: units, tenants, leases, payments, maintenance_requests, etc.
```

Seed data script for development:

```typescript
// packages/db/src/seeds/demo-data.ts
export async function seedDemoData(db: DrizzleClient) {
  const org = await db.insert(organizations).values({
    name: "Sunrise Property Management",
    slug: "sunrise-pm",
    planTier: "professional",
  }).returning();

  const admin = await db.insert(users).values({
    organizationId: org[0].id,
    email: "admin@sunrise-pm.com",
    passwordHash: await hash("demo-password"),
    firstName: "Sarah",
    lastName: "Chen",
    role: "admin",
    permissions: ["properties.read", "properties.write", "leases.read", "leases.write",
                   "tenants.read", "tenants.write", "accounting.read", "accounting.write"],
  }).returning();
  // ... additional seed properties, units, tenants, leases
}
```

**Testing**:
- `Unit: drizzle-kit generate produces migration SQL without errors`
- `Unit: drizzle-kit push applies schema to fresh database`
- `Integration: seed script populates demo organization with 3 properties, 12 units, 8 tenants, 8 active leases`
- `Integration: RLS policy blocks queries without app.current_org_id set`
- `Integration: RLS policy allows queries with matching org_id`
- `Integration: UNIQUE constraint on (organization_id, email) prevents duplicate user emails within org`
- `Integration: UNIQUE constraint on (property_id, unit_number) prevents duplicate units`
- `Unit: Zod schemas reject invalid input (missing required fields, wrong types, out-of-range values)`
- `Unit: Zod schemas accept valid input and apply defaults`

---

#### 1.3 — Authentication & Authorization

**What**: Implement email/password authentication with JWT sessions, role-based access control, and organization-scoped middleware.

**Design**:

Auth plugin for Fastify:

```typescript
// apps/api/src/plugins/auth.ts
import fp from "fastify-plugin";
import jwt from "@fastify/jwt";

export interface JWTPayload {
  userId: string;
  organizationId: string;
  email: string;
  role: string;
  permissions: string[];
}

export default fp(async function authPlugin(fastify) {
  await fastify.register(jwt, {
    secret: process.env.JWT_SECRET!,
    sign: { expiresIn: "24h" },
  });

  fastify.decorate("authenticate", async (request, reply) => {
    try {
      await request.jwtVerify();
    } catch (err) {
      reply.status(401).send({ error: "Unauthorized" });
    }
  });
});
```

Multi-tenant middleware:

```typescript
// apps/api/src/plugins/multi-tenant.ts
export default fp(async function multiTenantPlugin(fastify) {
  fastify.addHook("preHandler", async (request) => {
    if (request.user?.organizationId) {
      // Set PostgreSQL session variable for RLS
      await request.server.db.execute(
        sql`SELECT set_config('app.current_org_id', ${request.user.organizationId}, true)`
      );
    }
  });
});
```

Permission check middleware:

```typescript
// apps/api/src/middleware/permission-check.ts
export function requirePermission(permission: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.user as JWTPayload;
    if (!user.permissions.includes(permission) && user.role !== "admin") {
      reply.status(403).send({ error: "Forbidden", required: permission });
    }
  };
}
```

Auth routes:

```typescript
// POST /api/v1/auth/register
// Request: { email, password, firstName, lastName, organizationName, organizationSlug }
// Response: { user: User, organization: Organization, token: string }

// POST /api/v1/auth/login
// Request: { email, password }
// Response: { user: User, token: string }

// POST /api/v1/auth/refresh
// Request: (Authorization header with valid token)
// Response: { token: string }

// GET /api/v1/auth/me
// Request: (Authorization header)
// Response: { user: User, organization: Organization }
```

**Testing**:
- `Unit: JWT token generation includes userId, organizationId, email, role, permissions`
- `Unit: JWT token verification rejects expired tokens`
- `Unit: JWT token verification rejects tampered tokens`
- `Unit: password hashing uses bcrypt with cost factor 12`
- `Unit: requirePermission("properties.write") blocks user without that permission`
- `Unit: requirePermission("properties.write") allows user with admin role regardless of permissions array`
- `Integration: POST /auth/register creates organization and user, returns valid JWT`
- `Integration: POST /auth/register with duplicate email returns 409 Conflict`
- `Integration: POST /auth/login with valid credentials returns JWT`
- `Integration: POST /auth/login with wrong password returns 401`
- `Integration: GET /auth/me with valid token returns user and organization`
- `Integration: API request without Authorization header returns 401`
- `Integration: multi-tenant middleware sets app.current_org_id on PostgreSQL session`

---

#### 1.4 — Property & Unit CRUD API

**What**: Implement full CRUD endpoints for properties and units with JSON Schema validation, pagination, filtering, and organization isolation.

**Design**:

```typescript
// apps/api/src/routes/v1/properties.ts
// All endpoints require authentication + org context

// GET /api/v1/properties
// Query: { page?: number, limit?: number, type?: string, city?: string, active?: boolean }
// Response: { data: Property[], meta: { total, page, limit, totalPages } }

// GET /api/v1/properties/:id
// Response: { data: Property & { units: Unit[] } }

// POST /api/v1/properties
// Body: CreatePropertySchema
// Response: { data: Property } (201 Created)

// PATCH /api/v1/properties/:id
// Body: Partial<CreatePropertySchema>
// Response: { data: Property }

// DELETE /api/v1/properties/:id
// Response: 204 No Content (soft delete: sets is_active = false)

// GET /api/v1/properties/:id/units
// Response: { data: Unit[] }

// POST /api/v1/properties/:id/units
// Body: CreateUnitSchema (propertyId from URL param)
// Response: { data: Unit } (201 Created)

// PATCH /api/v1/units/:id
// Body: Partial<CreateUnitSchema>
// Response: { data: Unit }
```

Service layer:

```typescript
// apps/api/src/services/property.service.ts
export class PropertyService {
  constructor(private db: DrizzleClient) {}

  async list(orgId: string, filters: PropertyFilters): Promise<PaginatedResult<Property>> {
    const query = this.db.select().from(properties)
      .where(and(
        eq(properties.organizationId, orgId),
        filters.type ? eq(properties.propertyType, filters.type) : undefined,
        filters.city ? ilike(properties.city, `%${filters.city}%`) : undefined,
        filters.active !== undefined ? eq(properties.isActive, filters.active) : undefined,
      ))
      .limit(filters.limit)
      .offset((filters.page - 1) * filters.limit)
      .orderBy(desc(properties.createdAt));
    // ...
  }

  async getById(orgId: string, propertyId: string): Promise<Property & { units: Unit[] }> { ... }
  async create(orgId: string, data: CreatePropertyInput): Promise<Property> { ... }
  async update(orgId: string, propertyId: string, data: Partial<CreatePropertyInput>): Promise<Property> { ... }
  async softDelete(orgId: string, propertyId: string): Promise<void> { ... }
}
```

**Testing**:
- `Unit: PropertyService.list returns paginated results with correct meta`
- `Unit: PropertyService.list with type filter returns only matching property types`
- `Unit: PropertyService.create validates required fields and inserts property`
- `Unit: PropertyService.softDelete sets is_active = false, does not remove record`
- `Integration: GET /api/v1/properties returns only properties belonging to the authenticated org`
- `Integration: POST /api/v1/properties with valid body creates property and returns 201`
- `Integration: POST /api/v1/properties with missing address_line1 returns 400 with validation error`
- `Integration: GET /api/v1/properties/:id/units returns units for that property only`
- `Integration: POST /api/v1/properties/:propertyId/units with duplicate unit_number returns 409`
- `Integration: PATCH /api/v1/units/:id with jurisdiction_rules JSONB updates correctly`
- `Integration: user from org-A cannot access properties from org-B (RLS enforced)`

---

#### 1.5 — Web Dashboard Shell

**What**: Scaffold the Next.js application with authentication flow, sidebar navigation, and empty dashboard pages for each module.

**Design**:

Layout structure:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1 overflow-y-auto p-6">
        <Header />
        {children}
      </main>
    </div>
  );
}
```

Sidebar navigation items:

```typescript
const navItems = [
  { label: "Dashboard", href: "/", icon: LayoutDashboard },
  { label: "Properties", href: "/properties", icon: Building2 },
  { label: "Units", href: "/units", icon: DoorOpen },
  { label: "Tenants", href: "/tenants", icon: Users },
  { label: "Leases", href: "/leases", icon: FileText },
  { label: "Maintenance", href: "/maintenance", icon: Wrench },
  { label: "Accounting", href: "/accounting", icon: Calculator },
  { label: "Reports", href: "/reports", icon: BarChart3 },
  { label: "Settings", href: "/settings", icon: Settings },
];
```

API client:

```typescript
// apps/web/src/lib/api-client.ts
export class APIClient {
  private baseUrl: string;
  private token: string | null;

  async get<T>(path: string, params?: Record<string, string>): Promise<T> { ... }
  async post<T>(path: string, body: unknown): Promise<T> { ... }
  async patch<T>(path: string, body: unknown): Promise<T> { ... }
  async delete(path: string): Promise<void> { ... }
}
```

**Testing**:
- `E2E: /login page renders email and password fields`
- `E2E: login with valid credentials redirects to dashboard`
- `E2E: login with invalid credentials shows error message`
- `E2E: dashboard sidebar shows all navigation items`
- `E2E: clicking Properties navigates to /properties`
- `E2E: unauthenticated user accessing /properties redirects to /login`
- `Unit: APIClient.get appends query parameters correctly`
- `Unit: APIClient includes Authorization header when token is set`

---

## Phase 2: Tenant & Lease Management

### Purpose
Implement the tenant lifecycle: create tenants, manage leases with multi-tenant support, track lease status transitions, and support lease documents with eSignature placeholders. After this phase, a property manager can onboard tenants, create and sign leases, and track lease expiry.

### Tasks

#### 2.1 — Tenant CRUD & Portal User Invitation

**What**: Full tenant management with the ability to invite tenants as portal users.

**Design**:

```typescript
// packages/shared/src/types/tenant.ts
export interface Tenant {
  id: string;
  organizationId: string;
  firstName: string;
  lastName: string;
  email: string | null;
  phone: string | null;
  profile: TenantProfile;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface TenantProfile {
  dateOfBirth?: string;
  ssnLastFour?: string;
  emergencyContact?: {
    name: string;
    phone: string;
    relationship?: string;
  };
  vehicles?: Array<{
    make: string;
    model: string;
    year: number;
    plate: string;
    state: string;
  }>;
  pets?: Array<{
    type: string;
    breed?: string;
    weightLbs?: number;
    name?: string;
  }>;
  employer?: {
    name: string;
    phone?: string;
    monthlyIncome?: number;
  };
}

// API Routes:
// GET    /api/v1/tenants            — list tenants (paginated, filterable by name, email, active)
// GET    /api/v1/tenants/:id        — get tenant with lease history
// POST   /api/v1/tenants            — create tenant
// PATCH  /api/v1/tenants/:id        — update tenant
// DELETE /api/v1/tenants/:id        — soft delete
// POST   /api/v1/tenants/:id/invite — send portal invitation email
```

**Testing**:
- `Unit: CreateTenantSchema rejects missing firstName`
- `Unit: CreateTenantSchema accepts profile with nested vehicles array`
- `Integration: POST /tenants creates tenant scoped to current org`
- `Integration: GET /tenants filters by last name case-insensitively`
- `Integration: GET /tenants/:id includes lease history with unit and property names`
- `Integration: POST /tenants/:id/invite sends email and creates pending portal user record`
- `Integration: tenant from org-A is invisible to org-B`

---

#### 2.2 — Lease Creation & Status Management

**What**: Lease CRUD with state machine (draft -> active -> expired/terminated/renewed), multi-tenant assignment, and recurring charge support.

**Design**:

Lease status transitions (state machine):

```
draft ──── sign ────> active
                        │
                        ├── expire (end_date passed) ──> expired
                        ├── terminate (early) ──────────> terminated
                        └── renew ─────────────────────> renewed (new lease created as active)
```

```typescript
// packages/shared/src/types/lease.ts
export interface Lease {
  id: string;
  unitId: string;
  leaseStatus: LeaseStatus;
  leaseType: LeaseType;
  startDate: string;
  endDate: string | null;
  monthlyRent: number;
  securityDeposit: number | null;
  rentDueDay: number;
  leaseTermMonths: number;
  terms: LeaseTerms;
  accountingData: LeaseAccountingData;
  subsidyData: LeaseSubsidyData;
  signedAt: string | null;
  createdAt: string;
  updatedAt: string;
  tenants: LeaseTenant[];
}

export type LeaseStatus = "draft" | "active" | "expired" | "terminated" | "renewed";
export type LeaseType = "fixed_term" | "month_to_month" | "commercial_net" | "commercial_gross";

export interface LeaseTerms {
  lateFeeAmount?: number;
  lateFeeGraceDays?: number;
  autoRenew?: boolean;
  renewalTermMonths?: number;
  utilitiesTenantPays?: string[];
  parkingIncluded?: boolean;
  petDeposit?: number;
  recurringCharges?: Array<{
    type: string;
    amount: number;
    frequency: "monthly" | "quarterly" | "annually" | "one_time";
  }>;
}

// API Routes:
// GET    /api/v1/leases                 — list leases (filter by status, unit, property, tenant)
// GET    /api/v1/leases/:id             — get lease with tenants, unit, property
// POST   /api/v1/leases                 — create lease (status = draft)
// PATCH  /api/v1/leases/:id             — update lease (only if draft)
// POST   /api/v1/leases/:id/sign        — transition draft -> active, set signedAt
// POST   /api/v1/leases/:id/terminate   — transition active -> terminated
// POST   /api/v1/leases/:id/renew       — create new lease, mark current as renewed
// GET    /api/v1/leases/expiring        — leases expiring in N days (query param: days=30)
```

Lease service methods:

```typescript
// apps/api/src/services/lease.service.ts
export class LeaseService {
  async create(orgId: string, data: CreateLeaseInput): Promise<Lease> {
    return this.db.transaction(async (tx) => {
      const lease = await tx.insert(leases).values({
        unitId: data.unitId,
        leaseStatus: "draft",
        leaseType: data.leaseType,
        startDate: data.startDate,
        endDate: data.endDate,
        monthlyRent: data.monthlyRent,
        securityDeposit: data.securityDeposit,
        rentDueDay: data.rentDueDay,
        leaseTermMonths: data.leaseTermMonths,
        terms: data.terms ?? {},
        accountingData: data.accountingData ?? {},
        subsidyData: data.subsidyData ?? {},
      }).returning();

      // Link tenants
      for (const tenantId of data.tenantIds) {
        await tx.insert(leaseTenants).values({
          leaseId: lease[0].id,
          tenantId,
          isPrimary: tenantId === data.primaryTenantId,
          moveInDate: data.startDate,
        });
      }

      // Update unit status
      await tx.update(units)
        .set({ status: "occupied" })
        .where(eq(units.id, data.unitId));

      return lease[0];
    });
  }

  async sign(orgId: string, leaseId: string): Promise<Lease> {
    // Validate current status is "draft"
    // Transition to "active", set signedAt = now()
  }

  async terminate(orgId: string, leaseId: string, moveOutDate: string): Promise<Lease> {
    // Validate current status is "active"
    // Transition to "terminated"
    // Update unit status to "vacant"
    // Record move-out date on lease_tenants
  }

  async renew(orgId: string, leaseId: string, renewalData: RenewalInput): Promise<Lease> {
    // Validate current status is "active"
    // Mark current lease as "renewed"
    // Create new lease linked to same unit and tenants
    // Return new lease
  }
}
```

**Testing**:
- `Unit: lease creation with invalid status transition throws InvalidStatusTransitionError`
- `Unit: lease sign sets signedAt and transitions status to active`
- `Unit: lease terminate updates unit status to vacant`
- `Unit: lease renew creates new active lease and marks old as renewed`
- `Integration: POST /leases creates draft lease with tenant associations`
- `Integration: POST /leases/:id/sign transitions draft to active`
- `Integration: POST /leases/:id/sign on already-active lease returns 409`
- `Integration: POST /leases/:id/terminate on draft lease returns 409`
- `Integration: POST /leases/:id/renew creates new lease with updated rent`
- `Integration: GET /leases/expiring?days=30 returns leases ending within 30 days`
- `Integration: creating a lease for an already-occupied unit returns 409`
- `Unit: lease terms JSONB stores recurring charges with correct structure`

---

#### 2.3 — Lease Document Management

**What**: Upload, manage, and track lease documents (agreements, addenda, disclosures) with S3 storage and eSignature status tracking.

**Design**:

```typescript
// packages/shared/src/types/lease.ts
export interface LeaseDocument {
  id: string;
  leaseId: string;
  documentType: "lease_agreement" | "addendum" | "disclosure" | "notice";
  fileName: string;
  fileUrl: string;
  fileSizeBytes: number;
  mimeType: string;
  esignatureStatus: "pending" | "signed" | "declined" | "expired" | null;
  esignatureCompletedAt: string | null;
  uploadedBy: string;
  createdAt: string;
}

// API Routes:
// GET    /api/v1/leases/:id/documents         — list documents for a lease
// POST   /api/v1/leases/:id/documents         — upload document (multipart form)
// GET    /api/v1/leases/:id/documents/:docId   — get pre-signed download URL
// DELETE /api/v1/leases/:id/documents/:docId   — delete document
// POST   /api/v1/leases/:id/documents/:docId/sign — mark as signed (placeholder for eSign integration)
```

S3 upload service:

```typescript
// apps/api/src/services/file-storage.service.ts
export class FileStorageService {
  private s3: S3Client;

  async uploadDocument(
    orgId: string,
    leaseId: string,
    file: MultipartFile,
  ): Promise<{ key: string; url: string }> {
    const key = `orgs/${orgId}/leases/${leaseId}/documents/${ulid()}-${file.filename}`;
    await this.s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: file.file,
      ContentType: file.mimetype,
    }));
    return { key, url: `s3://${process.env.S3_BUCKET}/${key}` };
  }

  async getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
    return getSignedUrl(this.s3, new GetObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: key,
    }), { expiresIn });
  }
}
```

**Testing**:
- `Integration: POST /leases/:id/documents uploads file to S3 and creates document record`
- `Integration: POST /leases/:id/documents rejects files over 25MB`
- `Integration: POST /leases/:id/documents rejects non-PDF/image MIME types`
- `Integration: GET /leases/:id/documents/:docId returns pre-signed URL valid for 1 hour`
- `Integration: DELETE /leases/:id/documents/:docId removes S3 object and database record`
- `Unit: FileStorageService generates correct S3 key path with org and lease IDs`
- `Integration (mocked S3): upload with S3 error returns 500 and does not create DB record`

---

#### 2.4 — Lease Management UI

**What**: Web dashboard pages for tenant and lease management including list views, detail pages, create/edit forms, and lease signing workflow.

**Design**:

Pages:
- `/tenants` — DataTable with search, filter by active/inactive, pagination
- `/tenants/new` — Create tenant form with profile section (emergency contact, pets, vehicles)
- `/tenants/[id]` — Tenant detail with tabs: Overview, Leases, Payments, Documents
- `/leases` — DataTable filterable by status, property, expiring-soon badge
- `/leases/new` — Multi-step form: select unit -> add tenants -> set terms -> review
- `/leases/[id]` — Lease detail with status badge, tenant list, documents, action buttons (Sign, Terminate, Renew)

```typescript
// apps/web/src/app/(dashboard)/leases/new/page.tsx
// Multi-step lease creation form
type LeaseFormStep = "select-unit" | "add-tenants" | "set-terms" | "review";

interface LeaseFormState {
  step: LeaseFormStep;
  unitId: string | null;
  tenantIds: string[];
  primaryTenantId: string | null;
  leaseType: LeaseType;
  startDate: string;
  endDate: string;
  monthlyRent: number;
  securityDeposit: number;
  terms: LeaseTerms;
}
```

**Testing**:
- `E2E: navigate to /tenants, see list of tenants from seed data`
- `E2E: create new tenant with emergency contact, verify appears in list`
- `E2E: navigate to /leases/new, complete multi-step form, verify draft lease created`
- `E2E: on lease detail page, click Sign button, verify status changes to Active`
- `E2E: on lease detail page for active lease, click Terminate, confirm dialog, verify status changes`
- `E2E: lease list shows "Expiring Soon" badge for leases ending within 30 days`
- `E2E: upload document to lease, verify it appears in documents tab`

---

## Phase 3: Rent Collection & Payment Processing

### Purpose
Implement online rent collection via ACH (Dwolla) and card (Stripe), automatic charge generation, payment tracking, and payment status webhooks. After this phase, tenants can pay rent online and property managers can track payment status and history.

### Tasks

#### 3.1 — Payment Processor Integration

**What**: Integrate Stripe (card payments) and Dwolla (ACH) with webhook handlers for payment status updates.

**Design**:

```typescript
// packages/shared/src/types/payment.ts
export interface Payment {
  id: string;
  organizationId: string;
  leaseId: string | null;
  tenantId: string | null;
  paymentType: "rent" | "security_deposit" | "late_fee" | "utility" | "refund";
  paymentMethod: "ach" | "credit_card" | "debit_card" | "check" | "cash";
  amount: number;
  currency: string;
  status: PaymentStatus;
  paymentDate: string;
  postedDate: string | null;
  processorData: ProcessorData;
  journalEntryId: string | null;
  notes: string | null;
  createdAt: string;
  updatedAt: string;
}

export type PaymentStatus = "pending" | "processing" | "completed" | "failed" | "refunded";

export interface ProcessorData {
  processor: "stripe" | "dwolla";
  referenceId: string;
  achSecCode?: string;    // WEB, TEL, PPD, CCD (NACHA compliance)
  bankName?: string;
  lastFour?: string;
  failureReason?: string | null;
  feeAmount?: number;
}

// API Routes:
// POST   /api/v1/payments                  — initiate payment
// GET    /api/v1/payments                  — list payments (filter by tenant, lease, status, date range)
// GET    /api/v1/payments/:id              — get payment detail
// POST   /api/v1/payments/:id/refund       — initiate refund
// POST   /api/webhooks/stripe              — Stripe webhook handler
// POST   /api/webhooks/dwolla              — Dwolla webhook handler
```

Payment service:

```typescript
// apps/api/src/services/payment.service.ts
export class PaymentService {
  constructor(
    private db: DrizzleClient,
    private stripe: Stripe,
    private dwolla: DwollaClient,
    private accountingService: AccountingService,
  ) {}

  async initiateACHPayment(input: InitiatePaymentInput): Promise<Payment> {
    // 1. Create payment record (status: pending)
    // 2. Call Dwolla API to initiate transfer
    // 3. Store processor reference ID
    // 4. Return payment with status "processing"
  }

  async handleStripeWebhook(event: Stripe.Event): Promise<void> {
    switch (event.type) {
      case "payment_intent.succeeded":
        // Update payment status to "completed"
        // Create journal entry via AccountingService
        break;
      case "payment_intent.payment_failed":
        // Update payment status to "failed"
        // Store failure reason in processorData
        break;
    }
  }

  async handleDwollaWebhook(event: DwollaWebhookEvent): Promise<void> {
    switch (event.topic) {
      case "customer_transfer_completed":
        // Update payment status to "completed", set postedDate
        // Create journal entry
        break;
      case "customer_transfer_failed":
        // Update payment status to "failed"
        break;
    }
  }
}
```

Webhook signature verification:

```typescript
// apps/api/src/routes/webhooks/stripe.ts
fastify.post("/webhooks/stripe", {
  config: { rawBody: true }, // Need raw body for signature verification
}, async (request, reply) => {
  const sig = request.headers["stripe-signature"];
  const event = stripe.webhooks.constructEvent(
    request.rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET!
  );
  await paymentService.handleStripeWebhook(event);
  reply.send({ received: true });
});
```

**Testing**:
- `Unit: PaymentService.initiateACHPayment creates payment record with status "processing"`
- `Unit: PaymentService.handleStripeWebhook with payment_intent.succeeded updates status to "completed"`
- `Unit: PaymentService.handleStripeWebhook with payment_intent.payment_failed updates status to "failed" and stores reason`
- `Integration (mocked Stripe): POST /payments with credit_card method creates Stripe PaymentIntent`
- `Integration (mocked Dwolla): POST /payments with ach method initiates Dwolla transfer with WEB SEC code`
- `Integration: POST /webhooks/stripe with invalid signature returns 400`
- `Integration: POST /webhooks/stripe with valid signature and payment_intent.succeeded updates payment status`
- `Integration (mocked Dwolla): POST /webhooks/dwolla with customer_transfer_completed creates journal entry`
- `Unit: refund creates Stripe refund and updates payment status to "refunded"`

---

#### 3.2 — Automated Rent Charges & Late Fees

**What**: Background jobs to generate monthly rent charges, send payment reminders, and apply late fees based on lease terms and jurisdiction rules.

**Design**:

```typescript
// apps/worker/src/processors/rent-reminder.processor.ts
export async function processRentReminders(job: Job) {
  // Query all active leases where rent_due_day = today + reminder_days_before
  // For each lease:
  //   1. Calculate total amount due (base rent + recurring charges from lease.terms)
  //   2. Send reminder notification (email + in-app)
  //   3. Log notification in messages table
}

// apps/worker/src/processors/late-fee.processor.ts
export async function processLateFees(job: Job) {
  // Query all active leases where:
  //   rent_due_day + late_fee_grace_days < today
  //   AND no completed payment for current period exists
  // For each lease:
  //   1. Check jurisdiction_rules.late_fee_max_pct (cap the fee)
  //   2. Create late fee payment record (type: "late_fee", status: "pending")
  //   3. Send late fee notification to tenant
  //   4. Create journal entry for late fee accrual
}
```

BullMQ job scheduling:

```typescript
// apps/api/src/jobs/scheduler.ts
// Rent reminders: daily at 8 AM in org's timezone
await rentReminderQueue.add("daily-check", {}, {
  repeat: { pattern: "0 8 * * *" },
});

// Late fee assessment: daily at midnight
await lateFeeQueue.add("daily-check", {}, {
  repeat: { pattern: "0 0 * * *" },
});

// Lease expiry check: daily at 9 AM
await leaseExpiryQueue.add("daily-check", {}, {
  repeat: { pattern: "0 9 * * *" },
});
```

**Testing**:
- `Unit: rent reminder job sends notifications for leases with rent due in 3 days`
- `Unit: rent reminder job calculates total including recurring charges (pet_rent, parking)`
- `Unit: late fee job applies late fee after grace period expires`
- `Unit: late fee job respects jurisdiction max late fee percentage cap`
- `Unit: late fee job does not apply fee if payment already completed for current period`
- `Integration: late fee job creates journal entry with correct debit/credit`
- `Unit: lease expiry job flags leases expiring in 30 and 60 days`
- `Integration: BullMQ jobs are scheduled with correct cron patterns`

---

#### 3.3 — Tenant Payment Portal

**What**: Tenant-facing portal for making payments, viewing payment history, setting up autopay, and downloading receipts.

**Design**:

```typescript
// Tenant portal pages:
// /tenant-portal/payments          — current balance, pay now button, payment history
// /tenant-portal/payments/pay      — payment form (select method: ACH or card)
// /tenant-portal/payments/autopay  — setup/manage autopay schedule
// /tenant-portal/payments/history  — full payment history with downloadable receipts

// Tenant portal API (separate auth scope):
// GET    /api/v1/tenant-portal/balance       — current balance (rent + charges - payments)
// GET    /api/v1/tenant-portal/payments      — payment history for authenticated tenant
// POST   /api/v1/tenant-portal/payments      — submit payment
// POST   /api/v1/tenant-portal/autopay       — enable autopay
// DELETE /api/v1/tenant-portal/autopay       — disable autopay
// GET    /api/v1/tenant-portal/receipts/:id  — download payment receipt PDF
```

Balance calculation:

```typescript
export async function calculateBalance(tenantId: string, leaseId: string): Promise<TenantBalance> {
  // Sum all charges (rent + recurring + late fees) from lease start to now
  // Subtract all completed payments
  // Return: { totalCharges, totalPayments, balance, currentMonthDue, dueDate }
}
```

**Testing**:
- `E2E: tenant logs into portal, sees current balance and due date`
- `E2E: tenant clicks Pay Now, selects ACH, enters bank details, submits payment`
- `E2E: tenant views payment history with status badges (completed, processing, failed)`
- `E2E: tenant enables autopay, sees confirmation message`
- `Integration: tenant portal balance calculation includes base rent + recurring charges - completed payments`
- `Integration: tenant cannot access another tenant's payment history`
- `Unit: balance calculation handles partial payments correctly`

---

## Phase 4: Maintenance & Work Order Management

### Purpose
Implement the maintenance request lifecycle: tenant submission, staff triage, vendor assignment, work order tracking, and completion. After this phase, tenants can submit maintenance requests with photos, and property managers can manage the full repair workflow.

### Tasks

#### 4.1 — Maintenance Request CRUD

**What**: Tenant-submitted maintenance requests with photo uploads, category classification, priority assignment, and status tracking.

**Design**:

```typescript
// packages/shared/src/types/maintenance.ts
export interface MaintenanceRequest {
  id: string;
  unitId: string;
  reportedByTenantId: string | null;
  reportedByUserId: string | null;
  category: MaintenanceCategory;
  priority: MaintenancePriority;
  title: string;
  description: string;
  status: MaintenanceStatus;
  permissionToEnter: boolean;
  aiTriage: AITriageResult | null;
  photos: MaintenancePhoto[];
  completedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export type MaintenanceCategory = "plumbing" | "electrical" | "hvac" | "appliance" | "structural" | "pest" | "general";
export type MaintenancePriority = "emergency" | "urgent" | "normal" | "low";
export type MaintenanceStatus = "open" | "assigned" | "in_progress" | "completed" | "cancelled";

export interface MaintenancePhoto {
  url: string;
  filename: string;
  uploadedBy: "tenant" | "staff" | "vendor";
  uploadedAt: string;
}

// API Routes:
// GET    /api/v1/maintenance                    — list requests (filter by status, priority, property, unit)
// GET    /api/v1/maintenance/:id                — get request with photos and work orders
// POST   /api/v1/maintenance                    — create request (staff)
// PATCH  /api/v1/maintenance/:id                — update request
// POST   /api/v1/maintenance/:id/photos         — upload photos
// POST   /api/v1/tenant-portal/maintenance      — create request (tenant)
// GET    /api/v1/tenant-portal/maintenance      — list tenant's requests
```

**Testing**:
- `Integration: POST /maintenance creates request with status "open"`
- `Integration: POST /maintenance/:id/photos uploads to S3 and appends to photos JSONB array`
- `Integration: GET /maintenance with priority=emergency returns only emergency requests`
- `Integration: tenant portal POST creates request linked to tenant's current unit`
- `Integration: tenant portal GET returns only requests for the authenticated tenant's units`
- `Unit: maintenance request rejects invalid category`
- `Unit: photo upload rejects non-image MIME types`

---

#### 4.2 — Vendor Management

**What**: Vendor registry with specialty tracking, rating aggregation, and service area management.

**Design**:

```typescript
// packages/shared/src/types/vendor.ts
export interface Vendor {
  id: string;
  organizationId: string;
  companyName: string;
  contactName: string | null;
  email: string | null;
  phone: string | null;
  specialties: string[];
  hourlyRate: number | null;
  details: VendorDetails;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface VendorDetails {
  insuranceExpiry?: string;
  licenseNumber?: string;
  licenseState?: string;
  avgRating?: number;
  totalJobs?: number;
  avgResponseHours?: number;
  avgCompletionDays?: number;
  serviceAreaZipCodes?: string[];
  preferredContactMethod?: "email" | "phone" | "sms";
  w9OnFile?: boolean;
}

// API Routes:
// GET    /api/v1/vendors                    — list vendors (filter by specialty, active, rating)
// GET    /api/v1/vendors/:id                — get vendor with job history
// POST   /api/v1/vendors                    — create vendor
// PATCH  /api/v1/vendors/:id                — update vendor
// GET    /api/v1/vendors/match              — find best vendor for a job (query: specialty, zipCode)
```

Vendor matching service:

```typescript
export class VendorService {
  async findBestMatch(orgId: string, specialty: string, propertyZip: string): Promise<Vendor[]> {
    // Query vendors with matching specialty
    // Filter by service area (if serviceAreaZipCodes includes propertyZip)
    // Sort by: avg_rating DESC, avg_response_hours ASC
    // Return top 5
  }
}
```

**Testing**:
- `Integration: POST /vendors creates vendor with specialties array`
- `Integration: GET /vendors/match?specialty=plumbing&zipCode=90001 returns matching vendors sorted by rating`
- `Integration: vendor rating updates when work order is rated by tenant`
- `Unit: VendorService.findBestMatch filters by service area zip codes`
- `Unit: VendorService.findBestMatch returns empty array when no vendors match`

---

#### 4.3 — Work Order Lifecycle

**What**: Work orders assigned from maintenance requests to vendors, with scheduling, cost tracking, and tenant feedback.

**Design**:

Work order status transitions:

```
pending ──> accepted ──> scheduled ──> in_progress ──> completed
    │           │                                         │
    └── cancelled  └── cancelled                      (tenant rates)
```

```typescript
// packages/shared/src/types/maintenance.ts
export interface WorkOrder {
  id: string;
  maintenanceRequestId: string;
  vendorId: string | null;
  assignedBy: string;
  status: WorkOrderStatus;
  scheduledDate: string | null;
  details: WorkOrderDetails;
  createdAt: string;
  updatedAt: string;
}

export interface WorkOrderDetails {
  scheduledTimeStart?: string;
  scheduledTimeEnd?: string;
  actualStart?: string;
  actualEnd?: string;
  laborCost?: number;
  materialsCost?: number;
  materials?: Array<{ item: string; qty: number; cost: number }>;
  totalCost?: number;
  vendorNotes?: string;
  tenantRating?: number;
  tenantFeedback?: string;
}

// API Routes:
// POST   /api/v1/maintenance/:id/work-orders        — create work order (assign vendor)
// GET    /api/v1/work-orders/:id                     — get work order detail
// PATCH  /api/v1/work-orders/:id                     — update status, costs, schedule
// POST   /api/v1/work-orders/:id/complete            — mark complete with costs
// POST   /api/v1/tenant-portal/work-orders/:id/rate  — tenant rates completed work
```

**Testing**:
- `Integration: POST /maintenance/:id/work-orders creates work order and updates request status to "assigned"`
- `Integration: PATCH /work-orders/:id with status "scheduled" stores scheduled date`
- `Integration: POST /work-orders/:id/complete updates maintenance request to "completed"`
- `Integration: tenant rating updates vendor avg_rating in vendor.details JSONB`
- `Unit: work order rejects invalid status transitions (e.g., pending -> completed)`
- `Unit: totalCost calculation equals laborCost + materialsCost`
- `Integration: completing work order creates accounting journal entry for maintenance expense`

---

#### 4.4 — Maintenance Dashboard UI

**What**: Property manager dashboard showing open requests, assignment workflow, and work order tracking with real-time status updates.

**Design**:

Pages:
- `/maintenance` — Kanban board view (columns: Open, Assigned, In Progress, Completed) + list/table view toggle
- `/maintenance/[id]` — Request detail with photos, conversation thread, work order history
- `/maintenance/new` — Staff-created request form (select property -> unit -> category -> priority)
- `/vendors` — Vendor directory with filter by specialty
- `/vendors/[id]` — Vendor detail with job history and performance metrics

**Testing**:
- `E2E: navigate to /maintenance, see kanban board with requests in correct columns`
- `E2E: drag request from Open to Assigned column, vendor assignment dialog appears`
- `E2E: create work order from request detail page, select vendor, set scheduled date`
- `E2E: complete work order with cost details, verify request moves to Completed`
- `E2E: tenant rates work order, vendor rating updates on vendor detail page`
- `E2E: maintenance list view supports filtering by property and priority`

---

## Phase 5: Accounting & Financial Reporting

### Purpose
Implement double-entry bookkeeping, chart of accounts, owner statements, bank reconciliation, and financial reports (P&L, balance sheet). After this phase, property managers can track all financial transactions with proper accounting and generate owner statements.

### Tasks

#### 5.1 — Chart of Accounts & Journal Entries

**What**: Double-entry accounting engine with chart of accounts, journal entry creation (manual and auto-generated from payments/expenses), and balance computation.

**Design**:

Default chart of accounts (auto-generated per organization):

```typescript
const DEFAULT_ACCOUNTS = [
  // Assets
  { code: "1000", name: "Operating Bank Account", type: "asset", subtype: "operating_bank" },
  { code: "1010", name: "Trust Bank Account", type: "asset", subtype: "trust_bank" },
  { code: "1100", name: "Accounts Receivable - Tenants", type: "asset", subtype: "accounts_receivable" },
  { code: "1200", name: "Security Deposits Held", type: "asset", subtype: "security_deposits" },
  // Liabilities
  { code: "2000", name: "Accounts Payable", type: "liability", subtype: "accounts_payable" },
  { code: "2100", name: "Security Deposit Liability", type: "liability", subtype: "security_deposit_liability" },
  { code: "2200", name: "Prepaid Rent", type: "liability", subtype: "prepaid_rent" },
  // Revenue
  { code: "4000", name: "Rental Income", type: "revenue", subtype: "rent_income" },
  { code: "4100", name: "Late Fee Income", type: "revenue", subtype: "late_fee_income" },
  { code: "4200", name: "Other Income", type: "revenue", subtype: "other_income" },
  // Expenses
  { code: "5000", name: "Maintenance & Repairs", type: "expense", subtype: "maintenance" },
  { code: "5100", name: "Management Fees", type: "expense", subtype: "management_fees" },
  { code: "5200", name: "Insurance", type: "expense", subtype: "insurance" },
  { code: "5300", name: "Property Taxes", type: "expense", subtype: "property_taxes" },
  { code: "5400", name: "Utilities", type: "expense", subtype: "utilities" },
];
```

Journal entry service:

```typescript
// apps/api/src/services/accounting.service.ts
export class AccountingService {
  async createJournalEntry(input: CreateJournalEntryInput): Promise<JournalEntry> {
    // Validate: sum of debits must equal sum of credits
    const totalDebits = input.lines.reduce((sum, l) => sum + l.debit, 0);
    const totalCredits = input.lines.reduce((sum, l) => sum + l.credit, 0);
    if (Math.abs(totalDebits - totalCredits) > 0.001) {
      throw new UnbalancedEntryError(totalDebits, totalCredits);
    }
    // Insert entry + lines in transaction
  }

  async createPaymentEntry(payment: Payment): Promise<JournalEntry> {
    // Rent payment:
    //   Debit:  1000 Operating Bank Account  $amount
    //   Credit: 4000 Rental Income           $amount
    return this.createJournalEntry({
      entryDate: payment.paymentDate,
      description: `Rent payment - ${tenantName} - ${unitNumber}`,
      referenceType: "payment",
      referenceId: payment.id,
      propertyId: property.id,
      lines: [
        { accountCode: "1000", debit: payment.amount, credit: 0 },
        { accountCode: "4000", debit: 0, credit: payment.amount },
      ],
    });
  }

  async getAccountBalance(orgId: string, accountCode: string, asOfDate?: string): Promise<number> {
    // Sum debits - credits for asset/expense accounts
    // Sum credits - debits for liability/equity/revenue accounts
  }

  async getTrialBalance(orgId: string, asOfDate: string): Promise<TrialBalanceRow[]> { ... }
}
```

**Testing**:
- `Unit: createJournalEntry rejects unbalanced entries (debits != credits)`
- `Unit: createPaymentEntry creates correct debit/credit lines for rent payment`
- `Unit: getAccountBalance returns correct balance for asset account (debits - credits)`
- `Unit: getAccountBalance returns correct balance for revenue account (credits - debits)`
- `Integration: default chart of accounts created when organization is created`
- `Integration: payment completion triggers journal entry creation`
- `Integration: trial balance debits equal credits across all accounts`
- `Unit: void journal entry creates reversing entry`

---

#### 5.2 — Owner Statements & Distributions

**What**: Generate monthly owner statements showing income, expenses, management fees, and net distribution amounts. Support owner distributions via ACH.

**Design**:

```typescript
// packages/shared/src/types/accounting.ts
export interface OwnerStatement {
  ownerId: string;
  propertyId: string;
  periodStart: string;
  periodEnd: string;
  grossIncome: number;
  expenses: ExpenseLineItem[];
  totalExpenses: number;
  managementFee: number;
  managementFeeRate: number;
  netDistribution: number;
  payments: PaymentSummary[];
}

export interface OwnerDistribution {
  id: string;
  organizationId: string;
  ownerId: string;
  propertyId: string;
  distributionDate: string;
  grossIncome: number;
  totalExpenses: number;
  managementFee: number;
  netDistribution: number;
  paymentMethod: string | null;
  status: "pending" | "paid" | "held";
  journalEntryId: string | null;
  createdAt: string;
}

// API Routes:
// GET  /api/v1/owners/:id/statements       — list statements for owner
// GET  /api/v1/owners/:id/statements/:period — get statement for period (YYYY-MM)
// POST /api/v1/owners/:id/distributions     — create distribution
// GET  /api/v1/owner-portal/statements      — owner portal: view statements
// GET  /api/v1/owner-portal/statements/:period/pdf — download PDF statement
```

**Testing**:
- `Unit: owner statement calculates gross income from all rent payments for property in period`
- `Unit: owner statement deducts maintenance expenses, management fee, insurance`
- `Unit: management fee calculated as percentage of gross income`
- `Unit: net distribution = gross income - total expenses - management fee`
- `Integration: POST /owners/:id/distributions creates journal entry crediting owner payable`
- `Integration: owner portal shows statements for properties owned by authenticated owner`
- `Unit: statement handles partial month correctly (pro-rated)`

---

#### 5.3 — Financial Reports & Tax Preparation

**What**: Generate P&L by property, balance sheet, rent roll, and 1099-ready reports. Support US Schedule E categorization.

**Design**:

```typescript
// API Routes:
// GET /api/v1/reports/profit-loss         — P&L report (query: propertyId, startDate, endDate)
// GET /api/v1/reports/balance-sheet       — balance sheet (query: asOfDate)
// GET /api/v1/reports/rent-roll           — current rent roll (all units, occupancy, rent amounts)
// GET /api/v1/reports/1099                — 1099-NEC data for vendors paid > $600
// GET /api/v1/reports/schedule-e          — Schedule E data grouped by property
// GET /api/v1/reports/aging               — accounts receivable aging (30/60/90 day buckets)

export interface ProfitLossReport {
  propertyId: string | null;
  propertyName: string | null;
  periodStart: string;
  periodEnd: string;
  income: ReportLineItem[];
  totalIncome: number;
  expenses: ReportLineItem[];
  totalExpenses: number;
  netOperatingIncome: number;
}

export interface RentRollReport {
  generatedAt: string;
  totalUnits: number;
  occupiedUnits: number;
  vacantUnits: number;
  occupancyRate: number;
  totalMonthlyRent: number;
  units: RentRollUnit[];
}
```

**Testing**:
- `Integration: P&L report sums correct income and expense accounts for date range`
- `Integration: P&L report filters by property when propertyId provided`
- `Integration: rent roll shows correct occupancy rate and total monthly rent`
- `Integration: 1099 report includes only vendors paid >= $600 in calendar year`
- `Integration: Schedule E report groups income/expenses by property`
- `Integration: aging report places overdue amounts in correct 30/60/90 day buckets`
- `Unit: balance sheet assets equal liabilities + equity`

---

#### 5.4 — Accounting Dashboard UI

**What**: Web dashboard for accounting module with journal entry browser, P&L charts, rent roll, and report generation.

**Design**:

Pages:
- `/accounting` — Overview with key metrics (total income, total expenses, NOI, outstanding balance)
- `/accounting/journal` — Journal entry list with date range filter, search, and manual entry form
- `/accounting/reports` — Report generator with selectable report type, date range, property filter, export to CSV/PDF
- `/accounting/owners` — Owner list with statement generation and distribution management

**Testing**:
- `E2E: navigate to /accounting, see income and expense summary for current month`
- `E2E: create manual journal entry with balanced debit/credit lines`
- `E2E: generate P&L report for specific property and date range`
- `E2E: export rent roll as CSV, verify correct column headers and data`
- `E2E: navigate to owner detail, view monthly statement, download PDF`

---

## Phase 6: Listings & Applicant Pipeline

### Purpose
Implement vacancy listing syndication (Zillow, Apartments.com), online rental applications, and tenant screening integration (TransUnion). After this phase, property managers can market vacant units, collect applications, and screen prospective tenants.

### Tasks

#### 6.1 — Listing Management

**What**: Create and manage vacancy listings with RESO-aligned fields, syndication tracking, and application links.

**Design**:

```typescript
// packages/shared/src/types/listing.ts
export interface Listing {
  id: string;
  unitId: string;
  title: string;
  description: string | null;
  listedRent: number;
  availableDate: string;
  status: "active" | "paused" | "filled" | "expired";
  listingData: ListingData;
  publishedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface ListingData {
  syndicatedTo?: string[];               // ["zillow", "apartments_com", "rentler"]
  zillowListingId?: string;              // RESO: ListingId
  petPolicy?: "no_pets" | "cats_only" | "dogs_only" | "cats_and_dogs";
  petDeposit?: number;
  parkingIncluded?: boolean;
  utilitiesIncluded?: string[];
  virtualTourUrl?: string;
  photos?: string[];
  showingInstructions?: string;
}

// API Routes:
// GET    /api/v1/listings                — list listings (filter by status, property)
// POST   /api/v1/listings                — create listing from unit
// PATCH  /api/v1/listings/:id            — update listing
// POST   /api/v1/listings/:id/publish    — publish to syndication platforms
// POST   /api/v1/listings/:id/pause      — pause syndication
// GET    /api/v1/listings/:id/public     — public listing page (no auth required)
```

**Testing**:
- `Integration: POST /listings creates listing with status "active"`
- `Integration: POST /listings/:id/publish updates syndicatedTo array`
- `Integration: GET /listings/:id/public returns listing without requiring authentication`
- `Integration: creating listing for occupied unit returns 409`
- `Unit: listing title auto-generated from unit attributes if not provided`

---

#### 6.2 — Online Applications & Screening

**What**: Online rental applications linked to listings, with TransUnion tenant screening integration (FCRA-compliant consent tracking and adverse action notices).

**Design**:

```typescript
// packages/shared/src/types/screening.ts
export interface Application {
  id: string;
  listingId: string | null;
  unitId: string;
  status: "submitted" | "under_review" | "approved" | "denied" | "withdrawn";
  applicantData: ApplicantData;
  reviewedBy: string | null;
  reviewedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface ScreeningRequest {
  id: string;
  organizationId: string;
  tenantId: string | null;
  applicationId: string | null;
  screeningProvider: "transunion" | "experian";
  status: "pending" | "completed" | "failed";
  consentGivenAt: string;
  consentMethod: "electronic" | "paper";
  overallRecommendation: "approve" | "conditional" | "deny" | null;
  reportData: ScreeningReportData;
  createdAt: string;
}

export interface ScreeningReportData {
  providerReferenceId?: string;
  creditScore?: number;
  criminalStatus?: "clear" | "flagged" | "pending";
  evictionStatus?: "clear" | "flagged" | "pending";
  incomeVerified?: boolean;
  incomeToRentRatio?: number;
  adverseActionSent?: boolean;         // FCRA requirement
  adverseActionSentAt?: string;
  reportExpiresAt?: string;
}

// API Routes:
// POST   /api/v1/applications                       — submit application (public, no auth)
// GET    /api/v1/applications                       — list applications (staff, filter by status, unit)
// PATCH  /api/v1/applications/:id                   — update status (approve/deny)
// POST   /api/v1/applications/:id/screen            — initiate screening
// GET    /api/v1/applications/:id/screening-results  — get screening results
// POST   /api/v1/applications/:id/adverse-action     — send FCRA adverse action notice
```

**Testing**:
- `Integration: POST /applications creates application from public listing page`
- `Integration: POST /applications/:id/screen records FCRA consent timestamp and method`
- `Integration (mocked TransUnion): screening returns credit score and recommendations`
- `Integration: approve application creates tenant record from applicant data`
- `Integration: deny application triggers FCRA adverse action notice requirement`
- `Unit: screening request rejects missing consent_given_at (FCRA compliance)`
- `Unit: adverse action notice includes required FCRA disclosure text`

---

## Phase 7: Communication & Notifications

### Purpose
Implement a unified messaging system spanning portal messages, email, and SMS with real-time notifications. After this phase, property managers, tenants, and owners receive timely notifications and can communicate through a unified inbox.

### Tasks

#### 7.1 — Unified Messaging System

**What**: Threaded messaging between property managers, tenants, owners, and vendors with context linking (tie messages to leases, maintenance requests, properties).

**Design**:

```typescript
// packages/shared/src/types/notification.ts
export interface Message {
  id: string;
  organizationId: string;
  threadId: string;
  channel: "portal" | "email" | "sms";
  direction: "inbound" | "outbound";
  senderType: "user" | "tenant" | "owner" | "vendor" | "system";
  senderId: string | null;
  recipientType: "user" | "tenant" | "owner" | "vendor";
  recipientId: string;
  subject: string | null;
  body: string;
  metadata: MessageMetadata;
  createdAt: string;
}

export interface MessageMetadata {
  relatedEntityType?: "property" | "unit" | "lease" | "maintenance_request" | "work_order";
  relatedEntityId?: string;
  deliveryStatus?: "pending" | "sent" | "delivered" | "failed";
  readAt?: string;
  attachments?: Array<{ filename: string; url: string; sizeBytes: number }>;
}

// API Routes:
// GET    /api/v1/messages                 — unified inbox (paginated, filterable by entity)
// GET    /api/v1/messages/threads/:id     — get thread with all messages
// POST   /api/v1/messages                 — send message (choose channel: portal/email/sms)
// PATCH  /api/v1/messages/:id/read        — mark as read
// GET    /api/v1/tenant-portal/messages   — tenant's messages
// POST   /api/v1/tenant-portal/messages   — tenant sends message
```

**Testing**:
- `Integration: POST /messages sends portal message and creates thread`
- `Integration: reply to existing thread appends message to same thread_id`
- `Integration: email message dispatched via Resend API (mocked)`
- `Integration: SMS message dispatched via Twilio API (mocked)`
- `Integration: tenant portal shows only messages for authenticated tenant`
- `Unit: message linked to maintenance request via metadata.relatedEntityId`

---

#### 7.2 — Notification Engine

**What**: Event-driven notifications dispatched via multiple channels (in-app, email, SMS, push) based on user preferences.

**Design**:

Notification types and their triggers:

```typescript
const NOTIFICATION_TRIGGERS = {
  rent_due: { trigger: "3 days before rent_due_day", channels: ["email", "in_app"] },
  rent_overdue: { trigger: "rent_due_day + grace_days", channels: ["email", "sms", "in_app"] },
  payment_received: { trigger: "payment.status = completed", channels: ["email", "in_app"] },
  payment_failed: { trigger: "payment.status = failed", channels: ["email", "sms", "in_app"] },
  maintenance_submitted: { trigger: "maintenance_request.created (by tenant)", channels: ["email", "in_app"] },
  maintenance_update: { trigger: "maintenance_request.status changed", channels: ["email", "in_app"] },
  work_order_scheduled: { trigger: "work_order.status = scheduled", channels: ["email", "in_app"] },
  lease_expiring: { trigger: "30 days before lease.end_date", channels: ["email", "in_app"] },
  lease_signed: { trigger: "lease.status = active", channels: ["email", "in_app"] },
  application_received: { trigger: "application.created", channels: ["email", "in_app"] },
  screening_complete: { trigger: "screening.status = completed", channels: ["email", "in_app"] },
};
```

```typescript
// apps/api/src/services/notification.service.ts
export class NotificationService {
  async dispatch(notification: NotificationInput): Promise<void> {
    // 1. Check user/tenant/owner preferences for enabled channels
    // 2. For each enabled channel:
    //    - in_app: insert into notifications table
    //    - email: enqueue email job via BullMQ
    //    - sms: enqueue SMS job via BullMQ
    //    - push: send via web push API
    // 3. Record dispatch status
  }
}
```

**Testing**:
- `Unit: dispatch sends to email and in_app channels based on user preferences`
- `Unit: dispatch skips SMS when user has sms=false in preferences`
- `Integration: rent_due notification sent 3 days before due date`
- `Integration: payment_received creates in-app notification for tenant`
- `Integration (mocked Resend): email notification dispatched with correct template`
- `Integration (mocked Twilio): SMS notification dispatched with correct body`
- `E2E: in-app notification badge appears in navigation when new notification exists`
- `E2E: clicking notification marks it as read and navigates to related entity`

---

## Phase 8: Inspections & Compliance

### Purpose
Implement property inspections (move-in, move-out, annual, HQS), Section 8/HCV voucher tracking, and jurisdiction-specific compliance rules. After this phase, property managers can conduct and record inspections and track subsidized housing compliance.

### Tasks

#### 8.1 — Inspection Management

**What**: Create, schedule, and complete property inspections with room-by-room condition tracking and photo documentation.

**Design**:

```typescript
// packages/shared/src/types/inspection.ts
export interface Inspection {
  id: string;
  propertyId: string;
  unitId: string | null;
  inspectionType: "move_in" | "move_out" | "annual" | "hqs" | "safety";
  inspectorUserId: string | null;
  inspectorVendorId: string | null;
  scheduledDate: string | null;
  completedDate: string | null;
  status: "scheduled" | "in_progress" | "completed" | "cancelled";
  overallCondition: "excellent" | "good" | "fair" | "poor" | null;
  items: InspectionItem[];
  notes: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface InspectionItem {
  id: string;
  roomArea: string;       // kitchen, bathroom, bedroom_1, living_room, exterior
  itemName: string;       // walls, flooring, fixtures, appliances, windows
  condition: "excellent" | "good" | "fair" | "poor" | "damaged";
  notes: string | null;
  photoUrl: string | null;
  requiresAction: boolean;
  maintenanceRequestId: string | null;   // auto-created if requiresAction = true
}

// API Routes:
// GET    /api/v1/inspections                         — list inspections
// POST   /api/v1/inspections                         — create/schedule inspection
// GET    /api/v1/inspections/:id                      — get inspection with items
// PATCH  /api/v1/inspections/:id                      — update inspection
// POST   /api/v1/inspections/:id/items                — add inspection items
// POST   /api/v1/inspections/:id/complete              — complete inspection
// GET    /api/v1/inspections/:id/report                — generate inspection report PDF
```

**Testing**:
- `Integration: POST /inspections creates inspection with status "scheduled"`
- `Integration: POST /inspections/:id/items adds room-by-room condition records`
- `Integration: item with requiresAction=true auto-creates linked maintenance request`
- `Integration: POST /inspections/:id/complete sets completedDate and overall condition`
- `Integration: move_in and move_out inspections linked to same unit enable condition comparison`
- `Unit: inspection report PDF includes all items, photos, and conditions`

---

#### 8.2 — Section 8 / HCV Compliance Tracking

**What**: Track Housing Choice Voucher data, HAP payments, HQS inspection schedules, and housing authority communications.

**Design**:

HCV data stored in lease `subsidyData` JSONB (per data-model-suggestion-3):

```typescript
export interface LeaseSubsidyData {
  program?: "section_8_hcv";
  voucherNumber?: string;
  housingAuthority?: string;
  hapAmount?: number;           // Housing Assistance Payment (HA pays)
  tenantPortion?: number;       // Tenant pays
  paymentStandard?: number;     // Maximum voucher amount for area
  contractStart?: string;
  contractEnd?: string;
  lastHqsInspection?: string;
  nextHqsInspectionDue?: string;
}

// API Routes:
// GET    /api/v1/leases/hcv              — list all HCV leases with voucher status
// GET    /api/v1/leases/hcv/inspections  — upcoming HQS inspections
// PATCH  /api/v1/leases/:id/subsidy      — update subsidy data
```

**Testing**:
- `Integration: GET /leases/hcv returns only leases with section_8_hcv subsidy data`
- `Integration: GET /leases/hcv/inspections returns HQS inspections due within 90 days`
- `Integration: PATCH /leases/:id/subsidy updates HAP amount and tenant portion`
- `Unit: HCV lease total (hapAmount + tenantPortion) validates against paymentStandard`

---

## Phase 9: AI-Powered Features

### Purpose
Implement the AI-native differentiators: predictive delinquency alerting, AI maintenance triage, dynamic rent pricing recommendations, and compliance summarization. These are the features no incumbent fully offers and represent the core competitive advantage.

### Tasks

#### 9.1 — AI Service Infrastructure

**What**: Deploy the Python FastAPI microservice for ML inference, with model versioning, health checks, and internal API authentication.

**Design**:

```python
# apps/ai-service/src/main.py
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI(title="PMS AI Service", version="1.0.0")

class PredictionRequest(BaseModel):
    organization_id: str
    prediction_type: str  # "delinquency" | "rent_pricing" | "maintenance_triage"
    entity_type: str
    entity_id: str
    features: dict

class PredictionResponse(BaseModel):
    prediction_type: str
    entity_id: str
    results: dict
    model_version: str
    confidence: float

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest, api_key: str = Depends(verify_api_key)):
    model = MODEL_REGISTRY[request.prediction_type]
    result = model.predict(request.features)
    return PredictionResponse(
        prediction_type=request.prediction_type,
        entity_id=request.entity_id,
        results=result,
        model_version=model.version,
        confidence=result.get("confidence", 0.0),
    )
```

**Testing**:
- `Unit: /predict endpoint rejects requests without valid API key`
- `Unit: /predict endpoint returns 422 for invalid prediction_type`
- `Integration: /health endpoint returns status and loaded model versions`
- `Integration: /predict with delinquency type returns risk_score and risk_level`
- `Unit: model registry loads correct model version for each prediction type`

---

#### 9.2 — Predictive Delinquency Alerting

**What**: ML model that analyzes tenant payment history, lease terms, and seasonal patterns to predict missed payments 30-60 days ahead.

**Design**:

```python
# apps/ai-service/src/models/delinquency.py
class DelinquencyModel:
    version = "delinquency-v1.0"

    def extract_features(self, tenant_data: dict) -> np.ndarray:
        """Extract features from tenant payment history."""
        return np.array([
            tenant_data["late_payment_count_6m"],     # Late payments in last 6 months
            tenant_data["late_payment_count_12m"],    # Late payments in last 12 months
            tenant_data["avg_days_late_6m"],           # Average days late in last 6 months
            tenant_data["payment_consistency_score"],  # Std dev of payment timing
            tenant_data["lease_tenure_months"],        # How long tenant has been on lease
            tenant_data["rent_to_income_ratio"],       # Monthly rent / monthly income
            tenant_data["month_of_year"],              # Seasonal factor (1-12)
            tenant_data["local_unemployment_rate"],    # Macro indicator
        ])

    def predict(self, features: dict) -> dict:
        feature_vector = self.extract_features(features)
        probability = self.model.predict_proba(feature_vector)[0][1]
        risk_level = self._classify_risk(probability)
        return {
            "risk_score": round(probability, 4),
            "risk_level": risk_level,
            "contributing_factors": self._explain(feature_vector),
            "recommended_action": self._recommend_action(risk_level),
            "30_day_probability": round(probability * 0.6, 4),
            "60_day_probability": round(probability, 4),
        }

    def _classify_risk(self, probability: float) -> str:
        if probability >= 0.7: return "critical"
        if probability >= 0.5: return "high"
        if probability >= 0.3: return "moderate"
        return "low"

    def _recommend_action(self, risk_level: str) -> str:
        return {
            "low": "no_action",
            "moderate": "send_reminder",
            "high": "outreach_call",
            "critical": "payment_plan",
        }[risk_level]
```

AI prediction storage and API:

```typescript
// API Routes:
// POST   /api/v1/ai/predict/delinquency           — run prediction for tenant
// GET    /api/v1/ai/predictions/delinquency        — list recent predictions (filter by risk_level)
// GET    /api/v1/tenants/:id/delinquency-risk      — get latest risk score for tenant

// BullMQ job: runs nightly for all active tenants
// apps/worker/src/processors/ai-prediction.processor.ts
export async function processDelinquencyPredictions(job: Job) {
  // For each active lease:
  //   1. Gather payment history for tenant
  //   2. Call AI service /predict
  //   3. Store result in ai_predictions table
  //   4. If risk_level >= "high", send notification to property manager
}
```

**Testing**:
- `Unit: delinquency model returns risk_score between 0 and 1`
- `Unit: tenant with 4/6 late payments classified as "high" risk`
- `Unit: tenant with 0 late payments and low rent-to-income ratio classified as "low"`
- `Unit: contributing_factors includes top 3 factors driving the score`
- `Integration: nightly job runs predictions for all active tenants and stores results`
- `Integration: high-risk prediction triggers notification to property manager`
- `Integration: GET /tenants/:id/delinquency-risk returns latest prediction`
- `Unit: model handles missing income data gracefully (uses default ratio)`

---

#### 9.3 — AI Maintenance Triage

**What**: Classify maintenance request urgency from description and photos, recommend a category and vendor, and auto-assign when confidence is high.

**Design**:

```python
# apps/ai-service/src/models/maintenance_triage.py
class MaintenanceTriageModel:
    version = "maint-triage-v1.0"

    def predict(self, features: dict) -> dict:
        """
        Input features:
          - title: str
          - description: str
          - photo_descriptions: list[str]  (from vision model)
          - property_type: str
          - unit_type: str
        """
        # Use LLM to classify urgency and category
        prompt = f"""Classify this maintenance request:
Title: {features['title']}
Description: {features['description']}
Photos: {features.get('photo_descriptions', 'none')}

Return JSON with:
- priority: emergency|urgent|normal|low
- category: plumbing|electrical|hvac|appliance|structural|pest|general
- reasoning: one sentence explanation
- auto_assign_confidence: 0.0-1.0
"""
        response = self.llm.invoke(prompt)
        parsed = json.loads(response)

        vendor_id = None
        if parsed["auto_assign_confidence"] >= 0.85:
            vendor_id = self._find_best_vendor(
                features["organization_id"],
                parsed["category"],
                features["property_zip"],
            )

        return {
            "ai_priority": parsed["priority"],
            "ai_category": parsed["category"],
            "confidence": parsed["auto_assign_confidence"],
            "recommended_vendor_id": vendor_id,
            "reasoning": parsed["reasoning"],
        }
```

**Testing**:
- `Unit: "water pouring from ceiling" classified as priority=emergency, category=plumbing`
- `Unit: "squeaky door hinge" classified as priority=low, category=general`
- `Unit: confidence >= 0.85 triggers vendor recommendation`
- `Unit: confidence < 0.85 returns null vendor_id`
- `Integration: POST /maintenance with AI triage enabled stores ai_triage JSONB on request`
- `Integration: auto-assigned request creates work order and updates status to "assigned"`
- `Unit: photo descriptions extracted from uploaded images enhance classification accuracy`

---

#### 9.4 — Dynamic Rent Pricing Recommendations

**What**: Recommend optimal rent prices based on market comparables, vacancy rates, seasonal demand, and unit attributes.

**Design**:

```python
# apps/ai-service/src/models/rent_pricing.py
class RentPricingModel:
    version = "rent-pricing-v1.0"

    def predict(self, features: dict) -> dict:
        """
        Input features:
          - current_rent: float
          - bedrooms: int
          - bathrooms: int
          - living_area_sqft: float
          - property_type: str
          - zip_code: str
          - month: int (1-12)
          - occupancy_rate_local: float
          - days_on_market_local: float
          - comparable_rents: list[float]
        """
        market_average = np.mean(features["comparable_rents"])
        vacancy_adjustment = self._vacancy_factor(features["occupancy_rate_local"])
        seasonal_adjustment = self._seasonal_factor(features["month"])

        recommended = market_average * vacancy_adjustment * seasonal_adjustment

        return {
            "current_rent": features["current_rent"],
            "recommended_rent": round(recommended, 2),
            "market_average": round(market_average, 2),
            "vacancy_rate_local": features["occupancy_rate_local"],
            "confidence": self._calculate_confidence(len(features["comparable_rents"])),
            "comparable_units": len(features["comparable_rents"]),
            "reasoning": self._generate_reasoning(features, recommended),
        }
```

```typescript
// API Routes:
// POST   /api/v1/ai/predict/rent-pricing         — get rent recommendation for unit
// GET    /api/v1/units/:id/rent-recommendation    — get latest recommendation
// GET    /api/v1/ai/predictions/rent-pricing      — list all recent recommendations
```

**Testing**:
- `Unit: recommendation adjusts up when vacancy rate is low (< 5%)`
- `Unit: recommendation adjusts down when vacancy rate is high (> 10%)`
- `Unit: seasonal factor increases rent in peak months (May-Aug)`
- `Unit: confidence increases with number of comparable units`
- `Unit: recommendation never exceeds 120% of market average (sanity cap)`
- `Integration: weekly job generates recommendations for all vacant units`
- `Integration: recommendation stored in ai_predictions table with model_version`

---

## Phase 10: Open API & Developer Platform

### Purpose
Expose a fully documented public REST API with OpenAPI 3.1 specification, API key management, rate limiting, and webhook subscriptions. This enables third-party integrations and positions the platform as extensible.

### Tasks

#### 10.1 — Public API with OpenAPI Documentation

**What**: Expose the internal API as a versioned public REST API with auto-generated OpenAPI 3.1 specification and interactive documentation.

**Design**:

```typescript
// apps/api/src/plugins/openapi.ts
await fastify.register(swagger, {
  openapi: {
    info: {
      title: "Property Management System API",
      version: "1.0.0",
      description: "Open REST API for property management operations",
    },
    servers: [
      { url: "https://api.example.com", description: "Production" },
      { url: "http://localhost:3001", description: "Development" },
    ],
    components: {
      securitySchemes: {
        apiKey: { type: "apiKey", in: "header", name: "X-API-Key" },
        bearer: { type: "http", scheme: "bearer", bearerFormat: "JWT" },
      },
    },
  },
});

// API key management routes:
// POST   /api/v1/api-keys                 — create API key
// GET    /api/v1/api-keys                 — list API keys
// DELETE /api/v1/api-keys/:id             — revoke API key
// POST   /api/v1/webhooks/subscriptions   — subscribe to events
// GET    /api/v1/webhooks/subscriptions   — list subscriptions
// DELETE /api/v1/webhooks/subscriptions/:id — unsubscribe
```

Rate limiting:

```typescript
// apps/api/src/plugins/rate-limit.ts
await fastify.register(rateLimit, {
  max: 100,           // requests per window
  timeWindow: "1 minute",
  keyGenerator: (request) => {
    return request.headers["x-api-key"] || request.ip;
  },
  // Higher limits for authenticated API keys
  allowList: [],
  onExceeding: (request) => { /* log */ },
  onExceeded: (request) => { /* log + alert */ },
});
```

**Testing**:
- `Integration: GET /docs returns OpenAPI 3.1 JSON specification`
- `Integration: OpenAPI spec includes all v1 routes with request/response schemas`
- `Integration: POST /api-keys creates API key and returns it (shown once)`
- `Integration: request with valid API key authenticated successfully`
- `Integration: request with revoked API key returns 401`
- `Integration: exceeding rate limit returns 429 with Retry-After header`
- `Unit: rate limit key generator uses API key when present, falls back to IP`

---

#### 10.2 — Webhook System

**What**: Allow API consumers to subscribe to events (payment.completed, maintenance.created, lease.signed) and receive webhook callbacks with signature verification.

**Design**:

```typescript
export interface WebhookSubscription {
  id: string;
  organizationId: string;
  url: string;
  events: string[];      // ["payment.completed", "maintenance.created", "lease.signed"]
  secret: string;        // HMAC signing secret
  isActive: boolean;
  createdAt: string;
}

// Webhook delivery service:
export class WebhookService {
  async deliver(orgId: string, eventType: string, payload: unknown): Promise<void> {
    const subscriptions = await this.getActiveSubscriptions(orgId, eventType);
    for (const sub of subscriptions) {
      const signature = createHmac("sha256", sub.secret)
        .update(JSON.stringify(payload))
        .digest("hex");

      await this.webhookQueue.add("deliver", {
        url: sub.url,
        payload,
        headers: {
          "X-PMS-Signature": `sha256=${signature}`,
          "X-PMS-Event": eventType,
          "X-PMS-Delivery-ID": ulid(),
        },
      }, {
        attempts: 5,
        backoff: { type: "exponential", delay: 1000 },
      });
    }
  }
}
```

**Testing**:
- `Integration: POST /webhooks/subscriptions creates subscription with signing secret`
- `Integration: payment completion triggers webhook delivery to subscribed URL`
- `Integration (mocked HTTP): webhook includes HMAC signature in X-PMS-Signature header`
- `Integration: failed webhook delivery retries with exponential backoff (5 attempts)`
- `Unit: HMAC signature verification matches expected hash`
- `Unit: webhook delivery skips inactive subscriptions`

---

## Phase 11: Reporting & Analytics Dashboard

### Purpose
Build interactive analytics dashboards with portfolio-level metrics, occupancy trends, revenue forecasting, and AI prediction visualizations. After this phase, property managers have a comprehensive view of their portfolio performance.

### Tasks

#### 11.1 — Portfolio Dashboard

**What**: Executive dashboard with KPIs (occupancy rate, total revenue, outstanding balance, maintenance backlog), trend charts, and alerts.

**Design**:

Dashboard metrics:

```typescript
export interface PortfolioMetrics {
  totalProperties: number;
  totalUnits: number;
  occupiedUnits: number;
  vacantUnits: number;
  occupancyRate: number;
  totalMonthlyRent: number;
  collectedThisMonth: number;
  collectionRate: number;
  outstandingBalance: number;
  openMaintenanceRequests: number;
  avgMaintenanceResolutionDays: number;
  leasesExpiringIn30Days: number;
  leasesExpiringIn60Days: number;
  highRiskTenants: number;
  revenueThisMonth: number;
  expensesThisMonth: number;
  netOperatingIncome: number;
}

// API Routes:
// GET /api/v1/dashboard/portfolio          — portfolio-level KPIs
// GET /api/v1/dashboard/occupancy-trend    — 12-month occupancy trend
// GET /api/v1/dashboard/revenue-trend      — 12-month revenue/expense trend
// GET /api/v1/dashboard/collection-rate    — monthly collection rate trend
// GET /api/v1/dashboard/delinquency        — AI delinquency risk overview
```

**Testing**:
- `Integration: GET /dashboard/portfolio returns correct occupancy rate from seed data`
- `Integration: GET /dashboard/revenue-trend returns 12 monthly data points`
- `Integration: delinquency overview includes count of high/critical risk tenants`
- `E2E: dashboard loads within 2 seconds with portfolio summary cards`
- `E2E: occupancy trend chart renders 12-month line chart`
- `E2E: clicking "Leases Expiring" card navigates to filtered lease list`

---

#### 11.2 — Custom Report Builder

**What**: User-configurable report builder for generating custom financial and operational reports with export to CSV, Excel, and PDF.

**Design**:

```typescript
export interface ReportConfig {
  name: string;
  reportType: "financial" | "operational" | "compliance";
  columns: string[];
  filters: ReportFilter[];
  groupBy: string[];
  sortBy: { field: string; direction: "asc" | "desc" }[];
  dateRange: { start: string; end: string };
  propertyIds: string[] | null;   // null = all properties
}

// API Routes:
// POST   /api/v1/reports/generate       — generate report from config
// GET    /api/v1/reports/templates       — list saved report templates
// POST   /api/v1/reports/templates       — save report template
// GET    /api/v1/reports/:id/export/csv  — export as CSV
// GET    /api/v1/reports/:id/export/pdf  — export as PDF
```

**Testing**:
- `Integration: POST /reports/generate with P&L config returns correct totals`
- `Integration: export to CSV includes correct headers and formatted data`
- `Integration: report filtered by property returns only that property's data`
- `E2E: user creates custom report, selects columns, applies date filter, sees results`
- `E2E: user saves report as template and reloads it later`

---

## Phase 12: MCP Server & Advanced Integrations

### Purpose
Build an MCP (Model Context Protocol) server so AI assistants can query the platform's data directly, and integrate with RESO Web API for listing syndication. This is the final differentiating capability that no incumbent offers.

### Tasks

#### 12.1 — MCP Server

**What**: Expose property, lease, tenant, maintenance, and financial data through the Model Context Protocol, enabling AI assistants (Claude, ChatGPT, Gemini) to query and act on property management data.

**Design**:

```typescript
// apps/mcp-server/src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server";

const server = new Server({
  name: "property-management-system",
  version: "1.0.0",
});

// Tools exposed to AI assistants:
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "list_properties",
      description: "List all properties with their unit counts and occupancy rates",
      inputSchema: {
        type: "object",
        properties: {
          type: { type: "string", enum: ["residential", "commercial", "mixed_use"] },
          city: { type: "string" },
        },
      },
    },
    {
      name: "get_tenant_balance",
      description: "Get the current rent balance for a tenant",
      inputSchema: {
        type: "object",
        properties: { tenantId: { type: "string" } },
        required: ["tenantId"],
      },
    },
    {
      name: "list_maintenance_requests",
      description: "List open maintenance requests, optionally filtered by priority or property",
      inputSchema: { ... },
    },
    {
      name: "get_portfolio_summary",
      description: "Get a high-level portfolio summary including occupancy, revenue, and maintenance metrics",
      inputSchema: { type: "object", properties: {} },
    },
    {
      name: "get_delinquency_risks",
      description: "List tenants at high risk of missed payment in the next 30-60 days",
      inputSchema: { type: "object", properties: {} },
    },
  ],
}));
```

**Testing**:
- `Integration: MCP server responds to tools/list with all available tools`
- `Integration: list_properties tool returns properties with correct data`
- `Integration: get_tenant_balance tool returns current balance for valid tenant`
- `Integration: MCP server rejects unauthenticated connections`
- `Unit: tool input validation rejects invalid parameters`
- `Integration: get_delinquency_risks returns tenants with risk_level >= "high"`

---

#### 12.2 — RESO Web API Listing Syndication

**What**: Implement RESO Web API (OData V4) compatible listing feeds for syndication to MLS providers, Zillow, and Apartments.com.

**Design**:

```typescript
// RESO-compatible listing endpoint:
// GET /api/v1/reso/Property?$filter=StandardStatus eq 'Active'&$select=ListingId,ListPrice,BedroomsTotal
// Response follows OData V4 JSON format with RESO Data Dictionary field names

export interface RESOPropertyResource {
  ListingId: string;                    // RESO: Listing identifier
  ListPrice: number;                    // Monthly rent
  StandardStatus: "Active" | "Closed" | "Coming Soon";
  BedroomsTotal: number;
  BathroomsFull: number;
  LivingArea: number;                   // Square footage
  StreetNumber: string;
  StreetName: string;
  City: string;
  StateOrProvince: string;
  PostalCode: string;
  ListingDescription: string;
  Media: RESOMediaResource[];
}
```

**Testing**:
- `Integration: GET /reso/Property returns OData V4 formatted response`
- `Integration: $filter parameter filters by StandardStatus`
- `Integration: $select parameter limits returned fields`
- `Integration: response includes @odata.context and @odata.count`
- `Unit: RESO field mapping converts internal field names to RESO Data Dictionary names`
- `Integration: OAuth 2.0 Bearer Token authentication on RESO endpoints`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffolding      ─── required by everything
    │
    ├── Phase 2: Tenant & Lease Management      ─── requires Phase 1
    │       │
    │       ├── Phase 3: Rent Collection        ─── requires Phase 2
    │       │       │
    │       │       └── Phase 5: Accounting     ─── requires Phase 3
    │       │               │
    │       │               └── Phase 11: Analytics Dashboard ─── requires Phase 5
    │       │
    │       ├── Phase 6: Listings & Applicants  ─── requires Phase 2, can parallel with Phase 3
    │       │
    │       └── Phase 8: Inspections            ─── requires Phase 2, can parallel with Phase 3
    │
    ├── Phase 4: Maintenance & Work Orders      ─── requires Phase 1, can parallel with Phase 2
    │       │
    │       └── Phase 9: AI Features            ─── requires Phase 3, Phase 4
    │
    ├── Phase 7: Communication & Notifications  ─── requires Phase 1, can start after Phase 2
    │
    ├── Phase 10: Open API & Developer Platform ─── requires Phase 5 (all core APIs complete)
    │
    └── Phase 12: MCP Server & RESO            ─── requires Phase 10
```

### Parallelism Opportunities

- **Phases 2 and 4** can be developed concurrently after Phase 1 (tenants/leases and maintenance are independent modules)
- **Phases 3, 6, and 8** can be developed concurrently after Phase 2 (payments, listings, and inspections are independent)
- **Phase 7** (communication) can start after Phase 2 and run alongside Phases 3-6
- **Phase 9** (AI features) requires both Phase 3 (payment data for delinquency) and Phase 4 (maintenance data for triage)
- **Phase 11** (analytics) requires Phase 5 (accounting data) but can start alongside Phase 9

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with passing code.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (with PostgreSQL and Redis running).
4. Biome linting passes with zero diagnostics (`pnpm lint`).
5. TypeScript type checking passes (`pnpm typecheck`).
6. Docker Compose builds succeed for all modified services (`docker compose build`).
7. The feature works end-to-end through the web UI (where applicable).
8. Database migrations are generated and apply cleanly to a fresh database.
9. New API endpoints appear in the auto-generated OpenAPI specification.
10. New environment variables and configuration options are documented in `.env.example`.
11. API endpoints return correct HTTP status codes (201 for creation, 409 for conflicts, 400 for validation errors).
12. Multi-tenant isolation verified (org-A cannot access org-B's data).
13. Audit log captures all create/update/delete operations on sensitive entities.
