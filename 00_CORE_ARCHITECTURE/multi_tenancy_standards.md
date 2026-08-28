# Multi-Tenancy Standards

Status: Mandatory for all new schema and contracts

Isolation model: Shared application and database with row-level tenant discrimination unless an approved ADR specifies stronger isolation

## Tenant Identity

- Canonical identifier: `tenant_id`, UUID, immutable.
- Every tenant-owned aggregate, row, event, command, query, cache key, audit record, and idempotency key includes tenant identity.
- Global data is exceptional and explicitly classified `GLOBAL`; absence of `tenant_id` never implies global status.
- Tenant scope comes from authenticated identity plus validated active server-side membership, or an approved trusted-service policy. Client input and unvalidated token claims are never Tenant authority.

## Request Flow and Runtime Authority

The Transportation MVP uses `SERVER_SIDE_MEMBERSHIP_RESOLUTION`:

1. Verify the bearer token and load the active authenticated user.
2. Resolve the user's current `ACTIVE` `TenantMembership` from server persistence.
3. Resolve and verify the associated `ACTIVE` Tenant.
4. Establish `TenantExecutionContext`; business code consumes `CurrentTenant`, not Spring Security, JWT, or HTTP details.
5. Pass tenant identity explicitly to tenant-owned use cases and persistence operations; every query predicates on it.
6. Clear HTTP context in `finally`.

`CurrentTenant` / `TenantExecutionContext` is authoritative at runtime. Payload/query `tenantId`, browser storage, and arbitrary `X-Tenant-ID` are `UNTRUSTED`. The single-membership MVP has no tenant switcher. A future selector may select only a currently authorized membership.

HTTP `ThreadLocal` context is request-scoped only. Scheduled, background, and asynchronous work receives Tenant identity explicitly and establishes/clears a bounded context; it never blindly inherits request context.

## JWT Tenant Authority

- JWT is authentication evidence, not authoritative Tenant ownership.
- A mandatory immutable `tenant_id` JWT claim is `NOT REQUIRED` for the current MVP. Existing access-token and refresh-token contracts remain unchanged.
- Login, refresh, and every bearer request revalidate active membership and active Tenant server-side, so changes take effect without waiting for old access-token expiry.
- A future Tenant claim may be an optimization or selection hint only when validated against current server membership; it is never sole authority.

## Authorization and RBAC

`ACCESS = Authenticated User AND Active Tenant Membership AND Active Tenant AND Required RBAC Permission AND Tenant-Owned Resource Scope`

Every term is required. Admin-style roles do not bypass isolation; interactive cross-tenant administration and impersonation are deferred.

### Global RBAC definitions

- `app_permission`: `GLOBAL` permission catalogue.
- `app_role`: `GLOBAL` role-template catalogue.
- `app_role_permission`: `GLOBAL` template-to-permission mapping.

These define capabilities, not Tenant ownership. Username and email remain globally unique credentials. Tenant-custom roles are deferred.

### Tenant-scoped authorization assignment

Target: `TenantMembership -> Membership Role Assignment -> Global Role Template -> Global Permissions`.

Current `app_user_role` is `LEGACY_UNSCOPED_ROLE_ASSIGNMENT` and `TRANSITIONAL`. It may temporarily preserve existing RBAC behavior but is not final Tenant-safe authority. `TENANT-OPERATIONAL-DATA-RETROFIT-001` must define a forward migration, compatibility window, and deprecation/removal path toward a repository-convention name such as `membership_role` or `tenant_membership_role`; this document does not authorize that schema change.

The database currently enforces one `tenant_membership` record per user. Membership-scoped assignment remains required so future multi-membership can grant different global role templates in different Tenants.

## Database Controls

- Tenant-owned tables define `tenant_id UUID NOT NULL` and tenant-leading indexes.
- Business uniqueness is tenant-scoped, for example `UNIQUE (tenant_id, code)`.
- Same-module relationships preserve Tenant consistency where practical.
- Unscoped tenant-owned repository lookups are forbidden; use Tenant-qualified methods/specifications.
- Bulk mutations, native SQL, jobs, exports, and reports require explicit Tenant predicates.
- New writes derive ownership from `CurrentTenant`, never client input.
- PostgreSQL RLS is defense in depth, not a substitute for application isolation.

## API, Events, and Operations

- Public `tenantId` values are accepted only under an explicit trust model and must match authenticated context.
- Event envelopes include Tenant, event, version, time, producer, aggregate, correlation, and optional causation identity.
- Consumers partition idempotency by `(tenantId, eventId)`.
- Logs, traces, storage, caches, and search indexes are Tenant-scoped.

## Testing

Tenant-aware features prove same-Tenant access, cross-Tenant denial for every operation, Tenant-scoped uniqueness, Tenant-bearing events, and isolated background/bulk paths.

## Transportation Foundation Status

Decision: `ADR-TENANT-AUTHORITY-AND-RBAC-001.md`

Verified implementation: `TENANT-FOUNDATION-IMPLEMENTATION-001` (2026-08-28)

| Capability | Status |
| :--- | :--- |
| First-class Tenant aggregate and persistence | `IMPLEMENTED` |
| Explicit Tenant membership and persistence | `IMPLEMENTED` |
| `CurrentTenant` / `TenantExecutionContext` | `IMPLEMENTED` |
| Per-request server-side membership/Tenant validation | `IMPLEMENTED` |
| Canonical `CLTS-LK` clean bootstrap | `IMPLEMENTED` |
| Operational business-row scoping | `PENDING` |
| Repository/event/job/cache/report isolation | `PENDING` |
| Overall Tenant isolation | `PARTIAL` |

Platform `tenancy` owns Tenant lifecycle. Identity owns membership and authenticated resolution; Organization and `shared` do not own Tenant lifecycle. The MVP permits exactly one membership per user; multi-membership is deferred.

### Canonical clean initialization

- UUID: `4f8b6a3b-2c1e-4d89-9a72-f9e4c5b3671a`
- Code: `CLTS-LK`
- Name: Ceylon Logistics & Transport Solutions (Pvt) Ltd
- Currency/time zone/status: `LKR` / `Asia/Colombo` / `ACTIVE`
- Migration: `V43__tenant_foundation.sql`

No recoverable legacy production database was found. Legacy preservation is `NOT REQUIRED`; legacy reconciliation and backfill are `NOT APPLICABLE`; this is a `CLEAN_INITIALIZATION_TARGET`. Historical migrations remain immutable, and legacy mapping/backfill gates must not be reintroduced for this environment.

### Remaining operational gap

Foundation implementation is not complete multi-tenancy. Existing operational tables remain predominantly without `tenant_id`; repositories, events, jobs, caches, offline operations, and reporting sources are not fully isolated. `TENANT-OPERATIONAL-DATA-RETROFIT-001` is next and must scope data module by module through forward migrations and isolation tests.

US-29 Freight Reporting remains `BLOCKED_BY_TENANT_FOUNDATION` until Freight and Reporting sources are tenant-scoped and operational isolation acceptance passes.
