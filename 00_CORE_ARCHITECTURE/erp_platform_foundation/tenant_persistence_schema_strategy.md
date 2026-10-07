# Tenant Persistence & Schema Strategy Assessment

Status: **RECOMMENDED — OWNER DECISION REQUIRED**.

## Schema alternatives

| Dimension | A: common full ERP schema per tenant | B: subscribed-module schemas | C: governed profiles/module migration sets |
| :--- | :--- | :--- | :--- |
| Advantages | Uniform versions/support; simple activation; smaller test matrix; consistent recovery | Fewer unused objects | Bounded variants may reduce fleet overhead while remaining governed |
| Disadvantages | Catalog/migration/backup overhead grows with tenants even when tables are empty | High divergence; activation migrations; dependency/rollback complexity | Profile design/version combinations and dependency ordering remain complex |
| Failure modes | Partial rollout/drift; long fleet migration | Missing dependencies, failed activation, irreproducible tenant shapes | Profile explosion, incompatible module versions |
| Migration/support | One package, independent per-DB history and staged rollout | Many conditional histories/runbooks | Multiple governed sets and compatibility matrices |
| Testing | Fresh/upgrade plus representative tenant data | Combinatorial subscription matrix | Each profile and transition path |
| Recommendation | **Initial recommendation** | Not recommended initially | Future ADR only after evidence |

### Empty-table concern

An unused PostgreSQL table is not equivalent to a populated table in data-storage cost, so empty capability tables are not automatically a design failure. The real fleet concern is multiplied catalog objects, indexes, migration work, backups, monitoring, connection/resource management and operational support across many databases. Conditional schemas avoid some unused objects but create schema divergence, subscription-activation migrations, dependency ordering, recovery variants and a much larger test/support matrix. No numerical cost claim is made here.

Future review should measure tenant count, table/index count per tenant database, unused-capability percentage, migration duration, catalog overhead, backup/restore duration, connection/resource overhead, fleet-management cost and support/testing complexity. **RECOMMENDED — OWNER DECISION REQUIRED:** begin with common governed schema and consider profiles/module sets only through a new ADR when evidence shows material inefficiency.

## Persistence topology

| Option | Assessment |
| :--- | :--- |
| P1 one operational ERP database per organization with multiple strictly owned contexts | Lowest database-count burden; logical ownership and no foreign owner access remain mandatory. |
| P2 per-domain database per organization | Strong physical separation but database-count, migration, backup, connection and support explosion. |
| P3 primary tenant ERP database plus justified specialized persistence | **RECOMMENDED — OWNER DECISION REQUIRED:** balances fleet operations with workload-specific stores. Tracking is factual evidence for Kafka/Redis/Timescale specialization, not a template for every domain. |

## Migration/release strategy

**RECOMMENDED — OWNER DECISION REQUIRED:** one approved release/migration package for the chosen schema line, executed independently per tenant database through validation, pilot/rings, pause/recovery and per-database verification/history. “Global migration” never means one distributed transaction, simultaneous completion, or hiding one tenant failure behind a global success flag. Modules retain migration semantics; Platform Management orchestrates truthful state. Unused capability tables may remain empty.

Schema presence does not grant access: industry classification, capability entitlement, feature activation, user permission and resource scope are separate controls.
