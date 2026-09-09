# State Machines and Lifecycles

## External Integration (US-73 Implemented and Accepted)

### Configuration lifecycle

`DRAFT -> ACTIVE -> DISABLED`; `DISABLED -> ACTIVE` is allowed only after a successful probe against the current configuration version within 15 minutes. `ACTIVE` configuration and mapping are immutable; material change requires disablement and an immutable next mapping version. There is no delete state or route. Disabling stops new exchange claims/attempts until re-enabled.

Lifecycle is independent of derived provider health: `UNKNOWN`, `HEALTHY`, `DEGRADED`, `UNAVAILABLE`, or `AUTH_FAILED`. A transient provider failure changes health and exchange outcome but never silently disables configuration or marks synchronization successful.

### External exchange lifecycle

`PENDING -> IN_PROGRESS -> SUCCEEDED` for success. Retryable failure produces `IN_PROGRESS -> RETRY_SCHEDULED -> IN_PROGRESS`; five total attempts are bounded to immediate, 30-second, 2-minute, 10-minute, and 30-minute scheduling. Exhaustion or permanent failure produces terminal `FAILED`. A five-minute lease recovers abandoned `IN_PROGRESS` claims without changing the stable idempotency identity. Historical attempts and terminal state cannot be edited, manually retried, or marked successful in US-73.

These lifecycles are `IMPLEMENTED_ACCEPTED_US73`; they add no state to any Transportation, Delivery, Finance, HRM, Fuel, Tracking, Compliance, or Document aggregate.

## Operational Exception Case (US-78 Accepted, Complete)

`OPEN -> ACKNOWLEDGED -> IN_PROGRESS -> RESOLVED -> CLOSED` is the standard path. `OPEN -> IN_PROGRESS` is permitted only as an explicit acknowledge-and-start command. Resolution validation rejection produces `RESOLVED -> IN_PROGRESS`. An authorized reasoned reopen for recurrence or ineffective resolution produces `CLOSED -> IN_PROGRESS`. Direct `OPEN -> RESOLVED/CLOSED` and `ACKNOWLEDGED -> CLOSED` are forbidden.

Escalation is a monotonic `L0..L3` level/history fact, not a lifecycle state. Acknowledgement stops the response clock; resolution stops the resolution clock. Closure requires completed required corrective actions, approved required RCA, successful resolution validation, and `OPERATIONAL_EXCEPTION_CLOSE`. For high/critical cases, closer differs from resolver and RCA approver differs from RCA author. Optimistic versions govern every mutable case/action/RCA transition.

Corrective-action states are `OPEN -> IN_PROGRESS -> COMPLETED`, with reasoned `OPEN/IN_PROGRESS -> CANCELLED`. US-78 never uses those transitions to mutate the source aggregate. This lifecycle is `COMPLETE_US78` after independent final acceptance and adds no state to Routing, Delivery, Trip, Freight, Driver/Fleet, Fuel, Tracking, Notification, Document, or customer aggregates.

## Delivery Operations

### US-70 Customer Self-Service (Implemented and Accepted)

- Access has no client-set status column. It is effective `ACTIVE` only while `revoked_at IS NULL`, server time is before `expires_at`, the bound Tenant/Delivery/Customer/contact/action checks pass, and the active-token cap is satisfied. Revocation is terminal; expiry is a derived terminal condition. Re-entry for the same Notification delivery-attempt issuance key rotates the hash in the same access record and immediately invalidates the prior raw token.
- `DELIVERY_PREFERENCE`, `REDELIVERY_REQUEST`, and `ISSUE` submissions start `SUBMITTED`. A later operator-authorized workflow may move a preference/redelivery request to terminal `ACCEPTED`, `DECLINED`, or `SUPERSEDED`; customer self-service cannot choose outcome status. An accepted request does not itself mutate Delivery—the established US-60/US-64 action remains authoritative.
- `FEEDBACK` is created as terminal `RECORDED`; duplicate submission is prevented by Tenant/Delivery/Customer uniqueness and request idempotency.
- These are `IMPLEMENTED_ACCEPTED_US70` lifecycles and add no current DeliveryOrder state. Final acceptance confirmed the V59 persistence constraints, fail-closed access lifecycle, and non-binding submission behavior. Existing US-56–US-69 transitions and events remain unchanged.

The accepted US-56 implementation contains only the current production Delivery states below. Earlier foundation documents discussed later candidate states, but they are not implemented authority.

| State | Status |
| :--- | :--- |
| `DRAFT` | IMPLEMENTED_US56 |
| `READY_FOR_ASSIGNMENT` | IMPLEMENTED_US56 |

Later states remain story-scoped and must be registered before implementation.

### US-56 Frozen Readiness Semantics

`MVP-1.3-US56-PRODUCT-DECISIONS-001` freezes and the accepted US-56 implementation enforces this lifecycle subset:

- Create produces `DRAFT`.
- Successful order-readiness validation produces `READY_FOR_ASSIGNMENT`.
- Changing priority, service type, window, instructions or references in `DRAFT`/`READY_FOR_ASSIGNMENT` produces `DRAFT` and requires revalidation.
- Assignment is not performed by US-56. Assignment target and target eligibility remain deferred.
- Later `ASSIGNED` or execution states make US-56 requirement fields immutable when those states are implemented.

### US-57 Frozen POD and Completion Semantics

`MVP-1.3-US57-POD-PRODUCT-DECISIONS-001` freezes but does not implement:

- POD lifecycle: `DRAFT -> FINALIZED`; one POD per Delivery Order.
- Draft metadata/evidence is mutable with optimistic versioning. Finalized proof and evidence are immutable.
- US-57 adds only `DELIVERED` when its implementation is accepted.
- Transitional Delivery transition: `READY_FOR_ASSIGNMENT -> DELIVERED`, performed atomically with valid POD finalization.
- A Delivery in `DRAFT` is ineligible; an already delivered order rejects duplicate finalization.
- This transition records no assignment, Rider ownership, Trip execution or `OUT_FOR_DELIVERY` fact. When an authoritative execution model is introduced, POD eligibility must be narrowed through a separately approved forward decision.
- Successful finalization uses the server acceptance UTC instant for both POD acceptance and Delivery completion.
- Storage failure leaves the POD draft retryable and the Delivery unchanged; a POD cannot appear finalized with missing/unverified evidence.
# US-35 Fuel Card Local Lifecycle (Complete; Final Acceptance Passed at V65 Head)

US-35 freezes the local Fuel-owned lifecycle as `DRAFT`, `ACTIVE`, `SUSPENDED`, `BLOCKED`, `EXPIRED`, and `CANCELLED`. Issue creates `DRAFT`; a valid bound/restricted non-expired draft may activate; active may suspend or block; suspended may resume, block, or cancel; blocked may only cancel; cancellation is final. Expiry is effective after the expiry month in the Tenant timezone and prevents activation/resume. Blocked, expired, and cancelled cards never reactivate; replacement creates a new card. Lost/stolen cards are blocked immediately with reason.

This lifecycle expresses local operational control only. It neither activates nor blocks the provider account, and the UI must not imply provider confirmation. No arbitrary status mutation exists. Imported facts for inactive cards remain immutable evidence and become `CARD_INACTIVE / REVIEW_REQUIRED`.

# US-38 Fuel Exception Local Lifecycle (Accepted)

Technical closure identified persistence and retry compliance gaps without changing this lifecycle. Forward-only V67 remediation is complete: immutable correction attempts and durable handoff failure/retry evidence are implemented and verified. A CRITICAL case may enter `RESOLVED` only after handoff is `PUBLISHED` or `ACCEPTED`; this is a frozen precondition, not a new state.

The Fuel-owned lifecycle is exactly `OPEN -> UNDER_REVIEW -> CORRECTION_PENDING -> AWAITING_APPROVAL -> RESOLVED`. `OPEN -> UNDER_REVIEW -> RESOLVED` supports reasoned no-action resolution. Correction rejection returns `AWAITING_APPROVAL -> UNDER_REVIEW`; owner-command failure returns the case to visible retryable `CORRECTION_PENDING`. There is no generic status patch, delete, cancellation, category edit, or Fuel-local reopen.

Operations handoff is orthogonal `NOT_REQUIRED | PENDING | PUBLISHED | ACCEPTED | FAILED`; it does not duplicate the US-78 lifecycle. An escalated unresolved Fuel case remains `UNDER_REVIEW`. Resolution outcomes are `NO_ACTION_REQUIRED`, `CORRECTION_APPLIED`, `RECONCILED`, `EMERGENCY_REFUEL_ACCEPTED`, or `REFERRED_TO_OPERATIONS`. Corrections that affect inventory, cost, effective price, reconciliation, or source lifecycle require a requester-distinct approver and execute only through the owning module command or compensating fact.
# US-46 Driver Payroll-Input Batch (Accepted)

The V71 technical remediation now projects successful Integration delivery from `EXPORT_REQUESTED` to `EXPORTED` only from durable controlled-file/hash evidence. A fully released correction may set its referenced original to `SUPERSEDED`; no HRMS acknowledgement, rejection, posting, settlement, or payment state is introduced. PostgreSQL races and controlled-file Chromium evidence verify the transition and replay behavior; final acceptance remains pending.

`DRAFT -> VALIDATED -> APPROVED -> EXPORT_REQUESTED -> EXPORTED`.

Validation failure leaves the batch `DRAFT`. External failure leaves it `EXPORT_REQUESTED` while Integration records retry/attempt state. Approved content is immutable. A later approved/exported `CORRECTION` batch may mark its predecessor `SUPERSEDED`; correction lines are compensating deltas and preserve the original. There is no delete, generic status patch, reopen, paid, posted, reconciled, or acknowledged state. The preparer cannot approve the same batch, and optimistic version conflicts fail closed.

# US-47 Transport Billing Record (Implemented / Acceptance Pending)

The frozen regular lifecycle is `DRAFT -> VALIDATED -> APPROVED -> FINALIZED -> EXPORT_REQUESTED -> EXPORTED`. Validation failure and edits leave/return the record to `DRAFT`. A reasoned `DRAFT -> CANCELLED` transition is final. The preparer cannot approve the same record.

Finalization locks source/Customer/currency/lines/tax/cost-centre/calculation/approval facts. A new `REVERSAL` record references and exactly compensates one finalized original; the original becomes `REVERSED` only when that reversal finalizes. There is no delete, reopen, generic status patch, tax-invoice-issued, posted, booked, settled, paid or accounting-acknowledged state. `EXPORTED` proves only controlled Integration file/hash delivery. This lifecycle is `PRODUCT_DECISIONS_FROZEN_US47`; implementation has not started.

# US-48 Provider Connection and Device Binding

The pluggable supported-adapter decision authorizes provider connections, Tracking devices and device-provider bindings with `DRAFT -> ACTIVE <-> DISABLED -> RETIRED`; RETIRED is terminal. New devices/connections begin DRAFT. Activation requires supported adapter type, valid bounded configuration, resolvable credential, successful connection validation where applicable, and a non-conflicting same-Tenant device binding. Provider disable prevents new coordinator claims but preserves position history and last-known state.

Provider connection test status is orthogonal: `NOT_TESTED`, `PASS`, `AUTH_FAILED`, `UNREACHABLE`, or `INVALID_CONFIGURATION`. It creates no telemetry state. The provider-connection lifecycle is `IMPLEMENTED_US48_V75`. Device-provider binding lifecycle and atomic rebinding are `IMPLEMENTED_US48_V76`: rebinding disables the previous active binding, creates the replacement and updates the legacy compatibility projection in one transaction; it never rewrites effective-dated Vehicle association or position history. ACTIVE requires both device and provider connection ACTIVE, RETIRED is terminal, and optimistic version conflicts fail closed. V1–V75 remain immutable.

CS04 execution is `IMPLEMENTED_US48_V76`: only ACTIVE, due, unleased/expired connections are claimable. A claim has one runtime-stable owner and bounded expiry; only the current owner may renew or release it, and expiry permits safe recovery. Disable prevents future claims, while authority is reloaded after claim and again at ingestion so in-flight disabled/rebound work fails closed. Binding source/message watermarks advance only for ACCEPTED or DUPLICATE normalized outcomes; REJECTED and failed outcomes retain their prior watermark for at-least-once retry. Connection and binding next-poll cursors provide bounded fair scheduling without changing lifecycle state.
