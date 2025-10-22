---
status: review
date: 02/10/2025
decision-makers: Marim Sabah (Architect)
consulted: Module tutor
informed: Module tutor
---

## Context and Problem Statement
I have to implement a Complaint Management System (CMS) for five roles (Consumer, Help Desk Agent, Support Person, Help Desk Manager, System Administrator) and meet key NFRs:
- **P01/P02** performance (Create p95 < 1.5 s, Read p95 < 300 ms)
- **AV01** availability (read only when IdP is down)
- **S01/S02/S03** security, SSO, privacy (GDPR)
- **SC01** scalability, OBS01 observability, AUD01 auditability


**Problem**: Choose an architecture that keeps the happy path synchronous and fast, pushes variable I/O (email, exports, AV) off-path, is simple to operate now, and maps cleanly to C4 L1/L2 and our test plan.

Architectural quantum. I will be adopting a single architectural quantum (one application + job runner sharing one database) to optimise operational simplicity and transactional integrity, consciously trading off independent deployability you’d get from multiple quanta (microservices) [1]


## Decision Drivers

**Performance (P01)**: The user-facing synchronous path for creating/updating tickets must have a p95 latency under 1.5s. This mandates moving slow I/O operations (email, AV scanning) to an asynchronous, non-blocking flow. [2]
**Simplicity and Low Operational Cost**: Minimize the number of deployable units to reduce operational complexity, simplify CI/CD, and streamline observability. [1][2]
**Transactional Integrity**: Guarantee atomicity for core business operations. For example, creating a ticket and scheduling its notification email must succeed or fail together to prevent data inconsistency. [6]
**Security & Auditability (S01, AUD01)**: Centralize the enforcement of tenant isolation and security controls within a single, well-defined boundary to simplify auditing and reduce the attack surface. [2]
**Evolvability**: The architecture must be pragmatic for initial delivery but not a dead end. It should allow for future, targeted scaling if specific components become bottlenecks.

## Considered Options

1. Three-Tier Layered Web App + Background Jobs (Transactional Outbox): Chosen
Presentation (web UIs) → Application (Spring Boot) → Data (PostgreSQL, Object Storage, Redis). We persist domain state and an outbox record in one DB commit; a job runner drains the outbox to send emails, build CSVs, and finalize AV scan statuses (a broker can be added later) [6], [7]
2. Microservices (tickets, users, files, notifications, reporting) [1]
3. Serverless-first (functions + managed queues + cloud storage).
4. Two-tier synchronous (UI→App→DB; everything inline)
   
## Decision Outcome

Chosen option: Three-Tier Layered Web App with Background Jobs (Transactional Outbox).

**Meets Performance (P01)**: The Transactional Outbox pattern allows us to commit a business entity (`Ticket`) and its corresponding event (`TicketCreatedEvent`) to the database in a single atomic transaction. A separate background worker process polls this outbox table, ensuring that slow operations like sending emails (F07) or processing file scans (F11) happen asynchronously and do not impact the synchronous API response time. [2], [6], [7]

**Guarantees Reliability**: Unlike a dual-write approach (writing to the DB and then a message broker), the outbox pattern prevents data loss if the system fails after the DB commit but before the message is sent. This provides a strong consistency guarantee without the complexity of distributed transactions (2PC).

**Simplifies Security (S01)**: With a single deployable API, tenant isolation and RBAC logic are centralized in one place (e.g., middleware, repository decorators). This creates a single, auditable enforcement point, drastically reducing the risk of cross-tenant data leakage compared to a distributed architecture where every service would need to reimplement or share this logic.

**Lowers Operational Overhead**: A single application and database are fundamentally simpler to deploy, monitor, and manage than a distributed fleet of services, which would require a service mesh, distributed tracing infrastructure, and complex CI/CD orchestration from day one.
  
### Consequences

### Good
**High Cohesion**: Business logic is co-located, making it easier to refactor and maintain.
**Simplified Observability (OBS01)**: Tracing a request is trivial as it remains within a single process boundary, avoiding the need for complex distributed tracing correlation across network calls.
**Strong Consistency**: Core operations remain ACID-compliant within the PostgreSQL database, which is the simplest and most robust model for the defined functional requirements.

### Bad (and our Mitigation Strategy)
**Risk of Database Contention**: The `Ticket` table could become a hotspot with frequent updates from consumers and agents.
**Mitigation**: We will implement **optimistic locking** using a `version` column on the `Ticket` entity. This prevents lost updates by checking the version on write, failing fast, and forcing the client to retry, which is a safer contention management strategy than pessimistic locking for web-scale workloads.

**Coarse-Grained Scalability**: The entire application must be scaled as a single unit, even if only one part (e.g., reporting) is under heavy load.
**Mitigation**: We will enforce strict module boundaries within the monolith using tools like **ArchUnit**. This prevents coupling and ensures that if a module like `Reporting` needs to be extracted into a separate service later, the codebase is already prepared for this evolution. This makes future decomposition a planned activity, not a costly rewrite.

**Single Point of Failure Risk**: A critical bug in the monolith could bring down the entire system.
**Mitigation**: This is a standard characteristic of the architectural quantum. The mitigation is operational, not architectural: the stateless web application will be deployed as **multiple redundant instances (≥3)** behind a load balancer with health checks. This ensures that the failure of a single instance does not result in system downtime, satisfying the availability requirement (AV01).

  
### Confirmation
- **C4 L1:** One **Web Application** serving five roles; externals: **Organisation IdP** (SSO), **Email (SMTP)** (notifications), **File Store** (pre-signed uploads), **AV Scanner** (callback), **Monitoring** (SLOs/alerts). Tenant isolation is enforced centrally.[4][5]
- **C4 L2 (containers):** [4][5][6][7].
  - **Web Application (MVC)** — request/response, tenancy & RBAC, domain logic, AV callback endpoint  
  - **Background Job Runner** — processes DB outbox (emails, CSV exports, AV status updates)  
  - **PostgreSQL** — OLTP + Outbox + Audit (append-only)  
  - **File/ Object Store** — S3/MinIO (pre-signed PUT/GET; lifecycle/retention per tenant)  
  - **Redis (optional)** — cache/rate limits  
  - **AV Scanner** — ClamAV callback to Web App  
  - **Email (SMTP)** — outbound notifications  
- **Tests planned:** **NFT-P01/P02**, **NFT-AV01**, **NFT-SEC01/02**, **NFT-OBS01**, **NFT-AUD01** align to the above containers and NFRs.
- **Traceability:** F01/F03/F07/F09/F11 map to this style; NFR scopes in the table use the same container names, ensuring doc/diagram/test consistency.



## Pros and Cons of the Options

### 1) Three-Tier + Background Jobs (**Chosen**)
- **Good:** Simple ops; local transactions; outbox makes async reliable; easy to test and explain.  
- **Neutral:** Requires discipline to keep internal modules decoupled.  
- **Bad:** Coarser scaling; may later add a broker if job volume grows.

### 2) Microservices
- **Good:** Independent scaling/deploys; strong bounded contexts.  
- **Neutral:** Needs mature platform and team capacity.  
- **Bad:** Distributed transactions, network hops, higher ops burden for this scope/timebox.

### 3) Serverless-first
- **Good:** Scales to zero; managed queues; minimal server management.  
- **Neutral:** Cloud/vendor coupling; more complex local/dev story.  
- **Bad:** Cold starts; cross-function tracing; consistency still distributed.

### 4) Two-tier synchronous
- **Good:** Easiest to start.  
- **Neutral:** Acceptable only for very small loads.  
- **Bad:** Fails **P01** (blocking email/AV); brittle under load; poor separation of concerns.

## Overall why the alternatives were not chosen (critical engagement)

- Microservices now: Multiple quanta would force cross-service sagas (Ticket - Files - Notifications), adding failure modes and network hops that hurt P01 and team velocity unnecessary until metrics demand independent scaling [1] [2]

- Serverless-first: Cold starts + tracing seams complicate p95 guarantees and observability. Strong vendor coupling for DSAR/export pipelines adds risk without offsetting gains at our scale. [2]

- Two-tier synchronous: Blocks on email/exports/AV; breaks P01 quickly and degrades UX under modest load.[2]

## More Information (patterns & sourcing)
- **Patterns:** Layered architecture; **Transactional Outbox**; Pre-signed uploads; AV callback; OIDC + PKCE; Background jobs; Rate limiting.
- **Maps to this project:** F01/F03/F07/F09/F11; P01/P02, AV01, S01/S02/S03, SC01, OBS01, AUD01.

**References:** 
[1] M. Richards & N. Ford, Fundamentals of Software Architecture: An Engineering Approach, O’Reilly, 2020 — architectural quanta, trade-off analysis.
[2] L. Bass, P. Clements, R. Kazman, Software Architecture in Practice, 4th ed., Addison-Wesley, 2021 — quality attribute tactics (asynchrony, queues, bulkheads, throttling, monitoring).
[3] G. Fairbanks, Just Enough Software Architecture: A Risk-Driven Approach, Marshall & Brainerd, 2010 — risk-driven decision-making.
[4] S. Brown, “The C4 Model for Visualising Software Architecture,” c4model.com — C4 Context/Container views and consistency.
[5] P. Kruchten, “Architectural Blueprints—The ‘4+1’ View Model of Software Architecture,” IEEE, 1995 — multi-view documentation.
[6] M. Fowler, “Transactional Outbox” & “Reliable Messaging,” martinfowler.com — outbox pattern rationale and pitfalls.
[7] Debezium, “Outbox Pattern,” debezium.io — implementation guidance and exactly-once semantics.
