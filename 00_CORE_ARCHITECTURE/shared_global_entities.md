# Shared Global Entities & Primitives

Status: Controlled vocabulary; not a shared business-model library

All business modules must reference these canonical structures for cross-cutting concepts.

## 1. Tenant Context

- **Representation:** `tenant_id` (UUID)
- **Runtime authority:** `CurrentTenant` / `TenantExecutionContext`, established after authenticated identity is validated against an active server-side Tenant membership and active Tenant.
- **Rule:** JWT claims, payloads, query parameters, browser storage, and arbitrary Tenant headers are not sole Tenant authority. Business modules consume the approved context and never query Tenant-owned data without Tenant scope.
- **Background rule:** Async, scheduled, and background work receives Tenant identity explicitly and establishes/clears a bounded context; it never blindly inherits an HTTP `ThreadLocal`.

## 2. Audit Envelope (Base Entity Model)

All tenant-owned database tables must include these standard audit columns:

| Column Name | Type | Nullable | Default | Description |
| :----------- | :---------- | :------- | :------------------ | :----------------------------------------------------- |
| `id` | UUID | NO | gen_random_uuid() | Primary Key |
| `tenant_id` | UUID | NO | - | Multi-Tenant Identifier (Indexed) |
| `created_at` | TIMESTAMPTZ | NO | NOW() | Timestamp when created |
| `updated_at` | TIMESTAMPTZ | NO | NOW() | Timestamp when last updated |
| `created_by` | UUID | YES | NULL | User ID who created the record |
| `updated_by` | UUID | YES | NULL | User ID who last updated the record |
| `deleted_at` | TIMESTAMPTZ | YES | NULL | Soft delete timestamp (Index: `tenant_id`, `deleted_at`) |

## Classification

| Concept | Canonical representation | Ownership | Notes |
| :--- | :--- | :--- | :--- |
| Tenant | `tenant_id: UUID` | Platform Tenancy | Root isolation boundary; immutable |
| Actor | `actor_id: UUID`, `actor_type` | Identity | Human or service principal |
| Correlation | `correlation_id: UUID/string` | Technical | Propagated across calls and events |
| Event identity | `event_id: UUID`, `event_version: integer` | Producer | Globally unique event occurrence |
| Money | decimal amount + ISO-4217 currency | Owning module | Never binary floating point |
| Time | UTC instant / `TIMESTAMPTZ` | Owning module | Local date/time requires explicit zone |
| Country | ISO-3166 code | Reference data | Do not duplicate country names as keys |
| Language | BCP-47 tag | Reference data | Used for locale/template selection |
| Measurement | decimal value + unit code | Owning module | Unit semantics belong to the domain |

## Tenant Entity

Minimum platform contract: `id`, `code`, `displayName`, `status`, `defaultCurrency`, `defaultTimeZone`, `createdAt`, `updatedAt`, and `version`. No business module may own or mutate tenant lifecycle directly.

Transportation persists this contract via `V43__tenant_foundation.sql`. Its canonical clean-initialization record is UUID `4f8b6a3b-2c1e-4d89-9a72-f9e4c5b3671a`, code `CLTS-LK`, Ceylon Logistics & Transport Solutions (Pvt) Ltd, `LKR`, `Asia/Colombo`, `ACTIVE`.

## Audit Metadata

The audit envelope above is mandatory for tenant-owned tables. Add a `version` column when optimistic concurrency control is required. Audit actor IDs are logical Identity references and must not create physical cross-module foreign keys.

## Identity References

Modules store identity IDs as UUID primitives and resolve display data through contracts. They must not copy authentication secrets or join identity tables. Employee identity and login identity are distinct concepts and may be linked by HRM through a documented logical reference.

## Durable Internal Event Boundary (P1-01)

The shared technical boundary owns the single `integration_outbox_event` table and exposes provider-neutral `DurableEventPublisher`, `DurableEventHandler`, and worker contracts. Business modules never import the outbox JPA entity or repository. The current approved durable family is Delivery's five-type version-1 US-69 customer-notification contract; Notification's existing execution key is its consumer dedupe/inbox boundary.

Durable records carry stable event identity, trusted Tenant identity, event type/version, aggregate type/ID, minimized JSON payload, occurrence time, bounded claim/retry state, and sanitized terminal error codes. Background processing establishes and clears Tenant context explicitly. Delivery is at-least-once without global ordering; no broker or exactly-once claim is approved.

## Rules

- Reuse syntax and semantics, not mutable shared domain entities.
- A module remains authoritative for its identifiers and lifecycle.
- Cross-module references are logical unless both tables belong to the same bounded context.
- Additions require an owner, definition, compatibility policy, and registry update.
