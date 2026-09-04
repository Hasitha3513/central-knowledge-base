# Transportation and Logistics

Lifecycle: IN DEVELOPMENT
Source repository: current workspace
Schema baseline: Flyway V1–V60 (V60: P1-01 shared durable-event outbox)
Delivery US-56, US-57, US-58, US-59, US-60, US-61, US-62: COMPLETE
MVP 1.3 Delivery Operations: 7/7 COMPLETE (CLOSED)
US-63 Manage Delivery Zones: COMPLETE (MVP-1.4-US63-DELIVERY-ZONES-FINAL-ACCEPTANCE-001) — V52 Flyway, pure Java ray-casting PiP, DeliveryZoneController, DeliveryZoneListPage.tsx
US-64 Manage Delivery Slots: COMPLETE (MVP-1.4-US64-DELIVERY-SLOTS-FINAL-ACCEPTANCE-001) — V53 Flyway, DeliverySlotController, DeliverySlotListPage.tsx
US-65 Manage Riders: COMPLETE (MVP-1.4-US65-RIDERS-FINAL-ACCEPTANCE-001-RERUN) — V54 Flyway, DeliveryRiderController, DeliveryRiderListPage.tsx, deliveryRiders.spec.ts
US-66 Batch Delivery Orders: COMPLETE (MVP-1.4-US66-BATCH-DELIVERY-ORDERS-FINAL-ACCEPTANCE-001) — V55 Flyway, DeliveryBatchController, DeliveryBatchListPage.tsx, deliveryBatches.spec.ts
US-67 Calculate Last-Mile ETA: COMPLETE (MVP-1.4-US67-LAST-MILE-ETA-FINAL-ACCEPTANCE-001-RERUN) — HEURISTIC_ONLY computed projection, RiderEtaContextPort, tenant-scoped generation-aware cache/invalidation, V56 Rider transport-mode migration; acceptance-time Flyway head V57. Final acceptance: Maven 1,195/0/0/15, architecture 40/40, and real PostgreSQL-backed Chromium 6/6 PASS.
US-68 Handle Last-Mile Exceptions: COMPLETE (MVP-1.4-US68-LAST-MILE-EXCEPTIONS-FINAL-ACCEPTANCE-001) — read-only Delivery Planner projection over US-59/60/62/57/65/66/67 capabilities; no migration, aggregate, persistence, permission, or duplicate API family. Final acceptance: Maven 1,200/0/0/15, architecture 45/45, and real PostgreSQL-backed Chromium 3/3 PASS.
US-69 Receive Delivery Notifications: COMPLETE (MVP-1.4-US69-DELIVERY-NOTIFICATIONS-FINAL-ACCEPTANCE-001-RERUN) — Batch `READY` emits zero `DELIVERY_OUT_FOR_DELIVERY` events; committed `DISPATCHED` emits exactly one per active member; removed members and rollback emit zero. Final evidence: Maven 1,223/0/0/15, architecture 42/42, and real PostgreSQL-backed Chromium 7/7 PASS.
US-70 Use Customer Self-Service: COMPLETE (MVP-1.4-US70-CUSTOMER-SELF-SERVICE-FINAL-ACCEPTANCE-001) — V59 opaque per-Delivery magic-link access; customer-safe tracking/preferences/issues/feedback and non-binding redelivery requests; no Customer-to-app_user association, direct slot scheduling, IN_APP, OTP, Rider data, or POD evidence exposure. Final evidence: focused 28/28, PostgreSQL 4/4, Maven 1,238/0/0/15, architecture 42/42, frontend Vitest 259/259, and real PostgreSQL-backed Chromium 9/9 PASS.
MVP 1.4 Last-Mile Delivery: 8/8 COMPLETE (CLOSED). Overall register: 68/87 COMPLETE and 19/87 REMAINING after US-37 final acceptance.
P1-01 Event Contract Durability and Envelope Hardening: COMPLETE — 32 events inventoried; the US-69 five-type family uses the shared V60 outbox with at-least-once delivery, while nine consumed local events remain after-commit and 22 unconsumed events remain unchanged. Program accounting is unchanged.
Delivery US-66 final acceptance gate: COMPLETE (MVP-1.4-US66-BATCH-DELIVERY-ORDERS-FINAL-ACCEPTANCE-001)
Delivery US-65 final acceptance gate: COMPLETE (MVP-1.4-US65-RIDERS-FINAL-ACCEPTANCE-001-RERUN)
Delivery US-64 final acceptance gate: COMPLETE (MVP-1.4-US64-DELIVERY-SLOTS-FINAL-ACCEPTANCE-001)
Delivery US-63 final acceptance gate: COMPLETE (MVP-1.4-US63-DELIVERY-ZONES-FINAL-ACCEPTANCE-001)
Delivery US-62 final acceptance gate: COMPLETE (MVP-1.3-US62-DELIVERY-EXCEPTIONS-FINAL-ACCEPTANCE-001)
Delivery US-61 acceptance gate: COMPLETE (MVP-1.3-US61-ANALYTICS-FINAL-ACCEPTANCE-001)
Delivery US-60 acceptance gate: COMPLETE (MVP-1.3-US60-REDELIVERY-FINAL-ACCEPTANCE-001)
US-30 Cargo Exceptions: COMPLETE (P2-CARGO-EXCEPTION-001)
US-29 Freight Reporting: COMPLETE (P2-FREIGHT-REPORTING-001)
Freight release status: 7/7 COMPLETE
Tenant readiness: FOUNDATION IMPLEMENTED / OPERATIONAL ISOLATION ACCEPTED_P0-04_CURRENT_SCOPE

## Mission and Bounded Contexts

Transportation manages fleet master/usage, drivers (legacy ownership), routing, trips, fuel and bunker operations, freight orders/manifests/load planning/insurance/exceptions, delivery operations foundation, operational notifications, reporting, identity/organization references, and offline command ingestion.

| Context | Principal models | Representative use cases |
| :--- | :--- | :--- |
| Fleet | Vehicle, Category, Type, Document, Reading, Meter Reset, Maintenance Schedule, Lubricant Log | manage vehicles; availability; compliance; readings/corrections/resets |
| Driver | Driver, License, Exception, Violation, Medical Record, Drug Test | eligibility, availability, compliance, performance |
| Routing | Route, Stop, Revision, Disruption | CRUD, revision history, disruption lifecycle, optimization |
| Trip | Trip, Assignment, Dispatch, Status History, Operational Event | create, submit, approve, assign, dispatch, start, complete, close, cancel |
| Fuel | Station, Limit Policy, Issue, Purchase, Price, Bunker Tank/Movement | issue lifecycle, purchase lifecycle, reconciliation, stock control, trip cost |
| Freight | Order, Manifest, Load Plan, Insurance Policy/Claim/Settlement, Cargo Exception | order intake, manifest finalization, load placement, weight/volume validation, claim lifecycle, cargo exceptions |
| Delivery | Delivery Order, Delivery Number, Priority, Service Type, Window, Status | create/search/read/update requirements and validate readiness; authoritative Tenant context and logical organization references |
| Notification | Rule, Policy, Template, Notification, Delivery Attempt | event routing, suppression, escalation, delivery diagnostics |
| Offline Sync | Offline Operation | idempotent command inbox and conflict outcomes |

## P0-06 Aggregate Catalogue and Boundaries

| Aggregate root | Owned entities / value objects | Protected invariants and transaction boundary | Repository | Referenced aggregates | Module owner |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Vehicle | Vehicle identity, registration, classification references, ownership, capacity and operational state | Required explicit Category/Type references; registration and capacity/meter/status rules; one Vehicle write | `VehicleRepository` | Category ID, Type ID | Fleet |
| Vehicle Document | Document identity, type/number, validity, mandatory/status/audit facts | Document date/status consistency and active duplicate prevention; one document write | `VehicleDocumentRepository` | Vehicle ID | Fleet |
| Driver | Driver identity, employee/contact and operational state | Addressed root identity is immutable across update/deactivation; one Driver write | `DriverRepository` | None | Fleet |
| Driver Qualification / Availability records | Licence, medical, drug-test, exception and violation roots; `DriverAvailability` result value | Each record protects its own validity/status lifecycle; availability composes read-only eligibility reasons | Focused repository per independently managed record | Driver ID; Trip assignment queried through published boundary | Fleet |
| Trip | Current Trip order, assignment-state and execution fields | Order validity, assignment completeness/overlap, lifecycle, actual times and odometer progression within the existing Trip transaction | `TripRepository`; separate dispatch/history/event repositories are audit/evidence stores | Customer, Department, Project, Route, Vehicle, Driver and Type IDs | Trip |
| Route | Ordered stop-location ID values | Distinct origin/destination, positive distance/duration, unique bounded stops; one Route write | `RouteRepository` | Location IDs | Routing |
| Fuel Issue | Fuel issue facts and lifecycle; issue history is immutable audit | Draft validity, authorization/issue/cancel transitions, limits, price snapshot and bunker deduction transaction | `FuelIssueRepository`; `FuelIssueHistoryRepository` for audit | Vehicle, Trip, Driver, Station and Actor IDs | Fuel |
| Delivery | Narrow roots for Order, POD, failed attempt/contact/escalation, redelivery, exception case/evidence, zone, slot/reservation, rider/shift/assignment and batch/membership | Each root protects its own lifecycle and concurrency rules; Delivery Exception evidence is parent-owned and cascaded only by its Exception Case | Focused root/read repositories | Customer, Location, Driver, Trip/Freight and related Delivery root IDs | Delivery |

P0-06 deliberately retains Trip as one current aggregate because order, assignment state, dispatch/start/complete transitions and concurrency are presently one consistency boundary. `TripDispatch`, `TripHistoryEntry`, and `TripOperationalEvent` are independently persisted audit/evidence, not a JPA object graph. Vehicle does not own allocations: existing Vehicle/Driver allocations are Trip assignment facts, queried through allocation/eligibility contracts. Vehicle documents and Driver qualifications remain separate roots because they have independent identity, lifecycle, authorization, query, and retention needs.

All current cross-aggregate references are IDs/value references. The only Delivery JPA object graph is `DeliveryExceptionCaseEntity -> DeliveryExceptionEvidenceEntity`, matching the parent-owned evidence lifecycle. P0-06 adds automated enforcement against unreviewed JPA associations and cascades. No repository was removed: repositories for histories, evidence, independent compliance records, and read models remain justified and do not grant cross-module access.

## Published Events

P1-01 inventoried 32 production event classes or persisted event records: 22 have no consumer, nine remain local after-commit, and the five-type `DeliveryCustomerNotificationEvent` family is the one durable-internal family. Exact classifications, payloads, and consumers are registered in `../01_INTEGRATION_REGISTRY/event_contracts.md` and the application P1-01 architecture record. P0-07 continues to govern local publication: events registered inside an owning transaction are delivered only after commit, never after rollback. `OperationalNotificationEvent` now implements the canonical version-1 envelope with explicit Tenant and aggregate identity but remains local because Notification already persists and retries channel work.

Primary state transitions—including Trip assignment/dispatch/start/complete, Fuel Issue stock/history/reading changes, Fleet readings, and Delivery order/batch/rider changes—remain synchronous owning-module ACID operations. Delivery ETA cache eviction and `OperationalNotificationEvent` handling remain secondary local after-commit reactions. V60 atomically persists only accepted US-69 Delivery facts through the shared technical outbox; a Tenant-aware worker delivers them outside the source transaction with bounded retries. Notification execution keys provide the consumer inbox/dedupe boundary. No external broker, global ordering, exactly-once claim, general event export, or rewrite of unused events is approved.

US-56 publishes no cross-module event because it captures requirements and readiness only; no downstream workflow is triggered. Future Delivery events require registration before implementation.

## Phase 1 Delivery Operations Foundation (MVP 1.3)

- Delivery is a dedicated Spring Modulith module at `com.transportlogistics.app.delivery`.
- US-56 adds the framework-free `DeliveryOrder`, priority/service/window value types, application service, ports, adapters, persistence and REST/UI slice to the existing foundation.
- All future Delivery operational aggregates, rows, commands, queries, APIs, repository operations, events, jobs, caches and analytics are tenant-owned and must derive `tenant_id` from authoritative server-side context; client-supplied Tenant authority is forbidden.
- Public `DeliveryLookupPort`, `DeliveryReference`, `DeliverySummary`, and the customer/location/freight-order/trip/slot/evidence/offline/notification outbound ports are preserved `PRE-EXISTING_UNCOMMITTED_FUTURE_SLICE` contracts. They have no adapters or implemented use cases and must not be represented as completed capability.
- No synthetic foundation-status application use case, placeholder bean, or runtime permission catalogue exists. Permission names are frozen governance metadata only and remain unseeded/unassigned until their source-defined story actions are implemented.
- Implementations must not query another module's repositories, JPA entities, services, adapters, or tables directly.
- V46 adds `delivery_order`, `delivery_number_counter`, four permissions and supporting indexes. US-56 REST/UI workflows are implemented; US-57 through US-62 remain unimplemented.

### US-56 Product Decisions

`MVP-1.3-US56-PRODUCT-DECISIONS-001` is implemented as follows:

- Priority: Delivery-owned fixed catalogue `LOW`, `NORMAL`, `HIGH`, `URGENT`; default `NORMAL`; recorded urgency only.
- Service type: Delivery-owned fixed catalogue `STANDARD`, `EXPRESS`, `SAME_DAY`, `SCHEDULED`; default `STANDARD`; explicit delivery window required and no implicit cross-scope behavior.
- Assignment: `NONE_IN_US56` and `NO_ASSIGNMENT_COLUMNS_IN_US56`. US-56 validates readiness only.
- Readiness facts: authoritative Tenant context; active same-Tenant customer and origin/destination locations; distinct locations; valid window; supported catalogues. Unknown external facts fail closed.
- Lifecycle subset: create `DRAFT`; validation may produce `READY_FOR_ASSIGNMENT`; requirement edits return the order to `DRAFT`.
- Delivery number: immutable, server-generated `DEL-YYYY-NNNNNN` (regex `^DEL-[0-9]{4}-[0-9]{6}$`), allocated atomically from a Delivery-owned counter scoped by `(tenant_id, tenant-local calendar year)`. The sequence starts at `000001`, permits permanent gaps, never reuses or wraps, and fails with `DELIVERY_NUMBER_SEQUENCE_EXHAUSTED` after `999999`.
- Number allocation uses a database uniqueness guard on `(tenant_id, delivery_number)` and must not use `MAX + 1`. A uniqueness collision permits at most three fresh allocation attempts before sanitized `DELIVERY_NUMBER_ALLOCATION_FAILED`. US-56 introduces no explicit idempotency-key contract.
- Status: US-56 `COMPLETE`; US-57 through US-62 remain unimplemented.
- Final acceptance: `MVP-1.3-US56-DELIVERY-ORDERS-FINAL-ACCEPTANCE-002` verified remote application commit `40eb120ac64cce44716598d267c68901127dd44a`, focused backend 51/51 PASS, full backend 972 tests with zero failures/errors (15 skipped), frontend 234/234 PASS, Chromium 2/2 PASS, and PostgreSQL 16 Flyway V1–V46 PASS.

### US-57 Proof-of-Delivery Product Decisions

`MVP-1.3-US57-POD-PRODUCT-DECISIONS-001` is implemented and verified (US-57 online POD + US-58 offline POD).

### US-59 Failed Deliveries Product Decisions & Implementation

`MVP-1.3-US59-FAILED-DELIVERIES-PRODUCT-DECISIONS-001` is implemented and verified (US-59 Failed Deliveries):

- **Failure Reason Taxonomy:** Standardized enums (`CUSTOMER_UNAVAILABLE`, `WRONG_ADDRESS`, `CUSTOMER_REFUSED`, `ACCESS_RESTRICTED`, `DAMAGED_CARGO`, `DOCUMENT_OR_PAYMENT_ISSUE`, `OTHER`). Arbitrary free-text-only status mutation is prohibited; `OTHER` requires mandatory non-empty notes (>= 10 chars), `CUSTOMER_REFUSED` and `DAMAGED_CARGO` require non-empty notes (>= 5 chars).
- **Delivery Lifecycle Extension:** `READY_FOR_ASSIGNMENT` can transition to `FAILED_ATTEMPT` (non-terminal, redelivery eligible), `RETURN_TO_BASE` (terminal/return custody), or `ESCALATED` (management hold). Finalized `DELIVERED` orders remain immutable and can never be marked failed.
- **Delivery Attempt & Contact Model:** Separate immutable entities `DeliveryAttempt` and `DeliveryContactAttempt` capturing sequential attempt numbering, UTC timestamps, failure reasons, contact channels (`PHONE`, `SMS`, `WHATSAPP`, `EMAIL`, `IN_PERSON`), contact outcomes, operator IDs, and tenant isolation.
- **Privacy & PII Protection:** Contact attempts record only channel and outcome metadata; customer phone numbers/emails remain referenced from Customer master and are not duplicated into logs or attempt payloads.
- **Escalation & RTO Semantics:** Local operational escalation tracking (`status`, `reason`, actor, timestamps). Return-to-Base marks orders as permanently failed in the field and triggers return custody.
- **Story Boundaries:** US-59 determines that another attempt is needed (`REDELIVERY_ELIGIBLE`). US-60 owns customer time preference collection and slot scheduling. US-61 owns analytics. US-62 owns specialized exception gates.
- **Offline Policy:** `ONLINE_ONLY_FOR_US59` in MVP Phase 1.3.
- **RBAC:** `DELIVERY_FAIL_RECORD`, `DELIVERY_FAIL_VIEW`, `DELIVERY_FAIL_ESCALATE`, `DELIVERY_RETURN_INITIATE`.
- **Persistence (V48):** Forward migration V48 adds tables `delivery_attempt`, `delivery_contact_attempt`, `delivery_escalation` and seeds US-59 permissions.

### US-60 Schedule Re-Delivery Product Decisions

`MVP-1.3-US60-REDELIVERY-PRODUCT-DECISIONS-001` is frozen as follows:

- **Eligibility Gate:** `DeliveryOrder.status == FAILED_ATTEMPT` AND latest `DeliveryAttempt.disposition == REDELIVERY_ELIGIBLE`. `DELIVERED`, `RETURN_TO_BASE`, `DRAFT`, and `ESCALATED` cannot enter redelivery directly.
- **Lifecycle Transition:** `FAILED_ATTEMPT` $\to$ `READY_FOR_ASSIGNMENT` atomically upon persisting confirmed redelivery schedule. Delivery window is updated to the newly scheduled window.
- **Schedule Model & History:** Entity `DeliveryRedeliverySchedule` (table `delivery_redelivery_schedule`, V49) capturing `schedulingMethod` (`AUTOMATIC`, `AGENT_ASSISTED`), advisory `customerPreference` (start/end time, notes), `scheduledWindow` (start/end time), status (`CONFIRMED`, `SUPERSEDED`, `CANCELLED`), operator identity, and timestamps. Prior schedules are superseded and retained for audit.
- **Slot Availability (MVP 1.3):** Validates scheduled windows against operational depot hours and checks concurrent delivery capacity per tenant/window. Advanced dynamic micro-zoning and customer booking portals remain deferred to US-64.
- **Concurrency & Races:** Optimistic concurrency via `DeliveryOrder.version`. Schedule vs POD race and Schedule vs RTO race fail on version conflict (`409 Conflict`).
- **RBAC:** `DELIVERY_REDELIVERY_SCHEDULE`, `DELIVERY_REDELIVERY_VIEW`.
- **Offline & Notification Policy:** `ONLINE_ONLY_FOR_US60` for MVP 1.3. Internal domain event `DeliveryRedeliveryScheduledEvent` emitted. Customer notifications (US-69) deferred.

### US-61 Analyze Delivery Performance Product Decisions

`MVP-1.3-US61-ANALYTICS-PRODUCT-DECISIONS-001` is frozen as follows:

- **Mathematical KPIs:**
  - `Order Success Rate (%)`: $\frac{DELIVERED}{DELIVERED + RETURN\_TO\_BASE} \times 100$ over terminal completed outcomes. In-flight orders excluded from denominator; returns `null` (N/A) when denominator is 0.
  - `First-Attempt Success Rate (%)`: $\frac{DELIVERED \text{ with 0 failed attempts}}{Total DELIVERED} \times 100$.
  - `On-Time Delivery Rate (%)`: $\frac{Delivered \text{ with completion} \le \text{committedWindowEnd}}{Total DELIVERED} \times 100$. (Reference window is original `window_end` or latest confirmed redelivery `scheduled_end_time`; actual completion is `proof_of_delivery.accepted_at`).
  - `Average Delay (Minutes)`: Calculated strictly over late deliveries ($\text{completion} - \text{committedWindowEnd}$).
  - `Attempt Distribution`: Count of orders grouped by failed attempts (0, 1, 2, 3+).
  - `Failure Reason Distribution`: Grouped by US-59 `failure_reason` with count and percentage.
  - `Redelivery Performance`: Total redelivery orders, redelivery rate, and redelivery success rate.
- **Regional Grouping:** Aggregated by destination location / region via `OrganizationLookupPort`; falls back to `"UNCLASSIFIED"` if location metadata is absent.
- **Query Bounds & Timezone:** Default 30-day range, maximum 365-day range; daily/weekly/monthly aggregations use tenant operating timezone (`Asia/Colombo`).
- **Read-Only Invariant:** Strictly read-only operational queries; zero mutation of orders, PODs, attempts, or schedules.
- **RBAC & Security:** Permission `DELIVERY_ANALYTICS_VIEW` (to be seeded in V50); multi-tenancy derived strictly from `CurrentTenant`. Customer PII and raw binary POD evidence excluded from payloads.
- **Module Ownership:** Owned by `delivery` module with public read interface `DeliveryReportingQuery` for centralized reporting integration.

## Phase 1 Freight Manifest Special-Cargo Classification & Cargo Measurements (US-27)

- Manifest item create/update commands and public REST payloads carry nullable `fragile` and `temperatureSensitive` fields without collapsing UNKNOWN to `false`.
- Manifest item create/update commands and public REST payloads carry nullable `unitWeight`, `weightUnit` ('KG', 'G', 'TONNE'), `length`, `width`, `height`, and `dimensionUnit` ('M', 'CM', 'MM') for US-27 weight, volume, and vehicle capacity validation.
- First-party item creation/editing requires explicit Yes/No decisions for special cargo fields; historical UNKNOWN remains visible as `CLASSIFICATION REQUIRED`.
- Cargo items with missing measurements are displayed with `WEIGHT REQUIRED` / `DIMENSIONS REQUIRED` tags and evaluated as `INCOMPLETE` in US-27 Weight/Volume validation.
- Unfinalized UNKNOWN items may be classified by actors with `CARGO_MANIFEST_MANAGE`; finalized manifests remain read-only, including historical UNKNOWN records.
- `SPECIAL_CARGO_CLASSIFICATION_MISSING` is returned through readiness and the standard finalization error envelope until every item is explicitly classified.
- Load Planning consumes manifest item measurements via `CargoManifestLookupPort` to execute US-27 pure calculation engine (`WeightVolumeCalculationEngine`), producing legitimate PASS, FAIL, or INCOMPLETE outcomes.

## External Integration Points

- Publishes trip/freight execution, vehicle usage, route disruption, fuel, and delivery facts after contracts are approved.
- Consumes HRM driver qualification, Maintenance hold/release, Finance controls/posting outcomes, Sales order/customer references, and Inventory parts/fuel references through registered contracts.
- Current local `driver`, `customer`, `project`, `vendor`, and `maintenance_schedule` ownership is legacy. Extraction or reassignment requires ADRs and compatibility plans.
- US-73 freezes a separate top-level Integration context; Transportation retains every business fact and rule. No Transportation event family is approved for external delivery by US-73. The only frozen executable contract is Integration's own non-sensitive `US73_PLATFORM_PROBE_V1` for controlled-sandbox evidence. Future Transportation-to-Integration contracts require their own exact payload, classification, ownership, retention, and consumer tests.

## Database Schema Data Dictionary

### Schema-wide tenancy status

Flyway V43 implements the first-class Tenant and Tenant membership foundation. V44 adds membership-scoped role assignment and non-null, indexed `tenant_id` ownership to current-scope operational tables across Identity token persistence, Organization, Fleet/Driver, Routing/Trip, Fuel, Freight, Notification, and Offline Sync. V57 replaces legacy global operational business-key constraints with Tenant-local uniqueness for Organization, Fleet/Driver, Routing/Trip, Fuel, and Freight; it also Tenant-scopes notification execution keys and bunker movement idempotency. Existing physical foreign keys reflect the current modular monolith and are factual documentation, not approval for future cross-module coupling. Historical migrations are immutable.

P0-05 hardens the existing Identity model without schema changes. Identity administration resolves an explicit application context from the authenticated request: Tenant ID, actor, and current server-side permissions. User lookup/list/mutation is Tenant-scoped, user creation assigns membership to the actor's Tenant, permission grants are capped by the actor's current permission set, and role templates assigned in another Tenant cannot be updated or deleted. Unmatched HTTP routes deny by default. JWT structure remains unchanged and embedded authorities are not trusted without server-side reload.

### P0-02 authoritative table ownership registry

| Owner | Tables |
| :--- | :--- |
| tenancy | `tenant` |
| identity | `app_user`, `app_role`, `app_permission`, `app_user_role`, `app_role_permission`, `refresh_token`, `tenant_membership`, `tenant_membership_role` |
| organization | `customer`, `department`, `location`, `project`, `vendor` |
| fleet | `driver`, `driver_license`, `driver_exception`, `driver_violation`, `driver_medical_record`, `driver_drug_test`, `vehicle_category`, `vehicle_type`, `vehicle`, `vehicle_document`, `vehicle_reading`, `vehicle_meter_reset`, `maintenance_schedule`, `lubricant_log` |
| routing | `route`, `route_stop`, `route_revision`, `route_revision_stop`, `route_disruption` |
| trip | `trip`, `trip_status_history`, `trip_dispatch`, `trip_operational_event` |
| fuel | `fuel_station`, `fuel_limit_policy`, `fuel_issue`, `fuel_issue_history`, `fuel_price`, `fuel_purchase`, `fuel_purchase_history`, `bunker_tank`, `bunker_dip_reading`, `bunker_stock_adjustment`, `bunker_stock_movement` |
| freight | `freight_order`, `freight_order_line`, `cargo_manifest`, `cargo_manifest_item`, `load_plan`, `load_plan_item_placement`, `freight_insurance_policy`, `freight_insurance_claim`, `freight_insurance_settlement`, `cargo_exception`, `cargo_exception_history` |
| delivery | `delivery_order`, `delivery_number_counter`, `proof_of_delivery`, `pod_evidence`, `delivery_attempt`, `delivery_contact_attempt`, `delivery_escalation`, `delivery_redelivery_schedule`, `delivery_exception_case`, `delivery_exception_evidence`, `delivery_zone`, `delivery_slot`, `delivery_slot_reservation`, `delivery_rider`, `delivery_rider_zone`, `delivery_rider_shift`, `delivery_order_rider_assignment`, `delivery_batch`, `delivery_batch_order`, `delivery_batch_counter` |
| notification | `notification`, `notification_template`, `notification_rule`, `notification_rule_policy`, `notification_rule_quiet_day`, `notification_rule_execution`, `notification_delivery_attempt`, `customer_notification_preference` |
| offlinesync | `offline_sync_operation` |
| shared technical events | `integration_outbox_event` |

Entityless join, collection, counter, and permission-catalogue tables remain owned by the listed module. Reporting is a consumer of published query contracts and owns no operational source table. P0-03 removed the two production-code direct-SQL violations: Freight reads Fleet capacity through `FleetReportingQuery`, while the development-only System fixture no longer probes an Organization-owned table or repository. The production foreign-SQL baseline is empty. The opt-in multi-owner development fixture and test-only cross-owner SQL setup/cleanup remain provisioning/test-infrastructure debt and are not production ownership authority.

### Tenant foundation tables (V43)

#### Table: `tenant`

- **Purpose:** Platform-owned root Tenant identity and lifecycle record.
- **Primary Key:** `tenant_id` (UUID)
- **Multi-Tenant Key:** Not applicable — root Tenant catalogue.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | PRIMARY KEY | Immutable Tenant identifier |
| `tenant_code` | VARCHAR(40) | NO | - | UNIQUE; nonblank CHECK | Stable Tenant code |
| `tenant_name` | VARCHAR(200) | NO | - | Nonblank CHECK | Legal/display name |
| `default_currency` | VARCHAR(3) | NO | - | Nonblank CHECK | ISO-4217 currency |
| `default_time_zone` | VARCHAR(80) | NO | - | Nonblank CHECK | IANA time zone |
| `status` | VARCHAR(20) | NO | - | CHECK in `ACTIVE`, `INACTIVE` | Lifecycle status |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation time |
| `created_by` | VARCHAR(120) | NO | - | - | Creation actor |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update time |
| `updated_by` | VARCHAR(120) | NO | - | - | Last update actor |
| `version` | BIGINT | NO | `0` | - | Optimistic version |

V43 deterministically seeds UUID `4f8b6a3b-2c1e-4d89-9a72-f9e4c5b3671a`, `CLTS-LK`, Ceylon Logistics & Transport Solutions (Pvt) Ltd, `LKR`, `Asia/Colombo`, `ACTIVE`.

#### Table: `tenant_membership`

- **Purpose:** Identity-owned association between a global credential user and an operational Tenant.
- **Primary Key:** `membership_id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `membership_id` | UUID | NO | - | PRIMARY KEY | Membership identifier |
| `tenant_id` | UUID | NO | - | FK -> `tenant(tenant_id)`; indexed | Authorized Tenant |
| `user_id` | UUID | NO | - | FK -> `app_user(id)` ON DELETE CASCADE; UNIQUE | User; uniqueness enforces one MVP membership |
| `status` | VARCHAR(20) | NO | - | CHECK in `ACTIVE`, `INACTIVE` | Membership status |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation time |
| `created_by` | VARCHAR(120) | NO | - | - | Creation actor |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update time |
| `updated_by` | VARCHAR(120) | NO | - | - | Last update actor |
| `version` | BIGINT | NO | `0` | - | Optimistic version |

### Baseline master and operations tables (V1–V10)

| Table | Purpose | Primary key | Columns (type; `?` nullable) | Constraints and indexes |
| :--- | :--- | :--- | :--- | :--- |
| `app_user` | Login user | `id UUID` | username varchar80, email varchar160, password_hash varchar255, first_name/last_name varchar100, phone varchar40?, active boolean, created_at/updated_at timestamptz | unique username/email |
| `app_role` | Global role-template catalogue | `id UUID` | name varchar80, description varchar255?, active boolean | unique name; `GLOBAL` RBAC definition |
| `customer` | Transport customer reference | `id UUID` | code varchar40, name varchar160, contact_person varchar160?, phone/email?, active boolean | unique code |
| `department` | Department reference | `id UUID` | code varchar40, name varchar160, description varchar255?, active boolean | unique code |
| `location` | Geographical/business location | `id UUID` | code varchar40, name varchar160, address varchar255?, latitude/longitude double?, active boolean | unique code |
| `project` | Project reference | `id UUID` | code varchar40, name varchar160, department_id UUID?, active boolean | unique code; FK department |
| `driver` | Legacy driver master | `id UUID` | employee_number varchar60, first_name/last_name varchar100, phone/email?, status varchar40, active boolean | unique employee_number; index active/status |
| `vehicle_category` | Vehicle classification | `id UUID` | code varchar40, name varchar160, description varchar255?, active boolean | unique code |
| `vehicle_type` | Vehicle type | `id UUID` | category_id UUID, code varchar40, name varchar160, description varchar255?, active boolean | unique code; FK category |
| `vehicle` | Vehicle master | `id UUID` | registration_number varchar80, chassis_number/engine_number varchar120?, category_id/type_id UUID, manufacturer/model?, manufacture_year int?, ownership_type/operational_status varchar40, odometer/engine_hours/capacity double?, active boolean, capacity_kg numeric(19,4)?, tare_weight_kg numeric(19,4)?, gross_vehicle_weight_kg numeric(19,4)?, cargo_volume_capacity_m3 numeric(19,4)?, axle_count int?, max_axle_load_kg numeric(19,4)? | unique registration; FKs category/type; status and category/type indexes; capacity constraints |
| `route` | Route master | `id UUID` | code varchar40, name varchar160, origin_location_id/destination_location_id UUID, distance double?, duration int?, active boolean | unique code; FKs locations; search index |
| `route_stop` | Ordered route stop | `(route_id,stop_order)` | route_id UUID, stop_order int, location_id UUID | FK route cascade/location; unique route/location |
| `trip` | Trip aggregate | `id UUID` | trip_number varchar60, customer/department/project/route UUID?, priority/status varchar, origin/destination UUID, requested times timestamptz, required vehicle/capacity?, cargo/passenger/instructions/notes?, assigned vehicle/driver?, actual times/readings/remarks?, created_at/updated_at | unique trip_number; lifecycle/period/allocation indexes; current physical FKs to references |
| `vehicle_document` | Vehicle compliance document | `id UUID` | vehicle_id UUID, type/number, issue/expiry date?, file_reference?, mandatory boolean, status, active, audit timestamps/users | FK vehicle; date/status checks; vehicle/dispatch indexes |
| `driver_license` | Driver licence | `id UUID` | driver_id UUID, number/class, issue/expiry date, status, active, audit timestamps/users | unique number; FK driver; date/status checks and availability index |
| `trip_status_history` | Trip lifecycle audit | `id UUID` | trip_id UUID, from_status?, to_status, action, vehicle_id?, driver_id?, license_class?, actor, details?, occurred_at | FKs trip/vehicle/driver; trip-time and driver indexes |
| `trip_dispatch` | Dispatch record | `trip_id UUID` | dispatched_at, dispatched_by, remarks? | FK trip |

### Identity and authorization tables (V2 and permission migrations)

| Table | Purpose | Primary key | Columns | Constraints and indexes |
| :--- | :--- | :--- | :--- | :--- |
| `app_permission` | Global permission catalogue | `code varchar100` | description varchar255, active boolean | PK code; `GLOBAL` RBAC definition |
| `app_user_role` | Legacy user-role assignment | `(user_id,role_id)` | user_id UUID, role_id UUID | cascade FKs user/role; `LEGACY_UNSCOPED_ROLE_ASSIGNMENT`, transitional |
| `tenant_membership_role` | Tenant-scoped role-template assignment | `(membership_id,role_id)` | membership_id UUID, role_id UUID | cascade FKs membership/role; runtime authorization authority |
| `app_role_permission` | Global role-template permissions | `(role_id,permission_code)` | role_id UUID, permission_code varchar100 | FKs role/permission; `GLOBAL` RBAC definition |
| `refresh_token` | Hashed refresh token | `id UUID` | user_id UUID, token_hash varchar64, created/expires/revoked timestamps, replaced_by UUID? | unique hash; FK user cascade; user/expiry indexes |

### Fuel and bunker tables (V11–V12, V18)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `fuel_station` | Internal/external station | `id UUID` | code, name, station_type, active, vendor_id?, location_id? | unique code; type `INTERNAL/EXTERNAL`; FK location |
| `fuel_limit_policy` | Per-vehicle/default issue limit | `id UUID` | vehicle_id?, maximum_quantity_per_issue numeric19,3, active | positive check; FK vehicle; lookup index |
| `fuel_issue` | Fuel issue voucher | `id UUID` | voucher, vehicle/trip/driver/station/user IDs, fuel type, quantities/prices, meter values, timestamps, status, notes | unique voucher; status `DRAFT/PENDING_AUTHORIZATION/AUTHORIZED/ISSUED/CANCELLED`; FKs and status/date indexes |
| `fuel_issue_history` | Issue lifecycle audit | `id UUID` | fuel_issue_id, statuses/action, actor_id/name, comment?, occurred_at | FKs issue/user; history index |
| `vendor` | Legacy fuel vendor | `id UUID` | code, name, contacts, active | unique code; active/name index |
| `fuel_price` | Vendor price period | `id UUID` | vendor_id, fuel_type, effective dates, unit_price, currency_code, active, timestamps | FK vendor; positive/period checks; lookup index |
| `fuel_purchase` | Fuel procurement record | `id UUID` | purchase/invoice/vendor/station IDs, fuel/date/quantity/price/tax/totals/currency, status/reconciliation, receipt/approval/audit fields | unique purchase and vendor/invoice; lifecycle checks; vendor/status/date/type indexes |
| `fuel_purchase_history` | Purchase audit | `id UUID` | purchase/status/action/actor, comment, variances, occurred_at | FKs purchase/user; history index |
| `bunker_tank` | Fuel tank master | `id UUID` | station, code/name, fuel type, capacity/current stock/reorder level, active, version, timestamps | station reference; capacity/stock checks; station/type indexes |
| `bunker_stock_movement` | Immutable tank movement | `id UUID` | tank, movement type, quantity, balance, reference type/id, actor, occurred/created, idempotency | tank reference; quantity/balance checks; unique idempotency; time/reference indexes |
| `bunker_dip_reading` | Physical stock reading | `id UUID` | tank, measured quantity/time, actor, notes, created | tank reference; nonnegative check; tank/time index |
| `bunker_stock_adjustment` | Reconciliation adjustment | `id UUID` | tank, quantity, reason/reference, actor, occurred/created | tank reference; nonzero check; tank/time index |

### Vehicle usage, maintenance, and driver compliance (V14–V23)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `vehicle_reading` | Immutable odometer/hour reading | `reading_id UUID` | vehicle, type, value/unit, epoch, source/reference, recorded/received, actor, correction fields, idempotency, notes, created | type/unit/source/value checks; self correction FK; unique idempotency/correction; chronology/source indexes |
| `vehicle_meter_reset` | Meter epoch reset | `id UUID` | vehicle/type, old/new epoch, last/new values, effective/recorded, actor, reason | epoch/value checks; vehicle reference; lookup index |
| `maintenance_schedule` | Legacy maintenance reservation | `id UUID` | vehicle, type/description, scheduled window, status, completion/audit fields | vehicle reference; date/status checks; vehicle-status/date indexes |
| `driver_exception` | Driver unavailability exception | `id UUID` | driver, type, start/end, status, reason, audit | driver reference; time/status checks; status/time indexes |
| `driver_violation` | Driver violation and payment state | `id UUID` | driver, type/date/location/description, points/fine, payment/status/dispute fields, audit | driver reference; value/status checks; driver/payment indexes |
| `driver_medical_record` | Medical fitness record | `id UUID` | driver, examination/valid dates, fitness status, provider/restrictions/document/audit | driver reference; date/status checks; driver/status indexes |
| `driver_drug_test` | Drug-test lifecycle | `id UUID` | driver, test type, scheduled/sample/result dates, result, laboratory/reference, return-to-duty and audit fields | driver reference; lifecycle/result checks; driver/result indexes |
| `lubricant_log` | Vehicle lubricant usage | `id UUID` | vehicle, fluid type/product, quantity/unit, vendor, cost/currency, meter values, recorded/actor/notes | positive checks; vehicle/vendor references; vehicle/type/vendor indexes |

### Trip operational event table (V24)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `trip_operational_event` | Checkpoint, delay, and incident timeline | `id UUID` | trip, event_type, occurred_at, actor, location/coordinates, delay/incident/checkpoint details, created_at | trip reference; type-specific checks; trip/time and type indexes |

### Notification and offline-sync tables (V25–V29, V58)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `notification_rule` | Event routing rule | `id UUID` | code/name/event_type/channel/recipient settings/template/enabled/audit/version | unique code; channel/recipient checks; event/template indexes |
| `notification` | Notification instance | `id UUID` | event/aggregate/rule/template, recipient/channel/content/status, delivery/read/schedule/escalation/audit | status/channel/escalation checks; parent/template FKs; recipient/event/time indexes |
| `notification_template` | Versioned message template | `id UUID` | code/version/event/channel/subject/body/required variables/active/audit | unique code/version; channel checks; lookup index |
| `notification_rule_policy` | Quiet hours/suppression/escalation policy | `rule_id UUID` | timezone, quiet settings, suppression window, escalation settings, audit/version | FK rule cascade; window/time checks |
| `notification_rule_quiet_day` | Quiet weekdays | `(rule_id,day_of_week)` | rule, weekday | FK policy; weekday check |
| `notification_rule_execution` | Rule decision audit | `id UUID` | execution/event/aggregate/rule/recipient/channel/outcome/suppression/control/failure/timestamps | unique execution key; FKs; channel/outcome checks; audit indexes |
| `notification_delivery_attempt` | Durable Email/SMS attempt | `id UUID` | notification, attempt number/state/due/start/end/error/provider/created | unique notification/attempt; attempt/state checks; due/history indexes |
| `customer_notification_preference` | Complete customer Email/SMS operational preference profile | `id UUID` | tenant, logical customer reference, channel flags, audit/version | unique and indexed `(tenant_id, customer_id)`; no Organization FK |
| `offline_sync_operation` | Idempotent server inbox | `operation_id UUID` | operation type/version, actor/client, aggregate, request hash, result/version, processed/created | actor FK; version/result checks; actor/aggregate indexes |

#### Table: `customer_notification_preference`

- **Purpose:** Stores the Notification-owned complete operational Email/SMS preference profile for one Customer.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Preference profile identifier |
| `tenant_id` | UUID | NO | - | UNIQUE with `customer_id`; indexed | Authoritative Tenant scope |
| `customer_id` | UUID | NO | - | Logical FK -> Organization `customer(id)`; no physical FK | Same-Tenant Customer reference validated through the public lookup |
| `email_enabled` | BOOLEAN | NO | - | - | Whether future operational Delivery Email is enabled |
| `sms_enabled` | BOOLEAN | NO | - | - | Whether future operational Delivery SMS is explicitly enabled |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last replacement timestamp |
| `version` | BIGINT | NO | `0` | Optimistic version | Concurrency-control version |

V58 also permits `SMS` in Notification channel constraints, permits `EVENT_CUSTOMER` in rule recipient types, expands `notification.recipient` to `VARCHAR(320)`, adds `(tenant_id, aggregate_type, aggregate_id, created_at DESC)` on `notification_rule_execution`, and seeds ten version-1 templates plus ten Tenant-scoped rules for the five frozen Delivery events.

### Customer self-service tables (V59)

#### Table: `delivery_self_service_access`

- **Purpose:** Stores Delivery-owned hash-only, action-scoped customer access credentials and their lifecycle facts.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` for composite references | Access-record identifier |
| `tenant_id` | UUID | NO | - | UNIQUE with `issuance_idempotency_key`; indexed | Authoritative Tenant scope |
| `delivery_order_id` | UUID | NO | - | Physical same-module FK with `tenant_id` -> `delivery_order(id, tenant_id)`; indexed | Delivery Order authorized by the token |
| `customer_id` | UUID | NO | - | Logical FK -> Organization `customer(id)`; indexed | Same-Tenant Customer reference |
| `recipient_contact_hash` | CHAR(64) | NO | - | Lowercase hex check | HMAC-SHA-256 fingerprint of normalized issuance contact |
| `contact_hash_key_version` | VARCHAR(32) | NO | - | - | External contact-HMAC key version |
| `token_hash` | CHAR(64) | NO | - | UNIQUE; lowercase hex check | SHA-256 hash of the opaque token bytes |
| `allowed_actions` | VARCHAR[] | NO | - | Cardinality greater than zero | Token action scopes |
| `issuance_idempotency_key` | VARCHAR(128) | NO | - | UNIQUE with `tenant_id` | Stable issuance retry key |
| `issued_at` | TIMESTAMPTZ | NO | - | `expires_at > issued_at` | Server issuance time |
| `expires_at` | TIMESTAMPTZ | NO | - | Active-expiry partial index | Server-enforced expiry time |
| `revoked_at` | TIMESTAMPTZ | YES | NULL | Revoked/expiry index | Revocation time |
| `last_used_at` | TIMESTAMPTZ | YES | NULL | - | Last successful use time |
| `use_count` | BIGINT | NO | `0` | Nonnegative check | Successful use count |
| `revocation_reason` | VARCHAR(64) | YES | NULL | - | Controlled revocation reason |
| `version` | BIGINT | NO | `0` | Optimistic version | Concurrency-control version |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update timestamp |
| `created_by` | VARCHAR(255) | NO | - | - | Controlled creator actor |
| `updated_by` | VARCHAR(255) | NO | - | - | Controlled last-updater actor |

Indexes: `(tenant_id, delivery_order_id, customer_id)`, active `(tenant_id, expires_at) WHERE revoked_at IS NULL`, and `(tenant_id, revoked_at, expires_at)`.

#### Table: `delivery_customer_submission`

- **Purpose:** Stores Delivery-owned customer preferences, redelivery requests, issues, and feedback without directly mutating operational scheduling.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Submission identifier |
| `tenant_id` | UUID | NO | - | Tenant-qualified indexes and idempotency | Authoritative Tenant scope |
| `delivery_order_id` | UUID | NO | - | Physical same-module FK with `tenant_id` -> `delivery_order(id, tenant_id)`; indexed | Related Delivery Order |
| `customer_id` | UUID | NO | - | Logical FK -> Organization `customer(id)`; indexed | Same-Tenant Customer reference |
| `access_id` | UUID | NO | - | Physical same-module FK with `tenant_id` -> `delivery_self_service_access(id, tenant_id)` | Authorizing access record |
| `submission_type` | VARCHAR(32) | NO | - | `DELIVERY_PREFERENCE`, `REDELIVERY_REQUEST`, `ISSUE`, or `FEEDBACK` | Submission discriminator |
| `category` | VARCHAR(64) | YES | NULL | Allow-listed issue category; required only for issues | Customer-safe issue category |
| `description` | VARCHAR(1000) | YES | NULL | Type-specific length/required-field check | Customer statement or preference note |
| `rating` | SMALLINT | YES | NULL | 1–5; feedback only | Customer feedback rating |
| `preferred_start_at` | TIMESTAMPTZ | YES | NULL | Paired with end; end must be later | Requested non-binding window start |
| `preferred_end_at` | TIMESTAMPTZ | YES | NULL | Paired with start; later than start | Requested non-binding window end |
| `status` | VARCHAR(32) | NO | `'SUBMITTED'` | `SUBMITTED`, `RECORDED`, `ACCEPTED`, `DECLINED`, or `SUPERSEDED` | Submission handling state |
| `idempotency_key` | VARCHAR(128) | NO | - | 16–128 chars; UNIQUE with Tenant/access/type | Client retry key |
| `request_hash` | CHAR(64) | NO | - | Lowercase hex check | Canonical request fingerprint |
| `operator_outcome` | VARCHAR(64) | YES | NULL | - | Later operator disposition |
| `operator_outcome_at` | TIMESTAMPTZ | YES | NULL | - | Operator disposition time |
| `operator_outcome_by` | VARCHAR(255) | YES | NULL | - | Operator disposition actor |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update timestamp |
| `created_by` | VARCHAR(255) | NO | - | - | Controlled creator actor |
| `updated_by` | VARCHAR(255) | NO | - | - | Controlled last-updater actor |
| `version` | BIGINT | NO | `0` | Optimistic version | Concurrency-control version |

Indexes: unique `(tenant_id, access_id, submission_type, idempotency_key)`; unique active feedback `(tenant_id, delivery_order_id, customer_id) WHERE submission_type = 'FEEDBACK' AND status <> 'SUPERSEDED'`; and `(tenant_id, delivery_order_id, customer_id, submission_type, created_at DESC)`.

V59 also appends only the controlled `[[SELF_SERVICE_LINK]]` placeholder to the five US-69 Delivery Email/SMS template families. The provider worker replaces it transiently at final send; raw tokens are not persisted in Notification records.

### Durable internal event table (V60)

#### Table: `integration_outbox_event`

- **Purpose:** Atomically records explicitly approved durable internal events for bounded Tenant-aware background delivery.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Indexed)
- **Owner:** Shared technical event boundary; business modules publish only through `DurableEventPublisher`.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Outbox row identity |
| `event_id` | UUID | NO | - | UNIQUE with Tenant and consumer; indexed with Tenant | Stable logical event identity reused across retries |
| `tenant_id` | UUID | NO | - | Tenant-qualified polling and retention indexes | Trusted Tenant processing scope |
| `consumer_name` | VARCHAR(100) | NO | - | UNIQUE with Tenant and event | Explicit registered durable consumer |
| `event_type` | VARCHAR(100) | NO | - | - | Stable event contract type |
| `event_version` | INTEGER | NO | - | CHECK `>= 1` | Explicit contract version |
| `aggregate_type` | VARCHAR(100) | NO | - | - | Owning aggregate type |
| `aggregate_id` | UUID | NO | - | Logical aggregate reference only | Owning aggregate identity |
| `payload` | JSONB | NO | - | Deterministic, allow-listed, maximum 32 KiB in application | Minimal consumer facts; never entities or secrets |
| `occurred_at` | TIMESTAMPTZ | NO | - | - | Source event occurrence time |
| `status` | VARCHAR(24) | NO | `'PENDING'` | CHECK in `PENDING`, `PROCESSING`, `RETRY`, `PUBLISHED`, `FAILED`, `UNSUPPORTED` | Delivery lifecycle |
| `attempt_count` | INTEGER | NO | `0` | CHECK from 0 through 5 | Bounded claim count |
| `next_attempt_at` | TIMESTAMPTZ | NO | - | Partial ready/retry index | Earliest next claim time |
| `locked_until` | TIMESTAMPTZ | YES | NULL | Partial processing index | Recoverable claim lease expiry |
| `published_at` | TIMESTAMPTZ | YES | NULL | - | Successful consumer acknowledgement time |
| `last_error_code` | VARCHAR(80) | YES | NULL | Sanitized code only | Retry/terminal diagnostic without payload/provider body |
| `created_at` | TIMESTAMPTZ | NO | - | - | Row creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | Partial terminal-status retention index | Last lifecycle update |
| `row_version` | BIGINT | NO | `0` | Optimistic version | Concurrent update guard |

Constraints/indexes: unique `(tenant_id,event_id,consumer_name)`; ready/retry `(tenant_id,status,next_attempt_at,occurred_at)`; stale claims `(tenant_id,status,locked_until)`; terminal retention `(tenant_id,status,updated_at)`; and event identity `(tenant_id,event_id)`. Claims are bounded to 50 per Tenant with a five-minute lease and at most five attempts. Published rows are retained at least 30 days and failed/unsupported rows at least 90 days; purge is not automatic and requires a separately approved Tenant-qualified action.

### Routing history and freight tables (V30–V42)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `route_revision` | Route snapshot | `id UUID` | route, revision, code/name/locations/distance/duration/active/change audit | FK route cascade; unique route/revision; positive revision; route index |
| `route_revision_stop` | Snapshot stops | `(route_revision_id,stop_order)` | revision, location, order | FK revision cascade |
| `route_disruption` | Route disruption | `id UUID` | route/type/severity/description/window/detour/status/create/resolve audit | route references; status `ACTIVE/RESOLVED`; severity/type/window checks; indexes |
| `freight_order` | Freight service request | `id UUID` | order number, customer, locations, pickup/delivery, service/priority/instructions/version/audit | unique number; references; location/window checks; customer/pickup indexes |
| `freight_order_line` | Freight order line | `id UUID` | order, description, quantity, line_order | FK order cascade; unique order/position; positive checks |
| `cargo_manifest` | Manifest aggregate | `id UUID` | number, freight order/id snapshot, version/audit/finalization | unique number; FK order; paired finalization check; indexes |
| `cargo_manifest_item` | Manifest cargo item | `id UUID` | manifest/order line, description/quantity/packing/classification/customs/hazardous/fragile/temperature-sensitive/measurements/order | FKs; unique item order; quantity/position/measurement checks; special-cargo and measurement fields nullable for legacy UNKNOWN state |
| `load_plan` | Vehicle loading plan | `id UUID` | number, manifest, vehicle, notes, version/readiness/audit | unique number; manifest/vehicle FKs and indexes; readiness status |
| `load_plan_item_placement` | Item placement | `id UUID` | plan, manifest item, placement/zone/stack/container/loading/special notes | FKs; unique item/order per plan; nonnegative checks |
| `freight_insurance_policy` | Cargo policy | `id UUID` | number, freight order/manifest, provider/type, coverage/premium/currency/validity/status/version/audit | unique number; order FK; positive/window business validation; order index |
| `freight_insurance_claim` | Insurance claim | `id UUID` | number, policy/order, incident/damage, claimed/assessed, assessor/status/resolution/version/audit | unique number; policy/order FKs; positive amount; indexes |
| `freight_insurance_settlement` | Claim settlement | `id UUID` | claim, reference, amount/currency/notes/settlement audit | FK claim cascade; positive amount; claim index |
| `cargo_exception` | Cargo exception record (US-30) | `id UUID` | exception_number varchar32 UNIQUE, exception_type varchar40 CHECK(6 types), status varchar20 DEFAULT 'OPEN' CHECK(OPEN/HELD/ESCALATED/RESOLVED/REJECTED), severity varchar20 DEFAULT 'MEDIUM' CHECK(LOW/MEDIUM/HIGH/CRITICAL), freight_order_id UUID NOT NULL FK freight_order, manifest_id UUID?, manifest_item_id UUID?, description varchar2000 NOT NULL, impact varchar2000?, restriction varchar1000?, corrective_action varchar2000?, resolution varchar2000?, resolved_at timestamptz?, resolved_by varchar128?, version bigint DEFAULT 0, created_at/updated_at timestamptz, created_by/updated_by varchar128 | Sequence cargo_exception_number_sequence; FK freight_order(id); CHECK type/status/severity; indexes on freight_order_id, manifest_id, exception_type, status, created_at DESC |
| `cargo_exception_history` | Exception lifecycle audit (US-30 AC3) | `id UUID` | exception_id UUID NOT NULL FK cargo_exception ON DELETE CASCADE, action varchar60 NOT NULL (HOLD_APPLIED/ESCALATED/RELEASED/REJECTED/RESOLVED), actor varchar128 NOT NULL, occurred_at timestamptz NOT NULL, reason varchar2000?, details varchar2000? | FK exception cascade; index on exception_id |

#### Table: `cargo_manifest_item`

- **Purpose:** Stores execution-grade cargo items owned by a Cargo Manifest, including structured customs, hazardous, fragile, temperature-sensitive classification, and physical cargo measurements (weight and dimensions).
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** Not present — legacy single-tenant schema; tenant remediation remains separately governed.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Unique manifest-item identifier |
| `cargo_manifest_id` | UUID | NO | - | Internal FK -> `cargo_manifest(id)` ON DELETE CASCADE | Owning Cargo Manifest |
| `freight_order_line_id` | UUID | NO | - | Internal FK -> `freight_order_line(id)` | Referenced Freight Order line |
| `description` | VARCHAR(500) | NO | - | - | Traceable cargo description |
| `quantity` | DECIMAL(19,4) | NO | - | CHECK (`quantity > 0`) | Manifested quantity |
| `packing_information` | VARCHAR(500) | NO | - | - | Supplier/operator packing description |
| `commodity_classification` | VARCHAR(120) | NO | - | Provider-neutral code validated by domain | Commodity classification |
| `customs_applicable` | BOOLEAN | NO | FALSE | - | Whether customs information applies |
| `customs_information` | VARCHAR(1000) | YES | NULL | Required by domain when customs applies | Customs details |
| `hazardous` | BOOLEAN | NO | FALSE | - | Whether hazardous-goods information applies |
| `hazardous_classification` | VARCHAR(120) | YES | NULL | Required by domain when hazardous | Provider-neutral hazardous classification |
| `hazardous_details` | VARCHAR(1000) | YES | NULL | Required by domain when hazardous | Hazardous handling details |
| `fragile` | BOOLEAN | YES | NULL | TRUE/FALSE/NULL tri-state | Manifest-owned fragile classification; NULL means UNKNOWN |
| `temperature_sensitive` | BOOLEAN | YES | NULL | TRUE/FALSE/NULL tri-state | Manifest-owned temperature-sensitive classification; NULL means UNKNOWN |
| `unit_weight` | DECIMAL(19,4) | YES | NULL | CHECK (`unit_weight IS NULL OR unit_weight > 0`) | Weight per unit; NULL means UNKNOWN |
| `weight_unit` | VARCHAR(16) | YES | NULL | CHECK (`weight_unit IS NULL OR weight_unit IN ('KG','G','TONNE')`) | Weight measurement unit |
| `length` | DECIMAL(19,4) | YES | NULL | CHECK (`length IS NULL OR length > 0`) | Cargo package length; NULL means UNKNOWN |
| `width` | DECIMAL(19,4) | YES | NULL | CHECK (`width IS NULL OR width > 0`) | Cargo package width; NULL means UNKNOWN |
| `height` | DECIMAL(19,4) | YES | NULL | CHECK (`height IS NULL OR height > 0`) | Cargo package height; NULL means UNKNOWN |
| `dimension_unit` | VARCHAR(16) | YES | NULL | CHECK (`dimension_unit IS NULL OR dimension_unit IN ('M','CM','MM')`) | Linear dimension unit |
| `item_order` | INTEGER | NO | - | UNIQUE (`cargo_manifest_id`, `item_order`); CHECK (`item_order >= 0`) | Stable item position within the manifest |

Indexes: `idx_manifest_item_parent (cargo_manifest_id)`. V42 adds `unit_weight`, `weight_unit`, `length`, `width`, `height`, and `dimension_unit` as nullable columns without backfill. Manifest validation continues to permit saving unmeasured items, while Load Planning weight/volume validation reports `INCOMPLETE` with missing fact diagnostics when measurements are absent.

## Migration Inventory

#### Table: `delivery_order`

- **Purpose:** Tenant-owned US-56 Delivery Order requirements and readiness state.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed through composite operational indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Delivery Order identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)`; UNIQUE with `delivery_number` | Authoritative tenant scope |
| `delivery_number` | VARCHAR(15) | NO | - | `DEL-YYYY-NNNNNN`; immutable; tenant-scoped unique | Server-generated order number |
| `customer_id` | UUID | NO | - | Logical FK -> Organization Customer | Customer reference validated through public port |
| `origin_location_id` | UUID | NO | - | Logical FK -> Organization Location; differs from destination | Origin reference |
| `destination_location_id` | UUID | NO | - | Logical FK -> Organization Location; differs from origin | Destination reference |
| `priority` | VARCHAR(20) | NO | `NORMAL` | CHECK LOW/NORMAL/HIGH/URGENT | Delivery urgency |
| `service_type` | VARCHAR(20) | NO | `STANDARD` | CHECK STANDARD/EXPRESS/SAME_DAY/SCHEDULED | Delivery service classification |
| `window_start` | TIMESTAMPTZ | NO | - | `window_start <= window_end` | Delivery-window start |
| `window_end` | TIMESTAMPTZ | NO | - | `window_start <= window_end` | Delivery-window end |
| `instructions` | TEXT | YES | NULL | - | Optional requirements |
| `status` | VARCHAR(40) | NO | `DRAFT` | CHECK DRAFT/READY_FOR_ASSIGNMENT | US-56 lifecycle state |
| `version` | BIGINT | NO | `0` | optimistic lock | Concurrent-update version |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update time |
| `created_by` | VARCHAR(128) | NO | - | - | Creating actor |
| `updated_by` | VARCHAR(128) | NO | - | - | Last modifying actor |

Indexes: `(tenant_id,status)`, `(tenant_id,customer_id)`, `(tenant_id,window_start,window_end)`. No assignment columns and no physical cross-module foreign keys exist.

#### Table: `delivery_number_counter`

- **Purpose:** Atomic tenant/year sequence allocation for immutable Delivery numbers.
- **Primary Key:** (`tenant_id`, `calendar_year`)
- **Multi-Tenant Key:** `tenant_id`

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | PRIMARY KEY component; logical FK -> `tenant(id)` | Tenant sequence scope |
| `calendar_year` | INTEGER | NO | - | PRIMARY KEY component; 1000–9999 | Tenant-local calendar year |
| `last_value` | INTEGER | NO | - | 1–999999 | Last permanently allocated number |

#### Table: `proof_of_delivery`

- **Purpose:** Tenant-owned US-57 Proof of Delivery aggregate tracking proof lifecycle, signer, and coordinates.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE (`id`, `tenant_id`) | Proof of Delivery identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)`; UNIQUE (`tenant_id`, `delivery_order_id`) | Authoritative tenant scope |
| `delivery_order_id` | UUID | NO | - | FK -> `delivery_order(id, tenant_id)` | Associated Delivery Order |
| `status` | VARCHAR(20) | NO | `DRAFT` | CHECK (`status IN ('DRAFT', 'FINALIZED')`) | POD lifecycle state |
| `device_captured_at` | TIMESTAMPTZ | YES | NULL | - | Optional client timestamp |
| `latitude` | NUMERIC(10,7) | YES | NULL | CHECK (`latitude BETWEEN -90 AND 90`) | Optional geo latitude |
| `longitude` | NUMERIC(10,7) | YES | NULL | CHECK (`longitude BETWEEN -180 AND 180`) | Optional geo longitude |
| `accuracy_meters` | NUMERIC(12,3) | YES | NULL | CHECK (`accuracy_meters > 0`) | Optional geo accuracy |
| `signer_name` | VARCHAR(200) | YES | NULL | Required when signature evidence present | Signer full name |
| `signer_relationship` | VARCHAR(100) | YES | NULL | - | Signer relationship to recipient |
| `accepted_at` | TIMESTAMPTZ | YES | NULL | Server UTC timestamp populated on finalization | Final acceptance timestamp |
| `accepted_by` | VARCHAR(128) | YES | NULL | Populated on finalization | Finalizing actor username |
| `version` | BIGINT | NO | `0` | optimistic lock | Concurrency version |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Update timestamp |
| `created_by` | VARCHAR(128) | NO | - | - | Creating actor |
| `updated_by` | VARCHAR(128) | NO | - | - | Modifying actor |

Indexes: `(tenant_id, status)`.

#### Table: `pod_evidence`

- **Purpose:** Stores binary/textual evidence items (Signature, Photo, Barcode) associated with a POD.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Evidence item identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)` | Authoritative tenant scope |
| `proof_of_delivery_id` | UUID | NO | - | FK -> `proof_of_delivery(id, tenant_id)` ON DELETE CASCADE | Parent POD reference |
| `evidence_type` | VARCHAR(20) | NO | - | CHECK (`evidence_type IN ('SIGNATURE', 'PHOTO', 'BARCODE')`) | Evidence type |
| `storage_reference` | VARCHAR(255) | YES | NULL | Required for SIGNATURE/PHOTO | Relative storage reference key |
| `barcode_value` | VARCHAR(64) | YES | NULL | Required for BARCODE (`DEL-YYYY-NNNNNN`) | Normalized barcode value |
| `detected_content_type` | VARCHAR(50) | YES | NULL | `image/png` or `image/jpeg` for files | MIME type |
| `content_length` | BIGINT | YES | NULL | > 0 for binary files | File size in bytes |
| `sha256_checksum` | VARCHAR(64) | YES | NULL | 64-char hex SHA-256 | Content checksum |
| `original_filename` | VARCHAR(255) | YES | NULL | - | Uploaded file name |
| `capture_source` | VARCHAR(20) | NO | - | CHECK (`capture_source IN ('CAMERA', 'FILE', 'SCANNER', 'MANUAL')`) | Capture source channel |
| `created_by` | VARCHAR(128) | NO | - | - | Uploading actor |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |

Indexes: `(tenant_id, proof_of_delivery_id)`, unique partial index for single signature per POD, unique partial index for single barcode per POD.

#### US-58 Offline Proof of Delivery Synchronization
- **Operation Type:** `DELIVERY_POD_OFFLINE_SYNC` (Aggregate: `DELIVERY`)
- **Queue / Storage:** Local client IndexedDB outbox queue (`features/offlineSync`), server `offline_sync_operation` (V29).
- **Composite Payload:** Signer metadata, consent record (`consentGiven: true`, `consentVersion: "POD-CONSENT-V1"`, timestamp), device coordinates, and ordered evidence list (`DeliveryPodOfflineEvidenceItem` with base64 PNG/JPEG or barcode value).
- **Execution Boundary:** `DeliveryPodOfflineOperationHandler` invokes `OfflineProofOfDeliveryRecorder` on `delivery` module.
- **Idempotency & Concurrency:** Server checks delivery state (`READY_FOR_ASSIGNMENT`), optimistic lock version, duplicate POD recording, and ensures atomic database finalization into `DELIVERED`.

#### Table: `delivery_attempt` (V48)

- **Purpose:** Stores sequential, immutable failure attempt records for a delivery order.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Attempt record identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)` | Authoritative tenant scope |
| `delivery_id` | UUID | NO | - | FK -> `delivery_order(id, tenant_id)` ON DELETE CASCADE | Delivery order reference |
| `attempt_number` | INTEGER | NO | - | CHECK (`attempt_number >= 1`) | Sequential attempt number (1, 2, 3...) |
| `attempt_timestamp` | TIMESTAMPTZ | NO | - | - | UTC timestamp of the attempt |
| `failure_reason` | VARCHAR(40) | NO | - | CHECK (`failure_reason IN (...)`) | Standardized failure reason enum |
| `notes` | VARCHAR(500) | YES | NULL | - | Operator attempt notes |
| `disposition` | VARCHAR(40) | NO | - | CHECK (`disposition IN (...)`) | Outcome disposition |
| `recorded_by` | VARCHAR(128) | NO | - | - | Recording operator |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |

Unique Constraints: `(tenant_id, delivery_id, attempt_number)`.
Indexes: `(tenant_id, delivery_id)`.

#### Table: `delivery_contact_attempt` (V48)

- **Purpose:** Stores contact communication attempts during or following a delivery attempt.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Contact attempt identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)` | Authoritative tenant scope |
| `delivery_attempt_id` | UUID | NO | - | FK -> `delivery_attempt(id, tenant_id)` ON DELETE CASCADE | Parent attempt reference |
| `channel` | VARCHAR(20) | NO | - | CHECK (`channel IN ('PHONE', 'SMS', 'WHATSAPP', 'EMAIL', 'IN_PERSON')`) | Interaction channel |
| `contact_timestamp` | TIMESTAMPTZ | NO | - | - | UTC timestamp of contact |
| `outcome` | VARCHAR(30) | NO | - | CHECK (`outcome IN ('ANSWERED', 'NO_ANSWER', 'BUSY', 'WRONG_NUMBER', 'MESSAGE_LEFT', 'REJECTED')`) | Contact outcome |
| `notes` | VARCHAR(500) | YES | NULL | - | Operator contact notes (no PII) |
| `recorded_by` | VARCHAR(128) | NO | - | - | Recording operator |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation timestamp |

Indexes: `(tenant_id, delivery_attempt_id)`.

#### Table: `delivery_escalation` (V48)

- **Purpose:** Stores operational and management escalations for failed deliveries.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Escalation record identifier |
| `tenant_id` | UUID | NO | - | Logical FK -> `tenant(id)` | Authoritative tenant scope |
| `delivery_id` | UUID | NO | - | FK -> `delivery_order(id, tenant_id)` ON DELETE CASCADE | Delivery order reference |
| `delivery_attempt_id` | UUID | YES | NULL | FK -> `delivery_attempt(id, tenant_id)` ON DELETE SET NULL | Associated attempt reference |
| `reason` | VARCHAR(500) | NO | - | - | Escalation explanation |
| `status` | VARCHAR(20) | NO | - | CHECK (`status IN ('OPEN', 'UNDER_REVIEW', 'RESOLVED')`) | Escalation status |
| `escalated_by` | VARCHAR(128) | NO | - | - | Operator who raised escalation |
| `escalated_at` | TIMESTAMPTZ | NO | - | - | Escalation timestamp |
| `resolution_notes` | VARCHAR(500) | YES | NULL | - | Manager resolution notes |
| `resolved_by` | VARCHAR(128) | YES | NULL | - | Manager who resolved escalation |
| `resolved_at` | TIMESTAMPTZ | YES | NULL | - | Resolution timestamp |

Indexes: `(tenant_id, delivery_id)`, `(tenant_id, status)`.

V1 baseline; V2 identity; V3 documents; V4 licences; V5 stops; V6–V8 trip audit/dispatch; V9 permissions; V10 integrity; V11–V12 fuel; V13 permissions; V14–V16 readings/reset; V17 permissions; V18 bunker; V19 maintenance; V20–V22 driver compliance; V23 lubricant; V24 operational events; V25–V28 notifications; V29 offline sync; V30 routing history; V31–V32 freight order/manifest; V33 permissions; V34 load plan; V35 permissions; V36 insurance; V37 Cargo Manifest special-cargo classification; V38 load plan readiness; V39 vehicle capacity master data; V40 cargo exception permissions; V41 cargo exception tables; V42 cargo manifest item measurements; V43 Tenant, membership, and canonical clean bootstrap; V44 operational tenant scoping and membership-role authority; V45 Freight reporting view/export permissions; V46 US-56 Delivery Orders, number counter and permissions; V47 US-57 Proof of Delivery, evidence, and POD permissions; V48 US-59 Failed Deliveries, attempts, contact attempts, escalations, and permissions; V49 US-60 Re-Delivery schedules, counter, and permissions; V50 US-61 Delivery Performance Analytics composite query indexes and DELIVERY_ANALYTICS_VIEW permission; V51–V56 Delivery exceptions, zones, slots, riders, batches, and rider transport mode; V57 Tenant-local operational business keys and idempotency constraints; V58 US-69 Delivery customer preferences, Email/SMS templates/rules, channel/recipient constraint extensions, and history index; V59 US-70 customer self-service access/submissions and Notification link placeholders; V60 P1-01 shared durable-event outbox.

US-56, US-57, US-59, US-60, and US-61 Delivery persistence and indexes are introduced by forward migrations V46, V47, V48, V49, and V50.

V51: US-62 Delivery Exception Management — `delivery_exception_case` table, `delivery_exception_evidence` table.
V52: US-63 Delivery Zone Management — `delivery_zone` table, `delivery_order.delivery_zone_id` column and FK, permissions `DELIVERY_ZONE_CREATE`, `DELIVERY_ZONE_VIEW`, `DELIVERY_ZONE_UPDATE`, `DELIVERY_ZONE_ACTIVATE`, `DELIVERY_ZONE_OVERRIDE`.
V53: US-64 Delivery Slot Management — `delivery_slot` table, `delivery_slot_reservation` table, `delivery_order.delivery_slot_id` column and FK, permissions `DELIVERY_SLOT_CREATE`, `DELIVERY_SLOT_VIEW`, `DELIVERY_SLOT_UPDATE`, `DELIVERY_SLOT_ACTIVATE`, `DELIVERY_SLOT_ASSIGN`, `DELIVERY_SLOT_OVERRIDE`.
V54: US-65 Delivery Rider Management — `delivery_rider`, `delivery_rider_zone`, `delivery_rider_shift`, `delivery_order_rider_assignment` tables, `delivery_order.current_rider_id` column, permissions `DELIVERY_RIDER_VIEW`, `DELIVERY_RIDER_CREATE`, `DELIVERY_RIDER_UPDATE`, `DELIVERY_RIDER_ACTIVATE`, `DELIVERY_RIDER_ASSIGN`, `DELIVERY_RIDER_OVERRIDE`.
V55: US-66 Delivery Batch Orders & Clustering — `delivery_batch`, `delivery_batch_order`, `delivery_batch_counter` tables, permissions `DELIVERY_BATCH_VIEW`, `DELIVERY_BATCH_CREATE`, `DELIVERY_BATCH_UPDATE`, `DELIVERY_BATCH_ASSIGN`, `DELIVERY_BATCH_DISPATCH`, `DELIVERY_BATCH_CANCEL`.

### US-65 Delivery Rider Management (Implemented)

#### Tables
- **`delivery_rider`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `rider_code` VARCHAR(64) NOT NULL, `driver_id` UUID NOT NULL, `rider_type` VARCHAR(32) NOT NULL, `primary_zone_id` UUID NOT NULL, `max_concurrent_deliveries` INT NOT NULL DEFAULT 3, `status` VARCHAR(32) NOT NULL DEFAULT 'ACTIVE', `version` BIGINT NOT NULL DEFAULT 0, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ, `created_by` VARCHAR(128), `updated_by` VARCHAR(128). Constraints: `uk_delivery_rider_code_tenant (tenant_id, rider_code)`, `uk_delivery_rider_id_tenant (id, tenant_id)`, `uk_active_driver_rider (tenant_id, driver_id) WHERE status = 'ACTIVE'`.
- **`delivery_rider_zone`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `rider_id` UUID NOT NULL, `zone_id` UUID NOT NULL, `zone_role` VARCHAR(32) NOT NULL DEFAULT 'SECONDARY'. Composite unique: `uk_rider_zone_unique (tenant_id, rider_id, zone_id)`.
- **`delivery_rider_shift`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `rider_id` UUID NOT NULL, `shift_date` DATE NOT NULL, `start_time` TIME NOT NULL, `end_time` TIME NOT NULL, `delivery_slot_id` UUID, `max_capacity` INT NOT NULL DEFAULT 5, `status` VARCHAR(32) NOT NULL DEFAULT 'SCHEDULED', `actual_start_time` TIMESTAMPTZ, `actual_end_time` TIMESTAMPTZ, `version` BIGINT NOT NULL DEFAULT 0.
- **`delivery_order_rider_assignment`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `delivery_order_id` UUID NOT NULL, `rider_id` UUID NOT NULL, `status` VARCHAR(32) NOT NULL DEFAULT 'ACTIVE', `is_override` BOOLEAN NOT NULL DEFAULT FALSE, `override_reason` TEXT, `assigned_at` TIMESTAMPTZ NOT NULL, `assigned_by` VARCHAR(128) NOT NULL, `unassigned_at` TIMESTAMPTZ, `unassigned_by` VARCHAR(128), `version` BIGINT NOT NULL DEFAULT 0. Unique partial index: `uk_active_delivery_order_rider (tenant_id, delivery_order_id) WHERE status = 'ACTIVE'`.

#### REST API Endpoints
- `POST /api/v1/deliveries/riders` — Onboard delivery rider (requires `DELIVERY_RIDER_CREATE`)
- `GET /api/v1/deliveries/riders` — List delivery riders (requires `DELIVERY_RIDER_VIEW`)
- `GET /api/v1/deliveries/riders/{id}` — Get rider details (requires `DELIVERY_RIDER_VIEW`)
- `PUT /api/v1/deliveries/riders/{id}` — Update rider profile (requires `DELIVERY_RIDER_UPDATE`)
- `PATCH /api/v1/deliveries/riders/{id}/activate` — Activate rider (requires `DELIVERY_RIDER_ACTIVATE`)
- `PATCH /api/v1/deliveries/riders/{id}/deactivate` — Deactivate rider (requires `DELIVERY_RIDER_ACTIVATE`)
- `PATCH /api/v1/deliveries/riders/{id}/suspend` — Suspend rider (requires `DELIVERY_RIDER_ACTIVATE`)
- `POST /api/v1/deliveries/riders/{id}/shifts` — Schedule rider shift (requires `DELIVERY_RIDER_UPDATE`)
- `GET /api/v1/deliveries/riders/{id}/shifts` — List rider shifts (requires `DELIVERY_RIDER_VIEW`)
- `PATCH /api/v1/deliveries/riders/{id}/shifts/{shiftId}/start` — Start shift (requires `DELIVERY_RIDER_UPDATE`)
- `PATCH /api/v1/deliveries/riders/{id}/shifts/{shiftId}/end` — End shift (requires `DELIVERY_RIDER_UPDATE`)
- `PATCH /api/v1/deliveries/riders/{id}/shifts/{shiftId}/cancel` — Cancel shift (requires `DELIVERY_RIDER_UPDATE`)
- `GET /api/v1/deliveries/riders/available` — Query available riders for zone/slot/date (requires `DELIVERY_RIDER_VIEW`)
- `POST /api/v1/deliveries/orders/{id}/assign-rider` — Assign rider to order (requires `DELIVERY_RIDER_ASSIGN` or `DELIVERY_RIDER_OVERRIDE`)
- `POST /api/v1/deliveries/orders/{id}/reassign-rider` — Reassign rider for order (requires `DELIVERY_RIDER_ASSIGN` or `DELIVERY_RIDER_OVERRIDE`)
- `POST /api/v1/deliveries/orders/{id}/unassign-rider` — Unassign active rider (requires `DELIVERY_RIDER_ASSIGN`)
- `GET /api/v1/deliveries/orders/{id}/rider-assignments` — Get order rider assignment history (requires `DELIVERY_RIDER_VIEW`)

### US-66 Batch Delivery Orders & Clustering (Implemented)

#### Tables
- **`delivery_batch_counter`**: `tenant_id` UUID NOT NULL, `calendar_year` INT NOT NULL, `current_val` BIGINT NOT NULL DEFAULT 0, PK `(tenant_id, calendar_year)`. Monotonic format `BAT-YYYY-NNNNNN`.
- **`delivery_batch`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `batch_code` VARCHAR(64) NOT NULL, `delivery_zone_id` UUID NOT NULL, `delivery_slot_id` UUID, `rider_id` UUID, `status` VARCHAR(32) NOT NULL DEFAULT 'DRAFT', `max_batch_size` INT NOT NULL DEFAULT 5, `active_order_count` INT NOT NULL DEFAULT 0, `total_order_count` INT NOT NULL DEFAULT 0, `version` BIGINT NOT NULL DEFAULT 0, `created_at` TIMESTAMPTZ NOT NULL, `updated_at` TIMESTAMPTZ NOT NULL, `created_by` VARCHAR(128) NOT NULL, `updated_by` VARCHAR(128) NOT NULL. Constraints: `uk_delivery_batch_code_tenant (tenant_id, batch_code)`, `uk_delivery_batch_id_tenant (id, tenant_id)`, `fk_delivery_batch_zone_tenant (delivery_zone_id, tenant_id) REFERENCES delivery_zone(id, tenant_id)`.
- **`delivery_batch_order`**: `id` UUID PK, `tenant_id` UUID NOT NULL, `batch_id` UUID NOT NULL, `delivery_order_id` UUID NOT NULL, `sequence_hint` INT, `status` VARCHAR(32) NOT NULL DEFAULT 'ACTIVE', `added_at` TIMESTAMPTZ NOT NULL, `added_by` VARCHAR(128) NOT NULL, `removed_at` TIMESTAMPTZ, `removed_by` VARCHAR(128), `version` BIGINT NOT NULL DEFAULT 0. Constraints: `uk_active_batch_order (tenant_id, delivery_order_id) WHERE status = 'ACTIVE'`, `fk_delivery_batch_order_batch_tenant (batch_id, tenant_id) REFERENCES delivery_batch(id, tenant_id)`, `fk_delivery_batch_order_order_tenant (delivery_order_id, tenant_id) REFERENCES delivery_order(id, tenant_id)`.

#### REST API Endpoints
- `GET /api/v1/deliveries/batches` — List/filter delivery batches (requires `DELIVERY_BATCH_VIEW`)
- `GET /api/v1/deliveries/batches/{id}` — Get batch details (requires `DELIVERY_BATCH_VIEW`)
- `GET /api/v1/deliveries/batches/{id}/orders` — Get contained batch order memberships (requires `DELIVERY_BATCH_VIEW`)
- `POST /api/v1/deliveries/batches` — Create manual delivery batch (requires `DELIVERY_BATCH_CREATE`)
- `POST /api/v1/deliveries/batches/auto-cluster` — Auto-cluster unbatched orders into batches (requires `DELIVERY_BATCH_CREATE`)
- `PUT /api/v1/deliveries/batches/{id}` — Update batch configuration (requires `DELIVERY_BATCH_UPDATE`)
- `POST /api/v1/deliveries/batches/{id}/orders` — Add orders to batch (requires `DELIVERY_BATCH_UPDATE`)
- `DELETE /api/v1/deliveries/batches/{id}/orders/{deliveryOrderId}` — Remove order from batch (requires `DELIVERY_BATCH_UPDATE`)
- `POST /api/v1/deliveries/batches/{id}/ready` — Mark batch ready (requires `DELIVERY_BATCH_UPDATE`)
- `POST /api/v1/deliveries/batches/{id}/assign-rider` — Assign rider to batch (requires `DELIVERY_BATCH_ASSIGN`)
- `POST /api/v1/deliveries/batches/{id}/dispatch` — Dispatch batch (requires `DELIVERY_BATCH_DISPATCH`)
- `POST /api/v1/deliveries/batches/{id}/complete` — Complete batch (requires `DELIVERY_BATCH_UPDATE`)
- `POST /api/v1/deliveries/batches/{id}/cancel` — Cancel batch and unbatch active memberships (requires `DELIVERY_BATCH_CANCEL`)

### US-67 Calculate Last-Mile ETA (Complete)

#### Architecture & Contract
- **Domain Owner**: `com.transportlogistics.app.delivery` (orchestrates order/batch ETA calculation and SLA status projection).
- **Routing Boundary**: Consumes domain-neutral `LastMileRoutingPort` interface. Route module line-haul algorithms (`US-18`/`US-20`) remain decoupled.
- **Computation Model**: Operational computed projection cached in-memory with 15-minute freshness TTL. US-67 owns Flyway `V56__delivery_rider_transport_mode_us67.sql`; the current repository/runtime head is the later tenant-local-key migration `V57`.
- **Calculation Formulation**: Cumulative transit time across stops factoring transport mode speed (`BICYCLE`, `MOTORBIKE`, `VAN`, `CAR`, `WALKER`), delivery zone classification (`URBAN_DENSE`, `SUBURBAN`, `RURAL`, `SPECIAL_SECURITY`), and fixed doorstep delivery service buffers.
- **SLA Risk Evaluation**: Projects delivery arrival against `DeliverySlot` time-window to classify `ON_TIME`, `AT_RISK`, or `LATE`.
- **Rider Context and Cache**: `RiderEtaContextPort` supplies the Rider's persisted transport mode. Cache keys include tenant identity, stale writes are rejected by per-subject generations, and Rider mode, assignment, batch membership, dispatch, and destination changes invalidate affected order/batch entries.
- **REST Endpoints (Active)**:
  - `GET /api/v1/deliveries/orders/{orderId}/eta` — Get single order ETA projection (requires `DELIVERY_VIEW`)
  - `POST /api/v1/deliveries/orders/{orderId}/eta/calculate` — Force recalculation of single order ETA (requires `DELIVERY_UPDATE`; literal `/api/v1` matcher and method security both enforced)
  - `GET /api/v1/deliveries/batches/{batchId}/eta` — Get multi-stop batch cumulative ETA projection (requires `DELIVERY_BATCH_VIEW`)
  - `POST /api/v1/deliveries/batches/{batchId}/eta/calculate` — Force recalculation of batch ETA (requires `DELIVERY_BATCH_UPDATE`)
- **Final Acceptance**: `MVP-1.4-US67-LAST-MILE-ETA-FINAL-ACCEPTANCE-001-RERUN` accepted the feature. Full Maven verify: 1,195 tests with 0 failures/errors and 15 skips; architecture 40/40; real PostgreSQL-backed Chromium ETA acceptance 6/6. Tenant-B IDOR returns 404, tenant spoofing is ineffective, and a `DELIVERY_VIEW`-only user receives 403 on direct single-order calculate.

### US-68 Handle Last-Mile Exceptions (Complete)

- **Owner and Boundary**: Delivery-owned Last-Mile Planner orchestration, not a new generic exception aggregate. It routes rider no-show/replacement to Rider and Batch assignment, multiple attempts/access/payment to US-59, wrong-address investigation to US-62 plus Location, contactless delivery to US-57 POD, and scheduling to US-60.
- **No New Persistence or API Family**: Existing `delivery_attempt`, `delivery_escalation`, and `delivery_exception_case` remain authoritative. No `LastMileException` table, new permissions, or `/last-mile-exceptions` API is approved. A future durable-fact proposal must re-evaluate the then-current Flyway head before choosing a migration number.
- **Operational Rules**: Reporting alone does not change DeliveryOrder status, batch membership/sequence, slot/zone master data, or ETA. Actual failed attempts use US-59 lifecycle/disposition; real Rider/batch/destination mutations use existing US-67 invalidation. No route optimisation, GPS/telematics, customer notification (US-69), or customer self-service (US-70) is in scope.
- **Security**: Tenant and actor are server-derived; existing method-level permissions apply. Raw access codes, PINs, OTPs, credentials, copied contact data, and live location are prohibited. The workflow is online-only and requires normal tenant/IDOR/version/after-commit controls.
- **Implemented Planner Projection**: `GET /api/v1/deliveries/{id}/last-mile-planner` composes the tenant-scoped Delivery Order, failed-attempt history, escalation history, and US-62 exception cases into a read-only context with available links/actions. It accepts `DELIVERY_FAIL_VIEW` or `DELIVERY_EXCEPTION_VIEW`; it creates no attempt/case, schedules no redelivery, changes no Rider/Batch, calculates no ETA, sends no notification, and persists no US-68 state.
- **Final Acceptance**: `MVP-1.4-US68-LAST-MILE-EXCEPTIONS-FINAL-ACCEPTANCE-001` accepted the read-only Planner. Full Maven verification passed 1,200 tests with 0 failures/errors and 15 skips; architecture passed 45/45; the real PostgreSQL-backed US-68 Chromium suite passed 3/3 and retained US-67 at 6/6. Flyway remains V57 and US-68 has no migration.

### US-69 Receive Delivery Notifications (Complete)

- **Requirement and boundary**: Customers/recipients receive operational delivery status, ETA-risk, completion, failure, and redelivery notices. Delivery owns the committed facts; Notification owns US-77 rule evaluation, versioned templates, recipient resolution, delivery attempts, provider abstraction, retry, status, and diagnostics. US-68 Planner never sends. US-69 creates no second rules engine.
- **Implemented events**: Version-1 `DELIVERY_OUT_FOR_DELIVERY`, `DELIVERY_ETA_RISK_CHANGED`, `DELIVERY_COMPLETED`, `DELIVERY_FAILED_ATTEMPT_RECORDED`, and `DELIVERY_REDELIVERY_SCHEDULED` use the standard event identity/Tenant/time/version envelope, aggregate type `DELIVERY_ORDER`, and exact allow-listed payload fields including the server-derived actor. They contain no phone, email, free-text notes, OTP, access code, coordinates, provider data, or whole aggregate. Publication is local after commit; the current handoff is not crash-durable.
- **ETA rule**: Notification consumes a Delivery-owned order ETA risk fact only when the new US-67 SLA state is `AT_RISK` or `LATE` and the prior current cache state is absent/different. `slaStatus` is the suppression milestone with a 1,440-minute window. Notification never calculates or polls ETA.
- **Recipient and consent**: Organization remains canonical owner of Customer phone/email and exposes a new tenant-aware `CustomerNotificationContactLookup`; Notification stores only logical `customer_id` preferences plus normalized destination snapshots on accepted notification/execution records. With no profile, operational Email is enabled when valid and SMS is disabled; once a profile exists, only explicitly enabled channels are eligible. No marketing behavior is approved.
- **Channels/providers**: Email reuses the existing SMTP/provider-neutral port. SMS is added behind a provider-neutral Notification port with a deterministic/local MVP adapter that never logs destination/body; no SMS vendor SDK is approved. Customer IN_APP/push is deferred until US-70 supplies an authenticated customer-user relationship. WhatsApp, callbacks/receipts, manual send/resend, and real-provider selection are deferred.
- **Templates and privacy**: Existing global, versioned, channel-specific Notification templates remain authoritative; tenant rules select compatible templates. Allowed variables are limited to delivery number/status, customer display name, ETA/SLA, completion time, failure disposition, and redelivery window. No tenant template override/localization framework, secret, raw internal UUID, copied customer aggregate, or provider payload is approved.
- **Reliability**: P1-01 upgrades the Delivery-to-Notification handoff to `AT_LEAST_ONCE` through the shared V60 outbox. Notification's existing Tenant-qualified execution key deduplicates replay; once persisted, Email/SMS retain the existing three-attempt provider retry schedule (1 minute, then 2 minutes). `SENT` means provider/local-adapter acceptance, not device delivery. Delivery state never changes because Notification failed.
- **Persistence**: `V58__delivery_notifications_us69.sql` creates Notification-owned `customer_notification_preference(id, tenant_id, customer_id, email_enabled, sms_enabled, created_at, updated_at, version)`, unique `(tenant_id, customer_id)`, plus SMS/`EVENT_CUSTOMER` constraint extensions, recipient width alignment to 320, and a `(tenant_id, aggregate_type, aggregate_id, created_at DESC)` execution index. `customer_id` is a logical Organization reference with no cross-module physical FK. Historical migrations remain immutable.
- **Security/API/UX**: Existing `NOTIFICATION_RULE_VIEW` protects tenant-scoped history/preference reads and `NOTIFICATION_RULE_MANAGE` protects preference writes. Notification owns the aggregate-filtered delivery diagnostics and preference APIs; there is no Delivery notification CRUD/send endpoint. The only new operator UX is a masked read-only Notifications timeline on Delivery Order details. US-70 retains customer portal/login/preference UI and secure tracking-link ownership.
- **Trigger remediation**: `DeliveryBatchService.markReady` now commits Batch status `READY` without publishing `DELIVERY_OUT_FOR_DELIVERY`. `DeliveryBatchService.dispatchBatch` publishes the unchanged customer event after saving `DISPATCHED`, exactly once per active Batch member. Deterministic tests prove READY=0, two active plus one removed produces two events, and rollback produces zero after-commit events.
- **Technical verification**: US-69 acceptance passed Maven 1,223/0/0/15, architecture 42/42, all static/frontend gates, and real PostgreSQL-backed Chromium 7/7. Its own migration remains V58; the current repository head is V60 after P1-01.
- **Final acceptance**: `MVP-1.4-US69-DELIVERY-NOTIFICATIONS-FINAL-ACCEPTANCE-001-RERUN` accepted the remediated trigger and all frozen Notification, security, privacy, Tenant, persistence, and UI contracts. Focused tests passed 20/20; Delivery/Notification/PostgreSQL regressions passed 193/193; full Maven passed 1,223/0/0/15; architecture passed 42/42; and real Chromium passed 7/7.
- **Program state**: US-69 and US-70 are `COMPLETE`. MVP 1.4 is 8/8 and `CLOSED`; overall is 65/87 and deferred is 22/87. P1-01 is complete and the authoritative next queue item is `DEFERRED-BACKLOG-REPRIORITIZATION-001`.

### US-70 Customer Self-Service (Implemented and Accepted)

- **Access and identity:** The sole MVP model is a 256-bit opaque Delivery access token transported in an HTTPS link fragment and sent as `Authorization: DeliveryAccess <token>`. Delivery stores only a SHA-256 hash. The token binds server-derived Tenant, Delivery, Organization Customer/contact fingerprint, expiry, and allow-listed actions. Current source has no Customer/Recipient-to-`app_user` relationship; US-70 adds none and does not reuse operator JWT/RBAC.
- **Scope:** A customer-safe projection exposes public delivery number, friendly status, Tenant-timezone window, US-67 ETA/freshness, available actions, masked destination summary, POD availability only, and effective masked Email/SMS preferences. Customers may replace the existing Notification preference profile through a new Notification-root published contract, submit a categorized issue, submit one post-delivery rating/comment, and submit a non-binding redelivery or pre-delivery preference-change request. Delivery does not import Notification's internal application use case.
- **Scheduling boundary:** A customer request never changes Delivery status/window, reserves capacity, or creates/supersedes a redelivery schedule. US-60 remains the only redelivery scheduling authority and US-64 remains the slot/cutoff/capacity authority. A committed operator schedule continues to produce the existing US-69 notification event after commit.
- **Privacy:** No internal IDs/enums, full address, Rider identity/contact/location, batch/zone/slot internals, ETA heuristic/provider/cache facts, internal exception investigation, Notification body/provider diagnostics, or POD signature/photo/barcode/geotag/content is exposed.
- **Notification integration:** US-69 events remain unchanged. Notification may persist a controlled link placeholder only. Immediately before provider delivery, it calls a published Delivery link-issuance port and substitutes the raw link transiently; raw tokens are never persisted in application Notification/event/audit/log data. Customer IN_APP/push and OTP remain deferred.
- **Public API:** Delivery owns `GET /api/public/v1/delivery-self-service`, `GET/PUT /api/public/v1/delivery-self-service/notification-preferences`, and `POST` routes for `/issues`, `/feedback`, and `/redelivery-requests`. Routes contain no target identifiers and authorize only through the scoped token. Invalid/expired/revoked/mismatched/cross-Tenant access is an indistinguishable 404.
- **Persistence:** Delivery owns `delivery_self_service_access` (hash-only credential, Tenant/Delivery/Customer/contact/action/lifetime/usage/version/audit facts) and `delivery_customer_submission` (typed preference/redelivery/issue/feedback submissions with idempotency/concurrency/audit facts and a composite Tenant-consistent access foreign key). Customer remains a logical Organization reference with no cross-module FK. V59 remains the US-70 migration; the current repository head is V60.
- **Frontend:** A mobile-friendly `/track` route sits outside `ProtectedRoute` and `AppLayout`; the fragment is consumed once, immediately removed, and retained in memory only. The operator shell, offline storage, customer analytics, and native app are excluded.
- **Final acceptance:** `MVP-1.4-US70-CUSTOMER-SELF-SERVICE-FINAL-ACCEPTANCE-001` accepted the opaque-token, Tenant/IDOR, projection privacy, Notification preference/link, issue, feedback, non-binding request, browser-memory, and US-60/64/67/69 boundaries. Full Maven passed 1,238 tests with 0 failures/errors and 15 skips in 4:56 against only `transport_logistics_acceptance`; focused backend/security passed 28/28; PostgreSQL passed 4/4 with clean V1→V59; architecture passed 42/42; Checkstyle, PMD, and SpotBugs passed; frontend TypeScript, 59-file/259-test Vitest, production build, and changed-file lint passed; real Chromium passed 9/9. Global lint retains 71 unrelated pre-existing errors and US-70 introduces none.
- **Program state:** US-70 is `COMPLETE`. MVP 1.4 is 8/8 `COMPLETE — CLOSED`; overall 65/87, deferred 22/87. P1-01 is complete; the next authoritative queue item is `DEFERRED-BACKLOG-REPRIORITIZATION-001`. No US-88/89/90 is defined.

## Remaining Suite Integration Work

### Governed full-product completion plan

`DEFERRED-BACKLOG-REPRIORITIZATION-001` is complete as a planning task. Following US-37 final acceptance, story accounting is 68 / 87 accepted and 19 / 87 remaining. The exact remainder is US-35, US-38, US-46, US-47, US-48 through US-55, US-72, US-76, US-82, and US-84 through US-87. US-88, US-89, and US-90 remain undefined.

The remaining stories are reprioritized into governed waves: Wave A (US-73/78 integration and exception foundations) is 2/2 COMPLETE and CLOSED; Wave B is open with US-37 complete, US-35 decisions frozen, and US-35 implementation plus US-38 Fuel and US-46/47 financial links remaining; Wave C covers US-48..55 GPS/tracking; Wave D covers US-72/76 compliance/mobile; and Wave E covers US-85/84/87/82/86 integrity, resilience, risk, analytics, and disruption. The single next task is `US-35-FUEL-CARDS-IMPLEMENTATION-001`.

The authoritative DOCX/UML titles govern over stale roadmap aliases: US-35 Manage Fuel Cards, US-37 Analyze Fuel Performance, US-38 Handle Fuel Exceptions, US-46 Process Driver Payroll Link, US-47 Manage Transport Billing, and US-48..55 Track Vehicles Live / Manage Geofences / Monitor Speed / Monitor Idle Time / Monitor Route Deviations / Replay Journeys / View Tracking Dashboard / Handle GPS Edge Cases.

After 87/87, `FULL-SOURCE-PARITY-AUDIT-001` must compare the mind map, DOCX, all UML, implementation, and accepted contracts without silently reopening accepted stories or inventing IDs. Full Maintenance Management beyond US-07 linkage, Workshop, Work Orders, Job Cards, Parts Inventory, and Inspection Management remain `OUTSIDE_CURRENT_87_STORY_REGISTER` unless separately authorized. Only after parity disposition should `FULL-PLATFORM-END-TO-END-ACCEPTANCE-001` run.

1. Apply the P1-01 canonical Tenant/version/aggregate envelope whenever a new consumed cross-module contract is approved; do not modernize unused events without a real consumer.
2. Resolve driver, customer, project, vendor, maintenance, billing, tracking, integration, compliance, and operational-exception ownership through story-scoped ADR/product decisions.
3. Replace cross-boundary physical references with logical IDs/contracts as modules become independent.

### US-73 Accepted Integration Platform Boundary

- Owner: dedicated top-level `integration` context; status `COMPLETE` after independent final acceptance.
- Only accepted implementation target: Tenant-owned `FILE_EXCHANGE` / `FILE_JSON_V1` / `OUTBOUND`, evidenced by real isolated filesystem I/O at `CONTROLLED_SANDBOX` tier.
- Integration configuration, immutable declarative mappings, exchange/attempt state, health, credential references, and Integration audit are owned by Integration. No raw secret, operator-controlled path/URL, arbitrary script, foreign repository/table, or cross-module SQL is allowed.
- P1-01 remains the sole durable event outbox. The frozen handler is `integration-outbound-exchange`; no second outbox/inbox, broker, exactly-once, or global ordering is approved.
- V61 creates the Integration-owned `integration_configuration`, `integration_mapping`, `integration_exchange`, `integration_exchange_attempt`, and `integration_audit_event` tables, all Tenant-owned with Tenant-consistent same-module constraints and Tenant-leading indexes. The exact schema is documented in `integration_module.md`; the current Flyway head is V61.
- ERP, Accounting, CRM, HRMS, fuel-vendor, telematics, payment, insurance, DMS, API, webhook, inbound, and bidirectional capabilities remain future consumers requiring separate contract/security/acceptance decisions.
- Final acceptance passed all three source criteria with focused Integration 24/24, P1-01 plus US-69/70 regressions 40/40, Maven 1,276/0/0 with 15 skips in 05:04, architecture 44/44, static/frontend gates, and real PostgreSQL/filesystem Chromium 6/6. At that acceptance checkpoint, program accounting was 66/87 accepted and 21/87 remaining; the current post-US-78 state is recorded below.

### US-78 Accepted Operational Exception Boundary

- Status: `COMPLETE / FINAL_ACCEPTANCE_PASS`; accounting is 67/87 accepted and 20/87 remaining.
- Owner/model: a dedicated top-level `operations` context with a hybrid lifecycle aggregate and logical source references. Detecting domains retain source meaning/state/evidence/correction; Operations owns classification, severity, assignment, SLA, escalation, corrective action, RCA, resolution/closure/reopen, and immutable case history.
- Active producers: Routing disruption creation (actual current US-22 owner) and Delivery exception creation (US-62) now publish atomically through their existing source transactions. The stale planning aliases “US-15 Trip Exceptions” and “US-23 Route Disruptions” do not change accepted Trip US-13/15 or Routing US-22/23 scope. US-68 remains read-only.
- Intake: `OperationalExceptionFactV1` through P1-01's shared durable outbox, at-least-once, deduplicated by Tenant/source event in the Operations case table. There is no second outbox/inbox/broker and no manual-create API.
- Lifecycle/security: `OPEN -> ACKNOWLEDGED -> IN_PROGRESS -> RESOLVED -> CLOSED`, reasoned reject/reopen, four severities, seven triage categories, role/user assignment, server SLA, idempotent escalation, seven narrow permissions, high/critical SoD, append-only history, and Tenant-safe not-found.
- Boundaries: Notification/US-77 receives only a safe durable escalation fact; US-81 supplies the accepted Tenant-aware scheduling pattern; US-80 remains the workflow boundary; US-83 owns document content. Operations stores minimized facts/references only and exposes no customer portal.
- Persistence/API/UI: V62 creates five normalized Operations-owned Tenant table families. Seventeen `/api/v1/operational-exceptions` list/detail/history and explicit command routes are implemented, and the operator queue is active at `/operations/exceptions` under `AppLayout`. The exact data dictionary is maintained in `operations.md`.
- Future: US-38 Fuel, US-55 Tracking, and US-86 disruption coordination use the frozen intake/boundary only after their own decisions. US-78 does not implement their semantics.
- Independent final acceptance: focused 41/41, concurrency 6/6, PostgreSQL V1-V62 3/3 against only `transport_logistics_acceptance`, regressions 84/84, Maven 1,296/0/0 with 15 skips in 05:06, architecture 46/46, Chromium 6/6 in 19.3 seconds, and static/frontend gates pass. The 71 global ESLint errors remain pre-existing in unchanged Delivery files; US-78 scoped lint is clean.
- Wave A is 2/2 COMPLETE and CLOSED. This was the US-78 acceptance checkpoint; current accounting and queue are recorded in the US-37 section below.

### US-37 Fuel Performance Implementation

- Status: `COMPLETE / FINAL_ACCEPTANCE_PASS`; accounting is 68/87 accepted and 19/87 remaining. Wave B remains open.
- Owner/source: Fuel owns metric calculation and interpretation over `ISSUED` US-31 facts, accepted price/cost facts from US-32/34, US-36 bunker facts where actually referenced, and validated usage/attribution projections from Fleet/Trip. Reporting may consume only `FuelPerformanceQuery`; no foreign repository/entity/table/SQL is allowed.
- Metrics: separate `DISTANCE` (`L/km`, `L/100km`, `km/L`, cost/km) and `ENGINE_HOURS` (`L/hour`, cost/hour) modes plus quantities, cost/trip, samples, historical variance and trends. Rated/manufacturer efficiency is unsupported. `BigDecimal`, explicit units/currency, no FX, and no zero/missing denominator inference are mandatory.
- Quality/windows: fixed `COMPLETE`, `PARTIAL`, `INSUFFICIENT`, and `INVALID_SOURCE_DATA`; default 30 days, presets 7/30/90 and custom maximum 365 Tenant-calendar days; daily/weekly/monthly grain. Source exclusions are reasoned and never delete or mutate raw facts.
- Baseline/comparison: same-Vehicle immediately preceding equal window, same fuel/mode, minimum three valid issued samples. Peers require same Tenant, Vehicle type, fuel type and mode with at least three eligible Vehicles. Driver attribution requires authoritative Driver and matching Trip Vehicle/Driver context. No punitive or ordinal leaderboard is approved.
- Anomaly: transparent consumption-intensity variance. Adverse variance ≥20% is `EFFICIENCY_DEVIATION / REVIEW_REQUIRED`; possible leakage requires two consecutive valid buckets each ≥30%. Language is non-accusatory. No ML, theft/fraud conclusion, automatic US-38/US-78 case, discipline, blocking, correction, or event is approved.
- Read model/security: bounded on-demand Fuel query with no projection table/cache/event/P1-01/scheduler. Six read-only `/api/v1/fuel/performance` routes and `FUEL_PERFORMANCE_VIEW` are implemented; V63 seeds only the permission and creates no analytics table. Driver results are minimal internal operational-review data.
- UI/acceptance: `/fuel/performance` under existing `AppLayout`, with summary, trend, compatible Vehicle/Driver comparisons, quality and safe review indicators. PostgreSQL acceptance uses only `transport_logistics_acceptance`; real browser evidence must prove metrics, anomaly/insufficient states, Tenant denial and exact raw-source immutability.
- US-35 Fuel Cards, US-38 exception operations, US-46 payroll, US-47 billing, US-82 predictive analytics, export and configurable thresholds remain outside US-37.
- Technical closure: PASS. PostgreSQL V1–V63 ran against only `transport_logistics_acceptance`; focused analytics/security 14/14, Maven 1,310/0/0/15 in 05:14, architecture 46/46, Vitest 262/262, and Chromium 6/6 in 23.1 seconds all pass. The browser fixture proves 101 bounded Vehicle comparison rows and paginated exact source immutability. Empty trend buckets are explicit `INSUFFICIENT` null gaps, threshold decisions use unrounded `BigDecimal` precision, and the source window fails closed above 100,000 rows. Global ESLint's 71 unchanged Delivery errors remain pre-existing; US-37 introduced errors are zero.
- Independent final acceptance: PASS. Fresh focused analytics/security 14/14, complete Maven 1,310/0/0/15 in 05:15, architecture 46/46, Vitest 262/262, and real PostgreSQL-backed Chromium 6/6 in 21.7 seconds all pass against only `transport_logistics_acceptance`; Flyway reached V63 and exact source immutability passed.
- This was the US-37 acceptance checkpoint; current Wave B decisions and queue are recorded in the US-35 section below.

### US-35 Fuel Card Implementation

- Status: `IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING`; accounting remains 68/87 accepted and 19/87 remaining. Wave B remains open. V64 is the US-35 migration and the current repository head.
- Owner/boundary: Fuel owns a limited masked card-reference master, local lifecycle/restrictions, exactly one active Driver-or-Vehicle binding with immutable history, normalized imported provider facts, reconciliation, deterministic review indicators, and safe audit. The external provider owns account, authorization, settlement, ledger, provider IDs and merchant facts. No payment engine, banking, billing/payroll, generic fraud engine, or US-38 investigation is approved.
- Sensitive data: UUID internal identity, Organization provider logical ID, opaque provider card reference, masked identifier, optional last four and expiry month/year. No PAN, CVV, PIN, stripe data, balance, credential, secret, raw file/body, or unmasked reference appears in responses/UI/logs/audit.
- Lifecycle/binding: `DRAFT`, `ACTIVE`, `SUSPENDED`, `BLOCKED`, `EXPIRED`, `CANCELLED`; explicit commands only and no reactivation from terminal states. Exactly one same-Tenant active Vehicle or Driver binding is allowed; reassignment appends effective-dated history through published contracts only.
- Restrictions: per-transaction amount, daily/monthly amount, daily litre quantity, allowed fuel types, exact provider station/site allow-list and binding validation. Positive BigDecimal, one ISO-4217 currency, server time and Tenant timezone apply; imported policy violations/inactive use are retained and flagged rather than falsely claimed as provider-rejected.
- Import/evidence: Fuel owns an authenticated 1 MiB/1,000-record UTF-8 JSON `FUEL_CARD_TRANSACTIONS_V1` endpoint under `CONTROLLED_PROVIDER_FIXTURE`. Real parsing/validation/hashing/PostgreSQL persistence/dedupe are required. This proves no provider authenticity, settlement or external block. US-73 remains outbound-only; no new Integration inbound capability is claimed.
- Idempotency/reconciliation: batch key is Tenant/provider/batch plus hash; transaction key is Tenant/provider/provider-transaction ID plus canonical hash. Exact replay is idempotent and conflicting replay fails closed. Provider values are immutable; reversal is a new linked fact. Reconciliation links at most one purchase transaction to one existing US-32 Fuel Purchase and may retain a validated Trip reference; import never creates or mutates a Fuel Purchase.
- Review/security: exact indicators are binding mismatch, fuel type or station not allowed, limit exceeded, inactive card, integrity conflict and reversal review. They are non-fraud review signals; US-38 owns investigation. Permissions are `FUEL_CARD_VIEW`, `FUEL_CARD_MANAGE`, `FUEL_CARD_BLOCK`, `FUEL_CARD_IMPORT`, and `FUEL_CARD_RECONCILE`; importer cannot reconcile/reject the same transaction. Tenant is server-derived and P1-01 is `NONE` because no current consumer exists.
- Persistence: V64 creates eight Fuel-owned Tenant tables for card, binding history, restriction, import batch, immutable transaction, reconciliation history, indicators and audit; Tenant-consistent same-module FKs, logical cross-module UUIDs, optimistic versions, Tenant-leading indexes, no destructive delete, and external retention policy.
- Acceptance: isolated `transport_logistics_acceptance`, deterministic concurrency, literal `/api/v1` security, full quality gates, and a real Chromium journey covering masked display, binding/restrictions, canonical import/replay/conflict, reconciliation, review flags, immutable values, Tenant non-inference and limited-user denial.
- Technical closure: `US-35-FUEL-CARDS-TECHNICAL-CLOSURE-001` PASS. Fixture-baseline isolation was repaired without changing production behavior; the repaired test passed 1/1 and the complete focused group passed 23/23, including PostgreSQL 7/7 and literal `/api/v1/...` security coverage.
- Fresh closure evidence: only `transport_logistics_acceptance`; Flyway V1→V64; complete Maven 1,332/0/0/15 in 04:57; architecture 46/46; Checkstyle, PMD and SpotBugs PASS; TypeScript PASS; Vitest 263/263 across 63 files; production build and changed-file lint PASS; real PostgreSQL-backed Chromium 6/6 PASS; `git diff --check` PASS. Global ESLint retains 71 unrelated pre-existing Delivery errors and introduces zero US-35 errors.
- Next task: `US-35-FUEL-CARDS-FINAL-ACCEPTANCE-001`.

#### Table: `fuel_card`

- **Purpose:** Tenant-owned masked local fuel-card reference and lifecycle.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| `id`, `tenant_id`, `provider_id` | UUID, NOT NULL; PK on `id`; unique `(tenant_id,id)`; provider is a logical Organization reference |
| `alias`, `provider_card_reference`, `provider_reference_hash`, `masked_identifier`, `last_four` | VARCHAR(100), VARCHAR(255), CHAR(64), VARCHAR(32) NOT NULL; optional CHAR(4); unique Tenant/provider/reference; reference never leaves persistence unmasked |
| `expiry_month`, `expiry_year`, `status`, `provider_sync_status` | SMALLINT NOT NULL with month/year checks; lifecycle VARCHAR(20) check; sync is `NOT_CONFIGURED` only |
| `version`, `created_by`, `created_at`, `updated_at` | BIGINT NOT NULL default 0; UUID actor; TIMESTAMPTZ timestamps, all NOT NULL |

#### Table: `fuel_card_binding_history`

- **Purpose:** Immutable Vehicle-or-Driver binding history with one current binding.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| `id`, `tenant_id`, `card_id` | UUID NOT NULL; PK; Tenant-consistent FK to `fuel_card`; unique `(tenant_id,id)` |
| `binding_type`, `binding_id` | `VEHICLE` or `DRIVER`; UUID logical cross-module reference, both NOT NULL |
| `effective_from`, `effective_to`, `reason`, `changed_by`, `created_at` | TIMESTAMPTZ start/optional end; VARCHAR(500) reason; UUID actor; creation timestamp |
| Active-binding invariant | Partial unique index `(tenant_id,card_id)` where `effective_to IS NULL` |

#### Table: `fuel_card_restriction`

- **Purpose:** Current card spend, volume, fuel-type and station restrictions.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)

| Columns | Definition and constraints |
| :--- | :--- |
| `id`, `tenant_id`, `card_id` | UUID NOT NULL; Tenant-consistent card FK; unique one row per `(tenant_id,card_id)` |
| `currency`, amount limits | CHAR(3); transaction/daily/monthly NUMERIC(19,2), positive and NOT NULL |
| `max_daily_litres`, allowlists | positive NUMERIC(19,4); fuel types and station references stored as non-null TEXT |
| `version`, `changed_by`, `changed_at` | optimistic BIGINT default 0, UUID actor and TIMESTAMPTZ, NOT NULL |

#### Table: `fuel_card_import_batch`

- **Purpose:** Idempotent bounded canonical import receipt.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| Identity | UUID `id`, `tenant_id`, `provider_id`; unique `(tenant_id,id)` |
| Replay keys | VARCHAR(120) `provider_batch_id`, CHAR(64) `file_hash`; each unique within Tenant/provider |
| Counts/time | `generated_at`, `created_at` TIMESTAMPTZ; transaction count 1..1000; imported/review INTEGER counts |
| Actor | UUID `imported_by`, NOT NULL |

#### Table: `fuel_card_transaction`

- **Purpose:** Immutable normalized provider transaction facts plus local disposition.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| Identity/ownership | UUID `id`, `tenant_id`, `batch_id`, `provider_id`, `card_id`; Tenant-consistent batch/card FKs |
| Provider identity | VARCHAR(120) `provider_transaction_id`, CHAR(64) `canonical_hash`; unique Tenant/provider transaction |
| Kind/link | `PURCHASE` or `REVERSAL`; optional original provider transaction ID |
| Provider facts | transaction/optional posted TIMESTAMPTZ, optional station, fuel VARCHAR(40), positive quantity/unit price/amount, CHAR(3) currency, optional provider Vehicle/Driver references |
| Local references/state | optional logical UUID `trip_id` and `reconciled_purchase_id`; provider `POSTED/REVERSED`; local `IMPORTED/REVIEW_REQUIRED/RECONCILED/REJECTED/REVERSED` |
| Control | UUID `imported_by`, BIGINT `version` default 0, TIMESTAMPTZ `created_at`; one active reconciled purchase per Tenant |

#### Table: `fuel_card_reconciliation_history`

- **Purpose:** Append-only reconciliation decision history.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| `id`, `tenant_id`, `transaction_id` | UUID NOT NULL; Tenant-consistent transaction FK |
| `action`, `purchase_id`, `reason` | checked `MATCH/UNMATCH/REJECT/REVERSAL_DISPOSITION`; optional logical Fuel Purchase UUID; VARCHAR(500) reason |
| `actor_id`, `created_at` | UUID and TIMESTAMPTZ, NOT NULL |

#### Table: `fuel_card_transaction_indicator`

- **Purpose:** Deterministic non-fraud review indicators on imported evidence.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| `id`, `tenant_id`, `transaction_id` | UUID NOT NULL; Tenant-consistent transaction FK |
| `code`, `detail_code` | checked indicator VARCHAR(40), optional detail VARCHAR(60); unique per Tenant/transaction/code/detail |
| `created_at`, `acknowledged_by`, `acknowledged_at` | creation timestamp; optional actor and acknowledgement timestamp |

#### Table: `fuel_card_audit_event`

- **Purpose:** Safe append-only operational audit without raw card data.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Columns | Definition and constraints |
| :--- | :--- |
| References | UUID `id`, `tenant_id`; optional same-module `card_id` and `transaction_id` logical audit references |
| Decision | VARCHAR(50) `action`, VARCHAR(30) `result`, optional VARCHAR(80) `reason_code` |
| Safe change evidence | optional CHAR(64) `before_hash` and `after_hash`; no PAN or raw import body |
| Actor/time | UUID `actor_id`, TIMESTAMPTZ `created_at`, NOT NULL |

The foundation, operational repository isolation, scheduled-job isolation, Freight isolation, and Reporting-source isolation are `ACCEPTED_FOR_CURRENT_SCOPE`. US-29 Freight Reporting is `IMPLEMENTED`: Reporting exposes tenant-scoped summaries, pageable shipment/capacity results, insurance/claim/settlement/exception distributions, and a 5,000-row bounded CSV export through the Freight-owned public query boundary. Missing cargo measurements or vehicle capacity facts produce `INCOMPLETE`; they are never inferred. Access requires `FREIGHT_REPORT_VIEW`, while export independently requires `FREIGHT_REPORT_EXPORT`. Legacy preservation and backfill remain not applicable to this clean-initialization environment.
