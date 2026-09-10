# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; pluggable-onboarding CS01–CS10 is technically complete and independently verified at V76. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts. The trusted provider/Tenant authority, retention, observability, audit and rebuild remediation is implemented. Accounting remains 72/87 with 15 remaining; physical-device/real-provider final acceptance is still required.

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

## Implemented persistence (V73–V76)

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

The CS09 backend-contract remediation makes the frozen onboarding sequence executable: create a DRAFT device, create an ACTIVE binding to an ACTIVE same-Tenant provider connection, start an effective-dated same-Tenant Vehicle association, then explicitly activate the device. Association and binding preparation accept only DRAFT/ACTIVE devices; DISABLED/RETIRED devices are ineligible and RETIRED remains terminal. Activation atomically requires both the active provider authority and current Vehicle association and fails with safe `TRACKING_DEVICE_NOT_READY` or `TRACKING_STALE_VERSION` semantics. Device list/detail and mutation responses add current provider-binding and Vehicle-association summaries. Managers receive the binding/provider IDs, safe lifecycle/version/display/type/alias facts, masked external reference and bounded safe configuration; view-only callers receive no binding mutation version or credential material. The deterministic current-binding read prefers ACTIVE, then DRAFT, then DISABLED and enables refresh-safe rebind.

The CS09 throughput remediation preserves the unchanged signed-ingress workload and `>=1000 msg/s` burst gate. `JdbcTrackingStore` now reuses only transaction-stable batch facts: one retention-policy read per batch and one transaction-scoped advisory-lock/authority reload per unique device and Vehicle. Per-message validation, source-time association, dedupe/conflict, ordering, trust, immutable history and latest projections are unchanged. Three accepted burst runs measured 1,300.5, 1,675.3 and 1,229.4 msg/s (median 1,300.5); a post-functional run measured 1,390.9 msg/s, sustained runs remained above 200 msg/s, complete Tracking Chromium passed 24/24, Tracking Java passed 112/112 and full Maven passed 1,513 tests with zero failures/errors and 15 skipped. Flyway remains V76 with no schema/index change. This synchronizes the accepted technical performance fact only; CS09 final frontend acceptance and physical FMC130/Flespi evidence remain separate.

CS05 registers exactly one FLESPI adapter with capabilities `POLLING`, `SOURCE_TIMESTAMP`, `ACCURACY`, `SPEED`, `HEADING` and `HISTORY`. `DISCOVERY`, `REPLAY`, `MESSAGE_ID`, `SEQUENCE`, `IGNITION`, `ODOMETER` and `ENGINE_HOURS` remain absent/pending real capture. The adapter uses persisted connection endpoint/alias/safe configuration plus bounded device cursors, performs bounded sequential per-device HTTPS requests with no device threads, and maps provider DTOs directly to normalized candidates inside the adapter package. V76 per-binding watermarks are authoritative; cold/retry overlap is at most five minutes and Tracking dedupe remains final authority.

The legacy `FlespiPollingAdapter`, device-specific `FlespiAdapterProperties`, in-memory business watermark and `TrackingIngressBridge` loopback are removed. The coordinator alone resolves `credentialReference` through `IntegrationSecretResolver` per execution and clears the transient buffer; rotation therefore needs no restart. Flespi owns no scheduler or job retry. Safe transient provider failures are returned to coordinator backoff, and connection-scoped adapter health remains separate from aggregate Tracking health. Rollback disables coordinator/connection execution and retains all V76 data; dual polling is not supported. CS05 introduced no migration, permission, event, outbox or Integration packet route. CS06 subsequently added the authorized management APIs, CS07 hardened their security boundary, CS08 delivered the Provider Connections UI and CS09 completed the DRAFT-first device-onboarding workflow plus backend and throughput remediations. CS10 completes controlled technical acceptance: PostgreSQL 47/47, Tracking Java 112/112, Chromium 24/24, Maven 1,513/0/0/15 and architecture 49/49 passed. Transaction-local latest-trusted source-time reuse is confined to each locked Vehicle and transaction; three unchanged post-fix bursts measured 1,601.5, 1,752.8 and 1,516.2 msg/s, with sustained results 774.9, 649.9 and 546.2 msg/s. Real FMC130/Flespi evidence remains mandatory. Accounting stays 72/87; US-49 remains blocked.

## External acceptance preparation

`US-48-LIVE-VEHICLE-TRACKING-EXTERNAL-ACCEPTANCE-PREPARATION-001` is COMPLETE. The application repository contains an operator runbook plus capture and final-acceptance templates. Preparation does not constitute physical-provider evidence and does not change US-48 acceptance or story accounting. The required source remains one physical FMC130 using real LTE/GNSS and a real Teltonika channel/device in Flespi, with a least-privilege token resolved through an opaque environment reference. Flespi capture must verify `ident`, `timestamp`, `position.latitude`, `position.longitude` and truthful optional `position.accuracy`, `position.speed` and `position.direction`; ignition, odometer, engine hours, message identity and sequence remain unsupported and must not be synthesized. Current head remains V76 with no V77. Next task: `US-48-FMC130-FLESPI-EXTERNAL-CAPTURE-001`; US-49 remains blocked until independent final acceptance passes.

The first operational readiness execution is `BLOCKED / PHYSICAL_HARDWARE_REQUIRED`: no physical FMC130 or genuine FMC130-originated Flespi message was available or verifiable, so the mandated hard gate stopped before acceptance-database and application runtime startup. No provider account, credential, connection or capture was claimed. The capture template remains `NOT_EXECUTED`, US-48 remains `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`, and accounting remains 72/87. After physical FMC130/LTE/GNSS/Flespi acquisition, rerun `US-48-FMC130-FLESPI-EXTERNAL-CAPTURE-READINESS-001`; only readiness PASS advances to the capture rerun. US-49 remains blocked.

## External hold and downstream disposition

The ARB approves `ON_HOLD_EXTERNAL_PREREQUISITE` for US-48 physical capture after two unchanged hardware-blocked readiness attempts. No further readiness task is scheduled until a physical FMC130 or verified live Flespi prerequisite materially changes. Physical capture and independent final acceptance remain mandatory; US-48 is not complete or waived, Flyway remains V76, and accounting remains 72/87.

The frozen technical Tracking contract is sufficient to begin downstream decisions without acceptance inheritance. US-49 is `TECHNICAL_DEPENDENCY_SATISFIED / READY_FOR_PRODUCT_DECISIONS` because it needs trusted WGS84 position, Vehicle and source time. US-50 and US-52 are likewise ready for product decisions against normalized speed and trusted-position/Routing contracts respectively; real speed fidelity remains a US-50 final-evidence gate. US-53 immutable-history dependency is satisfied but overlay decisions follow earlier producers. US-51 remains `BLOCKED_BY_REQUIRED_TELEMETRY_CAPABILITY` because current FLESPI does not advertise IGNITION and no accepted alternate engine-state source exists. US-54 remains blocked by US-49..53 producers. Full US-55 remains blocked because tamper, spoofing and battery signals/product semantics are not established, despite existing loss/delay/trust support. Next task: `US-49-MANAGE-GEOFENCES-PRODUCT-DECISIONS-001`.

## US-49 Manage Geofences — frozen product decision

`US-49-MANAGE-GEOFENCES-PRODUCT-DECISIONS-001` is `PRODUCT_DECISIONS_COMPLETE / IMPLEMENTATION_PENDING`. Tracking owns the aggregate, evaluation state, transitions and proposed APIs; US-63 Delivery Zones remain a distinct last-mile serviceability/capacity concept and are not reused. The only Phase 1 types are `DEPOT`, `CUSTOMER_SITE` and `UNAUTHORIZED_ZONE`.

Geometry is polygon-only WGS84 in `(longitude, latitude)` order. The API accepts an open ring of 3–100 distinct vertices; persistence will store a canonical closed ring. Consecutive duplicates, zero-area and self-intersecting polygons are invalid. Boundary points are inside. Orientation is irrelevant and preserved. The encoded polygon is limited to 16 KiB. No PostGIS or map-provider dependency is approved: the proposed representation is PostgreSQL JSONB vertices plus numeric bounding-box columns, with application-owned pure-Java geometry and at most 500 ACTIVE geofences per Tenant.

The proposed `Geofence` aggregate carries Tenant, unique Tenant-scoped name, type, polygon, optional logical Organization location ID, alert configuration, lifecycle, audit facts and optimistic version. `DEPOT` and `CUSTOMER_SITE` require an active same-Tenant Organization location; `UNAUTHORIZED_ZONE` is free-standing and cannot reference one. The lifecycle is `DRAFT -> ACTIVE <-> DISABLED -> RETIRED`, with RETIRED terminal and configuration editable only in DRAFT or DISABLED.

Evaluation consumes only nonduplicate, TRUSTED, valid WGS84, IN_ORDER accepted positions no older than five minutes at evaluation. Delayed, out-of-order, future, stale and untrusted positions cannot change current geofence state. Ordering uses source timestamp and position UUID as deterministic tie-break. The first eligible observation initializes state silently. ENTERED or EXITED is confirmed only after two distinct consecutive eligible observations agree with the candidate side. There is no dwell event. Overlapping geofences evaluate independently, and an unauthorized-zone result is never suppressed by another overlap.

Confirmed unauthorized entry is `UNAUTHORIZED_ZONE_ENTERED`, severity HIGH and always alertable; discipline and exception-case creation are outside US-49. All same-Tenant Vehicles are evaluated; no Trip dependency or selector model is approved. A bounded Tracking-owned durable evaluation job is created atomically with an accepted position. Raw packets do not become cross-module events. Each transition has deterministic SHA-256 identity over Tenant, geofence, definition version, Vehicle, from/to state and confirming position.

The proposed minimized P1-01 event is `VehicleGeofenceTransitionedV1`, consumed by Notification only after implementation. It contains geofence ID, Vehicle ID, nullable Organization location ID, geofence type, transition, severity, source timestamp and definition version; it excludes coordinates, device/provider facts and person/Customer data. Operations integration is NONE for US-49; US-55 owns any later GPS-exception integration.

The proposed permissions are `GEOFENCE_VIEW`, `GEOFENCE_MANAGE` and `GEOFENCE_EVENT_VIEW`, with Tenant isolation only and no generic ABAC engine. Proposed APIs are under `/api/v1/tracking/geofences`. Likely forward migration V77 would add Tracking-owned geofence definition, Vehicle state, transition and evaluation-job persistence; V77 is not created by this decision task. The next controlled task is `US-49-MANAGE-GEOFENCES-CS01-DOMAIN-PORTS-001`. Accounting remains 72/87 and US-48 remains on external-prerequisite hold.
