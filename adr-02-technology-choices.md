---
status: review
date: 09/10/2025
decision-makers: Marim Sabah (Architect)
consulted: Module tutor
informed: Module tutor
---

## Context and Problem Statement
I have to select a technology stack for CMS that satisfies functional scope (F01–F12) and stringent NFRs (P01/P02, AV01, S01–S03, SC01, OBS01, AUD01) while enabling a fast Task‑2 proof of concept and straightforward progression to production.

Scope includes: frontend, API framework, database, cache, message broker, object storage, anti‑virus scanning, identity/SSO, observability, CI/CD/testing tooling.


## Decision Drivers

- **Performance/latency:** p95 create < 1.5s; read < 300ms (P01/P02).
- **Security/privacy:** OIDC + PKCE SSO; GDPR DSAR & retention (S02/S03, AUD01).
- **Availability & operability:** 99.9% target; clear telemetry (AV01, OBS01, SLO01).
- **Developer productivity:** strong typing, mature libraries, excellent docs.
- **Deployment flexibility:** cloud/on‑prem; open standards; cost‑effective.
- **Accessibility:** first‑class a11y/testing tooling (A01–A03, FRONT01).

## Considered Options

1. **React + TypeScript (Web), Spring Boot 3 / Java 21 (API), PostgreSQL, Redis, RabbitMQ, S3/MinIO, ClamAV, OIDC + PKCE, OpenTelemetry + Prometheus/Grafana, GitHub Actions + Docker** **← chosen**
2. **Angular + .NET 8 (ASP.NET Core), SQL Server, Azure Service Bus, Azure Blob, Microsoft Defender AV, Entra ID (OIDC), Azure Monitor**
3. **Next.js (SSR) + NestJS (Node), MongoDB, NATS, GCS, VirusTotal API, Auth0, Datadog**
4. **Django + DRF (Python), Celery, PostgreSQL, SQS, S3, ClamAV, Cognito, CloudWatch**
   
## Decision Outcome

Chosen option:  React + TS; Spring Boot 3 (Java 21); PostgreSQL; Redis; RabbitMQ; S3/MinIO; ClamAV; OIDC + PKCE; OTel/Prom/Grafana; GitHub Actions + Docker, because it provides:
- Mature ecosystems with excellent security, validation, and a11y/test support.
- Strong alignment with **async** patterns (outbox → RabbitMQ) to meet **P01**.
- SQL + JSONB, indexing, and optional RLS in **PostgreSQL** support **S01** and reporting.
- **MinIO/S3 presigned** flows keep file uploads off OLTP path; ClamAV integrates cleanly.
- **OTel/Prom/Grafana** provide portable observability and SLO dashboards (OBS01/SLO01).
- Broad on‑prem and multi‑cloud portability; open‑source friendly.
  
### Consequences

* Good, because
- Proven stack for enterprise web APIs; abundant examples and libraries.
- Clear mapping to **C4 L2** containers and earlier ADR‑001 style.
- Strong toolchain for CI security gates (SEC04) and performance testing.
  
* Bad, because
- Two languages (TS + Java) increase skill breadth needed.
- RabbitMQ/Redis add moving parts (must be monitored; runbooks required).
- Spring Boot memory footprint can be higher than some alternatives.

### Confirmation
- POC spikes: presigned uploads, outbox→RabbitMQ, OIDC login, OTel traces.
- Passing CI gates for SAST/DAST/dependency scans; basic load tests meet P01/P02.
- C4 L2 shows all chosen components and their edges.


## Pros and Cons of the Options

### 1) React + TS / Spring / PostgreSQL / Redis / RabbitMQ / S3 / ClamAV (Chosen)
- Good: balanced performance, security, tooling; portable; async‑friendly.  
- Neutral: polyglot team; infra to operate.  
- Bad: JVM footprint; broker adds ops overhead.

### 2) Angular + .NET + Azure native
- Good: tight Azure integration; powerful tooling; enterprise support.  
- Neutral: cloud bias; license/hosting cost considerations.  
- Bad: portability reduced; lock‑in risk for assessment neutral stance.


### 3)  Next.js/NestJS + MongoDB/NATS
- Good: unified language; fast iteration.  
- Neutral: Mongo schema flexibility.  
- Bad: complex consistency for transactions; heavy SSR infra for simple portal.

### 4) Django/Celery + Postgres/SQS 
- Good: rapid CRUD; Python ecosystem.  
- Neutral: Celery + SQS fit async.  
- Bad: a11y/testing and typing less rigorous out of the box; team skill fit uncertain.

## More Information
- Security: OWASP ASVS; OAuth 2.1/OIDC; TLS/HSTS; CSRF protections.  
- Observability: OTel specs; RED/USE metrics; SLO/error budgets.

---

## Mapping to this Project
- **FRs:** F01–F07, F08–F11 directly; F12 optional via search integration.  
- **NFRs:** P01, P02, AV01, S01–S03, SC01, OBS01, AUD01, SEC04, SLO01, FRONT01.  
- **C4:** L2 containers use these technologies; L3 components use Spring modules.
