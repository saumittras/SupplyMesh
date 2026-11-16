# Deliverable 05 — Architecture Decision Records (ADRs)
# SupplyMesh: Global Procurement & Supplier Intelligence Platform

---

## Overview

This document records the 10 most significant architectural decisions made for SupplyMesh. Each ADR explains the problem, options considered, the decision made, and the trade-offs accepted.

> **Important**: These decisions are requirements-driven. Each choice is justified by the specific needs of a procurement SaaS platform — not by personal technology preferences.

---

## ADR-001: PostgreSQL over MongoDB

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
SupplyMesh manages complex, deeply relational data: Organizations → Departments → Users → Purchase Requests → Approval Steps → Purchase Orders → Invoice Items → Payments. Financial workflows require strict data consistency, referential integrity, and atomic multi-table transactions.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL (Selected)** | ACID transactions, FK constraints, mature tooling, row-level security, powerful indexing, JSON support for flexible fields | Vertical scaling limitations (solvable), schema migrations require care |
| MongoDB | Flexible schema, easy horizontal sharding, good for document-shaped data | No multi-document ACID transactions until v4+ (and they're complex to use), poor for complex joins, referential integrity is application-level only |
| MySQL | Widely known, fast reads | Fewer advanced features (partial indexes, JSONB, RLS), weaker JSON support |

### Decision
**PostgreSQL 15+**

### Reasoning
SupplyMesh's core data — POs, invoices, payments, approval chains — is fundamentally relational. Consider this scenario: when a PO is closed, we need to atomically update the PO status, create a performance record, and trigger an invoice match — all in one transaction. If any step fails, the entire operation must roll back. PostgreSQL handles this natively. With MongoDB, we'd need application-level saga orchestration for every financial operation.

Additionally, our multi-tenant isolation strategy uses PostgreSQL Row-Level Security — a feature not available in MongoDB.

### Trade-offs Accepted
- PostgreSQL requires careful schema migration management (mitigated by Drizzle ORM's migration system)
- Scaling PostgreSQL horizontally is more complex than MongoDB (mitigated by read replicas and future sharding strategy)

---

## ADR-002: Drizzle ORM over Prisma

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
We need a type-safe database access layer for TypeScript/NestJS that integrates well with PostgreSQL and provides good performance visibility.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Drizzle ORM (Selected)** | SQL-first (you write SQL-like queries), zero-overhead abstraction, full TypeScript inference, lightweight, works well with NestJS | Smaller community than Prisma, less auto-magic |
| Prisma | Excellent DX, Prisma Studio, large community, auto-generated types | Heavy query engine (Rust binary), "black box" SQL generation (hard to optimize), Prisma's connection pooling is PrismaClient-based (not PgBouncer-compatible), schema changes require learning Prisma's DSL |
| TypeORM | Familiar for Spring/Hibernate developers, decorators-based | Complex, inconsistent API, performance issues, poor PostgreSQL-specific feature support |
| Raw SQL (pg) | Maximum control | No type safety, verbose, maintenance burden |

### Decision
**Drizzle ORM**

### Reasoning
SupplyMesh contains complex financial queries — spend aggregations, multi-join analytics, approval chain resolution. With Prisma, generated SQL can be unpredictable for complex queries; developers end up using `$queryRaw` anyway. With Drizzle, the SQL is explicit and auditable from day one. For a financial platform where performance and correctness matter, SQL-first is the right approach.

Drizzle also integrates naturally with PgBouncer for connection pooling, which is essential for a multi-tenant SaaS platform where we might have hundreds of concurrent organizational queries.

### Trade-offs Accepted
- No Prisma Studio (mitigated by using TablePlus or pgAdmin for DB inspection)
- Smaller ecosystem (Drizzle is growing rapidly; this is an acceptable trade-off)

---

## ADR-003: Modular Monolith over Microservices

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
How should the SupplyMesh application be structured? The choice determines team velocity, operational complexity, and future scalability.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Modular Monolith (Selected)** | Simple deployment, database transactions across modules, single codebase, easy debugging, low DevOps overhead | Cannot scale modules independently (acceptable at this scale), one deployment unit means one failure domain |
| Microservices | Independent scaling, independent deployment, technology diversity | High operational overhead (service discovery, distributed tracing, inter-service auth), distributed transactions (Sagas) required for PO/Invoice/Payment consistency, significant DevOps maturity needed |
| Layered Monolith | Simple | No domain boundaries → becomes a "big ball of mud" over time |

### Decision
**Modular Monolith with strict module boundaries**

### Reasoning
Consider the PO → Invoice → Payment workflow. This requires a database transaction across three entities. In microservices, this would require a distributed saga with compensating transactions — dramatically increasing complexity and failure surface area for a team building an MVP.

Furthermore, our domain boundaries are still being discovered. The classic microservices mistake is decomposing too early, before the domain model is stable. A modular monolith lets us establish clear boundaries (each NestJS module = one domain) while retaining the simplicity of a single process. When the Notification or Analytics module clearly needs independent scaling, it can be extracted with minimal refactoring.

### Trade-offs Accepted
- Cannot independently scale the Workflow Engine if it becomes a bottleneck (mitigated by horizontal scaling of the entire application)
- One deployment unit means all modules deploy together (mitigated by feature flags and trunk-based development)

---

## ADR-004: Redis for Caching and Job Queues

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
Does SupplyMesh need Redis? What specific problems would it solve?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Redis (Selected)** | Sub-millisecond reads, pub/sub, sorted sets for queues, battle-tested for BullMQ, session storage | Additional infrastructure component to manage |
| No Redis (PostgreSQL only) | Simpler infrastructure | SKIP — PostgreSQL-based job queues (pg-boss) are slower and add write pressure to primary DB; no good session cache alternative |
| Memcached | Simple cache | No persistence, no pub/sub, no sorted sets (can't do BullMQ queues) |

### Decision
**Redis for two purposes:**
1. **Cache**: User permissions, org settings, supplier lists, analytics KPIs
2. **Job Queue**: BullMQ for background job processing (approval escalation, notifications, performance scoring)

### Reasoning
**Cache use case**: Permission checks happen on every authenticated request. Loading user roles and permissions from PostgreSQL on every API call is wasteful. Redis caches this data with a TTL matching the JWT lifetime.

**Queue use case**: Several operations in SupplyMesh should not block the HTTP response:
- Sending email notifications (supplier emails, approval alerts)
- Recalculating supplier performance scores
- Running the approval SLA escalation check
- Writing audit logs

BullMQ (backed by Redis) is the industry standard for Node.js job queues. Using PostgreSQL as a job queue (via pg-boss) would add significant write pressure to the primary database and is slower.

### Trade-offs Accepted
- One additional infrastructure component (Redis container) — acceptable; Docker Compose makes this trivial

---

## ADR-005: Multi-Tenancy via Shared Schema with organization_id Row Isolation

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
How should SupplyMesh isolate data between organizations (tenants)?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Shared Schema + org_id (Selected)** | Simplest deployment, cost-effective, easy cross-tenant analytics for SaaS operator, single schema to maintain | Risk of data leakage if `organization_id` filter is missed in a query |
| Separate Schema per Tenant | Good isolation, schema-per-tenant allows per-tenant customization | Complex migration management (N schemas to migrate), expensive for many small tenants, complex connection routing |
| Separate Database per Tenant | Perfect isolation | Very expensive, operational nightmare (thousands of databases), impractical for MVP |

### Decision
**Shared Schema with `organization_id` column on all tenant-scoped tables**

### Three-Layer Enforcement:
1. **NestJS Middleware**: Extracts `orgId` from JWT and injects into request context
2. **Service Layer**: Every database query includes `WHERE organization_id = :orgId` — enforced by code review and linting rules
3. **PostgreSQL RLS** (backup safety net): Row-Level Security policies ensure even a compromised query cannot read cross-tenant data

```sql
-- RLS Policy example
ALTER TABLE purchase_orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON purchase_orders
  USING (organization_id = current_setting('app.current_org_id')::uuid);
```

### Trade-offs Accepted
- Application code must always include `organization_id` filter (mitigated by RLS as safety net + linting rules)
- Large tenants share DB resources with small tenants (mitigated by future shard-by-org-id strategy)

---

## ADR-006: Approval Workflow Rules Stored as JSONB

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
Different organizations have different procurement approval policies. How do we store these configurable workflow rules?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **JSONB in workflow_templates (Selected)** | Flexible schema, no migration needed for new rule types, queryable JSON in PostgreSQL | Requires application-level schema validation; no DB-level schema enforcement |
| Dedicated workflow rules tables (EAV model) | Fully relational | Extremely complex queries; Entity-Attribute-Value is an anti-pattern for rule storage |
| External workflow engine (e.g., Camunda, Temporal) | Powerful, visual workflow designer | Huge operational overhead; external dependency; overkill for procurement approval rules |
| Hardcoded thresholds in application code | Simple | Cannot be customized per organization; requires code deployment for every policy change |

### Decision
**JSONB in `workflow_templates.rules`**

### Reasoning
Procurement approval rules have a well-defined but flexible structure: a list of steps, each with a threshold, required approver role, and SLA. This is naturally represented as a JSON array. JSONB in PostgreSQL allows us to query and validate these rules while keeping the schema flexible for future rule types (e.g., parallel approvals, geographic rules, budget-code-based routing) without database migrations.

The rule schema is validated at the application layer using Zod before storage.

### Rule Schema (Validated with Zod)
```typescript
const WorkflowRuleSchema = z.object({
  steps: z.array(z.object({
    step: z.number().int().positive(),
    name: z.string(),
    role: z.enum(['department_manager', 'procurement_manager', 'finance_officer']),
    min_amount: z.number().optional(),
    max_amount: z.number().nullable(),
  })),
  escalation_hours: z.number().default(24),
});
```

### Trade-offs Accepted
- No DB-level schema enforcement (mitigated by application-layer Zod validation before any write)

---

## ADR-007: Maintaining PO and Invoice Consistency

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
The PO → Invoice → Payment chain involves multiple tables. How do we ensure data consistency when any step in this chain fails?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **PostgreSQL ACID Transactions (Selected)** | Native database-level atomicity, simple to implement, no distributed coordination | Only works within a single database (fine for our monolith) |
| Application-level saga | Works across services | Complex compensating transaction logic, error-prone |
| Event sourcing | Full audit trail built-in | Extreme complexity, steep learning curve |

### Decision
**PostgreSQL ACID Transactions for all PO/Invoice/Payment operations**

### Implementation Approach

```typescript
// All financial state changes wrapped in transactions
await db.transaction(async (tx) => {
  // 1. Approve invoice
  await tx.update(invoices).set({ status: 'approved', reviewedBy: userId })
    .where(eq(invoices.id, invoiceId));

  // 2. If fully paid, close the PO
  const totalPaid = await tx.select({ sum: sum(payments.amount) })
    .from(payments).where(eq(payments.invoiceId, invoiceId));

  if (totalPaid >= invoice.total_amount) {
    await tx.update(invoices).set({ status: 'paid' });
    await tx.update(purchaseOrders).set({ status: 'closed', closedAt: new Date() });
    // 3. Trigger performance record creation (still in transaction)
    await tx.insert(supplierPerformanceRecords).values({ ... });
  }
  // If any step throws, entire transaction rolls back
});
```

### Additional Consistency Measures
- **Optimistic locking**: `version` column on PO and Invoice; UPDATE checks `version = :expected` before writing
- **Duplicate prevention**: UNIQUE constraints on invoice numbers
- **Overpayment prevention**: DB CHECK constraint + application validation

---

## ADR-008: Background Job Processing with BullMQ

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
Several SupplyMesh operations should not block HTTP responses: email notifications, approval escalation checks, supplier performance recalculation, document expiry checks.

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **BullMQ + Redis (Selected)** | Reliable, retryable jobs, priority queues, cron support, excellent Node.js ecosystem | Requires Redis |
| pg-boss (PostgreSQL-backed queue) | No extra infrastructure | Slower, adds write load to primary DB, less feature-rich |
| AWS SQS | Fully managed, reliable | External dependency, adds latency, more complex local dev setup |
| Synchronous execution | Simple | Blocks HTTP responses; a failed email notification would fail the API request |
| Agenda (MongoDB-backed) | Popular | Requires MongoDB (we're not using it) |

### Decision
**BullMQ with Redis**

### Queue Design

```
Queues:
├── approval-escalation    (cron: every 30 min)
├── notification-email     (event-driven, concurrency: 5)
├── performance-scoring    (event-driven, after PO close)
├── document-expiry-check  (cron: daily 06:00 UTC)
└── audit-write            (event-driven, concurrency: 10, priority: low)
```

### Retry Strategy
- Notification emails: 3 retries with exponential backoff (1s, 5s, 30s)
- Performance scoring: 2 retries, then dead-letter queue
- Approval escalation: No retry (it's a cron job, next run will catch it)

---

## ADR-009: NestJS over Plain Express for Backend

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
Which Node.js framework should power the SupplyMesh backend?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **NestJS (Selected)** | Dependency injection, module system, decorators, built-in validation, OpenAPI generation, Passport.js integration, testing utilities | Learning curve for developers unfamiliar with Angular-style patterns, more boilerplate than Express |
| Express.js | Minimal, flexible, familiar | No built-in structure; a team of developers will create inconsistent patterns; no DI container |
| Fastify | Faster than Express, TypeScript-friendly | Less ecosystem maturity than NestJS for enterprise patterns |
| Hono | Extremely fast, edge-compatible | Very new, small ecosystem |

### Decision
**NestJS**

### Reasoning
SupplyMesh's backend has 10+ domain modules, each with controllers, services, guards, pipes, and DTOs. Without a structured framework, different developers will organize code differently, leading to an unmaintainable codebase. NestJS's module system enforces domain boundaries that align with our modular monolith architecture decision.

NestJS's dependency injection container makes services testable in isolation — critical for a financial application where unit test coverage of business logic (approval routing, invoice matching, payment validation) is non-negotiable.

---

## ADR-010: JWT Authentication with Short-Lived Tokens + Redis Refresh Tokens

- **Date**: 2026-11-16
- **Status**: ✅ Accepted

### Problem
How should SupplyMesh handle user authentication in a stateless, multi-tenant environment?

### Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **JWT + Redis Refresh Tokens (Selected)** | Stateless access tokens (no DB hit on every request), refresh tokens stored in Redis (revocable), short expiry reduces risk | Requires Redis; refresh token rotation adds complexity |
| Session-based (server-side sessions) | Easy token revocation | Stateful; requires sticky sessions or session store for horizontal scaling |
| JWT only (long-lived, no refresh) | Simplest | Cannot revoke tokens (security risk for financial application) |
| OAuth2 with external provider | Enterprise SSO support | External dependency; complex for self-registration flow |

### Decision
**JWT access tokens (15-minute expiry) + Refresh tokens stored in Redis (7-day expiry)**

### Token Structure
```
Access Token (15 min): Contains userId, orgId, roles, permissions
Refresh Token (7 days): Opaque random string stored in Redis with userId mapping
```

### Security Measures
- Access tokens: 15-minute expiry minimizes damage from token theft
- Refresh token rotation: Every refresh issues a new refresh token and invalidates the old one
- Refresh tokens stored in Redis with `orgId:userId:tokenHash` key → enables single-click "logout all devices"
- `httpOnly` + `SameSite=Strict` cookies prevent XSS token theft
- Role changes take effect within 15 minutes (next access token refresh)

### Why Not Sessions?
SupplyMesh will run multiple API instances behind a load balancer. Session-based auth would require sticky sessions or a shared session store (which is essentially what we're doing with Redis, but JWT is more flexible and cacheable).

---

## Decision Summary Table

| ADR | Decision | Key Reason |
|-----|----------|-----------|
| 001 | PostgreSQL over MongoDB | Relational financial data requires ACID + FK constraints |
| 002 | Drizzle ORM over Prisma | SQL-first transparency for complex financial queries |
| 003 | Modular Monolith over Microservices | Financial transactions require DB-level atomicity; domain boundaries not yet stable |
| 004 | Redis for cache + queues | Permission caching + BullMQ job queues |
| 005 | Shared schema multi-tenancy | Cost-effective; enforced by RLS as safety net |
| 006 | JSONB for workflow rules | Flexible approval policies without schema migrations |
| 007 | PostgreSQL transactions for PO/Invoice/Payment | Native ACID within single DB instance |
| 008 | BullMQ for background jobs | Reliable, retryable, non-blocking async processing |
| 009 | NestJS over Express | Enforced module structure; DI for testability |
| 010 | JWT + Redis refresh tokens | Stateless + revocable; secure for financial SaaS |
