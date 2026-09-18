# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; pluggable-onboarding CS01–CS10 is technically complete and independently verified through V76. The current repository Flyway head is V100. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts and Tracking-owned geofence, speed and route-deviation evaluation. US-49 is `COMPLETE / ACCEPTED`; US-50 and US-52 are technically complete but independently blocked on their physical external-acceptance evidence. Accounting is 73/87 with 14 remaining. US-53 and US-54 are technically complete with independent external field-acceptance holds. US-55 is `TECHNICALLY_COMPLETE / IMPLEMENTATION_COMPLETE_ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; its field matrix is 0 PASS, 0 FAIL and 9 externally blocked requirements. The open acceptance queue remains `US-55-HANDLE-GPS-EDGE-CASES-FINAL-ACCEPTANCE-001`. Canonical polling, the bounded Traccar 6.15.3 production adapter and guided Flespi/Traccar onboarding UI are complete; Generic signed-HMAC ingress remains available. The independent queue is `US-55-HEALTH-RECOVERY`.

US-49 CS06 adds the operator frontend using the existing React Router, Ant Design, TanStack Query, React Hook Form/Zod, Axios and AuthContext architecture. It provides Tracking > Geofences list/new/detail/edit routes, server filters and pagination, exact permission/lifecycle affordances, accessible open-ring editing, local SVG preview, optimistic concurrency, idempotent lifecycle commands, stable memberships, and privacy-minimized transition history. No backend contract, dependency, map provider, dashboard or Operations workflow changed. Real PostgreSQL-backed Chromium evidence includes signed trusted telemetry and a confirmed HIGH `UNAUTHORIZED_ZONE_ENTERED` transition. CS07 and CS07A concurrency, performance and V80 physical-design hardening are complete; independent final acceptance passed.

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
- Storage: the accepted high-throughput platform assigns Kafka to the durable Tenant/Vehicle-ordered telemetry stream, Redis to replaceable Tenant-qualified live state, TimescaleDB to append-only normalized history, and PostgreSQL to provider/device configuration, nonce/dedupe authority, audit and detector state. Redis Streams are superseded; PostGIS remains deferred.
- UI: minimal list/map point/latest/last-known/freshness/connectivity/accuracy/history/device association; 15-second visible polling with 30/60-second failure backoff; no US-54 dashboard.
- Privacy: precise location requires same-Tenant `TRACKING_VIEW`; history also requires `TRACKING_HISTORY_VIEW`; no Customer exposure, Driver profile, raw payload or credential exposure.
- P1-01: no per-packet event. `VehicleTrackingStateChangedV1` is inactive until a consumer is approved and is then coalesced/state-change-only through the shared durable outbox.

## Implemented persistence (V73–V92)

The hybrid platform is `IMPLEMENTATION_IN_PROGRESS / TS04_COMPLETE`. V86 provides the real
TimescaleDB extension and Tenant-qualified telemetry-history hypertable; local Compose provides
TimescaleDB PostgreSQL 16 plus Redis 7.4 AOF/noeviction, and production hybrid mode is explicit.
Flespi, Traccar and Generic normalizers are implemented behind a fail-fast registry. Secure
dynamic Kafka ingress and the Redis projector are complete. V87 implements the transactional
Timescale batch consumer/policies. Gateway UI and live Fleet map are complete; physical acceptance remains. US-52
route-geometry/deviation persistence is V88 and its API permission seed is V89. Accounting remains 73/87 and US-48
physical acceptance remains blocked independently.

The governed provider boundary is plug-and-play for supported adapters, not universal protocol
compatibility. Flespi production HTTPS polling and Generic signed-HMAC ingress remain supported.
Traccar production HTTPS polling is approved but implementation-pending, using a separate adapter,
verified TLS, an opaque bearer-token reference, bounded `/api/positions` paging and a restart-safe
watermark. Flespi and Traccar are peers; neither is the other's fallback. Adapters normalize into the
canonical V1/V2 boundary and never write Redis, TimescaleDB or another module's persistence directly.
Onboarding remains DRAFT-first through connectivity test, discovery/manual identity, same-Tenant
Vehicle binding, normalized-sample validation and explicit activation. Disablement, rebinding and
credential rotation preserve effective-dated audit evidence. `TRACKING_DEVICE_MANAGE` is sufficient;
no new permission, public API or migration is introduced by this architecture decision.

#### Table: `tracking_telemetry_evaluation_dispatch`

- **Purpose:** Durable, history-backed scheduling for geofence, speed and route-deviation evaluation.
- **Primary Key:** `dispatch_id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, leading in identity/query indexes)
- **History relationship:** logical immutable identity (`tenant_id`, `source_timestamp`, `history_id`),
  verified during the atomic history/dispatch insertion transaction because a normal-table foreign key
  cannot target the current Timescale hypertable uniqueness without its partition key.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `dispatch_id` | UUID | NO | `gen_random_uuid()` | PRIMARY KEY | Durable dispatch identity |
| `tenant_id` | UUID | NO | - | Tenant-scoped unique/index component | Trusted Tenant scope |
| `source_timestamp` | TIMESTAMPTZ | NO | - | History identity/index component | Immutable source time |
| `history_id` | UUID | NO | - | Logical history identity | Exact Timescale fact |
| `dedupe_identity` | CHAR(64) | NO | - | Tenant/source/evaluator unique | Canonical telemetry identity |
| `vehicle_id` | UUID | NO | - | Logical Fleet Vehicle reference | Evaluation ordering scope |
| `evaluator_type` | VARCHAR(24) | NO | - | CHECK GEOFENCE/SPEED/ROUTE_DEVIATION | Existing evaluator to invoke |
| `status` | VARCHAR(16) | NO | `PENDING` | CHECK PENDING/PROCESSING/COMPLETED/FAILED | Durable lifecycle |
| `attempt_count` | INTEGER | NO | `0` | CHECK 0–1000 | Claim attempts |
| `next_attempt_at` | TIMESTAMPTZ | NO | - | Due index | Retry/recovery schedule |
| `lease_owner` | VARCHAR(120) | YES | NULL | Required only while PROCESSING | Claim owner |
| `lease_until` | TIMESTAMPTZ | YES | NULL | Required only while PROCESSING | Crash-recovery deadline |
| `completed_at` | TIMESTAMPTZ | YES | NULL | Required only when COMPLETED | Completion evidence |
| `last_error_code` | VARCHAR(120) | YES | NULL | Bounded safe classification | No raw exception detail |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | - | Last lifecycle change |
| `version` | BIGINT | NO | `0` | Non-negative | Optimistic change counter |

#### Table: `tracking_position_history`

- **Purpose:** TimescaleDB append-only normalized telemetry history populated by the promoted micro-batch path.
- **Primary Key:** (`tenant_id`, `source_timestamp`, `id`)
- **Multi-Tenant Key:** `tenant_id` (UUID, leading in primary/query indexes)
- **Hypertable partition:** `source_timestamp`, seven-day chunks
- **Compression:** after seven days; segment by `tenant_id,vehicle_id`; order by `source_timestamp DESC,id DESC`
- **Raw retention:** 180 days
- **Database idempotency:** UNIQUE (`tenant_id`, `source_timestamp`, `dedupe_identity`)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | PRIMARY KEY component | Trusted Tenant scope |
| `source_timestamp` | TIMESTAMPTZ | NO | - | PRIMARY KEY/partition component | Device source time |
| `id` | UUID | NO | - | PRIMARY KEY component | Immutable position identity |
| `event_version` | INTEGER | NO | `1` | CHECK IN (1,2) | Canonical telemetry schema version |
| `device_id` | UUID | NO | - | Logical Tracking device reference | Source device |
| `vehicle_id` | UUID | NO | - | Logical Fleet Vehicle reference | Source-time Vehicle |
| `provider_alias` | VARCHAR(80) | NO | - | - | Trusted provider alias |
| `provider_message_id` | VARCHAR(160) | YES | NULL | - | Stable provider identity when supplied |
| `provider_sequence` | BIGINT | YES | NULL | - | Provider sequence when supplied |
| `dedupe_identity` | CHAR(64) | NO | - | - | Canonical dedupe identity |
| `received_at` | TIMESTAMPTZ | NO | - | - | Independent server receipt time |
| `latitude` | NUMERIC(10,7) | NO | - | CHECK `[-90,90]` | WGS84 latitude |
| `longitude` | NUMERIC(10,7) | NO | - | CHECK `[-180,180]` | WGS84 longitude |
| `horizontal_accuracy_meters` | NUMERIC(10,3) | YES | NULL | CHECK non-negative | Provider accuracy |
| `speed_kph` | NUMERIC(8,3) | YES | NULL | CHECK `[0,400]` | Normalized speed |
| `heading_degrees` | NUMERIC(7,3) | YES | NULL | CHECK `[0,360)` | Heading |
| `altitude_meters` | NUMERIC(12,3) | YES | NULL | - | Altitude |
| `engine_state` | VARCHAR(10) | NO | - | CHECK ON/OFF/UNKNOWN | Engine state |
| `odometer_km` | NUMERIC(14,3) | YES | NULL | - | Provider odometer |
| `engine_hours` | NUMERIC(14,3) | YES | NULL | - | Provider engine hours |
| `trust` | VARCHAR(12) | NO | - | CHECK TRUSTED/UNTRUSTED/UNKNOWN | Trust decision |
| `quality` | VARCHAR(32) | NO | - | - | Quality classification |
| `ordering_classification` | VARCHAR(20) | NO | - | CHECK frozen ordering vocabulary | Arrival ordering |
| `retention_policy` | VARCHAR(80) | NO | - | - | Retention policy identity |
| `retention_policy_version` | VARCHAR(40) | NO | - | - | Applied policy version |
| `retain_until` | TIMESTAMPTZ | YES | NULL | Tenant-leading partial index | Optional retention boundary |
| `safe_metadata` | JSONB | NO | `{}` | - | Bounded non-secret metadata |
| `tamper_state` | VARCHAR(16) | YES | NULL | CHECK DETECTED/CLEAR/UNKNOWN; V2 only | Provider tamper evidence |
| `battery_level_percent` | NUMERIC | YES | NULL | CHECK 0–100 and scale ≤ 3; V2 only | Provider battery percentage |
| `battery_voltage_volts` | NUMERIC | YES | NULL | CHECK 0–1000 and scale ≤ 6; V2 only | Provider battery voltage |
| `external_power_state` | VARCHAR(16) | YES | NULL | CHECK CONNECTED/DISCONNECTED/UNKNOWN; V2 only | Provider external-power evidence |
| `battery_charging_state` | VARCHAR(20) | YES | NULL | CHECK CHARGING/NOT_CHARGING/UNKNOWN; V2 only | Provider charging evidence |

V96 enforces append-only `tracking_position_history` at the database boundary. V1 rows must keep all V2-only evidence null. Missing V2 evidence remains null while an explicit provider-reported `UNKNOWN` remains a distinct stored value.

#### Table: `tracking_device_telemetry_capability`

- **Purpose:** Effective-dated, immutable device capability evidence used by source-time telemetry interpretation.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, leading in uniqueness and lookup indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Capability-history identity |
| `tenant_id` | UUID | NO | - | Same-Tenant FK component; index leader | Trusted Tenant scope |
| `tracking_device_id` | UUID | NO | - | Same-Tenant FK → `tracking_device(tenant_id,id)` | Tracking-owned device |
| `capability` | VARCHAR(32) | NO | - | CHECK approved position/speed/accuracy/heading/ignition/tamper/battery/power vocabulary | Capability name |
| `capability_state` | VARCHAR(16) | NO | - | CHECK SUPPORTED/UNSUPPORTED/UNKNOWN | Explicit effective state |
| `effective_from` | TIMESTAMPTZ | NO | - | Half-open interval; effective lookup index | Inclusive source-time boundary |
| `effective_to` | TIMESTAMPTZ | YES | NULL | Must be greater than `effective_from` | Exclusive source-time boundary |
| `recorded_at` | TIMESTAMPTZ | NO | - | Immutable | Evidence recording time |
| `recorded_by` | UUID | NO | - | Immutable actor identity | Recording actor |

Only a one-time close of an open interval is mutable. Deletes, fact rewriting, overlapping intervals and more than one active Tenant/device/capability row are rejected. The JDBC lookup requires Tenant, device, capability and source time and returns conservative `UNKNOWN` when no row applies. No credential value is stored.

Tracking owns `tracking_device`, `tracking_vehicle_device_assignment`, `tracking_position`, `tracking_vehicle_latest`, `tracking_ingest_nonce`, and `tracking_audit_event`. Every table is Tenant-owned. Device/association/latest same-module relationships are Tenant-consistent; `vehicle_id` is a logical Fleet reference without a physical cross-module FK. Tenant-leading indexes cover Vehicle/source time, device/source time, latest lookup, active associations, provider-message/dedupe identity, nonce expiry and audit time. `tracking_position` is trigger-enforced append-only and association history allows only its one-time close operation.

`tracking_device` stores UUID `id`, required indexed UUID `tenant_id`, external/provider/hardware references, DRAFT/ACTIVE/DISABLED/RETIRED lifecycle, registration actor/time, last-seen time, optimistic `version`, and created/updated timestamps. Creation defaults to DRAFT and RETIRED is terminal. `tracking_vehicle_device_assignment` stores UUID identity/Tenant/device/logical Vehicle, required `effective_from`, optional exclusive `effective_to`, and creation actor/time. `tracking_position` stores immutable UUID/Tenant/device/logical Vehicle identity; provider/message/sequence/dedupe/hash facts; source/receipt timestamps; WGS84 coordinates and optional quality/engine/meter facts; trust/quality/order classifications; retention policy/version/optional retain-until; and safe JSON metadata. `tracking_vehicle_latest` stores the Tenant+Vehicle key, latest received/trusted position references, last receipt, policy version, updated time and optimistic version. `tracking_ingest_nonce` stores Tenant/provider/nonce hash and use/expiry times. `tracking_audit_event` stores Tenant/actor/action/target/safe detail/time without per-packet audit.

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

### Table: `tracking_device_provider_binding`

- **Purpose:** Sole runtime authority joining a Tracking device to a provider connection and provider-side external identity.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, tenant-leading indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Binding identity |
| `tenant_id` | UUID | NO | - | Same-Tenant composite FKs; indexed | Trusted Tenant scope |
| `tracking_device_id` | UUID | NO | - | `(tenant_id,id)` FK → `tracking_device`, ON DELETE RESTRICT; one partial-unique ACTIVE row | Tracking device |
| `provider_binding_id` | UUID | NO | - | `(tenant_id,id)` FK → `tracking_provider_binding`, ON DELETE RESTRICT | Provider connection |
| `external_device_reference` | VARCHAR(160) | NO | - | Trimmed nonblank; UNIQUE with Tenant and provider binding | Provider-side device identity |
| `safe_configuration` | JSONB | NO | `{}` | JSON object; serialized size <=4,096 bytes; application rejects secret-like keys | Non-secret device/connection configuration |
| `lifecycle` | VARCHAR(16) | NO | - | CHECK `DRAFT`,`ACTIVE`,`DISABLED`,`RETIRED`; RETIRED terminal | Binding lifecycle |
| `watermark_source_timestamp` | TIMESTAMPTZ | YES | NULL | - | Last committed provider source cursor time |
| `watermark_message_identity` | VARCHAR(160) | YES | NULL | Trimmed nonblank when present | Last committed provider message cursor |
| `next_poll_at` | TIMESTAMPTZ | YES | NULL | Partial Tenant-leading due index | Device polling cursor |
| `created_at`, `updated_at` | TIMESTAMPTZ | NO | - | - | Audit timestamps |
| `created_by`, `updated_by` | UUID | NO | - | Logical actor references | Audit actors |
| `version` | BIGINT | NO | `0` | CHECK >=0; optimistic concurrency | Mutation version |

Indexes enforce unique `(tenant_id,provider_binding_id,external_device_reference)`, at most one ACTIVE binding per `(tenant_id,tracking_device_id)`, bounded connection lists and Tenant/lifecycle/due-work queries. V76 backfills every legacy device through exactly one exact same-Tenant provider-alias match, fails closed for zero/ambiguous/cross-Tenant-only matches, and chooses ACTIVE only when both parents are ACTIVE. Legacy device provider columns remain an atomically maintained compatibility projection. Rebind disables the current active binding, creates its replacement and updates that projection in one transaction while preserving binding and Vehicle-association history.

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

V76 is implemented as the current head and V1–V75 remain immutable. V75 extends the existing `tracking_provider_binding` as the sole runtime provider-connection authority; V76 adds exactly one Tracking-owned `tracking_device_provider_binding` table as the sole device/connection/external-identity runtime authority. CS03 lifecycle, Tenant-scoped storage, safe configuration, deterministic fail-closed backfill, transactional rebind, compatibility projection, watermarks, next-poll persistence and optimistic concurrency are implemented.

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

## Historical flespi Level-1 inbound adapter (superseded by CS05)

`US-48-FLESPI-LEVEL1-INBOUND-ADAPTER-IMPLEMENTATION-001` originally implemented the authorized provider-specific adapter under `com.transportlogistics.app.tracking.adapters.inbound.flespi`; its real-capture status remains pending. Its bounded HTTPS client and mappings were retained/refactored by CS05, while its singleton scheduler, in-memory watermark, device properties, nested retry schedule and loopback bridge were retired.

The historical Level-1 path resolved the least-privilege token and signed a private loopback invocation. CS05 supersedes that execution model: only the coordinator resolves the opaque credential reference, the Flespi SPI receives the transient token, and `TrackingProviderIngestionPort` reloads database authority before normalized ingestion. The public signed endpoint remains unchanged for external push providers but is not used by Flespi polling.

Documented candidate mappings remain flespi `ident`, `timestamp`, `position.latitude`, `position.longitude`, and optional `position.accuracy`, `position.speed` and `position.direction`. Every field remains pending confirmation from a real FMC130-generated flespi capture; ignition, odometer, engine hours and immutable message identity/sequence remain unsupported/UNKNOWN. Documentation-aligned fixtures are implementation evidence only. Existing Tracking dedupe is authoritative, delivery is at least once, and V76 per-binding watermarks replace the retired in-memory cursor.

Controlled documentation-aligned tests and real PostgreSQL-backed controlled-provider Chromium evidence pass, but they are not real provider evidence. US-48 remains `ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; accounting remains 72/87 complete with 15 remaining. Physical hardware, provider activation and real telemetry are still required. The pluggable-onboarding amendment below supersedes the former immediate capture queue; US-49 must not start.

## Authorized pluggable supported-adapter architecture (CS01–CS08 and CS09 backend remediation implemented)

`US-48-PLUGGABLE-DEVICE-ONBOARDING-ARCHITECTURE-001` approves `PLUG_AND_PLAY_FOR_SUPPORTED_ADAPTERS`. Runtime administrators may onboard many devices and provider connections across Tenants without restart once the provider adapter is installed. New proprietary protocol code still requires a reviewed deployment; dynamic JAR upload is prohibited. CS01–CS08 and the CS09 backend-contract remediation are COMPLETE: provider SPI/registry, connection persistence, device binding persistence, coordinator, Flespi SPI cutover, provider-neutral management APIs, security hardening, Provider Connections UI and the executable onboarding contract. The CS09 frontend rerun is next.

The implemented provider-neutral outbound SPI is `TrackingProviderAdapter`, discovered through the immutable `TrackingProviderAdapterRegistry`. `ProviderType` is a strict uppercase value (`[A-Z][A-Z0-9_]{0,63}`); duplicate types fail startup and unsupported required lookups fail with `TRACKING_PROVIDER_TYPE_UNSUPPORTED`. The exact immutable capability catalogue is `POLLING`, `WEBHOOK`, `MQTT`, `DISCOVERY`, `SOURCE_TIMESTAMP`, `ACCURACY`, `SPEED`, `HEADING`, `IGNITION`, `ODOMETER`, `ENGINE_HOURS`, `MESSAGE_ID`, `SEQUENCE`, `HISTORY`, and `REPLAY`. The SPI returns only normalized candidates and bounded provider-neutral cursors; provider DTOs, raw payloads, transport types, Tenant claims and secrets remain inside adapter packages. Unsupported discovery is explicit and safe. Initially only FLESPI is supported.

V75 extends `tracking_provider_binding` into the runtime provider-connection persistence authority with provider type, display/endpoint/bounded safe configuration, poll/page limits, test/health state, scheduling cursor and lease facts. Lifecycle is DRAFT/ACTIVE/DISABLED/RETIRED. Tenant-scoped management reads and optimistic updates are implemented through `TrackingProviderConnectionStore`. V76 and `TrackingDeviceProviderBindingStore` implement the separate device-provider binding authority, compatibility projection, watermarks and atomic rebinding.

The implemented, default-off `ProviderPollingCoordinator` has one scheduler entry point and dynamically claims bounded due-connection batches using database leases and `FOR UPDATE SKIP LOCKED`. Owner-only renew/release, lease-expiry recovery, fixed bounded workers/queue, connection single-flight, provider-type semaphores, bounded deadlines/responses and safe backoff provide failure isolation and backpressure; no per-device scheduler/thread or environment variable exists. It pages/batches ACTIVE bindings fairly, discovers hot additions and observes device/binding/provider disables without restart.

Internal polling uses the non-web `TrackingProviderIngestionPort`. Its adapter reloads the leased ACTIVE connection and persistence-derived Tenant, then locks and reloads each ACTIVE binding/device before calling the identical normalized ingestion service. Source-time Vehicle association, dedupe, trust, retention and latest projections are therefore unchanged. Per-candidate ACCEPTED/DUPLICATE/REJECTED outcomes make partial batches safe; only ACCEPTED/DUPLICATE advance source/message watermarks. External push providers retain the V74 signed HMAC/nonce endpoint, and the coordinator performs no HMAC loopback. Metrics and health use bounded non-sensitive labels/state; one provider failure does not make Tracking unavailable.

Provider-connection/device-binding persistence, the provider execution coordinator and the CS06 management APIs are implemented at current head V76; UI remains deferred. CS06 exposes installed provider types, Tenant-scoped connection create/list/get/update/test/lifecycle, capability-gated discovery, device bind/rebind and explicit device retirement. It validates through the adapter registry, resolves and clears opaque credentials for test/discovery/activation, maps persistence conflicts safely and exposes no credential reference. V76 creates only `tracking_device_provider_binding` with same-Tenant device/provider foreign keys, Tenant-scoped external-reference uniqueness, one ACTIVE binding per device, the four-state lifecycle, bounded non-secret JSON configuration, execution watermarks, next-poll cursor, audit facts and optimistic version. Real FMC130/Flespi evidence remains mandatory. Accounting stays 72/87; US-49 remains blocked.

CS07 reuses only `TRACKING_DEVICE_MANAGE` for provider/device management at both the HTTP and method-security layers. Tenant authority comes from active authenticated membership; persistence and internal provider execution revalidate same-Tenant ACTIVE provider, binding, device and lease authority. Successful provider connection and device-binding management changes write safe audit facts atomically with state, while authenticated denied management commands are best-effort audited without changing the denial. Audit, responses, logs, metrics and health expose no credential reference, secret, raw provider response, external identity or packet payload. The Flespi adapter accepts only HTTPS/443 endpoints in the provider-controlled `flespi.io` DNS zone, follows no redirects and retains bounded timeouts and response reads, preventing arbitrary loopback/private/link-local/metadata targets without a new policy service or schema.

CS08 implements the Provider Connections frontend at `/tracking/provider-connections` within the existing Tracking/AppLayout shell. The permission-filtered Tracking navigation exposes Live Vehicles, Devices and Provider Connections; `TRACKING_DEVICE_MANAGE` is required for the provider-management route and controls, while backend authorization remains final. The page uses only CS06 management APIs and backend-reported installed provider types/capabilities. It supports bounded list/detail, DRAFT-first create, optimistic update, safe Test Connection results, activate, disable, permanent retire and capability-gated bounded discovery preview. RETIRED is terminal and no hard delete or device onboarding workflow exists. React Hook Form/Zod enforce user-facing bounds, while backend validation and SSRF policy remain authoritative. Existing credential references are never returned, prefilled, rendered or stored; edit accepts an optional blank-by-default replacement input and displays only `credentialConfigured`.

`US-55-PROVIDER-UI` completes the guided supported-provider journey without changing those APIs. The UI separates connection saved, connection verified, device bound, same-Tenant Vehicle associated, polling enabled and telemetry received; a passing connection test is never presented as telemetry evidence. It provides provider-specific Flespi Cloud and Traccar HTTPS guidance, opaque-reference-only credential handling, deployment-admin-managed Traccar private endpoint guidance, capability-gated discovery/manual identity entry and a direct handoff into the existing effective-dated device binding and Vehicle association workflow. Generic signed-HMAC ingress remains explicitly documented as a governed push path, not a polling adapter. Technical fixtures pass frontend and real PostgreSQL-backed Chromium gates, but physical/provider acceptance remains externally blocked.

The CS09 backend-contract remediation makes the frozen onboarding sequence executable: create a DRAFT device, create an ACTIVE binding to an ACTIVE same-Tenant provider connection, start an effective-dated same-Tenant Vehicle association, then explicitly activate the device. Association and binding preparation accept only DRAFT/ACTIVE devices; DISABLED/RETIRED devices are ineligible and RETIRED remains terminal. Activation atomically requires both the active provider authority and current Vehicle association and fails with safe `TRACKING_DEVICE_NOT_READY` or `TRACKING_STALE_VERSION` semantics. Device list/detail and mutation responses add current provider-binding and Vehicle-association summaries. Managers receive the binding/provider IDs, safe lifecycle/version/display/type/alias facts, masked external reference and bounded safe configuration; view-only callers receive no binding mutation version or credential material. The deterministic current-binding read prefers ACTIVE, then DRAFT, then DISABLED and enables refresh-safe rebind.

The CS09 throughput remediation preserves the unchanged signed-ingress workload and `>=1000 msg/s` burst gate. `JdbcTrackingStore` now reuses only transaction-stable batch facts: one retention-policy read per batch and one transaction-scoped advisory-lock/authority reload per unique device and Vehicle. Per-message validation, source-time association, dedupe/conflict, ordering, trust, immutable history and latest projections are unchanged. Three accepted burst runs measured 1,300.5, 1,675.3 and 1,229.4 msg/s (median 1,300.5); a post-functional run measured 1,390.9 msg/s, sustained runs remained above 200 msg/s, complete Tracking Chromium passed 24/24, Tracking Java passed 112/112 and full Maven passed 1,513 tests with zero failures/errors and 15 skipped. Flyway remains V76 with no schema/index change. This synchronizes the accepted technical performance fact only; CS09 final frontend acceptance and physical FMC130/Flespi evidence remain separate.

CS05 registers exactly one FLESPI adapter with capabilities `POLLING`, `SOURCE_TIMESTAMP`, `ACCURACY`, `SPEED`, `HEADING` and `HISTORY`. `DISCOVERY`, `REPLAY`, `MESSAGE_ID`, `SEQUENCE`, `IGNITION`, `ODOMETER` and `ENGINE_HOURS` remain absent/pending real capture. The adapter uses persisted connection endpoint/alias/safe configuration plus bounded device cursors, performs bounded sequential per-device HTTPS requests with no device threads, and maps provider DTOs directly to normalized candidates inside the adapter package. V76 per-binding watermarks are authoritative; cold/retry overlap is at most five minutes and Tracking dedupe remains final authority.

The legacy `FlespiPollingAdapter`, device-specific `FlespiAdapterProperties`, in-memory business watermark and `TrackingIngressBridge` loopback are removed. The coordinator alone resolves `credentialReference` through `IntegrationSecretResolver` per execution and clears the transient buffer; rotation therefore needs no restart. Flespi owns no scheduler or job retry. Safe transient provider failures are returned to coordinator backoff, and connection-scoped adapter health remains separate from aggregate Tracking health. Rollback disables coordinator/connection execution and retains all V76 data; dual polling is not supported. CS05 introduced no migration, permission, event, outbox or Integration packet route. CS06 subsequently added the authorized management APIs, CS07 hardened their security boundary, CS08 delivered the Provider Connections UI and CS09 completed the DRAFT-first device-onboarding workflow plus backend and throughput remediations. CS10 completes controlled technical acceptance: PostgreSQL 47/47, Tracking Java 112/112, Chromium 24/24, Maven 1,513/0/0/15 and architecture 49/49 passed. Transaction-local latest-trusted source-time reuse is confined to each locked Vehicle and transaction; three unchanged post-fix bursts measured 1,601.5, 1,752.8 and 1,516.2 msg/s, with sustained results 774.9, 649.9 and 546.2 msg/s. Real FMC130/Flespi evidence remains mandatory. Accounting stays 72/87; US-49 remains blocked.

## External acceptance preparation

`US-48-LIVE-VEHICLE-TRACKING-EXTERNAL-ACCEPTANCE-PREPARATION-001` is COMPLETE. The application repository contains an operator runbook plus capture and final-acceptance templates. Preparation does not constitute physical-provider evidence and does not change US-48 acceptance or story accounting. The required source remains one physical FMC130 using real LTE/GNSS and a real Teltonika channel/device in Flespi, with a least-privilege token resolved through an opaque environment reference. Flespi capture must verify `ident`, `timestamp`, `position.latitude`, `position.longitude` and truthful optional `position.accuracy`, `position.speed` and `position.direction`; ignition, odometer, engine hours, message identity and sequence remain unsupported and must not be synthesized. Current head remains V76 with no V77. Next task: `US-48-FMC130-FLESPI-EXTERNAL-CAPTURE-001`; US-49 remains blocked until independent final acceptance passes.

The first operational readiness execution is `BLOCKED / PHYSICAL_HARDWARE_REQUIRED`: no physical FMC130 or genuine FMC130-originated Flespi message was available or verifiable, so the mandated hard gate stopped before acceptance-database and application runtime startup. No provider account, credential, connection or capture was claimed. The capture template remains `NOT_EXECUTED`, US-48 remains `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`, and accounting remains 72/87. After physical FMC130/LTE/GNSS/Flespi acquisition, rerun `US-48-FMC130-FLESPI-EXTERNAL-CAPTURE-READINESS-001`; only readiness PASS advances to the capture rerun. US-49 remains blocked.

## External hold and downstream disposition

The ARB approves `ON_HOLD_EXTERNAL_PREREQUISITE` for US-48 physical capture after two unchanged hardware-blocked readiness attempts. No further readiness task is scheduled until a physical FMC130 or verified live Flespi prerequisite materially changes. Physical capture and independent final acceptance remain mandatory; US-48 is not complete or waived, its migration baseline remains through V76 while the repository head is V77, and accounting remains 72/87.

The frozen technical Tracking contract is sufficient to begin downstream decisions without acceptance inheritance. US-49 is `TECHNICAL_DEPENDENCY_SATISFIED / READY_FOR_PRODUCT_DECISIONS` because it needs trusted WGS84 position, Vehicle and source time. US-50 and US-52 are likewise ready for product decisions against normalized speed and trusted-position/Routing contracts respectively; real speed fidelity remains a US-50 final-evidence gate. US-53 immutable-history dependency is satisfied but overlay decisions follow earlier producers. US-51 remains `BLOCKED_BY_REQUIRED_TELEMETRY_CAPABILITY` because current FLESPI does not advertise IGNITION and no accepted alternate engine-state source exists. US-54 remains blocked by US-49..53 producers. Full US-55 remains blocked because tamper, spoofing and battery signals/product semantics are not established, despite existing loss/delay/trust support. Next task: `US-49-MANAGE-GEOFENCES-PRODUCT-DECISIONS-001`.

## US-49 Manage Geofences — frozen product decision

`US-49-MANAGE-GEOFENCES-CS01-DOMAIN-PORTS-001` is `COMPLETE`; US-49 remains implementation-in-progress without story acceptance credit. Tracking owns the framework-free aggregate, evaluation state, transitions, evaluation-job model and provider-neutral ports; US-63 Delivery Zones remain a distinct last-mile serviceability/capacity concept and are not reused. The only Phase 1 types are `DEPOT`, `CUSTOMER_SITE` and `UNAUTHORIZED_ZONE`.

Geometry is polygon-only WGS84 in `(longitude, latitude)` order. The API accepts an open ring of 3–100 distinct vertices; persistence will store a canonical closed ring. Consecutive duplicates, zero-area and self-intersecting polygons are invalid. Boundary points are inside. Orientation is irrelevant and preserved. The encoded polygon is limited to 16 KiB. No PostGIS or map-provider dependency is approved: the proposed representation is PostgreSQL JSONB vertices plus numeric bounding-box columns, with application-owned pure-Java geometry and at most 500 ACTIVE geofences per Tenant.

The proposed `Geofence` aggregate carries Tenant, unique Tenant-scoped name, type, polygon, optional logical Organization location ID, alert configuration, lifecycle, audit facts and optimistic version. `DEPOT` and `CUSTOMER_SITE` require an active same-Tenant Organization location; `UNAUTHORIZED_ZONE` is free-standing and cannot reference one. The lifecycle is `DRAFT -> ACTIVE <-> DISABLED -> RETIRED`, with RETIRED terminal and configuration editable only in DRAFT or DISABLED.

Evaluation consumes only nonduplicate, TRUSTED, valid WGS84, IN_ORDER accepted positions no older than five minutes at evaluation. Delayed, out-of-order, future, stale and untrusted positions cannot change current geofence state. Ordering uses source timestamp and position UUID as deterministic tie-break. The first eligible observation initializes state silently. ENTERED or EXITED is confirmed only after two distinct consecutive eligible observations agree with the candidate side. There is no dwell event. Overlapping geofences evaluate independently, and an unauthorized-zone result is never suppressed by another overlap.

Confirmed unauthorized entry is `UNAUTHORIZED_ZONE_ENTERED`, severity HIGH and always alertable; discipline and exception-case creation are outside US-49. All same-Tenant Vehicles are evaluated; no Trip dependency or selector model is approved. A bounded Tracking-owned durable evaluation job is created atomically with an accepted position. Raw packets do not become cross-module events. Each transition has deterministic SHA-256 identity over Tenant, geofence, definition version, Vehicle, from/to state and confirming position.

The implemented outbound publication-port payload is the minimized `VehicleGeofenceTransitionedV1`, to be consumed by Notification only after its later implementation slice. It contains geofence ID, Vehicle ID, nullable Organization location ID, geofence type, transition, severity, source timestamp and definition version; it excludes coordinates, device/provider facts and person/Customer data. CS01 adds no durable event or Notification adapter. Operations integration is NONE for US-49; US-55 owns any later GPS-exception integration.

The proposed permissions are `GEOFENCE_VIEW`, `GEOFENCE_MANAGE` and `GEOFENCE_EVENT_VIEW`, with Tenant isolation only and no generic ABAC engine. Proposed APIs remain under `/api/v1/tracking/geofences`; CS01/CS02 implement no REST surface. The completed framework-neutral inbound ports are `GeofenceManagementUseCase`, `GeofenceQuery` and `GeofenceEvaluationUseCase`; outbound ports cover definition/state/transition/job persistence, explicit-Tenant Organization location lookup and transition publication. Organization's published `LocationLookup` includes an explicit `(tenantId, locationId)` operation backed by tenant-scoped persistence while retaining its established consumer-compatible operation. V77 implements the four Tracking-owned persistence tables and JDBC adapters described below. The next controlled task is `US-49-MANAGE-GEOFENCES-CS03-EVALUATION-TRANSITIONS-001`. Accounting remains 72/87 and US-48 remains on external-prerequisite hold.

## US-49 V77 persistence (CS02 complete)

At CS02 completion V77 was the Flyway head; V1–V76 are immutable. It creates only the four Tracking-owned tables below.
Organization `location_id` and Fleet `vehicle_id` remain logical UUID references without physical
cross-module foreign keys. PostgreSQL structural constraints supplement, but do not replace, domain rules.
No PostGIS extension, permission seed, API, outbox event or evaluator wiring is included.

#### Table: `tracking_geofence`

- **Purpose:** Tenant-owned geofence definition and lifecycle authority.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Tenant-leading indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Geofence identity |
| `tenant_id` | UUID | NO | - | Tenant scope; UNIQUE with `id` and `name` | Trusted Tenant |
| `name` | VARCHAR(160) | NO | - | Trimmed nonblank; UNIQUE with `tenant_id` | Operator name |
| `type` | VARCHAR(24) | NO | - | DEPOT, CUSTOMER_SITE, UNAUTHORIZED_ZONE | Frozen type |
| `polygon_vertices` | JSONB | NO | - | Array length 4–101; encoded size <=16 KiB | Canonical closed WGS84 ring |
| `min_longitude` | NUMERIC(10,7) | NO | - | -180..180; <= max | Derived bounding box |
| `max_longitude` | NUMERIC(10,7) | NO | - | -180..180 | Derived bounding box |
| `min_latitude` | NUMERIC(10,7) | NO | - | -90..90; <= max | Derived bounding box |
| `max_latitude` | NUMERIC(10,7) | NO | - | -90..90 | Derived bounding box |
| `location_id` | UUID | YES | NULL | Logical Organization reference; required for DEPOT/CUSTOMER_SITE and absent for UNAUTHORIZED_ZONE | Optional site |
| `alert_enter_enabled` | BOOLEAN | NO | - | TRUE for UNAUTHORIZED_ZONE | Entry alert policy |
| `alert_exit_enabled` | BOOLEAN | NO | - | - | Exit alert policy |
| `lifecycle` | VARCHAR(16) | NO | - | DRAFT, ACTIVE, DISABLED, RETIRED | Definition lifecycle |
| `version` | BIGINT | NO | 0 | >=0 | Optimistic version |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation time |
| `created_by` | UUID | NO | - | - | Creating actor |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update time |
| `updated_by` | UUID | NO | - | - | Last updating actor |

Indexes are `idx_tracking_geofence_active_bbox(tenant_id,lifecycle,min_longitude,max_longitude,min_latitude,max_latitude)`
and `idx_tracking_geofence_location(tenant_id,location_id)`. The activation-count persistence primitive
uses a Tenant-keyed transaction advisory lock; CS03/CS04 will enforce the 500-ACTIVE limit atomically.

#### Table: `tracking_vehicle_geofence_state`

- **Purpose:** Rebuildable current per-Vehicle/per-geofence membership and hysteresis state.
- **Primary Key:** (`tenant_id`, `geofence_id`, `vehicle_id`)
- **Multi-Tenant Key:** `tenant_id` (composite primary key and index)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | Composite PK; same-module FK with `geofence_id` | Tenant scope |
| `geofence_id` | UUID | NO | - | Composite PK; FK → `tracking_geofence(tenant_id,id)` RESTRICT | Definition |
| `vehicle_id` | UUID | NO | - | Composite PK; logical Fleet reference | Vehicle |
| `definition_version` | BIGINT | NO | - | >=0 | Evaluated definition version |
| `stable_state` | VARCHAR(8) | YES | NULL | INSIDE or OUTSIDE | Stable membership; null before initialization |
| `pending_candidate` | VARCHAR(8) | YES | NULL | INSIDE or OUTSIDE; coherent with pending fields | Pending side |
| `pending_count` | SMALLINT | NO | 0 | 0 or 1 through coherence check | Confirmation count |
| `pending_position_id` | UUID | YES | NULL | Logical Tracking position reference | First confirmation |
| `last_evaluated_position_id` | UUID | YES | NULL | Paired with source timestamp | Ordering tie-break |
| `last_evaluated_source_timestamp` | TIMESTAMPTZ | YES | NULL | Paired with position ID | Latest evaluated source time |
| `version` | BIGINT | NO | 0 | >=0 | Optimistic version |
| `created_at` | TIMESTAMPTZ | NO | - | - | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last update time |

Index `idx_tracking_geofence_state_vehicle(tenant_id,vehicle_id,geofence_id)` supports bounded memberships.

#### Table: `tracking_geofence_transition`

- **Purpose:** Append-only immutable geofence transition evidence.
- **Primary Key:** `id` (UUID, deterministic transition UUID)
- **Multi-Tenant Key:** `tenant_id` (Tenant-leading uniqueness and history indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Transition UUID |
| `tenant_id` | UUID | NO | - | Tenant scope | Trusted Tenant |
| `geofence_id` | UUID | NO | - | Same-module FK → `tracking_geofence(tenant_id,id)` RESTRICT | Definition |
| `vehicle_id` | UUID | NO | - | Logical Fleet reference | Vehicle |
| `location_id` | UUID | YES | NULL | Logical Organization reference | Optional site |
| `geofence_type` | VARCHAR(24) | NO | - | Frozen geofence values | Type snapshot |
| `transition` | VARCHAR(32) | NO | - | ENTERED, EXITED, UNAUTHORIZED_ZONE_ENTERED | Transition classification |
| `severity` | VARCHAR(8) | NO | - | NORMAL or HIGH | Severity |
| `source_timestamp` | TIMESTAMPTZ | NO | - | - | Confirming source time |
| `definition_version` | BIGINT | NO | - | >=0 | Definition snapshot version |
| `confirming_position_id` | UUID | NO | - | Same-module FK → `tracking_position(tenant_id,id)` RESTRICT | Confirming position |
| `from_state` | VARCHAR(8) | NO | - | INSIDE/OUTSIDE; differs from `to_state` | Previous membership |
| `to_state` | VARCHAR(8) | NO | - | INSIDE/OUTSIDE | Confirmed membership |
| `transition_identity` | UUID | NO | - | UNIQUE with `tenant_id` | Deterministic idempotency identity |
| `created_at` | TIMESTAMPTZ | NO | - | - | Persistence time |

Tenant-leading history indexes support geofence/source time, Vehicle/source time and partial unauthorized
source-time queries. Trigger `trg_tracking_geofence_transition_immutable` rejects UPDATE and DELETE. No
coordinates, geometry, raw payload, provider/device identity or person/customer data are stored.

#### Table: `tracking_geofence_evaluation_job`

- **Purpose:** Durable bounded evaluation work keyed idempotently by accepted Tracking position.
- **Primary Key:** (`tenant_id`, `position_id`)
- **Multi-Tenant Key:** `tenant_id` (composite primary key and due-job index)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | Composite PK; same-module FK with position | Tenant scope |
| `position_id` | UUID | NO | - | Composite PK; FK → `tracking_position(tenant_id,id)` RESTRICT | One logical job per position |
| `status` | VARCHAR(16) | NO | - | PENDING, PROCESSING, COMPLETED, FAILED | Job state |
| `attempt` | INTEGER | NO | 0 | >=0 | Claim attempt count |
| `next_attempt_at` | TIMESTAMPTZ | NO | - | - | Due time |
| `lease_owner` | VARCHAR(120) | YES | NULL | Both lease fields null or non-null | Current worker |
| `lease_until` | TIMESTAMPTZ | YES | NULL | Both lease fields null or non-null | Lease expiry |
| `created_at` | TIMESTAMPTZ | NO | - | - | Enqueue time |
| `updated_at` | TIMESTAMPTZ | NO | - | - | Last state change |

Index `idx_tracking_geofence_job_due(tenant_id,status,next_attempt_at,lease_until,position_id)` supports
bounded `FOR UPDATE SKIP LOCKED` claims. JDBC primitives implement idempotent enqueue, claim, renew,
release, retry, complete and expired-lease reclaim with owner-safe predicates. CS02 does not wire ingestion
or start a worker.

## US-49 geofence evaluation and transition production (CS03 complete)

Every newly accepted eligible Tracking position atomically enqueues exactly one Tenant/position evaluation
job inside the existing ingestion transaction. Duplicate facts enqueue nothing and rollback leaves no job.
Ingress performs no polygon evaluation and emits no per-position cross-module event.

One disabled-by-default, narrowly feature-flagged Tracking scheduler claims bounded V77 work using
`FOR UPDATE SKIP LOCKED`, fixed workers and a bounded queue. Default bounds are claim 16, four workers,
queue 32 and a two-minute lease. Expired leases recover; only the current unexpired owner may renew,
release, complete, retry or terminally fail work. Persistent retry is at least once. Queue/worker/backlog,
oldest-due, result and transition-latency observations use bounded metric dimensions.

Execution reloads the authoritative position using persisted Tenant and position ID, then revalidates
TRUSTED, Vehicle-associated, valid WGS84, IN_ORDER and at-most-five-minute eligibility. The evaluator
fails closed above 500 ACTIVE definitions for one Tenant. V77 bounding boxes reduce polygon candidates;
definitions outside the box without current state initialize silently as OUTSIDE using a lightweight
identity/version scan, while definitions with current state remain candidates so exits are detected.

Each candidate uses an owning Tracking transaction with definition lock/lifecycle/version revalidation and
serialized Tenant/geofence/Vehicle state locking, including concurrent first-row creation. CS01 geometry,
source-time plus UUID ordering and two-position hysteresis remain authoritative. Initial observation and
definition-version reset are silent. Confirmed state and deterministic immutable transition commit
atomically; overlap is independent. Unauthorized entry is `UNAUTHORIZED_ZONE_ENTERED`, HIGH and mandatory
alert intent. The publication port is invoked only after commit for configured transitions, but CS03 binds
a no-op adapter: CS05 still owns durable P1-01 and Notification activation. Flyway remains V77; no REST,
permission, frontend, event-contract or external dependency change is part of CS03.

Verification: focused CS03 PostgreSQL selection 22/22, complete Tracking Java 152/152, architecture 52/52,
full Maven 1,558 tests with zero failures/errors and 15 skipped in 09:38, Chromium evaluator-enabled ingress
580.9 msg/s sustained and 1,577.1 msg/s burst, latest p95 0.380 ms and history p95 0.273 ms. Accepted
database evidence used only `transport_logistics_acceptance`. US-49 remains implementation-in-progress;
next is `US-49-MANAGE-GEOFENCES-CS04-APIS-RBAC-AUDIT-001`. Accounting remains 72/87 and the US-48
external hold is unchanged.

## US-49 V78 permission seed (CS04A complete)

At CS04A completion V78 was the Flyway head; V1–V77 remain immutable. It changes only global Identity
RBAC metadata by seeding the three frozen active permission codes `GEOFENCE_VIEW`, `GEOFENCE_MANAGE` and
`GEOFENCE_EVENT_VIEW`. It conditionally grants them only to existing `ADMIN` and `LOCAL_MVP_ADMIN` roles
and creates neither roles nor non-administrative grants. The local Identity bootstrap catalogue is aligned
to the same codes and its canonical permission count is 181.

Clean V1→V78 and explicit V77→V78 paths pass on `transport_logistics_acceptance`; focused Identity/RBAC
and Tracking security compatibility is 46/46, complete Tracking is 154/154, full Maven is 1,560 tests with
zero failures/errors and 15 skipped in 09:34, and architecture is 52/52. Checkstyle reports zero configured
violations, PMD passes and SpotBugs reports zero findings/errors. No API, controller, frontend, event,
Notification, domain, geofence-table, public-contract or accounting change belongs to CS04A. US-49 remains
implementation-in-progress, accounting remains 72/87 and the US-48 external hold is unchanged. Next is
`US-49-MANAGE-GEOFENCES-CS04-APIS-RBAC-AUDIT-001-RERUN`.

## US-49 geofence APIs, RBAC and audit (CS04 complete)

CS04 implements the exact `/api/v1/tracking/geofences` definition, membership, transition and
unauthorized-transition route family. Explicit create/update/activate/disable/retire commands replace any
generic status mutation; DISABLED-to-ACTIVE is the reactivation path and RETIRED remains terminal. Definition
and membership lists are page-bounded to 100; transition history is source-time filtered with a deterministic
cursor and maximum 100. Only stable memberships are returned.

The web adapter resolves Tenant and actor facts through trusted `CurrentTenant` and passes them explicitly to
the application service. DEPOT/CUSTOMER_SITE validation consumes only Organization's published explicit-Tenant
`LocationLookup`; Tracking stores a logical UUID and has no Organization implementation, repository, entity,
SQL join or physical foreign key. Cross-Tenant identifiers are not-found-shaped.

`GEOFENCE_VIEW`, `GEOFENCE_MANAGE` and `GEOFENCE_EVENT_VIEW` are enforced independently in the HTTP chain and
at the direct use-case boundary. Existing `tracking_audit_event` persistence provides Tenant-scoped durable
idempotency claims for create/activate/disable/retire and safe management audit without a new schema. Audit
details exclude polygons, coordinates, raw telemetry, provider/device secrets, Driver PII and Customer data.
The Tenant-wide 500 ACTIVE limit is serialized with a Tenant advisory lock.

Final evidence: focused API/PostgreSQL 10/10, complete Tracking 164/164, security regression 51/51,
architecture 52/52 and Maven 1,570/0/0/15 in 10:07 all pass. Checkstyle reports zero violations, PMD passes,
SpotBugs reports zero findings, and the real PostgreSQL-backed Chromium gate measures 424.0 msg/s sustained
and 1,459.5 msg/s burst. All authoritative database evidence uses `transport_logistics_acceptance`. Flyway
remained V78 at CS04. Accounting remains 72/87 and US-48's external hold is unchanged.

## US-49 V79 Notification catalogue seed (CS05A complete)

At CS05A completion V79 was the Flyway head; V1–V78 remain immutable. It creates no table and
uses only Notification-owned catalogue tables. It seeds one global active version-1 IN_APP template for
`VEHICLE_GEOFENCE_TRANSITIONED_V1`, plus one enabled Tenant-scoped `ROLE` / `DISPATCHER` rule and its
existing policy row per current Tenant. The policy has no quiet hours, zero suppression and no escalation.

The template renders only Vehicle ID, geofence ID/type, transition and source timestamp. Coordinates,
polygon, device/provider facts, credentials, raw telemetry, Driver PII and Customer data are neither
required nor rendered. CS05 activates the contract through Tracking's shared P1-01 durable publisher and
Notification's registered durable bridge. The bridge resolves the V79 Tenant rule, eligible same-Tenant
Dispatcher recipients and the version-1 template, then persists the existing Notification model's IN_APP
equivalent of a delivery attempt. Execution identity prevents duplicates during consumer or producer replay.

Clean V1→V79 passes on `transport_logistics_acceptance` and proves initial/pending silence, one confirmed
transition, one outbox event, one Tenant-A Dispatcher notification, Tenant-B exclusion, minimized payload
and rendered-content privacy, and replay idempotency. Tracking is 171/171, Notification is 164/164,
security/privacy is 78/78, full Maven is 1,583/0/0/15 in 10:16, architecture is 52/52, and the configured
static gates pass. Accounting remains 72/87 and US-48's external hold is unchanged. Next task:
`US-49-MANAGE-GEOFENCES-CS06-FRONTEND-001`.

## US-49 V80 geofence index hardening (CS07A complete)

V80 is the current Flyway head; V1–V79 remain immutable and no V81 exists. It adds only two
Tracking-owned physical-design indexes. `idx_tracking_geofence_job_global_due` is ordered by
`(next_attempt_at, tenant_id, position_id)` and includes `status` and `lease_until`, matching the global
bounded `FOR UPDATE SKIP LOCKED` due-job claim. Partial
`idx_tracking_geofence_active_bbox_upper` is ordered by `(tenant_id, max_longitude)`, includes the other
bbox coordinates and definition ID, and contains only ACTIVE definitions.

The candidate lookup remains explicitly Tenant-scoped and logically unchanged: it unions definitions whose
bbox contains the position with ACTIVE definitions already represented in the same Vehicle's current state,
then performs the bounded definition lookup. Coordinate parameters are explicitly cast to PostgreSQL
`numeric` so the indexed numeric columns are not cast to `double precision`. This preserves exit detection,
initialization, overlap, lifecycle, ordering and the 500-candidate bound.

At 5,000 definitions the pre-V80 bbox plan sequentially scanned and removed 4,990 rows (2.405 ms); V80 uses
the new index-only scan and existing state-vehicle index and completes in 0.118 ms. At 5,000 jobs the pre-V80
claim sequentially scanned and sorted (1.112 ms); V80 uses the ordered claim index with no sequential scan or
explicit sort and completes in 0.036 ms. Clean V1→V80 and V79→V80 both pass on
`transport_logistics_acceptance`.

Verification: focused PostgreSQL 31/31, full Maven 1,595 tests with zero failures/errors and 15 skipped,
architecture 52/52, configured static gates, TypeScript/build, Vitest 299/299, Chromium US-49 6/6 and
performance 1/1 all pass. Performance measured 1,083.4 msg/s sustained, 1,423.4 msg/s burst, latest p95
16.2 ms and history p95 13.3 ms. No public API, permission, event, frontend, lifecycle, accounting or external
dependency changed. Accounting remains 72/87; US-48's external hold is unchanged. Next task:
`US-49-MANAGE-GEOFENCES-TECHNICAL-CLOSURE-001`.

## US-49 PostgreSQL concurrency and performance (CS07 complete)

The unchanged 31-test PostgreSQL race matrix passed three consecutive executions on
`transport_logistics_acceptance`: 31/31 each and 93/93 combined. The 500-ACTIVE-definition initialization
workloads completed in 822 ms, 2,581 ms and 917 ms, each producing exactly 500 silent initial states and
zero transition/outbox events. The matrix covers duplicate initialization and confirmation serialization,
source-time ordering and rewind prevention, lifecycle/version races, the 499+2 activation boundary,
Tenant-independent limits, immutable transition uniqueness, overlapping-geofence independence, durable
job claim/lease ownership and recovery, idempotent publication/Notification consumption, controlled
failure atomicity, and bounded-worker saturation release.

Across the three executions, the approximately 5,000-row bbox lookup used
`idx_tracking_geofence_active_bbox_upper` as an index-only scan, returned 10 candidates, and completed in
0.097–0.107 ms without filtering the full Tenant population. The approximately 5,000-row global due-job
claim used `idx_tracking_geofence_job_global_due` as an ordered index scan, returned the bounded 16 rows,
and completed in 0.027–0.033 ms without a full sequential scan or global explicit sort. No deadlock,
connection leak, pool exhaustion, timeout storm or global isolation-level change was observed.

Closure regression evidence: Tracking 183/183, Notification 164/164, security/Tenant/privacy/RBAC
152/152, full Maven 1,595 tests with zero failures/errors and 15 skipped in 10:44, architecture 52/52,
configured Checkstyle/PMD/SpotBugs gates, TypeScript and production build, Vitest 299/299, and real
PostgreSQL-backed Chromium 7/7 all pass. Signed-ingress performance measured 1,287.7 msg/s sustained,
1,442.4 msg/s burst, latest p95 10.5 ms and history p95 14.8 ms. The global ESLint baseline remains 71
unrelated Delivery findings; CS07 changed no frontend files and introduced no lint debt. No production,
schema, API, event, permission, lifecycle, frontend, dependency or accounting change was required. V80
remains current, no V81 exists, US-49 remains `IMPLEMENTATION_IN_PROGRESS`, and US-48's external hold is
unchanged. Next task: `US-49-MANAGE-GEOFENCES-TECHNICAL-CLOSURE-001`.

## US-49 technical closure

Technical closure is `COMPLETE`: the frozen domain, geometry, lifecycle, location ownership, eligibility,
ordering, hysteresis, unauthorized-zone, overlap, persistence, API, three-permission RBAC, Tenant,
idempotency, audit, durable `VehicleGeofenceTransitionedV1`, Notification and existing-stack frontend
contracts match the implementation. No missing use case, ownership drift, implementation defect, product
decision or unapproved dependency was found. Independent closure reruns passed PostgreSQL/API/security/
Notification 40/40 on `transport_logistics_acceptance`, architecture 52/52 and focused frontend 8/8.
The latest complete evidence remains Maven 1,595/0/0/15, Tracking 183/183, Notification 164/164,
security/Tenant/privacy/RBAC 152/152, Vitest 299/299 and Chromium 7/7. V80 remains current and no V81
exists. US-49 is `IMPLEMENTATION_COMPLETE / READY_FOR_FINAL_ACCEPTANCE`; accounting stays 72/87 and
US-48's external hold is unchanged. Next task: `US-49-MANAGE-GEOFENCES-FINAL-ACCEPTANCE-001`.

## US-49 independent final acceptance

`US-49-MANAGE-GEOFENCES-FINAL-ACCEPTANCE-001` is `PASS`. Fresh acceptance-database evidence used only
`transport_logistics_acceptance`: focused geofence/PostgreSQL/API/security/Notification tests passed 76/76;
the complete Maven verification passed 1,595 tests with zero failures, zero errors and 15 skipped;
architecture passed 52/52; Checkstyle, PMD and SpotBugs passed; TypeScript and production build passed;
focused Vitest passed 8/8 and the complete frontend suite passed 299/299. The fresh real PostgreSQL-backed
Chromium run passed all six US-49 journeys plus the performance gate (7/7 total), measuring 1,147.3 msg/s
sustained, 1,731.5 msg/s burst, 8.3 ms latest-query p95 and 13.9 ms history-query p95. A test-only Playwright
startup correction explicitly enables the documented geofence evaluator feature flag; production defaults and
contracts did not change. V80 remains the Flyway head and no V81 exists. US-49 is `COMPLETE / ACCEPTED`,
accounting is 73/87 with 14 remaining, US-48's external hold is unchanged, and the next task is
`US-50-MONITOR-SPEED-PRODUCT-DECISIONS-001`.

## US-50 Monitor Speed — CS01 domain and ports implemented

US-50 is `IMPLEMENTATION_IN_PROGRESS`; CS01 is COMPLETE, accounting remains 73/87, Flyway remains
V80 and US-48's external hold is unchanged. Tracking owns evaluation and evidence. The canonical signal is
an eligible nonduplicate TRUSTED/IN_ORDER/source-time-associated `PositionEvent.speedKph` no more than five
minutes old; missing speed remains UNKNOWN and historical late evaluation produces no alert. Kilometres per
hour is canonical, tolerance is zero, equality is normal and invalid speed is rejected by US-48 normalization.

Phase 1 uses explicit Tracking-owned route/version operational configuration with an ACTIVE Tenant fallback.
There is no external road-law provider, segment map matching, Vehicle-class rule, Vehicle rule or default
threshold, and the product makes no authoritative live-road-limit claim. Two consecutive eligible samples
above the same rule version confirm a `SpeedingEpisode`; one eligible sample at/below threshold clears it.
The first episode is WARNING; a new episode under the same rule within ten minutes is a HIGH repeat. CS01
models one minimized `VehicleSpeedingDetectedV1` publication request per confirmed episode; durable publication
and Notification consumption remain unimplemented. Driver, Trip and route attribution is nullable and comes
solely from the published source-time Trip lookup;
missing attribution never drops Vehicle evidence and Tracking causes no Driver/disciplinary/payroll effect.

The exact permissions are `SPEED_MONITOR_VIEW`, `SPEED_MONITOR_MANAGE` and `SPEED_EVENT_VIEW`. The planned
bounded API family is `/api/v1/tracking/speed-monitoring`; the existing frontend stack supplies rule, current
state and episode-history operator views without US-54 dashboard or a map dependency. V81 is likely for
Tracking-owned `tracking_speed_rule`, `tracking_speed_state`, `tracking_speed_episode` and
`tracking_speed_evaluation_job`, but CS01 creates no migration. Framework-free domain models implement rule
lifecycle, threshold resolution, eligibility, state transitions, repeat/severity semantics, source ordering
and deterministic identity; domain-neutral ports cover evaluation, management, queries, future persistence,
attribution and publication. Technical closure may use signed
fixtures; final real-speed fidelity requires verified physical device/provider speed and otherwise remains
`ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`.

## US-50 V81 speed-monitoring persistence (CS02 complete)

V81 is the current Flyway head; V1–V80 remain immutable and no V82 exists. CS02 creates exactly four
Tracking-owned Tenant-scoped tables and JDBC adapters. It adds no API, permission, audit/outbox table,
Notification catalogue, frontend, scheduler, worker or cross-module physical foreign key. US-50 remains
`IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and US-48's external hold is unchanged.

#### Table: `tracking_speed_rule`

- **Purpose:** Versioned Tenant or route-version speed threshold configuration.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed through Tenant-leading uniqueness/lookups)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; UNIQUE with `tenant_id` | Deterministic rule identity |
| `tenant_id` | UUID | NO | - | Tenant scope | Owning Tenant |
| `name` | VARCHAR(120) | NO | - | Trimmed and non-empty | Operator name |
| `scope` | VARCHAR(16) | NO | - | `TENANT`, `ROUTE_VERSION` | Resolution scope |
| `route_id` | UUID | YES | NULL | Logical Routing reference; required only for route scope | Route identity |
| `route_version` | VARCHAR(120) | YES | NULL | Trimmed; required only for route scope | Immutable route version |
| `threshold_kph` | NUMERIC(7,3) | NO | - | `> 0 AND <= 400` | Canonical threshold |
| `lifecycle` | VARCHAR(16) | NO | - | `DRAFT`, `ACTIVE`, `DISABLED`, `RETIRED` | Rule lifecycle |
| `version` | BIGINT | NO | - | `> 0`; optimistic update token | Rule version |
| `effective_at` | TIMESTAMPTZ | YES | NULL | Required for ACTIVE | Activation instant |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Creation instant |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | - | Last update instant |

Partial unique indexes enforce one ACTIVE Tenant fallback and one ACTIVE rule per
`(tenant_id,route_id,route_version)`. Tenant-leading partial covering indexes serve route/version and fallback
resolution.

#### Table: `tracking_speed_state`

- **Purpose:** One durable current speed-monitoring state per Tenant and Vehicle.
- **Primary Key:** `(tenant_id, vehicle_id)`
- **Multi-Tenant Key:** `tenant_id` (UUID, leading primary-key column)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | Composite PRIMARY KEY | Owning Tenant |
| `vehicle_id` | UUID | NO | - | Composite PRIMARY KEY; logical Fleet reference | Vehicle identity |
| `state` | VARCHAR(16) | NO | - | `UNKNOWN`, `NORMAL`, `SPEEDING` | Current state |
| `availability` | VARCHAR(32) | NO | - | `AVAILABLE`, `NOT_EVALUATED`, `CONFIGURATION_UNAVAILABLE` | Evaluation availability |
| `effective_rule_id` | UUID | YES | NULL | Paired with positive rule version | Applied rule |
| `effective_rule_version` | BIGINT | YES | NULL | Positive when present | Applied version |
| `candidate_position_id` | UUID | YES | NULL | Candidate tuple is all-present or all-absent | First sample |
| `candidate_source_timestamp` | TIMESTAMPTZ | YES | NULL | Candidate tuple | First-sample source time |
| `candidate_observed_speed_kph` | NUMERIC(7,3) | YES | NULL | `0..400` when present | Candidate speed |
| `candidate_sample_count` | SMALLINT | NO | `0` | `0` or `1` coherently | Pending confirmation count |
| `active_episode_id` | UUID | YES | NULL | Required exactly when SPEEDING | Active episode |
| `last_evaluated_source_timestamp` | TIMESTAMPTZ | YES | NULL | Paired with position ID | Ordering watermark |
| `last_evaluated_position_id` | UUID | YES | NULL | Paired with source timestamp | Ordering tie-breaker |
| `version` | BIGINT | NO | `0` | `>= 0`; optimistic token | State version |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Creation instant |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | - | Last update instant |

#### Table: `tracking_speed_episode`

- **Purpose:** Append-preserved confirmed speeding evidence and repeat attribution.
- **Primary Key:** `id` (domain-generated deterministic UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, Tenant-leading unique/history indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; no database generation | Deterministic episode identity |
| `tenant_id` | UUID | NO | - | UNIQUE with `id` | Owning Tenant |
| `vehicle_id` | UUID | NO | - | Logical Fleet reference | Vehicle evidence owner |
| `trip_id` | UUID | YES | NULL | Logical Trip reference | Source-time Trip attribution |
| `driver_id` | UUID | YES | NULL | Logical Driver reference | Source-time Driver attribution |
| `route_id` | UUID | YES | NULL | Logical Routing reference | Source-time route attribution |
| `route_version` | VARCHAR(120) | YES | NULL | Logical Routing reference | Route version |
| `rule_id` | UUID | NO | - | Logical immutable rule reference | Applied rule |
| `rule_version` | BIGINT | NO | - | `> 0` | Applied rule version |
| `threshold_source` | VARCHAR(16) | NO | - | `ROUTE_CONFIG`, `TENANT_CONFIG` | Resolution source |
| `effective_threshold_kph` | NUMERIC(7,3) | NO | - | `> 0 AND <= 400` | Effective threshold |
| `start_source_timestamp` | TIMESTAMPTZ | NO | - | Chronology constrained | First-sample time |
| `confirmation_source_timestamp` | TIMESTAMPTZ | NO | - | `>= start` | Confirmation time |
| `end_source_timestamp` | TIMESTAMPTZ | YES | NULL | NULL while active; `>= confirmation` | Closure time |
| `max_observed_speed_kph` | NUMERIC(7,3) | NO | - | `0..400`; cannot decrease | Maximum speed |
| `eligible_above_threshold_sample_count` | INTEGER | NO | - | `>= 2`; cannot decrease | Evidence count |
| `severity` | VARCHAR(8) | NO | - | `WARNING`, `HIGH` | Frozen severity |
| `repeat_count` | INTEGER | NO | - | `>= 0` | Ten-minute repeat count |
| `first_candidate_position_id` | UUID | NO | - | Logical Tracking position reference | First evidence identity |
| `confirming_position_id` | UUID | NO | - | Logical Tracking position reference | Confirming evidence identity |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Immutable | Creation instant |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Monotonic while active | Last progress instant |

One partial unique index allows only one active episode per Tenant and Vehicle. Vehicle history, severity
history and same-rule/version closed-repeat indexes use stable descending source-time/ID ordering. Trigger
`trg_tracking_speed_episode_protection` rejects DELETE, every mutation after closure, immutable identity-field
changes and backwards active progress.

#### Table: `tracking_speed_evaluation_job`

- **Purpose:** Durable idempotent work queue for accepted Tracking positions.
- **Primary Key:** `(tenant_id, position_id)`
- **Multi-Tenant Key:** `tenant_id` (UUID, included in key and every repository mutation)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id` | UUID | NO | - | Composite PRIMARY KEY; same-module FK to `tracking_position` | Owning Tenant |
| `position_id` | UUID | NO | - | Composite PRIMARY KEY; same-module FK to `tracking_position` | Idempotent work identity |
| `vehicle_id` | UUID | NO | - | Logical Fleet reference; validated from position | Vehicle identity |
| `source_timestamp` | TIMESTAMPTZ | NO | - | UTC-compatible source time | Evaluation time |
| `status` | VARCHAR(16) | NO | - | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED` | Job lifecycle |
| `attempt_count` | INTEGER | NO | `0` | `>= 0` | Claim attempts |
| `next_attempt_at` | TIMESTAMPTZ | NO | - | Global due-order key | Next eligibility |
| `lease_owner` | VARCHAR(120) | YES | NULL | Required exactly while PROCESSING | Worker owner |
| `lease_until` | TIMESTAMPTZ | YES | NULL | Required exactly while PROCESSING | Lease deadline |
| `last_error_code` | VARCHAR(120) | YES | NULL | Trimmed non-empty when present | Safe failure code |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Creation instant |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | - | Last transition instant |

`idx_tracking_speed_job_global_due(next_attempt_at,tenant_id,position_id) INCLUDE(status,lease_until)` aligns
with the global bounded claim order and `FOR UPDATE SKIP LOCKED`. JDBC mutations are Tenant- and owner-qualified;
expired leases are reclaimable and stale owners cannot renew, release, complete, retry or fail work.

Clean V1→V81 and V80→V81 pass on `transport_logistics_acceptance`; focused persistence/domain/ownership is
32/32, Tracking is 212/212, Trip is 101/101, architecture is 52/52 and full Maven is 1,624 tests with zero
failures/errors and 15 skipped. Checkstyle, PMD, SpotBugs and `git diff --check` pass. Next:
`US-50-MONITOR-SPEED-CS03-EVALUATION-EPISODES-001`.

## US-50 runtime evaluation and episodes (CS03 complete)

CS03 atomically adds one idempotent V81 speed-evaluation job for every accepted trusted, in-order, recent,
Vehicle-associated Tracking position carrying valid normalized `speedKph`. Missing speed and ineligible
positions do not create work. The one global `SpeedEvaluationCoordinator` is feature-gated by
`app.tracking.speed-evaluator.enabled` (disabled by default), claims through the V81 global due-job index and
`FOR UPDATE SKIP LOCKED`, and uses bounded claims, a fixed worker pool, bounded queue, owner-qualified lease
renewal/release, maximum-five-attempt retry and privacy-safe error codes. There is no Tenant-, Vehicle- or
device-specific scheduler.

Evaluation orders observations by `(sourceTimestamp, positionId)` and locks Tenant/Vehicle state in PostgreSQL,
so delayed/replayed work cannot rewind evidence and different Vehicles are not globally serialized. Trip
attribution uses only `VehicleTripAssignmentLookup.findAt(tenantId,vehicleId,sourceTimestamp)` through the
Trip-owned JDBC provider. Empty or safely failed attribution leaves nullable Trip/Driver/route/version facts
and continues with the Tenant fallback. An ACTIVE matching route/version rule wins over the ACTIVE Tenant
fallback; absence of both produces `CONFIGURATION_UNAVAILABLE` without an episode.

The first above-threshold sample is a silent candidate and a second distinct consecutive sample under the same
rule version confirms one deterministic episode. Continued speeding updates the same episode; one eligible
sample at/below threshold closes it, while missing/ineligible data never falsely clears it. Active episodes
retain their frozen rule facts. Same-rule episodes inside the inclusive ten-minute repeat window are HIGH and
increment repeat count; first, different-rule and outside-window episodes are WARNING. Concurrent confirmation
converges on one episode and one logical `SpeedingEpisodePublisherPort` invocation. Publication occurs only at
confirmation and uses the confirming speed/source time with the minimized CS01 model. CS03 wires an explicit
default no-op publisher only; durable P1-01 publication and Notification consumption remain deferred to CS05.
There is no Driver mutation, API, permission seed, management audit flow or frontend behavior in CS03.

Accepted evidence uses only `transport_logistics_acceptance`: focused runtime/domain/architecture 80/80,
Tracking 227/227, Trip 101/101, architecture 52/52 and complete Maven 1,639 tests with zero failures/errors and
15 skipped. A signed speed-bearing Chromium ingress smoke with the evaluator enabled sustained 432.7 msg/s and
burst 1,563.8 msg/s. Checkstyle, PMD, SpotBugs, V1→V81 and `git diff --check` pass; V82 is absent. US-50 remains
`IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87 and US-48's external hold is unchanged. Next:
`US-50-MONITOR-SPEED-CS04-APIS-RBAC-AUDIT-001`.

## US-50 management/query APIs, RBAC and audit (CS04 complete)

CS04 implements the Tenant-scoped `/api/v1/tracking/speed-monitoring` rule, state and episode family using
V81 evidence tables. Rule/state page sizes default to 20 and cap at 100. Episode history requires an ordered
UTC range no greater than 31 days, defaults to 100, caps at 500 and uses stable descending
`(startSourceTimestamp,id)` cursor ordering. Public DTOs contain configured operational threshold and frozen
episode facts only; candidate/position identity, coordinates, raw telemetry, device/provider/IMEI facts,
credentials and Driver/Customer PII remain excluded.

V82 is a narrow permission migration containing no schema/index/catalogue change. It seeds exactly
`SPEED_MONITOR_VIEW`, `SPEED_MONITOR_MANAGE` and `SPEED_EVENT_VIEW` and grants them only to existing `ADMIN`
and `LOCAL_MVP_ADMIN`. Literal HTTP matchers and a secured use-case decorator enforce the permissions
independently. Tenant comes solely from authenticated `CurrentTenant`; repositories remain Tenant-qualified
and foreign-Tenant IDs are not-found-shaped.

Create and lifecycle commands use persistent Tenant-scoped idempotency claims in existing
`tracking_audit_event`; same request replay is safe, changed reuse conflicts and another Tenant is independent.
PUT uses optimistic concurrency and V81 uniqueness remains authoritative for active fallback and route rules.
Successful management commands produce minimized safe audit facts, while reads, telemetry and evaluation do
not create management audit noise. CS04 adds no frontend, durable event adapter, Notification catalogue/
consumer, Driver mutation or evaluation semantic change. Accepted evidence on `transport_logistics_acceptance`
is focused 72/72, security/permission 12/12, Tracking 238/238, Trip 101/101, architecture 52/52 and Maven
1,650 tests with zero failures/errors and 15 skipped. Flyway V1→V82 and V81→V82 pass; V83 is absent. Next:
`US-50-MONITOR-SPEED-CS05-NOTIFICATION-INTEGRATION-001`.

## US-50 durable Notification integration (CS05 complete)

CS05 replaces the temporary no-op publication adapter with atomic P1-01 outbox publication of
`VehicleSpeedingDetectedV1`. The canonical event and aggregate ID is the deterministic SpeedingEpisode UUID,
the aggregate type is `SPEEDING_EPISODE`, producer is `TRACKING`, and `occurredAt` is the confirmation source
timestamp. The exact minimized payload is speed episode and Vehicle identity, nullable Driver/Trip/route/
route-version attribution, observed and configured effective speed, threshold source, rule identity/version,
WARNING/HIGH severity, confirmation source time and repeat count. Coordinates, Position/device/provider
identity, raw telemetry, credentials and Driver/Customer PII are forbidden.

The registered Notification bridge validates the exact version-1 envelope and payload, fails malformed events
permanently, maps WARNING to Notification WARNING and HIGH to Notification CRITICAL, and relies on the existing
Notification execution key for Tenant/event/rule/channel/recipient replay idempotency. V83 changes no Tracking
schema: it is a narrow Notification catalogue seed for one IN_APP template and same-Tenant ROLE/DISPATCHER
rule. Continued packets in one episode do not publish, and no Driver, payroll, licence, disciplinary or
Operations behavior is introduced.

Accepted CS05 evidence used only `transport_logistics_acceptance`: the PostgreSQL WARNING/repeat-HIGH,
outbox, recipient, Tenant-B, privacy, non-flooding and replay journey passes; Tracking is 240/240, Notification
is 165/165, architecture is 52/52, and complete Maven is 1,656 tests with zero failures/errors and 15 skipped.
Checkstyle, PMD and SpotBugs pass. The signed Chromium ingestion smoke sustains 461.9 messages/second, reaches
1,498.7 messages/second burst, and records 18.2 ms latest and 17.9 ms history p95. Flyway V1→V83 and V82→V83
pass; V84 is absent. US-50 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and US-48's
external hold is unchanged. Next: `US-50-MONITOR-SPEED-CS06-FRONTEND-001`.

## US-50 operator frontend (CS06 complete)

US-50 CS06 adds the permission-aware React operator workflow for the existing V83 speed-monitoring contracts.
`SPEED_MONITOR_VIEW` exposes rule and current-state reads, `SPEED_MONITOR_MANAGE` exposes rule creation,
editing and lifecycle commands, and `SPEED_EVENT_VIEW` exposes bounded episode history/detail. Backend
authorization remains authoritative, including literal `/api/v1/tracking/speed-monitoring` denial.

Rules support Tenant fallback and route/version configuration, thresholds greater than zero and at most 400
km/h, stable per-attempt idempotency keys, exact optimistic versions, and DRAFT/ACTIVE/DISABLED/RETIRED
lifecycle controls. Retirement is permanent and the UI offers no delete action. Current state renders UNKNOWN,
NORMAL and SPEEDING truthfully; unavailable or missing data is never presented as zero. Episode history uses the
server cursor and an ordered UTC range of at most 31 days, preserves exact WARNING/HIGH Tracking severity and
repeat evidence, and does not expose coordinates, raw telemetry, provider/device credentials, Customer data or
inferred Driver identity. The UI does not claim live legal limits or provide discipline, a map or US-54 dashboard.

Verification passed with focused Vitest 10/10, full Vitest 309/309, TypeScript, changed-file ESLint, production
build, real PostgreSQL-backed Chromium 10/10, focused API/security 5/5 and architecture 52/52. Flyway remains
V83 and V84 is absent. US-50 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and US-48's
external hold is unchanged. Next: `US-50-MONITOR-SPEED-CS07-POSTGRES-CONCURRENCY-PERFORMANCE-001`.

## US-50 PostgreSQL concurrency and performance (CS07 complete)

The V83 implementation passes the PostgreSQL concurrency/race matrix 41/41 on three consecutive runs, with
deterministic state/episode/event behavior, bounded disjoint `FOR UPDATE SKIP LOCKED` claims, lease recovery,
Tenant isolation, immutable evidence and Notification replay idempotency. At 5,000 rows, all critical global
claim, rule, state, repeat and episode-history queries use their intended V81 indexes; V84 is not required.
Signed ingress with evaluation and durable Notification enabled measured 441.3 msg/s sustained and 1,265.5
msg/s burst, with latest p95 19.6 ms and history p95 18.4 ms. Full Maven passes 1,656 tests with zero
failures/errors and 15 skipped; Chromium is 11/11 and architecture is 52/52. US-50 remains
`IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and technical closure is next.

## US-50 technical closure complete

Source-to-implementation reconciliation confirms the V81-V83 speed-monitoring implementation satisfies the
frozen domain, Tenant, API/RBAC, audit, durable event/Notification, frontend, concurrency, query-plan,
performance and privacy contracts. Representative closure revalidation passes 73/73 backend tests and 10/10
focused frontend tests; no implementation defect, product decision or V84 index is required. US-50 is
`IMPLEMENTATION_COMPLETE / TECHNICAL_CLOSURE_COMPLETE`, but receives no acceptance credit: verified physical
device/provider speed and unit fidelity remains an external final-acceptance evidence gate. Accounting stays
73/87, US-48 remains externally blocked with no acceptance inheritance, and the next task is
`US-50-MONITOR-SPEED-FINAL-ACCEPTANCE-001`.

## US-50 final acceptance external hold

Independent final acceptance reconfirmed the technical implementation with 78/78 representative backend and
10/10 focused frontend tests. It did not receive acceptance credit because no genuine physical device/provider
speed field, native-unit mapping, normalization proof or physical episode-to-Notification journey is available.
The exact status is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM` with reason
`PHYSICAL_SPEED_FIDELITY_EVIDENCE_PENDING`. Fixtures and simulators remain valid technical evidence but cannot
substitute for physical fidelity. Accounting remains 73/87, Flyway remains V83, US-48 remains independently
externally blocked, and Wave C proceeds with
`US-52-MONITOR-ROUTE-DEVIATIONS-PRODUCT-DECISIONS-001`.

## US-52 Route Deviation Monitoring Frozen Product Decision

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS01_COMPLETE`; accounting remains 73/87 and Flyway is V85
after the Trip route-revision assignment prerequisite. Tracking owns planned-versus-actual comparison, current state, durable evaluation jobs,
immutable deviation episodes, operational review and minimized durable publication. Routing retains route,
revision, ordered immutable geometry and disruption ownership; Trip retains source-time assignment authority.

Tracking reuses `VehicleTripAssignmentLookup.findAt(tenantId,vehicleId,sourceTimestamp)`, with Trip populating
canonical `REVISION:<positive-integer>` route versions. Routing will publish the additive Tenant-qualified
`PlannedRouteGeometryLookup`, returning a 2–2,000-point immutable WGS84 `(longitude,latitude)` revision
snapshot. Trip's V85 snapshot and source-time lookup are implemented. CS01 completes the validated immutable
published geometry result, exact Tenant/route/revision lookup and truthful provider; current Routing data lacks
immutable coordinates, so the provider returns empty without foreign lookup, synthesized chord or latest
fallback until CS02 persistence. Missing attribution/geometry is NOT_EVALUATED.

Each route revision has one ACTIVE Tracking tolerance rule from 10 through 5,000 metres. Effective tolerance
equals configured tolerance plus known accuracy from 0 through 1,000 metres; missing accuracy is NOT_EVALUATED.
Minimum local tangent-plane point-to-polyline distance is used, equality is on-route, two distinct consecutive
outside observations confirm and one inside observation clears. Eligible inputs are nonduplicate TRUSTED,
IN_ORDER, Vehicle-associated valid WGS84 positions no older than five minutes.

Episodes use deterministic identity and WARNING (`distance <= 2x effective tolerance`) or HIGH (`distance >
2x`). HIGH requires operational review; APPROVED/REJECTED annotates evidence and never mutates Routing or
suppresses monitoring. Later Tracking tables are rule, state, episode, immutable review and durable evaluation
job. The API family is `/api/v1/tracking/route-deviations`; permissions are ROUTE_DEVIATION_VIEW,
ROUTE_DEVIATION_MANAGE, ROUTE_DEVIATION_EVENT_VIEW and ROUTE_DEVIATION_APPROVE.

Detection and bounded HIGH/rejection escalation use the shared P1-01 outbox to Notification only; coordinates,
geometry, provider/device facts, credentials and personal data are excluded. Technical acceptance may use
deterministic fixtures; independent final acceptance requires safe physical position and accuracy fidelity.
The CS01 Tracking foundation retains the deterministic local tangent-plane point-to-polyline calculator,
10–5,000 metre tolerance, 0–1,000 metre accuracy eligibility, effective-tolerance arithmetic, inclusive
inside boundary, WARNING/HIGH rules and non-downgrading severity. Availability distinguishes absent Trip,
route/revision, geometry/rule, accuracy, ordering/trust/coordinate and provider/configuration conditions.
Domain and ports remain framework-neutral and dormant workflow/persistence/event surfaces are not activated.

## US-52 CS02 V88 Persistence

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS02_COMPLETE`; accounting remains 73/87. V88 adds
Tenant-scoped Tracking persistence for tolerance rules, stable Vehicle state with complete
candidate evidence, immutable episode evidence and append-only review evidence. The repositories
implement the dormant CS01 ports with Tenant predicates, bounded reads, source-time ordering,
advisory serialization and optimistic lock versions. Routing geometry is consumed only through
`PlannedRouteGeometryLookup`; Tracking has no Routing/Trip SQL, entity, repository or physical FK.

### Table: `tracking_route_deviation_rule`

- **Purpose:** Versioned Tracking-owned tolerance configuration for one route revision.
- **Primary Key:** `(tenant_id, id)`
- **Multi-Tenant Key:** `tenant_id` (indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id`, `id` | UUID | NO | - | composite primary key | Tenant and rule identity |
| `route_id` | UUID | NO | - | logical Routing reference | Route identity |
| `route_version` | VARCHAR(120) | NO | - | `REVISION:<positive integer>` | Exact revision |
| `configured_tolerance_meters` | NUMERIC(10,3) | NO | - | 10–5,000 | Frozen tolerance |
| `rule_version` | BIGINT | NO | - | positive; Tenant revision unique | Version |
| `active`, `created_at`, `updated_at` | BOOLEAN, TIMESTAMPTZ, TIMESTAMPTZ | NO | governed defaults | active-rule partial unique index | Lifecycle evidence |

### Table: `tracking_route_deviation_state`

- **Purpose:** One stable evaluation state and optional complete candidate per Tenant/Vehicle.
- **Primary Key:** `(tenant_id, vehicle_id)`
- **Multi-Tenant Key:** `tenant_id` (indexed)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `tenant_id`, `vehicle_id` | UUID | NO | - | composite primary key | Scope and Vehicle |
| `last_position_id`, `last_source_timestamp` | UUID, TIMESTAMPTZ | NO | - | Tenant/source ordering index | Latest processed source |
| `trip_id`, `route_id`, `route_version`, `rule_id`, `rule_version` | UUID/VARCHAR/BIGINT | YES | - | logical foreign references; no physical cross-module FK | Attribution/rule snapshot |
| `candidate_*` | UUID/TIMESTAMPTZ/NUMERIC | YES | NULL | all-or-nothing completeness constraint | First/latest candidate identity, coordinates, accuracy, distances and counters |
| `active_episode_id` | UUID | YES | NULL | Tracking-local logical identity | Open episode |
| `lock_version`, `created_at`, `updated_at` | BIGINT/TIMESTAMPTZ | NO | governed defaults | nonnegative lock | Concurrency/audit timestamps |

### Table: `tracking_route_deviation_episode`

- **Purpose:** Durable deterministic deviation episode evidence.
- **Primary Key:** `(tenant_id, id)`
- **Multi-Tenant Key:** `tenant_id` (indexed)

The table stores Tenant/Vehicle/Trip/Driver/route/revision/rule snapshots, lifecycle and severity,
start/latest/end source identity and timestamps, maximum distance and configured/effective
tolerance evidence, review requirement, lock version and creation/update timestamps. Constraints
bound lifecycle, severity and nonnegative numeric values. A Tenant/Vehicle partial unique index
permits only one open episode.

### Table: `tracking_route_deviation_review`

- **Purpose:** Immutable versioned review evidence owned by Tracking.
- **Primary Key:** `(tenant_id, id)`
- **Multi-Tenant Key:** `tenant_id` (indexed)

The table stores Tenant-local episode identity, positive review version, APPROVED/REJECTED outcome,
reviewer UUID, bounded note, reviewed/created timestamps, with a Tenant-local episode foreign key
and `(tenant_id, episode_id, review_version)` uniqueness.

V88 also adds Routing-owned immutable geometry tables documented by the Transportation/Routing
context. PostgreSQL acceptance is 7/7, affected regression 224/224, architecture 58/58 and full
Maven 1,718/1,718. No detector, workflow, event, Notification, audit publisher, scheduled job, API
or frontend behavior is activated.

Next: `US-52-MONITOR-ROUTE-DEVIATIONS-CS03-EVALUATION-EPISODES-001`.

## US-52 CS03 Evaluation and Episode Lifecycle

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS03_COMPLETE`; accounting remains 73/87 and Flyway
remains V88. Tracking now exposes `RouteDeviationEvaluationUseCase` as the narrow internal
position-processing boundary. No Kafka consumer, scheduler, REST API, permission, external event,
Notification integration, review command or frontend behavior is activated by CS03.

For each eligible trusted position, Tracking resolves the source-time assignment only through
Trip's published `VehicleTripAssignmentLookup` and retrieves only the exact assigned immutable
revision through Routing's published `PlannedRouteGeometryLookup`. Cross-module reads complete
before the Tracking transaction and expose no foreign persistence. Missing Trip, route, revision,
geometry or rule and invalid, stale, duplicate, out-of-order, untrusted or unsuitable-accuracy
positions remain explicitly non-evaluable rather than being treated as on-route.

The evaluator uses the minimum clamped distance over the complete route polyline. The inclusive
effective boundary is configured tolerance plus eligible accuracy. Two distinct consecutive
outside positions confirm one episode; one eligible inside position resets a candidate or closes
an open episode. Continued deviation advances the same episode and permits severity escalation but
never downgrade. Trip, route or revision change closes the prior episode as `SUPERSEDED` before a
fresh lifecycle begins. The first candidate's identity, source time, coordinates, accuracy,
distance, severity and attribution are preserved verbatim at confirmation.

Ordering is `(sourceTimestamp, positionId)`. A Tenant/Vehicle advisory lock serializes the
Tracking-owned transaction, the existing optimistic lock version protects state writes, and the
V88 partial unique index prevents multiple open episodes. State plus episode mutation is atomic;
failure rolls back the complete transition. PostgreSQL concurrent confirmation converges on one
episode. All lookups and writes are Tenant-qualified, and no provider secret, raw payload, Driver
PII or Customer PII enters deviation evidence.

Verification: focused evaluation 25/25, V88 PostgreSQL 9/9, affected regression 231/231,
architecture/ownership 58/58 and complete clean Maven 1,726/1,726 PASS. Checkstyle, PMD, SpotBugs,
dependency analysis, Compose validation and diff hygiene pass. No migration was added.

Next: `US-52-MONITOR-ROUTE-DEVIATIONS-CS04-APIS-RBAC-AUDIT-001`.

## US-52 CS04 APIs, RBAC and Audit

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS04_COMPLETE`; accounting remains 73/87 and Flyway advances to
V89 solely for the four approved route-deviation permissions. Tracking implements Tenant-scoped rule and
state list/detail, rule create/update/activate/disable/retire, bounded episode list/detail and immutable review
history, plus approve/reject/correct-review under `/api/v1/tracking/route-deviations`.

Rule/state pages default to 20 and cap at 100. Episode searches require at most 31 days, default to 100, cap
at 500 and use descending `(start_source_timestamp,id)` keyset pagination; review history caps at 100. DTOs
exclude exact coordinates, provider/device facts, credentials, raw telemetry and PII. Tenant, actor and
correlation facts are server-derived. The four V89 permissions are independently enforced at literal HTTP and
secured use-case boundaries. Foreign-Tenant identifiers use safe absence behavior.

Commands use idempotency claims and optimistic expected versions. Only HIGH episodes are reviewable.
Approve/reject and a different authorized actor's correction append immutable review rows and update the
episode review projection atomically; the current reviewer cannot reverse their own result. Every successful
rule/review action writes one minimized typed `tracking_audit_event` record in the same transaction. Failed,
denied or rolled-back commands create no success audit, and retries create no duplicate transition/audit.

V89 inserts exactly `ROUTE_DEVIATION_VIEW`, `ROUTE_DEVIATION_MANAGE`, `ROUTE_DEVIATION_EVENT_VIEW` and
`ROUTE_DEVIATION_APPROVE`, conditionally granting them only to existing `ADMIN` and `LOCAL_MVP_ADMIN` roles.
Clean V1→V89 and V88→V89 pass. Focused CS04 evidence is 26/26, architecture/ownership 58/58 and complete
Maven 1,738/1,738 PASS. CS04 adds no event, Notification, Operations integration, frontend or physical-device
acceptance. Next: `US-52-MONITOR-ROUTE-DEVIATIONS-CS05-NOTIFICATION-INTEGRATION-001`.

## US-52 CS05 durable Notification integration

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS05_COMPLETE`; accounting remains 73/87 and Flyway advances to
V90. Tracking now publishes minimized `VehicleRouteDeviationDetectedV1` and
`VehicleRouteDeviationEscalatedV1` envelopes through the P1-01 transactional outbox. Notification resolves
active same-Tenant Dispatcher members, delivers IN_APP only and deduplicates replay with its existing stable
execution identity. Notification failure is isolated from committed Tracking evidence.

Detection occurs once at episode confirmation. WARNING maps to Notification WARNING and domain HIGH maps to
the platform's existing CRITICAL value. Direct HIGH confirmation produces detection only; the first later
WARNING-to-HIGH transition produces one `DISTANCE_HIGH` escalation, and the first rejected HIGH review
produces one `REVIEW_REJECTED` escalation. Continued HIGH observations, approval, closure, normal progress,
non-evaluable input and review correction are silent. Rendered content uses deterministic whole-metre
rounding and UTC ISO source time, and excludes driver identity, coordinates, geometry, review notes, raw
telemetry, provider/device details, credentials/signatures and Driver/Customer PII.

V90 is catalogue-only: it seeds exactly `TRACKING_ROUTE_DEVIATION_DETECTED_V1` and
`TRACKING_ROUTE_DEVIATION_ESCALATED_V1`, with Tenant ROLE/DISPATCHER rules and required policy associations.
It creates no role, permission, severity value or schema object. PostgreSQL outbox-to-Notification,
same-Tenant/Tenant-B and replay evidence passes; architecture is 59/59 and complete Maven is 1,745/1,745.
Next: `US-52-MONITOR-ROUTE-DEVIATIONS-CS06-FRONTEND-001`.

## US-52 CS06 frontend and V91 durable evaluation dispatch

US-52 is `IMPLEMENTATION_IN_PROGRESS / CS06_COMPLETE`; accounting remains 73/87 and Flyway advances
to V91. The permission-aware rule/state/episode/review frontend and real Chromium journey are complete.
Every retained hybrid history fact atomically creates exactly one GEOFENCE, SPEED and ROUTE_DEVIATION
dispatch. Kafka is acknowledged only after history and all intents commit. Evaluator failure retains
accepted history and durable retry state; Redis remains an independent consumer. The worker reads the
exact Tenant-qualified immutable fact and reuses the three existing application evaluators. IDLE is
excluded because US-51 still lacks authoritative engine-state capability.

V91 preserves legacy geofence/speed jobs and accepts geofence transition evidence from either same-Tenant
legacy position or Timescale history. Claims use bounded `FOR UPDATE SKIP LOCKED`, leases and deterministic
Tenant/Vehicle/source-time order. Replay is constrained by Tenant/history/evaluator and
Tenant/dedupe/evaluator uniqueness. CS07 is now complete; the current queue is recorded below.

## US-52 V92 maintenance-window index hardening and CS07 closure

V92 is a normal transactional maintenance-window migration. It creates only
`idx_tracking_route_deviation_episode_keyset` on
`(tenant_id, vehicle_id, start_source_timestamp DESC, id DESC)` and the Trip-owned
`idx_trip_tenant_vehicle_source_assignment` on
`(tenant_id, vehicle_id, actual_start_time DESC, id DESC)`, including
`actual_end_time,status,driver_id,route_id,route_version`, for non-null starts whose status is not
`CANCELLED` or `REJECTED`. Both indexes lead with Tenant scope, are ready/valid, and match their
production queries. The Trip index changes only physical access; Trip retains ownership and Tracking
continues to use its published source-time lookup contract.

The migration intentionally uses ordinary `CREATE INDEX`, not `CONCURRENTLY`, and must run only after
draining writers for `tracking_route_deviation_episode` and `trip`, checking long transactions and disk
headroom, and retaining Kafka backlog. Operators validate Flyway V92 and both indexes before restoring
traffic. Transaction rollback removes incomplete builds before commit; after successful deployment V92
is immutable, and removal would require a separately reviewed forward migration. Local plan timings are
acceptance evidence, not production guarantees.

The route-deviation keyset plan changed from a 10,000-row sequential scan plus top-N sort to an
index-only scan without a sort. The Trip lookup changed from examining 10,002 Vehicle-index entries and
discarding 9,940 to an ordered index-only lookup. Deterministic PostgreSQL concurrency, durable dispatch,
lease recovery, idempotency and Tenant isolation pass with no deadlocks.

Retained Chromium initially timed out because the host-run E2E application did not enable hybrid storage
and Compose exposed Kafka only inside its Docker network. The governed E2E environment now retains
internal `kafka:9092`, exposes host-only acceptance access at `localhost:9094`, enables hybrid storage and
topic management, and gives the US-49 telemetry scenario its own unique active geofence. No timeout was
increased and no detector state was pre-seeded. US-49 scenario 4 and US-50 scenario 1 each passed three
independent repetitions; final US-49/US-50/US-52 Chromium continuity passed 26/26. Final evidence is
focused 488/488, architecture 59/59, Maven 1,754/1,754, and Vitest 319/319. The consolidated
prerequisite and CS01–CS07 technical closure is `PASS`; US-52 is `TECHNICALLY_COMPLETE /
ACCEPTANCE_PENDING`. Its independent physical coordinate/accuracy, safe-route and operational journey
must not inherit evidence from US-48 or US-50. Next:
`US-52-MONITOR-ROUTE-DEVIATIONS-FINAL-ACCEPTANCE-001`.

## US-52 independent final-acceptance hold

The final-acceptance prerequisite gate stopped before field execution. No physical supported device,
genuine provider-origin telemetry, authenticated provider channel/device, safe controlled route, real
same-Tenant Dispatcher delivery witness, complete acceptance-role actor set or authorized operator sign-off
was available or verifiable. Automated and simulated evidence, and evidence owned by US-48 or US-50, was
not inherited. No production defect was observed because no physical session ran.

US-52 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; Flyway remains V92 and accounting
remains 73/87. Rerun `US-52-MONITOR-ROUTE-DEVIATIONS-FINAL-ACCEPTANCE-001` only when all external facts are
available together. Approved Wave C sequencing permits core US-53 product decisions to proceed; optional
route-deviation overlays must consume accepted producer evidence and cannot imply US-52 acceptance. US-53
work follows the frozen controlled change-set decomposition without inheriting producer acceptance.

## US-53 Replay Journeys frozen product decisions

US-53 is `TECHNICALLY_COMPLETE / IMPLEMENTATION_COMPLETE_ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; its required Trip published-query prerequisite is
complete, accounting remains 73/87 and Flyway
is V93. Tracking-owned `tracking_position_history` is the sole movement source. Replay is Tenant-first,
source-time ordered by `(source_timestamp,id)`, limited to seven days and 20,000 browser-session points, and
uses no Redis or legacy-history reconstruction. Phase 1 includes Vehicle/Trip replay, exact route-revision
context, deterministic stop analysis, client-side playback and eligible incident overlays. Export and
engine-on comparison are excluded.

A stop requires trusted points with known accuracy no worse than 100 m, a known speed no greater than
3 km/h or spatial-only evidence when speed is absent, five minutes of visible dwell, a 50 m centroid radius
and no internal source-time gap over two minutes. Gaps and range truncation remain explicit. US-49 overlays
are accepted; US-50 and US-52 overlays are opt-in and labelled technical/field-acceptance-pending; US-51 is
unavailable and engine state is never inferred.

The API uses privacy-preserving read-only POST bodies under `/api/v1/tracking/journey-replays` for
points, stops and incidents. V93 seeds only `JOURNEY_REPLAY_VIEW` and
`JOURNEY_REPLAY_INCIDENT_VIEW`; no replay table/projection, retention change or extra index is authorized.
CS01 adds immutable Tenant-explicit Vehicle/Trip selection, requested/effective ranges, bounded pagination,
`sourceTimestamp ASC, historyId ASC` ordering, Tenant/query-bound opaque-cursor state, explicit coverage/gaps,
stop-analysis inputs/results and producer-labelled privacy-safe overlays. Narrow Tracking ports expose read-only
history, stop, overlay, Trip-attribution, immutable route-revision and cursor boundaries without another
module's persistence or framework types. CS02 now reads points directly from the Timescale history using a
Tenant/Vehicle half-open source-time query, stable `(source_timestamp,id)` keyset, fixed receipt snapshot,
explicit 180-day retention coverage, one bounded predecessor query and an HMAC-SHA256 Tenant/query-bound
cursor. Trip replay scope and one bounded range lookup enrich points without N+1 queries; Routing supplies only
exact immutable route/revision context with per-request caching. CS06 queries same-Tenant geofence transitions,
speed episodes and route-deviation episodes/reviews through Tracking-owned JDBC adapters, preserves the frozen
producer acceptance labels, and returns chronological limit-plus-one pages using the existing authenticated
Tenant/query-bound cursor. CS04 exposes all three endpoints with server-derived Tenant/actor context,
safe-absence behavior, `no-store`/`no-referrer` responses, secured-use-case enforcement and minimized
initial-query audit evidence. Incident queries require both replay permissions. CS05 provides permission-gated
`Tracking → Journey Replay` navigation, Vehicle/Trip and seven-day
range validation, source-time map/timeline playback at the five frozen speeds, accessible seek/stop evidence,
explicit quality/gap/retention warnings, responsive phone/tablet behavior and explicit coordinate expansion.
Cursors remain in query memory and never enter URLs or persistent browser storage. Trip provides one
replay-scope lookup and one half-open, seven-day, 2,000-interval bounded assignment
range query using `LIMIT 2001`; overflow is a stable failure with no partial or N+1 fallback. CS06 adds
permission-aware geofence-default and speed/route-deviation opt-in controls, independent incident retry, and
persistent technical-evidence warnings without upgrading physical producer acceptance.

CS03 streams those pages under one immutable snapshot and derives stops without persistence. It uses trusted
points, known accuracy at most 100 metres, known speed at most 3 km/h or explicit missing-speed spatial
evidence, five-minute dwell, two-minute maximum gaps, and a 50-metre all-point radius. Haversine distance uses
6,371,008.8 metres and centroids use spherical vectors weighted by inverse squared accuracy with a one-metre
floor. Adjacent candidates merge only after combined-radius revalidation. Boundary evidence preserves
truncation, SHA-256 identities bind Tenant/Vehicle/snapshot/evidence bounds, and dedicated authenticated stop
cursors paginate by `(startSourceTimestamp,stopId)`. The 20,001st point and non-advancing cursors fail without
partial output. CS07 adds privacy-safe operation/coverage/rejection/size/latency/overlay metrics,
process-local fixed-window admission of 30 requests per actor/minute and 120 per Tenant/minute,
standard HTTP 429 with `Retry-After: 60`, deterministic producer-query limits and coordinated backend/frontend
rollback flags. The 20-session, 40,000-point PostgreSQL acceptance workload passed with initial p95 158 ms
and continuation p95 119 ms; these are environment evidence, not production SLO guarantees. Field acceptance
recorded PASS 0 and FAIL 0 because no field phase started; all remaining mandatory cases are
`BLOCKED_EXTERNAL_PREREQUISITE`. Simulation was not substituted and technical closure remains valid. The
deferred task is `US-53-REPLAY-JOURNEYS-FINAL-ACCEPTANCE-001`, which resumes only with genuine retained
provider/device journey evidence and operator sign-off. The active queue is
`US-54-VIEW-TRACKING-DASHBOARD-PRODUCT-DECISIONS-001`.

## US-54 View Tracking Dashboard frozen product decisions

US-54 is `IMPLEMENTATION_IN_PROGRESS / CS02_COMPLETE`; accounting remains 73/87 and Flyway remains
V93. The Phase 1 dashboard is a read-only, same-Tenant operational summary for authorized
Dispatchers and administrators. It consumes bounded Tracking live state and Tracking-owned US-49/50/52
evidence, one published bulk Trip context query and Notification's own unread-count API. It contains no
detector, producer mutation, cross-module persistence access, persisted dashboard projection or new event.

The frozen surface is `POST /api/v1/tracking/dashboard/query`, using a body so selectors do not enter URL
history. It returns at most 100 Vehicles/markers, 100 server-binned `0.01° × 0.01°` LIVE/RECENT density
cells and 50 incidents from the previous 24 hours, with an authenticated Tenant/filter-bound cursor.
Existing LIVE/RECENT/STALE and CONNECTED/DEGRADED/OFFLINE thresholds remain authoritative. Observed motion
is `MOVING` only for known speed above 3 km/h, `STATIONARY` for known speed at or below 3 km/h, and otherwise
`UNKNOWN`; it is never engine/idle evidence. US-51 idle remains unavailable.

V94 is reserved only for `TRACKING_DASHBOARD_VIEW`, granted idempotently to existing `ADMIN`,
`LOCAL_MVP_ADMIN` and `DISPATCHER` roles. Coordinates/heat cells additionally require `TRACKING_VIEW`;
producer incidents require their existing event-view permissions; Journey Replay navigation requires
`JOURNEY_REPLAY_VIEW`. Responses are no-store/no-referrer, optional unauthorized sections are omitted, and
Driver/Customer PII, provider/device facts, credentials, raw telemetry and review notes are prohibited.

Frontend polling is every 15 seconds only while visible/online, with 30/60-second transient-failure backoff,
manual recovery and table-first map-failure behavior. Admission is 10 requests per actor/minute and 40 per
Tenant/minute per instance. The coordinated backend/frontend feature flags provide rollback. Producer labels
remain `FIELD_ACCEPTANCE_PENDING` for US-48/52/53, `FIELD_FIDELITY_PENDING` for US-50, `ACCEPTED` for US-49
and `UNAVAILABLE` for US-51; US-54 never upgrades them.

CS01 adds framework-neutral domain and port contracts only. `TrackingDashboardModels` defines the
Tenant-scoped request, disclosure, live-state, Trip-context, producer-labelled incident, summary,
heat-cell and page vocabulary. `TrackingDashboardPolicy` enforces the 100-Vehicle/100-row bounds,
derives motion only from trusted observed speed and produces deterministic 0.01-degree density cells
only from eligible trusted LIVE/RECENT positions. `TrackingDashboardQueryUseCase` is the inbound port;
the narrow outbound ports are `TrackingDashboardLiveStatePort`, `TrackingDashboardIncidentPort`,
`TrackingDashboardTripContextPort` and `TrackingDashboardCursorPort`. CS01 adds no adapter, API,
permission, migration, persistence query, event or frontend behavior.

CS02 implements `TrackingDashboardQueryService`, bounded Tracking-owned incident queries and the hybrid
live-state adapter. Redis supplies newest live candidates; one Tenant-scoped PostgreSQL batch query supplies
authoritative latest-trusted enrichment and the explicit `DEGRADED` fallback. Untrusted latest-received
evidence cannot replace trusted map truth. Incident access is limited to the prior 24 hours, 20 rows per
authorized producer and 50 total. Tracking consumes the published `TripDashboardQuery` only through
`TripTrackingDashboardAdapter`; the contract accepts at most 100 Vehicle IDs and returns only Trip ID,
lifecycle and route identity/version in one Tenant-qualified bulk operation. No Driver identity, foreign persistence access,
public API, permission, migration, event or frontend behavior is introduced. The exact active queue is
`US-54-VIEW-TRACKING-DASHBOARD-CS03-V94-API-RBAC-AUDIT-001`.

## US-54 Tracking Dashboard API, RBAC and Audit (CS03 complete)

US-54 is `IMPLEMENTATION_IN_PROGRESS / CS03_COMPLETE`; accounting remains 73/87 and Flyway head is V94.
V94 seeds only `TRACKING_DASHBOARD_VIEW` and its idempotent grants to existing `ADMIN`, `LOCAL_MVP_ADMIN`
and `DISPATCHER` roles. It creates no role, table, index or unrelated permission.

`POST /api/v1/tracking/dashboard/query` now exposes the bounded CS02 query behind both literal-path and
secured-use-case enforcement. Broad Tracking permissions do not imply dashboard access. Coordinates/heat
cells and each incident/replay section remain conjunctively protected by their existing permissions and are
omitted without count leakage. The purpose-separated HMAC cursor expires after five minutes and is bound to
Tenant, filters, page size and snapshot. Responses are no-store/no-referrer; invalid filters or cursors return
400, admission rejection returns 429 with a 60-second retry instruction, total live-source unavailability
returns 503, and truthful partial degradation remains 200.

Initial queries, authenticated denials and rate-limit rejections retain only privacy-safe audit categories,
bounded counts/statuses and correlation context. Cursor continuations are not audited per page. Audit and
metrics prohibit selectors, domain IDs, coordinates, cursors, raw telemetry, provider/device details,
credentials, signatures and personal data. `app.tracking.dashboard.enabled` controls the endpoint without
altering producer evidence. CS03 adds no frontend behavior; the exact queue is
`US-54-VIEW-TRACKING-DASHBOARD-CS04-FRONTEND-001`.

## US-54 Tracking Dashboard frontend (CS04 complete)

US-54 is `IMPLEMENTATION_IN_PROGRESS / CS04_COMPLETE`; accounting remains 73/87 and Flyway head remains
V94. Authorized operators can open `/tracking/dashboard` when the frontend feature flag is enabled and
their authenticated permission set includes `TRACKING_DASHBOARD_VIEW`. Navigation visibility is only a UX
guard; backend authorization remains authoritative.

The screen submits filters exclusively in the secured POST body, keeps selectors out of URLs and browser
storage, polls every 15 seconds only while visible and online, and uses bounded 30/60-second retry delays.
Its responsive operational table remains authoritative when the internal SVG map or heat-density layer is
unavailable. Live/recent/stale/degraded/offline truth, observed motion, producer acceptance labels, incident
availability and replay availability remain explicit. Notification unread counts are queried only when the
same user also has `NOTIFICATION_VIEW`.

No provider credentials, signatures, device details, Driver PII or Customer PII are stored or rendered.
Coordinate display continues to depend on backend conjunctive disclosure. Real isolated PostgreSQL-backed
Chromium acceptance passed 6/6 scenarios. CS04 does not inherit or replace any physical telemetry/operator
acceptance gate. Next queue:
`US-54-VIEW-TRACKING-DASHBOARD-CS05-POSTGRES-REDIS-PERFORMANCE-OPERATIONS-001`.

## US-54 Tracking Dashboard PostgreSQL/Redis performance and operations (CS05 complete)

US-54 is `IMPLEMENTATION_IN_PROGRESS / CS05_COMPLETE`; accounting remains 73/87 and Flyway head is V95.
V95 adds only `idx_tracking_speed_episode_dashboard_recent`, a B-tree over
`tracking_speed_episode (tenant_id, confirmation_source_timestamp DESC, id DESC)`. The index matches the
Tenant-scoped 24-hour recent-speed query and deterministic ordering. It adds no table, column, constraint,
permission, event, API or frontend contract.

Representative PostgreSQL acceptance used 12,000 speed episodes for plan comparison and proved identical
results with an index scan replacing the full scan and top-N sort. The dashboard load gate used 100 Vehicles,
200 incidents and 20 coordinated sessions. Redis failure returns truthful `DEGRADED` source status with the
governed PostgreSQL fallback and returns to `AVAILABLE` after recovery. Operational metrics use only bounded
outcome, source-status, included-category and fixed degradation-reason tags; Tenant, actor, Vehicle, cursor,
coordinate and personal values are prohibited as tags.

V95 uses ordinary transactional `CREATE INDEX` and requires a maintenance window that drains affected speed
episode writers while retaining Kafka backlog, checks locks and disk headroom, validates Flyway plus index
readiness/validity, then restores traffic. A pre-commit failure rolls back atomically; after success V95 is
immutable and removal requires a reviewed forward migration. Exact next queue:
`US-54-VIEW-TRACKING-DASHBOARD-TECHNICAL-CLOSURE-001`.

## US-54 Tracking Dashboard technical closure

US-54 is `TECHNICALLY_COMPLETE / ACCEPTANCE_PENDING`; accounting remains 73/87 and Flyway remains V95.
An independent rerun from the committed CS05 baseline passed focused PostgreSQL/Redis/dashboard tests 18/18,
the complete Maven suite 1,838/1,838, architecture 59/59, Vitest 330/330 and real Chromium 6/6, together
with TypeScript, production build, scoped lint and static-analysis gates. All accepted database evidence used
`transport_logistics_acceptance`.

Technical closure does not supply or inherit genuine provider/device telemetry, an authorized field operator
session, operator confirmation, privacy review or field sign-off. Exact next queue:
`US-54-VIEW-TRACKING-DASHBOARD-FINAL-ACCEPTANCE-001`.

## US-54 final-acceptance external hold

US-54 is `TECHNICALLY_COMPLETE / IMPLEMENTATION_COMPLETE_ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`.
Technical closure remains valid at V95, but final acceptance requires genuine provider/device telemetry,
same-Tenant field context, live-to-stale/offline recovery observation, privacy review and authorized operator
sign-off. No synthetic Kafka record, seeded database fact, Testcontainer, Playwright fixture, mock provider or
simulated track substitutes for those facts. No physical case was executed, so none is PASS or FAIL.

Accounting remains 73/87. Deferred acceptance is
`US-54-VIEW-TRACKING-DASHBOARD-FINAL-ACCEPTANCE-001`; active queue is
`US-55-HANDLE-GPS-EDGE-CASES-PRODUCT-DECISIONS-001`.

## US-55 Handle GPS Edge Cases frozen product decisions

US-55 is `IMPLEMENTATION_IN_PROGRESS / CS03_COMPLETE`; accounting remains 73/87 and Flyway is
V96. Tracking owns reliability classification and immutable GPS-exception evidence. Exact Phase 1 boundaries are
60 seconds LIVE, five minutes STALE/OFFLINE, 24 hours LATE, 120 seconds future tolerance, good accuracy through
100 metres, low accuracy through 1,000 metres, and an impossible jump of at least 2 km within 10 minutes with
implied speed above 250 km/h. Recovery from suspect movement requires two consecutive eligible points.

Invalid coordinates never become `(0,0)`; actual `(0,0)` is retained only as untrusted Null Island evidence.
Missing tamper/battery capabilities remain UNKNOWN and are never inferred as healthy or faulty. History remains
append-only, Redis cannot regress or promote ineligible facts, and geofence/speed/route-deviation consume only
trusted, in-order, good-accuracy evidence at most five minutes old. Journey Replay may show retained uncertain
evidence with warnings; Dashboard map/motion uses latest trusted only.

Tracking owns a bounded exception workflow; minimized HIGH facts integrate with accepted US-78 and IN_APP
Notification through P1-01. Permissions are `GPS_EXCEPTION_VIEW` and `GPS_EXCEPTION_REVIEW`. V99 implements
those permissions and V100 implements durable acknowledgement replay without changing detector or notification
semantics.

CS01 implements framework-neutral `GpsCoordinate`, reliability observation/context/assessment types,
`GpsReliabilityPolicy` and `GpsExceptionEpisode`. It freezes coordinate/accuracy/time/order/connectivity,
impossible-movement, battery and two-point recovery behavior in pure Tracking domain code. Repeated evidence
updates one episode, severity cannot downgrade, resolved evidence is immutable and binding/processing failures
require confirmed correction. Tenant identity is mandatory on observations, episodes, inbound evaluation/review
operations and outbound repository operations. CS01 adds no adapter, persistence, API, permission, event, Kafka,
Redis or frontend behavior. Focused tests pass 12/12, architecture passes 71/71 and the isolated complete backend
passes 1,850/1,850.

CS02 keeps the V1 topic and payload immutable, adds canonical `tracking.telemetry.ingested.v2` and
its `.v2.dlt`, retains V1/V2 deserializers for Timescale and Redis, and moves normalized ingress to
exactly one V2 publication. Optional tamper, battery-level, battery-voltage, external-power and
charging observations preserve absence as not reported. Flespi and Traccar use approved explicit
mappings; Generic ingress rejects arbitrary signal claims. A Tenant-qualified source-time
`TelemetryCapabilityLookupPort` defines `SUPPORTED`, `UNSUPPORTED` and conservative `UNKNOWN`;
capability persistence is not part of CS02. Cross-version uniqueness uses the existing canonical
dedupe identity and never includes event version. Focused CS02/architecture passes 76/76, real Kafka
passes 2/2, Timescale passes 11/11, Redis passes 4/4 and the clean backend passes 1,861/1,861.
CS03 reuses the V75 provider registry/opaque credential reference, V76 device/provider binding and polling
watermark, V73 source-time Vehicle assignment, and V86/V87 canonical dedupe identity. V96 adds nullable V2
history evidence, database-enforced append-only history and the effective-dated capability registry described
above. Clean V1→V96, V95→V96, transactional failure/retry, compressed Timescale upgrade, Tenant isolation,
architecture 59/59 and complete Maven 1,867/1,867 pass. No API, permission, event or frontend contract changes.
Exact next queue:
`US-55-HANDLE-GPS-EDGE-CASES-CS04-EVALUATION-REDIS-DETECTOR-GUARDS-001`.

## US-55 CS04 evaluation, authoritative episodes and detector guards

US-55 is `IMPLEMENTATION_IN_PROGRESS / CS04_COMPLETE`; accounting remains 73/87 and Flyway head is
V97. V97 creates exactly two Tracking-owned tables. `tracking_gps_exception_episode` is the authoritative
Tenant/device/type lifecycle record with optimistic versioning and one active episode per logical key.
`tracking_gps_exception_evidence` stores deterministic, minimized assessment evidence and rejects update or
delete at the database boundary. Composite Tenant keys prevent cross-Tenant device, episode and evidence
relationships. Neither table stores coordinates, raw payload, credentials, signatures or PII.

The evaluator reconstructs decisions from PostgreSQL plus immutable telemetry and effective-dated capability
history. Unsupported optional V2 signals remain UNKNOWN; two eligible recovery observations are required;
stale or reversed observations cannot regress durable state. Signal loss can be recorded without fabricating a
telemetry-history row. Episode transition and evidence append share one transaction.

The Kafka live projector evaluates reliability before the existing atomic Tenant-qualified Redis
compare-and-apply. Only trusted, in-order eligible observations advance live state. Geofence, speed and route
deviation dispatch additionally require an accurate, non-null-island, recent observation. Redis remains a
disposable projection and is never episode authority.

Verification passed: V97 migration/rollback 3/3, focused evaluator/projector/dispatch 16/16, architecture
59/59 and complete Maven 1,872/1,872. Checkstyle, PMD, SpotBugs, dependency analysis, Compose validation and
diff hygiene passed. CS04 changes no public API, permission, event contract, notification or frontend.

Exact next queue:
`US-55-HANDLE-GPS-EDGE-CASES-CS05-OPERATIONS-NOTIFICATION-INTEGRATION-001`.

## US-55 CS05 Operations and Notification integration

CS05 is `COMPLETE`; US-55 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and Flyway
head is V98. Tracking publishes a minimized `TrackingGpsExceptionOpenedV1` exactly once when an episode
opens at WARNING or HIGH and a minimized `TrackingGpsExceptionHighV1` exactly once when an episode is
first HIGH. WARNING-to-HIGH produces no second Notification, but does produce the one HIGH Operations fact.
Repeated evidence, recovery and escalation remain silent.

Notification resolves active same-Tenant Dispatcher recipients and uses only IN_APP template
`TRACKING_GPS_EXCEPTION_ALERT_V1`. Domain WARNING maps to stored Notification WARNING. Domain HIGH maps
to the platform's existing stored CRITICAL value while the approved user-facing severity variable remains
HIGH. Operations receives HIGH-only facts with summary code `TRACKING_GPS_EXCEPTION_HIGH`, the exact
canonical source type, and only `episodeId`, `exceptionType`, `severity`, `deviceId`, `vehicleId`, `openedAt`
and `lastObservedAt` metadata. `PROCESSING_FAILURE` maps to `TRACKING_DATA_QUALITY`; no error text, stack,
payload, coordinate, credential or infrastructure detail crosses either boundary.

V98 extends the Operations allowlists and seeds the single template plus same-Tenant Dispatcher rules and
policies only for Tenants that exist when the migration runs. Automatic future-Tenant defaults are explicitly
`DEFERRED_PENDING_GOVERNED_TENANT_CREATION_WORKFLOW`; V98 creates no trigger or speculative provisioning
workflow. Delivery is at least once through P1-01 and consumers retain Tenant-qualified idempotency.

## US-55 CS06 API, RBAC and audit

CS06 is `COMPLETE`; US-55 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and Flyway head is
V100. The bounded no-store `/api/v1/tracking/gps-exceptions` API provides same-Tenant list, detail, immutable
evidence and acknowledgement. VIEW and REVIEW are independent permissions enforced at literal HTTP and secured
use-case boundaries. List range is required, UTC, `[from,to)` and at most seven days; page size is 1–500 and
Tenant/filter-bound HMAC cursors preserve deterministic keyset order.

V100 records completed successful acknowledgement commands only. A Tenant-qualified PostgreSQL transaction
advisory lock serializes the key before the row-locked episode mutation, response-snapshot insert and minimized
`GPS_EXCEPTION_ACKNOWLEDGED` audit commit together. Retry reauthorizes the actor and returns the original
allow-listed snapshot; reason is never returned or audited. There is no expiry until a separate retention policy.

Verification includes concurrent retries, conflict/replay/lifecycle cases, forced audit rollback, V1/V98/V99
upgrade paths, literal API security, architecture 59/59 and complete backend 1,890/1,890.

## US-55 CS07 operator frontend

CS07 is `COMPLETE`; US-55 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and Flyway head is
V100. The permission-gated `/tracking/gps-exceptions` workflow requires an explicit UTC range of at most seven
days before it queries. It provides bounded filters, opaque cursor paging, same-Tenant episode detail and
immutable evidence, privacy-safe exception labels, and responsive table/card presentation without exposing
coordinates, provider/device secrets, raw telemetry, internal errors or personal data.

`GPS_EXCEPTION_VIEW` and `GPS_EXCEPTION_REVIEW` remain independent: VIEW is required for every read, while
REVIEW adds acknowledgement only. Acknowledgement validates a trimmed 1–500 character reason, sends the
visible version and one cryptographically generated idempotency key, disables duplicate submission and
preserves that exact request across an uncertain retry. Version/lifecycle conflicts refresh authoritative
state without automatic resubmission. Session-qualified query keys, cancellation and cache clearing prevent
cross-session leakage. Real PostgreSQL-backed Chromium acceptance passed 6/6 and the complete frontend and
backend regressions passed without an API, permission, event, schema or dependency change.

Next queue: `US-55-HANDLE-GPS-EDGE-CASES-CS08-POSTGRES-KAFKA-REDIS-CONCURRENCY-PERFORMANCE-001`.

## US-55 CS08 PostgreSQL, Kafka and Redis concurrency/performance

CS08 is `COMPLETE`; US-55 remains `IMPLEMENTATION_IN_PROGRESS`, accounting remains 73/87, and Flyway
head remains V100. Concurrent first evidence is serialized by a Tenant/device-qualified PostgreSQL
transaction advisory lock before the active-episode lookup, closing the absent-row race without changing
the V97 uniqueness contract. Different Tenants retain independent lock identities.

Signed telemetry batches retain per-item normalization, device authority and Tenant validation, then publish
to Kafka as one bounded asynchronous batch and wait for all durable acknowledgements. Kafka delivery remains
at-least-once; dedupe identities keep business effects idempotent. The existing live-state REST contract reads
the legacy PostgreSQL state first and falls back to the governed Redis projection when hybrid ingress has no
legacy live row. No public API, permission, event payload or migration changed.

Isolated acceptance evidence passed PostgreSQL/Kafka/Redis/security 39/39, architecture 59/59 and complete
backend 1,892/1,892. Real Chromium measured 882.2 msg/s sustained, 3,884.2 msg/s burst, latest p95 21.9 ms and
history p95 19.1 ms. These are controlled-environment results, not production SLO certification or physical
device acceptance. Next queue: `US-55-HANDLE-GPS-EDGE-CASES-TECHNICAL-CLOSURE-001`.

## US-55 consolidated technical closure

US-55 is `TECHNICALLY_COMPLETE / ACCEPTANCE_PENDING`; accounting remains 73/87 and Flyway remains
V100. Closure reconciled CS01–CS08 against the frozen decisions, API/event/RBAC registries and both
roadmaps. The hybrid latest query now compares the Tenant-qualified PostgreSQL latest-trusted state
with the eligible Redis live projection and returns the newer source timestamp. An older Redis value
cannot regress durable state, uncertain telemetry cannot be promoted as trusted, and Redis dependency
failure is not rewritten as absence.

Closure verification passes focused US-55 infrastructure/security 62/62, complete Maven 1,895/1,895,
architecture/Modulith 59/59, frontend Vitest 336/336, real PostgreSQL-backed GPS-exception Chromium
6/6 and hybrid performance Chromium 1/1. The controlled acceptance environment measured 576.6
messages/second sustained, 4,693.7 messages/second burst, 16.6 ms latest-query p95 and 19.2 ms
history-query p95; these are technical environment observations, not production capacity guarantees.

Flespi polling and Generic signed ingress have production adapters and controlled fixtures, but neither
has real-provider physical evidence. Traccar has canonical normalization fixtures only; its production
adapter/onboarding remains governed by the US-48 provider roadmap. Future-Tenant Notification catalogue
provisioning remains explicitly deferred. These limitations do not invalidate technical closure and are
not physical acceptance. Next queue: `US-55-HANDLE-GPS-EDGE-CASES-FINAL-ACCEPTANCE-001`.

## US-55 independent final-acceptance hold

Final acceptance is `IMPLEMENTATION_COMPLETE_ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`. No physical GPS device,
real provider account/channel/device identity, genuine provider-origin telemetry or authorized field operator
is available. The nine required field areas—provider identity, real binding, loss/recovery, reassignment,
supported tamper, supported battery/power, provider burst, Dispatcher/Operations handling, and privacy/operator
sign-off—are all `BLOCKED_EXTERNAL_PREREQUISITE`; none passed, failed or qualified as not applicable.

Flespi production polling and Generic signed ingress remain implemented but without genuine field evidence.
Traccar now has a bounded production HTTPS bearer-token polling adapter qualified against the official 6.15.3
OpenAPI contract. Guided Flespi/Traccar onboarding and provider health recovery are implemented; real Traccar
provider/device evidence remains pending and the open physical final-acceptance queue is unchanged. Flyway and
accounting remain V100 and 73/87. The next independent queue is `US-55-IDEMPOTENCY-CLOSURE`.

## Provider-polling canonical Kafka prerequisite

`US-55-PROVIDER-POLLING-CANONICAL-KAFKA-REMEDIATION-001` is complete. All migrated polling providers resolve
server-authoritative same-Tenant Device and source-time Vehicle authority and publish deterministic canonical V2
facts durably through Kafka. Only an all-record acknowledgement advances successful polling watermarks; timeout,
partial acknowledgement and publication failure retain the prior watermark and enter bounded backoff. Retried
publication uses the same event and deduplication identities. Polling adapters do not independently write legacy
positions, Redis or TimescaleDB; established consumers own history, eligible trusted live state and detector
evaluation. Nullable tamper, battery, voltage, external-power and charging observations are preserved.

The approved Traccar boundary targets the officially verified Traccar 6.15.3 OpenAPI contract and its `ApiKey`
HTTP bearer scheme. It uses one-MiB/500-observation bounded time-ranged responses, rejects overflow without
watermark advancement, and never claims latest-only retrieval as complete journey history. Public HTTPS is the
default. Private/self-hosted HTTPS requires a deployment-admin-managed destination-and-port allowlist, connection-
time DNS validation and an explicitly trusted CA where required. Redirect bypass, Basic fallback, disabled TLS
verification, URL credentials and Tenant-controlled allowlist expansion are prohibited. Flyway remains V100.

## Traccar production polling adapter

The `TRACCAR` provider SPI is implemented for the qualified Traccar 6.15.3 OpenAPI contract. It uses opaque
bearer credentials, bounded per-device `GET /api/positions?deviceId&from&to` history retrieval, complete-response
validation and deterministic source ordering. Record/byte overflow rejects the whole poll; the common coordinator
retains the prior watermark and applies bounded backoff. Latest-only retrieval is never represented as journey
history. The normalizer preserves approved V2 optional signals and publishes only through the canonical Kafka
boundary established above.

Public HTTPS/443 is allowed by default. Private/self-hosted HTTPS requires an exact deployment-admin-owned
`host:port` entry in `TRACKING_TRACCAR_PRIVATE_ENDPOINT_ALLOWLIST`; Tenant configuration cannot expand it.
Destination DNS is revalidated before every request. Loopback, link-local, multicast, metadata and unapproved
private/ULA addresses are blocked, redirects are not followed and TLS verification remains mandatory. A governed
private CA is installed through the JVM trust store, never by disabling verification. The provider UI and health
recovery slices described below supersede that historical queue; real-provider acceptance remains open.

## Provider health and recovery

`US-55-HEALTH-RECOVERY` is complete. Flespi and Traccar adapters expose only bounded privacy-safe failure
categories to the polling coordinator. Overflow, authentication, endpoint-policy, provider rejection,
unavailability, transient/rate-limit and canonical Kafka publication failures retain the prior watermark and
enter bounded backoff. Provider payloads, exception text, credentials, endpoints and precise locations are not
persisted or rendered. A successful empty response records reachability and a completed poll but does not update
the provider-message timestamp, manufacture telemetry, resolve GPS exceptions or claim live recovery.

Required Kafka acknowledgements remain the durable progress boundary. Interrupted/partial publication leaves
polling progress unchanged and retries with deterministic canonical identities; database leases and local
single-flight coordination prevent overlapping connection work and recover expired leases after restart. The
Provider Connections UI renders safe recovery guidance while keeping connection verification, successful poll,
downstream telemetry receipt and freshness distinct. Generic signed-HMAC ingress remains unchanged. Controlled
PostgreSQL/Kafka/provider fixtures and Chromium evidence pass, but they are not physical/provider acceptance.
Flyway remains V100, accounting remains 73/87 and future-Tenant Notification provisioning remains deferred.

#### Table: `tracking_gps_exception_episode`

- **Purpose:** Authoritative lifecycle state for a governed GPS reliability exception.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, composite references and Tenant-leading indexes)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY; unique with `tenant_id` | Episode identity |
| `tenant_id` | UUID | NO | - | Composite FK scope | Trusted Tenant |
| `tracking_device_id` | UUID | NO | - | Composite FK → `tracking_device(tenant_id,id)` | Tracking-owned device |
| `vehicle_id` | UUID | YES | NULL | Logical same-Tenant Vehicle reference | Source-time associated Vehicle when known |
| `exception_type` | VARCHAR(32) | NO | - | Governed type CHECK | Frozen GPS exception type |
| `severity` | VARCHAR(8) | NO | - | `WARNING` or `HIGH` | Non-downgrading severity |
| `status` | VARCHAR(16) | NO | - | `OPEN`, `ACKNOWLEDGED`, `RECOVERING`, `RESOLVED` | Lifecycle state |
| `opened_at` | TIMESTAMPTZ | NO | - | `opened_at <= last_observed_at` | First source/assessment time |
| `last_observed_at` | TIMESTAMPTZ | NO | - | Time-order CHECK | Latest accepted lifecycle evidence time |
| `resolved_at` | TIMESTAMPTZ | YES | NULL | Required only for `RESOLVED` | Resolution time |
| `evidence_count` | BIGINT | NO | - | >= 1 | Accepted observation count |
| `consecutive_recovery_points` | INTEGER | NO | 0 | >= 0 | Durable recovery progress |
| `version` | BIGINT | NO | 0 | >= 0 | Optimistic version |
| `acknowledgement_reason` | VARCHAR(500) | YES | NULL | Trimmed length 1..500 when present | Minimized review reason |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | - | Latest mutation time |

The partial unique index `uq_tracking_gps_exception_active` covers `(tenant_id,
tracking_device_id, exception_type)` for `OPEN`, `ACKNOWLEDGED` and `RECOVERING`. Tenant list access uses
`idx_tracking_gps_exception_tenant_list(tenant_id,last_observed_at DESC,id DESC)`.

#### Table: `tracking_gps_exception_evidence`

- **Purpose:** Immutable, minimized evidence supporting one GPS-exception lifecycle assessment.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, composite FK and Tenant-leading index)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Evidence identity |
| `tenant_id` | UUID | NO | - | Composite FK scope | Trusted Tenant |
| `episode_id` | UUID | NO | - | Composite FK → `tracking_gps_exception_episode(tenant_id,id)` | Owning episode |
| `evidence_identity` | CHAR(64) | NO | - | Unique with Tenant and episode | Deterministic SHA-256 retry identity |
| `telemetry_history_id` | UUID | YES | NULL | Paired with source timestamp; logical history reference | Canonical telemetry identity when applicable |
| `telemetry_source_timestamp` | TIMESTAMPTZ | YES | NULL | Both history fields present or absent | Immutable telemetry source time |
| `assessed_at` | TIMESTAMPTZ | NO | - | - | Assessment time |
| `trust` | VARCHAR(12) | NO | - | Governed trust CHECK | Trust classification |
| `ordering_classification` | VARCHAR(20) | NO | - | Governed ordering CHECK | In-order/out-of-order/equal-time classification |
| `reliability_state` | VARCHAR(16) | NO | - | Governed reliability CHECK | Resulting reliability state |
| `quality_codes` | VARCHAR(400) | NO | - | Length 1..400 | Sorted minimized quality facts |

#### Table: `tracking_gps_exception_acknowledgement_command`

- **Purpose:** Durable immutable replay record for successful GPS-exception acknowledgements.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, unique key and composite same-Tenant episode FK)

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Command identity |
| `tenant_id` | UUID | NO | - | Unique with idempotency key; composite FK scope | Trusted Tenant |
| `idempotency_key` | VARCHAR(160) | NO | - | Length 16–160 | Opaque replay key; never logged |
| `request_fingerprint` | CHAR(64) | NO | - | Lowercase hexadecimal SHA-256 CHECK | Canonical Tenant/operation/actor/request fingerprint |
| `actor_id` | UUID | NO | - | Authenticated actor | Original authorized reviewer |
| `episode_id` | UUID | NO | - | Composite FK → episode, `ON DELETE RESTRICT` | Acknowledged episode |
| `expected_version` | BIGINT | NO | - | >= 0 | Original optimistic version |
| `response_snapshot` | JSONB | NO | - | JSON object CHECK | Allow-listed acknowledgement response only |
| `completed_at` | TIMESTAMPTZ | NO | - | - | Successful command completion |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Durable record creation |
| `transition` | VARCHAR(20) | NO | - | Governed transition CHECK | `OPENED`, `OBSERVED`, `RECOVERING` or `RESOLVED` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | - | Evidence creation time |

The index `idx_tracking_gps_exception_evidence_tenant_episode(tenant_id,episode_id,assessed_at DESC,id
DESC)` supports Tenant-qualified episode history. Trigger
`trg_tracking_gps_exception_evidence_immutable` rejects every update or delete.

## Hybrid Telemetry TS02 secure Kafka ingress

TS02 is `COMPLETE`. The existing signed `/api/integration/v1/tracking/positions` boundary validates
HMAC, timestamp freshness, nonce replay, size and active provider binding, then selects the governed
Flespi, Traccar or Generic normalizer. Tenant, Device and source-time Vehicle authority come only
from same-Tenant Tracking persistence. The controller publishes canonical version-1 normalized facts
to `tracking.telemetry.ingested.v1`, keyed `{tenantId}:{vehicleId}`, and returns 202 only after durable
broker acknowledgement. Producer idempotence, `acks=all`, six configurable partitions, LZ4,
20-millisecond linger and 65,536-byte batching are active.

TS02 makes no Redis or TimescaleDB write, adds no table or migration, and leaves Flyway at V86.
Provider credentials, raw signatures and raw payloads never enter Kafka. Focused Tracking regression
passes 243/243, architecture passes 58/58, and the complete Maven suite passes 1,695/1,695.
Accounting remains 73/87. Next: `HYBRID-TELEMETRY-TS03-KAFKA-REDIS-LIVE-PROJECTOR`.

## Hybrid Telemetry TS03 Kafka-to-Redis live projection

TS03 is `COMPLETE`. Tracking consumes `TRACKING_TELEMETRY_INGESTED_V1` through consumer group
`tracking-live-projector-v1`, validates the Kafka key, required headers and payload authority, and
manually acknowledges only after the atomic Redis projection succeeds. Poison records follow
bounded recovery to `tracking.telemetry.ingested.v1.dlt`. Redis dependency failures are retried and
then propagated without committing the source offset.

Live state is stored at `tracking:live:{tenantId}:{vehicleId}` with an exact sliding 24-hour TTL.
The Tenant-scoped sorted-set index `tracking:live-index:{tenantId}` is expiry-scored, lazily pruned,
capped at 10,000 Vehicles and queried with a maximum result size of 500; keyspace scans are not
used. A single Lua operation prevents stale records from regressing state, resolves equal source
timestamps by lexicographically greatest immutable event UUID, refreshes TTL for exact replays and
keeps the live value and index consistent in the same Redis hash slot.

TS03 adds no REST API, frontend surface, database table or Flyway migration and performs no
TimescaleDB write. Focused real Kafka/Redis, DLT, Tracking regression, architecture and the complete
Maven suite pass; the final suite reports 1,702 tests with zero failures, errors or skips. Flyway
remains V86 and accounting remains 73/87. Next:
`HYBRID-TELEMETRY-TS04-V87-TIMESCALE-CONSUMER-AND-POLICIES`.

## Hybrid Telemetry TS04 Kafka-to-Timescale history persistence

TS04 is `COMPLETE`. Tracking consumes the canonical V1 telemetry stream through exact group
`tracking-telemetry-persister-group` in configurable batches up to 500. Key, headers, payload,
identity and normalized facts must agree. Manual acknowledgement occurs only after the atomic
Tracking database batch commits; a database failure rolls back the batch and leaves the source
offset uncommitted. Malformed/authority-mismatched records use the bounded access-controlled DLT.

History is append-only and preserves valid late/out-of-order facts independently of Redis live
projection. V87 adds event-version validation, Tenant-scoped database idempotency, seven-day chunks,
compression after seven days segmented by Tenant/Vehicle, and 180-day raw retention. Static
reduction uses exact decimal coordinates and only suppresses a consecutive zero-speed point with
unchanged non-null engine, quality, trust, accuracy and meter facts; uncertainty is retained.

Real Kafka/Timescale/DLT focused acceptance passes 11/11, affected Tracking regression passes
247/247, architecture passes 58/58, and full Maven passes 1,711/1,711. No REST API or frontend was
added. Flyway is V87; accounting remains 73/87. Next:
`US-52-MONITOR-ROUTE-DEVIATIONS-CS02-V88-PERSISTENCE-001`.

## US-55 idempotency closure

`US-55-IDEMPOTENCY-CLOSURE` is `COMPLETE` at Flyway V100. Provider polling publishes
tenant-qualified deterministic canonical identities and advances a device watermark only after all
required Kafka acknowledgements. A crash before cursor persistence therefore replays safely through
the same PostgreSQL history identity. V1 and V2 converge on the same tenant-qualified business
identity; distinct provider messages remain distinct.

Timescale history, evaluator dispatch, GPS-exception evidence, acknowledgement commands, durable
Notification/Operations events and audit records retain their existing PostgreSQL uniqueness and
transaction boundaries. Duplicate GPS recovery evidence cannot increment recovery counters twice.
Redis remains a disposable ordered projection and is not an idempotency authority. This is
at-least-once delivery with idempotent logical effects, not exactly-once transport.

Focused verification passed 31/31 and isolated PostgreSQL/Timescale/Kafka/Redis verification passed
34/34. No production code, migration, API, permission, dependency or event contract changed.
Physical/provider acceptance remains blocked externally, future-Tenant Notification provisioning
remains deferred, accounting remains 73/87, and the next independent queue is
`US-55-TECHNICAL-CLOSURE`.

## US-55 post-extension technical closure

`US-55-TECHNICAL-CLOSURE` is `COMPLETE` at Flyway V100. It is a consolidation gate over the
previously completed CS01-CS08 GPS-edge technical closure, canonical provider polling through
Kafka, bounded Traccar 6.15.3 polling, guided Flespi/Traccar onboarding, provider health recovery
and logical idempotency. It adds no production behavior, schema, API, permission, dependency or
event contract.

The integrated path preserves Tracking ownership: provider polling resolves Tenant-qualified
effective device bindings and publishes canonical V2 telemetry; Tracking consumers own durable
Timescale history, detector evaluation and eligible ordered Redis projection; GPS-exception state
owns Notification/Operations facts; protected APIs and UI expose minimized operational state.
Reachability, polling/publication, downstream processing, telemetry receipt/freshness and genuine
physical acceptance remain distinct.

Flespi, Traccar and Generic signed-HMAC technical paths have controlled evidence, but no fixture,
local browser journey or connection check is credited as genuine provider/device acceptance.
`US-55-HANDLE-GPS-EDGE-CASES-FINAL-ACCEPTANCE-001` remains externally blocked with nine field
requirements. Future-Tenant Notification provisioning remains deferred, accounting remains 73/87,
and no independent implementation queue is selected after this closure.

## US-51 idle-monitoring prerequisite decisions selected

`US-51-MONITOR-IDLE-TIME-PREREQUISITE-AND-PRODUCT-DECISIONS-001` is a newly created, explicitly
selected governance task. Its consolidated decisions are implementation-ready for review but
remain `PROPOSED`; no production code, migration, permission, API or event contract is authorized.
Software implementation and controlled technical fixtures no longer depend on physical hardware.
Production `ENGINE_RUNNING` activation and physical acceptance remain separate mandatory gates.

Current canonical V1/V2 `engineState` is populated from ignition semantics by Traccar/Generic and
is always UNKNOWN from Flespi. Provider-level field support is not device-level truth, and ignition
is not engine-running evidence. The proposed design therefore separates ignition from authoritative
engine-running state in additive canonical V3, requires an effective-dated `ENGINE_RUNNING`
capability at source time and forbids inference from zero speed, movement, external power,
charging or connectivity.

Proposed Phase-1 idle means supported combustion/hybrid Vehicle plus authoritative engine-running,
speed at most 3 km/h, WGS84 distance adjusted by two reported accuracies and at most 50 m, two or
more observations and five continuous minutes, with a two-minute maximum evidence gap. Accuracy is
required and capped at 100 m; gaps/contradictions become UNKNOWN and are not counted. Credited time
is the sum of bounded qualifying source-time intervals. No Notification, Operations fact or
fuel-waste estimate is proposed without a separately approved contract. Proposed V101-V103
boundaries remain unreserved pending approval/version preflight. The proposed first change set is
`US-51-MONITOR-IDLE-TIME-CS01-CANONICAL-ENGINE-SEMANTICS-001`; production mappings remain disabled.

### US-51 CS01 canonical engine semantics

D1-D11 were approved on 2026-09-18. CS01 implements the provider-neutral V3 contract with nullable
separate ignition and authoritative engine-running state/source fields, governed topic/DLT names,
exact enum validation and retained canonical Tenant/Vehicle/identity semantics. Legacy V1/V2
`engineState` remains ignition-only and unchanged.

Flyway remains V100. No V3 topic bean, producer routing, durable consumer, persistence column or
production capability was activated. The publisher rejects V3 before Kafka interaction so it
cannot acknowledge data that current history cannot store. Flespi, Traccar and Generic stay V2-only
and cannot advertise `ENGINE_RUNNING`; V3 fixture construction exists only in test sources.

Next: `US-51-MONITOR-IDLE-TIME-CS02-V101-HISTORY-CAPABILITY-001`. It must verify V101 is free and
add the approved immutable history/capability boundary before any V3 production cutover. Verified
production device/protocol mapping and physical acceptance remain independent later gates.
Accounting remains 73/87 and Flyway remains V100.
