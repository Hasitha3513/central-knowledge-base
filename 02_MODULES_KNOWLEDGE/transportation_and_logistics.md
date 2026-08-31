# Transportation and Logistics

Lifecycle: IN DEVELOPMENT
Source repository: current workspace
Schema baseline: Flyway V1–V47 (V47: tenant-scoped US-57 Proof of Delivery and evidence)
Delivery US-56, US-57, US-58: COMPLETE; US-59 through US-62 remain unimplemented
MVP 1.3 Delivery Operations: 3/7 COMPLETE
Delivery US-59 decision gate: PRODUCT_DECISIONS_FROZEN; implementation NOT_STARTED
US-30 Cargo Exceptions: COMPLETE (P2-CARGO-EXCEPTION-001)
US-29 Freight Reporting: COMPLETE (P2-FREIGHT-REPORTING-001)
Freight release status: 7/7 COMPLETE
Tenant readiness: FOUNDATION IMPLEMENTED / OPERATIONAL ISOLATION ACCEPTED_FOR_CURRENT_SCOPE

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

## Published Events

Current internal event types are `VehicleReadingRecorded`, `VehicleReadingCorrected`, `VehicleMeterResetRecorded`, `RouteDisruptionCreatedEvent`, `RouteDisruptionResolvedEvent`, and `OperationalNotificationEvent`. Exact payloads and tenant deficiencies are registered in `../01_INTEGRATION_REGISTRY/event_contracts.md`.

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

### US-59 Failed Deliveries Product Decisions

`MVP-1.3-US59-FAILED-DELIVERIES-PRODUCT-DECISIONS-001` freezes the following without implementing US-59:

- **Failure Reason Taxonomy:** Standardized enums (`CUSTOMER_UNAVAILABLE`, `WRONG_ADDRESS`, `CUSTOMER_REFUSED`, `ACCESS_RESTRICTED`, `DAMAGED_CARGO`, `DOCUMENT_OR_PAYMENT_ISSUE`, `OTHER`). Arbitrary free-text-only status mutation is prohibited; `OTHER` requires mandatory non-empty notes (>= 10 chars).
- **Delivery Lifecycle Extension:** `READY_FOR_ASSIGNMENT` can transition to `FAILED_ATTEMPT` (non-terminal, redelivery eligible), `RETURN_TO_BASE` (terminal/return custody), or `ESCALATED` (management hold). Finalized `DELIVERED` orders remain immutable and can never be marked failed.
- **Delivery Attempt & Contact Model:** Separate immutable entities `DeliveryAttempt` and `DeliveryContactAttempt` capturing sequential attempt numbering, UTC timestamps, failure reasons, contact channels (`PHONE`, `SMS`, `WHATSAPP`, `EMAIL`, `IN_PERSON`), contact outcomes, operator IDs, and tenant isolation.
- **Privacy & PII Protection:** Contact attempts record only channel and outcome metadata; customer phone numbers/emails remain referenced from Customer master and are not duplicated into logs or attempt payloads.
- **Escalation & RTO Semantics:** Local operational escalation tracking (`status`, `reason`, actor, timestamps). Return-to-Base marks orders as permanently failed in the field and triggers return custody.
- **Story Boundaries:** US-59 determines that another attempt is needed (`REDELIVERY_ELIGIBLE`). US-60 owns customer time preference collection and slot scheduling. US-61 owns analytics. US-62 owns specialized exception gates.
- **Offline Policy:** `ONLINE_ONLY_FOR_US59` in MVP Phase 1.3.
- **RBAC:** `DELIVERY_FAIL_RECORD`, `DELIVERY_FAIL_VIEW`, `DELIVERY_FAIL_ESCALATE`, `DELIVERY_RETURN_INITIATE`.
- **Expected Persistence:** Forward migration V48 when implementation begins (tables: `delivery_attempt`, `delivery_contact_attempt`, `delivery_escalation`). Current schema remains V1–V47 until implementation.

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

## Database Schema Data Dictionary

### Schema-wide tenancy status

Flyway V43 implements the first-class Tenant and Tenant membership foundation. V44 adds membership-scoped role assignment and non-null, indexed `tenant_id` ownership to current-scope operational tables across Identity token persistence, Organization, Fleet/Driver, Routing/Trip, Fuel, Freight, Notification, and Offline Sync. Existing physical foreign keys reflect the current modular monolith and are factual documentation, not approval for future cross-module coupling. Historical migrations are immutable.

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

### Notification and offline-sync tables (V25–V29)

| Table | Purpose | Primary key | Key columns | Constraints / indexes |
| :--- | :--- | :--- | :--- | :--- |
| `notification_rule` | Event routing rule | `id UUID` | code/name/event_type/channel/recipient settings/template/enabled/audit/version | unique code; channel/recipient checks; event/template indexes |
| `notification` | Notification instance | `id UUID` | event/aggregate/rule/template, recipient/channel/content/status, delivery/read/schedule/escalation/audit | status/channel/escalation checks; parent/template FKs; recipient/event/time indexes |
| `notification_template` | Versioned message template | `id UUID` | code/version/event/channel/subject/body/required variables/active/audit | unique code/version; channel checks; lookup index |
| `notification_rule_policy` | Quiet hours/suppression/escalation policy | `rule_id UUID` | timezone, quiet settings, suppression window, escalation settings, audit/version | FK rule cascade; window/time checks |
| `notification_rule_quiet_day` | Quiet weekdays | `(rule_id,day_of_week)` | rule, weekday | FK policy; weekday check |
| `notification_rule_execution` | Rule decision audit | `id UUID` | execution/event/aggregate/rule/recipient/channel/outcome/suppression/control/failure/timestamps | unique execution key; FKs; channel/outcome checks; audit indexes |
| `notification_delivery_attempt` | Durable email attempt | `id UUID` | notification, attempt number/state/due/start/end/error/provider/created | unique notification/attempt; attempt/state checks; due/history indexes |
| `offline_sync_operation` | Idempotent server inbox | `operation_id UUID` | operation type/version, actor/client, aggregate, request hash, result/version, processed/created | actor FK; version/result checks; actor/aggregate indexes |

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

V1 baseline; V2 identity; V3 documents; V4 licences; V5 stops; V6–V8 trip audit/dispatch; V9 permissions; V10 integrity; V11–V12 fuel; V13 permissions; V14–V16 readings/reset; V17 permissions; V18 bunker; V19 maintenance; V20–V22 driver compliance; V23 lubricant; V24 operational events; V25–V28 notifications; V29 offline sync; V30 routing history; V31–V32 freight order/manifest; V33 permissions; V34 load plan; V35 permissions; V36 insurance; V37 Cargo Manifest special-cargo classification; V38 load plan readiness; V39 vehicle capacity master data; V40 cargo exception permissions; V41 cargo exception tables; V42 cargo manifest item measurements; V43 Tenant, membership, and canonical clean bootstrap; V44 operational tenant scoping and membership-role authority; V45 Freight reporting view/export permissions; V46 US-56 Delivery Orders, number counter and permissions; V47 US-57 Proof of Delivery, evidence, and POD permissions.

US-56 and US-57 Delivery persistence is introduced by forward migrations V46 and V47.

## Remaining Suite Integration Work

1. Add Tenant envelopes whenever new cross-module integration events are approved or existing event contracts are versioned.
2. Resolve driver, customer, project, vendor, and maintenance ownership through ADRs.
3. Replace cross-boundary physical references with logical IDs/contracts as modules become independent.

The foundation, operational repository isolation, scheduled-job isolation, Freight isolation, and Reporting-source isolation are `ACCEPTED_FOR_CURRENT_SCOPE`. US-29 Freight Reporting is `IMPLEMENTED`: Reporting exposes tenant-scoped summaries, pageable shipment/capacity results, insurance/claim/settlement/exception distributions, and a 5,000-row bounded CSV export through the Freight-owned public query boundary. Missing cargo measurements or vehicle capacity facts produce `INCOMPLETE`; they are never inferred. Access requires `FREIGHT_REPORT_VIEW`, while export independently requires `FREIGHT_REPORT_EXPORT`. Legacy preservation and backfill remain not applicable to this clean-initialization environment.
