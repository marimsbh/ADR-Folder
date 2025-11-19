---
status: review
date: 02/10/2025
decision-makers: Marim Sabah (Architect)
consulted: Module tutor
informed: Module tutor
---

## Context and Problem Statement

ABC Limited is building a multi-tenant Complaint Management System (CMS) for large Banking and Telecom clients (e.g. NatWest, Barclays, Vodafone, O2). The system must support:

- Five roles: **Consumer, Help Desk Agent, Support Person, Help Desk Manager, System Administrator**  
- Functional scope F01–F12 / user stories US1–US17:
  - F01/F02/F03/F05/F06/F11: ticket creation (web/phone), status/timeline, resolve/close with CSAT, comments & attachments
  - F04/F07/F09: assign/reassign, notifications, reporting & CSV exports
  - F08/F10: multi-tenant admin, audit & GDPR/DSAR
- Non-functional requirements (subset):
  - **P01/P02** performance — Create p95 < 1.5 s; Read p95 < 300 ms
  - **AV01** availability — 99.9%; read-only status even if IdP is down
  - **A01–A03** accessibility — WCAG 2.2 AA, keyboard/SR, mobile/touch
  - **S01/S02/S03** security & privacy — tenant isolation, OIDC SSO, GDPR
  - **SC01** scalability — 20M+ consumers per tenant, 10% YoY growth
  - **OBS01/AUD01/SLO01** observability & auditability
  - **DR01/BKP01/KMS01/DATA01** data residency, backup & encryption

The **architecture style** is already chosen in ADR-01: a single architectural quantum (three-tier layered web app + background jobs with a transactional outbox).  

**Problem**: Choose a technology stack (languages, frameworks, infrastructure components) that:

- Implements the ADR-01 architecture effectively
- Supports the ERD you designed (PostgreSQL-style schema with tenants, tickets, timeline, attachments, outbox, audit, templates)
- Lets me realistically build the PoC (F01 + F03 + attachments + notifications) in the module timeframe
- Is credible for enterprise customers (banks/telecoms) and supports future evolution (possible service extraction, multi-region, more channels)

## Decision Drivers

1. **Fit to ADR-01 architecture**  
   - Must support layered architecture + background job runner + transactional outbox pattern.
   - Must integrate with S3-compatible file/object storage and AV scanner (HTTP callback).

2. **Meeting key NFRs**
   - **P01/P02**: High throughput REST API, efficient DB integration, async jobs for email/exports/AV.
   - **AV01/SC01**: Horizontal scaling (stateless app containers), DB connection pooling.
   - **S01–S03, DATA01, KMS01**: Mature security libraries (OIDC/OAuth2, TLS, encryption).
   - **OBS01/AUD01/SLO01**: Logging, metrics, tracing support.

3. **Developer productivity & learning curve**
   - I need to deliver a PoC plus documentation, not just a code spike.
   - Tooling and ecosystem should minimise accidental complexity.

4. **Alignment with course material & industry practice**
   - Stack should be recognised as “industry-standard” to strengthen the report.
   - Should align with examples from lectures (Spring Boot / layered architectures) and typical enterprise SaaS stacks.

5. **Portability & local development**
   - Must run on student laptops and SHU lab machines with minimal cloud dependencies.
   - Prefer dockerisable stack with local equivalents for cloud services (e.g. MinIO vs S3).

## Considered Options

1. **Java 21 / Spring Boot 3.x backend + React 18 (TypeScript) frontend + PostgreSQL 15 + S3-compatible storage (MinIO) + Redis + Docker** (**Chosen**)
2. **.NET 8 / ASP.NET Core + React + SQL Server + Azure Blob + Azure Cache for Redis**
3. **Node.js 22 / NestJS + React + PostgreSQL + S3-compatible storage + Redis**
4. **Serverless-first (AWS Lambda / API Gateway / DynamoDB / S3 / Cognito / SES)**

## Decision Outcome

**Chosen option**: Java / Spring Boot backend + React frontend + PostgreSQL + S3/MinIO + Redis + Docker.

This stack will be used as follows (mapping to C4 and ERD):

- **Web Application container** (C4 L2):
  - Java 21, Spring Boot 3.x, Spring MVC, Spring Security (OIDC + PKCE), Spring Data JPA.
  - Implements controllers/services/repositories for F01/F03/F04/F05/F06/F07/F08/F10/F11.
  - Persists to **PostgreSQL** using the ERD tables: `tenant`, `user_account`, `role`, `ticket`, `timeline_event`, `attachment`, `outbox_event`, `audit_log`, `email_template`, `tenant_config`, etc.
- **Background Job Runner container** (C4 L2):
  - Spring Boot application with scheduled tasks or Spring AMQP later.
  - Reads `outbox_event` and sends:
    - lifecycle emails via SMTP (F07),
    - CSV exports (F09),
    - AV status updates, etc.
- **Database container**:
  - PostgreSQL 15 with schema aligned to ERD and NFRs (indexes on `(tenant_id, status, updated_at)` and `(tenant_id, assignee_user_id, status)` for P02).
- **Object Storage container**:
  - MinIO (S3-compatible) in dev; S3-like APIs for pre-signed uploads (`attachment.object_key`, `av_status`).
- **Cache container**:
  - Redis for session storage, rate-limit counters (RL01), and hot queries if needed.
- **Frontend**:
  - React + TypeScript SPA for Consumer, Agent/Manager, and Admin consoles (C4 L1 & L2 Tier 1).

This option best supports the decision drivers and keeps the “golden thread” from case study → user stories → NFRs → C4 L1–L4 → ERD → PoC.

### Consequences

#### Good

- **NFR alignment**  
  - Spring Boot + PostgreSQL can comfortably meet **P01/P02** for typical bank/telecom volumes in a monolith.
  - Asynchronous jobs + outbox pattern support F07/F09/F11 and protect P01 by moving slow I/O off the request path.
  - Spring Security OIDC, JWT support and well-known hardening patterns help with **S01–S03**.

- **Excellent tooling & ecosystem**  
  - Mature libraries for:
    - OIDC + PKCE (sign-in with external IdP),
    - validation, error handling, testing (JUnit, Testcontainers),
    - observability (Micrometer, OpenTelemetry) for **OBS01/SLO01**.

- **Great fit for ERD and multi-tenancy**  
  - JPA/Hibernate map cleanly onto the ERD with `tenant_id` on every aggregate.
  - Transactions map directly to outbox pattern (single DB commit for domain + outbox rows).

- **Credible enterprise story**  
  - Stack is very common in regulated sectors; good story to VC / “ABC Limited” context.
  - React + REST API is mainstream and future-proof for adding mobile or chat-bot channels.

#### Bad (and mitigations)

- **Higher baseline resource usage than lightweight stacks**  
  - JVM + Spring Boot are heavier than a minimal Node or Go service.
  - **Mitigation**: Use Docker for local dev; keep dev profile lean; avoid unnecessary dependencies.

- **Two-language stack (Java + TypeScript)**  
  - Increases cognitive load for a single student.
  - **Mitigation**: Focus PoC depth on backend (F01 + F03 + notifications); keep frontend thin and wireframe-like but accessible (A01–A03).

- **Spring complexity for a small team**  
  - Many features; easy to misconfigure security or performance.
  - **Mitigation**: Follow recommended Spring Boot defaults; follow “production-ready” guidance but only enable what’s needed; document key config choices in other ADRs (auth, rate limiting, etc.).

## Pros and Cons of the Options

### 1) Java / Spring Boot / React / PostgreSQL / S3 / Redis (**Chosen**)

- **Pros**
  - Very common enterprise SaaS stack: credible for banks/telecoms, easier to justify in report.
  - Spring Boot directly supports layered architecture, transactional outbox, scheduled jobs and integration with SMTP, S3, Redis.
  - Strong security support: OIDC/OAuth2, CSRF, CORS, encryption best-practices.
  - Excellent testing tooling (JUnit, Testcontainers) supports NFR validation and design verification.
  - Maps cleanly to C4 L2/L3 and ERD tables; natural fit for multi-tenant PostgreSQL design.

- **Cons**
  - Learning curve and initial setup are non-trivial.
  - JVM footprint higher than e.g. serverless or Go; dev environment must be sized appropriately.
  - Requires juggling Java and TypeScript in one project.

---

### 2) .NET 8 / ASP.NET Core + React + SQL Server + Azure Blob + Azure Cache for Redis

- **Pros**
  - Modern, high-performance framework with excellent tooling (Visual Studio, Rider).
  - Built-in primitives for MVC, Razor, minimal APIs, authentication & identity.
  - Maps one-to-one to the ADR-01 architecture (monolith + background workers + SQL database).
  - SQL Server + Azure Blob + Azure Cache form a coherent Microsoft ecosystem, convenient on Azure.

- **Cons**
  - Stronger coupling to Windows/Azure ecosystem; less portable for mixed environments.
  - Course and labs focus more on Java/Spring style examples; extra time to adjust.
  - For PoC, spinning up Azure services adds overhead vs local Docker PostgreSQL/MinIO.

**Why not chosen now?**  
Technically a valid choice, but adds Azure + .NET ecosystem learning overhead without changing the architectural story. For a single-student PoC, Spring Boot + PostgreSQL is more aligned with course material and is simpler to run locally.

---

### 3) Node.js / NestJS + React + PostgreSQL + S3 + Redis

- **Pros**
  - Single language (TypeScript) across backend and frontend.
  - NestJS gives a structured framework (modules, providers, controllers).
  - Good fit for simple REST APIs and websockets if needed later (e.g. live updates).

- **Cons**
  - Less built-in support for enterprise patterns such as outbox, complex transaction handling.
  - Many choices for security, ORMs, background jobs (TypeORM/Prisma/Knex + Bull/Agenda, etc.) – more design surface for the same result.
  - Community solutions for observability, rate limiting, etc., are more varied in quality.

**Why not chosen now?**  
Although NestJS could implement the design, the number of decisions around ORMs, DI, job queues, etc., would dilute the focus of the report from architecture trade-offs to library selection. Spring Boot gives a more coherent “batteries included” story.

---

### 4) Serverless-first (AWS Lambda / API Gateway / DynamoDB / S3 / Cognito / SES)

- **Pros**
  - Very good horizontal scalability and pay-per-use pricing.
  - Many managed services: queues, email, monitoring, identity.
  - Potentially attractive long-term option once CMS is successful and event-driven.

- **Cons**
  - Strong vendor lock-in; architecture becomes tightly coupled to AWS primitives.
  - Cold starts and cross-function tracing complicate meeting **P01/P02** and **OBS01**.
  - Harder to demonstrate and test locally in SHU lab environment.
  - Data model would need to be re-designed for DynamoDB rather than relational ERD.

**Why not chosen now?**  
The assignment emphasises C4 + ERD + layered architecture. Serverless would shift a lot of complexity into infrastructure and AWS-specific services, distracting from the architectural reasoning. For the PoC, serverless would increase risk without improving the marking story.

## Overall why the alternatives were not chosen (critical engagement)

- **.NET 8 / ASP.NET Core stack** would deliver similar architectural benefits but at the cost of adopting a full Microsoft ecosystem (VS, Azure services) that is less aligned with the teaching context and my existing skillset. It does not significantly improve any NFR and would reduce the time available for deeper architectural analysis.

- **Node.js / NestJS stack** offers language unification (TypeScript everywhere) but requires more decisions and third-party tooling to match Spring Boot’s capabilities for transactional outbox, background jobs, and security. That extra decision surface does not improve the “golden thread” from NFRs to architecture and risks a shallower implementation.

- **Serverless-first stack** would require re-thinking the ERD for DynamoDB and designing function boundaries, queues, and orchestration (Step Functions, EventBridge, etc.). Cold starts and distributed tracing also complicate hitting p95 latency targets (P01/P02). This is interesting research but out-of-scope for a single-student L6 module with a PoC.

## Confirmation and Traceability

- **C4 L1:** CMS as a single web application with three UIs (Consumer, Agent/Manager, Admin) implemented in React, backed by a Spring Boot API. External systems (Organisation IdP, File Storage, AV Scanner, Email) are integrated via HTTPS and SMTP using this stack.

- **C4 L2:**  
  - `Web Application (Spring Boot)` + `Background Job Runner (Spring Boot)` + `PostgreSQL` + `MinIO` + `Redis` containers concretely implement the chosen ADR-01 quantum.  
  - Technology labels in the container diagram match this ADR.

- **C4 L3/L4:**  
  - Components (Request Gateway, Ticketing Module, File & Attachment Module, Auth/RBAC, Outbox Writer, Data Access Layer) are implemented using Spring MVC controllers, services, repositories and domain classes.  
  - Your L4 sequence diagrams (Create Ticket with AV gating; Agent Assign/Resolve flow) assume Spring controllers/services and relational transactions.

- **ERD:**  
  - PostgreSQL schema for `tenant`, `ticket`, `timeline_event`, `attachment`, `outbox_event`, `audit_log`, etc., assumes a relational database with ACID transactions, which is satisfied by PostgreSQL in this stack.

- **User Stories & NFRs:**  
  - This decision supports all user stories US1–US17 that require secure, transactional operations, multi-tenant isolation, and async notifications.  
  - NFRs mapped to containers (P01/P02, AV01, S01–S03, SC01, OBS01, AUD01, RL01, DR01, BKP01, KMS01, DATA01) are achievable with Spring Boot + PostgreSQL + Redis + S3/MinIO and standard libraries.

## References

Same reference set as ADR-01 (extended for technology-specific docs in other ADRs).

