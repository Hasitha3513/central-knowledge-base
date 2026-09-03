# Operations Bounded Context

Status: `US78_PRODUCT_DECISIONS_FROZEN / IMPLEMENTATION_NOT_STARTED`
Owner: Operations
Decision: `ADR-US78-OPERATIONAL-EXCEPTION-BOUNDARY.md`

## Phase 1: Current MVP Scope

US-78 will own a cross-domain operational-exception **lifecycle**, not source-domain detection or meaning. The future `OperationalExceptionCase` owns confirmed classification/severity, assignment, response/resolution SLA, escalation, corrective actions, RCA, resolution, closure/reopen, and append-only history.

The originating domain owns the source aggregate, detection, source evidence, and every business correction. Operations uses only typed events and logical UUID references; it has no foreign repository, entity, JPA relationship, physical foreign key, or SQL access.

### First accepted producers

- Routing: accepted Route disruption creation under current US-22 ownership.
- Delivery: accepted specialized Delivery exception creation under US-62 ownership.

The same durable `OperationalExceptionFactV1` contract and Operations lifecycle must be proven for both. The accepted US-68 Last-Mile Planner remains a Delivery read-only orchestration and is not an intake producer.

### Lifecycle and policy

- States: `OPEN`, `ACKNOWLEDGED`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`.
- Escalation: monotonic `L0..L3` level/history fact, not a case state.
- Severity: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
- Category: `OPERATIONAL`, `SAFETY`, `COMPLIANCE`, `CUSTOMER`, `FINANCIAL`, `TECHNICAL`, `SECURITY`.
- Assignment: same-Tenant `ROLE_QUEUE` or eligible `USER`; no arbitrary external assignee.
- SLA: server-side 24x7 elapsed targets of 8h/72h, 4h/24h, 1h/8h, and 15m/2h by ascending severity; at-risk at 75%, with critical escalation at intake.
- Corrective actions: typed `CORRECTIVE`/`PREVENTIVE` owner/due/status records; source correction remains with the source domain.
- RCA: mandatory and independently approved for high/critical; controlled cause category plus bounded code/summary/contributing factors.
- Closure: resolved, required actions complete, required RCA approved, resolution validated, authorized closer. High/critical closer differs from resolver; RCA approver differs from author.
- Reopen: supported for recurrence or ineffective prior resolution with reason and audit.

### Security and privacy

Every future aggregate, row, event, idempotency key, worker query, and cache key is Tenant-owned. Frozen permissions are `OPERATIONAL_EXCEPTION_VIEW`, `OPERATIONAL_EXCEPTION_MANAGE`, `OPERATIONAL_EXCEPTION_ASSIGN`, `OPERATIONAL_EXCEPTION_ESCALATE`, `OPERATIONAL_EXCEPTION_RCA`, `OPERATIONAL_EXCEPTION_CLOSE`, and `OPERATIONAL_EXCEPTION_AUDIT_VIEW`.

Operations stores minimized facts and logical evidence/US-83 Document references. It never copies POD images/signatures, medical certificates, full GPS tracks, financial documents, credentials, OTPs, addresses, contact destinations, provider payloads, or whole source logs. No authoritative retention duration exists; no delete/purge API is frozen.

### Planned persistence

No Operations table or migration exists yet. Implementation is expected to introduce normalized, Operations-owned equivalents of `operational_exception_case`, `operational_exception_assignment_history`, `operational_exception_corrective_action`, `operational_exception_rca`, and `operational_exception_history`. All tables require `tenant_id UUID NOT NULL`, Tenant-leading indexes, Tenant-consistent same-module foreign keys, immutable history, and optimistic versions for mutable records. The implementation task must inspect the then-current Flyway head and document the exact post-migration data dictionary; V62 is not reserved.

### Planned APIs and UI

The authenticated route family is `/api/v1/operational-exceptions`: pageable list/detail/history plus explicit classify, acknowledge, assign, start, escalate, corrective-action, RCA, resolve, close, reject-resolution, and reopen commands. There is no manual create, generic status PATCH, delete, source correction, raw evidence, customer, or export route.

The operator feature is planned at `/operations/exceptions` under the shared `AppLayout`, with queue filters, badges, assignment, SLA indicators, gated actions/RCA, and pageable timeline. It is not an analytics dashboard or customer portal.

## Integrations

- Intake and escalation facts reuse P1-01's shared durable outbox with at-least-once delivery and `(tenantId,eventId)` consumer dedupe.
- Notification/US-77 owns channel/template/recipient/quiet-hour/provider behavior for escalation.
- US-81's accepted scheduling pattern owns Tenant-aware trigger execution; Operations owns due instants and idempotent escalation behavior.
- US-80 remains the reusable workflow boundary; Operations v1 owns only its fixed aggregate transitions.
- US-83 owns document content/version/access/retention; Operations holds a logical reference only.

## Phase 2: Post-MVP / Future Roadmap

US-38 Fuel and US-55 Tracking may publish the frozen intake after their own detector semantics are accepted. US-86 may use the Operations lifecycle for real-world disruption cases, but owns coordinated constraint/replanning semantics separately. Trip, Cargo, Driver/Fleet, Compliance, and Integration producer activation also requires exact registered event mappings and tests.

Case merge/parent-child relationships, configurable business calendars, automatic person load balancing, customer visibility, analytics, and automated retention remain deferred.
