# Operations Guide — Operational Exceptions

Status: US-78 implementation complete; independent acceptance pending

## Purpose and prerequisites

The Operations queue lets authorized internal operators triage and close operational exceptions detected by Routing and Delivery. It does not create source exceptions or change Route/Delivery source records. A case appears only after an authoritative Routing disruption or Delivery exception has committed and its durable fact has been processed.

Sign in to an active Tenant account and open **Operations → Operational Exceptions** (`/operations/exceptions`). The menu and actions are permission-gated; backend authorization remains authoritative.

## Permissions

| Permission | User-visible capability |
| :--- | :--- |
| `OPERATIONAL_EXCEPTION_VIEW` | View the same-Tenant queue and non-sensitive detail |
| `OPERATIONAL_EXCEPTION_MANAGE` | Classify, acknowledge, start, manage corrective actions, and resolve; eligible users may self-assign |
| `OPERATIONAL_EXCEPTION_ASSIGN` | Assign or reassign to a validated user or role queue |
| `OPERATIONAL_EXCEPTION_ESCALATE` | Escalate manually with a reason |
| `OPERATIONAL_EXCEPTION_RCA` | View, author, and approve RCA subject to segregation of duties |
| `OPERATIONAL_EXCEPTION_CLOSE` | Close, reject resolution, and reopen |
| `OPERATIONAL_EXCEPTION_AUDIT_VIEW` | View the full immutable timeline |

## Queue and search

The queue shows case reference, safe source summary, category, severity, lifecycle status, assignment, response due, resolution due, and SLA state. Filter by status, severity, category, source module, assigned user, assigned role, SLA state, or opened date range. Search accepts a case-reference exact/prefix value, a registered summary code, or an exact source UUID; free-text note/RCA search is intentionally unavailable.

Queue pages default to 20 rows and permit at most 100. Sorting is limited to opened time, updated time, severity, response due, resolution due, or status. A case from another Tenant is never visible or inferable.

## Standard workflow

1. Open a case and review its safe source reference, confirmed category/severity, SLA, current assignment, corrective actions, RCA, and permitted commands.
2. **Classify** only when category or severity needs correction. Supply a reason; the current case version is required.
3. **Acknowledge** the open case, then **Start** it. Starting directly from open is supported as the explicit acknowledge-and-start equivalent.
4. **Assign** the case to an active same-Tenant eligible user or approved role queue. Assignment changes require a reason. Critical cases cannot be left unassigned.
5. Add corrective or preventive actions. Choose a validated user/role owner, optional due time, and optional logical evidence/result reference. Start and complete each required action.
6. Use **Escalate** when manual escalation is justified. A reason and current version are required. Escalation moves only from L0 toward L3; it does not change severity.
7. For high/critical cases, record RCA with cause category, code, summary, and optional contributing factors. A different authorized actor must approve it.
8. **Resolve** only after required actions are complete. Enter a plain-text resolution note and optional safe source-result reference.
9. An authorized closer reviews the result and **Closes** it. For high/critical cases the closer must differ from the resolver and required RCA must already be independently approved.

If validation fails, **Reject resolution** with a reason to return the case to in-progress. If a closed issue recurs or the prior resolution proves ineffective, **Reopen** it with a reason; the resolution SLA restarts.

## SLA behavior

SLA is continuous server time (24x7): low 8h response/72h resolution, medium 4h/24h, high 1h/8h, and critical 15m/2h. At-risk begins at 75% of the resolution window. Critical intake escalates immediately. The Tenant-aware worker processes a bounded due set and emits safe Notification facts; operators do not need to keep the page open.

SLA states are `ON_TRACK`, `AT_RISK`, `BREACHED`, and `MET`. Resolved/closed cases cannot receive a stale worker escalation.

## Validation and conflicts

- Notes, reasons, action descriptions, RCA summary/factors, and cancellation reasons are plain text up to 2,000 characters; HTML is not interpreted.
- Assignment must identify exactly one eligible user or role queue.
- Every mutation uses the displayed expected version. If another actor changes the same record first, reload after `OPERATIONAL_EXCEPTION_CONFLICT` and deliberately retry.
- Direct open-to-resolved/closed and acknowledged-to-closed transitions are rejected.
- Resolution is rejected while a required action is incomplete; high/critical closure is rejected without approved RCA or required actor separation.

## History, privacy, and source corrections

Authorized audit viewers can page through immutable history (default 50, maximum 200), including lifecycle, classification, assignment, escalation, corrective-action, and RCA facts with actor/time/version context.

Operations displays and stores only safe logical references. It does not contain POD images/signatures, medical records, full GPS tracks, financial documents, credentials, OTPs, addresses/contact destinations, provider payloads, or whole source objects. When a Route or Delivery correction is required, use that owning feature's authorized workflow separately and retain only its safe result reference in Operations.

## Current limitations

- Only Routing disruption and Delivery exception producers are active.
- Fuel, Tracking, Trip, Cargo, Driver/Fleet, Compliance, Integration, and US-86 disruption-replanning producers remain unavailable until separately approved.
- There is no manual case creation, generic status edit, delete/purge, export, raw evidence, analytics dashboard, customer portal, case merge/parent-child graph, configurable business calendar, or automatic person load balancing.
- Retention is governed externally; Operations exposes no purge action.
