# Global Event Contracts Registry

All domain events crossing module boundaries must be documented here with their JSON payload specifications.

Statuses: `ACTIVE_INTERNAL` exists in current source but is not yet a tenant-ready external contract; `PROPOSED` requires approval and implementation.

## Standard Domain Event Envelope

```json
{
  "eventId": "UUID",
  "eventType": "String",
  "tenantId": "UUID",
  "occurredAt": "2026-08-26T14:44:54Z",
  "version": 1,
  "payload": {}
}
```

Every registered cross-module event must use this envelope. Its contract entry must define the exact `eventType`, versioned payload schema, producer, aggregate identity where applicable, known consumers, delivery semantics, ordering, idempotency, security classification, retention, and correlation/causation behavior.

Compatibility rules: additive optional fields are backward compatible; renames, removals, semantic changes, type changes, and newly required fields require a new `version`. Consumers must ignore unknown fields and deduplicate by `(tenantId, eventId)`.

## Current Transportation Events

| Event | Status | Producer | Current payload | Known consumers | Tenant gap |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `VehicleReadingRecorded` | ACTIVE_INTERNAL | Fleet | `readingId`, `vehicleId`, `readingType`, `value`, `unit`, `sourceType`, `sourceReferenceId`, `recordedAt`, `receivedAt` | Fleet projections/integrations | No `tenantId` or standard envelope |
| `VehicleReadingCorrected` | ACTIVE_INTERNAL | Fleet | `correctionReadingId`, `originalReadingId`, `vehicleId`, `readingType`, `correctedValue`, `actorId`, `correctedAt` | Fleet projections/integrations | No `tenantId` or standard envelope |
| `VehicleMeterResetRecorded` | ACTIVE_INTERNAL | Fleet | `resetId`, `vehicleId`, `readingType`, `fromEpoch`, `toEpoch`, `lastReadingValue`, `newMeterValue`, `effectiveAt` | Fleet reading flow | No `tenantId` or standard envelope |
| `RouteDisruptionCreatedEvent` | ACTIVE_INTERNAL | Routing | `disruptionId`, `routeId`, `disruptionType`, `severity`, `detourRouteId`, `effectiveFrom`, `effectiveUntil` | Spring event adapters | No `tenantId` or standard envelope |
| `RouteDisruptionResolvedEvent` | ACTIVE_INTERNAL | Routing | `disruptionId`, `routeId`, `resolvedAt`, `resolvedBy` | Spring event adapters | No `tenantId` or standard envelope |
| `DeliveryRiderOnboardedEvent` | ACTIVE_INTERNAL | Delivery | `tenantId`, `riderId`, `driverId`, `riderCode`, `riderType`, `primaryZoneId`, `onboardedAt`, `actor` | Spring event adapters | Standard tenant envelope |
| `DeliveryRiderAssignedEvent` | ACTIVE_INTERNAL | Delivery | `tenantId`, `deliveryOrderId`, `riderId`, `assignmentId`, `isOverride`, `assignedAt`, `actor` | Spring event adapters | Standard tenant envelope |
| `DeliveryRiderReassignedEvent` | ACTIVE_INTERNAL | Delivery | `tenantId`, `deliveryOrderId`, `previousRiderId`, `newRiderId`, `assignmentId`, `isOverride`, `reassignedAt`, `actor` | Spring event adapters | Standard tenant envelope |
| `DeliveryRiderUnassignedEvent` | ACTIVE_INTERNAL | Delivery | `tenantId`, `deliveryOrderId`, `riderId`, `unassignedAt`, `actor` | Spring event adapters | Standard tenant envelope |
| `OperationalNotificationEvent` | ACTIVE_INTERNAL | Multiple transportation producers | `eventId`, `eventType`, `aggregateType`, `aggregateId`, `severity`, `title`, `message`, `occurredAt`, `metadata`, `tenantId`, `schemaVersion` | Notification | Tenant and schema identity are explicit; producer/correlation/causation remain absent because current local consumers do not use them |

`OperationalNotificationEvent` severities are `INFO`, `WARNING`, and `CRITICAL`. Its event-type catalogue is application-owned and must be reviewed in source before adding a producer. Notification infrastructure enriches a missing `tenantId` from the authoritative execution context at publication time; it never accepts client Tenant authority. Version `1` is the current local schema. Existing source constructors remain compatible, but every event delivered through the production publisher contains Tenant identity.

## P0-07 Local Event Delivery Semantics

- Aggregate/application state and invariant-supporting history, allocation, stock, dispatch, meter, and delivery writes remain in their owning module's single ACID transaction.
- Spring-local secondary reactions are registered against the active transaction and emitted only after a successful commit. A rollback emits nothing.
- A local consumer exception is logged after commit and cannot roll back or misrepresent the already-committed primary operation.
- Notification rule execution deduplicates by its stable execution key derived from event, rule, channel, and recipient. SMTP delivery is independently claimed and retried from database-backed Notification records with a stable attempt idempotency key.
- Delivery ETA cache invalidation is a local after-commit reaction. Repeated eviction is intentionally idempotent.
- Ordering is local publication order only; consumers must not assume global ordering. Where order matters, it is scoped to one aggregate and its version/time facts.
- The current Spring event path is not durable across process failure. It must not be described as guaranteed integration delivery. A database outbox/inbox is required before any external or independently retryable consumer is approved; no Kafka or RabbitMQ infrastructure is approved by P0-07.

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
