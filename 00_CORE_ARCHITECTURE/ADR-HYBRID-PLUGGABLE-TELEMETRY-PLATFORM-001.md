# ADR — Hybrid Pluggable Telemetry Platform

**Status:** ACCEPTED / MVP PROMOTED  
**Date:** 2026-09-13  
**Scope:** US-48 and Wave C Tracking platform enabler

## Decision

The current MVP promotes a provider-neutral multi-gateway normalization pipeline, Redis
Tenant-qualified live-state projection and ingestion stream, TimescaleDB append-only telemetry
history, gateway administration UX, and a permission-aware live Fleet map. The work extends
US-48 and supports US-49 through US-55; it creates no new story ID and does not waive physical
US-48 acceptance.

The V75/V76 provider-connection and device-binding aggregates remain authoritative. Secrets
remain opaque `IntegrationSecretResolver` references. The signed endpoint at
`POST /api/integration/v1/tracking/positions` remains the external trust boundary. An ACTIVE
provider key resolves provider type, alias and Tenant before Flespi, Traccar or Generic
normalization; URL and payload values never supply Tenant authority.

Redis keys are `tracking:live:{tenantId}:{vehicleId}` and streams are
`tracking:stream:{tenantId}`. TimescaleDB stores normalized append-only history. PostgreSQL
continues to own configuration, nonce/dedupe authority, audit and detector state. Stream items
are acknowledged only after an idempotent historical commit. Production readiness fails when
either promoted storage capability is unavailable; no silent durability downgrade is allowed.

The provider UI enhances existing connection APIs rather than adding a singleton Tenant config.
The Redis-backed `GET /api/v1/tracking/vehicles/live` requires `TRACKING_VIEW`, is Tenant scoped,
bounded and PII-minimized. The React-Leaflet map polls at ten seconds while visible/online.

V86 is allocated to this hybrid foundation. US-52 immutable route-geometry persistence moves to
V87 after the hybrid change sets. Redis, TimescaleDB, Spring Data Redis, Leaflet and React-Leaflet
are approved dependencies. Accounting remains 73/87.
