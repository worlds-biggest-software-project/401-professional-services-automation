# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: 401 — Professional Services Automation
> Approach: PostgreSQL with strategic JSONB columns for flexible schema areas

---

## Summary

A pragmatic hybrid architecture that uses PostgreSQL as a single data store, combining normalised relational tables for core financial and operational data with JSONB columns for areas that require schema flexibility. The relational backbone handles the entities where referential integrity and financial precision are non-negotiable (contracts, invoices, revenue recognition, time entries), while JSONB columns absorb the variability found in project templates, custom fields, approval workflow configurations, integration metadata, and AI-generated content. This approach avoids the operational complexity of a polyglot persistence strategy (separate document database) while still providing the flexibility that a multi-vertical PSA platform requires.

---

## Design Principles

1. **Relational for money.** Any field that appears on an invoice, feeds revenue recognition, or drives a financial report is a typed, indexed relational column -- never buried in JSON.
2. **JSONB for variability.** Custom fields, configurable workflow definitions, integration payloads, AI suggestions, and vertical-specific metadata live in JSONB columns.
3. **Single database.** PostgreSQL handles both relational and document workloads. No separate MongoDB or DynamoDB instance to operate, back up, or keep in sync.
4. **Typed views over JSONB.** Where JSONB data is frequently queried, create generated columns or expression indexes to provide typed, indexed access without sacrificing flexibility.

---

## Core Schema

### 1. Organisation and Tenancy

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    -- JSONB: tenant-level configuration that varies by deployment
    settings        JSONB NOT NULL DEFAULT '{
        "default_currency": "USD",
        "fiscal_year_start_month": 1,
        "time_entry_increment_minutes": 15,
        "approval_required": true,
        "revenue_recognition_standard": "ASC_606",
        "billing_day_of_month": 1
    }',
    -- JSONB: custom field definitions for this tenant
    custom_field_defs JSONB NOT NULL DEFAULT '{
        "project": [],
        "client": [],
        "resource": [],
        "time_entry": []
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 2. Clients

```sql
CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    -- Relational: core billing and financial fields
    billing_address TEXT,
    tax_id          VARCHAR(50),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    -- Relational: CRM linkage
    crm_external_id VARCHAR(255),
    crm_source      VARCHAR(50),
    -- JSONB: flexible metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- JSONB: tenant-defined custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- JSONB: contact persons (variable structure per client)
    contacts        JSONB NOT NULL DEFAULT '[]',
    /*  contacts example:
        [
            { "name": "Jane Doe", "email": "jane@acme.com", "role": "billing", "phone": "+1..." },
            { "name": "Bob Smith", "email": "bob@acme.com", "role": "project_sponsor" }
        ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3. Contracts and Billing Terms

```sql
CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    contract_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    start_date      DATE NOT NULL,
    end_date        DATE,
    total_value     NUMERIC(14,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB: contract terms that vary by billing method and vertical
    billing_terms   JSONB NOT NULL DEFAULT '{}',
    /*  billing_terms examples:

        Time & Materials:
        { "method": "time_and_materials", "rate_card_id": "uuid", "cap_amount": 50000, "cap_type": "hard" }

        Fixed Price with milestones:
        { "method": "fixed_price", "milestones": [
            { "id": "uuid", "name": "Discovery Complete", "amount": 15000, "due_date": "2026-04-15" },
            { "id": "uuid", "name": "MVP Delivered", "amount": 35000, "due_date": "2026-07-01" }
        ]}

        Retainer:
        { "method": "retainer", "monthly_amount": 10000, "hours_included": 40, "overage_rate": 275 }
    */
    -- JSONB: revenue recognition configuration
    revenue_recognition JSONB NOT NULL DEFAULT '{}',
    /*  { "standard": "ASC_606", "method": "percentage_of_completion", "measure": "hours",
          "total_estimated_hours": 2000, "total_estimated_cost": 300000 } */
    -- JSONB: amendment history (append-only within the JSON array)
    amendments      JSONB NOT NULL DEFAULT '[]',
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, contract_number)
);
```

### 4. Projects and Tasks

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    contract_id     UUID REFERENCES contracts(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'planning',
    start_date      DATE,
    target_end_date DATE,
    budget_hours    NUMERIC(10,2),
    budget_amount   NUMERIC(14,2),
    -- JSONB: AI-generated health scoring and risk assessment
    health_score    JSONB NOT NULL DEFAULT '{}',
    /*  { "score": 72, "risk_level": "medium", "factors": [
            { "type": "budget_burn", "severity": "warning", "detail": "67% budget consumed at 45% completion" },
            { "type": "schedule_slip", "severity": "info", "detail": "2 tasks behind schedule" }
        ], "updated_at": "2026-05-20T14:30:00Z" } */
    -- JSONB: project-specific settings that override tenant defaults
    settings        JSONB NOT NULL DEFAULT '{}',
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE project_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    parent_task_id  UUID REFERENCES project_tasks(id),
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'not_started',
    estimated_hours NUMERIC(8,2),
    start_date      DATE,
    due_date        DATE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: task-level metadata (assignee preferences, labels, custom fields)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- JSONB: dependencies stored as structured array within the task
    dependencies    JSONB NOT NULL DEFAULT '[]'
    /*  [
            { "task_id": "uuid", "type": "FS", "lag_days": 0 },
            { "task_id": "uuid", "type": "SS", "lag_days": 2 }
        ] */
);
```

### 5. Resources and Skills

```sql
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    display_name    VARCHAR(255) NOT NULL,
    resource_type   VARCHAR(30) NOT NULL DEFAULT 'employee',
    role            VARCHAR(100),
    cost_rate       NUMERIC(10,2),
    default_bill_rate NUMERIC(10,2),
    capacity_hours_per_week NUMERIC(5,2) DEFAULT 40.00,
    -- JSONB: skills with proficiency (avoids join table for simple skill matching)
    skills          JSONB NOT NULL DEFAULT '[]',
    /*  [
            { "name": "Python", "category": "Programming", "proficiency": 4 },
            { "name": "AWS", "category": "Cloud", "proficiency": 3 },
            { "name": "Project Management", "category": "Soft Skills", "proficiency": 5 }
        ] */
    -- JSONB: certifications, education, HR-system metadata
    qualifications  JSONB NOT NULL DEFAULT '[]',
    -- JSONB: availability overrides (holidays, part-time weeks)
    availability_overrides JSONB NOT NULL DEFAULT '[]',
    /*  [
            { "start": "2026-06-15", "end": "2026-06-20", "type": "leave", "hours_per_day": 0 },
            { "start": "2026-07-01", "end": "2026-07-31", "type": "reduced", "hours_per_day": 4 }
        ] */
    hr_external_id  VARCHAR(255),
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE resource_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    task_id         UUID REFERENCES project_tasks(id),
    role            VARCHAR(100),
    start_date      DATE NOT NULL,
    end_date        DATE,
    allocated_hours_per_week NUMERIC(5,2),
    bill_rate       NUMERIC(10,2),
    status          VARCHAR(30) NOT NULL DEFAULT 'tentative'
);
```

### 6. Time and Expense Tracking

```sql
CREATE TABLE time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    project_id      UUID NOT NULL REFERENCES projects(id),
    task_id         UUID REFERENCES project_tasks(id),
    entry_date      DATE NOT NULL,
    hours           NUMERIC(5,2) NOT NULL CHECK (hours > 0),
    description     TEXT,
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    bill_rate       NUMERIC(10,2),
    cost_rate       NUMERIC(10,2),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    -- JSONB: AI suggestion provenance (if AI-generated)
    ai_provenance   JSONB,
    /*  { "source": "calendar", "calendar_event_id": "abc123",
          "confidence": 0.87, "suggested_at": "2026-05-20T09:00:00Z",
          "original_suggestion": { "hours": 2.0, "project_id": "uuid" } } */
    -- JSONB: approval workflow trail
    approval_trail  JSONB NOT NULL DEFAULT '[]',
    /*  [
            { "action": "submitted", "by": "uuid", "at": "2026-05-20T17:00:00Z" },
            { "action": "rejected", "by": "uuid", "at": "2026-05-21T09:30:00Z", "reason": "Wrong project code" },
            { "action": "resubmitted", "by": "uuid", "at": "2026-05-21T10:00:00Z" },
            { "action": "approved", "by": "uuid", "at": "2026-05-21T11:00:00Z" }
        ] */
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    project_id      UUID NOT NULL REFERENCES projects(id),
    expense_date    DATE NOT NULL,
    category        VARCHAR(100) NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    description     TEXT,
    receipt_url     VARCHAR(2048),
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- JSONB: receipt OCR data, policy validation results
    metadata        JSONB NOT NULL DEFAULT '{}',
    approval_trail  JSONB NOT NULL DEFAULT '[]',
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7. Invoicing

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    contract_id     UUID REFERENCES contracts(id),
    invoice_number  VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    issue_date      DATE NOT NULL,
    due_date        DATE NOT NULL,
    subtotal        NUMERIC(14,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(14,2) NOT NULL DEFAULT 0,
    total           NUMERIC(14,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    -- JSONB: line items stored as document (denormalised for invoice immutability)
    line_items      JSONB NOT NULL DEFAULT '[]',
    /*  [
            { "description": "Senior Consultant - May 2026", "hours": 120, "rate": 250,
              "amount": 30000, "type": "time", "project_id": "uuid", "source_ids": ["uuid", "uuid"] },
            { "description": "Travel expenses - Client site visit", "amount": 1250,
              "type": "expense", "source_ids": ["uuid"] },
            { "description": "Milestone: Discovery Complete", "amount": 15000,
              "type": "milestone", "milestone_id": "uuid" }
        ] */
    -- JSONB: accounting system sync state
    accounting_sync JSONB NOT NULL DEFAULT '{}',
    /*  { "system": "quickbooks", "external_id": "INV-4521", "last_synced": "2026-05-20T12:00:00Z",
          "sync_status": "synced" } */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, invoice_number)
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(14,2) NOT NULL,
    payment_date    DATE NOT NULL,
    payment_method  VARCHAR(50),
    reference       VARCHAR(255),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 8. Revenue Recognition

```sql
CREATE TABLE revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    contract_id     UUID NOT NULL REFERENCES contracts(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    recognised_amount NUMERIC(14,2) NOT NULL,
    deferred_amount NUMERIC(14,2) NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- JSONB: calculation details for audit trail
    calculation     JSONB NOT NULL DEFAULT '{}',
    /*  { "method": "percentage_of_completion", "measure": "hours",
          "total_estimated": 2000, "completed": 900, "percentage": 45.0,
          "total_contract_value": 500000, "recognised_to_date": 225000 } */
    posted_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 9. Workflow Definitions

```sql
CREATE TABLE workflow_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    entity_type     VARCHAR(50) NOT NULL,    -- 'time_entry', 'expense', 'invoice'
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: entire workflow definition stored as document
    definition      JSONB NOT NULL,
    /*  {
            "states": ["draft", "submitted", "manager_review", "finance_review", "approved", "rejected"],
            "transitions": [
                { "from": "draft", "to": "submitted", "action": "submit", "allowed_roles": ["member"] },
                { "from": "submitted", "to": "manager_review", "action": "auto", "condition": "hours > 8" },
                { "from": "submitted", "to": "approved", "action": "approve", "allowed_roles": ["manager"],
                  "condition": "hours <= 8" },
                { "from": "manager_review", "to": "approved", "action": "approve", "allowed_roles": ["manager"] },
                { "from": "manager_review", "to": "rejected", "action": "reject", "allowed_roles": ["manager"] }
            ]
        } */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10. Integration Events and Audit Log

```sql
CREATE TABLE integration_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source          VARCHAR(50) NOT NULL,     -- 'salesforce', 'hubspot', 'quickbooks', 'xero'
    event_type      VARCHAR(100) NOT NULL,
    -- JSONB: full webhook/event payload preserved as-is
    payload         JSONB NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'received',
    processed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    -- JSONB: before/after snapshots for change tracking
    changes         JSONB NOT NULL DEFAULT '{}',
    /*  { "before": { "status": "draft", "hours": 4.0 },
          "after":  { "status": "submitted", "hours": 4.5 } } */
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Indexing Strategy for JSONB Columns

```sql
-- GIN index on custom_fields for tenant-defined field queries
CREATE INDEX idx_projects_custom_fields ON projects USING GIN (custom_fields);
CREATE INDEX idx_time_entries_custom_fields ON time_entries USING GIN (custom_fields);

-- Expression index for AI health score queries
CREATE INDEX idx_projects_risk_level ON projects ((health_score->>'risk_level'))
    WHERE health_score->>'risk_level' IS NOT NULL;

-- GIN index on resource skills for skill-based matching
CREATE INDEX idx_resources_skills ON resources USING GIN (skills);

-- Expression index for accounting sync status
CREATE INDEX idx_invoices_sync_status ON invoices ((accounting_sync->>'sync_status'));

-- Composite index for tenant-scoped queries (RLS performance)
CREATE INDEX idx_time_entries_tenant_date ON time_entries (tenant_id, entry_date);
CREATE INDEX idx_time_entries_tenant_resource ON time_entries (tenant_id, resource_id, entry_date);
CREATE INDEX idx_invoices_tenant_status ON invoices (tenant_id, status, issue_date);
```

---

## Where Relational Wins vs. Where JSONB Wins

| Data Category | Storage | Rationale |
|--------------|---------|-----------|
| Invoice totals, amounts, dates | Relational columns | Financial precision, aggregation, referential integrity |
| Time entry hours, rates, dates | Relational columns | High-frequency queries, SUM/AVG aggregations, billing calculations |
| Revenue recognition amounts | Relational columns | Audit compliance, period-end reporting, GL integration |
| Contract billing terms | JSONB | Varies by billing method (T&M, fixed, retainer, block hours) |
| Custom fields | JSONB | Tenant-defined, unknown at schema design time |
| Approval workflow trails | JSONB | Variable-length event arrays, append-only within a row |
| AI suggestions and provenance | JSONB | Evolving schema as AI capabilities expand |
| Integration payloads | JSONB | Preserve third-party webhook data as-is |
| Workflow definitions | JSONB | Complex state machine definitions that vary by tenant |
| Resource skills and qualifications | JSONB | Variable number of skills, avoids many-to-many join tables |
| Project health scores | JSONB | AI-generated, schema evolves with model improvements |

---

## Pros

- **Single database, no polyglot overhead.** One PostgreSQL instance to operate, back up, monitor, and scale. No data synchronisation between relational and document stores.
- **Flexibility where it matters.** Custom fields, workflow definitions, and integration payloads can evolve without schema migrations. New AI features can store structured data in JSONB columns immediately.
- **Financial integrity preserved.** All monetary amounts, dates, and foreign keys remain in typed relational columns with referential integrity. No risk of financial data buried in untyped JSON.
- **Query performance.** GIN and expression indexes on JSONB columns provide fast lookups. Standard relational indexes handle the high-frequency queries (time entry aggregation, invoice summaries).
- **Tenant customisation.** The `custom_field_defs` and `custom_fields` pattern lets each tenant define their own fields without schema changes, enabling vertical-specific deployments (legal, IT consulting, creative agency).
- **Simpler than event sourcing.** The development team uses familiar CRUD patterns for most operations. JSONB `approval_trail` arrays provide an audit log within each record without requiring a full event-sourcing infrastructure.

## Cons

- **JSONB is not self-documenting.** The structure of JSONB columns is not enforced by the database. Application-layer validation (JSON Schema, Zod, Pydantic) is required to prevent malformed data.
- **JSONB query limitations.** While PostgreSQL JSONB queries are powerful, complex aggregations across JSONB fields are slower and harder to write than equivalent relational queries. If a JSONB field is queried heavily, it should be promoted to a relational column.
- **Hybrid complexity.** Developers must make ongoing decisions about whether new data belongs in a relational column or JSONB. Without clear guidelines, the boundary drifts and the schema becomes inconsistent.
- **Invoice line items in JSONB.** Storing invoice line items as a JSONB array sacrifices the ability to query individual line items with standard SQL joins. If line-item-level reporting is required (e.g., "total billed for task X across all invoices"), a separate `invoice_lines` relational table may be needed alongside the JSONB.
- **Migration from JSONB to relational.** If a JSONB field is promoted to a relational column later, migrating existing data requires careful backfill scripts.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ (JSONB, GIN indexes, RLS, generated columns) |
| JSONB validation | JSON Schema at application layer (Zod for TypeScript, Pydantic for Python) |
| Connection pooling | PgBouncer |
| Migrations | Prisma Migrate or Alembic with JSONB-aware migration scripts |
| Full-text search | PostgreSQL tsvector for basic search; Elasticsearch for advanced faceted search |
| Caching | Redis for session data and hot query results |
| Multi-tenancy | Row-Level Security with tenant_id on all tenant-scoped tables |

---

## Migration and Scaling Considerations

- **JSONB-to-relational promotion path.** Establish a clear threshold: if a JSONB field is used in WHERE clauses, JOINs, or aggregations in more than 3 queries, promote it to a relational column. Use PostgreSQL generated columns to extract frequently accessed JSONB fields without breaking existing queries.
- **Partitioning.** Partition `time_entries`, `expenses`, and `audit_log` by date range. Partition `integration_events` by date to enable fast pruning of old webhook data.
- **Read replicas.** Offload reporting and dashboard queries to streaming replicas. The JSONB columns replicate identically.
- **Custom field indexing.** When a tenant defines a custom field that is used in filters or sorts, create a partial expression index dynamically: `CREATE INDEX ON projects ((custom_fields->>'department')) WHERE tenant_id = 'uuid';`
- **Horizontal scaling.** Citus distributed PostgreSQL can shard by `tenant_id`. JSONB columns distribute with their parent rows, requiring no special handling.
- **JSONB size monitoring.** Set alerts for JSONB column sizes exceeding thresholds (e.g., `approval_trail` arrays with 100+ entries). Consider archiving or compacting very large JSONB values.
