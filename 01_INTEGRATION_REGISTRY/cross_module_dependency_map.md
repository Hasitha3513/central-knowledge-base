# Cross-Module Dependency Map

## Directional Integration Map

| Consumer | Provider | Needed information | Approved mechanism | Status |
| :--- | :--- | :--- | :--- | :--- |
| Finance | Transportation | completed trips, freight charges, fuel/maintenance costs | Events; query API for reconciliation | PROPOSED |
| Transportation | Finance | payment/credit approval, cost posting result | Events; focused API | PROPOSED |
| Transportation | HRM | driver identity, qualification, availability | API/port plus change events | PROPOSED; current driver data is local legacy |
| Maintenance | Transportation | vehicle identity, meter readings, availability | Events and vehicle query API | PROPOSED |
| Transportation | Maintenance | maintenance hold/release and forecast | Events and availability API | PROPOSED; current schedules are local legacy |
| Maintenance | Inventory | parts availability, reservations, issues | API/port and stock events | PROPOSED |
| Procurement | Inventory | replenishment demand and receipt | Events/API | PROPOSED |
| Procurement | Finance | budget and payment controls | API/events | PROPOSED |
| Projects | HRM | staffing and time facts | Events/API | PROPOSED |
| Projects | Finance | budgets and actual costs | Events/API | PROPOSED |
| Sales/CRM | Inventory | available-to-promise | API | PROPOSED |
| Sales/CRM | Transportation | shipment plan and delivery status | Commands/API and events | PROPOSED |
| Delivery (Transportation internal) | Freight, Trip, Organization, Notification, Offline Sync | Organization-owned customer/location/contact references; Delivery-owned evidence, failed-attempt, redelivery, exception, zone, slot, Rider, batch, ETA, Planner, US-69 customer-notification facts, and US-70 token-scoped self-service projection/submissions; Notification-owned Email/SMS preferences; Offline Sync for POD | Public provider-neutral ports, canonical local event envelopes, the shared `DurableEventPublisher` technical port for US-69 facts, and `OfflineOperationHandler`; no direct repositories/JPA/tables | US56–US70_ACCEPTED; P1-01_HARDENED |
| Notification | Delivery, Organization | US-69 committed Delivery facts, tenant-scoped active Customer display/contact projection, and a transient US-70 self-service link immediately before provider delivery | Delivery version-1 events through the V60 shared durable outbox, Organization public `CustomerNotificationContactLookup`, Delivery public `CustomerSelfServiceLinkIssuer`, and Notification public `FinalSendCustomerLinkIssuer`/`CustomerOperationalPreferenceManagement`; no raw token persistence and no source repository/entity/table access | US69_ACCEPTED; US70_ACCEPTED; P1-01_AT_LEAST_ONCE |
| Integration | Shared P1-01 durable event boundary; future approved domain producers | Canonical Tenant/version/aggregate event envelope for governed external exchange | `DurableEventPublisher` and `integration_outbox_event`, followed by Integration-owned idempotent exchange/attempt state and provider-neutral adapter port; no foreign repository/table | US73_PLATFORM_PROBE_ACCEPTED; all business families require separate approval |
| Operations | Routing and Delivery first; future Trip, Freight, Driver/Fleet, Fuel, Tracking, Compliance and Integration producers | Minimized detected exception fact with trusted Tenant, stable source event, logical source reference, candidates, summary code and safe metadata | `OperationalExceptionFactV1` through P1-01 shared `DurableEventPublisher`; no foreign repository/entity/table/API scraping | COMPLETE_US78; active producers ROUTING + DELIVERY |
| Notification | Operations | Safe escalation fact only | `OPERATIONAL_EXCEPTION_ESCALATED_V1` through P1-01; Notification owns rule/channel/template/recipient/retry | COMPLETE_US78 |
| Operations | Identity, US-80, US-81, US-83 | Validate same-Tenant user/role queue, apply fixed workflow/scheduling boundaries, and reference governed documents | Focused published ports/logical IDs only; no foreign persistence | COMPLETE_US78 |
| Fuel Performance | Fleet, Trip and Tenancy | Bulk Tenant-scoped compatible Vehicle/Driver context, authoritative Driver/Trip attribution, and trusted Tenant/timezone/currency context | Published provider-neutral root query contracts only; no foreign repository/entity/table/SQL and no N+1 calls | COMPLETE_US37_FINAL_ACCEPTANCE_PASS |
| Reporting | Fuel Performance | Fuel-owned summary, Vehicle/Driver comparison and trend projection with quality/lineage | Fuel-root `FuelPerformanceQuery`; Reporting does not redefine metrics or access Fuel persistence | COMPLETE_US37_FINAL_ACCEPTANCE_PASS |

## Ownership Decisions

- Transportation owns trips, routes, freight execution, operational vehicle readings, and transport-specific fuel facts.
- Transportation contains a dedicated Delivery Modulith boundary. US-56 and US-57 are implemented and accepted; US-58 offline POD product decisions are frozen; US-59 through US-62 remain pending.
- HRM will own employee master and employment lifecycle; migration/bridging of the current `driver` model requires an ADR.
- Vehicle Maintenance will own maintenance work execution; migration/bridging of current `maintenance_schedule` requires an ADR.
- Finance owns ledgers, invoices, payments, tax accounting, and financial posting—not operational source facts.
- Inventory owns stock balances and movements; Procurement owns purchasing intent and purchase-order lifecycle.
- Sales/CRM owns prospects, customer relationships, quotations, and sales orders. Customer-master ownership requires an ADR because transportation currently owns a `customer` table/API.
- For current US-69 scope, Organization's existing Customer model is the authoritative contact source. Notification owns only channel preferences and accepted destination snapshots; this does not settle future suite-wide Sales/CRM customer-master ownership.
- US-70 does not create a Customer/Recipient-to-Identity relationship. Delivery owns opaque per-Delivery access tokens and customer submissions; Organization remains Customer/contact authority and Notification remains Email/SMS preference/delivery authority. Customer requests do not bypass US-60 scheduling or US-64 slot capacity.
- The shared technical boundary solely owns `integration_outbox_event`. Business modules publish through `DurableEventPublisher` and never import its JPA repository. P1-01 carries the accepted Delivery-to-Notification US-69 family and the accepted Integration-owned `US73_PLATFORM_PROBE_V1`; all other event-family classifications remain local or unused until a real consumer is approved.
- US-73 ratifies `integration` as a dedicated top-level bounded context for connectivity, declarative mapping, configuration, health, exchange attempts, external retry, and audit. It owns no domain business meaning and may consume only explicitly registered minimized facts. Its first approved capability is outbound `FILE_JSON_V1` under `CONTROLLED_SANDBOX`; every named vendor ecosystem remains future scope.
- US-78 implements and accepts `operations` as a dedicated top-level hybrid exception-lifecycle context. Operations owns triage, assignment, SLA, escalation, corrective action, RCA, resolution/closure/reopen and append-only case history only after a producer detects a domain exception. Producers retain source state/meaning/evidence/correction. Routing and Delivery are active producers; US-68 remains read-only and is not duplicated.
- US-37 implements Fuel-owned deterministic, read-only historical Fuel performance interpretation. It uses issued Fuel facts plus published bulk Fleet/Trip projections and trusted Tenancy context, never foreign SQL. Reporting is presentation/composition only. No raw mutation, persisted projection, cache, event, P1-01 use, export, ML, punitive Driver ranking, or automatic US-38/US-78 action exists.
- Project Management owns projects, work structures, milestones, and project budgets. Current transportation `project` references are legacy and require ownership reconciliation.

## Forbidden Edges

No module may use another module's SQL, JPA repository, entity, table-level join, or physical foreign key. IDs crossing boundaries are UUID logical references. A read model may combine events in its own schema but must not query producer tables.

### P0-03 remediated persistence violations

| Consumer | Owner | Table | Current access | Required remediation | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Freight reporting adapter | Fleet | `vehicle` | Published synchronous `FleetReportingQuery.findVehicle(UUID)` | Fleet-owned capacity facts; tenant isolation remains enforced by the provider | `REMEDIATED_P0-03` |

The production foreign-SQL baseline is empty. The approved Java dependency graph permits only listed consumer/provider edges, and all such dependencies must target contracts published in the provider module root package.

## Integration Review Checklist

Confirm ownership, tenant propagation, data classification, contract version, source of truth, consistency expectation, idempotency, ordering, retry/dead-letter behavior, authorization, observability, retention, and producer/consumer contract tests.

## Full-Product 87-Story Planning Boundary

Following US-37 final acceptance, 68 / 87 stories are accepted and exactly 19 remain: US-35, US-38, US-46, US-47, US-48 through US-55, US-72, US-76, US-82, and US-84 through US-87. They remain planned in four open waves after Wave A closure:

1. US-73 external integrations and US-78 operational-exception lifecycle;
2. US-37/35/38 Fuel plus US-46 payroll link and US-47 transport billing;
3. US-48 through US-55 provider-neutral GPS/tracking;
4. US-72 compliance and US-76 mobile operations;
5. US-85/84/87/82/86 integrity, resilience, user risk, analytics, and disruption.

Expected future owners remain planning hypotheses except that US-73 has established and accepted the dedicated `integration` owner as `COMPLETE_US73` and US-78 has established and accepted the dedicated `operations` owner as `COMPLETE_US78`. Existing `fuel`, `driver`, `offlinesync`, `reporting`, `identity`, and `system` boundaries retain their data. Distinct `tracking`, `compliance`, and `billing` contexts remain justified candidates requiring their own story gate. Finance continues to own ledger/payment/tax posting and HRM/payroll continues to own final salary processing.

US-73 final acceptance covers exactly one governed outbound JSON-file adapter with controlled-sandbox evidence; it does not imply that any named business ecosystem is implemented. US-78 implements the first additional durable families by reusing P1-01; no second outbox, broker, exactly-once claim, foreign repository, or cross-module SQL exists. US-78 final acceptance passes, Wave A is 2/2 closed, and US-37 final acceptance passes with Wave B open. The queue head is `US-35-FUEL-CARDS-PRODUCT-DECISIONS-001`.
