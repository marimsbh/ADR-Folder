# ADR-02 – Technology Stack

status: review  
date: 09/10/2025  
decision-makers: Marim Sabah (Architect)  
consulted: Module tutor  
informed: Module tutor  

## Context and Problem Statement

The CMS must be delivered as a proof-of-concept but also be credible as a
production-ready SaaS platform. The case study requires:

- Web access for five roles (Consumer, Help Desk Agent, Support Person, Help
  Desk Manager, System Administrator).
- 24/7 availability for online channels and WCAG 2.x accessibility.
- Integration with external systems (Organisation IdP, File/Object Storage,
  AV Scanner, Email System).
- A 10% projected yearly increase in users, starting from large enterprises
  like NatWest/Barclays with ~20M customers.
- Implementation of non-functional requirements P01/P02 (latency), AV01
  (availability), S01–S03 (security/privacy), SC01 (scalability), OBS01
  (observability), AUD01 (auditability), FRONT01 (web performance).

A technology stack is needed that:

- Supports Java-based layered architecture chosen in ADR-01.
- Has strong ecosystem support for HTTP APIs, security, and database access.
- Is realistic for a student proof-of-concept yet representative of industry.

## Decision Drivers

- **Alignment with architectural style:** Needs robust support for layered web
  app + background jobs + relational DB + outbox.
- **Developer productivity:** Strong frameworks and documentation.
- **Ecosystem & maturity:** Libraries for OIDC, AV integration, S3-compatible
  storage, SMTP, etc.
- **Fit with NFRs:** Performance, observability, rate limiting, security.
- **Realistic in industry:** Technologies commonly used in enterprise SaaS.

## Considered Options

1. **Java / Spring Boot backend + React frontend + PostgreSQL + S3/MinIO + Redis + Docker**  
   (Chosen)

2. **.NET 8 / ASP.NET Core backend + React frontend + SQL Server + Azure Blob + Redis**

3. **Node.js / NestJS backend + React frontend + PostgreSQL + S3/MinIO + Redis**

4. **Full serverless stack (AWS Lambda + API Gateway + DynamoDB + S3 + Cognito)**

## Decision Outcome

**Chosen option: Java / Spring Boot + React + PostgreSQL + S3/MinIO + Redis + Docker.**

- **Backend:** Spring Boot 3.x (Java 21)  
  Provides MVC controllers, validation, Spring Data JPA for PostgreSQL,
  Spring Security for OIDC/OAuth2, and Spring AMQP/scheduling for background
  jobs (used for the outbox worker).

- **Frontend:** React SPA for Consumer Web App, Agent/Manager Console, Admin
  Console. This aligns with the C4 L2/L3 diagrams where Tier 1 is “Web app
  (React)”.

- **Database:** PostgreSQL 16 for OLTP, shared-schema multi-tenancy, append-only
  audit log, and outbox_event table.

- **Cache & rate limiting:** Redis as infrastructure cache (sessions, rate-limit
  counters, hot lists).

- **File/Object Storage:** MinIO (S3-compatible) for attachments and CSV
  exports, accessed via pre-signed URLs.

- **AV Scanner:** ClamAV exposed via HTTP callback.

- **Email System:** Any SMTP-compatible transactional email provider
  (SendGrid-like abstraction), called by background jobs.

- **Containerisation:** Docker images for Web Application, Background Job
  Runner, PostgreSQL, MinIO, ClamAV (local dev); deployable to Kubernetes or
  container platform in future.

This stack is reflected in the C4 L2 container diagram:

- Web Application (Spring Boot)
- Background Job Runner (Spring Boot worker)
- Database (PostgreSQL)
- Cache (Redis)
- File/Object Storage (S3/MinIO)
- AV Scanner (ClamAV)
- Email System
- Organisation IdP (OIDC / SSO)

## Consequences

### Positive

- **Strong alignment with ADR-01:** Spring Boot naturally supports layered
  architecture and the Transactional Outbox pattern.
- **Security and SSO:** Spring Security has first-class OIDC + PKCE support,
  making S01/S02 realistic in the proof-of-concept.
- **Performance and scalability:** Java + PostgreSQL + Redis is a proven stack
  for low-latency CRUD workloads (P01/P02, SC01).
- **Developer productivity:** Spring Data JPA, migrations (Flyway/Liquibase),
  and React component libraries reduce boilerplate.
- **Portability:** S3-compatible storage (MinIO) + Dockerised stack avoid
  vendor lock-in.

### Negative / Trade-offs

- **Learning curve:** Spring Boot + React + Docker is heavier than e.g. Node.js
  or pure serverless for a small team; mitigated by focusing PoC only on F01
  and F03 plus selected NFRs.
- **Operational overhead vs serverless:** Requires container orchestration and
  monitoring; mitigated by keeping a single architectural quantum and using
  simple Docker Compose for PoC.
- **TypeScript vs Java:** Type safety is split between Java and TypeScript;
  this is mitigated by using OpenAPI-generated client types for the React app.

## Traceability

- **Requirements:** F01–F12, P01/P02, AV01, S01–S03, SC01, OBS01, AUD01,
  FRONT01.
- **C4:** L2 containers (Web Application, Background Job Runner, Database,
  Cache, File Store, AV Scanner, Email System, IdP).
- **ERD:** Implemented in PostgreSQL according to the CMS ERD.
- **ADRs:** Builds directly on ADR-01 and is referenced by ADR-03, ADR-04,
  ADR-05, ADR-07, ADR-08, ADR-09, ADR-10, ADR-11.

