# API and Synchronous Interface Registry

## Contract Rules

- New APIs use `/api/v1` unless an existing public contract requires compatibility.
- Tenant context comes from active server-side membership and Tenant validation after authentication. `CurrentTenant` / `TenantExecutionContext` is runtime authority; client-supplied or JWT Tenant IDs are never sole authority.
- Commands are idempotent where retries are expected and expose explicit lifecycle operations.
- Errors follow the suite envelope: `timestamp`, `status`, `error`, `code`, `message`, `path`, `correlationId`, `fieldErrors`.
- Consumers call through an outbound port and adapter; they do not import provider application services.

## Transportation — Implemented Route Families

The precise request/response schemas remain authoritative in controller DTOs and generated OpenAPI. This registry records capability families.

| Capability | Implemented route family | Owner | Status / notes |
| :--- | :--- | :--- | :--- |
| Identity | `/auth`, `/users`, `/roles` | Identity | Login, refresh, and bearer requests revalidate active membership/Tenant server-side. `/users` administration is limited to the actor's active Tenant; new users join that Tenant. Role/user grants cannot exceed the actor's current permissions, and global role templates assigned outside the actor Tenant cannot be mutated. Assignments use `tenant_membership_role`; JWT and REST schemas are unchanged. |
| Organization reference data | `/customers`, `/departments`, `/locations`, `/projects`, `/vendors` | Organization | Implemented; some ownership will need suite-level reconciliation |
| Fleet and drivers | `/vehicles`, `/vehicle-categories`, `/vehicle-types`, `/drivers` | Fleet | Implemented |
| Fleet compliance/usage | `/vehicles/{id}/documents`, `/readings`, `/meter-resets`, `/maintenance-schedules`, `/lubricant-logs`; driver licenses/exceptions/violations/medical/drug tests | Fleet | Implemented; HRM/Maintenance ownership must be resolved before extraction |
| Routing | `/routes`, revisions, disruptions, optimization, performance | Routing | Implemented |
| Trips | `/trips` plus explicit submit/approve/reject/assign/dispatch/start/complete/close/cancel and operational-event commands | Trip | Implemented |
| Fuel | `/fuel-issues`, `/fuel-purchases`, `/fuel-prices`, `/bunker-tanks`, `/trips/{tripId}/fuel-cost` | Fuel | Implemented |
| Fuel Performance (US-37) | `GET /api/v1/fuel/performance/summary`, `/vehicles`, `/vehicles/{vehicleId}`, `/drivers`, `/drivers/{driverId}`, `/trends` | Fuel | `COMPLETE / FINAL_ACCEPTANCE_PASS`. Read-only on-demand projection; `FUEL_PERFORMANCE_VIEW`; 7/30/90/custom≤365 Tenant-calendar-day ranges; pageable comparisons default 20/max 100; no write/export/configuration route. |
| Fuel Cards (US-35) | `/api/v1/fuel/cards`, card lifecycle/binding/restriction/history commands; `/api/v1/fuel/card-imports`; `/api/v1/fuel/card-transactions` plus explicit match/unmatch/reject/reversal-disposition commands | Fuel | `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED`; US-35 remains at V64 and current repository head is V65 for internal Bunker ledger ordering. Masked card-reference model; local-only lifecycle; controlled `FUEL_CARD_TRANSACTIONS_V1` multipart JSON import; immutable provider facts; explicit reconciliation to an existing Fuel Purchase; safe filters/sorts and default page 20/maximum 100; no delete/raw edit/payment/provider-sync route. |
| Freight | `/v1/freight/orders`, `/manifests`, `/load-plans`, `/insurance/policies`, `/insurance/claims`, `/exceptions` | Freight | Implemented path prefix differs from `/api/v1`; preserve until approved migration |
| Notifications | `/notifications`, `/notification-rules`, `/notification-templates`, delivery diagnostics | Notification | Implemented |
| Reporting | `/dashboard/operations`, `/reports/*` | Reporting | Implemented read models; source repositories are tenant-isolated through authoritative runtime context |
| Freight reporting (US-29) | `GET /reports/freight/summary`, `GET /reports/freight/shipments`, `GET /reports/freight/export` | Reporting (inbound/API), Freight (source query contract) | Implemented tenant-scoped read-only summaries, pageable shipment/capacity rows, and bounded CSV; permissions `FREIGHT_REPORT_VIEW` and `FREIGHT_REPORT_EXPORT`; missing measurements/capacity remain `INCOMPLETE` |
| Delivery Orders (US-56) | `POST/GET /v1/deliveries`, `GET/PATCH /v1/deliveries/{id}`, `POST /v1/deliveries/{id}/validate-readiness` | Delivery | Implemented tenant-scoped requirements and readiness workflow |
| Proof of Delivery (US-57/58) | `GET/POST /v1/deliveries/{id}/proof`, `/evidence`, `/finalize` | Delivery | Implemented online and offline POD capture, evidence storage, and finalization |
| Failed Deliveries (US-59) | `POST /v1/deliveries/{id}/failed-attempt`, `GET /v1/deliveries/{id}/attempts`, `POST /v1/deliveries/{id}/attempts/{attemptId}/contacts`, `POST /v1/deliveries/{id}/escalate`, `PATCH /v1/deliveries/{id}/escalations/{escalationId}`, `POST /v1/deliveries/{id}/return-to-base` | Delivery | Implemented failed attempts, contact attempts, escalations, return-to-base, and failure history |
| Last-Mile Planner (US-68) | `GET /v1/deliveries/{id}/last-mile-planner` | Delivery | Accepted read-only projection over existing order, attempt, escalation, and exception capabilities; requires `DELIVERY_FAIL_VIEW` or `DELIVERY_EXCEPTION_VIEW`; no mutation or duplicate exception API family |
| Delivery Notifications (US-69) | `GET /notification-deliveries?aggregateType=DELIVERY_ORDER&aggregateId={deliveryId}&limit={1..200}`; existing `GET /notification-deliveries/{notificationId}/attempts`; `GET/PUT /notification-customer-preferences/{customerId}` | Notification | `IMPLEMENTED / ACCEPTANCE_PENDING`. External URLs are under `/api/v1`. History/preferences are Tenant-scoped; reads require `NOTIFICATION_RULE_VIEW`, preference replacement requires `NOTIFICATION_RULE_MANAGE`. GET preference returns customer/profile flags, effective Email/SMS flags, masked destinations, and nullable version. PUT body is `{emailEnabled:boolean,smsEnabled:boolean,version:long|null}` and replaces the complete profile. No send/resend endpoint or Delivery-owned notification CRUD exists. |
| Customer Self-Service (US-70) | `GET /api/public/v1/delivery-self-service`; `GET/PUT /api/public/v1/delivery-self-service/notification-preferences`; `POST /api/public/v1/delivery-self-service/issues`; `POST /api/public/v1/delivery-self-service/feedback`; `POST /api/public/v1/delivery-self-service/redelivery-requests` | Delivery | `IMPLEMENTED / ACCEPTED`. Every request uses `Authorization: DeliveryAccess <opaque-token>`; routes/body contain no authoritative Tenant, Delivery, or Customer ID. Tracking is a minimized projection. Writes cover existing Email/SMS preferences, issue/feedback submissions, and a non-binding scheduling request only. No direct slot booking or Delivery mutation. Final acceptance passed 28/28 focused, 4/4 PostgreSQL, 1,238-test Maven, 42/42 architecture, 259/259 Vitest, and 9/9 real Chromium gates. |
| Delivery Zones (US-63) | `POST/GET /v1/delivery-zones`, `GET/PUT /v1/delivery-zones/{id}`, `POST /v1/delivery-zones/{id}/activate`, `/deactivate`, `POST /v1/delivery-zones/resolve` | Delivery | Implemented GeoJSON polygon zone management, PiP resolution, and serviceability controls |
| Delivery Slots (US-64) | `POST/GET /api/v1/deliveries/slots`, `GET/PUT /api/v1/deliveries/slots/{id}`, `PATCH /api/v1/deliveries/slots/{id}/activate`, `/close`, `POST /api/v1/deliveries/slots/{id}/reserve`, `POST /api/v1/deliveries/orders/{id}/assign-slot`, `GET /api/v1/deliveries/slots/available` | Delivery | Implemented time-window capacity slots, reservation concurrency control, cutoffs, and overrides |
| Delivery Riders (US-65) | `POST/GET /api/v1/deliveries/riders`, `GET/PUT /api/v1/deliveries/riders/{id}`, `PATCH /api/v1/deliveries/riders/{id}/activate`, `/deactivate`, `/suspend`, `POST/GET /api/v1/deliveries/riders/{id}/shifts`, `PATCH /api/v1/deliveries/riders/{id}/shifts/{shiftId}/start`, `/end`, `/cancel`, `GET /api/v1/deliveries/riders/available`, `POST /api/v1/deliveries/orders/{id}/assign-rider`, `POST /api/v1/deliveries/orders/{id}/reassign-rider`, `POST /api/v1/deliveries/orders/{id}/unassign-rider`, `GET /api/v1/deliveries/orders/{id}/rider-assignments` | Delivery | Implemented rider onboarding, duty shift schedules, concurrency-safe order assignment, and cross-zone/capacity overrides |
| Offline sync | `/offline-sync/operations` | Offline Sync | Implemented inbox; HTTP requires authentication and each batch item is authorized in the application service using its existing operation-specific permission |
| Health | `/health` | System | Implemented |
| External Integrations (US-73) | `GET/POST /api/v1/integrations`; `GET/PUT /api/v1/integrations/{id}`; `POST /api/v1/integrations/{id}/test`, `/enable`, `/disable`; `GET /api/v1/integrations/{id}/exchanges` | Integration | `IMPLEMENTED / ACCEPTED`. Tenant-scoped configuration and read-only exchange evidence for the single governed outbound JSON-file adapter. Default page size 20, maximum 100. No delete, arbitrary send, manual retry, mark-success, reconciliation mutation, secret/raw-payload read, public webhook, or operator-controlled path/URL route exists. |

### US-73 Accepted REST Semantics

- `POST` creates a `DRAFT`; `PUT` is allowed only while `DRAFT` or `DISABLED` and uses optimistic versioning. Operators choose a server-managed endpoint alias, not a raw path or URL.
- `test` performs a rate-limited atomic write/read/delete probe using non-business data; limit is five probes per minute per user and configuration. A successful probe remains activation-eligible for 15 minutes.
- `enable` and `disable` are explicit commands. Unsupported capability/protocol/direction combinations fail closed. Disabling prevents new claims/attempts until re-enabled.
- List/detail/history never return secret material or unmasked payload. Same-Tenant absence and cross-Tenant identifiers return safe not-found behavior.
- Standard errors are `INTEGRATION_NOT_FOUND`, `INTEGRATION_DISABLED`, `INTEGRATION_CONFIGURATION_INVALID`, `INTEGRATION_CAPABILITY_UNSUPPORTED`, `INTEGRATION_AUTH_FAILED`, `INTEGRATION_MAPPING_INVALID`, `INTEGRATION_PAYLOAD_INVALID`, `INTEGRATION_DUPLICATE`, `INTEGRATION_RATE_LIMITED`, `INTEGRATION_PROVIDER_UNAVAILABLE`, `INTEGRATION_TERMINAL_FAILURE`, `INTEGRATION_CONFLICT`, and `INTEGRATION_FILE_INTEGRITY_FAILURE` in the suite error envelope.

### P0-03 Published Internal Query Contracts

- US-37 publishes the provider-neutral Fuel-root `FuelPerformanceQuery` returning `FuelPerformanceSummary`, `VehicleFuelPerformance`, `DriverFuelPerformance`, and `FuelPerformanceTrend`. Reporting may consume this contract but may not redefine Fuel metrics or access Fuel persistence. Fuel consumes minimal bulk Tenant-scoped `FleetFuelPerformanceLookup` and `TripFuelPerformanceLookup` root projections for compatible Vehicle/Driver/usage dimensions; no per-row/N+1 foreign calls are used.
- US-35 validates provider, Vehicle, Driver, Trip and existing Fuel Purchase logical references only through published same-Tenant contracts. The implemented inbound adapter is Fuel-owned and accepts one authenticated bounded JSON file; it does not expand US-73's accepted outbound-only capability. No inbound Integration API, P1-01 event, provider API, payment API, raw-card-data API, or US-38 investigation API is approved.

- Fleet publishes `FleetReportingQuery.findVehicle(UUID vehicleId): Optional<FleetVehicleSummary>`. Freight reporting uses it synchronously for tenant-scoped payload and volume capacity facts; consumers do not access Fleet persistence.
- Organization publishes `CustomerLookup.find(UUID customerId): Optional<CustomerReference>` for tenant-scoped business lookups and `CustomerDataReadiness.anyCustomerExists(): boolean` for the global opt-in local-data readiness probe. Consumers do not access Organization persistence.
- Organization publishes `CustomerNotificationContactLookup.find(UUID customerId): Optional<CustomerNotificationContact>` for the Notification-owned US-69 recipient flow. The same-Tenant projection contains only customer ID, active flag, display name, phone, and email; Notification does not access Organization persistence.
- Delivery publishes a customer-self-service facade that consumes the Organization contact projection and the Notification-root `CustomerOperationalPreferenceManagement` contract under token-derived Tenant/Customer authority. It does not import Notification's internal application use case and never accesses either provider's persistence.
- Delivery publishes the provider-neutral `CustomerSelfServiceLinkIssuer` for Notification, while Notification publishes `FinalSendCustomerLinkIssuer` as its final-send seam. Immediately before an Email/SMS provider attempt, Notification supplies trusted Tenant, Delivery, Customer, normalized destination fingerprint, allowed actions, and its stable delivery-attempt idempotency key; Delivery returns a raw HTTPS fragment link transiently and persists only its token hash. Notification persists only `[[SELF_SERVICE_LINK]]`/redacted state. These contracts are implemented for US-70; existing US-69 events are unchanged.
- Both business contracts and the provisioning-readiness contract are in their provider module root packages. Business query adapters enforce the current Tenant context; the readiness result is intentionally global to preserve the existing bootstrap skip behavior. These are internal Java contracts and add or change no REST endpoint, JSON schema, event, or database schema.

### Freight Manifest Internal Application Contract

`CargoManifestUseCase.ItemCommand` is an internal inbound-port command used by Manifest item create/update orchestration. Its active fields are `version: Long?`, `freightOrderLineId: UUID`, `description: String`, `quantity: Decimal`, `packingInformation: String`, `commodityClassification: String`, `customsApplicable: boolean`, `customsInformation: String?`, `hazardous: boolean`, `hazardousClassification: String?`, `hazardousDetails: String?`, `fragile: Boolean?`, `temperatureSensitive: Boolean?`, `unitWeight: BigDecimal?`, `weightUnit: String?`, `length: BigDecimal?`, `width: BigDecimal?`, `height: BigDecimal?`, and `dimensionUnit: String?`.

The special-cargo fields use tri-state semantics: `TRUE` and `FALSE` are explicit classifications; `NULL` is UNKNOWN. Omitted values remain UNKNOWN for compatibility and block Manifest finalization through `SPECIAL_CARGO_CLASSIFICATION_MISSING`. Measurement fields use nullable numerical decimals; missing measurements preserve `null` and cause downstream US-27 weight/volume validation to report `INCOMPLETE` with missing fact diagnostics.

### Freight Manifest Public Item Contract

`POST /v1/freight/manifests/{manifestId}/items` and `PATCH /v1/freight/manifests/{manifestId}/items/{itemId}` accept additive nullable JSON properties:
- `fragile: boolean | null`
- `temperatureSensitive: boolean | null`
- `unitWeight: number | null` (positive decimal)
- `weightUnit: "KG" | "G" | "TONNE" | null`
- `length: number | null` (positive decimal)
- `width: number | null` (positive decimal)
- `height: number | null` (positive decimal)
- `dimensionUnit: "M" | "CM" | "MM" | null`

Omitted or explicit `null` values remain UNKNOWN; the adapter never defaults them to `false` or `0`. Cargo Manifest responses expose these properties with the same nullable semantics.

The first-party Manifest UI provides input fields for special cargo classification and physical cargo measurements. Historical UNKNOWN classifications are rendered as `CLASSIFICATION REQUIRED`, and missing measurements as `WEIGHT REQUIRED` / `DIMENSIONS REQUIRED`. Permissions remain `CARGO_MANIFEST_VIEW`, `CARGO_MANIFEST_MANAGE`, and `CARGO_MANIFEST_FINALIZE`.

## US-78 Operations API (Accepted, Complete)

Operations implements authenticated `/api/v1/operational-exceptions` list/detail/history routes and explicit commands for `classify`, `acknowledge`, `assign`, `start`, `escalate`, corrective-action create/start/complete, RCA record/approve, `resolve`, `close`, `reject-resolution`, and `reopen`. Every mutation carries the expected version. There is no manual create, generic PATCH/status, delete, source mutation, arbitrary send, raw-evidence, customer, or export route.

List filters are limited to status, severity, category, source module, assigned user/role, SLA status, and opened date range. Pagination defaults to 20 and caps at 100; history defaults to 50 and caps at 200. Safe sort keys are `openedAt`, `updatedAt`, `severity`, `responseDueAt`, `resolutionDueAt`, and `status`. Search is limited to exact/prefix case reference, registered summary code, or exact source UUID. Cross-Tenant IDs return safe not-found and requests cannot set Tenant/source/case/audit/SLA/escalation/approval authority.

The public cross-module intake is the Operations-root `OperationalExceptionFactV1` event contract, not a REST create API. Routing and Delivery are the active V62 producers. Source correction remains a separate call to the source owner's published use case. Implementation is `COMPLETE_US78` after independent final acceptance.

Exact implemented routes and authority:

| Method and path | Permission |
| :--- | :--- |
| `GET /api/v1/operational-exceptions` | `OPERATIONAL_EXCEPTION_VIEW` |
| `GET /api/v1/operational-exceptions/{id}` | `OPERATIONAL_EXCEPTION_VIEW`; RCA fields additionally gated by `OPERATIONAL_EXCEPTION_RCA` |
| `GET /api/v1/operational-exceptions/{id}/history` | `OPERATIONAL_EXCEPTION_AUDIT_VIEW` |
| `POST /api/v1/operational-exceptions/{id}/classify` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/acknowledge` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/assign` | `OPERATIONAL_EXCEPTION_ASSIGN`, or `MANAGE` for validated self-assignment only |
| `POST /api/v1/operational-exceptions/{id}/start` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/escalate` | `OPERATIONAL_EXCEPTION_ESCALATE` |
| `POST /api/v1/operational-exceptions/{id}/corrective-actions` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/corrective-actions/{actionId}/start` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/corrective-actions/{actionId}/complete` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/rca` | `OPERATIONAL_EXCEPTION_RCA` |
| `POST /api/v1/operational-exceptions/{id}/rca/approve` | `OPERATIONAL_EXCEPTION_RCA` plus SoD |
| `POST /api/v1/operational-exceptions/{id}/resolve` | `OPERATIONAL_EXCEPTION_MANAGE` |
| `POST /api/v1/operational-exceptions/{id}/close` | `OPERATIONAL_EXCEPTION_CLOSE` plus closure/SoD rules |
| `POST /api/v1/operational-exceptions/{id}/reject-resolution` | `OPERATIONAL_EXCEPTION_CLOSE` |
| `POST /api/v1/operational-exceptions/{id}/reopen` | `OPERATIONAL_EXCEPTION_CLOSE` |

## Proposed Cross-Module Interfaces

### Delivery Foundation and Preserved Future-Slice Contracts

- `DeliveryTenantContextPort`, implemented by the `CurrentTenant` adapter, is the only actively wired foundation port.
- `DeliveryLookupPort.findReference(UUID deliveryId): Optional<DeliveryReference>` is a preserved, unimplemented read-only contract for a future cross-module consumer.
- Preserved future-slice outbound contracts are `DeliveryCustomerLookupPort`, `DeliveryLocationLookupPort`, `DeliveryFreightOrderLookupPort`, `DeliveryTripLookupPort`, `DeliverySlotAvailabilityPort`, `DeliveryEvidenceStoragePort`, `DeliveryOfflineSyncPort`, and `DeliveryNotificationPort`.
- These provider-neutral contracts do not constitute implemented use cases and do not approve cross-module SQL, direct repository access, REST APIs, events, schema, adapters, or workflow behavior. No synthetic foundation-status use case or runtime permission-catalogue bean exists.

### US-56 Delivery Order Contract (Implemented)

- Delivery-owned priority values: `LOW`, `NORMAL`, `HIGH`, `URGENT`; create default `NORMAL`. Priority records urgency only in US-56.
- Delivery-owned service types: `STANDARD`, `EXPRESS`, `SAME_DAY`, `SCHEDULED`; create default `STANDARD`. Each uses an explicit valid delivery window and implies no pricing, routing, SLA, POD or assignment behavior.
- US-56 creates/updates/reads Delivery Orders and validates readiness. Assignment target is `NONE_IN_US56`; no assignment request field, response field, selector or persistence column is approved.
- Readiness fails closed unless server Tenant context and active same-Tenant customer/origin/destination references resolve; origin differs from destination and window start precedes end.
- `deliveryNumber` is an immutable server-generated response field and is excluded from create/update requests. Its exact format is `DEL-YYYY-NNNNNN`, allocated atomically per Tenant and Tenant-local calendar year; gaps are allowed, values are never reused, and US-56 defines no client idempotency key.
- Implemented routes are `POST/GET /api/v1/deliveries`, `GET/PATCH /api/v1/deliveries/{deliveryId}`, and `POST /api/v1/deliveries/{deliveryId}/validate-readiness`; application controller mappings omit the deployment-level `/api` prefix and use `/v1/deliveries`.
- Requests never accept `tenantId` or `deliveryNumber`. PATCH and readiness commands require the optimistic `version`; stale versions return the standard conflict error.

### US-57 Online Proof-of-Delivery Contract (Product Decisions Frozen; Not Implemented)

- US-57 is online-only. US-58 owns offline signature/photo capture, quality/retake, consent and Offline Sync integration.
- A valid POD requires at least one primary evidence type from `SIGNATURE`, `PHOTO` or `BARCODE`; any non-empty combination is allowed. Limits are one signature, three photos and one barcode.
- The barcode is the target Delivery Order's immutable `DEL-YYYY-NNNNNN` number. Server acceptance time is the authoritative UTC completion instant; optional device time is audit-only. Geo-tag is optional where available and missing GPS does not block valid proof.
- Delivery owns metadata and uses its provider-neutral `DeliveryEvidenceStoragePort` for binary evidence. Binary upload is multipart at the inbound adapter; public URLs, filesystem paths, object keys and request `tenantId` are forbidden.
- Conceptual routes are `POST/GET /v1/deliveries/{deliveryId}/proof`, `POST /v1/deliveries/{deliveryId}/proof/evidence`, `DELETE /v1/deliveries/{deliveryId}/proof/evidence/{evidenceId}`, `GET /v1/deliveries/{deliveryId}/proof/evidence/{evidenceId}/content`, and `POST /v1/deliveries/{deliveryId}/proof/finalize`.
- `DELIVERY_POD_CAPTURE` governs draft/evidence/finalization actions. Privacy-sensitive metadata/content viewing uses `DELIVERY_POD_VIEW`.
- Finalization requires matching Delivery/POD optimistic versions, durable validated evidence and same-Tenant ownership. It atomically finalizes the one POD and transitions the current `READY_FOR_ASSIGNMENT` Delivery to `DELIVERED`; this is an explicit transitional US-57 path and does not claim assignment or `OUT_FOR_DELIVERY` facts.
- Finalized POD/evidence is immutable. US-57 exposes no finalized edit/delete/correction endpoint and no public integration event.

| Provider | Interface purpose | Consumers | Status |
| :--- | :--- | :--- | :--- |
| Transportation | Trip/freight chargeable facts, vehicle/route lookup, delivery status | Finance, Sales/CRM, Maintenance | PROPOSED |
| Finance | Account validation, budget check, invoice/payment status | Procurement, Projects, Sales/CRM, Transportation | PROPOSED |
| HRM | Employee/driver qualification and availability | Transportation, Projects, Maintenance | PROPOSED |
| Inventory | Stock availability/reservation and issue | Procurement, Maintenance, Projects | PROPOSED |
| Procurement | Purchase-order and supplier-contract status | Inventory, Maintenance, Finance | PROPOSED |
| Projects | Project/cost-code validation | Finance, Procurement, HRM | PROPOSED |
| Sales/CRM | Customer/account and sales-order lookup | Transportation, Finance | PROPOSED |
| Vehicle Maintenance | Vehicle maintenance hold and return-to-service status | Transportation | PROPOSED |

An interface becomes `ACTIVE` only after its path or protocol, operation IDs, schemas, permissions, tenant behavior, errors, versioning, timeouts, retries, and contract tests are registered.
