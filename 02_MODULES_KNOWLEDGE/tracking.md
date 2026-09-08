# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING`; technical closure is `FAIL / REMEDIATION_REQUIRED`. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts. V73, APIs, exact permissions, controlled-provider evidence and the minimal UI are implemented. The minimum security, retention, observability, audit and rebuild remediation is authorized under `US-48-LIVE-VEHICLE-TRACKING-TECHNICAL-REMEDIATION-AUTHORIZATION-001`. Accounting remains 72/87 with 15 remaining.

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

## Implemented persistence (V73)

Tracking owns `tracking_device`, `tracking_vehicle_device_assignment`, `tracking_position`, `tracking_vehicle_latest`, `tracking_ingest_nonce`, and `tracking_audit_event`. Every table is Tenant-owned. Device/association/latest same-module relationships are Tenant-consistent; `vehicle_id` is a logical Fleet reference without a physical cross-module FK. Tenant-leading indexes cover Vehicle/source time, device/source time, latest lookup, active associations, provider-message/dedupe identity, nonce expiry and audit time. `tracking_position` is trigger-enforced append-only and association history allows only its one-time close operation.

`tracking_device` stores UUID `id`, required indexed UUID `tenant_id`, external/provider/hardware references, ACTIVE/DISABLED lifecycle, registration actor/time, last-seen time, optimistic `version`, and created/updated timestamps. `tracking_vehicle_device_assignment` stores UUID identity/Tenant/device/logical Vehicle, required `effective_from`, optional exclusive `effective_to`, and creation actor/time. `tracking_position` stores immutable UUID/Tenant/device/logical Vehicle identity; provider/message/sequence/dedupe/hash facts; source/receipt timestamps; WGS84 coordinates and optional quality/engine/meter facts; trust/quality/order classifications; retention policy/version/optional retain-until; and safe JSON metadata. `tracking_vehicle_latest` stores the Tenant+Vehicle key, latest received/trusted position references, last receipt, policy version, updated time and optimistic version. `tracking_ingest_nonce` stores Tenant/provider/nonce hash and use/expiry times. `tracking_audit_event` stores Tenant/actor/action/target/safe detail/time without per-packet audit.

## Performance and acceptance

ARB initial scale assumption—not a source fact—is 10,000 active vehicles at one message/minute (about 167/s and 14.4M rows/day), with a 5x 15-minute burst. Closure targets >=200 accepted/s sustained, 1,000/s burst, latest p95 <=200ms and 24-hour single-Vehicle history-page p95 <=500ms on the acceptance environment with index-backed plans.

PostgreSQL acceptance must prove migration, Tenant constraints, association uniqueness/history, dedupe/conflict, immutable history, association-at-source-time, trusted/latest compare-and-set correctness, rollback atomicity, out-of-order behavior, retention metadata, nine deterministic races and query plans. Final real E2E additionally proves physical-device/provider time/accuracy, duplicate, delay/order, invalid position, stale/disconnect/reconnect, reassignment history, Tenant/RBAC and credential privacy.

## Authorized technical remediation

V74 is required and authorized as the next forward migration, but is not yet created. V1–V73 remain immutable. V74 is limited to a Tracking-owned trusted provider binding, binding-scoped replay protection and Tenant retention-policy persistence. It must not contain US-49..55 schema or an outbox.

Inbound Tenant authority will be derived from a globally unique opaque provider key ID resolved to an active Tracking-owned binding containing Tenant, bounded provider alias and an opaque credential reference. The secret is resolved only through Integration's published `IntegrationSecretResolver`. Provider alias may repeat across Tenants and never establishes authority. Caller Tenant headers/payloads cannot select or override Tenant. Authentication verifies the binding-derived credential, signed timestamp/body and binding-scoped nonce before resolving the device and source-time association solely inside the derived Tenant. All failures are sanitized and fail closed.

The current US-73 configuration schema is not extended and its tables/repositories are not accessed: its accepted `FILE_EXCHANGE / FILE_JSON_V1 / OUTBOUND` capability cannot represent inbound telematics. No telemetry packet traverses Integration exchange processing. No new public human API, permission or event is authorized; only the provider authentication header/canonical-signature contract changes before acceptance.

Retention remains external policy. An absent policy means no automatic purge and no age-only `TRACKING_POSITION_TOO_OLD`; greater-than-24-hour packets retain the existing LATE behavior. With a configured duration, timestamps before `receivedAt - duration` are too old, equality is accepted, and accepted history records policy/version/retain-until metadata.

Tracking must add an internal per-Tenant/Vehicle transactional rebuild of `tracking_vehicle_latest` from retained immutable history using the identical deterministic receipt/trust ordering as ingress. It is idempotent and has no public endpoint or foreign-table dependency. Missing metrics, sanitized management/provider/retention audit, and a Tracking contributor to the existing health surface are also authorized. Metrics exclude UUIDs, coordinates, nonce/signature/credential and person/customer data. One stale device cannot declare a provider outage.

Closure rerun requires the full signed-ingress negative matrix, exact freshness/connectivity and retention boundaries, deterministic PostgreSQL projection rebuild, V74 binding/nonce/Tenant constraints, the existing nine races, complete regression/static/frontend/browser gates and eventual physical provider/device evidence.

## Next task

`US-48-LIVE-VEHICLE-TRACKING-TECHNICAL-REMEDIATION-001`. Story completion accounting does not advance until independent final acceptance.
