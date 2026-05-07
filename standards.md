# Standards & API Reference

> Project: Professional Services Automation · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 21500:2012 — Guidance on Project Management**
- URL: https://www.iso.org/standard/50003.html
- Defines 39 project management processes grouped into five process groups (initiating, planning, implementing, controlling, closing) and ten subject groups including scope, resources, time, cost, risk, quality, procurement, and communication. A PSA platform's project and task model should be aligned with these subject groups to ensure interoperability with enterprise project governance frameworks.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The globally recognised information security management standard. PSA platforms handle sensitive commercial data including client contracts, financial projections, staff salaries, and project financials. ISO 27001 certification (or SOC 2 Type II as its US-market equivalent) is a de-facto buyer requirement for enterprise PSA deals.

**ISO/IEC 27701:2019 — Privacy Information Management**
- URL: https://www.iso.org/standard/71670.html
- Extension to ISO 27001 addressing privacy information management. Directly relevant to GDPR compliance obligations when a PSA system stores personally identifiable information about employees, contractors, and client contacts across jurisdictions.

**ISO 8601 — Date and Time Format**
- URL: https://www.iso.org/iso-8601-date-and-time-format.html
- Defines the canonical `YYYY-MM-DD` and `YYYY-MM-DDTHH:MM:SSZ` formats for dates and timestamps. All time-tracking, scheduling, and billing date fields in a PSA API should emit and accept ISO 8601 formatted values.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization protocol used by every major PSA, CRM, accounting, and HR platform for API access delegation. A PSA system must implement OAuth 2.0 (Authorization Code flow for interactive users, Client Credentials for server-to-server integrations) to enable third-party app access and ecosystem integrations.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- The token format used ubiquitously alongside OAuth 2.0 for API access tokens. JWTs carry user identity and permission claims in a compact, self-verifiable form. All major PSA integrations (Salesforce, HubSpot, Workday) use JWT-based bearer tokens.

**RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Defines how Bearer Tokens are transmitted in HTTP requests (Authorization header). The standard pattern for authenticated PSA API calls.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- The base HTTP semantics specification covering methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation. All PSA REST APIs follow these conventions.

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Defines the `.ics` format for calendar events and scheduling data. Relevant for PSA resource scheduling and project milestone data that needs to integrate with Google Calendar, Microsoft Outlook, and Apple Calendar for staff scheduling and client milestone communications.

**RFC 4180 — Common Format and MIME Type for CSV Files**
- URL: https://datatracker.ietf.org/doc/html/rfc4180
- The de-facto standard for CSV export. PSA platforms universally offer CSV export for time entries, invoices, and resource reports. Adhering to RFC 4180 avoids encoding and delimiter edge cases.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing RESTful APIs in a machine-readable YAML or JSON document. All PSA platforms with public APIs (Kantata, Rocketlane, BigTime, Autotask) should publish an OpenAPI 3.1 spec to enable client code generation, API testing, and integration marketplace discoverability. OAS 3.1 is fully aligned with JSON Schema Draft 2020-12.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification.html
- The vocabulary for annotating and validating JSON data structures. Used to define and validate the request and response payloads for PSA API entities: projects, time entries, expenses, invoices, and resources.

**ASC 606 — Revenue from Contracts with Customers (FASB)**
- URL: https://asc.fasb.org/606
- The five-step US GAAP revenue recognition model (identify contract → identify performance obligations → determine transaction price → allocate price → recognise revenue). PSA billing engines must implement ASC 606 logic for milestone billing, percentage-of-completion, and retainer contracts. This is an accounting standard rather than a technical API spec but is the dominant compliance requirement for PSA financial modules in North American markets.

**IFRS 15 — Revenue from Contracts with Customers (IASB)**
- URL: https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/
- The IASB international equivalent of ASC 606, applying identical five-step logic. Required for PSA platforms serving European, Asia-Pacific, or multinational professional services firms filing under IFRS. PSA revenue recognition engines should support both standards as configuration options.

---

### Security & Authentication Standards

**OAuth 2.0 + PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Proof Key for Code Exchange — required extension to OAuth 2.0 for public clients (mobile apps, single-page apps). PSA mobile time-tracking apps must implement PKCE to prevent authorisation code interception attacks.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 providing user authentication and the `id_token` JWT. Enables SSO integration with enterprise identity providers (Okta, Azure AD, Google Workspace) — a table-stakes requirement for enterprise PSA buyers.

**GDPR — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- EU Regulation 2016/679 governing processing of personal data. Any PSA system storing employee timesheets, salaries, or client contact data for EU residents must comply: data residency controls, right-to-erasure workflows, processing records, and Data Processing Agreements with third-party integrations.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- AICPA attestation framework for security, availability, processing integrity, confidentiality, and privacy of SaaS systems. The de-facto security certification for US enterprise PSA buyers. Covers audit logging, access controls, encryption at rest and in transit, and incident response.

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/
- Payment Card Industry Data Security Standard. Relevant if the PSA platform processes credit card payments directly for client invoices. Most PSA tools avoid this by delegating payment capture to Stripe or a payment gateway, reducing PCI scope to SAQ A.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to external tools and data sources. A PSA platform with an MCP server would expose project status, time entries, resource availability, and client billing data to AI assistants (Claude, GitHub Copilot, etc.), enabling natural-language queries like "what projects are at risk this week?" or "draft a status report for Client X." This is an emerging AI-native integration layer not yet adopted by incumbent PSA vendors, representing a differentiation opportunity.

---

## Similar Products — Developer Documentation & APIs

### Kantata OX (formerly Mavenlink)

- **Description:** Enterprise PSA for mid-to-large professional services firms, combining deep resource management with financial reporting and project delivery. Formed by the merger of Mavenlink and Kimble.
- **API Documentation:** https://developer.kantata.com/
- **Knowledge Base API Docs:** https://knowledge.kantata.com/hc/en-us/sections/204410968-API-Docs
- **SDKs/Libraries:** Python pipeline via dltHub (https://dlthub.com/context/source/kantata); community SDK via Celigo (https://docs.celigo.com/hc/en-us/articles/5860685879451)
- **Developer Guide:** https://knowledge.kantata.com/hc/en-us/articles/202811760-Kantata-API-Overview
- **Standards:** REST/JSON, OAuth 2.0 Bearer tokens, paginated index responses (max 200 per page)
- **Authentication:** OAuth 2.0 — applications must be registered in Kantata to obtain tokens; requires Account Administrator role
- **Base URL:** `https://api.mavenlink.com/api/v1/`

---

### Rocketlane

- **Description:** AI-first PSA targeting post-sales delivery teams, with a client-facing project portal as its primary differentiator. Automates project handoffs from CRM deal close.
- **API Documentation:** https://developer.rocketlane.com/v1.3/docs/quick-start
- **Webhooks Guide:** https://developer.rocketlane.com/v1.3/docs/setting-up-webhooks-in-rocketlane
- **Webhook Events Reference:** https://developer.rocketlane.com/v1.3/docs/event-payload
- **SDKs/Libraries:** No official SDK published; REST/JSON API consumed directly
- **Standards:** REST/JSON, HTTPS-only webhooks, HTTP response-code-based delivery confirmation
- **Authentication:** OAuth 2.0 access tokens; webhook endpoints authenticated via Basic Authentication
- **Base URL:** `https://api.rocketlane.com/api/1.0`

---

### Autotask PSA (Datto/Kaseya)

- **Description:** Purpose-built PSA for managed service providers (MSPs) and IT services firms. Covers service desk, ticketing, SLA management, contract billing, and RMM integration.
- **API Documentation:** https://psa.datto.com/help/DeveloperHelp/Content/APIs/REST/REST_API_Home.htm
- **Developer Help (all APIs):** https://psa.datto.com/help/Content/1_Help_Support/APIs.htm
- **SDKs/Libraries:** Postman collection provided; community .NET library at https://github.com/panoramicdata/AutoTask.Psa.Api
- **Developer Guide:** https://psa.datto.com/help/DeveloperHelp/Content/APIs/REST/General_Topics/Intro_REST_API.htm
- **Standards:** REST/JSON with Swagger-documented endpoints; also exposes legacy SOAP API and ExecuteCommand API
- **Authentication:** API User (API-only security level) with credentials — no OAuth 2.0; bearer token issued on login
- **Note:** Supports GET, PUT, PATCH, POST, DELETE. Includes a Postman collection for rapid onboarding.

---

### BigTime

- **Description:** PSA and time-billing platform targeting professional services firms in engineering, consulting, architecture, and IT. Strong time tracking and billing with QuickBooks integration.
- **API Documentation:** https://iq.bigtime.net/BigtimeData/api/v2/Help/Async
- **Getting Started:** https://help.bigtime.net/hc/en-us/articles/10565842022423-Getting-Started-With-BigTime-s-API
- **SDKs/Libraries:** No official SDK; REST API consumed directly; XML and JSON both supported
- **Standards:** REST, supports both XML and JSON responses; async controller for long-running operations
- **Authentication:** Session-based authentication — create a session token per user/company, then include in subsequent requests
- **Rate Limits:** 30 API calls per minute per session token
- **Note:** Asynchronous processing (returning a TICKET GUID) is required for operations like posting time to QuickBooks

---

### Certinia PSA (formerly FinancialForce)

- **Description:** Cloud PSA built natively on the Salesforce platform, targeting technology and consulting firms. Covers project delivery, resource management, and revenue recognition aligned with ASC 606 / IFRS 15.
- **API Documentation:** https://help.certinia.com/psa-api-rest/16X/Resources.htm
- **Apex API Reference:** https://help.certinia.com/PSAApexAPI/2023.1/APICommonsService.htm
- **Developer Portal:** https://developer.financialforce.com/
- **SDKs/Libraries:** Salesforce Apex API; REST API; inherits Salesforce platform SDK ecosystem (Java, JavaScript, Python via Salesforce SDK)
- **Standards:** REST/JSON and SOAP/XML (Salesforce platform conventions); OpenAPI-compatible where REST endpoints are exposed
- **Authentication:** Salesforce OAuth 2.0 (inherits Salesforce connected app framework)

---

### Harvest

- **Description:** Time tracking and invoicing tool widely used by agencies and consultancies. Lighter-weight than full PSA but widely integrated and well-documented API serves as a reference for time-entry data models.
- **API Documentation:** https://help.getharvest.com/api-v2/
- **Time Entries API:** https://help.getharvest.com/api-v2/timesheets-api/timesheets/time-entries/
- **SDKs/Libraries:**
  - Python: https://www.getharvest.com/integrations/python-library (official); https://github.com/lionheart/python-harvest (community)
  - Code samples in Java, C#, Go, JavaScript, PHP, PowerShell, Ruby, Visual Basic, Google Apps Script
- **Standards:** REST/JSON; OAuth 2.0 and HTTP Basic Authentication supported
- **Authentication:** OAuth 2.0 (preferred) or HTTP Basic Authentication
- **Note:** Harvest's time entry data model (project, task, hours, billable flag, notes, date) is a useful reference for designing a PSA time-tracking schema.

---

### Salesforce CRM (Deal-to-Project Integration)

- **Description:** Leading enterprise CRM. A PSA platform must integrate Salesforce to automate the deal-to-project handoff — creating a project in the PSA when an Opportunity is closed-won in Salesforce.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- **OAuth 2.0 Guide:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_oauth_and_connected_apps.htm
- **SDKs/Libraries:** Official SDKs for Java, JavaScript/Node, Python, .NET, PHP, Ruby (https://developer.salesforce.com/tools/salesforcedx)
- **Standards:** REST/JSON; SOAP/XML; Bulk API 2.0; Streaming API (Platform Events); OpenAPI-documented
- **Authentication:** OAuth 2.0 via Connected Apps; supports Authorization Code, Client Credentials, and JWT Bearer flows
- **Base URL:** `https://<mydomain>.my.salesforce.com/services/data/vXX.X/`

---

### HubSpot CRM (Deal-to-Project Integration)

- **Description:** Mid-market CRM alternative to Salesforce. PSA platforms need HubSpot integration to automate project creation from closed deals for customers on the HubSpot stack.
- **API Documentation:** https://developers.hubspot.com/docs/guides/crm/understanding-the-crm
- **Deals Object Guide:** https://developers.hubspot.com/blog/a-developers-guide-to-hubspot-crm-objects-deals-object
- **SDKs/Libraries:** Official SDKs for JavaScript/Node, Python, PHP, Ruby, Java (https://developers.hubspot.com/docs/api/overview)
- **Standards:** REST/JSON; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0 (public apps) or Private App Access Token (private integrations)
- **Base URL:** `https://api.hubapi.com/crm/v3/objects/{objectTypeId}`

---

### QuickBooks Online (Accounting Integration)

- **Description:** Dominant SMB and mid-market accounting platform. PSA invoice data must sync to QuickBooks to avoid double-entry and enable financial reconciliation.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **API Reference:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting/most-commonly-used/account
- **SDKs/Libraries:** Official SDKs for Java, PHP, Python, .NET, Node.js; Intuit Developer Portal at https://developer.intuit.com
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON; OAuth 2.0; Webhooks for real-time change notification
- **Authentication:** OAuth 2.0 (Intuit's Intuit OAuth 2.0 flow via `https://<mydomain>.intuit.com/oauth2/token`)
- **Key Entities:** Invoice, Customer, Payment, Bill, Vendor, Account, ProfitAndLoss

---

### Xero (Accounting Integration)

- **Description:** QuickBooks Online's primary competitor, dominant in UK, Australia, and New Zealand markets. Required integration for PSA platforms targeting professional services firms outside North America.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **Invoices API:** https://developer.xero.com/documentation/api/accounting/invoices
- **SDKs/Libraries:** Official Node.js SDK: https://xeroapi.github.io/xero-node/accounting/index.html; Python, Java, .NET SDKs available at https://developer.xero.com
- **Standards:** REST/JSON; OAuth 2.0; Webhooks
- **Authentication:** OAuth 2.0 with scopes (e.g., `accounting.transactions` for invoices)
- **Developer Platform:** https://developer.xero.com/

---

### Workday (HR / Resource Integration)

- **Description:** Enterprise HCM (Human Capital Management) system. PSA platforms serving large professional services firms need Workday integration to sync employee records, roles, and cost rates for resource planning and billing rate management.
- **API Documentation (SOAP):** https://community-content.workday.com/en-us/public/products/platform-and-product-extensions/soap-api-reference.html
- **WSDL Directory:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html
- **SDKs/Libraries:** Python community wrapper at https://workday.readthedocs.io/en/latest/api.html; no official SDK
- **Standards:** SOAP/XML (primary); REST/JSON for custom objects; OAuth 2.0 for REST
- **Authentication:** Integration System User (ISU) credentials for SOAP; OAuth 2.0 Client Credentials for REST API
- **Note:** Workday's primary API is SOAP-based. REST API access is limited to custom objects. Most PSA-to-Workday integrations are built via middleware (Workato, Boomi, MuleSoft).

---

## Notes

**Revenue Recognition Standard Convergence:** ASC 606 (US GAAP) and IFRS 15 (international) are substantively identical in their five-step recognition model. A PSA revenue recognition engine can share the same core logic and expose jurisdiction as a configuration option, reducing duplication.

**Emerging Standard — MCP for AI Integration:** The Model Context Protocol is nascent (2024–2026) but gaining adoption rapidly as the standard integration layer between AI assistants and enterprise tools. No incumbent PSA vendor has published an MCP server as of mid-2026, making this a first-mover opportunity to enable AI-native workflows such as natural-language project reporting, intelligent resource suggestions, and automated risk summaries directly within AI assistant interfaces.

**Webhook Standardisation Gap:** There is no universal webhook standard across the PSA ecosystem. Each platform uses proprietary event schemas and delivery mechanisms. A PSA built to a consistent, OpenAPI-documented webhook specification with standard event naming conventions would ease the burden on system integrators.

**Data Residency and Sovereignty:** EU-based professional services firms increasingly require data residency guarantees (data stored within the EU). This intersects with GDPR Article 44–49 (transfers of personal data to third countries). Multi-region deployment support and regional data isolation are becoming table-stakes requirements for enterprise PSA bids in regulated industries.
