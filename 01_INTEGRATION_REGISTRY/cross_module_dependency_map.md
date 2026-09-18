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
| Operations | Routing, Delivery and accepted Fuel US-38 producers; future Trip, Freight, Driver/Fleet, Tracking, Compliance and Integration producers | Minimized detected exception fact with trusted Tenant, stable source event, logical source reference, candidates, summary code and safe metadata | `OperationalExceptionFactV1` through P1-01 shared `DurableEventPublisher`; no foreign repository/entity/table/API scraping | COMPLETE_US78; active ROUTING + DELIVERY + FUEL |
| Notification | Operations | Safe escalation fact only | `OPERATIONAL_EXCEPTION_ESCALATED_V1` through P1-01; Notification owns rule/channel/template/recipient/retry | COMPLETE_US78 |
| Operations | Identity, US-80, US-81, US-83 | Validate same-Tenant user/role queue, apply fixed workflow/scheduling boundaries, and reference governed documents | Focused published ports/logical IDs only; no foreign persistence | COMPLETE_US78 |
| Fuel Performance | Fleet, Trip and Tenancy | Bulk Tenant-scoped compatible Vehicle/Driver context, authoritative Driver/Trip attribution, and trusted Tenant/timezone/currency context | Published provider-neutral root query contracts only; no foreign repository/entity/table/SQL and no N+1 calls | COMPLETE_US37_FINAL_ACCEPTANCE_PASS |
| Reporting | Fuel Performance | Fuel-owned summary, Vehicle/Driver comparison and trend projection with quality/lineage | Fuel-root `FuelPerformanceQuery`; Reporting does not redefine metrics or access Fuel persistence | COMPLETE_US37_FINAL_ACCEPTANCE_PASS |
| Fuel Cards | Organization, Fleet/Driver, Trip and Fuel Purchase | Same-Tenant provider, one active Vehicle-or-Driver binding, optional supporting Trip and one existing Fuel Purchase reconciliation target | Implemented provider-neutral root contracts and UUID logical references only; no foreign repository/entity/table/SQL or physical cross-module FK | US35_COMPLETE_FINAL_ACCEPTANCE_PASS |
| Fuel Cards | External provider / Integration | Provider owns account, authorization, settlement and monetary ledger; Fuel accepts normalized transaction evidence | Phase 1 uses Fuel-owned authenticated controlled JSON import. US-73 remains outbound-only and is not reused; named provider/shared inbound capability require new approval | CONTROLLED_PROVIDER_FIXTURE_DECISION |
| Fuel Exceptions (US-38) | Fuel Issue/Purchase/Price/Card/Performance/Bunker facts; published Fleet reading correction; Operations US-78 intake; Audit/Notification/Document boundaries | Six safe Fuel categories, immutable evidence, owner-command correction, independent approval, and minimized operational handoff | Fuel-local ports and logical references; `OperationalExceptionFactV1` through P1-01 only when escalation is required; no foreign repository/table or distributed transaction | COMPLETE; FINAL_ACCEPTANCE_PASS; V66/V67 |
| Billing (US-47) | Trip, Freight, Organization, Tenancy and US-72 | Closed Trip or explicit Freight completed/closed billable projection, active Customer reference, Tenant currency, and tax/billing compliance decision | Implemented published provider-neutral minimized versioned facts only; no foreign repository/entity/table/SQL or physical FK | IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING |
| Integration | Billing (US-47) | Immutable finalized transport billing export fact | P1-01 `TransportBillingExportRequestedV1` / `TRANSPORT_BILLING_V1`, then controlled `FILE_JSON_V1`; no business-rule mutation | IMPLEMENTED_US47 / ACCEPTANCE_PENDING |
| US-72 Compliance | Billing (US-47) | Minimized finalized tax category, jurisdiction, taxable amount, supplied rate/amount, exemption and snapshot facts | Versioned Billing-owned fact/explicit decision contract; Compliance owns the decision | APPROVED_DEPENDENCY; US-72 implementation pending |

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
- US-35 implements Fuel ownership of the limited card-reference master, local controls, immutable imported facts, reconciliation and review indicators. US-32 remains the economic Fuel Purchase authority; imports never auto-create a purchase. The provider retains financial authority. US-35 has no P1-01 event or automatic US-38/US-78 handoff.
- US-47 ratifies `billing` as a dedicated top-level context. Billing owns operational transport charge composition, validation, approval, finalization, reversal and history; Finance/external accounting retains formal tax invoice, GL, AR, posting, payment and settlement authority. Organization owns Customer master. Only Closed Trip and explicit Freight completed/closed projections are billable in Phase 1. Controlled US-73 JSON-file delivery is evidence only and never means posted or paid.
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

Following US-46 final acceptance, 71 / 87 stories are accepted and exactly 16 remain: US-47, US-48 through US-55, US-72, US-76, US-82, and US-84 through US-87. They remain planned in four open waves after Wave A closure:

1. US-73 external integrations and US-78 operational-exception lifecycle;
2. US-37/35/38 Fuel plus US-46 payroll link and US-47 transport billing;
3. US-48 through US-55 provider-neutral GPS/tracking;
4. US-72 compliance and US-76 mobile operations;
5. US-85/84/87/82/86 integrity, resilience, user risk, analytics, and disruption.

Expected future owners remain planning hypotheses except that US-73 has established and accepted the dedicated `integration` owner as `COMPLETE_US73` and US-78 has established and accepted the dedicated `operations` owner as `COMPLETE_US78`. Existing `fuel`, `driver`, `offlinesync`, `reporting`, `identity`, and `system` boundaries retain their data. Distinct `tracking`, `compliance`, and `billing` contexts remain justified candidates requiring their own story gate. Finance continues to own ledger/payment/tax posting and HRM/payroll continues to own final salary processing.

US-73 final acceptance covers exactly one governed outbound JSON-file adapter with controlled-sandbox evidence; it does not imply a live named business ecosystem. US-78 and US-46 reuse P1-01; no second outbox, broker, exactly-once claim, foreign repository, or cross-module SQL exists. US-46 is accepted through V71 while controlled `FILE_JSON_V1` and all product boundaries remain unchanged. Wave B remains open for US-47, and the queue head is `US-47-TRANSPORT-BILLING-PRODUCT-DECISIONS-001`.
# US-46 Frozen Dependencies

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Trip | Driver/Fleet payroll-input feature | Tenant-scoped Completed/Closed Driver assignment and actual Trip timing projection | Active projection exists; payroll-specific minimum extension may be implemented without foreign persistence |
| Fleet Driver | Driver/Fleet payroll-input feature | `DriverLookup` identity/active status | Active (MVP) |
| Driver/Fleet payroll-input feature | Integration | P1-01 `DriverPayrollInputExportRequestedV1` / `DRIVER_PAYROLL_INPUT_V1` | Active (MVP); final acceptance PASS |
| Integration | Driver/Fleet payroll-input feature | Safe exchange/delivery status reference; controlled `FILE_JSON_V1` delivery | `DRIVER_PAYROLL_INPUT_V1` implemented; acceptance pending |
| Scheduling | Driver/Fleet payroll-input feature | Duty/roster baseline | Planned only; no current accepted dependency and no overtime inference |
| Payroll/HRMS | Transport suite | Final payroll calculation, acknowledgement and settlement outcome | External/deferred; no Phase 1 inbound contract |
| Finance | Payroll/HRMS outcome | Ledger/tax/bank/payment posting | External to US-46 |

# US-48 Frozen Dependencies

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; its provider-neutral onboarding and execution platform is technically complete through V76 while the repository Flyway head is V96. Flespi production HTTPS polling is retained, Generic signed-HMAC ingress is retained, and a separate Traccar HTTPS polling adapter is approved but remains implementation-pending. Supported-adapter plug-and-play means runtime onboarding without Tracking-domain redesign; it does not promise universal proprietary-protocol compatibility or dynamic code upload. Provider DTOs, Tenant claims and secrets cannot cross the SPI. Credential values remain behind Integration's published `IntegrationSecretResolver`, are resolved transiently by the coordinator and never grant Tracking access to Integration persistence. Tracking validates logical same-Tenant Vehicle UUIDs through Fleet's published query and stores no Fleet/Integration entity or cross-module FK. High-rate telemetry remains local Tracking/Kafka/Redis/Timescale state and does not traverse Integration exchange processing or P1-01. Teltonika FMC130 + Flespi remains the one-device physical-acceptance pilot; that evidence is still pending and is not inherited by downstream stories.

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Fleet | Tracking | Same-Tenant Vehicle identity/reference validation | ACTIVE_MVP / provider onboarding and ingestion |
| Trip | Tracking | Source-time Vehicle assignment fact used by implemented Tracking consumers | ACTIVE_MVP / published lookup, no foreign persistence |
| Tracking | US-49/50/52/53 | Trusted latest, normalized optional speed and immutable accepted position/history contracts | FROZEN_US48 / TECHNICAL_DEPENDENCY_SATISFIED; no physical-acceptance inheritance |
| Tracking | US-51 | Authoritative engine-running plus movement telemetry | IMPLEMENTATION_IN_PROGRESS / CS01_COMPLETE; V3 contract is verified but production publication, durable consumption and capability remain gated pending CS02 V101 and later verified source activation |
| Tracking | US-54 | US-49..53 producer states/events and Tracking freshness | BLOCKED_PRODUCERS; dashboard cannot recreate detector logic |
| Tracking | US-55 | Provider-independent loss/delay/trust plus coordinate, accuracy, ordering, impossible-movement, binding and recovery rules; minimized same-Tenant Notification and HIGH-only Operations integration | TECHNICALLY_COMPLETE / IMPLEMENTATION_COMPLETE_ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM at V100; field matrix 0 PASS / 0 FAIL / 9 external blocks; independent `US-55-TRACCAR-ADAPTER` pending |
| Integration | Tracking | Published `IntegrationSecretResolver` for opaque credential reference only; never table/repository access or packet transport | ACTIVE_MVP / V74_REMEDIATION_COMPLETE |

The ARB disposition keeps US-48 physical acceptance `ON_HOLD_EXTERNAL_PREREQUISITE` until a genuine supported device/provider journey exists. The hold is not completion, acceptance or waiver. Downstream technical work may consume proven contracts without inheriting field acceptance. US-49 is accepted; US-50, US-52, US-53, US-54 and US-55 are technically complete with governed field/external holds. US-51 D1-D11 are approved and CS01 canonical V3 semantics are complete at V100; CS02 V101 history/capability is next. Software fixtures do not require hardware, while production source activation still requires verified device-native engine-running evidence. Accounting remains 73/87. The exact open US-55 acceptance queue is `US-55-HANDLE-GPS-EDGE-CASES-FINAL-ACCEPTANCE-001`.

# US-49 Dependencies (CS01–CS05 Implemented)

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Tracking US-48 | Tracking US-49 | Same-Tenant TRUSTED, IN_ORDER WGS84 accepted position with Vehicle, source timestamp and position UUID | FROZEN_TECHNICAL_CONTRACT; no US-48 acceptance inheritance |
| Organization | Tracking US-49 | Published explicit-Tenant location lookup for DEPOT/CUSTOMER_SITE create/update/activation validation | CS04_ACTIVE_CONSUMPTION; root contract only, no foreign persistence or SQL |
| Tracking US-49 | Notification | Minimized `VehicleGeofenceTransitionedV1` transition fact through shared P1-01 outbox; V79 Tenant `ROLE` / `DISPATCHER` rule and IN_APP template | CS05_ACTIVE_DURABLE_AT_LEAST_ONCE / IDEMPOTENT_NOTIFICATION |
| Tracking US-49 | Operations | No direct contract | NONE; US-55 owns later GPS-exception integration |
| Delivery US-63 | Tracking US-49 | No shared aggregate, table or polygon | EXPLICITLY_DISTINCT; serviceability/capacity versus telemetry boundary detection |

US-49 CS01–CS05 are `COMPLETE`; US-49 remains implementation-in-progress without acceptance credit. Tracking owns geofence definitions, evaluation jobs, per-Vehicle state, immutable transitions, management APIs, safe audit and durable transition publication. Notification remains owner of rules, recipients, templates and notification persistence. Organization remains owner of depot/customer-site locations and is accessed only through its published Tenant-aware lookup. No cross-module physical foreign key, repository, join or entity relationship exists. Next task: `US-49-MANAGE-GEOFENCES-CS06-FRONTEND-001`.

### US-50 frozen dependencies

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Tracking US-48 | Tracking US-50 | Optional normalized `speedKph`, trusted/in-order Vehicle association, source time and immutable position identity | ACTIVE_TECHNICAL_CONTRACT; no US-48 acceptance inheritance |
| Trip | Tracking US-50 | Published `VehicleTripAssignmentLookup.findAt(tenantId,vehicleId,sourceTimestamp)` returning optional logical Trip/Driver/route/version attribution | CS03_PROVIDER_ADAPTER_IMPLEMENTED; Trip-owned source-time lookup, no Tracking access to Trip persistence and no foreign join |
| Tracking US-50 | Notification | Minimized `VehicleSpeedingDetectedV1` publication-port model; future shared P1-01 outbox | CS03_CONFIRMATION_PORT_INVOCATION_IMPLEMENTED / DURABLE_PUBLICATION_AND_CONSUMPTION_DEFERRED |
| Routing | Tracking US-50 | No dynamic legal road-limit or segment-matching contract | NONE_PHASE1; route/version is logical attribution only and thresholds are Tracking-owned operational configuration |
| Tracking US-50 | Driver | No mutation or event contract | NONE; Driver retains violation, discipline, performance, payroll and licence ownership |

### US-52 frozen dependencies

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Tracking US-48 | Tracking US-52 | TRUSTED, IN_ORDER, nonduplicate Vehicle position with WGS84, accuracy, source time and position ID | ACTIVE_TECHNICAL_CONTRACT; no US-48 acceptance inheritance |
| Trip | Tracking US-52 | Existing source-time assignment lookup with canonical persisted route revision | CS03_RUNTIME_CONSUMPTION_ACTIVE / V85; historical null remains absent, no latest fallback or foreign persistence |
| Routing | Tracking US-52 | Tenant-qualified immutable bounded route-revision geometry lookup | CS03_RUNTIME_CONSUMPTION_ACTIVE / V88 exact revision or truthful absence; no latest fallback |
| Tracking US-52 | Notification | Minimized detected/escalated durable facts through shared P1-01 outbox; V90 same-Tenant IN_APP Dispatcher catalogue | CS05_ACTIVE_DURABLE_AT_LEAST_ONCE / IDEMPOTENT_NOTIFICATION |
| Tracking US-52 | Operations US-78 | No automatic exception creation | NONE_PHASE1 |
| Routing US-22 | Tracking US-52 | Authorized changes use a new route revision and Trip attribution; optional disruption UUID only | FROZEN_OWNERSHIP; no foreign persistence |
| Tracking history | Tracking US-53 | Tenant-qualified immutable source-time positions from `tracking_position_history`; Redis and legacy history prohibited | CS03_ACTIVE; snapshot-bound keyset streaming, explicit boundary/retention evidence and deterministic bounded stop analysis |
| Trip | Tracking US-53 | Published replay-scope plus one-call, seven-day/2,000-interval bounded Vehicle assignment range with exact Trip/route-revision attribution | CS02_CONSUMER_ACTIVE; `LIMIT 2001`, no partial/N+1 fallback or Trip persistence access |
| Routing | Tracking US-53 | Published exact immutable route-revision geometry lookup | CS02_CONSUMER_ACTIVE; exact revision only, per-request context cache, no latest fallback |
| Tracking US-49/50/52 | Tracking US-53 | Tracking-owned query ports for labelled geofence, speed and route-deviation overlays | CS06_ACTIVE; same-Tenant bounded cursor-paged adapters retain producer acceptance status |
| Tracking US-51 | Tracking US-53 | Engine/idle comparison | NONE_PHASE1; authoritative engine-state capability unresolved |
| Trip | Tracking US-54 | `TripDashboardQuery.findActiveContexts(tenantId,vehicleIds,evaluatedAt)` returns minimized active Trip/Driver/route context for at most 100 Vehicles in one bulk query | CS02_ACTIVE_MVP; Tenant-qualified, no N+1 calls, no Tracking access to Trip persistence |

V91 adds no cross-module dependency. Tracking's Kafka history consumer now atomically writes three
Tracking-owned durable evaluation intents beside each retained Timescale fact. The asynchronous dispatcher
reuses existing Tracking geofence, speed and route-deviation ports; Trip and Routing remain accessible only
through their published source-time lookup contracts. Redis live projection remains independent. US-51 IDLE
dispatch is prohibited until authoritative engine state exists.

Approval annotates Tracking evidence only; it does not mutate Routing, Trip or Driver state. V85 is occupied
by the Trip route-revision prerequisite; CS01 is complete and later Tracking/Routing persistence expects V86
subject to head recheck. The subsequently approved hybrid telemetry platform uses V86 for its
verified TimescaleDB foundation. Kafka is the durable Tracking-local telemetry backbone and Redis
is only its live projection; neither changes foreign module ownership. V87 is reserved for
Timescale policy hardening, so US-52 immutable route-geometry persistence is resequenced to V88.
This sequencing change does not alter story accounting.

CS03 is complete at V88. Tracking consumes Trip source-time attribution and Routing exact immutable
revision geometry through published lookups, then atomically evaluates and persists its own
Tenant-scoped candidate, stable state and episode lifecycle. There are no cross-module database
queries or foreign keys. Kafka activation, publication, API, review workflow and UI remain deferred
to later governed slices. Next:
`US-52-MONITOR-ROUTE-DEVIATIONS-CS04-APIS-RBAC-AUDIT-001`.

TS02 is complete: Tracking secure ingress durably publishes the canonical
`tracking.telemetry.ingested.v1` record keyed by Tenant and Vehicle after trusted provider,
Tenant, Device and source-time association resolution. This remains Tracking-local infrastructure;
it adds no cross-module dependency and does not replace the shared business-event outbox. Redis
projection and Timescale consumption remain TS03 and TS04. Accounting remains 73/87.
