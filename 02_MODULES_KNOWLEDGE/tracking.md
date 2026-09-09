# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; V75 provider-connection persistence is complete and independently verified. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts. The trusted provider/Tenant authority, retention, observability, audit and rebuild remediation is implemented. Accounting remains 72/87 with 15 remaining; physical-device and real-provider final acceptance is still required.

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

## Implemented persistence (V73–V75)

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
| `provider_alias` | VARCHAR(80) | NO | - | UNIQUE with `tenant_id`; immutable | Trusted provider alias |
| `provider_type` | VARCHAR(64) | NO | - | `[A-Z][A-Z0-9_]{0,63}` | Installed adapter type |
| `display_name` | VARCHAR(120) | NO | - | Trimmed, nonblank; UNIQUE with `tenant_id` | Tenant-local operator name |
| `endpoint_uri` | VARCHAR(500) | YES | NULL | No embedded credentials | Optional provider endpoint |
| `safe_configuration` | JSONB | NO | `{}` | Object; serialized size <= 8,192 bytes; application rejects secrets | Non-secret provider configuration |
| `credential_reference` | VARCHAR(160) | NO | - | Opaque; no plaintext secret | `IntegrationSecretResolver` reference |
| `lifecycle` | VARCHAR(16) | NO | - | CHECK `DRAFT`,`ACTIVE`,`DISABLED`,`RETIRED` | Connection lifecycle; RETIRED terminal |
| `poll_interval_seconds` | INTEGER | NO | `5` | 5..86,400 | Poll cadence |
| `page_size` | INTEGER | NO | `500` | 1..500 | Provider page bound |
| `test_status` | VARCHAR(24) | NO | `NOT_TESTED` | Exact approved status vocabulary | Orthogonal connection-test result |
| `last_tested_at`, `last_successful_poll_at`, `last_provider_message_at` | TIMESTAMPTZ | YES | NULL | - | Operational observations |
| `last_error_category` | VARCHAR(40) | YES | NULL | Uppercase bounded safe category | Sanitized latest error category |
| `next_poll_at` | TIMESTAMPTZ | YES | NULL | Partial due-work index for ACTIVE rows | Scheduling cursor |
| `lease_owner` | VARCHAR(120) | YES | NULL | Paired with `lease_until` | Coordinator lease owner |
| `lease_until` | TIMESTAMPTZ | YES | NULL | Paired with `lease_owner` | Coordinator lease expiry |
| `created_at`, `updated_at` | TIMESTAMPTZ | NO | - | - | Audit timestamps |
| `created_by`, `updated_by` | UUID | NO | - | Logical actor references | Audit actors |
| `version` | BIGINT | NO | `0` | Optimistic version | Mutation version |

Indexes: global unique `provider_key_id`; unique `(tenant_id,provider_alias)` and `(tenant_id,display_name)`; `(tenant_id,provider_type,lifecycle,id)`; partial `(next_poll_at,id)` for due ACTIVE connections; legacy `(tenant_id,lifecycle,provider_alias,id)`. A trigger prevents Tenant or provider-key reassignment.

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

V75 is implemented as the current head and V1–V75 remain immutable. V75 extends the existing `tracking_provider_binding` as the sole runtime provider-connection authority and reserves the four-state lifecycle on `tracking_device`. Forward-only V76 is authorized to create exactly one Tracking-owned `tracking_device_provider_binding` table as the CS03 persistence prerequisite; it is not yet implemented.

Inbound Tenant authority will be derived from a globally unique opaque provider key ID resolved to an active Tracking-owned binding containing Tenant, bounded provider alias and an opaque credential reference. The secret is resolved only through Integration's published `IntegrationSecretResolver`. Provider alias may repeat across Tenants and never establishes authority. Caller Tenant headers/payloads cannot select or override Tenant. Authentication verifies the binding-derived credential, signed timestamp/body and binding-scoped nonce before resolving the device and source-time association solely inside the derived Tenant. All failures are sanitized and fail closed.

The current US-73 configuration schema is not extended and its tables/repositories are not accessed: its accepted `FILE_EXCHANGE / FILE_JSON_V1 / OUTBOUND` capability cannot represent inbound telematics. No telemetry packet traverses Integration exchange processing. No new public human API, permission or event is authorized; only the provider authentication header/canonical-signature contract changes before acceptance.

Retention remains external policy. An absent policy means no automatic purge and no age-only `TRACKING_POSITION_TOO_OLD`; greater-than-24-hour packets retain the existing LATE behavior. With a configured duration, timestamps before `receivedAt - duration` are too old, equality is accepted, and accepted history records policy/version/retain-until metadata.

Tracking must add an internal per-Tenant/Vehicle transactional rebuild of `tracking_vehicle_latest` from retained immutable history using the identical deterministic receipt/trust ordering as ingress. It is idempotent and has no public endpoint or foreign-table dependency. Missing metrics, sanitized management/provider/retention audit, and a Tracking contributor to the existing health surface are also authorized. Metrics exclude UUIDs, coordinates, nonce/signature/credential and person/customer data. One stale device cannot declare a provider outage.

Closure rerun requires the full signed-ingress negative matrix, exact freshness/connectivity and retention boundaries, deterministic PostgreSQL projection rebuild, V74 binding/nonce/Tenant constraints, the existing nine races, complete regression/static/frontend/browser gates and eventual physical provider/device evidence.

Fresh evidence is security/boundaries 17/17, PostgreSQL remediation/concurrency 15/15 with races 9/9, full Maven 1,430 tests with zero failures/errors and 15 skipped, architecture 49/49, static/frontend gates, Chromium 10/10, 483.5 msg/s sustained, 1,087.1 msg/s burst, latest p95 2.222 ms and history p95 0.989 ms. Authoritative database evidence used only `transport_logistics_acceptance`.

## Next task

`US-48-LIVE-VEHICLE-TRACKING-TECHNICAL-CLOSURE-001-RERUN` passes. Fresh evidence: focused security/status/PostgreSQL 32/32, exact concurrency 9/9, clean Flyway V1→V74, full Maven 1,430 tests with zero failures/errors and 15 skipped in 07:14, architecture 46/46, all static/frontend gates including Vitest 265/265, and real PostgreSQL-backed controlled-provider Chromium 11/11. Sustained ingestion measured 419.0 msg/s, burst 1,016.6 msg/s, latest p95 0.901 ms, and history p95 0.471 ms. Story completion accounting does not advance until independent real-device/real-provider final acceptance. Next task: `US-48-LIVE-VEHICLE-TRACKING-FINAL-ACCEPTANCE-001`.

## Real-device/provider acquisition selection

`US-48-REAL-DEVICE-PROVIDER-ACQUISITION-001` selects one Teltonika FMC130 with a customer LTE SIM feeding a flespi developer account as the preferred acceptance pilot. The physical device uses its native Teltonika protocol to the real provider platform; flespi exposes normalized provider-generated telemetry. Selection is not acquisition or real-source evidence.

## Authorized flespi Level-1 inbound adapter

`US-48-FLESPI-LEVEL1-INBOUND-ADAPTER-IMPLEMENTATION-001` implements the authorized provider-specific adapter entirely under `com.transportlogistics.app.tracking.adapters.inbound.flespi`; its state is `IMPLEMENTATION_COMPLETE / REAL_CAPTURE_PENDING`. Its transport is bounded HTTPS REST polling, not MQTT/webhook, with a five-second minimum/default interval, at most 500 messages per page, a 1 MiB response bound, a bounded five-minute cold-start overlap and bounded 5/15/30/60-second transient retry/backoff plus jitter. Provider DTOs, response parsing, in-memory watermark and mappings remain private to the adapter. Single-flight polling prevents overlap, the adapter is disabled by default, and provider failure degrades only the `trackingFlespi` health contributor while Tracking continues to serve last-known state.

The adapter resolves the least-privilege flespi token through the existing binding's opaque `credentialReference` and `IntegrationSecretResolver`. The same high-entropy resolved secret signs a private loopback HTTP invocation of the literal unchanged `POST /api/integration/v1/tracking/positions` contract using the exact canonical HMAC form and a fresh cryptographically random nonce. Direct `TrackingUseCase` invocation is absent because the existing binding, HMAC, nonce and admission chain remains authoritative. The in-memory timestamp watermark advances only after accepted ingress; delivery is at least once and existing Tracking dedupe remains authoritative. No new public API, permission, event, table, migration, domain/application contract or Integration telemetry path exists.

Documented candidate mappings are flespi `ident`, `timestamp`, `position.latitude`, `position.longitude`, and optional candidates `position.accuracy`, `position.speed`, `position.direction`, ignition, odometer, engine hours and immutable message identity/sequence. Every field remains pending confirmation from a real FMC130-generated flespi capture; absent or uncertain optional facts remain UNKNOWN/null. Documentation-aligned fixtures are authorized for implementation testing only and are not acceptance evidence. Existing Tracking dedupe is authoritative, delivery is at least once, and the in-memory watermark advances only after accepted or idempotent ingress.

Controlled documentation-aligned tests and real PostgreSQL-backed controlled-provider Chromium evidence pass, but they are not real provider evidence. US-48 remains `ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; accounting remains 72/87 complete with 15 remaining. Physical hardware, provider activation and real telemetry are still required. The pluggable-onboarding amendment below supersedes the former immediate capture queue; US-49 must not start.

## Authorized pluggable supported-adapter architecture (CS01 implemented)

`US-48-PLUGGABLE-DEVICE-ONBOARDING-ARCHITECTURE-001` approves `PLUG_AND_PLAY_FOR_SUPPORTED_ADAPTERS`. Runtime administrators may onboard many devices and provider connections across Tenants without restart once the provider adapter is installed. New proprietary protocol code still requires a reviewed deployment; dynamic JAR upload is prohibited. CS01 provider SPI/registry and CS02 provider-connection persistence are COMPLETE; CS03 device-provider binding persistence is next.

The implemented provider-neutral outbound SPI is `TrackingProviderAdapter`, discovered through the immutable `TrackingProviderAdapterRegistry`. `ProviderType` is a strict uppercase value (`[A-Z][A-Z0-9_]{0,63}`); duplicate types fail startup and unsupported required lookups fail with `TRACKING_PROVIDER_TYPE_UNSUPPORTED`. The exact immutable capability catalogue is `POLLING`, `WEBHOOK`, `MQTT`, `DISCOVERY`, `SOURCE_TIMESTAMP`, `ACCURACY`, `SPEED`, `HEADING`, `IGNITION`, `ODOMETER`, `ENGINE_HOURS`, `MESSAGE_ID`, `SEQUENCE`, `HISTORY`, and `REPLAY`. The SPI returns only normalized candidates and bounded provider-neutral cursors; provider DTOs, raw payloads, transport types, Tenant claims and secrets remain inside adapter packages. Unsupported discovery is explicit and safe. Initially only FLESPI is supported.

V75 extends `tracking_provider_binding` into the runtime provider-connection persistence authority with provider type, display/endpoint/bounded safe configuration, poll/page limits, test/health state, scheduling cursor and lease facts. Lifecycle is DRAFT/ACTIVE/DISABLED/RETIRED. Tenant-scoped management reads and optimistic updates are implemented through `TrackingProviderConnectionStore`. The separate `tracking_device_provider_binding` table, compatibility projections and device-binding execution authority remain explicitly deferred to CS03.

One bounded `ProviderPollingCoordinator` dynamically claims due connections using database leases and `FOR UPDATE SKIP LOCKED`, pages/batches active device bindings and enforces provider quotas; no per-device scheduler/thread or environment variable exists. Internal polling uses a non-web ingestion port that reloads ACTIVE connection, Tenant and device binding before calling the identical normalized ingestion service. External push providers retain the V74 signed HMAC/nonce endpoint. Provider disable stops future claims while last-known state remains available.

Provider-connection persistence is implemented; forward-only V76 is authorized for device-binding persistence, while coordinator, APIs and UI remain authorized but not implemented. V76 may create only `tracking_device_provider_binding` with same-Tenant device/provider foreign keys, Tenant-scoped external-reference uniqueness, one ACTIVE binding per device, the four-state lifecycle, bounded non-secret JSON configuration, execution watermarks, next-poll cursor, audit facts and optimistic version. It must deterministically backfill every legacy device only through exactly one same-Tenant alias match and fail safely on zero or ambiguity; ACTIVE requires both device and provider connection ACTIVE, otherwise the backfill is DISABLED. Legacy device provider columns remain unchanged. Current Flyway head is V75 and authorized next head is V76. Real FMC130/Flespi evidence remains mandatory. Accounting stays 72/87; US-49 remains blocked. Next task: `US-48-PLUGGABLE-DEVICE-ONBOARDING-CS03-DEVICE-PROVIDER-BINDING-001-RERUN`.
