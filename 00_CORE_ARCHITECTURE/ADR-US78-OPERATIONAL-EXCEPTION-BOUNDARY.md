# ADR: US-78 Operational Exception Boundary

Status: `IMPLEMENTED / ACCEPTANCE_PENDING`
Date: 2026-09-04
Decision owner: Architecture Review Board / Operations

## Context

The source requires one way to classify, prioritize, assign, track SLA, escalate, investigate, record corrective action/RCA, validate resolution, close, and reopen operational exceptions. Existing domains already own different exception meanings and lifecycles. Duplicating their records or querying their tables would break ownership and turn a cross-cutting queue into a generic god object.

## Decision

1. Ratify `operations` as a top-level bounded context for the lifecycle **after** a domain exception exists.
2. Use a hybrid `OperationalExceptionCase` aggregate with logical `sourceModule`, `sourceType`, and `sourceId` references. Domains retain detection, meaning, evidence, and corrective mutation.
3. Accept only registered `OperationalExceptionFactV1` events from trusted producers. There is no arbitrary/manual case-creation API in v1.
4. Make intake durable through P1-01's shared `DurableEventPublisher` and `integration_outbox_event`; the case row deduplicates `(tenantId, sourceEventId)`. No second outbox/inbox or broker is approved.
5. Integrate accepted Routing disruption and Delivery exception producers first. Trip, Freight, Driver/Fleet, Fuel US-38, Tracking US-55, and US-86 remain later producer contracts.
6. Freeze case states `OPEN -> ACKNOWLEDGED -> IN_PROGRESS -> RESOLVED -> CLOSED`, with reasoned `RESOLVED -> IN_PROGRESS` and `CLOSED -> IN_PROGRESS`; escalation is a monotonic level, not a state.
7. Use server-side continuous elapsed SLA clocks and the accepted US-81 Tenant-aware scheduling pattern. Operations owns no holiday/resource calendar and no scheduler engine. Fixed aggregate transitions follow the accepted US-80 workflow boundary without creating a configurable workflow engine.
8. Publish only the real-consumer `OPERATIONAL_EXCEPTION_ESCALATED_V1` fact to Notification through P1-01. Notification retains recipient, channel, template, quiet-hour, retry, and provider ownership.
9. Keep evidence as source or US-83 Document logical references. Never copy POD, medical, financial, location-track, credential, or provider payload content.
10. Require Tenant isolation, optimistic versions, append-only history, seven narrow permissions, and separate resolver/closer plus RCA author/approver for high/critical cases.

## Consequences

V62 implements five Operations-owned Tenant tables, seven permissions, validated role queues, 17 authenticated lifecycle/query routes, the internal operator queue, and the two frozen durable contracts. There are no foreign table reads or physical foreign-domain keys. Source corrections remain separate authorized source-domain commands. The Operations queue is internal/operator-only and does not expose cases to customer self-service. Implementation evidence passes, but independent technical closure and final acceptance remain pending.

## Deferred

Additional detector integrations, configurable business calendars, person load-balancing, case merge/parent-child graphs, generic manual incidents, analytics, customer visibility, retention purge, and US-86 replanning are not part of US-78 v1.
