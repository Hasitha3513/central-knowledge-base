# Multi-Tenancy Standards

Status: Mandatory for all new schema and contracts
Isolation model: Shared application and database with row-level tenant discrimination unless an approved module ADR specifies stronger isolation

## Tenant Identity

- Canonical identifier: `tenant_id`, UUID, immutable.
- Every tenant-owned aggregate, persistence row, integration event, command, query, cache key, audit record, and idempotency key includes tenant identity.
- Global reference data is exceptional and must be explicitly classified as `GLOBAL`; absence of `tenant_id` is never an implicit global classification.
- Tenant identity is derived from a verified token/session or trusted service credential, never from an untrusted payload alone.

## Request Flow

1. The inbound adapter authenticates the caller.
2. It resolves the active tenant and verifies membership/authorization.
3. It constructs a typed tenant context containing `tenantId`, actor, correlation ID, and request ID.
4. The inbound port receives the tenant context explicitly.
5. Every outbound persistence call carries `tenantId` and every query predicates on it.

## Database Controls

- New tenant-owned tables define `tenant_id UUID NOT NULL` and an index beginning with `tenant_id` for tenant-scoped access paths.
- Business uniqueness is tenant-scoped, for example `UNIQUE (tenant_id, code)` rather than global `UNIQUE (code)`.
- Foreign keys within one module include tenant consistency where practical, for example composite references `(tenant_id, parent_id)`.
- Repository methods such as `findById(UUID)` are forbidden for tenant-owned records; use `findByTenantIdAndId(tenantId, id)` or an equivalent specification.
- Bulk updates, deletes, native SQL, scheduled jobs, exports, and reports require explicit tenant predicates.
- PostgreSQL row-level security is recommended as defense in depth, not a substitute for application isolation.

## API, Events, and Operations

- Public payloads include `tenantId` only where the trust model permits; the server rejects mismatches with authenticated context.
- Event envelopes always include `tenantId`, `eventId`, `eventType`, `eventVersion`, `occurredAt`, `producer`, `aggregateType`, `aggregateId`, `correlationId`, and optional `causationId`.
- Consumers partition idempotency by `(tenantId, eventId)`.
- Logs, metrics, traces, object-storage paths, caches, and search indexes are tenant-scoped and must avoid sensitive payloads.
- Support impersonation must be explicit, authorized, audited, time-bounded, and visible in the tenant context.

## Testing

Every tenant-aware feature requires tests proving same-tenant access succeeds, cross-tenant read/write/update/delete fails, unique constraints are tenant-scoped, events carry the tenant, and background/bulk paths preserve isolation.

## Current Transportation Gap

The existing transportation migrations V1–V41 predate this standard and do not contain the required tenant foundation. They must be treated as legacy single-tenant schema. Remediation requires a forward-only migration plan with certified ownership mapping, backfill, reconciliation, constraints, indexes, repository changes, API/event compatibility review, and isolation tests. Historical migrations must never be edited.

## Transportation MVP Decision Freeze

Decision source: `TENANT-BUSINESS-AUTHORITY-001` (2026-08-27)

### DECIDED

- A dedicated platform `tenancy` bounded context owns the first-class Tenant aggregate. Identity owns membership and resolution; Organization and `shared` do not own Tenant lifecycle.
- Tenant identity consists of immutable UUID, code, name, status, default currency, default time zone, timestamps, and version.
- The MVP permits exactly one active tenant membership per active user. Membership is explicit; multi-membership and tenant switching are deferred.
- Username and email remain globally unique. Permissions and role definitions are global platform templates; membership/role assignments are tenant-scoped. Tenant-custom roles are deferred.
- No interactive cross-tenant global administrator or tenant impersonation is part of the MVP. Admin-style roles never bypass isolation.
- Authentication resolves active membership and Tenant before issuing a JWT containing immutable `tenant_id`. Tenant-owned inbound ports receive a typed context containing at least tenant, actor, and correlation identity.
- Every tenant-owned persisted row directly stores `tenant_id`, including child, history, execution, delivery, document, offline, and reporting/export records.
- System/default notification templates are global. Notification rules, executions, and deliveries are tenant-owned; tenant-custom template copies are deferred.
- Tenant-owned identifiers target `(tenant_id, business_identifier)` uniqueness. Existing global uniqueness may remain temporarily until collision and API impact are proven.
- Cross-tenant normal resource access uses 404; authentication remains 401 and in-tenant permission denial follows existing 403 behavior.
- Schedulers explicitly enumerate active tenants and isolate work. Offline idempotency is at least `(tenant_id, operation_id)`. Tenant events carry tenant identity. Reporting predicates apply before all filtering, aggregation, pagination, totals, and export generation.
- Migration is forward-only: expand, migrate, reconcile, contract. V1–V41 remain immutable, and the next migration number must be rechecked when implementation begins.

### IMPLEMENTATION PENDING

No Tenant aggregate/table, membership model, tenant-aware JWT/context, tenant columns, scoped repositories, tenant schedulers, tenant events, or tenant report/export isolation exists in the transportation application yet. The decision record is not evidence of runtime compliance.

### LEGACY MAPPING PENDING

No bootstrap tenant or V1–V41 ownership assignment is authorized. An accountable data/business owner must certify the canonical tenant identity, every user membership, deterministic business-row ownership, orphan/test/unknown handling, reconciliation counts, approver, and approval date before backfill. The historical single-tenant deployment is not sufficient ownership evidence.

Discovery task `TENANT-LEGACY-OWNERSHIP-AUTHORITY-001` is **BLOCKED** (2026-08-27). Repository evidence does not identify the canonical legal operator, Tenant code/name/default currency/default time zone, or actual runtime user/business-row inventory. Committed PostgreSQL/H2 datasets and the opt-in administrator are explicitly sample/local assets and must not be treated as ownership evidence. Actual row, orphan, relationship-conflict, identifier-collision, and active-membership counts remain unknown; formal business/data-governance approval is absent. No UUID, mapping, backfill, or migration is authorized.

Follow-up evidence task `TENANT-LEGACY-EVIDENCE-002` is `BLOCKED_RUNTIME_DATABASE_UNAVAILABLE` (2026-08-27). The execution environment could not open a direct PostgreSQL socket, Docker access was denied, and no local PostgreSQL client was available. Runtime Flyway state and reconciliation counts were therefore not fabricated. A canonical-owner approval template exists but remains unsigned; business-owner approval is still required independently of database access.

US-29 Freight Reporting remains `BLOCKED_BY_TENANT_FOUNDATION` until legacy mapping, tenant implementation, and isolation acceptance are complete.
