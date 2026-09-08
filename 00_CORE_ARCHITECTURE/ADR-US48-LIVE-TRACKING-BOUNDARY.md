# ADR — US-48 Live Vehicle Tracking Boundary

**Status:** ACCEPTED / PRODUCT_DECISIONS_FROZEN
**Date:** 2026-09-08
**Story:** US-48 Track Vehicles Live

## Decision

Create a dedicated top-level `tracking` bounded context inside the modular monolith. Tracking owns a narrow device-reference registry, effective-dated device/Vehicle association, provider-neutral telemetry ingestion, immutable normalized position history, dedupe/order/trust policy, last-received and last-trusted projections, freshness/connectivity, retention metadata, and safe operator reads.

Fleet retains Vehicle master/lifecycle. Trip retains assignment/execution. Routing retains planned routes. Organization retains sites. Tracking stores only same-Tenant logical references and uses published contracts; it never accesses foreign repositories/tables or adds physical cross-module foreign keys.

## Provider and integration boundary

Core is `PROVIDER_NEUTRAL`. The Phase-1 external adapter contract is signed HTTPS JSON at `/api/integration/v1/tracking/positions`, with single or bounded batch messages. Tracking terminates and normalizes high-rate ingress. US-73 may govern provider configuration and opaque credential references after an explicit telematics extension, but it does not transport telemetry packets or own Tracking state. MQTT, raw TCP, polling-provider, file import and vendor SDKs are deferred.

Implementation and technical closure may use a controlled protocol fixture. Final acceptance requires `REAL_DEVICE_REAL_PROVIDER`: physical GPS hardware and real provider-generated payloads. Without it, acceptance is `BLOCKED_BY_EXTERNAL_SYSTEM` and simulator evidence is not relabelled.

## State, storage and delivery

WGS84 source facts preserve distinct source and receipt instants. Accepted history is append-only. Exact replay is idempotent; conflict is fail-closed; older valid packets remain history but cannot replace newer trusted state. Default freshness is LIVE through 60 seconds, RECENT through 5 minutes, STALE thereafter, and UNKNOWN without trusted history. Connectivity is derived separately from successful provider receipt age.

Ordinary PostgreSQL owns history and latest projections. PostGIS, TimescaleDB, Kafka, Redis and partitioning are deferred pending measured need. Raw provider payload is not retained; normalized minimized facts and bounded safe metadata only. Retention duration is `EXTERNAL_POLICY`, no public purge route exists, and latest can be rebuilt from retained history.

Raw positions are local state and do not enter P1-01 per packet. A future coalesced `VehicleTrackingStateChangedV1` is frozen but inactive until a real consumer is registered; when activated it uses the shared durable envelope and emits only state changes or a bounded material-update cadence.

## Security and UI

Human permissions are `TRACKING_VIEW`, `TRACKING_HISTORY_VIEW`, and `TRACKING_DEVICE_MANAGE`. Provider ingress uses signed requests, TLS, timestamp/nonce replay protection and an opaque secret reference; human JWT is not device authentication. Tenant comes from authenticated membership for humans and trusted provider/device association for ingress, never payload authority.

Precise coordinates are sensitive Tenant-owned operational data. Customers receive none through US-48; ordinary responses omit Driver identity and mask device references. US-48 uses 15-second visible polling with bounded backoff. It supplies a minimal accessible map/status/history/device foundation, not the US-54 dashboard or US-49..55 detectors.

## Consequences

US-48 implementation is split into device/association, ingestion/trust, history/projection, security, API/frontend, real-provider/device, and performance/concurrency change sets. Current Flyway head V72 is not reserved. Story accounting remains 72/87 until independent final acceptance.
