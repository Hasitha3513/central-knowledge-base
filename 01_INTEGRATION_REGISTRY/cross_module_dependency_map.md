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

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM` at current head V76. Remediation and independent technical closure pass, and the Tracking-owned flespi adapter remains `IMPLEMENTATION_COMPLETE / REAL_CAPTURE_PENDING`. `PLUG_AND_PLAY_FOR_SUPPORTED_ADAPTERS` CS01–CS05 are COMPLETE: provider-neutral SPI/registry, provider-connection persistence, device-provider binding persistence, provider execution coordinator and single-runtime Flespi SPI cutover. V76 adds no cross-module dependency: both same-Tenant foreign keys remain inside Tracking, provider DTOs/Tenant claims/secrets cannot cross the SPI, and credential values remain behind Integration's published `IntegrationSecretResolver`. The coordinator resolves that existing port transiently per job; Flespi receives only the transient secret and does not access Integration persistence or route telemetry through Integration. Tracking validates logical same-Tenant Vehicle UUIDs through the published Fleet reporting query and stores no Fleet/Integration entity or cross-module FK. High-rate telemetry remains local Tracking state and does not traverse Integration exchange processing or P1-01. Teltonika FMC130 + flespi remains the one-device acceptance pilot; CS06 management APIs, hardware/provider activation and real evidence are still pending. US-49–55 remain unimplemented consumers.

| Provider | Consumer | Contract | Status |
| :--- | :--- | :--- | :--- |
| Fleet | Tracking | Same-Tenant Vehicle identity/reference validation | FROZEN_US48 / NOT_IMPLEMENTED |
| Trip | Tracking | Minimized active Vehicle assignment fact where later composition requires it | FROZEN_US48 / OPTIONAL_NOT_IMPLEMENTED |
| Tracking | US-49/50/52/53 | Trusted latest, normalized optional speed and immutable accepted position/history contracts | FROZEN_US48 / TECHNICAL_DEPENDENCY_SATISFIED; no physical-acceptance inheritance |
| Tracking | US-51 | Engine-state plus movement telemetry | BLOCKED_REQUIRED_CAPABILITY; current FLESPI does not advertise IGNITION and no accepted alternate source exists |
| Tracking | US-54 | US-49..53 producer states/events and Tracking freshness | BLOCKED_PRODUCERS; dashboard cannot recreate detector logic |
| Tracking | US-55 | Loss/delay/trust plus tamper/spoof/battery signals | PARTIAL_TECHNICAL_CONTRACT / BLOCKED_REQUIRED_SIGNALS_AND_DECISIONS |
| Integration | Tracking | Published `IntegrationSecretResolver` for opaque credential reference only; never table/repository access or packet transport | ACTIVE_MVP / V74_REMEDIATION_COMPLETE |

The ARB disposition places US-48 physical acceptance `ON_HOLD_EXTERNAL_PREREQUISITE` until a physical FMC130 and live Flespi prerequisites become available. The hold is not completion, acceptance or waiver: US-48 remains `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM` and accounting remains 72/87. Downstream implementation may consume technically proven frozen contracts without inheriting US-48 acceptance. US-49 is the next source-ordered task and is `TECHNICAL_DEPENDENCY_SATISFIED / READY_FOR_PRODUCT_DECISIONS`; US-50 and US-52 are likewise ready for decisions, and US-53 is ready after earlier Wave C decisions for its optional overlays. US-51 is blocked by missing engine-state capability, US-54 by US-49..53 producers, and full US-55 by unestablished tamper/spoof/battery signals and decisions.

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
| Trip | Tracking US-52 | Existing source-time assignment lookup with canonical persisted route revision | PREREQUISITE_COMPLETE / V85; historical null remains absent, no latest fallback |
| Routing | Tracking US-52 | Tenant-qualified immutable bounded route-revision geometry lookup | CS01_CONTRACT_PROVIDER_COMPLETE / EMPTY_UNTIL_CS02_IMMUTABLE_GEOMETRY_PERSISTENCE |
| Tracking US-52 | Notification | Minimized detected/escalated durable facts through shared P1-01 outbox | FROZEN_US52 / NOT_IMPLEMENTED |
| Tracking US-52 | Operations US-78 | No automatic exception creation | NONE_PHASE1 |
| Routing US-22 | Tracking US-52 | Authorized changes use a new route revision and Trip attribution; optional disruption UUID only | FROZEN_OWNERSHIP; no foreign persistence |

Approval annotates Tracking evidence only; it does not mutate Routing, Trip or Driver state. V85 is occupied
by the Trip route-revision prerequisite; CS01 is complete and later Tracking/Routing persistence expects V86
subject to head recheck. The subsequently approved hybrid telemetry platform allocates V86 to
Tracking infrastructure, so US-52 immutable route-geometry persistence is resequenced to V87
after the hybrid slices close. This sequencing change does not alter module ownership or story accounting.
