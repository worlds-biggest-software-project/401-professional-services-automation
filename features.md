# Professional Services Automation — Feature & Functionality Survey

> Candidate #401 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Kantata (Mavenlink + Kimble) | SaaS | Commercial (enterprise) | https://www.kantata.com |
| Rocketlane | SaaS | Commercial | https://www.rocketlane.com |
| Autotask PSA (Datto) | SaaS | Commercial (MSP-focused) | https://www.datto.com/autotask-psa |
| Birdview PSA | SaaS | Commercial | https://birdviewpsa.com |
| Polaris PSA | SaaS | Commercial | https://www.polaris.app |
| HaloPSA | SaaS | Commercial | https://halopsa.com |

## Feature Analysis by Solution

### Kantata

**Core features**
- Project delivery management: milestone tracking, task assignment, dependency mapping, and delivery risk alerts using AI-based forecasting
- Resource planning: capacity planning, role-based scheduling, skill matching, and utilisation dashboards showing billable vs. non-billable time
- Time and expense tracking: mobile and desktop timesheets, receipt capture, and automated policy enforcement
- Billing and revenue recognition: milestone billing, time-and-materials invoicing, retainer management, and ASC 606 / IFRS 15 compliance
- Real-time profitability dashboards at project, client, and portfolio level

**Differentiating features**
- Combined Mavenlink and Kimble product lineages deliver deep resource management plus financial reporting in one platform
- AI-driven delivery risk prediction flags projects at risk of slipping before deadlines pass
- Strong financial reporting: margin, realisation rate, and effective bill rate analytics for finance leaders

**UX patterns**
- Gantt-style project timeline with resource allocation visualisation
- Profitability heatmap at portfolio level showing which clients and project types drive margin
- Resource bench view highlighting underutilised staff with available capacity

**Integration points**
- CRM: Salesforce and HubSpot
- ERP: NetSuite and SAP
- Accounting: QuickBooks and Xero
- HR: Workday and BambooHR

**Known gaps**
- Implementation complexity and cost puts it out of reach for smaller professional services firms
- Client portal features are functional but less modern than purpose-built client experience platforms
- Mobile experience less polished than field-first tools

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Rocketlane

**Core features**
- Client-facing project portal: real-time project status visible to clients without exposing internal tooling
- AI-first platform: automated handoffs between project stages, AI-generated status updates, and time-tracking accuracy improvement through AI suggestions
- Delivery templates: standardised project blueprints for common service delivery patterns, reducing project setup time
- Time tracking with AI-suggested entries based on calendar and task activity
- Billing: time-and-materials and milestone invoicing with automated invoice generation

**Differentiating features**
- AI-powered automated handoffs and status generation distinguish it in the post-sales delivery segment
- Client portal is central to the product — Rocketlane's primary differentiator versus traditional PSA tools
- Intake and onboarding automation: new client projects auto-created from CRM deal close with templated workspace

**UX patterns**
- Unified workspace where internal team and client collaborate on the same project view
- Progress tracker customers can bookmark and self-serve for status
- AI-drafted weekly status report for client communication

**Integration points**
- Salesforce and HubSpot CRM for deal-to-project handoff
- Slack for team notifications and client communication
- Jira for engineering task sync

**Known gaps**
- Less mature financial reporting than Kantata; revenue recognition and ERP integration are not primary features
- Resource planning capabilities are lighter than enterprise PSA tools
- Best suited for post-sales customer success and implementation delivery; less suitable for consultancy resourcing at scale

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Autotask PSA (Datto)

**Core features**
- Purpose-built for managed service providers (MSPs) and IT services firms
- Service desk and ticketing: full helpdesk workflow with SLA management, escalation, and customer notification
- Contract management: recurring managed service contracts, retainers, and block-hour billing
- Billing automation: automated invoice generation from service tickets and contract entitlements
- Integration with RMM (remote monitoring and management) tools for automated ticket creation from monitoring alerts

**Differentiating features**
- Deep MSP-specific workflows: hardware procurement, vendor management, and recurring contract billing unavailable in general-purpose PSA tools
- Native integration with Datto RMM, Datto Backup, and the broader Datto/Kaseya stack
- Co-managed IT and sub-contractor billing models built in

**UX patterns**
- Ticket queue with SLA countdown timers and technician dispatch view
- Client asset register linked to service history
- MSP profitability dashboard by client, contract, and technician

**Integration points**
- Datto and Kaseya RMM tools (native)
- Microsoft 365 and Azure for asset tracking
- QuickBooks and Xero accounting integrations

**Known gaps**
- Primarily an MSP tool; not suitable for consulting firms, agencies, or non-IT service businesses
- UI design feels dated compared with newer PSA entrants
- Less suitable for project-based delivery versus recurring managed services

**Licence / IP notes**
- Proprietary SaaS (Datto acquired by Kaseya). No open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Project milestone and task tracking with deadline and dependency management
- Time and expense capture with mobile support and approval workflows
- Resource utilisation dashboard showing billable vs. non-billable allocation
- Invoice generation from tracked time and approved expenses
- CRM integration for deal-to-project handoff (Salesforce and HubSpot at minimum)

### Differentiating Features
- AI-driven delivery risk prediction before milestones slip
- Client-facing project portal providing real-time status without internal tool access
- ASC 606 / IFRS 15 revenue recognition automation built into billing engine
- AI-assisted time tracking reducing under-capture of billable hours
- Skill-based resource matching for role assignments

### Underserved Areas / Opportunities
- Mid-market PSA combining modern UX with financial-grade revenue recognition at an accessible price point
- Seamless CRM-to-SOW-to-project pipeline eliminating manual handoffs between sales and delivery
- Real-time project profitability visible to project managers without finance team involvement
- Vertical-specific PSA for creative agencies, legal services, or engineering consultancies with industry-appropriate billing models

### AI-Augmentation Candidates
- Automated project health scoring surfacing delivery risk without PM manual input
- AI-drafted client status reports summarising progress, risks, and next steps
- Resource allocation optimisation recommendations based on skills, availability, and project priority
- Billing anomaly detection flagging unbilled work or uncaptured expenses before invoice run

## Legal & IP Summary

All PSA platforms in this space are proprietary SaaS tools with no open-source equivalents at full-feature parity. Revenue recognition logic aligned with ASC 606 and IFRS 15 is based on published accounting standards, not proprietary algorithms, and can be implemented independently. Time-tracking and billing workflows do not carry patent risk. The primary competitive moats are the depth of ERP and CRM integrations and the quality of the AI delivery-risk models — both of which require sustained investment but not licensed IP from incumbents. A new entrant can build a full PSA platform without any licensing obligations.

## Recommended Feature Scope

**Must-have (MVP)**:
- Project and milestone management with task assignment and dependency mapping
- Time tracking: web, mobile, and timer-based entry with manager approval workflow
- Expense capture with receipt photo, category tagging, and billable flag
- Invoice generation from approved time and expense with time-and-materials and milestone billing modes
- Resource utilisation dashboard: billable vs. non-billable time by person, team, and project
- Salesforce and HubSpot CRM integration for deal-to-project handoff

**Should-have (v1.1)**:
- Client-facing project portal showing milestone status, documents, and approvals
- AI-assisted delivery risk flagging based on schedule and budget burn patterns
- ASC 606 / IFRS 15 revenue recognition automation for milestone and percentage-of-completion contracts
- Skill-based resource matching and capacity planning view
- QuickBooks and Xero accounting integration for invoice sync

**Nice-to-have (backlog)**:
- AI-generated client status reports and progress summaries
- SOW-to-project automatic setup triggered by CRM deal close
- Project profitability dashboard at portfolio level with margin and effective bill rate
- Subcontractor billing and vendor management module
- Mobile-first offline-capable time and expense capture for field-based consultants
