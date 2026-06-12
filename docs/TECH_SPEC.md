# TECH_SPEC.md – sla‑compass  

**Document version:** 1.0  
**Last updated:** 2026‑06‑12  
**Owner:** Product Engineering Lead – sla‑compass  
**Repository:** `arkashira/sla-compass`  

---  

## 1. Overview  

`sla-compass` is an **automated SLA compliance reporting tool** targeted at enterprise SaaS providers. It ingests service‑level metrics from customer‑facing systems, evaluates them against contractual SLA definitions, and generates periodic compliance reports (PDF/HTML/JSON) that can be delivered to clients via email, API, or a self‑service portal.

Key goals:  

| Goal | Success Metric |
|------|----------------|
| **Zero‑manual reporting** | > 95 % of scheduled reports generated without human intervention |
| **Accurate compliance calculation** | < 0.1 % deviation vs. manual audit |
| **Scalable ingestion** | Process ≥ 10 M metric points per day per tenant |
| **Secure multi‑tenant isolation** | No data leakage across tenants, SOC‑2 compliant |
| **Extensible SLA language** | Support custom KPI formulas via DSL or Python snippets |

---  

## 2. Architecture Overview  

```
+-------------------+       +-------------------+       +-------------------+
|   Data Sources    | <---> |   Ingestion Layer | <---> |   Metric Store    |
| (Prometheus,      |       | (Kafka + Flink)   |       | (ClickHouse)      |
|  Datadog, API)    |       +-------------------+       +-------------------+
+-------------------+                |                         |
                                     v                         v
                              +-------------------+   +-------------------+
                              |   SLA Engine      |   |   Report Engine   |
                              | (Flink + Python)  |   | (Node.js + React)|
                              +-------------------+   +-------------------+
                                     |                         |
                                     v                         v
                              +-------------------+   +-------------------+
                              |   Tenant DB (PG)  |   |   UI / API (FastAPI)|
                              +-------------------+   +-------------------+
```

* **Ingestion Layer** – Real‑time streaming of metric points via Apache Kafka. Flink jobs normalize, enrich, and write to ClickHouse (columnar store) for fast analytical queries.  
* **Metric Store** – ClickHouse cluster (3‑node replication) stores raw metric points and pre‑aggregated windows.  
* **SLA Engine** – Evaluates SLA definitions (stored in PostgreSQL) against metric windows using Flink SQL + embedded Python sandbox for custom formulas. Results are persisted as compliance events.  
* **Report Engine** – Generates scheduled reports using Node.js workers, templating with Handlebars (HTML) → PDF via `puppeteer`. Also produces JSON payloads for API consumption.  
* **Tenant DB** – PostgreSQL (12) holds tenant metadata, SLA definitions, report schedules, user accounts, and RBAC.  
* **API / UI** – FastAPI backend (Python 3.11) exposing REST/GraphQL endpoints; React (Vite) SPA for admin & client portals.  
* **Auth & Security** – OAuth2/OIDC (Keycloak) + per‑tenant JWT scopes; data‑at‑rest encryption (AES‑256) and TLS‑1.3 in‑flight.  

---  

## 3. Core Components  

| Component | Language / Runtime | Primary Responsibility | Key Libraries |
|-----------|-------------------|------------------------|----------------|
| **Ingestion Service** | Java (Kafka Streams) | Pull metrics from external APIs, push to Kafka topics | `kafka-clients`, `jackson` |
| **Flink Jobs** | Java / Python (PyFlink) | Normalization, windowing, SLA evaluation | `flink‑core`, `pyflink`, `pandas` |
| **Metric Store** | ClickHouse | High‑throughput storage & analytical queries | N/A (native) |
| **SLA Engine** | Python (sandbox) | Parse SLA DSL, compute compliance, emit events | `pydantic`, `numexpr`, `restrictedpython` |
| **Report Generator** | Node.js (v20) | Assemble templates, render PDFs, schedule jobs | `handlebars`, `puppeteer`, `bullmq` |
| **API Layer** | Python (FastAPI) | CRUD for tenants, SLA defs, report retrieval | `fastapi`, `sqlalchemy`, `uvicorn` |
| **Frontend** | React (TypeScript) | Tenant portal, dashboard, report viewer | `react`, `mui`, `axios` |
| **Auth Service** | Keycloak (Docker) | Centralised identity & access management | – |
| **Orchestration** | Docker‑Compose / Kubernetes | Deploy all services, health‑checks, scaling | `helm`, `istio` (service mesh) |

---  

## 4. Data Model  

### 4.1 PostgreSQL (Tenant DB)

```sql
-- tenants
CREATE TABLE tenant (
    id          UUID PRIMARY KEY,
    name        TEXT NOT NULL,
    domain      TEXT UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- users
CREATE TABLE user_account (
    id          UUID PRIMARY KEY,
    tenant_id   UUID REFERENCES tenant(id) ON DELETE CASCADE,
    email       TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role        TEXT CHECK (role IN ('admin','analyst','viewer')),
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- sla_definitions
CREATE TABLE sla_definition (
    id          UUID PRIMARY KEY,
    tenant_id   UUID REFERENCES tenant(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    description TEXT,
    dsl         TEXT NOT NULL,          -- custom DSL or Python snippet
    effective_from DATE NOT NULL,
    effective_to   DATE,
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- report_schedules
CREATE TABLE report_schedule (
    id          UUID PRIMARY KEY,
    tenant_id   UUID REFERENCES tenant(id) ON DELETE CASCADE,
    sla_id      UUID REFERENCES sla_definition(id),
    frequency   INTERVAL NOT NULL,      -- e.g. '1 month'
    next_run    TIMESTAMPTZ NOT NULL,
    format      TEXT CHECK (format IN ('pdf','html','json')),
    destination TEXT,                  -- email or webhook URL
    enabled     BOOLEAN DEFAULT true,
    created_at  TIMESTAMPTZ DEFAULT now()
);
```

### 4.2 ClickHouse (Metric Store)

| Table | Columns | Description |
|-------|---------|-------------|
| `metrics_raw` | `tenant_id UUID`, `source STRING`, `metric_name STRING`, `timestamp DateTime64(3)`, `value Float64` | Ingested points |
| `metrics_agg` | `tenant_id UUID`, `metric_name STRING`, `window_start DateTime64`, `window_end DateTime64`, `agg_value Float64` | Pre‑aggregated windows (5‑min, 1‑h) |
| `sla_events` | `tenant_id UUID`, `sla_id UUID`, `window_start DateTime64`, `window_end DateTime64`, `compliant BOOLEAN`, `details JSON` | Results of SLA evaluation |

---  

## 5. Key APIs / Interfaces  

### 5.1 Public REST API (FastAPI)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST /api/v1/metrics` | Ingest a batch of metric points (JSON) | Service token | Used by external collectors when push model is preferred |
| `GET /api/v1/tenants/{tenant_id}/sla` | List SLA definitions | JWT (admin) | |
| `POST /api/v1/tenants/{tenant_id}/sla` | Create SLA definition | JWT (admin) | Body: `{name, description, dsl, effective_from, effective_to}` |
| `GET /api/v1/tenants/{tenant_id}/reports/{report_id}` | Download generated report | JWT (analyst/viewer) | Supports `Accept: application/pdf` etc. |
| `POST /api/v1/webhooks/sla-event` | Receive SLA breach webhook (internal) | Service token | Payload: `{tenant_id, sla_id, window, compliant, details}` |

### 5.2 Internal Event Bus (Kafka)

| Topic | Producer | Consumer |
|-------|----------|----------|
| `metrics.ingest` | Ingestion Service | Flink Normalizer |
| `sla.evaluation` | Flink SLA Engine | Report Generator (for breach alerts) |
| `reports.schedule` | Scheduler (cron‑job) | Report Generator workers |
| `audit.log` | All services | Central logging (ELK) |

### 5.3 Report Generation Interface  

Node worker subscribes to `reports.schedule` → fetches SLA events from ClickHouse → renders Handlebars template → `puppeteer` → stores PDF in S3 bucket (`reports/{tenant_id}/{report_id}.pdf`).  

---  

## 6. Technology Stack  

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Streaming** | Apache Kafka 3.3 + Flink 1.18 | Exactly‑once semantics, low‑latency windowed calculations |
| **Time‑Series Store** | ClickHouse 24.3 | Columnar, high‑throughput reads for SLA windows |
| **Relational** | PostgreSQL 15 | Strong consistency for tenant metadata |
| **Backend** | Python 3.11 (FastAPI) | Rapid development, async I/O, OpenAPI auto‑gen |
| **Frontend** | React 18 + TypeScript + MUI | Modern SPA, reusable component library |
| **Auth** | Keycloak 24 (OIDC) | Enterprise‑grade identity federation |
| **Containerisation** | Docker + Kubernetes (v1.28) | Horizontal scaling, rolling updates |
| **Observability** | Prometheus + Grafana + Loki | Metrics, dashboards, log aggregation |
| **CI/CD** | GitHub Actions + ArgoCD | Automated testing, canary deployments |
| **Security** | TLS‑1.3, AES‑256‑GCM, OPA policies | Compliance (SOC‑2, GDPR) |

---  

## 7. Dependencies  

| Dependency | Version | License |
|------------|---------|---------|
| `org.apache.kafka:kafka-clients` | 3.5.1 | Apache‑2.0 |
| `org.apache.flink:flink-core` | 1.18.0 | Apache‑2.0 |
| `clickhouse-client` | 24.3 | Apache‑2.0 |
| `fastapi` | 0.110.0 | MIT |
| `uvicorn[standard]` | 0.27.0 | BSD‑3 |
| `react` | 18.2.0 | MIT |
| `keycloak` | 24.0.3 | Apache‑2.0 |
| `puppeteer` | 22.6.3 | Apache‑2.0 |
| `handlebars` | 4.7.8 | MIT |
| `restrictedpython` | 7.0 | BSD‑3 |
| `docker` | 24.0 | Apache‑2.0 |
| `helm` | 3.14 | Apache‑2.0 |

---  

## 8. Deployment Architecture  

### 8.1 Kubernetes Cluster (Production)

| Namespace | Service | Replicas | Resources (CPU/Memory) |
|-----------|---------|----------|------------------------|
| `infra` | Kafka (statefulset) | 3 | 4 CPU / 8 Gi |
| `infra` | Zookeeper | 3 | 2 CPU / 4 Gi |
| `infra` | ClickHouse | 3 | 8 CPU / 16 Gi |
| `infra` | PostgreSQL | 2 (primary/replica) | 4 CPU / 8 Gi |
| `auth` | Keycloak | 2 | 2 CPU / 4 Gi |
| `ingest` | metric‑ingest (Java) | 4 | 2 CPU / 2 Gi |
| `processing` | flink‑job‑manager | 1 | 4 CPU / 8 Gi |
| `processing` | flink‑task‑manager | 6 | 4 CPU / 8 Gi |
| `backend` | api‑gateway (FastAPI) | 4 | 2 CPU / 2 Gi |
| `backend` | report‑worker (Node) | 4 | 2 CPU / 2 Gi |
| `frontend` | spa (nginx) | 2 | 1 CPU / 1 Gi |
| `monitor` | prometheus | 2 | 2 CPU / 4 Gi |
| `monitor` | grafana | 1 | 1 CPU / 2 Gi |
| `monitor` | loki | 2 | 2 CPU / 4 Gi |

*Ingress* – Istio gateway with mTLS enforcement.  
*Secrets* – Managed via Kubernetes Secrets + sealed‑secrets for rotation.  
*Backup* – Daily snapshots of PostgreSQL and ClickHouse to S3 (versioned).  

### 8.2 CI/CD Pipeline  

1. **Push** → GitHub Actions lint (flake8, eslint) + unit tests.  
2. **Build** Docker images (multi‑arch).  
3. **Push** to private ECR.  
4. **ArgoCD** syncs Helm releases to `staging`.  
5. **Smoke tests** (Postman collection) → if pass, promote to `production`.  

---  

## 9. Security & Compliance  

* **Data Isolation** – Row‑level security in PostgreSQL (`tenant_id` filter) and ClickHouse (`tenant_id` partition).  
* **Encryption** – TLS for all network traffic; at‑rest encryption via KMS‑managed keys for S3 and DB disks.  
* **Audit Logging** – Every API call, SLA evaluation, and report generation writes to `audit.log` topic; retained 90 days.  
* **Vulnerability Scanning** – Trivy scan on each image; fail build on CVE > 7.  
* **Pen‑Test** – Annual external penetration test; internal static code analysis (Bandit, SonarQube).  

---  

## 10. Operational Considerations  

| Concern | Strategy |
|---------|----------|
| **Scaling ingestion** | Increase Kafka partitions; Flink parallelism auto‑adjusts based on metric volume. |
| **Cold‑start SLA evaluation** | Pre‑warm Flink windows for newly onboarded tenants using back‑fill jobs. |
| **Report latency** | Generate reports asynchronously; expose “ready” webhook for downstream systems. |
| **Disaster recovery** | Multi‑zone K8s cluster; cross‑region ClickHouse replica; point‑in‑time restore for PostgreSQL. |
| **Observability** | Prometheus alerts on lag (`consumer_lag > 5 min`), SLA breach rate spikes, storage utilization. |
| **Feature toggles** | ConfigMap‑driven flags (e.g., enable Python DSL sandbox). |

---  

## 11. Open Issues & Future Work  

| Issue ID | Description | Owner | Target Milestone |
|----------|-------------|-------|------------------|
| FR‑001 | Add support for SLA definitions via YAML files (GitOps) | Product | v0.2 |
| FR‑002 | Implement multi‑language DSL (e.g., Go templates) | Engineering | v0.3 |
| PERF‑001 | Benchmark ClickHouse query latency for > 50 M points/day | Data Engineering | v0.2 |
| SEC‑001 | Integrate OPA for fine‑grained API authorization | Security | v0.2 |

---  

## 12. Glossary  

| Term | Meaning |
|------|---------|
| **SLA** | Service‑Level Agreement – contractual metrics (e.g., uptime, latency). |
| **DSP** | Data‑Stream Processor – Flink jobs evaluating SLAs. |
| **DSL** | Domain‑Specific Language used to express SLA formulas. |
| **Tenant** | A SaaS customer; all data is scoped to a tenant ID. |
| **Compliance Event** | Result of evaluating a metric window against an SLA (pass/fail). |

---  

*Prepared by the sla‑compass engineering team. For questions, contact `eng-lead@axentx.com`.*
