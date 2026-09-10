# Global Event Contracts Registry

All domain events crossing module boundaries must be documented here with their JSON payload specifications.

Statuses: `NO_CONSUMER` exists but has no production consumer; `LOCAL_AFTER_COMMIT` has an actual in-process post-commit consumer; `DURABLE_INTERNAL_REQUIRED` uses the approved durable internal boundary; `FROZEN_NOT_IMPLEMENTED` is approved for the named story but absent from production source; `PROPOSED` requires approval and implementation.

## Standard Domain Event Envelope

```json
{
  "eventId": "UUID",
  "eventType": "String",
  "tenantId": "UUID",
  "occurredAt": "2026-08-26T14:44:54Z",
  "version": 1,
  "aggregateType": "String",
  "aggregateId": "UUID",
  "payload": {}
}
```

Every registered cross-module event must use this envelope. Its contract entry must define the exact `eventType`, versioned payload schema, producer, aggregate identity where applicable, known consumers, delivery semantics, ordering, idempotency, security classification, retention, and correlation/causation behavior.

Compatibility rules: additive optional fields are backward compatible; renames, removals, semantic changes, type changes, and newly required fields require a new `version`. Consumers must ignore unknown fields and deduplicate by `(tenantId, eventId)`.

## Current Transportation Events

| Event | Status | Producer | Current payload | Known consumers | Tenant gap |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `VehicleReadingRecorded` | NO_CONSUMER | Fleet | `readingId`, `vehicleId`, `readingType`, `value`, `unit`, `sourceType`, `sourceReferenceId`, `recordedAt`, `receivedAt` | None in production | No `tenantId` or standard envelope; unchanged because unused |
| `VehicleReadingCorrected` | NO_CONSUMER | Fleet | `correctionReadingId`, `originalReadingId`, `vehicleId`, `readingType`, `correctedValue`, `actorId`, `correctedAt` | None in production | No `tenantId` or standard envelope; unchanged because unused |
| `VehicleMeterResetRecorded` | NO_CONSUMER | Fleet | `resetId`, `vehicleId`, `readingType`, `fromEpoch`, `toEpoch`, `lastReadingValue`, `newMeterValue`, `effectiveAt` | None in production | No `tenantId` or standard envelope; unchanged because unused |
| `RouteDisruptionCreatedEvent` | NO_CONSUMER | Routing | `disruptionId`, `routeId`, `disruptionType`, `severity`, `detourRouteId`, `effectiveFrom`, `effectiveUntil` | None in production | No `tenantId` or standard envelope; unchanged because unused |
| `RouteDisruptionResolvedEvent` | NO_CONSUMER | Routing | `disruptionId`, `routeId`, `resolvedAt`, `resolvedBy` | None in production | No `tenantId` or standard envelope; unchanged because unused |
| `DeliveryRiderCreatedEvent` | NO_CONSUMER | Delivery | `tenantId`, `riderId`, `driverId`, `riderCode`, `primaryZoneId`, `riderType`, `transportMode`, `createdAt`, `actor` | None in production | Tenant-scoped but not a canonical envelope; unchanged because unused |
| `DeliveryRiderAssignedEvent` | LOCAL_AFTER_COMMIT | Delivery | `tenantId`, `deliveryOrderId`, `riderId`, `assignmentId`, `isOverride`, `assignedAt`, `actor` | Delivery ETA cache invalidation | Local Tenant-scoped event; eviction is idempotent |
| `DeliveryRiderReassignedEvent` | LOCAL_AFTER_COMMIT | Delivery | `tenantId`, `deliveryOrderId`, `previousRiderId`, `newRiderId`, `assignmentId`, `isOverride`, `reassignedAt`, `actor` | Delivery ETA cache invalidation | Local Tenant-scoped event; eviction is idempotent |
| `DeliveryRiderUnassignedEvent` | LOCAL_AFTER_COMMIT | Delivery | `tenantId`, `deliveryOrderId`, `riderId`, `unassignedAt`, `actor` | Delivery ETA cache invalidation | Local Tenant-scoped event; eviction is idempotent |
| `OperationalNotificationEvent` | LOCAL_AFTER_COMMIT | Multiple transportation producers and durable Delivery bridge | canonical envelope plus `severity`, `title`, `message`, `metadata` payload | Notification rule engine | Explicit Tenant/version/aggregate identity; local delivery is sufficient because Notification persists execution/attempts |

`OperationalNotificationEvent` severities are `INFO`, `WARNING`, and `CRITICAL`. Its event-type catalogue is application-owned and must be reviewed in source before adding a producer. Notification infrastructure enriches a missing `tenantId` from the authoritative execution context at publication time; it never accepts client Tenant authority. Version `1` is the current local schema. Existing source constructors remain compatible, but every event delivered through the production publisher contains Tenant identity.

## P0-07 Local Event Delivery Semantics

- Aggregate/application state and invariant-supporting history, allocation, stock, dispatch, meter, and delivery writes remain in their owning module's single ACID transaction.
- Spring-local secondary reactions are registered against the active transaction and emitted only after a successful commit. A rollback emits nothing.
- A local consumer exception is logged after commit and cannot roll back or misrepresent the already-committed primary operation.
- Notification rule execution deduplicates by its stable execution key derived from event, rule, channel, and recipient. Email/SMS delivery is independently claimed and retried from database-backed Notification records with a stable attempt idempotency key.
- Delivery ETA cache invalidation is a local after-commit reaction. Repeated eviction is intentionally idempotent.
- Ordering is local publication order only; consumers must not assume global ordering. Where order matters, it is scoped to one aggregate and its version/time facts.
- Events not selected for P1-01 durable handling remain non-durable across process failure and must not be described as guaranteed integration delivery. No Kafka or RabbitMQ infrastructure is approved.

## P1-01 Consumer Inventory and Durability Decision

Production-source inspection identified 32 event classes or persisted event records: 22 `NO_CONSUMER`, zero `LOCAL_EPHEMERAL`, nine `LOCAL_AFTER_COMMIT`, one `DURABLE_INTERNAL_REQUIRED`, and zero approved `EXTERNAL_INTEGRATION_CANDIDATE`. The complete publisher/consumer/classification matrix is maintained in the application architecture record `docs/architecture/P1-01-EVENT-CONTRACT-DURABILITY-AND-ENVELOPE-HARDENING.md`.

Only two event families were modernized. `OperationalNotificationEvent` now implements the canonical version-1 envelope but remains local after-commit because Notification already persists and retries channel execution. The five-type `DeliveryCustomerNotificationEvent` family implements the canonical envelope and is the sole `DURABLE_INTERNAL_REQUIRED` family because losing its Delivery-to-Notification handoff after a committed Delivery transaction would lose an accepted US-69 customer fact. The other 30 event classes/records were not gratuitously rewritten.

Flyway V60 creates the shared-technical `integration_outbox_event` table. Durable insertion participates in the originating transaction; rollback produces no row and insert/serialization failure fails that transaction. A Tenant-aware scheduler later claims at most 50 due rows per Tenant with row locking and a five-minute lease, reconstructs trusted Tenant context, and invokes the registered consumer outside the originating transaction. Retries reuse the original `eventId`, are bounded to five claims with backoff, and end as `FAILED`; unsupported versions end deterministically as `UNSUPPORTED`.

The producer uniqueness key is `(tenant_id,event_id,consumer_name)`. Notification's existing Tenant-scoped execution key `(eventId,rule,channel,normalizedRecipient)` is the consumer inbox/dedupe boundary, so P1-01 adds no duplicate inbox table. Delivery is `AT_LEAST_ONCE`, never exactly-once; a crash after consumer acceptance but before outbox acknowledgement can replay the same logical event. There is no global ordering promise. Rows are claimed by occurrence/creation time within one Tenant, but consumers must use aggregate facts and idempotency rather than depend on delivery order.

The durable payload is an explicit, deterministic, 32-KiB-bounded allowlist. It must never contain raw address/contact data, credentials, provider secrets, POD evidence, Rider private/medical data, or any US-70 raw token, token hash, magic-link URL, or access code. Published rows are retained for at least 30 days and failed/unsupported rows for at least 90 days; any purge must be a separately approved Tenant-qualified maintenance action.

## US-69 Delivery Customer Notification Events

Status for every contract in this section: `ACCEPTED_US69`, hardened by `P1-01`. Producer: Delivery. Consumer: Notification through the System integration bridge. Version: 1. Transport: atomic shared database outbox followed by Tenant-aware background delivery outside the originating transaction. Delivery semantics: at-least-once with crash recovery; never exactly-once. Ordering is not guaranteed globally; consumers use event time/aggregate facts and must not depend on claim order. Producer replay is deduplicated by `(tenantId,eventId,consumerName)`, followed by the Notification execution key `(tenantId,eventId,ruleId,channel,normalizedRecipient)`. Security classification is internal operational data. Outbox retention follows the P1-01 policy above and resulting Notification audit records follow Notification retention. Version 1 carries no separate correlation/causation identifiers; `eventId` is the stable trace identity, and adding such optional fields is backward compatible. Final acceptance verified that `DELIVERY_OUT_FOR_DELIVERY` is emitted only from committed Batch `DISPATCHED`: readiness emits zero customer events, dispatch emits exactly one event per active member, removed members are excluded, and rollback emits zero events.

Common envelope:

```json
{
  "eventId": "UUID",
  "eventType": "String",
  "tenantId": "UUID",
  "occurredAt": "OffsetDateTime",
  "version": 1,
  "aggregateType": "DELIVERY_ORDER",
  "aggregateId": "UUID",
  "payload": {}
}
```

Exact version-1 payloads:

| `eventType` | Trigger | Exact `payload` fields |
| :--- | :--- | :--- |
| `DELIVERY_OUT_FOR_DELIVERY` | US-66 commits batch `DISPATCHED`; one event per active member order | `customerId: UUID`, `deliveryNumber: String`, `status: "OUT_FOR_DELIVERY"`, `actor: String` |
| `DELIVERY_ETA_RISK_CHANGED` | US-67 order calculation changes/initially observes current SLA as `AT_RISK` or `LATE` | `customerId: UUID`, `deliveryNumber: String`, `estimatedArrivalAt: OffsetDateTime`, `slaStatus: "AT_RISK" | "LATE"`, `actor: String` |
| `DELIVERY_COMPLETED` | US-57 commits POD finalization and order `DELIVERED` | `customerId: UUID`, `deliveryNumber: String`, `status: "DELIVERED"`, `completedAt: OffsetDateTime`, `actor: String` |
| `DELIVERY_FAILED_ATTEMPT_RECORDED` | US-59 commits a failed attempt | `customerId: UUID`, `deliveryNumber: String`, `status: "FAILED_ATTEMPT"`, `failureDisposition: "REDELIVERY_ELIGIBLE" | "RETURN_TO_BASE_REQUIRED" | "ESCALATED"`, `actor: String` |
| `DELIVERY_REDELIVERY_SCHEDULED` | US-60 commits schedule/reschedule | `customerId: UUID`, `deliveryNumber: String`, `status: "CONFIRMED"`, `scheduleId: UUID`, `scheduledWindowStart: OffsetDateTime`, `scheduledWindowEnd: OffsetDateTime`, `actor: String` |

`aggregateId` is the Delivery Order ID. Tenant/event/time/version and payload values are produced from trusted committed Delivery state. Phone, email, message body, Customer/Rider aggregate, delivery instructions, free-text failure reason, coordinates, OTP/access secret, provider data, and credentials are prohibited. Notification maps these facts into its existing `OperationalNotificationEvent` and US-77 catalogue; it does not query or recalculate ETA. The ETA rule uses `slaStatus` as its milestone with a 1,440-minute suppression window so a cache-empty restart cannot immediately duplicate the same risk notice. `AT_RISK` to `LATE` remains a distinct milestone.

The implementation enforces the exact common/event-specific payload-key sets and rejects missing, blank, or additional fields. P1-01 persists the frozen Delivery fact in the originating transaction and invokes the bridge later through the durable worker; provider/consumer failure remains isolated from committed Delivery state. Duplicate delivery is safe through Notification's execution key. Final US-69 acceptance passed full Maven 1,223/0/0/15, architecture 42/42, and real PostgreSQL-backed Chromium 7/7; P1-01 hardening later passed full Maven 1,250/0/0/15 and architecture 45/45. US-69 remains complete.

## US-73 Controlled-Sandbox Contract (Implemented and Accepted)

`US73_PLATFORM_PROBE_V1` is the only event contract approved for the US-73 executable adapter. It is a non-business acceptance fact produced by an Integration-owned acceptance coordinator and consumed by the Integration handler named `integration-outbound-exchange` through the shared P1-01 outbox. It does not represent ERP, accounting, CRM, HRMS, fuel-vendor, telematics, payment, insurance, DMS, API, webhook, or file-import interoperability.

| Envelope field | Frozen value / rule |
| :--- | :--- |
| `tenantId` | Trusted active Tenant UUID; never accepted from payload |
| `eventId` | Stable UUID reused for retries/replay |
| `eventType` | `US73_PLATFORM_PROBE_V1` |
| `eventVersion` | `1` |
| `occurredAt` | Trusted committed UTC instant |
| `aggregateType` | `INTEGRATION_CONFIGURATION` |
| `aggregateId` | Same-Tenant Integration configuration UUID |
| `payload` | Exactly `probeId: UUID`, `probeType: "CONTROLLED_SANDBOX"`, and `sequence: long` |

Classification is `INTERNAL_OPERATIONAL_NON_SENSITIVE`; payloads are capped at 32 KiB and additional fields are rejected. The implemented consumer accepts the event idempotently using `(tenant_id, configuration_id, source_event_id, mapping_version_id)`, snapshots the immutable mapping version/hash, and then owns the external-delivery lifecycle. P1-01 delivery to Integration and external file delivery are both at-least-once; neither is globally ordered or exactly-once. V61 and the controlled-sandbox suite implement and accept this contract as `IMPLEMENTED_ACCEPTED_US73`; no business producer or vendor ecosystem is thereby approved.

## US-78 Operational Exception Contracts (Accepted, Complete)

### `OPERATIONAL_EXCEPTION_FACT_V1`

Status: `COMPLETE_US78`; Fuel US-38 producer is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING`. Active producers are Routing, Delivery, and authorized Fuel exception handoffs. Consumer: Operations handler `operations-exception-intake`. Transport: atomic P1-01 shared outbox. Semantics: at-least-once, no global ordering, producer dedupe `(tenantId,eventId,consumerName)`, consumer dedupe `UNIQUE (tenant_id,source_event_id)`. Security classification: minimized internal operational data. P1-01 transport retention applies; Operations case retention is separately governed. `eventId` is the stable source occurrence and `correlationId` is optional trace context, not case merging.

Envelope uses `eventType: OPERATIONAL_EXCEPTION_FACT_V1`, `version: 1`, source `aggregateType`, and source `aggregateId`. Exact payload:

```json
{
  "sourceModule": "ROUTING | DELIVERY | FUEL",
  "sourceType": "registered string <= 80",
  "sourceId": "UUID",
  "severityCandidate": "LOW | MEDIUM | HIGH | CRITICAL",
  "categoryCandidate": "OPERATIONAL | SAFETY | COMPLIANCE | CUSTOMER | FINANCIAL | TECHNICAL | SECURITY",
  "summaryCode": "registered string <= 80",
  "safeMetadata": {"allowListedKey": "bounded value"},
  "correlationId": "optional string <= 128"
}
```

US-38 implements the Fuel allow-list as source types `SUSPECTED_FUEL_LOSS`, `INCORRECT_READING`, `SUDDEN_PRICE_CHANGE`, `EMERGENCY_REFUEL`, `FUEL_CARD_POLICY_DEVIATION`, and `NEGATIVE_BUNKER_BALANCE`; summary code `FUEL_EXCEPTION_ESCALATED`; and safe metadata keys `fuelExceptionId`, `sourceType`, and `sourceId`. The envelope `sourceId` is the Fuel exception UUID and the stable `eventId` is the immutable handoff UUID reused on retry. Category candidates are OPERATIONAL for suspected loss/negative balance, TECHNICAL for incorrect reading, FINANCIAL for sudden price/emergency refuel, and SECURITY for card policy deviation. Fuel publication is `DURABLE_INTERNAL_REQUIRED` only after an authorized US-38 handoff and reuses the accepted shared outbox and Operations dedupe.

`safeMetadata` permits at most 20 allow-listed entries, 64-character keys, 256-character values, and a canonical payload no larger than 4 KiB. Tenant and occurrence time come from the trusted envelope. Entity serialization, free-form source evidence, addresses, contact data, credentials, OTPs, POD, medical/disciplinary data, full GPS tracks, financial documents, provider payloads, and arbitrary metadata keys are prohibited.

Implemented source allow lists are exact: Routing types `ROAD_CLOSURE`, `ACCIDENT`, `WEATHER`, `RESTRICTION`, summary `ROUTE_DISRUPTION_CREATED`, and metadata `routeId`, `detourRouteId`, `effectiveFrom`, `effectiveUntil`; Delivery types `DAMAGED_DELIVERY`, `WRONG_ADDRESS`, `PARTIAL_DELIVERY`, `OTP_MISMATCH`, `RECIPIENT_REFUSAL`, summary `DELIVERY_EXCEPTION_CREATED`, and metadata `deliveryOrderId`, `deliveryAttemptId`. Unknown source/type/summary/metadata combinations fail closed.

### `OPERATIONAL_EXCEPTION_ESCALATED_V1`

Status: `COMPLETE_US78` after independent final acceptance. Producer: Operations. Consumer: Notification/US-77 through the system bridge. Transport: P1-01 shared outbox. Semantics: at-least-once; Operations prevents duplicate case/level publication and Notification uses its stable execution key for rule/channel/recipient delivery. Security classification: minimized internal operational data. No ordering beyond case level/time is promised.

Envelope uses `aggregateType: OPERATIONAL_EXCEPTION_CASE`, the case UUID as `aggregateId`, trusted envelope `occurredAt`, and exact payload fields: `caseReference: String`, `sourceModule: String`, `sourceType: String`, `sourceId: UUID`, `category: String`, `severity: String`, `escalationLevel: "L1" | "L2" | "L3"`, `slaStatus: "ON_TRACK" | "AT_RISK" | "BREACHED" | "MET"`, and optional `correlationId: String`. Notification owns recipients, channels, templates, quiet hours, suppression, attempts, provider retry, and delivery history.

## Proposed Suite Event Families

These are discovery-level integration points, not approved payload contracts:

| Producer | Proposed families | Likely consumers |
| :--- | :--- | :--- |
| Transportation | trip lifecycle, freight order, fuel issue/purchase, vehicle availability, proof of delivery | Finance, Inventory, Maintenance, Sales/CRM |
| Finance | invoice, payment, journal posting, budget availability, asset capitalization | Sales/CRM, Procurement, Projects, Transportation, Maintenance |
| HRM | employee engagement, driver qualification/availability, leave | Transportation, Projects, Maintenance |
| Inventory | stock reservation, issue, receipt, adjustment | Procurement, Maintenance, Projects, Transportation |
| Procurement | purchase order lifecycle and goods receipt expectation | Inventory, Finance, Maintenance |
| Projects | project lifecycle, budget allocation, time/cost posting | Finance, HRM, Procurement |
| Sales/CRM | customer/account and sales order lifecycle | Finance, Transportation, Inventory |
| Vehicle Maintenance | work order, maintenance hold/release, parts consumption | Transportation, Inventory, Procurement, Finance |

No proposed family may be consumed until its owner registers an exact versioned payload, security classification, ordering/idempotency semantics, retention, and producer/consumer tests here.
# US-46 Driver Payroll-Input Export (Accepted)

`DriverPayrollInputExportRequestedV1` is implemented as a canonical P1-01 durable envelope. Producer: Driver within Fleet. Consumer: Integration handler `integration-outbound-exchange`. Event type/business family: `DRIVER_PAYROLL_INPUT_V1`; version 1; aggregate type `DRIVER_PAYROLL_INPUT_BATCH`; classification `FINANCIAL_CONFIDENTIAL`; delivery at-least-once with no global ordering.

The event is written in the same transaction that releases an approved immutable batch. Its stable event ID is reused for the same release; Integration deduplicates by `(tenantId, configurationId, sourceEventId, mappingVersionId)`. Retry never changes the approved payload. Driver release and Integration external delivery are not a distributed transaction.

The canonical business payload properties are exactly `schemaVersion`, `batchId`, `batchType`, `periodStart`, `periodEndExclusive`, `cutoffAt`, `generatedAt`, `currency`, `totals`, and `drivers`. `totals` contains `tripEarnings`, `allowances`, `overtime`, `deductions`, and `provisionalNetInput`. Each `drivers[]` item contains `driverId`, `externalWorkerReference`, and `lines`; each line contains `lineId`, `tripId`, `tripNumber`, `category`, `reasonCode`, `quantity`, `unit`, `rate`, `amount`, `originalLineId`, and `sourceSnapshotHash`. Categories are exactly `TRIP_EARNING`, `ALLOWANCE`, `OVERTIME`, and `DEDUCTION`; units are exactly `TRIP`, `HOUR`, and `FIXED`. Batch type is `REGULAR` or `CORRECTION`. The P1-01 envelope carries event/Tenant/aggregate/correlation identity outside the business payload. Arrays use deterministic UUID order, monetary strings use scale 2, and the accepted 32-KiB limit fails closed without truncation.

Names, contact data, medical/licence/drug-test data, bank/tax/pension data, salary facts, credentials, raw notes, and unrestricted metadata are forbidden. Successful file delivery means `EXPORTED` only; it never means imported, reconciled, posted, settled, or paid. No inbound acknowledgement event is approved.

# US-47 Transport Billing Export (Implemented / Acceptance Pending)

`TransportBillingExportRequestedV1` is the frozen canonical P1-01 family. Producer: Billing. Consumer: Integration handler `integration-outbound-exchange`. Event type/business family: `TRANSPORT_BILLING_V1`; version 1; aggregate type `TRANSPORT_BILLING_RECORD`; classification `FINANCIAL_CONFIDENTIAL`; delivery at-least-once with no global ordering.

The stable event ID is reused for the same finalized export request and Integration deduplicates by its accepted Tenant/configuration/source-event/mapping-version identity. The business payload properties are exactly `schemaVersion`, `billingRecordId`, `billingNumber`, `recordType`, `customerId`, `currency`, `source`, `amounts`, `tax`, `costCentres`, `originalBillingRecordId`, and `finalizedAt`. `source` contains only type, ID, business number and SHA-256 snapshot hash. `amounts` contains base charge, surcharges, penalties, credit adjustments, subtotal, tax amount and total. `tax` contains supplied/not-supplied status and bounded category, jurisdiction, taxable amount, rate and exemption reference. Cost-centre items contain code and allocation percent. The P1-01 envelope supplies event/Tenant/type/version/aggregate/time/correlation/causation identity.

Customer/contact names, addresses, cargo detail, notes, credentials, bank/payment/account data, raw sources and unrestricted metadata are forbidden. The canonical UTF-8 JSON is deterministic and fails closed above the accepted 32-KiB bound. `EXPORTED` means controlled JSON file/hash delivery only; no inbound acknowledgement, tax-invoice issuance, posting, booking, settlement or payment event exists. The family is active in implementation through the shared outbox and controlled Integration exchange; real canonical file/hash evidence passes.

# US-48 Tracking Event Decision (Frozen; No Active Event)

Raw accepted telemetry is `LOCAL_STATE_ONLY`; US-48 must not publish one P1-01 or Spring event per packet. This protects the shared outbox and consumers from high-rate event storms. `VehicleTrackingStateChangedV1` is reserved as `NOT_ACTIVATED_NO_CURRENT_CONSUMER`: activation requires a registered US-49..55 consumer and exact payload review. When activated it uses the canonical P1-01 envelope, Tenant/event idempotency, at-least-once semantics and no global ordering, and emits only a freshness/connectivity/trust state change or at most one materially newer trusted position per Vehicle per minute. It may contain minimized Tenant/Vehicle, trusted position/time/accuracy and state facts; Driver/Customer identity, device secret/reference, raw payload and unrestricted provider metadata are forbidden. No second outbox is approved.

## VehicleGeofenceTransitionedV1 (US-49; Approved for Implementation, Not Active)

- **Owner / producer:** Tracking.
- **Consumer:** Notification only in US-49; Operations integration is NONE.
- **Envelope:** canonical P1-01 Tenant envelope; `eventId` is the immutable transition UUID.
- **Delivery:** durable outbox, at least once, consumer idempotency by Tenant + event ID, no global ordering claim.
- **Emission:** confirmed ENTERED/EXITED stable-state changes only; never one event per position. Initial state is silent.
- **Payload:** `geofenceId`, `vehicleId`, nullable `locationId`, `geofenceType`, `transition`, `severity`, `sourceTimestamp`, `definitionVersion`.
- **Privacy:** no coordinates, device/provider identity, credentials, raw payload, Driver PII or Customer data.
- **Identity:** transition identity is deterministic SHA-256 over Tenant, geofence, definition version, Vehicle, from/to state and confirming position.

This contract is frozen by `US-49-MANAGE-GEOFENCES-PRODUCT-DECISIONS-001` but remains `NOT_ACTIVE` until its implementation change set is verified. Unauthorized-zone confirmed entry uses `UNAUTHORIZED_ZONE_ENTERED` with HIGH severity and mandatory alert publication; it does not create an Operations exception in US-49.
