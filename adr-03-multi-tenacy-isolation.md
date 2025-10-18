---
status: review
date: 13/10/2025
decision-makers: Marim Sabah (Architect)
consulted: Module tutor
informed: Module tutor
---

## Context and Problem Statement
CMS is multi‑tenant. Users from one organisation must never access another tenant’s data. I need strong isolation with good performance and manageable operations, plus GDPR support (DSAR, retention).

## Decision Drivers

- **Security (S01):** hard tenant isolation; defense‑in‑depth.
- **Performance (P02):** indexed queries across large datasets.
- **Operations:** simple provisioning; single schema migrations.
- **Compliance (S03/AUD01):** auditability; targeted exports/deletions.
- **Cost and complexity:** appropriate for small team and Task‑2 PoC.

## Considered Options

1. **Separate database per tenant**  
2. **Separate schema per tenant in one database**  
3. **Shared schema with `tenant_id` column + application checks only**  
4. **Shared schema with `tenant_id` + database Row Level Security (RLS) (optional)** **chosen**
   
## Decision Outcome

Chosen option:  **Shared schema + `tenant_id`** everywhere, **unique constraints include `tenant_id`**, **all repositories filter by tenant**, and **optional PostgreSQL RLS** enabled for sensitive tables (tickets, comments, attachments, audit). Because:
- Best balance of performance, cost, and simplicity; single migration path.
- Clear **C4 L2** mapping (API enforces tenant at handler/repo level; DB can additionally enforce via RLS policies).
- Indexes on `(tenant_id, status, updated_at)` satisfy **P02**; DSAR/export can filter by tenant efficiently.
  
### Consequences

* Good, because
- Simple onboarding: create tenant row + config; no new DB or schema.
- Query performance predictable with composite indexes.
- Defense‑in‑depth: app filter + optional RLS policies + tests (NFT‑SEC01).
  
* Bad, because
- Misconfigured queries could omit tenant filter if not guarded by repositories.
- RLS adds complexity to admin scripts and migrations; needs careful roles.
- Noisy neighbor potential at very high scale (mitigated by partitioning later).

### Confirmation
- Contract tests assert tenant scoping at repository layer.  
- Negative tests (NFT‑SEC01) prove cross‑tenant reads/writes are denied.  
- Optional RLS policies validated with unit/integ tests on DB roles.


## Pros and Cons of the Options

### 1) DB per tenant
- Good: strongest physical isolation; per‑tenant backup/restore.  
- Neutral: cost grows linearly with tenants.  
- Bad: heavy ops; complex reporting/analytics across tenants.

### 2) Schema per tenant
- Good: moderate isolation; shared instance.  
- Neutral: still many schemas to manage.  
- Bad: migration complexity; tooling friction.

### 3)  App checks only (no RLS)
- Good: simplest DB config.  
- Neutral: fast to start.  
- Bad: single point of failure in app code; weaker defense‑in‑depth.

### 4) Shared schema + RLS (Chosen)
- Good: balanced; DB enforces policies; indexes simple.  
- Neutral: admin complexity manageable with roles.  
- Bad: requires discipline and good test coverage.

## More Information
- PostgreSQL RLS, policies per role; composite unique keys with `tenant_id`.  
- Audit design includes `tenant_id` and immutable events.

---

## Mapping to this Project
- **FRs:** F01–F11 (data isolation everywhere).  
- **NFRs:** S01, S03, AUD01, DATA01, P02.  
- **C4:** L2 API/DB; L3 repositories; optional DB RLS policies.
