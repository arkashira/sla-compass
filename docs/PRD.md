# PRD – SLA‑Compass  
**Product:** Automated SLA Compliance Reporting Tool for Enterprise SaaS Companies  
**Owner:** Senior Product Lead – Axentx  
**Date:** 2026‑06‑12  
**Version:** 1.0  

---  

## 1. Problem Statement  

Enterprise SaaS providers must demonstrate to their customers that service‑level agreements (SLAs) are being met. Current processes are:

| Pain Point | Impact |
|------------|--------|
| **Manual data aggregation** – Ops teams pull logs from monitoring, billing, and ticketing systems, then stitch them together in spreadsheets. | 30 % of ops time spent on reporting; high error rate. |
| **Inconsistent metrics** – Different teams use divergent definitions for uptime, latency, and incident resolution. | SLA breaches go unnoticed until customer escalations. |
| **Late or missing reports** – Quarterly or monthly compliance reports are often delivered after the agreed deadline. | Customer trust erosion; risk of contract penalties. |
| **No audit trail** – Lack of versioned, tamper‑proof reports makes audits costly. | Legal & compliance overhead. |

**Result:** SaaS companies lose revenue through penalties, churn, and wasted internal effort.  

## 2. Target Users & Personas  

| Persona | Role | Primary Need |
|---------|------|--------------|
| **Operations Manager** | Head of Site Reliability / Cloud Ops | Automated, accurate SLA data collection & dashboard. |
| **Customer Success Lead** | CSM / Account Manager | Ready‑to‑share compliance reports for client meetings. |
| **Compliance Officer** | Legal / Auditing | Immutable audit trail, version control, and export compliance. |
| **Product Engineer** | Backend / Infra Engineer | Easy integration with existing monitoring & ticketing APIs. |

**Enterprise SaaS market** – $150 B total addressable market, focusing on companies with >10 k monthly active users and multi‑region deployments.

## 3. Goals & Success Metrics  

| Goal | Metric | Target (12 mo) |
|------|--------|----------------|
| **Reduce reporting effort** | Avg. ops hours per reporting cycle | ↓ from 30 h → ≤ 5 h |
| **Increase on‑time delivery** | % reports delivered before SLA deadline | ≥ 95 % |
| **Improve accuracy** | Report error rate (post‑release tickets) | ≤ 0.5 % |
| **Drive revenue** | New paying customers for SLA‑Compass | 30 enterprise contracts |
| **Retention** | Net Revenue Retention (NRR) of SLA‑Compass customers | ≥ 120 % |

## 4. Scope  

### In‑Scope (MVP)  

1. **Data Connectors** – Pre‑built adapters for:  
   - Monitoring (Prometheus, Datadog, New Relic)  
   - Incident Management (PagerDuty, ServiceNow)  
   - Billing & Usage (Stripe, Chargebee)  
2. **Metric Normalization Engine** – Unified schema for:  
   - Availability (Uptime %)  
   - Performance (Latency, Throughput)  
   - Incident metrics (MTTR, breach count)  
3. **Compliance Dashboard** – Real‑time view of SLA status per contract:  
   - Traffic light indicators (Green/Yellow/Red)  
   - Trend charts (30‑day rolling)  
4. **Report Generator** – Configurable PDF/HTML reports with:  
   - Executive summary, detailed tables, charts  
   - Automatic versioning & digital signature (SHA‑256)  
5. **Export & API** –  
   - REST endpoint to fetch raw SLA data (JSON)  
   - Scheduled email delivery (PDF/HTML)  
6. **User Management** – Role‑based access (Admin, Viewer, Auditor).  
7. **Audit Log** – Immutable, tamper‑evident log of report generation & data pulls (stored in append‑only object store).  

### Out‑of‑Scope (Future Phases)  

| Feature | Reason |
|---------|--------|
| **Custom connector builder UI** | MVP will ship with code‑first connectors; UI builder planned Q3. |
| **AI‑driven anomaly explanation** | Requires additional model training; deferred to Phase 2. |
| **Multi‑tenant SaaS hosting** | Initial release will be self‑hosted for enterprise customers. |
| **Integration with CRM (Salesforce, HubSpot)** | Low priority; can be added via webhook in Phase 2. |
| **Real‑time alerting** | Reporting focus; alerting will be handled by existing monitoring stacks. |

## 5. Key Features (Prioritized)  

| Priority | Feature | Description | Acceptance Criteria |
|----------|---------|-------------|----------------------|
| **P1** | **Unified Data Ingestion Layer** | Pulls raw metrics from connectors on a configurable schedule, stores in time‑series DB. | • All three built‑in connectors ingest data with ≤ 5 min latency.<br>• Failure retries with exponential back‑off. |
| **P1** | **SLA Metric Engine** | Transforms raw data into SLA‑specific KPIs per contract definition. | • Correctly computes uptime % per defined window.<br>• Handles time‑zone & maintenance windows. |
| **P1** | **Compliance Dashboard** | Single‑page UI showing current SLA health per client. | • Dashboard loads < 2 s for 10,000 contracts.<br>• Traffic‑light status matches engine output. |
| **P2** | **Report Builder** | Generates PDF/HTML reports from dashboard selections. | • PDF matches visual dashboard layout.<br>• Digital signature verified via checksum. |
| **P2** | **Scheduled Delivery** | Email scheduler to send reports on a recurring cadence. | • Supports daily, weekly, monthly schedules.<br>• Delivery logs stored in audit log. |
| **P3** | **Role‑Based Access Control (RBAC)** | Admins can create users, assign roles, and set contract visibility. | • Unauthorized user cannot view restricted contracts.<br>• Admin can audit all actions. |
| **P3** | **API Export** | REST endpoint for downstream consumption of SLA data. | • Returns JSON conforming to OpenAPI spec.<br>• Rate‑limited to 100 req/s. |
| **P4** | **Immutable Audit Log** | Append‑only log stored in object store with cryptographic hash chaining. | • Log entries cannot be altered without detection.<br>• Queryable via UI for compliance audits. |

## 6. User Journeys  

1. **Ops Manager** configures a new SaaS product contract → selects relevant connectors → defines SLA thresholds → dashboard instantly shows compliance.  
2. **Customer Success Lead** clicks “Generate Report” → selects client → PDF is produced, signed, and emailed to the client automatically.  
3. **Compliance Officer** opens audit log → verifies that the report generated at 2026‑05‑31 23:59 UTC matches the stored hash.  

## 7. Technical Requirements  

| Area | Requirement |
|------|-------------|
| **Platform** | Python 3.11 backend, FastAPI, PostgreSQL for metadata, TimescaleDB for time‑series. |
| **Connectors** | Implemented as pluggable Python packages; use async HTTP clients (httpx). |
| **UI** | React 18 + TypeScript, Ant Design, served via Vite. |
| **PDF Generation** | WeasyPrint or Puppeteer‑based rendering for pixel‑perfect output. |
| **Security** | OAuth2 + JWT for API auth, RBAC enforcement, at‑rest encryption (AES‑256). |
| **Scalability** | Horizontal scaling via Docker Compose → Kubernetes (Helm chart). |
| **Observability** | Export metrics to Prometheus; logs to Loki; tracing via OpenTelemetry. |
| **Compliance** | GDPR‑ready data deletion, SOC‑2 Type II controls. |

## 8. Milestones & Timeline  

| Milestone | Deliverable | Owner | Target Date |
|-----------|-------------|-------|-------------|
| **M1 – Foundations** | Repo scaffold, CI/CD, basic auth | Engineering Lead | 2026‑06‑30 |
| **M2 – Connectors** | Prometheus, PagerDuty, Stripe adapters | Data‑Engineers | 2026‑07‑21 |
| **M3 – Metric Engine** | SLA KPI calculations, unit tests | Backend Team | 2026‑08‑15 |
| **M4 – Dashboard MVP** | Real‑time UI, traffic‑light status | Frontend Team | 2026‑09‑05 |
| **M5 – Report Generator** | PDF/HTML output, digital signature | Full‑stack | 2026‑09‑30 |
| **M6 – Scheduler & Email** | Cron‑based delivery, audit log | Ops/Backend | 2026‑10‑15 |
| **M7 – RBAC & API** | Role management, OpenAPI spec | Security Lead | 2026‑11‑01 |
| **M8 – Beta Launch** | Pilot with 2 enterprise customers | PM/Customer Success | 2026‑11‑30 |
| **M9 – GA Release** | Full documentation, support hand‑off | Product Ops | 2026‑12‑31 |

## 9. Risks & Mitigations  

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **Connector incompatibility** (API changes) | Medium | High (data loss) | Versioned adapters, automated contract tests against sandbox APIs. |
| **Performance bottleneck on large contract sets** | Low | Medium | Use TimescaleDB hypertables, async ingestion, horizontal scaling. |
| **Regulatory compliance gaps** | Low | High | Early security review, SOC‑2 audit prep, GDPR data‑subject request tooling. |
| **Customer adoption** (perceived complexity) | Medium | Medium | Provide out‑of‑the‑box templates, onboarding wizard, professional services. |

## 10. Open Questions  

1. Should we support on‑prem deployment vs. managed SaaS in the same code base?  
2. What SLA definition language (DSL) is most flexible for enterprise contracts?  
3. Will we need multi‑currency support for billing connectors?  

---  

*Prepared by the Axentx Product Team. All stakeholders review and sign‑off required before proceeding to engineering.*
