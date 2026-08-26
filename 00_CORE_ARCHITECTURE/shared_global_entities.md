# Shared Global Entities & Primitives

Status: Controlled vocabulary; not a shared business-model library

All business modules must reference these canonical structures for cross-cutting concepts.

## 1. Tenant Context

- **Representation:** `tenant_id` (UUIDv7 / UUID)
- **Rule:** Resolve tenant context from verified JWT claims or trusted API Gateway headers at the inbound web-adapter layer. Never query tenant-owned database entities without tenant scoping.

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
| Tenant | `tenant_id: UUID` | Platform/Identity | Root isolation boundary; immutable |
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

## Audit Metadata

The audit envelope above is mandatory for tenant-owned tables. Add a `version` column when optimistic concurrency control is required. Audit actor IDs are logical Identity references and must not create physical cross-module foreign keys.

## Identity References

Modules store identity IDs as UUID primitives and resolve display data through contracts. They must not copy authentication secrets or join identity tables. Employee identity and login identity are distinct concepts and may be linked by HRM through a documented logical reference.

## Rules

- Reuse syntax and semantics, not mutable shared domain entities.
- A module remains authoritative for its identifiers and lifecycle.
- Cross-module references are logical unless both tables belong to the same bounded context.
- Additions require an owner, definition, compatibility policy, and registry update.
