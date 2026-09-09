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

## 2026-09-09 amendment — supported-adapter runtime onboarding

`US-48-PLUGGABLE-DEVICE-ONBOARDING-ARCHITECTURE-001` is APPROVED and amends only the provider/device onboarding edge. The product term is `PLUG_AND_PLAY_FOR_SUPPORTED_ADAPTERS`: administrators may create, bind, activate, disable, retire and reassign devices and provider connections at runtime when a reviewed adapter is installed. Unknown proprietary protocols still require an adapter release/deployment; arbitrary runtime JAR/plugin upload is prohibited.

Tracking remains the owner. CS01 has implemented the provider-neutral `TrackingProviderAdapter` outbound SPI, immutable `TrackingProviderAdapterRegistry`, strict `ProviderType`, exact immutable capability catalogue and bounded provider-neutral DTOs. Duplicate provider types fail startup, unsupported providers/discovery fail safely, and architecture tests prevent the SPI from depending on Spring, persistence, Jackson or Tracking adapters. Vendor DTOs, Tenant claims and secrets remain adapter-private. The existing V74 provider binding evolves into the runtime provider-connection aggregate in the next CS02; V75 is authorized but not yet implemented. Secrets remain opaque references resolved by `IntegrationSecretResolver`; no per-device environment variables are permitted.

One database-discovered coordinator claims due ACTIVE connections with bounded workers, provider quotas and PostgreSQL leases; no scheduler/thread exists per device. Activation and disable take effect without restart. Internal polling adapters use a private provider-neutral ingestion port that reloads ACTIVE connection, Tenant and device binding before delegating to the existing normalized ingestion service. The signed HMAC/nonce endpoint remains unchanged for external push providers. Neither path accepts payload Tenant authority.

Existing position/history contracts, effective-dated Vehicle association, dedupe/trust/freshness/connectivity/retention, permissions and real-device acceptance remain unchanged. Provider/device management reuses `TRACKING_DEVICE_MANAGE`; new provider-neutral management APIs are authorized but not implemented. Accounting remains 72/87 and US-49 remains blocked.
