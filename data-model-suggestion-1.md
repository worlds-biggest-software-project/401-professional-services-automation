# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: 401 — Professional Services Automation
> Approach: Fully normalized relational schema with PostgreSQL

---

## Summary

A traditional, fully normalized relational database design using PostgreSQL as the primary data store. Every entity is represented as a dedicated table with explicit foreign keys, referential integrity constraints, and a clear separation of concerns between domain areas (projects, resources, billing, time tracking). This is the most proven and well-understood approach for enterprise financial applications, and aligns with how incumbent PSA platforms (Kantata, Certinia, Autotask) structure their data models.

---

## Core Entity Groups

### 1. Organisation and Tenancy

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
```

### 2. Clients and Contracts

```sql
CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    billing_address TEXT,
    tax_id          VARCHAR(50),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    crm_external_id VARCHAR(255),       -- Salesforce/HubSpot ID
    crm_source      VARCHAR(50),        -- 'salesforce' | 'hubspot'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, contract_number)
);

CREATE TABLE contract_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id),
    project_id      UUID REFERENCES projects(id),
    billing_method  VARCHAR(30) NOT NULL,  -- 'time_and_materials' | 'fixed_price' | 'retainer'
    budget_amount   NUMERIC(14,2),
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE contract_milestones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_line_id UUID NOT NULL REFERENCES contract_lines(id),
    name            VARCHAR(255) NOT NULL,
    amount          NUMERIC(14,2) NOT NULL,
    due_date        DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    invoiced_at     TIMESTAMPTZ
);
```

### 3. Projects and Tasks

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'planning',
    start_date      DATE,
    target_end_date DATE,
    budget_hours    NUMERIC(10,2),
    budget_amount   NUMERIC(14,2),
    template_id     UUID REFERENCES project_templates(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE project_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    parent_task_id  UUID REFERENCES project_tasks(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'not_started',
    priority        SMALLINT NOT NULL DEFAULT 3,
    estimated_hours NUMERIC(8,2),
    start_date      DATE,
    due_date        DATE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_billable     BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE task_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    predecessor_id  UUID NOT NULL REFERENCES project_tasks(id),
    successor_id    UUID NOT NULL REFERENCES project_tasks(id),
    dependency_type VARCHAR(10) NOT NULL DEFAULT 'FS',  -- FS, FF, SS, SF
    lag_days        INTEGER NOT NULL DEFAULT 0,
    UNIQUE(predecessor_id, successor_id)
);

CREATE TABLE project_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    template_data   JSONB NOT NULL      -- serialised task structure
);
```

### 4. Resources and Skills

```sql
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    display_name    VARCHAR(255) NOT NULL,
    resource_type   VARCHAR(30) NOT NULL DEFAULT 'employee',  -- employee, contractor, generic
    role_id         UUID REFERENCES resource_roles(id),
    cost_rate       NUMERIC(10,2),          -- internal cost per hour
    default_bill_rate NUMERIC(10,2),
    capacity_hours_per_week NUMERIC(5,2) DEFAULT 40.00,
    department      VARCHAR(100),
    location        VARCHAR(100),
    hr_external_id  VARCHAR(255),           -- Workday / BambooHR ID
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE resource_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    default_bill_rate NUMERIC(10,2),
    UNIQUE(tenant_id, name)
);

CREATE TABLE skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(100),
    UNIQUE(tenant_id, name)
);

CREATE TABLE resource_skills (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    skill_id        UUID NOT NULL REFERENCES skills(id),
    proficiency     SMALLINT CHECK (proficiency BETWEEN 1 AND 5),
    PRIMARY KEY (resource_id, skill_id)
);

CREATE TABLE resource_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    task_id         UUID REFERENCES project_tasks(id),
    role_id         UUID REFERENCES resource_roles(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    allocated_hours_per_week NUMERIC(5,2),
    bill_rate       NUMERIC(10,2),
    status          VARCHAR(30) NOT NULL DEFAULT 'tentative'
);
```

### 5. Time and Expense Tracking

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
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, submitted, approved, rejected
    submitted_at    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    ai_suggested    BOOLEAN NOT NULL DEFAULT false,
    source          VARCHAR(30) DEFAULT 'manual',           -- manual, timer, ai_suggestion, calendar
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
    submitted_at    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 6. Billing and Invoicing

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
    accounting_external_id VARCHAR(255),  -- QuickBooks / Xero ID
    accounting_sync_status VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, invoice_number)
);

CREATE TABLE invoice_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    project_id      UUID REFERENCES projects(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(10,2),
    unit_price      NUMERIC(10,2),
    amount          NUMERIC(14,2) NOT NULL,
    line_type       VARCHAR(30) NOT NULL,  -- 'time' | 'expense' | 'milestone' | 'retainer' | 'adjustment'
    time_entry_id   UUID REFERENCES time_entries(id),
    expense_id      UUID REFERENCES expenses(id),
    milestone_id    UUID REFERENCES contract_milestones(id),
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(14,2) NOT NULL,
    payment_date    DATE NOT NULL,
    payment_method  VARCHAR(50),
    reference       VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7. Revenue Recognition (ASC 606 / IFRS 15)

```sql
CREATE TABLE revenue_recognition_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_line_id UUID NOT NULL REFERENCES contract_lines(id),
    standard        VARCHAR(10) NOT NULL,     -- 'ASC_606' | 'IFRS_15'
    method          VARCHAR(40) NOT NULL,     -- 'percentage_of_completion' | 'milestone' | 'straight_line'
    measure         VARCHAR(30),              -- 'hours' | 'cost' | 'output_units'
    total_estimated NUMERIC(14,2)
);

CREATE TABLE revenue_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES revenue_recognition_rules(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    recognised_amount NUMERIC(14,2) NOT NULL,
    deferred_amount NUMERIC(14,2) NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    posted_at       TIMESTAMPTZ
);
```

---

## Multi-Tenancy Strategy

Row-level security (RLS) with a `tenant_id` column on every tenant-scoped table. PostgreSQL RLS policies enforce isolation at the database level:

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

Composite indexes with `tenant_id` as the leading column on all major query paths ensure RLS does not degrade performance.

---

## Key Relationships (ERD Summary)

```
tenant ──< client ──< contract ──< contract_line ──< contract_milestone
                  └──< project ──< project_task ──< task_dependency
                                └──< resource_assignment
                                └──< time_entry
                                └──< expense
                  └──< invoice ──< invoice_line
tenant ──< resource ──< resource_skill
                    └──< resource_assignment
                    └──< time_entry
                    └──< expense
```

---

## Pros

- **Proven at scale.** Normalised relational models are the foundation of every successful ERP and PSA system in production today. PostgreSQL handles billions of rows with proper indexing.
- **Referential integrity.** Foreign keys enforce data consistency at the database level, preventing orphaned time entries, dangling invoice references, or broken contract relationships.
- **Strong tooling ecosystem.** Mature ORM support (SQLAlchemy, Prisma, TypeORM), migration tools (Flyway, Alembic, Prisma Migrate), and reporting/BI tools (Metabase, Looker) work seamlessly with normalised schemas.
- **Financial audit compliance.** ASC 606 / IFRS 15 revenue recognition requires traceable links between contracts, performance obligations, and recognised revenue. A normalised schema makes these relationships explicit and auditable.
- **Multi-tenant RLS.** PostgreSQL row-level security provides database-enforced tenant isolation without application-layer bugs leaking data.
- **Straightforward reporting.** Standard SQL joins produce the profitability dashboards, utilisation reports, and billing summaries that PSA buyers expect.

## Cons

- **Schema rigidity.** Every new field or entity requires a migration. Adding custom fields for different client verticals (legal vs. IT consulting vs. creative agency) means either nullable columns or a separate custom-fields mechanism.
- **Complex joins for analytics.** Real-time profitability queries that span contracts, time entries, expenses, and revenue schedules can require 6-8 table joins, which may need materialised views or pre-aggregated summary tables.
- **Impedance mismatch with flexible workflows.** Approval workflows, custom statuses, and configurable billing rules are harder to represent in rigid relational tables than in document-style structures.
- **Migration overhead at scale.** Schema migrations on large tables (hundreds of millions of time entries) require careful planning (online DDL, zero-downtime migrations) to avoid locking production tables.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Primary database | PostgreSQL 16+ |
| Connection pooling | PgBouncer |
| Migrations | Alembic (Python) or Prisma Migrate (TypeScript) |
| Full-text search | PostgreSQL tsvector or Elasticsearch sidecar |
| Caching | Redis for session/rate data; materialised views for dashboards |
| Multi-tenancy | Row-Level Security with tenant_id |
| Backup/PITR | pg_basebackup + WAL archiving |

---

## Migration and Scaling Considerations

- **Partitioning.** Partition `time_entries`, `expenses`, and `revenue_schedules` by date range (monthly or quarterly) to keep individual partition sizes manageable and enable fast archival of historical data.
- **Read replicas.** Offload reporting and analytics queries to streaming read replicas to protect transactional write performance.
- **Materialised views.** Pre-compute utilisation rates, project profitability, and billing summaries as materialised views refreshed on a schedule (every 5-15 minutes) to avoid expensive real-time aggregation.
- **Citus for horizontal scaling.** If single-node PostgreSQL becomes a bottleneck, Citus distributed PostgreSQL can shard by `tenant_id`, preserving the relational model while distributing data across nodes.
- **Zero-downtime migrations.** Use tools like `pg-osc` (online schema change) or Reshape for adding columns, indexes, or constraints on large tables without locking.
