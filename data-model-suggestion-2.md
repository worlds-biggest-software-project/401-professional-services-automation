# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: 401 — Professional Services Automation
> Approach: Event Sourcing with Command Query Responsibility Segregation (CQRS)

---

## Summary

An event-sourced architecture where every state change in the PSA system is captured as an immutable event in an append-only event store. The current state of any aggregate (project, time sheet, invoice, resource allocation) is derived by replaying its event stream. CQRS separates the write model (commands that produce events) from read models (projections optimised for specific query patterns like utilisation dashboards, profitability reports, and billing summaries). This approach provides a complete audit trail -- critical for financial compliance (ASC 606 / IFRS 15) and regulatory accountability -- and enables powerful temporal queries ("what was the project budget on March 15th?").

---

## Architecture Overview

```
┌─────────────────────┐    ┌──────────────┐    ┌─────────────────────┐
│   Command Handlers  │───>│  Event Store  │───>│   Event Processors  │
│  (Write Side)       │    │  (append-only)│    │   (Projectors)      │
└─────────────────────┘    └──────────────┘    └─────────────────────┘
                                                         │
                                                         ▼
                                               ┌─────────────────────┐
                                               │   Read Models       │
                                               │   (Query Side)      │
                                               │   - PostgreSQL      │
                                               │   - Elasticsearch   │
                                               │   - Redis caches    │
                                               └─────────────────────┘
```

---

## Event Store Schema

### Core Event Store Tables

```sql
-- The immutable, append-only event log
CREATE TABLE events (
    global_sequence   BIGSERIAL PRIMARY KEY,
    event_id          UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
    aggregate_type    VARCHAR(50) NOT NULL,    -- 'Project', 'TimeSheet', 'Invoice', 'Contract', 'Resource'
    aggregate_id      UUID NOT NULL,
    tenant_id         UUID NOT NULL,
    event_type        VARCHAR(100) NOT NULL,
    event_version     INTEGER NOT NULL,         -- aggregate version for optimistic concurrency
    payload           JSONB NOT NULL,
    metadata          JSONB NOT NULL DEFAULT '{}',  -- user_id, correlation_id, causation_id
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_id, event_version)
);

-- Snapshots to avoid replaying full event streams for large aggregates
CREATE TABLE snapshots (
    aggregate_type    VARCHAR(50) NOT NULL,
    aggregate_id      UUID NOT NULL,
    version           INTEGER NOT NULL,
    state             JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, version)
);

-- Projection checkpoints: track how far each projector has consumed
CREATE TABLE projection_checkpoints (
    projection_name   VARCHAR(100) PRIMARY KEY,
    last_sequence     BIGINT NOT NULL DEFAULT 0,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Indexes for Event Replay and Subscription

```sql
CREATE INDEX idx_events_aggregate ON events (aggregate_id, event_version);
CREATE INDEX idx_events_type_sequence ON events (aggregate_type, global_sequence);
CREATE INDEX idx_events_tenant ON events (tenant_id, global_sequence);
CREATE INDEX idx_events_event_type ON events (event_type, global_sequence);
```

---

## Domain Events by Aggregate

### Project Aggregate

```
ProjectCreated          { project_id, client_id, name, code, start_date, budget_hours, budget_amount }
ProjectStatusChanged    { project_id, old_status, new_status, reason }
ProjectBudgetUpdated    { project_id, old_budget, new_budget, approved_by }
TaskAdded               { project_id, task_id, name, estimated_hours, parent_task_id }
TaskStatusChanged       { project_id, task_id, old_status, new_status }
TaskDependencyAdded     { project_id, predecessor_id, successor_id, type, lag_days }
ResourceAssigned        { project_id, resource_id, role_id, start_date, end_date, bill_rate }
ResourceUnassigned      { project_id, resource_id, reason }
DeliveryRiskFlagged     { project_id, risk_level, risk_factors[], ai_confidence }
```

### TimeSheet Aggregate

```
TimeSheetCreated        { timesheet_id, resource_id, week_start_date }
TimeEntryRecorded       { timesheet_id, entry_id, project_id, task_id, date, hours, is_billable, description }
TimeEntryUpdated        { timesheet_id, entry_id, old_hours, new_hours, reason }
TimeEntryDeleted        { timesheet_id, entry_id, reason }
TimeSheetSubmitted      { timesheet_id, submitted_by, total_hours }
TimeSheetApproved       { timesheet_id, approved_by, approved_entries[] }
TimeSheetRejected       { timesheet_id, rejected_by, reason, rejected_entries[] }
AITimeEntrySuggested    { timesheet_id, entry_id, source, confidence, calendar_event_id }
```

### Contract Aggregate

```
ContractCreated         { contract_id, client_id, title, start_date, total_value }
ContractLineAdded       { contract_id, line_id, billing_method, budget_amount, project_id }
MilestoneAdded          { contract_id, line_id, milestone_id, name, amount, due_date }
MilestoneCompleted      { contract_id, milestone_id, completed_date }
ContractAmended         { contract_id, amendment_number, changes[], effective_date }
ContractSigned          { contract_id, signed_by, signed_date }
```

### Invoice Aggregate

```
InvoiceDrafted          { invoice_id, client_id, contract_id, line_items[], subtotal, tax, total }
InvoiceLineAdded        { invoice_id, line_id, description, amount, source_type, source_id }
InvoiceFinalized        { invoice_id, invoice_number, issue_date, due_date }
InvoiceSent             { invoice_id, sent_to, sent_at }
PaymentReceived         { invoice_id, payment_id, amount, payment_date, method }
InvoiceVoided           { invoice_id, reason, voided_by }
AccountingSynced        { invoice_id, external_system, external_id, sync_timestamp }
```

### Revenue Recognition Aggregate

```
RecognitionRuleCreated  { rule_id, contract_line_id, standard, method, total_estimated }
RevenueRecognised       { rule_id, period_start, period_end, amount, cumulative_percentage }
RevenueDeferred         { rule_id, period, deferred_amount, reason }
RecognitionAdjusted     { rule_id, old_amount, new_amount, adjustment_reason, period }
```

### Resource Aggregate

```
ResourceOnboarded       { resource_id, name, type, role_id, cost_rate, bill_rate, capacity }
ResourceRoleChanged     { resource_id, old_role, new_role, effective_date }
ResourceRateUpdated     { resource_id, old_rate, new_rate, rate_type, effective_date }
SkillAdded              { resource_id, skill_id, proficiency }
SkillUpdated            { resource_id, skill_id, old_proficiency, new_proficiency }
ResourceDeactivated     { resource_id, reason, effective_date }
```

---

## Read Model Projections

Each projection is a denormalised PostgreSQL table (or Elasticsearch index) optimised for a specific query pattern.

### Projection: Project Dashboard

```sql
CREATE TABLE rm_project_summary (
    project_id          UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    client_name         VARCHAR(255),
    project_name        VARCHAR(255),
    status              VARCHAR(30),
    budget_hours        NUMERIC(10,2),
    consumed_hours      NUMERIC(10,2),
    budget_amount       NUMERIC(14,2),
    actual_cost         NUMERIC(14,2),
    invoiced_amount     NUMERIC(14,2),
    risk_level          VARCHAR(20),
    team_size           INTEGER,
    completion_pct      NUMERIC(5,2),
    last_updated        TIMESTAMPTZ
);
```

### Projection: Resource Utilisation

```sql
CREATE TABLE rm_resource_utilisation (
    resource_id         UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    week_start          DATE NOT NULL,
    capacity_hours      NUMERIC(5,2),
    billable_hours      NUMERIC(5,2),
    non_billable_hours  NUMERIC(5,2),
    utilisation_pct     NUMERIC(5,2),
    allocated_projects  INTEGER,
    PRIMARY KEY (resource_id, week_start)
);
```

### Projection: Billing Pipeline

```sql
CREATE TABLE rm_billing_pipeline (
    tenant_id           UUID NOT NULL,
    client_id           UUID NOT NULL,
    project_id          UUID NOT NULL,
    month               DATE NOT NULL,
    unbilled_time_amount NUMERIC(14,2),
    unbilled_expense_amount NUMERIC(14,2),
    pending_milestones_amount NUMERIC(14,2),
    draft_invoice_amount NUMERIC(14,2),
    sent_invoice_amount NUMERIC(14,2),
    collected_amount    NUMERIC(14,2),
    PRIMARY KEY (tenant_id, client_id, project_id, month)
);
```

### Projection: Revenue Recognition Ledger

```sql
CREATE TABLE rm_revenue_ledger (
    tenant_id           UUID NOT NULL,
    contract_id         UUID NOT NULL,
    contract_line_id    UUID NOT NULL,
    period              DATE NOT NULL,
    standard            VARCHAR(10),
    recognised_revenue  NUMERIC(14,2),
    deferred_revenue    NUMERIC(14,2),
    cumulative_recognised NUMERIC(14,2),
    PRIMARY KEY (tenant_id, contract_line_id, period)
);
```

---

## Saga / Process Managers

Long-running business processes are coordinated by sagas that react to events and issue compensating commands:

### Invoice Generation Saga

```
1. Listen: TimeSheetApproved → accumulate approved billable entries
2. Listen: MilestoneCompleted → check if milestone billing triggered
3. On billing period end → Command: DraftInvoice
4. Listen: InvoiceFinalized → Command: SyncToAccounting
5. On sync failure → retry with backoff, alert finance team
```

### CRM-to-Project Handoff Saga

```
1. Listen: CRMDealClosed (webhook event) → Command: CreateContract
2. Listen: ContractSigned → Command: CreateProject (from template)
3. Listen: ProjectCreated → Command: AllocateResources (AI-recommended)
4. On allocation failure → Command: NotifyResourceManager
```

### Revenue Recognition Saga

```
1. Listen: TimeSheetApproved / MilestoneCompleted → recalculate completion percentage
2. Command: RecogniseRevenue for current period
3. Listen: ContractAmended → Command: AdjustRecognition retroactively
```

---

## Pros

- **Complete audit trail.** Every change to every entity is permanently recorded. Financial auditors can trace any invoice amount back through every approval, time entry edit, and rate change that produced it. This is the gold standard for ASC 606 / IFRS 15 compliance.
- **Temporal queries.** "What was the project budget on March 15th?" or "What was this resource's bill rate when this time entry was approved?" are trivial -- just replay events up to that point.
- **Retroactive corrections.** Contract amendments or rate corrections can be applied by appending new events rather than mutating historical records. The original data is never lost.
- **Optimised read models.** Each dashboard, report, or API query reads from a purpose-built projection, eliminating expensive multi-table joins. Utilisation dashboards read from `rm_resource_utilisation`, not from joining time entries with resources with projects.
- **Independent scalability.** The write side (event store) and read side (projections) can scale independently. Heavy reporting load does not impact command processing.
- **Event-driven integrations.** CRM sync, accounting sync, and AI processing naturally subscribe to event streams rather than polling for changes.

## Cons

- **Significantly higher complexity.** Event sourcing requires designing aggregates, events, projections, and sagas -- a fundamentally different programming model from CRUD. The team must understand eventual consistency, idempotency, and compensating actions.
- **Eventual consistency.** Read models are updated asynchronously. A user who submits a timesheet may not see it reflected in the utilisation dashboard for 100-500ms. For financial reports this delay must be carefully managed.
- **Event schema evolution.** As the domain evolves, event schemas change. Older events must still be deserialisable. This requires event versioning strategies (upcasting, schema registry) that add operational overhead.
- **Debugging difficulty.** Diagnosing a bug in a read model requires tracing through the event stream and the projector logic, rather than simply querying a table.
- **Storage growth.** The event store grows monotonically. A PSA system tracking 500 consultants generating 20+ time entries per week produces millions of events per year. Snapshots mitigate replay cost but the storage itself is unbounded.
- **Overkill for simpler domains.** Reference data (skills, roles, resource profiles) does not benefit from event sourcing and is better modelled as standard CRUD tables.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL (events table) or EventStoreDB |
| Event bus | Apache Kafka or NATS JetStream |
| Read model database | PostgreSQL for structured projections |
| Search projections | Elasticsearch for full-text search across projects, clients, time entries |
| Caching layer | Redis for hot projections (current utilisation, active project status) |
| Saga orchestration | Custom process managers or Temporal.io for durable workflow orchestration |
| Event schema registry | Confluent Schema Registry or custom JSON Schema validation |
| Serialisation | JSON (JSONB in PostgreSQL) with event type versioning |

---

## Migration and Scaling Considerations

- **Gradual adoption.** Not all aggregates need event sourcing from day one. Start with the financial aggregates (Invoice, Contract, Revenue Recognition) where the audit trail delivers the most value, and use standard CRUD for reference data (skills, roles, templates).
- **Snapshot strategy.** Take snapshots every N events (e.g., every 100 events per aggregate) to bound replay time. For long-lived project aggregates that accumulate thousands of events over months, snapshots are essential.
- **Projection rebuild.** Every read model must be rebuildable from scratch by replaying the event store. This is a powerful capability (fix a bug in a projector, replay, and the read model self-corrects) but requires the event store to be the durable source of truth.
- **Event store partitioning.** Partition the events table by `tenant_id` or by month to manage table size and enable tenant-level data isolation for GDPR compliance (data deletion becomes partition drop).
- **Archival.** Events older than the regulatory retention period (typically 7 years for financial records) can be archived to cold storage (S3 + Parquet) while keeping the hot event store lean.
- **CQRS without event sourcing.** If the team finds full event sourcing too complex, a lighter CQRS approach can still be used: separate write and read databases with change data capture (Debezium) feeding denormalised read replicas. This captures many of the read-side scaling benefits without the event sourcing complexity.
