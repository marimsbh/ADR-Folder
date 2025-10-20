---
status: review
date: 02/10/2025
decision-makers: Marim Sabah (Architect)
consulted: Module tutor
informed: Module tutor
---

## Context and Problem Statement
I have to implement the Complaint Management System (CMS) for five roles (Consumer, Help Desk Agent, Support Person, Help Desk Manager, System Administrator) with strong non functional guarantees:
- **P01/P02** performance (Create p95 < 1.5 s, Read p95 < 300 ms)
- **AV01** availability (read only when IdP is down)
- **S01/S02/S03** security, SSO, privacy (GDPR)
- **SC01** scalability, OBS01 observability, AUD01 auditability


**Problem**: Choose an architectural style that delivers FRs (F01–F12) and these NFRs with low operational complexity, high cohesion, and clear mapping to the container and component models.



## Decision Drivers

- Meet P01 and keep creates non‑blocking (email, AV) = asynchronous processing
- Avoid cross‑service transactional complexity early = fewer deployable units
- Enforce tenant isolation (S01) in one place (repos/filters; optional RLS)
- Enable observability and audit from day one (OBS01, AUD01)
- Deliver a Task‑2 PoC quickly but in a way that scales to production (SC01)

## Considered Options

1. **Microservices** (tickets, users, files, notifications, reporting)  
2. **Modular Monolith API and Async Workers** (outbox to broker) **chosen**  
3. **Single Monolith, synchronous only**  
4. **Serverless first** (functions + managed queues)
   
## Decision Outcome

Chosen option:  “Modular Monolith API + Async Workers (Outbox to RabbitMQ)”, because it provides:
- **Strong cohesion** in one deployable API while enforcing module boundaries in code
- **Asynchronous** flows for email, exports, AV (keeps P01 create latency low)
- **Simpler transactions**: DB writes and outbox in the same commit = reliable event delivery
- **Lower ops burden** than microservices, clearer learning curve for Task‑2
  
### Consequences

* Good, because
- Meets P01 with async notifications/AV; keeps UI snappy
- Easier to prove AUD01 (single audit log); OBS01 (central tracing)
- S01 simpler (one API enforces tenant filters; optional RLS later)
- Clean mapping to C4 L2 (API, Workers, RabbitMQ, DB, S3, IdP, Email, AV)
  
* Bad, because
- One deployable unit → potential hotspots; requires discipline on internal module boundaries
- Future independent scaling per module needs extra work (profiling + split)
- Reporting still off‑path (workers/export) to keep OLTP fast

### Confirmation
- **C4 L2** shows *API Application*, *Worker—Notifications*, *Worker—Exports*, *RabbitMQ*, *PostgreSQL (Outbox + Audit)*, *S3 (pre‑signed)*, *ClamAV (callback)*, *IdP*, *Email Provider*.   
- Tests planned: **NFT‑P01/P02**, **NFT‑AV01**, **NFT‑SEC01/02**, **NFT‑OBS01**, **NFT‑AUD01**.  
- Task‑2 PoC will load‑test creates and demonstrate outbox to broker delivery.


## Pros and Cons of the Options

### 1) Microservices
- **Good, because**: Independent scaling/deploys; strong isolation per bounded context.  
- **Neutral, because**: Requires mature platform/ops; may be better later.  
- **Bad, because**: Distributed transactions, service discovery, network latency; heavy for Task‑2.

### 2) Modular Monolith + Async Workers (**Chosen**)
- **Good, because**: Simple ops; transactions are local, async keeps p95 low, easy testing.  
- **Neutral, because**: Needs discipline to avoid coupling, still one deployable unit.  
- **Bad, because**: Scaling permodule is coarse; future extraction effort if needed.

### 3) Single Monolith (sync only)
- **Good, because**: Simplest deployment, easiest to start.  
- **Neutral, because**: Okay for very small scope only.  
- **Bad, because**: Fails P01 (blocking email/AV), brittle under load, poor separation of concerns.

### 4) Serverless‑first
- **Good, because**: Scales to zero; managed queues; low ops.  
- **Neutral, because**: Vendor coupling; local dev more complex.  
- **Bad, because**: Cold starts; tracing/debug harder; still distributed consistency issues.

## More Information
- Patterns used: **Outbox**, **CQRS-lite for exports**, **Pre‑signed uploads**, **Callback for AV**, **OIDC + PKCE**.  
- See report sections: **C4 L1/L2**, **NFR Table**, **Security**, **Observability**, **API**.  
- FR/NFR Mapping: F01/F07/F11 (async + files), F09 (exports), **P01/P02**, **S01/S02/S03**, **OBS01**, **AUD01**, **SC01**.

---

## Mapping to this Project
- **FRs**: F01, F02, F03, F04, F05, F06, F07, F08, F09, F10, F11 (primary), F12 (optional)  
- **NFRs**: P01, P02, AV01, S01, S02, S03, SC01, OBS01, AUD01, SLO01, RES01  
- **C4**: L1 (externals); L2 (API, Workers, RabbitMQ, PostgreSQL, Redis, S3, IdP, Email, AV, Monitoring)  
