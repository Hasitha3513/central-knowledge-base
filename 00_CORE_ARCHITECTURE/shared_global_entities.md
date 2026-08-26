# Shared Global Entities and Primitives

Status: Controlled vocabulary; not a shared business-model library

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

Recommended tenant-owned record metadata: `tenant_id`, `created_at`, `created_by`, `updated_at`, `updated_by`, and `version`. Soft deletion is used only when required by business retention rules; it is not a universal default.

## Identity References

Modules store identity IDs as UUID primitives and resolve display data through contracts. They must not copy authentication secrets or join identity tables. Employee identity and login identity are distinct concepts and may be linked by HRM through a documented logical reference.

## Rules

- Reuse syntax and semantics, not mutable shared domain entities.
- A module remains authoritative for its identifiers and lifecycle.
- Cross-module references are logical unless both tables belong to the same bounded context.
- Additions require an owner, definition, compatibility policy, and registry update.
