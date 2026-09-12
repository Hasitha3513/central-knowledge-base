# Tracking Module

## Status and scope

US-48 is `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED_EXTERNAL_SYSTEM`; pluggable-onboarding CS01–CS10 is technically complete and independently verified through V76. The current repository Flyway head is V80 for US-49 geofence index hardening. Tracking is a dedicated top-level bounded context for provider-neutral live Vehicle position facts and Tracking-owned geofence evaluation. US-49 is `COMPLETE / ACCEPTED`; CS01–CS07, CS07A, technical closure and independent final acceptance are complete. Accounting is 73/87 with 14 remaining and physical-device/real-provider US-48 final acceptance is still required.

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
