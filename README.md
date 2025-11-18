# Disney Annual Pass System — Roadmap & Implementation Documentation

> Purpose: a step-by-step roadmap and living documentation to design, build, test, deploy, and operate a production-grade Annual Pass & Reservations platform (accounts, annual passes, banking/deposits, restaurant bookings, date blocking, photos, discounts, monitoring, and more).

---

## 1. Executive summary

This project implements a scalable, resilient backend platform to support Disney-style annual passes, reservations, payments, restaurants, photo services, 
discounts, and entitlement logic. The platform will be built with **Java 17**, **Spring Boot + Project Reactor (WebFlux)** for reactive microservices, 
cloud native infrastructure on **AWS** (ECS, RDS/MariaDB, DynamoDB, S3, ElastiCache, Kinesis), and standard industry practices for security, observability, and CI/CD.

Key goals:

* Provide robust account and annual pass lifecycle management (purchase, renewal, transfer).
* Support reservations that respect pass entitlements and park blocking rules.
* Secure payment flows and bank deposit integrations with an immutable ledger.
* Offer restaurant bookings, photo uploads, discounts, and admin tools.
* Provide strong observability and SLAs for production operations.

Audience: interviewers, technical leads, engineers, and stakeholders.

---

## 2. Scope & non-goals

**In-scope (MVP):**

* User registration/login with secure auth (JWT + refresh tokens)
* Purchase of annual passes and storage of pass metadata
* Simple reservation flow that enforces blocking rules per pass type
* Tokenized payment deposits (via 3rd-party PSP) with ledger entries
* Photo upload via signed S3 URLs and simple metadata store
* Basic restaurant booking entity and API
* Admin UI for pass types, blocking rules, and availability
* Observability: structured logs + basic AppDynamics or OpenTelemetry, CloudWatch dashboards

**Out-of-scope (post-MVP / future):**

* Full PCI storage of card data (we’ll use tokenization/hosted pages)
* Complex pricing models and dynamic bundling (initially static discounts)
* Full marketing/CRM integrations (later via events)
* Multi-region active-active deployment (start with multi-AZ)

---

## 3. High-level architecture (summary)

Components:

* API Gateway (AWS API Gateway) with JWT validation
* BFF/Edge service for web/mobile tailoring
* Microservices (Auth, Accounts, Passes, Reservations, Payments, Restaurants, Photos, EntitlementEngine)
* Databases: MariaDB (RDS) for transactional data, DynamoDB for high-scale lookups, S3 for media
* Cache: Redis (ElastiCache) for entitlement & availability caches
* Streaming: Kinesis for events, Lambda/ECS workers for async processing
* CI/CD: Jenkins or Harness, container images in ECR, deployed to ECS (Fargate) or EKS
* Observability: AppDynamics / OpenTelemetry, CloudWatch, Splunk

(Refer to the separate architecture diagram for process flows — included in appendix.)

---

## 4. Milestones & deliverables (roadmap)

No time estimates are included here — pick the order or ask me to propose durations.

**Milestone 0 — Foundation & Project Setup**

* Create monorepo / microservice repo layout
* Basic Spring Boot WebFlux skeletons for each service
* R2DBC connectivity to MariaDB and DynamoDB client scaffolding
* Terraform / CloudFormation baseline: VPC, subnets, RDS (MariaDB), S3, ECR, ECS cluster (or EKS)
* CI pipeline skeleton (build, test, containerize, push to ECR)
* Local dev: docker-compose or localstack scripts for dev environments

**Milestone 1 — Auth & Accounts (core)**

* User registration, login, JWT + refresh tokens
* Basic profile endpoints and account wallet stub
* Payment provider integration: tokenized card flows (sandbox)
* Unit & integration tests for auth flows

**Milestone 2 — Annual Passes & Entitlement Engine (core)**

* Pass types CRUD (admin) and purchase endpoint
* Pass lifecycle (activate, renew, suspend, transfer)
* Persist pass metadata in MariaDB; indexed lookups in DynamoDB
* Entitlement engine: rule model & evaluation API
* Cache entitlements per user/date in Redis

**Milestone 3 — Reservation Engine**

* Availability API, reservation creation with idempotency
* Concurrency control to avoid overbooking (DB transactions / optimistic locking)
* Blocking dates logic integration (EntitlementEngine)
* Reservation cancellation & waitlist

**Milestone 4 — Payments & Ledger**

* Secure deposit flows, webhooks, provider reconciliation
* Bank account linking (micro-deposit flow or Plaid integration stub)
* Append-only ledger in MariaDB for audit
* Reconciliation jobs and alerts for failed settlements

**Milestone 5 — Restaurant Bookings & Discounts**

* Restaurant catalog, menus, booking API
* Discount rules tied to pass types or promotions
* BFF patterns for aggregated UI responses

**Milestone 6 — Photos & Media Pipeline**

* Signed S3 upload flows, resizing/thumbnail worker
* Metadata store in DynamoDB or MongoDB
* Privacy features: opt-in sharing, face blur pipeline (future)

**Milestone 7 — Observability, QA & Hardening**

* AppDynamics/OpenTelemetry instrumentation
* Splunk/CloudWatch logging and alerting
* Load tests, security scans, penetration testing
* SLOs and runbook creation

**Milestone 8 — Admin UI & Operations**

* Admin pages for pass types, blocking rules, reconcile tools
* Operational dashboards and outage runbooks

**Milestone 9 — Extras & Scaling**

* Multi-region support, performance tuning, analytics pipeline
* Advanced promotions engine, loyalty integration

---

## 5. Technical design highlights

### 5.1 Service boundaries & responsibilities

* **AuthService**: handle identity, MFA, password reset, JWT issuance
* **AccountService**: wallet, ledger pointer, payment methods
* **PassService**: pass lifecycle, entitlement mapping
* **ReservationService**: availability, booking, cancellation
* **PaymentService**: provider adapters, ledger writes
* **PhotoService**: signed URLs, metadata, processing
* **EntitlementEngine**: rule evaluation, precomputation for cache

### 5.2 Data partitioning & storage choices

* Use MariaDB for ACID-critical operations: payments, reservations, ledger
* Use DynamoDB for fast entitlement lookups and photo metadata at scale
* Use Redis to cache entitlements and computed availability per (park, date)

### 5.3 Concurrency & idempotency

* Idempotency keys for reservations and payment endpoints
* Reservation writes in a transaction, optimistic locking or serializable isolation
* Use event sourcing patterns for critical money flows if later required

### 5.4 Eventing & eventual consistency

* Emit domain events to Kinesis: `PassPurchased`, `ReservationCreated`, `PaymentSettled`
* Event consumers update caches, notify external systems, and run async jobs

### 5.5 Security

* TLS everywhere, secrets in Secrets Manager, RBAC for admin APIs
* Tokenization for payment data; minimize PCI scope
* Data retention and privacy controls for photos and PII

---

## 6. Acceptance criteria for each milestone (examples)

**Milestone 0**

* Repos exist and build successfully in CI
* Dev environment boots locally with DB and Redis
* Terraform applies baseline infra to a dev account

**Milestone 1**

* Users can register and login with JWTs
* Integration tests for registration/login pass

**Milestone 2**

* Create/Read/Delete pass types via admin API
* User can purchase a pass and pass record exists in DB
* EntitlementEngine returns correct evaluation for sample rules

**Milestone 3**

* Reservation can be created concurrently without overbooking under a fixed capacity test
* Blocking rules prevent reservations for blocked pass types

(…and so on for subsequent milestones)

---

## 7. Developer workflow & repo conventions

* Mono-repo (optional) or multiple repos per service (recommended for independent deployments)
* Use semantic versioning and conventional commits
* Branching: feature branches -> PRs -> CI -> merge to main
* Pre-merge checks: unit tests, static analysis (SpotBugs, Checkstyle), contract tests
* Post-merge: build container, run integration tests, deploy to dev via CI

---

## 8. Testing strategy

* Unit tests with JUnit 5 and Reactor Test
* Integration tests with Testcontainers (MariaDB, Redis)
* Contract tests with Pact between services
* End-to-end tests against staging (real-ish infra)
* Load tests with Gatling or k6 to simulate seasonal peaks

---

## 9. Observability & runbooks

* Instrumentation: OpenTelemetry + AppDynamics for JVM traces and DB spans
* Logs: structured JSON -> Splunk or CloudWatch Logs
* Metrics: request latency, error rate, reservation success, payment failure rate
* Runbooks for common incidents: DB failover, cache stampede, payment provider outages

---

## 10. Security & compliance checklist

* TLS 1.2+ enforced
* Secrets in AWS Secrets Manager
* PCI scope minimized via tokenization
* Audit logs for finance events
* Data retention policy and customer data deletion
* Regular vulnerability scanning and dependency patching

---

## 11. Risks & mitigation

* **Surge capacity (peak days)**: use pre-compute + queueing + throttling and reserve capacity testing
* **Payment reconciliation mismatches**: implement daily reconciliation jobs and alerting
* **Blocking rules complexity**: start with a simple rules format and extend to a DSL only when necessary
* **Cache staleness**: event-driven invalidation + short TTL + background recompute

---

## 12. Appendix: Next immediate steps (recommended)

1. Approve this roadmap and pick the ordering of milestones (or ask me to suggest durations).
2. Start **Milestone 0**: I will generate the repo skeleton with Spring WebFlux modules (Auth, Pass, Reservation, Payment, Photo) plus a Terraform baseline and a CI Jenkinsfile.
3. After scaffolding, we will implement Milestone 1 (Auth & Accounts) end-to-end with unit and integration tests.

---

## 13. Interview talking points (how to present this architecture)

* Talk about **ACID** vs **eventual consistency** trade-offs and where each is applied.
* Mention idempotency, audit logs for payments, and how you avoid overbooking.
* Explain caching strategy (Redis + precomputation) and cache invalidation via events.
* Highlight monitoring plans: AppDynamics for slow JVMs, Splunk for logs, CloudWatch for infra.
* Show you can deliver a working MVP quickly and iterate on complexity.

---

If you want, I can now:

* produce the **repo skeleton** (Java WebFlux services + build files + Dockerfiles + sample CI), or
* generate the **Terraform baseline** for VPC, RDS (MariaDB), ECR and ECS cluster, or
* **start implementing Milestone 1** (AuthService) with tests and sample DB migrations.

Pick one and I’ll generate code + configs right away.
