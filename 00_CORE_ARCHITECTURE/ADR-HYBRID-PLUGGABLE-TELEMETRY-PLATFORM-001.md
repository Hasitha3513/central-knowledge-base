# ADR — Hybrid Pluggable Telemetry Platform

**Status:** ACCEPTED / MVP PROMOTED  
**Date:** 2026-09-13  
**Scope:** US-48 and Wave C Tracking platform enabler

## Supported provider decision

Plug-and-play means supported devices are onboarded through registered adapters without changing
Tracking domain or the Kafka/Redis/Timescale pipeline. It is not universal proprietary-protocol
support. Phase 1 retains Flespi production polling with an environment-backed token, authorizes a
separate Traccar production HTTPS polling adapter with an environment-backed opaque bearer token,
and retains Generic signed-HMAC callback ingestion. Future protocols require a reviewed adapter or
an authorized signing gateway. Flespi and Traccar are first-class peers, not fallbacks.

```text
GPS devices -> Flespi | Traccar | custom signing gateway
            -> registered inbound provider adapter
            -> canonical V1/V2 boundary -> Kafka
            -> Timescale immutable history + Redis disposable live state
            -> US-48 and Tracking evaluators
```

```text
UNTRUSTED provider/device -> TLS credential or HMAC -> server Tenant and source-time binding
TRUSTED Tracking boundary -> normalization -> Kafka durable boundary
```

Adapters never write Redis, TimescaleDB, legacy Tracking tables or another module's persistence.
Frontend/payload Tenant values are never authority.

## Decision

The current MVP promotes a provider-neutral multi-gateway normalization pipeline, Kafka durable
telemetry backbone, Redis Tenant-qualified live-state projection, TimescaleDB append-only history,
gateway administration UX, and a permission-aware live Fleet map. The work extends
US-48 and supports US-49 through US-55; it creates no new story ID and does not waive physical
US-48 acceptance.

The V75/V76 provider-connection and device-binding aggregates remain authoritative. Secrets
remain opaque `IntegrationSecretResolver` references. The signed endpoint at
`POST /api/integration/v1/tracking/positions` remains the external trust boundary. An ACTIVE
provider key resolves provider type, alias and Tenant before Flespi, Traccar or Generic
normalization; URL and payload values never supply Tenant authority.

Kafka topic `tracking.telemetry.ingested.v1` is keyed by `{tenantId}:{vehicleId}`, uses
idempotent production with `acks=all`, and is the sole durable high-rate buffer. Redis Streams are
superseded. Redis stores only `tracking:live:{tenantId}:{vehicleId}` projections populated by a
Kafka consumer; TimescaleDB stores normalized append-only history through an independently
acknowledged batch consumer. PostgreSQL continues to own configuration, nonce/dedupe authority,
audit and detector state. Production readiness fails when Kafka or TimescaleDB is unavailable;
Redis failure degrades live reads but Kafka retains replay authority.

The provider UI enhances existing connection APIs rather than adding a singleton Tenant config.
The Redis-backed `GET /api/v1/tracking/vehicles/live` requires `TRACKING_VIEW`, is Tenant scoped,
bounded and PII-minimized. The React-Leaflet map polls at ten seconds while visible/online.

V86 is the immutable hybrid foundation. Forward V87 configures 7-day chunks, Tenant/Vehicle
segmented compression after 7 days and 180-day raw retention. US-52 immutable route-geometry
persistence moves to V88. Kafka, Spring Kafka, Redis, TimescaleDB, Spring Data Redis, Leaflet and
React-Leaflet are approved dependencies. Accounting remains 73/87.

Implementation status is `IMPLEMENTATION_IN_PROGRESS / TS04_COMPLETE`. V87 implements the
transactional `tracking-telemetry-persister-group` consumer, Tenant-scoped database idempotency,
deterministic exact static reduction and the approved Timescale policies. US-48 physical acceptance
remains independent; US-52 CS02 is next at V88.

V91 closes durable detector fanout. Timescale history and the canonical Tracking evaluation-dispatch
intents commit atomically before Kafka acknowledgement. GEOFENCE, SPEED and ROUTE_DEVIATION execute
asynchronously from exact Tenant-qualified immutable history with bounded claims, leases, retry and safe
failure classification. Evaluator failure never removes accepted history. Redis projection remains an
independent consumer, legacy detector jobs remain compatible, and IDLE is not activated.

Traccar Phase 1 is HTTPS-only bounded `/api/positions` polling with timeout, pagination,
restart-safe watermark, retry/backoff, rate-limit/circuit-breaker and sanitized health behavior.
Username/password, Basic authentication, plaintext secrets, disabled TLS verification and direct
unsigned callbacks are prohibited. An external signing proxy may submit Traccar-origin facts only
through the existing Generic HMAC contract.

Onboarding is DRAFT-first: choose provider, configure endpoint and opaque credential reference,
test connectivity, discover or manually identify and validate the device, bind it to authenticated
Tenant and one authorized Vehicle for the effective interval, validate a normalized sample, then
activate. Disablement, rebinding and credential rotation preserve effective-dated audit evidence.
One physical device has at most one authoritative active provider binding per source-time interval;
active-active Flespi/Traccar ingestion is not authorized. Existing `TRACKING_DEVICE_MANAGE` governs
the workflow; no new permission is approved.

Canonical V1 remains immutable. V2 uses `tracking.telemetry.ingested.v2` and `.v2.dlt`, separate
serialization, consumer-first rollout and common cross-version/cross-provider dedupe. Capabilities
are effective-dated registry data. Missing signal evidence remains absent and is never inferred.
Operational behavior includes provider clock validation, stale/accuracy/trust classification,
bounded DLT replay, safe adapter disablement and observability without secrets or precise location.

US-48 technical status and its external physical-evidence hold remain unchanged. US-55 is
`IMPLEMENTATION_IN_PROGRESS / CS03_COMPLETE`; accounting remains 73/87, Flyway is V96 and the
next queue is `US-55-HANDLE-GPS-EDGE-CASES-CS04-EVALUATION-REDIS-DETECTOR-GUARDS-001`. V96 retains
the provider-neutral authority model while adding only nullable canonical V2 history evidence,
append-only database enforcement and effective-dated Tenant/device capability history. Redis remains
disposable live state and no provider secret is persisted.
