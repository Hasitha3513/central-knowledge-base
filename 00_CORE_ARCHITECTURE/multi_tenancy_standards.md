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

The existing transportation migrations V1–V36 largely predate this standard and do not consistently contain `tenant_id`. They must be treated as legacy single-tenant schema. Remediation requires a separately approved, forward-only migration plan with backfill, constraints, indexes, repository changes, API/event compatibility review, and isolation tests. Historical migrations must never be edited.
