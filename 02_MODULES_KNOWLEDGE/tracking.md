# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING`; V74 technical remediation is complete and independent technical closure passes. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts. The trusted provider/Tenant authority, retention, observability, audit and rebuild remediation is implemented. Accounting remains 72/87 with 15 remaining; physical-device and real-provider final acceptance is still required.

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

## Implemented persistence (V73 + V74 remediation)

Tracking owns `tracking_device`, `tracking_vehicle_device_assignment`, `tracking_position`, `tracking_vehicle_latest`, `tracking_ingest_nonce`, and `tracking_audit_event`. Every table is Tenant-owned. Device/association/latest same-module relationships are Tenant-consistent; `vehicle_id` is a logical Fleet reference without a physical cross-module FK. Tenant-leading indexes cover Vehicle/source time, device/source time, latest lookup, active associations, provider-message/dedupe identity, nonce expiry and audit time. `tracking_position` is trigger-enforced append-only and association history allows only its one-time close operation.

`tracking_device` stores UUID `id`, required indexed UUID `tenant_id`, external/provider/hardware references, ACTIVE/DISABLED lifecycle, registration actor/time, last-seen time, optimistic `version`, and created/updated timestamps. `tracking_vehicle_device_assignment` stores UUID identity/Tenant/device/logical Vehicle, required `effective_from`, optional exclusive `effective_to`, and creation actor/time. `tracking_position` stores immutable UUID/Tenant/device/logical Vehicle identity; provider/message/sequence/dedupe/hash facts; source/receipt timestamps; WGS84 coordinates and optional quality/engine/meter facts; trust/quality/order classifications; retention policy/version/optional retain-until; and safe JSON metadata. `tracking_vehicle_latest` stores the Tenant+Vehicle key, latest received/trusted position references, last receipt, policy version, updated time and optimistic version. `tracking_ingest_nonce` stores Tenant/provider/nonce hash and use/expiry times. `tracking_audit_event` stores Tenant/actor/action/target/safe detail/time without per-packet audit.

### Table: `tracking_provider_binding`

- **Purpose:** Tracking-owned trusted provider configuration and Tenant authority.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Binding ID |
| `tenant_id` | UUID | NO | - | Tenant scope; immutable | Trusted Tenant authority |
| `provider_key_id` | VARCHAR(160) | NO | - | GLOBALLY UNIQUE; immutable | Opaque non-secret lookup ID |
| `provider_alias` | VARCHAR(80) | NO | - | Tenant aliases may repeat | Trusted provider alias |
| `credential_reference` | VARCHAR(160) | NO | - | Opaque; no plaintext secret | `IntegrationSecretResolver` reference |
| `lifecycle` | VARCHAR(16) | NO | - | CHECK `ACTIVE`,`DISABLED` | Authentication lifecycle |
| `created_at`, `updated_at` | TIMESTAMPTZ | NO | - | - | Audit timestamps |
| `created_by`, `updated_by` | UUID | NO | - | Logical actor references | Audit actors |
| `version` | BIGINT | NO | `0` | Optimistic version | Mutation version |

Indexes: global unique `provider_key_id`; `(tenant_id,lifecycle,provider_alias,id)`. A trigger prevents Tenant or provider-key reassignment.

### Table: `tracking_provider_ingest_nonce`

- **Purpose:** Binding-scoped signed-ingress replay protection.
- **Primary Key:** (`provider_binding_id`, `nonce_hash`)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | Tenant-consistent FK with binding | Trusted scope |
| `provider_binding_id` | UUID | NO | - | FK with `tenant_id` → provider binding | Replay authority |
| `nonce_hash` | CHAR(64) | NO | - | Composite PRIMARY KEY | SHA-256 nonce hash |
| `used_at` | TIMESTAMPTZ | NO | - | - | Reservation time |
| `expires_at` | TIMESTAMPTZ | NO | - | Indexed with Tenant/binding | Expiry time |

### Table: `tracking_retention_policy`

- **Purpose:** One current external retention policy per Tenant.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, unique and indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with Tenant | Policy ID |
| `tenant_id` | UUID | NO | - | UNIQUE | Tenant scope |
| `retention_duration_seconds` | BIGINT | NO | - | CHECK > 0 | Retention duration |
| `policy_version` | VARCHAR(40) | NO | - | - | External policy version |
| `effective_at` | TIMESTAMPTZ | NO | - | Tenant-leading effective index | Effective instant |
| `created_at`, `updated_at` | TIMESTAMPTZ | NO | - | - | Audit timestamps |
| `created_by`, `updated_by` | UUID | NO | - | Logical actor references | Audit actors |
| `version` | BIGINT | NO | `0` | Optimistic version | Mutation version |

## Performance and acceptance

ARB initial scale assumption—not a source fact—is 10,000 active vehicles at one message/minute (about 167/s and 14.4M rows/day), with a 5x 15-minute burst. Closure targets >=200 accepted/s sustained, 1,000/s burst, latest p95 <=200ms and 24-hour single-Vehicle history-page p95 <=500ms on the acceptance environment with index-backed plans.

PostgreSQL acceptance must prove migration, Tenant constraints, association uniqueness/history, dedupe/conflict, immutable history, association-at-source-time, trusted/latest compare-and-set correctness, rollback atomicity, out-of-order behavior, retention metadata, nine deterministic races and query plans. Final real E2E additionally proves physical-device/provider time/accuracy, duplicate, delay/order, invalid position, stale/disconnect/reconnect, reassignment history, Tenant/RBAC and credential privacy.

## Completed technical remediation

V74 is implemented as the current head; V1–V73 remain immutable. It contains only the Tracking-owned trusted provider binding, binding-scoped replay protection and Tenant retention-policy persistence. It contains no US-49..55 schema or outbox.

Inbound Tenant authority will be derived from a globally unique opaque provider key ID resolved to an active Tracking-owned binding containing Tenant, bounded provider alias and an opaque credential reference. The secret is resolved only through Integration's published `IntegrationSecretResolver`. Provider alias may repeat across Tenants and never establishes authority. Caller Tenant headers/payloads cannot select or override Tenant. Authentication verifies the binding-derived credential, signed timestamp/body and binding-scoped nonce before resolving the device and source-time association solely inside the derived Tenant. All failures are sanitized and fail closed.

The current US-73 configuration schema is not extended and its tables/repositories are not accessed: its accepted `FILE_EXCHANGE / FILE_JSON_V1 / OUTBOUND` capability cannot represent inbound telematics. No telemetry packet traverses Integration exchange processing. No new public human API, permission or event is authorized; only the provider authentication header/canonical-signature contract changes before acceptance.

Retention remains external policy. An absent policy means no automatic purge and no age-only `TRACKING_POSITION_TOO_OLD`; greater-than-24-hour packets retain the existing LATE behavior. With a configured duration, timestamps before `receivedAt - duration` are too old, equality is accepted, and accepted history records policy/version/retain-until metadata.

Tracking must add an internal per-Tenant/Vehicle transactional rebuild of `tracking_vehicle_latest` from retained immutable history using the identical deterministic receipt/trust ordering as ingress. It is idempotent and has no public endpoint or foreign-table dependency. Missing metrics, sanitized management/provider/retention audit, and a Tracking contributor to the existing health surface are also authorized. Metrics exclude UUIDs, coordinates, nonce/signature/credential and person/customer data. One stale device cannot declare a provider outage.

Closure rerun requires the full signed-ingress negative matrix, exact freshness/connectivity and retention boundaries, deterministic PostgreSQL projection rebuild, V74 binding/nonce/Tenant constraints, the existing nine races, complete regression/static/frontend/browser gates and eventual physical provider/device evidence.

Fresh evidence is security/boundaries 17/17, PostgreSQL remediation/concurrency 15/15 with races 9/9, full Maven 1,430 tests with zero failures/errors and 15 skipped, architecture 49/49, static/frontend gates, Chromium 10/10, 483.5 msg/s sustained, 1,087.1 msg/s burst, latest p95 2.222 ms and history p95 0.989 ms. Authoritative database evidence used only `transport_logistics_acceptance`.

## Next task

`US-48-LIVE-VEHICLE-TRACKING-TECHNICAL-CLOSURE-001-RERUN` passes. Fresh evidence: focused security/status/PostgreSQL 32/32, exact concurrency 9/9, clean Flyway V1→V74, full Maven 1,430 tests with zero failures/errors and 15 skipped in 07:14, architecture 46/46, all static/frontend gates including Vitest 265/265, and real PostgreSQL-backed controlled-provider Chromium 11/11. Sustained ingestion measured 419.0 msg/s, burst 1,016.6 msg/s, latest p95 0.901 ms, and history p95 0.471 ms. Story completion accounting does not advance until independent real-device/real-provider final acceptance. Next task: `US-48-LIVE-VEHICLE-TRACKING-FINAL-ACCEPTANCE-001`.
