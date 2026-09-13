# ADR — Hybrid Pluggable Telemetry Platform

**Status:** ACCEPTED / MVP PROMOTED  
**Date:** 2026-09-13  
**Scope:** US-48 and Wave C Tracking platform enabler

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
