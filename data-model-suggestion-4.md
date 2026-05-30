# Data Model Suggestion 4: Relational Core + Graph Layer for Resource Intelligence

> Project: 401 — Professional Services Automation
> Approach: PostgreSQL relational core for operations and finance, with a Neo4j graph layer for resource matching, skill-based allocation, and organisational intelligence

---

## Summary

This specialty approach addresses one of the most complex and differentiating aspects of Professional Services Automation: intelligent resource allocation and skill matching. The architecture uses PostgreSQL as the operational and financial data store (handling contracts, billing, time tracking, and revenue recognition), paired with a Neo4j graph database that models the rich, interconnected relationships between people, skills, projects, clients, roles, and availability. Graph queries that would require 6-8 table joins and recursive CTEs in SQL become single-hop traversals in Cypher. This dual-database approach is inspired by real-world implementations at organisations like NASA, which uses Neo4j for talent marketplace and staffing optimisation across programs.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Project &    │  │  Billing &   │  │  Resource     │  │
│  │  Time Mgmt   │  │  Finance     │  │  Intelligence │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬────────┘  │
│         │                 │                  │           │
│  ┌──────▼─────────────────▼──────┐  ┌───────▼────────┐  │
│  │     PostgreSQL (Primary)      │  │   Neo4j Graph   │  │
│  │  - Contracts & invoicing      │  │  - People       │  │
│  │  - Time & expense entries     │  │  - Skills       │  │
│  │  - Revenue recognition        │  │  - Projects     │  │
│  │  - Project tasks              │  │  - Clients      │  │
│  │  - Audit log                  │  │  - Assignments  │  │
│  └───────────────────────────────┘  │  - Availability │  │
│                                     └────────────────┘  │
│         ▲                                  ▲            │
│         └──────── Sync Service ────────────┘            │
└─────────────────────────────────────────────────────────┘
```

**Data flow:** PostgreSQL is the system of record. When resources, projects, skills, or assignments change in PostgreSQL, a sync service (Change Data Capture via Debezium, or application-level events) propagates the relevant data to Neo4j. The graph database is a read-optimised projection of the resource intelligence domain.

---

## PostgreSQL Schema (Operational Core)

The PostgreSQL schema follows the normalised relational approach from Suggestion 1 for all operational and financial entities. The tables below highlight the resource-related entities that feed the graph layer.

### Resources (PostgreSQL - System of Record)

```sql
CREATE TABLE resources (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id),
    user_id                 UUID REFERENCES users(id),
    display_name            VARCHAR(255) NOT NULL,
    resource_type           VARCHAR(30) NOT NULL DEFAULT 'employee',
    department              VARCHAR(100),
    location                VARCHAR(100),
    cost_rate               NUMERIC(10,2),
    default_bill_rate       NUMERIC(10,2),
    capacity_hours_per_week NUMERIC(5,2) DEFAULT 40.00,
    seniority_level         VARCHAR(30),           -- junior, mid, senior, principal, director
    hire_date               DATE,
    hr_external_id          VARCHAR(255),
    is_active               BOOLEAN NOT NULL DEFAULT true,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE resource_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    default_bill_rate NUMERIC(10,2),
    UNIQUE(tenant_id, name)
);

CREATE TABLE resource_role_assignments (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    role_id         UUID NOT NULL REFERENCES resource_roles(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    PRIMARY KEY (resource_id, role_id, effective_from)
);

CREATE TABLE skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(100),
    UNIQUE(tenant_id, name)
);

CREATE TABLE skill_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    parent_id       UUID REFERENCES skill_categories(id),
    UNIQUE(tenant_id, name)
);

CREATE TABLE resource_skills (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    skill_id        UUID NOT NULL REFERENCES skills(id),
    proficiency     SMALLINT NOT NULL CHECK (proficiency BETWEEN 1 AND 5),
    certified       BOOLEAN NOT NULL DEFAULT false,
    last_used_date  DATE,
    PRIMARY KEY (resource_id, skill_id)
);

CREATE TABLE resource_certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    name            VARCHAR(255) NOT NULL,
    issuing_body    VARCHAR(255),
    issued_date     DATE,
    expiry_date     DATE,
    credential_id   VARCHAR(255)
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
    status          VARCHAR(30) NOT NULL DEFAULT 'tentative',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE resource_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_id     UUID NOT NULL REFERENCES resources(id),
    date            DATE NOT NULL,
    available_hours NUMERIC(5,2) NOT NULL,
    reason          VARCHAR(100),   -- 'leave', 'public_holiday', 'training', 'bench'
    UNIQUE(resource_id, date)
);

-- Tracks which resources have worked with which clients (for relationship matching)
CREATE TABLE resource_client_history (
    resource_id     UUID NOT NULL REFERENCES resources(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    project_count   INTEGER NOT NULL DEFAULT 0,
    total_hours     NUMERIC(10,2) NOT NULL DEFAULT 0,
    last_project_date DATE,
    satisfaction_score NUMERIC(3,2),    -- client feedback score 1.0-5.0
    PRIMARY KEY (resource_id, client_id)
);
```

### Project Skill Requirements (PostgreSQL)

```sql
CREATE TABLE project_skill_requirements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id),
    skill_id        UUID NOT NULL REFERENCES skills(id),
    role_id         UUID REFERENCES resource_roles(id),
    min_proficiency SMALLINT NOT NULL DEFAULT 3,
    required_from   DATE,
    required_to     DATE,
    hours_per_week  NUMERIC(5,2),
    priority        VARCHAR(20) NOT NULL DEFAULT 'required',  -- required, preferred, nice_to_have
    filled_by       UUID REFERENCES resources(id)
);
```

---

## Neo4j Graph Model

### Node Types

```cypher
// Person (resource) node - synced from PostgreSQL resources table
CREATE (r:Resource {
    id: 'uuid',
    tenant_id: 'uuid',
    name: 'Alice Chen',
    type: 'employee',
    department: 'Consulting',
    location: 'London',
    seniority: 'senior',
    cost_rate: 85.00,
    bill_rate: 225.00,
    capacity_hours_per_week: 40,
    is_active: true
})

// Skill node
CREATE (s:Skill {
    id: 'uuid',
    name: 'Python',
    category: 'Programming'
})

// Role node
CREATE (role:Role {
    id: 'uuid',
    name: 'Technical Lead',
    default_bill_rate: 275.00
})

// Project node
CREATE (p:Project {
    id: 'uuid',
    name: 'CRM Migration Phase 2',
    code: 'ACME-CRM-P2',
    status: 'active',
    start_date: date('2026-03-01'),
    end_date: date('2026-09-30'),
    budget_hours: 2000
})

// Client node
CREATE (c:Client {
    id: 'uuid',
    name: 'Acme Corp',
    industry: 'Financial Services'
})

// Department node
CREATE (d:Department {
    id: 'uuid',
    name: 'Data Engineering'
})

// SkillCategory node (hierarchical)
CREATE (sc:SkillCategory {
    id: 'uuid',
    name: 'Cloud Platforms'
})
```

### Relationship Types

```cypher
// Resource HAS_SKILL with proficiency and recency
(r:Resource)-[:HAS_SKILL {proficiency: 4, certified: true, last_used: date('2026-04-15')}]->(s:Skill)

// Resource HAS_ROLE with effective dates
(r:Resource)-[:HAS_ROLE {is_primary: true, effective_from: date('2025-01-01')}]->(role:Role)

// Resource ASSIGNED_TO project with allocation details
(r:Resource)-[:ASSIGNED_TO {
    start_date: date('2026-03-01'),
    end_date: date('2026-06-30'),
    hours_per_week: 32,
    bill_rate: 250.00,
    status: 'confirmed'
}]->(p:Project)

// Resource BELONGS_TO department
(r:Resource)-[:BELONGS_TO]->(d:Department)

// Resource LOCATED_IN location
(r:Resource)-[:LOCATED_IN]->(l:Location)

// Resource WORKED_WITH client (historical relationship strength)
(r:Resource)-[:WORKED_WITH {
    project_count: 3,
    total_hours: 480,
    last_date: date('2026-01-15'),
    satisfaction: 4.5
}]->(c:Client)

// Resource REPORTS_TO manager
(r:Resource)-[:REPORTS_TO]->(m:Resource)

// Resource MENTORS another resource
(r:Resource)-[:MENTORS]->(junior:Resource)

// Project REQUIRES_SKILL with minimum proficiency
(p:Project)-[:REQUIRES_SKILL {min_proficiency: 3, priority: 'required', hours_per_week: 20}]->(s:Skill)

// Project FOR_CLIENT
(p:Project)-[:FOR_CLIENT]->(c:Client)

// Project REQUIRES_ROLE
(p:Project)-[:REQUIRES_ROLE {count: 2, hours_per_week: 40}]->(role:Role)

// Skill PART_OF category (hierarchical)
(s:Skill)-[:PART_OF]->(sc:SkillCategory)
(sc:SkillCategory)-[:CHILD_OF]->(parent:SkillCategory)
```

---

## Key Graph Queries

### 1. Find Best-Fit Resources for a Project

"Find available senior consultants with Python and AWS skills (proficiency 3+) who have worked with Financial Services clients before."

```cypher
MATCH (p:Project {id: $projectId})-[:REQUIRES_SKILL]->(reqSkill:Skill)
WITH p, collect(reqSkill) AS requiredSkills

MATCH (r:Resource {is_active: true})
WHERE r.tenant_id = $tenantId
  AND r.capacity_hours_per_week > 0

// Check skill match
WITH r, p, requiredSkills,
     [(r)-[hs:HAS_SKILL]->(s:Skill) WHERE s IN requiredSkills | {skill: s, proficiency: hs.proficiency}] AS matchedSkills

WHERE size(matchedSkills) > 0

// Check availability (not over-allocated)
OPTIONAL MATCH (r)-[a:ASSIGNED_TO]->(otherProject:Project)
WHERE a.status = 'confirmed'
  AND a.end_date >= date()
WITH r, p, matchedSkills,
     coalesce(sum(a.hours_per_week), 0) AS allocated_hours

WHERE allocated_hours < r.capacity_hours_per_week

// Check client history (bonus for prior relationship)
OPTIONAL MATCH (r)-[ww:WORKED_WITH]->(c:Client)<-[:FOR_CLIENT]-(p)
WITH r, matchedSkills, allocated_hours, ww.satisfaction AS client_satisfaction

// Score and rank
RETURN r.name AS resource_name,
       r.id AS resource_id,
       r.seniority AS seniority,
       size(matchedSkills) AS skills_matched,
       avg([ms IN matchedSkills | ms.proficiency]) AS avg_proficiency,
       r.capacity_hours_per_week - allocated_hours AS available_hours,
       coalesce(client_satisfaction, 0) AS prior_client_score
ORDER BY skills_matched DESC, avg_proficiency DESC, prior_client_score DESC, available_hours DESC
LIMIT 10
```

### 2. Utilisation and Bench View

"Show all resources on the bench (less than 50% allocated) with their skills."

```cypher
MATCH (r:Resource {is_active: true, tenant_id: $tenantId})
OPTIONAL MATCH (r)-[a:ASSIGNED_TO]->(p:Project)
WHERE a.status = 'confirmed' AND a.end_date >= date()
WITH r, coalesce(sum(a.hours_per_week), 0) AS allocated,
     collect(p.name) AS current_projects
WHERE allocated < r.capacity_hours_per_week * 0.5

MATCH (r)-[hs:HAS_SKILL]->(s:Skill)
WITH r, allocated, current_projects,
     collect({skill: s.name, proficiency: hs.proficiency}) AS skills
RETURN r.name, r.department, r.seniority,
       r.capacity_hours_per_week AS capacity,
       allocated,
       r.capacity_hours_per_week - allocated AS available,
       current_projects, skills
ORDER BY available DESC
```

### 3. Team Composition Analysis

"For project X, show the team's combined skill coverage vs. what the project requires."

```cypher
MATCH (p:Project {id: $projectId})-[:REQUIRES_SKILL {priority: 'required'}]->(reqSkill:Skill)
OPTIONAL MATCH (r:Resource)-[a:ASSIGNED_TO]->(p)
WHERE a.status = 'confirmed'
OPTIONAL MATCH (r)-[hs:HAS_SKILL]->(reqSkill)
RETURN reqSkill.name AS required_skill,
       collect(CASE WHEN hs IS NOT NULL THEN {resource: r.name, proficiency: hs.proficiency} END) AS covered_by,
       CASE WHEN size(collect(hs)) = 0 THEN 'GAP' ELSE 'COVERED' END AS coverage_status
ORDER BY coverage_status DESC, reqSkill.name
```

### 4. Organisational Network for Knowledge Transfer

"Find who can mentor a junior developer in Kubernetes skills."

```cypher
MATCH (s:Skill {name: 'Kubernetes'})
MATCH (expert:Resource)-[hs:HAS_SKILL]->(s)
WHERE hs.proficiency >= 4 AND expert.is_active = true
  AND expert.tenant_id = $tenantId

OPTIONAL MATCH (expert)-[:BELONGS_TO]->(d:Department)
OPTIONAL MATCH (expert)-[:LOCATED_IN]->(l:Location)

RETURN expert.name, expert.seniority, hs.proficiency, hs.certified,
       d.name AS department, l.name AS location
ORDER BY hs.proficiency DESC, hs.certified DESC
```

### 5. Resource Allocation Conflict Detection

"Find resources assigned to overlapping projects that exceed their capacity."

```cypher
MATCH (r:Resource {is_active: true, tenant_id: $tenantId})
MATCH (r)-[a:ASSIGNED_TO]->(p:Project)
WHERE a.status = 'confirmed' AND a.end_date >= date()
WITH r, sum(a.hours_per_week) AS total_allocated,
     collect({project: p.name, hours: a.hours_per_week, end: a.end_date}) AS assignments
WHERE total_allocated > r.capacity_hours_per_week
RETURN r.name, r.capacity_hours_per_week AS capacity,
       total_allocated, total_allocated - r.capacity_hours_per_week AS over_allocated,
       assignments
ORDER BY over_allocated DESC
```

---

## Synchronisation Strategy

### PostgreSQL to Neo4j Sync

```
PostgreSQL (writes) ──> Debezium CDC ──> Kafka ──> Neo4j Sink Connector
```

**Alternative (simpler):** Application-level events. When the application writes to PostgreSQL, it also publishes a domain event to a message queue (NATS, RabbitMQ), and a Neo4j sync worker consumes these events to update the graph.

### Sync Rules

| PostgreSQL Entity | Neo4j Node/Relationship | Sync Trigger |
|-------------------|------------------------|--------------|
| `resources` | `(:Resource)` | INSERT, UPDATE, DELETE |
| `resource_skills` | `(:Resource)-[:HAS_SKILL]->(:Skill)` | INSERT, UPDATE, DELETE |
| `resource_role_assignments` | `(:Resource)-[:HAS_ROLE]->(:Role)` | INSERT, UPDATE, DELETE |
| `skills` | `(:Skill)` | INSERT, UPDATE |
| `resource_roles` | `(:Role)` | INSERT, UPDATE |
| `projects` | `(:Project)` | INSERT, UPDATE |
| `resource_assignments` | `(:Resource)-[:ASSIGNED_TO]->(:Project)` | INSERT, UPDATE, DELETE |
| `clients` | `(:Client)` | INSERT, UPDATE |
| `resource_client_history` | `(:Resource)-[:WORKED_WITH]->(:Client)` | INSERT, UPDATE |
| `project_skill_requirements` | `(:Project)-[:REQUIRES_SKILL]->(:Skill)` | INSERT, UPDATE, DELETE |

### Consistency Model

- **Eventual consistency** between PostgreSQL and Neo4j (typically < 500ms lag).
- PostgreSQL is always the system of record. If the graph is stale, the worst outcome is a slightly outdated resource recommendation -- not a financial error.
- Graph can be fully rebuilt from PostgreSQL data at any time (disaster recovery).

---

## Pros

- **Superior resource matching.** Graph traversals for "find a senior Python developer with AWS certification who has worked with this client before and is available next month" are natural in Cypher and perform in milliseconds, even across thousands of resources. The same query in SQL requires multiple joins, subqueries, and is difficult to extend.
- **Relationship-rich queries.** Mentorship chains, team composition analysis, skill gap detection, and organisational network analysis are first-class operations in a graph database.
- **AI integration.** Graph data is an ideal substrate for AI-driven resource recommendations. Neo4j's graph algorithms (PageRank, community detection, similarity) can identify resource clusters, predict good team compositions, and surface hidden expertise.
- **Financial integrity preserved.** All financial operations (billing, invoicing, revenue recognition, time tracking) remain in PostgreSQL with full ACID guarantees and referential integrity. The graph never handles money.
- **Extensible knowledge model.** New relationship types (MENTORS, CERTIFIED_IN, PREVIOUSLY_MANAGED_BY, SPEAKS_LANGUAGE) can be added to the graph without schema migrations. This flexibility is critical for a PSA platform serving diverse professional services verticals.
- **Real-world validation.** NASA, professional services firms, and HR tech companies use graph databases for exactly this pattern: talent matching, resource allocation, and organisational intelligence.

## Cons

- **Operational complexity.** Two databases to operate, monitor, back up, and scale. The team must maintain expertise in both PostgreSQL and Neo4j.
- **Data synchronisation overhead.** Keeping Neo4j in sync with PostgreSQL adds infrastructure (CDC pipeline or event bus) and introduces eventual consistency. Sync failures or lag can cause stale recommendations.
- **Cost.** Neo4j Enterprise requires a commercial licence for production use. Neo4j Community Edition has clustering limitations. Alternatives like Apache AGE (graph extension for PostgreSQL) reduce cost but sacrifice query performance at scale.
- **Smaller talent pool.** Fewer developers know Cypher and graph modelling compared to SQL. Hiring and onboarding are harder.
- **Not needed for small deployments.** For a firm with 20-50 consultants, the PostgreSQL resource tables with simple joins are sufficient. The graph layer adds value primarily at scale (200+ resources, complex skill taxonomies, multi-project allocation).
- **Dual schema maintenance.** Changes to the resource model require updates in both PostgreSQL tables and Neo4j node/relationship definitions.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Operational database | PostgreSQL 16+ (all financial, time tracking, and billing data) |
| Graph database | Neo4j 5.x (resource intelligence layer) |
| Sync pipeline | Debezium + Kafka + Neo4j Kafka Connector, or application-level events via NATS |
| Graph algorithms | Neo4j Graph Data Science library (for AI-driven recommendations) |
| Alternative (lower cost) | Apache AGE extension for PostgreSQL (graph queries within PostgreSQL, lower performance ceiling) |
| Caching | Redis for hot resource availability data |
| Multi-tenancy | PostgreSQL RLS + Neo4j tenant_id property on all nodes |

---

## Migration and Scaling Considerations

- **Start with PostgreSQL only.** For MVP and early customers, implement resource matching with SQL queries against the normalised resource tables. The graph layer is an optimisation that becomes valuable as the resource pool grows beyond 100-200 people.
- **Apache AGE as stepping stone.** If the team wants graph queries without a separate database, Apache AGE adds Cypher query support directly to PostgreSQL. This provides a migration path: start with AGE, and move to standalone Neo4j when query volume or complexity justifies it.
- **Graph rebuild from source.** The Neo4j graph should always be rebuildable from PostgreSQL data. This means no write operations should go directly to Neo4j -- all mutations flow through PostgreSQL first.
- **Neo4j scaling.** Neo4j supports read replicas for query scaling. For multi-tenant isolation, consider separate Neo4j databases per large tenant or use the tenant_id property with query-time filtering.
- **Skill taxonomy evolution.** The graph model allows skill categories, synonyms (e.g., "React.js" = "React" = "ReactJS"), and hierarchies (e.g., "AWS" parent of "Lambda", "S3", "DynamoDB") to be modelled naturally. This taxonomy can be curated by the AI layer and refined over time.
- **AI-powered allocation.** The graph structure enables training ML models for resource recommendation: given a project's requirements, client history, and team composition, predict which resource assignments maximise utilisation and client satisfaction. The graph provides the feature vectors; the model runs externally and writes recommendations back.
