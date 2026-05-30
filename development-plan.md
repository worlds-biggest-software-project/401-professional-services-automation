# Professional Services Automation (PSA) — Phased Development Plan

> Project: 401-professional-services-automation · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The persistence design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the primary schema — it preserves financial-grade typed columns for money/dates while allowing flexible JSONB for billing terms, custom fields, workflow definitions, AI provenance, and integration payloads. The **graph layer from Suggestion 4 (Neo4j / Apache AGE)** is deferred to an optional advanced phase, in line with that suggestion's own "start with PostgreSQL only" guidance.

---

## Core Requirements (Synthesis)

**What it does.** A single platform unifying project delivery, resource planning, time and expense tracking, client billing, and ASC 606 / IFRS 15 revenue recognition for professional services firms — closing the CRM-to-delivery-to-billing gap that forces firms to stitch together separate tools.

**Who uses it.** Operations leaders, delivery/project managers, consultants (time/expense entry), finance teams (billing, revenue recognition), and clients (read-only portal). Primary market: mid-market consulting firms, agencies, and IT services providers underserved by enterprise-priced incumbents (Kantata, Certinia).

**Key differentiators.**
1. Open, self-hostable PSA with financial-grade revenue recognition (no incumbent offers this).
2. AI embedded across the lifecycle — project health scoring, AI-suggested time entries, resource-match recommendations, AI-drafted client status reports.
3. First-class client portal.
4. An **MCP server** exposing PSA data to AI assistants — no incumbent has one (first-mover opportunity per `standards.md`).

**Deployment model.** Self-hosted and cloud (SaaS) via Docker. Multi-tenant from day one (PostgreSQL RLS).

**Integration surface.** Salesforce + HubSpot (CRM, deal-to-project), QuickBooks Online + Xero (accounting, invoice sync), calendar (RFC 5545) for AI time suggestions, LLM provider for AI features, MCP for AI-assistant access.

**Standards compliance.** OAuth 2.0 (RFC 6749) + PKCE (RFC 7636) + OIDC for auth; JWT (RFC 7519) bearer tokens (RFC 6750); OpenAPI 3.1 + JSON Schema 2020-12 for the API; ISO 8601 for all dates; RFC 4180 for CSV export; RFC 5545 iCalendar for scheduling; ASC 606 / IFRS 15 for revenue recognition; GDPR + SOC 2 controls (audit log, encryption, RBAC); MCP for AI integration.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Integration-and-API-heavy domain; official SDKs exist for every required integration (Salesforce, HubSpot, QuickBooks, Xero) in Node; shared types across API, workers, and a future web UI; strong OpenAPI tooling. |
| API framework | NestJS (Fastify adapter) | Modular DI architecture maps cleanly to bounded contexts (projects, resources, time, billing, revenue, integrations); decorator-driven validation; first-class OpenAPI 3.1 generation; guards model RBAC + tenant scoping. |
| Validation / schemas | Zod + `nestjs-zod` | Runtime validation that doubles as the source of truth for JSON Schema 2020-12 / OpenAPI; validates JSONB payloads (billing terms, workflow defs, custom fields) the DB cannot enforce. |
| ORM / DB access | Prisma | Type-safe queries; first-class JSONB support; Prisma Migrate for versioned migrations; matches Suggestion 3's recommendation. |
| Primary database | PostgreSQL 16 | Suggestion 3 backbone — typed relational columns for money/dates, JSONB + GIN/expression indexes for flexible areas, RLS for tenant isolation, NUMERIC for exact financial math. |
| Multi-tenancy | PostgreSQL Row-Level Security, `tenant_id` on every tenant-scoped table | DB-enforced isolation prevents application-layer leak bugs; SOC 2 / GDPR posture. |
| Cache / rate limiting | Redis 7 | Session/token cache, idempotency keys, BullMQ backing store, dashboard query caching. |
| Task queue | BullMQ (Redis) | Async webhook processing, invoice runs, revenue posting, accounting sync with retry/backoff, AI calls, scheduled recomputation. |
| Auth | OAuth 2.0 Authorization Code + PKCE, OIDC, JWT access tokens | RFC 6749/7636/7519/6750 + OIDC for enterprise SSO (Okta, Azure AD). |
| LLM provider | Anthropic Claude via abstraction layer | AI health scoring, time suggestions, status reports; provider abstraction allows swap-out. Prompt caching for repeated system prompts. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Differentiating AI-native integration layer; exposes read tools + drafting tools. |
| Frontend | Next.js 16 (App Router, React 19) + shadcn/ui + Tailwind | Internal dashboard + client portal; server components for fast dashboards, PPR/Cache Components for profitability views. |
| Testing | Vitest (unit/integration), Supertest (HTTP), Playwright (E2E), Testcontainers (real Postgres/Redis) | Fast unit runner; real-dependency integration via ephemeral containers; browser E2E for portal + timesheet flows. |
| Code quality | ESLint (typescript-eslint), Prettier, `tsc --noEmit` | Standard TS toolchain. |
| Package manager / monorepo | pnpm workspaces + Turborepo | API, web, workers, MCP server, and shared packages (types, db, integrations) in one repo with cached task graph. |
| Containerisation | Docker + docker-compose | Self-hosted and cloud parity; Postgres + Redis + API + worker + web + MCP services. |
| Money handling | `decimal.js` mapped to PG `NUMERIC` | Never use floats for currency; mirror Suggestion 1/3 NUMERIC columns. |
| Object storage | S3-compatible (MinIO local, S3 prod) | Receipt images, generated invoice PDFs, exported reports. |
| Observability | Pino structured logs, OpenTelemetry traces | SOC 2 audit + debugging. |

### Project Structure

```
psa/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── Dockerfile.mcp
├── .env.example
├── packages/
│   ├── types/                  # shared TS types + Zod schemas (single source of truth)
│   │   └── src/
│   │       ├── projects.ts
│   │       ├── time.ts
│   │       ├── billing.ts
│   │       ├── revenue.ts
│   │       └── index.ts
│   ├── db/                     # Prisma schema, client, migrations, RLS helpers, seed
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   └── src/
│   │       ├── client.ts       # tenant-scoped Prisma client factory (sets app.current_tenant)
│   │       └── rls.ts
│   ├── money/                  # decimal helpers, currency, rounding
│   ├── integrations/           # CRM + accounting adapters behind common interfaces
│   │   └── src/
│   │       ├── crm/ (salesforce.ts, hubspot.ts, crm-adapter.ts)
│   │       ├── accounting/ (quickbooks.ts, xero.ts, accounting-adapter.ts)
│   │       └── calendar/ (ical.ts, google.ts, microsoft.ts)
│   └── ai/                     # LLM provider abstraction + prompt templates
│       └── src/
│           ├── provider.ts
│           └── prompts/
├── apps/
│   ├── api/                    # NestJS REST API
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/         # guards, interceptors, tenant context, RBAC
│   │       ├── auth/
│   │       ├── clients/
│   │       ├── contracts/
│   │       ├── projects/
│   │       ├── tasks/
│   │       ├── resources/
│   │       ├── time-entries/
│   │       ├── expenses/
│   │       ├── approvals/      # workflow engine
│   │       ├── invoices/
│   │       ├── revenue/        # ASC 606 / IFRS 15 engine
│   │       ├── reporting/      # utilisation, profitability
│   │       ├── webhooks/       # inbound CRM/accounting webhooks
│   │       ├── ai/             # health scoring, suggestions, reports
│   │       └── export/         # CSV (RFC 4180), iCal (RFC 5545), PDF
│   ├── worker/                 # BullMQ processors
│   │   └── src/processors/
│   ├── web/                    # Next.js dashboard + client portal
│   └── mcp/                    # MCP server
└── tests/
    ├── fixtures/               # sample SF/HubSpot/QBO payloads, ASC 606 scenarios
    └── e2e/                    # Playwright specs
```

---

## Phase 1: Foundation — Monorepo, Database, Tenancy, Auth

### Purpose
Establish the runnable skeleton: monorepo tooling, the PostgreSQL schema with RLS multi-tenancy, the core data-access layer, authentication, and RBAC. After this phase a developer can register a tenant, authenticate, and the API enforces tenant isolation on every request. Nothing domain-specific ships yet, but every later phase plugs into this spine without rework.

### Tasks

#### 1.1 — Monorepo & Tooling Bootstrap
**What**: Stand up the pnpm/Turborepo workspace, Docker Compose stack, and shared config.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `turbo.json` pipeline: `build` → `lint` → `typecheck` → `test`.
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, `api`, `worker`, `web`, `mcp`.
- Root tsconfig with project references; ESLint + Prettier shared configs.
- `.env.example` with: `DATABASE_URL`, `REDIS_URL`, `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, `S3_ENDPOINT/KEY/SECRET/BUCKET`, `ANTHROPIC_API_KEY`, `OIDC_*`, `BASE_URL`.

**Testing**:
- `Smoke: docker compose up` → postgres, redis, minio become healthy (healthchecks pass).
- `Unit: config loader with full .env → typed Config object; missing DATABASE_URL → startup error naming the variable.`
- `CI: turbo build && turbo typecheck` exit 0 on empty skeleton.

#### 1.2 — Database Schema & Migrations (Prisma, Suggestion 3)
**What**: Implement the hybrid relational+JSONB schema as Prisma models with the initial migration.

**Design**: Translate Suggestion 3 tables to Prisma. Money = `Decimal @db.Decimal(14,2)` (rates `(10,2)`, hours `(5,2)`/`(10,2)`). Dates = `@db.Date`; timestamps = `DateTime @db.Timestamptz(6)`. Tenant-scoped tables carry `tenantId String @db.Uuid`. Core models (full field lists per Suggestion 3): `Tenant`, `User`, `Client`, `Contract`, `Project`, `ProjectTask`, `Resource`, `ResourceAssignment`, `TimeEntry`, `Expense`, `Invoice`, `Payment`, `RevenueSchedule`, `WorkflowDefinition`, `IntegrationEvent`, `AuditLog`. JSONB columns typed in Prisma as `Json` and validated at the app layer via Zod (Task 1.5).

Status enums (string-backed for forward-compat with workflow engine):
```ts
type ProjectStatus  = 'planning'|'active'|'on_hold'|'completed'|'cancelled';
type TaskStatus     = 'not_started'|'in_progress'|'blocked'|'done';
type EntryStatus    = 'draft'|'submitted'|'approved'|'rejected';
type InvoiceStatus  = 'draft'|'finalized'|'sent'|'paid'|'void';
type ContractStatus = 'draft'|'active'|'completed'|'cancelled';
```
Indexes per Suggestion 3: GIN on `custom_fields`/`skills`; expression index on `health_score->>'risk_level'` and `accounting_sync->>'sync_status'`; composite `(tenant_id, entry_date)` and `(tenant_id, resource_id, entry_date)` on `time_entries`; `(tenant_id, status, issue_date)` on `invoices`.

**Testing**:
- `Integration (Testcontainers): run migration → all tables exist; \\d projects shows JSONB columns + GIN index.`
- `Unit: Decimal column round-trips 12345.67 with no float drift.`
- `Integration: insert time_entry with hours = 0 → CHECK (hours > 0) violation.`
- `Integration: insert duplicate (tenant_id, invoice_number) → unique violation.`

#### 1.3 — RLS & Tenant-Scoped DB Client
**What**: Enable Row-Level Security and a Prisma client factory that sets `app.current_tenant` per request.

**Design**:
- Migration runs `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and `CREATE POLICY tenant_isolation USING (tenant_id = current_setting('app.current_tenant')::uuid)` for every tenant-scoped table. App connects as a non-superuser role (RLS is bypassed by superusers).
- `getTenantDb(tenantId)` runs `SET LOCAL app.current_tenant = $1` inside a transaction wrapper so the setting is connection-scoped via PgBouncer-safe `SET LOCAL`.

**Testing**:
- `Integration: write project under tenant A; query under tenant B → 0 rows (RLS enforced at DB).`
- `Integration: query with app.current_tenant unset → 0 rows (fail closed).`
- `Unit: getTenantDb wraps calls in a transaction and issues SET LOCAL once.`

#### 1.4 — Authentication & RBAC
**What**: OAuth 2.0 Authorization Code + PKCE login, OIDC SSO, JWT issuance, and role guards.

**Design**:
- Endpoints: `GET /auth/authorize`, `POST /auth/token` (code+PKCE → access+refresh JWT), `POST /auth/refresh`, `GET /.well-known/openid-configuration`, `GET /auth/jwks`.
- Access JWT (RFC 7519) claims: `sub` (user id), `tid` (tenant id), `role`, `scope`, `exp` (15 min). Refresh token rotated, stored hashed in Redis.
- `TenantGuard` extracts `tid` from validated JWT → sets request tenant context (feeds 1.3). `RolesGuard` + `@Roles('admin'|'manager'|'finance'|'member'|'client')` decorator.
- OIDC: configurable upstream IdP; on callback map `email` → `users` row, provision on first login if domain allow-listed.
- Bearer tokens transmitted per RFC 6750 (`Authorization: Bearer`).

**Testing**:
- `Integration: full PKCE flow (authorize → code → token) → valid JWT with tid/role.`
- `Integration: token with tampered signature → 401.`
- `Integration: member calls finance-only endpoint → 403.`
- `Integration: expired access token → 401; valid refresh → new access token, old refresh rejected (rotation).`
- `Integration (mocked OIDC): callback with allow-listed domain → user provisioned; non-allow-listed → 403.`

#### 1.5 — JSONB Validation, Audit Log, OpenAPI Pipeline
**What**: Zod schemas for every JSONB shape, an audit interceptor, and auto-generated OpenAPI 3.1.

**Design**:
- Zod schemas in `packages/types` for `billing_terms`, `revenue_recognition`, `health_score`, `approval_trail`, `custom_fields`, `accounting_sync`, `skills`, `availability_overrides`, workflow `definition`. All writes touching JSONB validate before persistence; invalid → 422 with JSON Pointer path.
- `AuditInterceptor` writes `audit_log` rows (`before`/`after` JSONB, `user_id`, `entity_type/id`, `action`, `ip`) on every mutating request — SOC 2 control.
- `nestjs-zod` + `@nestjs/swagger` emit OpenAPI 3.1 at `/openapi.json`; CI fails if the committed `openapi.json` drifts.

**Testing**:
- `Unit: billing_terms retainer payload → valid; missing monthly_amount → ZodError with path 'monthly_amount'.`
- `Integration: PATCH project with malformed health_score JSONB → 422, no write.`
- `Integration: approve a time entry → audit_log row with before.status='submitted', after.status='approved'.`
- `Snapshot: generated openapi.json matches committed copy.`

---

## Phase 2: Clients, Contracts & Projects

### Purpose
Deliver the structural core of delivery management: clients, contracts (with JSONB billing terms), projects, the task tree, and dependencies. This is the foundation every downstream feature (time, billing, revenue, reporting) references. After this phase a firm can model its book of work even before tracking time.

### Tasks

#### 2.1 — Clients CRUD
**What**: Manage client companies with contacts, billing terms, and CRM linkage.

**Design**: REST under `/clients`. `Client` carries relational `currency_code`, `payment_terms_days`, `tax_id`, `crm_external_id`, `crm_source`, plus JSONB `contacts[]`, `metadata`, `custom_fields` (validated against tenant `custom_field_defs.client`). Endpoints: `POST /clients`, `GET /clients?search=&cursor=`, `GET /clients/:id`, `PATCH /clients/:id`, `DELETE /clients/:id` (soft delete). Cursor pagination (created_at, id).

**Testing**:
- `Integration: create client with contacts[] → 201, contacts validated.`
- `Integration: custom_field not in tenant defs → 422.`
- `Integration: list paginates with stable cursor across 250 rows.`

#### 2.2 — Contracts & Billing Terms
**What**: Contracts with method-specific JSONB `billing_terms` and `revenue_recognition` config.

**Design**: `POST /clients/:id/contracts`, CRUD under `/contracts`. `billing_terms` discriminated union (Zod) on `method`: `time_and_materials` (rate_card, cap_amount, cap_type), `fixed_price` (milestones[]), `retainer` (monthly_amount, hours_included, overage_rate). `revenue_recognition`: `{ standard: 'ASC_606'|'IFRS_15', method, measure, total_estimated_* }`. Amendments appended to JSONB `amendments[]` via `POST /contracts/:id/amendments` (never mutate prior entries — audit trail). `status` transitions guarded.

**Testing**:
- `Unit: fixed_price terms without milestones[] → ZodError.`
- `Integration: add amendment → appended to amendments[], original preserved, audit_log written.`
- `Integration: T&M contract with cap_amount → persisted and retrievable.`

#### 2.3 — Projects
**What**: Projects linked to client + contract, with budget hours/amount and JSONB settings/health_score.

**Design**: CRUD under `/projects`. Fields per Suggestion 3 (`code` unique per tenant, `budget_hours`, `budget_amount`, `health_score` JSONB written only by AI in Phase 8, `settings` override tenant defaults). Status machine: `planning → active → on_hold ↔ active → completed`/`cancelled`. `GET /projects/:id` returns rolled-up `consumed_hours`/`actual_cost` (computed from approved time entries — empty until Phase 4).

**Testing**:
- `Integration: create project with duplicate code per tenant → 409.`
- `Integration: illegal transition completed → active → 409 with allowed-transitions list.`
- `Integration: project under contract of different client → 422.`

#### 2.4 — Tasks & Dependencies
**What**: Hierarchical tasks with JSONB dependency arrays and cycle detection.

**Design**: CRUD under `/projects/:id/tasks`. `ProjectTask` has `parent_task_id` (self-ref tree), `estimated_hours`, `is_billable`, JSONB `dependencies[]` (`{task_id, type: FS|FF|SS|SF, lag_days}`). On dependency add, run DFS cycle check across the project's task graph; reject cycles. Endpoint `POST /tasks/:id/dependencies`, `DELETE /tasks/:id/dependencies/:depId`.

**Testing**:
- `Unit: dependency graph A→B→C, add C→A → cycle error.`
- `Integration: create subtask under parent in different project → 422.`
- `Unit: FS dependency with lag_days=2 stored and returned intact.`

---

## Phase 3: Resources, Skills & Allocation

### Purpose
Model the people who deliver the work: resources, skills (JSONB), roles, rates, availability, and project assignments with conflict detection. Enables capacity planning and feeds utilisation reporting (Phase 6) and AI resource matching (Phase 8).

### Tasks

#### 3.1 — Resources & Skills
**What**: Resource profiles with cost/bill rates, capacity, JSONB skills and availability overrides.

**Design**: CRUD under `/resources`. Per Suggestion 3: `cost_rate`, `default_bill_rate`, `capacity_hours_per_week`, JSONB `skills[]` (`{name, category, proficiency 1-5}`), `availability_overrides[]`, `qualifications[]`, `hr_external_id`. GIN index on `skills` enables `GET /resources?skill=Python&minProficiency=3`. Optional link to `users.id`.

**Testing**:
- `Integration: filter resources by skill+proficiency → only matching resources (GIN index used; verify via EXPLAIN in a perf test).`
- `Unit: proficiency outside 1-5 → 422.`
- `Integration: availability_override array round-trips.`

#### 3.2 — Assignments & Capacity Conflict Detection
**What**: Assign resources to projects/tasks with allocation hours and over-allocation detection.

**Design**: `POST /projects/:id/assignments`. `ResourceAssignment`: `start_date`, `end_date`, `allocated_hours_per_week`, `bill_rate` (defaults to resource `default_bill_rate`), `status: tentative|confirmed|completed`. On confirm, compute sum of confirmed allocations overlapping the date range vs `capacity_hours_per_week` (minus availability_overrides); if over, return `409` with conflict detail (caller may force with `?allowOverallocation=true`). `GET /resources/:id/allocation?from=&to=` returns a week-by-week allocation timeline.

**Testing**:
- `Unit: 40h capacity, existing 30h confirmed, new 20h overlapping → over-allocation conflict (sum 50).`
- `Integration: confirm with allowOverallocation=true → 201 + warning flag.`
- `Unit: non-overlapping date ranges → no conflict.`

---

## Phase 4: Time & Expense Tracking with Approval Workflow

### Purpose
Ship the highest-frequency interaction in any PSA — time and expense capture — plus the configurable approval workflow engine. This is where billable data originates, so it must be correct, auditable, and flexible across firms. After this phase the system holds approved billable records ready for invoicing.

### Tasks

#### 4.1 — Time Entries (web/timer/manual)
**What**: Create, edit, and list time entries with billable flag, rate snapshot, and source tracking.

**Design**: CRUD under `/time-entries`. `TimeEntry` per Suggestion 3: `entry_date`, `hours` (CHECK > 0), `is_billable`, `bill_rate`/`cost_rate` snapshotted at creation from the resource/assignment (immutability of billed amounts), `source: manual|timer|ai_suggestion|calendar`, JSONB `ai_provenance` (null unless AI), JSONB `approval_trail`. Timer endpoints: `POST /time-entries/timer/start`, `POST /time-entries/timer/stop` → creates entry with elapsed hours rounded to tenant `time_entry_increment_minutes`. Bulk weekly `POST /time-entries/bulk`.

**Testing**:
- `Unit: rate snapshot taken at creation; later resource rate change does not alter existing entry.`
- `Unit: timer 1h37m with 15-min increment → 1.5h (rounding rule documented).`
- `Integration: hours <= 0 → 422.`
- `Integration: bulk insert of a week → all rows created atomically; one bad row → whole batch rejected.`

#### 4.2 — Expenses with Receipt Upload
**What**: Expense capture with category, billable flag, and receipt image to object storage.

**Design**: CRUD under `/expenses`. `POST /expenses` returns a pre-signed S3 PUT URL for the receipt; `receipt_url` stored on confirmation. JSONB `metadata` reserved for OCR (later). Multi-currency `amount` + `currency_code`.

**Testing**:
- `Integration (MinIO): request upload URL → PUT receipt → expense.receipt_url resolves.`
- `Integration: billable expense in non-tenant currency → stored with currency_code, flagged for FX at invoice time.`

#### 4.3 — Approval Workflow Engine
**What**: A data-driven state machine (JSONB `workflow_definitions`) governing time/expense approval.

**Design**: `WorkflowDefinition.definition` JSONB = `{states[], transitions[{from,to,action,allowed_roles[],condition}]}` (Suggestion 3 example). Engine API: `POST /time-entries/:id/transition {action}` evaluates allowed transitions for the entity's current status, checks `allowed_roles` against caller role and `condition` (safe expression evaluator over entry fields, e.g. `hours > 8`), applies the transition, appends to `approval_trail`, writes audit_log. Default workflow seeded per tenant: `draft → submitted → approved|rejected`. Rejected entries return to `draft` on resubmit.

**Testing**:
- `Unit: transition not permitted for role → rejected with reason.`
- `Unit: condition hours>8 routes submitted→manager_review; hours<=8 routes submitted→approved.`
- `Integration: approve → status=approved, approval_trail appended, approved_by/at set, audit_log written.`
- `Integration: member tries to approve own entry → 403.`

---

## Phase 5: Billing & Invoicing

### Purpose
Convert approved billable time, expenses, and contract milestones into invoices across T&M, milestone, and retainer modes. This is the revenue-capture engine and the first feature finance teams will judge. Invoices are immutable once finalized (line items snapshotted to JSONB) for audit integrity.

### Tasks

#### 5.1 — Billable Work Aggregation & Draft Invoice
**What**: Gather unbilled approved time/expenses (and due milestones) for a client/contract into a draft invoice.

**Design**: `POST /invoices/draft {client_id, contract_id?, period_start, period_end}`. Logic by `billing_terms.method`:
- **T&M**: sum approved billable time `hours * bill_rate` grouped by resource/role; add billable expenses (FX-converted to invoice currency at period rate); enforce `cap_amount` if set.
- **fixed_price**: include milestones whose status is completed and not yet invoiced.
- **retainer**: monthly_amount line + overage (hours above `hours_included` at `overage_rate`).
Line items written to invoice JSONB `line_items[]` with `source_ids` linking back to time/expense/milestone rows; those source rows marked `invoiced` to prevent double-billing. `subtotal/tax/total` in typed NUMERIC columns. Invoice number from a per-tenant sequence.

**Testing**:
- `Unit: 120h @ 250 + 1,250 expense → subtotal 31,250.`
- `Integration: draft twice for same period → second draft has 0 lines (sources already invoiced).`
- `Unit: T&M with cap 50,000 and 60,000 of work → capped at 50,000, overage flagged.`
- `Unit: retainer 40h included, 50h worked, overage 275 → monthly + 10*275.`

#### 5.2 — Invoice Lifecycle, PDF & CSV Export
**What**: Finalize/send/void invoices, record payments, export PDF and RFC 4180 CSV.

**Design**: `POST /invoices/:id/finalize` (locks line_items, assigns issue/due dates from `payment_terms_days`), `/send`, `/void` (reason required, releases source rows back to unbilled). `POST /invoices/:id/payments`. `GET /invoices/:id/pdf` (server-rendered → S3). `GET /invoices/export.csv` per RFC 4180 (quoted fields, CRLF, UTF-8 BOM optional). Status: `draft→finalized→sent→paid|void`.

**Testing**:
- `Integration: finalize then edit line_items → 409 (immutable).`
- `Integration: void → source time entries return to unbilled, audit_log written.`
- `Unit: CSV with comma/quote/newline in description → RFC 4180-compliant escaping.`
- `Integration: payment >= total → status paid.`

---

## Phase 6: Reporting, Utilisation & Profitability

### Purpose
Surface the dashboards PSA buyers select on: resource utilisation (billable vs non-billable), real-time project/client/portfolio profitability, and realisation/effective-bill-rate analytics — visible to PMs without finance involvement. Built as denormalised read models (Suggestion 2's projection idea, applied pragmatically) refreshed by workers.

### Tasks

#### 6.1 — Utilisation Read Model
**What**: Per-resource, per-week billable/non-billable utilisation with a bench view.

**Design**: Materialised table `rm_resource_utilisation` (Suggestion 2 shape). A BullMQ scheduled job (every 15 min + on time-entry approval) recomputes affected resource-weeks: `billable_hours`, `non_billable_hours`, `capacity_hours`, `utilisation_pct = billable/capacity`. `GET /reports/utilisation?from=&to=&team=`; `GET /reports/bench` lists resources < 50% allocated with their skills.

**Testing**:
- `Integration: approve 30 billable + 10 non-billable on 40h capacity → utilisation 75%.`
- `Integration: approving a new entry enqueues recompute for that resource-week only.`
- `Integration: bench returns resources < 50% allocated.`

#### 6.2 — Profitability & Realisation
**What**: Project/client/portfolio profitability with margin, realisation rate, effective bill rate.

**Design**: `rm_project_summary` (Suggestion 2): `budget_hours/consumed_hours`, `actual_cost = Σ hours*cost_rate`, `invoiced_amount`, `revenue` (from Phase 7 once available, else invoiced), `margin = (revenue - actual_cost)/revenue`, `realisation = billed_value / standard_value`, `effective_bill_rate = billed_revenue / billable_hours`. `GET /reports/profitability?level=project|client|portfolio`. Computed via worker job + Redis cache; portfolio heatmap data endpoint for the UI.

**Testing**:
- `Unit: revenue 100k, cost 60k → margin 0.40.`
- `Unit: billed 90k vs standard 100k → realisation 0.90.`
- `Integration: profitability rolls up project→client→portfolio with matching totals.`

---

## Phase 7: Revenue Recognition (ASC 606 / IFRS 15)

### Purpose
Implement the differentiating, financial-grade capability incumbents charge enterprise prices for: automated revenue recognition under the shared ASC 606 / IFRS 15 five-step model. Produces an auditable revenue ledger with full calculation provenance. Standard is a per-contract configuration option (the two are substantively identical per `standards.md`).

### Tasks

#### 7.1 — Recognition Engine
**What**: Compute recognised vs deferred revenue per contract per period for the three methods.

**Design**: Engine over `contracts.revenue_recognition` config. Methods:
- **percentage_of_completion**: `pct = completed_measure / total_estimated_measure` (measure = hours or cost); `cumulative_recognised = pct * contract_value`; `period_amount = cumulative - prior_cumulative`.
- **milestone**: recognise milestone `amount` when status=completed in that period.
- **straight_line**: `contract_value / months` per period (retainers).
Writes `revenue_schedules` rows with typed `recognised_amount`/`deferred_amount` and JSONB `calculation` (Suggestion 3 audit shape: method, measure, completed, percentage, recognised_to_date). `POST /revenue/recognize {contract_id, period_end}` (idempotent per contract+period). All math via `decimal.js`; rounding to currency minor units with documented half-even rule.

**Testing**:
- `Unit: POC 900/2000 hrs on 500k contract → cumulative 225k; prior 180k → period 45k.`
- `Unit: milestone completed this period → recognise exactly its amount.`
- `Unit: straight_line 120k / 12 months → 10k/month, last period absorbs rounding remainder.`
- `Integration: re-run same period → idempotent (no duplicate schedule, amounts unchanged).`
- `Fixture: committed ASC 606 worked-example scenarios reproduce expected ledger exactly.`

#### 7.2 — Revenue Ledger & Contract Amendment Handling
**What**: Queryable revenue ledger and retroactive recognition adjustment on contract amendment.

**Design**: `rm_revenue_ledger` projection (cumulative recognised/deferred per contract-line-period). On a contract amendment that changes value or estimate, recompute cumulative expected recognition and post a catch-up/reversal adjustment in the current period (never mutate posted periods — append adjustment row). `GET /revenue/ledger?contract_id=`. `POST /revenue/post {period}` flips schedules pending→posted (sets `posted_at`); posted periods are locked.

**Testing**:
- `Integration: amend contract value 500k→600k mid-project → current-period catch-up adjustment, prior posted periods unchanged.`
- `Integration: attempt to recompute a posted period → 409.`
- `Integration: ledger cumulative equals sum of period amounts.`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the AI capabilities that justify the platform's positioning: project health scoring, AI-suggested time entries from calendar/task activity, resource-match recommendations, and AI-drafted client status reports. Each is additive and reads from the data the prior phases produce.

### Tasks

#### 8.1 — LLM Provider Abstraction & Prompt Library
**What**: A provider-agnostic AI client with prompt-cached system templates and structured output.

**Design**: `packages/ai` exposes `complete({system, user, schema})` returning Zod-validated structured output (tool-use / JSON mode). Anthropic Claude default; system prompts marked for prompt caching. Per-tenant budget guardrails + audit of every AI call (prompt hash, tokens, cost) into `audit_log`. All AI features run as BullMQ jobs.

**Testing**:
- `Unit (mocked provider): malformed model JSON → retried once then surfaced as AI error, never persisted.`
- `Unit: provider response validated against output Zod schema.`

#### 8.2 — Project Health Scoring
**What**: Compute a delivery-risk score and factors, written to `projects.health_score` JSONB.

**Design**: Scheduled job per active project gathers features (budget burn % vs completion %, tasks behind `due_date`, over-allocation, milestone slippage) → deterministic rule-based score first, then LLM narrative of `factors[]` with severity. Output schema matches Suggestion 3 `health_score` (`score`, `risk_level`, `factors[]`, `updated_at`). Expression index on `risk_level` powers `GET /reports/at-risk-projects`.

**Testing**:
- `Unit: 67% budget at 45% completion → factor budget_burn severity warning.`
- `Integration: scoring job writes valid health_score JSONB; at-risk endpoint filters via index.`
- `Unit (mocked LLM): narrative generation failure → numeric score still persisted, narrative omitted.`

#### 8.3 — AI Time-Entry Suggestions (calendar + task activity)
**What**: Suggest billable time entries from calendar events to reduce under-capture.

**Design**: Pull calendar events (RFC 5545 / Google/MS APIs) for a resource's day; map event titles/attendees to projects via heuristics + LLM; produce `draft` suggested entries (`source=ai_suggestion`, `ai_provenance` with `calendar_event_id`, `confidence`, `original_suggestion`). User confirms/edits before submit. `GET /time-entries/suggestions?date=`.

**Testing**:
- `Integration (mocked calendar): 3 events → 3 draft suggestions with ai_provenance populated.`
- `Unit: low-confidence (<0.5) suggestion flagged for review, not auto-created above draft.`
- `Integration: confirming a suggestion clears ai_suggested→manual edit trail intact.`

#### 8.4 — Resource-Match Recommendations & AI Status Reports
**What**: Recommend resources for a project's needs; draft client status reports.

**Design**:
- **Match**: `GET /projects/:id/resource-recommendations` ranks resources by skill match (JSONB skills vs project skill needs), availability (Phase 3.2), and prior client history; LLM explains top picks. (SQL/JSONB implementation now; graph optimisation deferred to Phase 10.)
- **Status report**: `POST /projects/:id/status-report` summarises milestone progress, risks (from health_score), and next steps into client-ready prose; stored as a draft for PM review before portal publish.

**Testing**:
- `Integration: project needing Python@3 + availability → ranks a 40h-free Python@4 resource above a fully-allocated Python@5.`
- `Integration (mocked LLM): status report references actual milestone names + current risk_level.`
- `Unit: recommendation excludes inactive and over-allocated resources.`

---

## Phase 9: Integrations (CRM & Accounting) + Webhooks

### Purpose
Close the CRM-to-delivery-to-billing loop — the dominant value proposition. Inbound CRM webhooks auto-create projects from closed-won deals; outbound sync pushes invoices to QuickBooks/Xero. Built on a common adapter interface so additional providers are incremental.

### Tasks

#### 9.1 — Inbound Webhook Gateway & Integration Events
**What**: A signed, idempotent webhook receiver persisting raw payloads to `integration_events`.

**Design**: `POST /webhooks/:source` (`salesforce|hubspot|quickbooks|xero`). Verify provider signature (HMAC/Basic per provider — Rocketlane-style Basic, QBO HMAC); reject invalid with 401. Persist raw payload to `integration_events` (status `received`), return 2xx fast, enqueue processing. Idempotency via provider event id in Redis.

**Testing**:
- `Integration: valid signature → 200, integration_event persisted, job enqueued.`
- `Integration: invalid signature → 401, nothing persisted.`
- `Integration: duplicate event id → processed once.`

#### 9.2 — CRM Deal-to-Project Handoff
**What**: On CRM closed-won, create client + contract + project from a template.

**Design**: `CrmAdapter` interface (`fetchDeal`, `mapToClient`, `mapToContract`); Salesforce + HubSpot implementations (OAuth 2.0 client credentials / private app token). Processor: closed-won deal → upsert client (`crm_external_id`), create contract from deal value, instantiate project from `project_templates.template_data`, optionally enqueue AI resource recommendations. Saga steps mirror Suggestion 2's CRM handoff saga.

**Testing**:
- `Integration (mocked SF API + fixture payload): closed-won deal → client+contract+project created, crm_external_id set.`
- `Integration: deal for existing client (matching crm_external_id) → reuses client, no duplicate.`
- `Integration: re-delivered webhook → no duplicate project (idempotent).`

#### 9.3 — Accounting Invoice Sync (QuickBooks & Xero)
**What**: Push finalized invoices to QuickBooks/Xero and reconcile payment status.

**Design**: `AccountingAdapter` (`upsertCustomer`, `upsertInvoice`, `fetchPayments`). On `InvoiceFinalized`, enqueue sync; write result to invoice JSONB `accounting_sync` (`system`, `external_id`, `sync_status`, `last_synced`). Retry with exponential backoff; on terminal failure flag for finance. Inbound accounting webhook updates payment status. OAuth 2.0 token storage encrypted per tenant.

**Testing**:
- `Integration (mocked QBO): finalize invoice → upsert customer+invoice, accounting_sync.external_id set, status synced.`
- `Integration: transient 500 → retried with backoff; succeeds on retry.`
- `Integration: QBO payment webhook → invoice status → paid, payment row created.`
- `Integration: Xero adapter passes the same adapter contract tests as QBO.`

---

## Phase 10: Web App, Client Portal, MCP Server & Hardening

### Purpose
Expose everything through human and AI interfaces and harden for production. Delivers the internal dashboard, the first-class client portal (a stated differentiator), the differentiating MCP server, and the compliance/ops work (RBAC review, GDPR erasure, CSV/iCal export, observability) needed for real deployments.

### Tasks

#### 10.1 — Internal Web Dashboard
**What**: Next.js app for PMs/finance: projects, timesheets, approvals, invoicing, reports.

**Design**: Next.js 16 App Router + shadcn/ui. Server components hit the API with the user's JWT. Key screens: timesheet grid (weekly entry + timer), approval queue, project board with health badges, utilisation/bench view, profitability heatmap, invoice builder. PPR/Cache Components for dashboards; optimistic updates for time entry.

**Testing**:
- `E2E (Playwright): consultant logs week of time → submits → manager approves → entry shows approved.`
- `E2E: finance drafts invoice from approved work → finalizes → PDF downloads.`
- `E2E: RBAC — member cannot see finance/profitability nav.`

#### 10.2 — Client Portal
**What**: Read-only (plus approvals) client-facing project portal without internal-tool access.

**Design**: Separate route group + `client` role JWT scoped to a single client's projects. Shows milestone status, shared documents, published AI status reports, and milestone/invoice approval actions. No cost/margin/internal-rate data ever exposed (enforced by serializer allow-list + RLS).

**Testing**:
- `E2E: client user sees only their client's projects; cross-client access → 403.`
- `Integration: portal serializer never includes cost_rate/margin fields (allow-list test).`
- `E2E: client approves a milestone → status updates, triggers billing eligibility.`

#### 10.3 — MCP Server
**What**: An MCP server exposing PSA read tools and drafting tools to AI assistants.

**Design**: `apps/mcp` using `@modelcontextprotocol/sdk`. Tools: `list_at_risk_projects`, `get_project_status`, `get_resource_availability`, `get_billing_pipeline`, `draft_status_report(project_id)`. Auth via per-user PSA token mapped to tenant + role; every tool call passes through the same RBAC + RLS path as the REST API. Read-only by default; drafting tools return content without committing.

**Testing**:
- `Integration: list_at_risk_projects returns only caller-tenant projects with risk_level high/medium.`
- `Integration: tool call with member-scoped token cannot reach finance pipeline tool → authorization error.`
- `Unit: tool input/output validated against declared JSON Schema.`

#### 10.4 — Compliance, Export & Hardening
**What**: GDPR erasure, encryption posture, full CSV/iCal export, rate limiting, observability.

**Design**: `POST /gdpr/erase {subject_id}` redacts PII while preserving financial aggregates (right-to-erasure vs audit retention). Encryption at rest (PG + S3 SSE) and in transit (TLS). Per-tenant rate limiting (Redis). RFC 4180 CSV export for time/invoices/reports; RFC 5545 iCal feed for resource schedules/milestones. OpenTelemetry traces + Pino logs shipped. Final RBAC matrix review.

**Testing**:
- `Integration: erase subject → PII columns redacted, time_entry hours/amounts retained for audit.`
- `Integration: iCal feed validates against an RFC 5545 parser.`
- `Integration: exceed rate limit → 429 with Retry-After.`
- `Security: automated check that no endpoint returns cross-tenant rows (RLS smoke across all list endpoints).`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB+RLS, auth, audit, OpenAPI)  ── required by everything
    │
Phase 2: Clients, Contracts, Projects, Tasks  ── requires 1
    │
    ├── Phase 3: Resources, Skills, Allocation       ── requires 2  ┐ can parallel
    │                                                                │
    └── Phase 4: Time & Expense + Approval Workflow   ── requires 2,3┘
             │
             ├── Phase 5: Billing & Invoicing          ── requires 4 (+2)
             │       │
             │       └── Phase 7: Revenue Recognition  ── requires 5 (+2)
             │
             └── Phase 6: Reporting/Utilisation/Profit ── requires 4,5  (can parallel Phase 7)
                      │
Phase 8: AI-Native Layer        ── requires 2,3,4,6 (health, suggestions, match, reports)
Phase 9: Integrations + Webhooks ── requires 2,5 (CRM→project, invoice→accounting); can parallel Phase 8
Phase 10: Web/Portal/MCP/Hardening ── requires all prior (UI surfaces every capability)
```

**Parallelism opportunities**
- After Phase 2: **Phase 3** and the early scaffolding of **Phase 4** can proceed together (4 depends on 3 for rate snapshots, so finish 3.1 first).
- After Phase 5: **Phase 6 (reporting)** and **Phase 7 (revenue recognition)** are independent and can be built concurrently.
- **Phase 8 (AI)** and **Phase 9 (integrations)** are independent of each other and can run in parallel once their data dependencies (Phases 2–6 / Phases 2 & 5 respectively) are met.
- Within Phase 10, **MCP server (10.3)** and **Client Portal (10.2)** can be built in parallel by separate developers.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; integration tests run against real Postgres + Redis via Testcontainers.
3. `eslint` and `prettier --check` pass with no errors.
4. `tsc --noEmit` passes across the workspace (no type errors).
5. `docker compose up` builds and starts all affected services with healthchecks green.
6. The phase's headline capability works end-to-end (demonstrated by at least one E2E or integration scenario).
7. Every new JSONB shape has a Zod schema and is validated on write.
8. New REST endpoints appear in the generated `openapi.json` (drift check passes); new MCP tools declare JSON Schemas.
9. Prisma migration(s) created, reversible where feasible, and applied cleanly to a fresh database.
10. RLS verified: no list/read endpoint added in the phase can return cross-tenant rows.
11. Mutating endpoints write `audit_log` entries (SOC 2 control).
12. Money math uses `decimal.js`/NUMERIC — no floating-point currency anywhere in the phase's code.
13. New config/environment variables documented in `.env.example`.
```
