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
| Delivery (Transportation internal) | Freight, Trip, Organization, Notification, Offline Sync | Organization-owned customer/location/contact references; Delivery-owned evidence, failed-attempt, redelivery, exception, zone, slot, Rider, batch, ETA, Planner, US-69 customer-notification facts, and frozen US-70 token-scoped self-service projection/submissions; Notification-owned Email/SMS preferences; Offline Sync for POD | Public provider-neutral ports, standard-envelope local events, and OfflineOperationHandler only; no direct repositories/JPA/tables | US56–US69_ACCEPTED; US70_DECISIONS_FROZEN_NOT_IMPLEMENTED |
| Notification | Delivery, Organization | US-69 committed Delivery facts, tenant-scoped active Customer display/contact projection, and a transient US-70 self-service link immediately before provider delivery | Delivery version-1 after-commit events, Organization public `CustomerNotificationContactLookup`, and frozen Delivery public `CustomerSelfServiceLinkIssuer`; no raw token persistence and no source repository/entity/table access | US69_ACCEPTED; US70_LINK_CONTRACT_FROZEN_NOT_IMPLEMENTED |

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
