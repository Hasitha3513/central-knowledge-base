# Tracking Module

## Status and scope

US-48 is `PRODUCT_DECISIONS_FROZEN / IMPLEMENTATION_NOT_STARTED`. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts. No production code, migration, API, permission or event is implemented by this decision. Current Flyway head remains V72; accounting remains 72/87 with 15 remaining.

Phase 1 owns a narrow `TrackingDevice` reference registry, effective-dated one-device/one-Vehicle active association, immutable normalized `PositionEvent` history, ingestion dedupe/conflict/order/trust, last-received and last-trusted projections, freshness/connectivity, retention metadata, safe queries, provider adapter health and minimal operator UI.

Fleet owns Vehicle master; Trip owns assignment/execution; Routing owns planned routes; Organization owns sites; Integration may govern configuration/opaque credentials but does not carry high-rate packets. Logical same-Tenant references only. US-49..55 own geofence, speed, idle, deviation, replay, dashboard and GPS-edge detectors.

## Frozen contracts

- Provider strategy: `PROVIDER_NEUTRAL`; signed HTTPS JSON Tracking ingress, maximum 500 messages/1 MiB; no vendor SDK.
- Acceptance: fixture for implementation/closure; physical device plus real provider payloads is mandatory for final acceptance or status is `BLOCKED_BY_EXTERNAL_SYSTEM`.
- Coordinates: WGS84, finite latitude `[-90,90]` and longitude `[-180,180]`; distinct immutable source and receipt UTC instants.
- Accuracy: optional non-negative metres; unknown is explicit; over 1,000 m cannot advance trusted position.
- Freshness: LIVE <=60s, RECENT <=5m, STALE >5m, UNKNOWN without trusted position. Connectivity derives separately as CONNECTED/DEGRADED/OFFLINE/UNKNOWN from receipt age.
- Clock/order: tolerate and mark up to 120s future skew; greater future skew is untrusted. Older arrivals remain ordered history, do not replace trusted latest; >24h at receipt is late, and pre-retention packets are too old.
- Identity: provider message identity where stable, otherwise canonical SHA-256 over Tenant/device/source-time/coordinate/optional sequence. Exact replay is idempotent; different payload under one identity conflicts.
- History: append-only normalized facts; raw provider payload is not retained. Retention duration is `EXTERNAL_POLICY`; policy/version/retain-until metadata is required; no public purge API.
- Storage: standard PostgreSQL history/latest/dedupe and association tables. PostGIS, TimescaleDB, Kafka, Redis and partitioning are deferred pending measured evidence.
- UI: minimal list/map point/latest/last-known/freshness/connectivity/accuracy/history/device association; 15-second visible polling with 30/60-second failure backoff; no US-54 dashboard.
- Privacy: precise location requires same-Tenant `TRACKING_VIEW`; history also requires `TRACKING_HISTORY_VIEW`; no Customer exposure, Driver profile, raw payload or credential exposure.
- P1-01: no per-packet event. `VehicleTrackingStateChangedV1` is inactive until a consumer is approved and is then coalesced/state-change-only through the shared durable outbox.

## Expected persistence (not implemented)

Expected Tracking-owned tables are `tracking_device`, `tracking_vehicle_device_assignment`, `tracking_position`, `tracking_vehicle_latest`, and either `tracking_ingest_identity` or equivalent database-enforced position identity. Every row is Tenant-owned; same-module relationships are Tenant-consistent and foreign-module UUIDs have no physical FK. Tenant-leading indexes cover Vehicle/source time, device/source time, latest lookup, active association and dedupe identity.

## Performance and acceptance

ARB initial scale assumption—not a source fact—is 10,000 active vehicles at one message/minute (about 167/s and 14.4M rows/day), with a 5x 15-minute burst. Closure targets >=200 accepted/s sustained, 1,000/s burst, latest p95 <=200ms and 24-hour single-Vehicle history-page p95 <=500ms on the acceptance environment with index-backed plans.

PostgreSQL acceptance must prove migration, Tenant constraints, association uniqueness/history, dedupe/conflict, immutable history, association-at-source-time, trusted/latest compare-and-set correctness, rollback atomicity, out-of-order behavior, retention metadata, nine deterministic races and query plans. Final real E2E additionally proves physical-device/provider time/accuracy, duplicate, delay/order, invalid position, stale/disconnect/reconnect, reassignment history, Tenant/RBAC and credential privacy.

## Next task

`US-48-LIVE-VEHICLE-TRACKING-IMPLEMENTATION-001` using controlled change sets CS01 through CS07. Story completion accounting does not advance until independent final acceptance.
