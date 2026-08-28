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
| Identity | `/auth`, `/users`, `/roles` | Identity | Login, refresh, and bearer requests revalidate active membership/Tenant server-side; assignments use `tenant_membership_role` with global role/permission definitions; JWT contract unchanged |
| Organization reference data | `/customers`, `/departments`, `/locations`, `/projects`, `/vendors` | Organization | Implemented; some ownership will need suite-level reconciliation |
| Fleet and drivers | `/vehicles`, `/vehicle-categories`, `/vehicle-types`, `/drivers` | Fleet | Implemented |
| Fleet compliance/usage | `/vehicles/{id}/documents`, `/readings`, `/meter-resets`, `/maintenance-schedules`, `/lubricant-logs`; driver licenses/exceptions/violations/medical/drug tests | Fleet | Implemented; HRM/Maintenance ownership must be resolved before extraction |
| Routing | `/routes`, revisions, disruptions, optimization, performance | Routing | Implemented |
| Trips | `/trips` plus explicit submit/approve/reject/assign/dispatch/start/complete/close/cancel and operational-event commands | Trip | Implemented |
| Fuel | `/fuel-issues`, `/fuel-purchases`, `/fuel-prices`, `/bunker-tanks`, `/trips/{tripId}/fuel-cost` | Fuel | Implemented |
| Freight | `/v1/freight/orders`, `/manifests`, `/load-plans`, `/insurance/policies`, `/insurance/claims`, `/exceptions` | Freight | Implemented path prefix differs from `/api/v1`; preserve until approved migration |
| Notifications | `/notifications`, `/notification-rules`, `/notification-templates`, delivery diagnostics | Notification | Implemented |
| Reporting | `/dashboard/operations`, `/reports/*` | Reporting | Implemented read models; source repositories are tenant-isolated through authoritative runtime context |
| Freight reporting (US-29) | `GET /reports/freight/summary`, `GET /reports/freight/shipments`, `GET /reports/freight/export` | Reporting (inbound/API), Freight (source query contract) | Implemented tenant-scoped read-only summaries, pageable shipment/capacity rows, and bounded CSV; permissions `FREIGHT_REPORT_VIEW` and `FREIGHT_REPORT_EXPORT`; missing measurements/capacity remain `INCOMPLETE` |
| Delivery foundation | No public REST route yet | Delivery | Foundation implemented: Spring Modulith module, framework-free domain primitives, preserved future-slice lookup/reference and outbound-port contracts, and authoritative tenant-context adapter; no application use case or permission-catalogue bean exists; US-56 through US-62 APIs remain pending |
| Offline sync | `/offline-sync/operations` | Offline Sync | Implemented inbox |
| Health | `/health` | System | Implemented |

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

## Proposed Cross-Module Interfaces

### Delivery Foundation and Preserved Future-Slice Contracts

- `DeliveryTenantContextPort`, implemented by the `CurrentTenant` adapter, is the only actively wired foundation port.
- `DeliveryLookupPort.findReference(UUID deliveryId): Optional<DeliveryReference>` is a preserved, unimplemented read-only contract for a future cross-module consumer.
- Preserved future-slice outbound contracts are `DeliveryCustomerLookupPort`, `DeliveryLocationLookupPort`, `DeliveryFreightOrderLookupPort`, `DeliveryTripLookupPort`, `DeliverySlotAvailabilityPort`, `DeliveryEvidenceStoragePort`, `DeliveryOfflineSyncPort`, and `DeliveryNotificationPort`.
- These provider-neutral contracts do not constitute implemented use cases and do not approve cross-module SQL, direct repository access, REST APIs, events, schema, adapters, or workflow behavior. No synthetic foundation-status use case or runtime permission-catalogue bean exists.

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
