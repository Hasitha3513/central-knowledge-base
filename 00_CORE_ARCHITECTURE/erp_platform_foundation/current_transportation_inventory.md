# Current Transportation Architecture Inventory

Status: **CURRENT VERIFIED IMPLEMENTATION** unless explicitly labelled otherwise. Source evidence is the current KB, especially `../../02_MODULES_KNOWLEDGE/transportation_and_logistics.md`, `tracking.md`, `transport_billing.md`, `integration_module.md`, `system.md`, `compliance.md`, and the canonical registries.

## Existing bounded contexts and ownership

| Context | Current authority / owned persistence | Enterprise overlap |
| :--- | :--- | :--- |
| Identity/Tenancy | Customer accounts, tenant membership, roles/permissions, authentication/JWT and actor authority in current Transportation runtime | Future Platform Identity and Platform Management trust handoff; one-account-one-organization target |
| Organization/Master Data | Customer/contact, internal organizational and reference facts used by operational modules | Future Sales/CRM Customer authority; subscribing Organization remains Platform concept |
| Fleet | Vehicle identity/master, category/type, documents, readings, meter resets, lubricant log and maintenance_schedule | Future Vehicle Maintenance work lifecycle; Fleet remains likely Vehicle master |
| Driver/Fleet | Driver operational profile, licence, exception, violation, medical/drug-test and availability/eligibility records | Future HRM Employee/Employment/general qualification boundary |
| Routing | Route, Stop, Revision and Disruption lifecycle | Stable Transport owner; Tracking consumes route facts |
| Trip | Trip order, assignments, dispatch, lifecycle, history and operational events | Finance/Billing consume completed facts; Projects/Customer references need reconciliation |
| Fuel | Station, limit policy, issue/purchase/price, bunker Tank/Movement, cards, reconciliation and performance | Procurement/Supplier, Inventory and Finance boundaries require explicit exceptions/contracts |
| Freight | Order, manifest, load plan, insurance and cargo exceptions | Sales Order/Customer and Finance interactions |
| Delivery | Order, POD, attempts, redelivery, exception, zone, slot, Rider, assignment, batch, ETA and customer self-service | Sales fulfillment/Customer relationship and Finance consequence boundaries |
| Notification | Rule/policy/template, notification, channel delivery attempt, preferences and retry evidence | Specialized service candidate; source domains retain meaning/recipient authority |
| Offline Sync | Idempotent command inbox and conflict outcome | Remains technical/operational boundary; no enterprise master ownership |
| Reporting/Analytics | Domain-defined operational projections and Reporting composition | Future enterprise read models must preserve metric ownership and lineage |
| Tracking | Device/provider registry, association, telemetry ingestion/history/live state, detector and audit state | Specialized workload boundary; Fleet/Trip/Route references stay logical |
| Integration | Configuration, declarative mapping, exchange/attempt/audit and controlled external delivery | Candidate gateway deployment; owns connectivity, never business meaning |
| Transport Billing | Operational transport charge composition/finalization/reversal/export | Finance owns formal accounting consequences under accepted boundary |
| Operations | Cross-domain detected exception case lifecycle | Producers retain source meaning/evidence/correction |
| Compliance | Default-off policy/evaluation/evidence capability | Qualified policy authority still pending; consumes minimized owner facts |
| System | Resilience/degraded-mode and data-integrity finding/correction orchestration | Not Platform Management and not a generic business owner |

## Current persistence and dependencies

- PostgreSQL/Flyway is the normal owner persistence. Tenant-owned rows and queries are tenant-qualified under current standards.
- The current aggregate catalogue treats Vehicle, Driver records, Route, Trip, Fuel Issue and focused Delivery roots as separate consistency boundaries. Audit/history/evidence stores are not used as cross-module aggregate graphs.
- The KB documents logical ID/value references and a remediated empty production foreign-SQL baseline. Published module-root ports are the approved synchronous seams; registered local/durable events are the asynchronous seams.
- Identity/Tenancy, Organization Customer, Vehicle, Driver, Project, Route, Trip and other current references must not be reassigned based only on similar future module names.

## Specialized infrastructure

- Tracking uses Kafka as the durable high-rate telemetry backbone, Redis for disposable tenant-qualified live projections, TimescaleDB for append-only normalized history, and PostgreSQL for configuration, nonce/dedupe authority, audit and detector state.
- This is evidence that workload-specific persistence can be justified. It is not approval to extract every Transportation context or to give generic ERP domains their own services/databases.
- Integration uses the accepted shared durable-event boundary only for registered families; no generic event export or second outbox is inferred.

## Existing investment statement

The Transportation application is not disposable prototype code. It contains accepted business behavior, tenant/security controls, migrations, compatibility contracts and test evidence. **RECOMMENDED — OWNER DECISION REQUIRED:** preserve it as an independently deployable domain application, reconcile overlapping masters through staged contracts/mappings, and require evidence before any merge, rewrite or further physical decomposition.
