# Integration Event Contracts

Statuses: `ACTIVE_INTERNAL` exists in current source but is not yet a tenant-ready external contract; `PROPOSED` requires approval and implementation.

## Standard Envelope (Required for New External Events)

```json
{
  "eventId": "uuid",
  "eventType": "module.aggregate.event.v1",
  "eventVersion": 1,
  "tenantId": "uuid",
  "producer": "module_name",
  "aggregateType": "string",
  "aggregateId": "uuid",
  "occurredAt": "RFC-3339 UTC timestamp",
  "correlationId": "string",
  "causationId": "string|null",
  "payload": {}
}
```

Compatibility rules: additive optional fields are backward compatible; renames, removals, semantic changes, type changes, and newly required fields require a new event version. Consumers must ignore unknown fields and deduplicate `(tenantId,eventId)`.

## Current Transportation Events

| Event | Status | Producer | Current payload | Known consumers | Tenant gap |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `VehicleReadingRecorded` | ACTIVE_INTERNAL | Fleet | `readingId`, `vehicleId`, `readingType`, `value`, `unit`, `sourceType`, `sourceReferenceId`, `recordedAt`, `receivedAt` | Fleet projections/integrations | No `tenantId` or standard envelope |
| `VehicleReadingCorrected` | ACTIVE_INTERNAL | Fleet | `correctionReadingId`, `originalReadingId`, `vehicleId`, `readingType`, `correctedValue`, `actorId`, `correctedAt` | Fleet projections/integrations | No `tenantId` or standard envelope |
| `VehicleMeterResetRecorded` | ACTIVE_INTERNAL | Fleet | `resetId`, `vehicleId`, `readingType`, `fromEpoch`, `toEpoch`, `lastReadingValue`, `newMeterValue`, `effectiveAt` | Fleet reading flow | No `tenantId` or standard envelope |
| `RouteDisruptionCreatedEvent` | ACTIVE_INTERNAL | Routing | `disruptionId`, `routeId`, `disruptionType`, `severity`, `detourRouteId`, `effectiveFrom`, `effectiveUntil` | Spring event adapters | No `tenantId` or standard envelope |
| `RouteDisruptionResolvedEvent` | ACTIVE_INTERNAL | Routing | `disruptionId`, `routeId`, `resolvedAt`, `resolvedBy` | Spring event adapters | No `tenantId` or standard envelope |
| `OperationalNotificationEvent` | ACTIVE_INTERNAL | Multiple transportation producers | `eventId`, `eventType`, `aggregateType`, `aggregateId`, `severity`, `title`, `message`, `occurredAt`, `metadata` | Notification | No `tenantId`, version, producer, correlation, or causation fields |

`OperationalNotificationEvent` severities are `INFO`, `WARNING`, and `CRITICAL`. Its event-type catalogue is application-owned and must be reviewed in source before adding a producer.

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
