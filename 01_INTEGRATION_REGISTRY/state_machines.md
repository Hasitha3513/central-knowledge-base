# State Machines and Lifecycles

## External Integration (US-73 Implemented and Accepted)

### Configuration lifecycle

`DRAFT -> ACTIVE -> DISABLED`; `DISABLED -> ACTIVE` is allowed only after a successful probe against the current configuration version within 15 minutes. `ACTIVE` configuration and mapping are immutable; material change requires disablement and an immutable next mapping version. There is no delete state or route. Disabling stops new exchange claims/attempts until re-enabled.

Lifecycle is independent of derived provider health: `UNKNOWN`, `HEALTHY`, `DEGRADED`, `UNAVAILABLE`, or `AUTH_FAILED`. A transient provider failure changes health and exchange outcome but never silently disables configuration or marks synchronization successful.

### External exchange lifecycle

`PENDING -> IN_PROGRESS -> SUCCEEDED` for success. Retryable failure produces `IN_PROGRESS -> RETRY_SCHEDULED -> IN_PROGRESS`; five total attempts are bounded to immediate, 30-second, 2-minute, 10-minute, and 30-minute scheduling. Exhaustion or permanent failure produces terminal `FAILED`. A five-minute lease recovers abandoned `IN_PROGRESS` claims without changing the stable idempotency identity. Historical attempts and terminal state cannot be edited, manually retried, or marked successful in US-73.

These lifecycles are `IMPLEMENTED_ACCEPTED_US73`; they add no state to any Transportation, Delivery, Finance, HRM, Fuel, Tracking, Compliance, or Document aggregate.

## Operational Exception Case (US-78 Implemented, Acceptance Pending)

`OPEN -> ACKNOWLEDGED -> IN_PROGRESS -> RESOLVED -> CLOSED` is the standard path. `OPEN -> IN_PROGRESS` is permitted only as an explicit acknowledge-and-start command. Resolution validation rejection produces `RESOLVED -> IN_PROGRESS`. An authorized reasoned reopen for recurrence or ineffective resolution produces `CLOSED -> IN_PROGRESS`. Direct `OPEN -> RESOLVED/CLOSED` and `ACKNOWLEDGED -> CLOSED` are forbidden.

Escalation is a monotonic `L0..L3` level/history fact, not a lifecycle state. Acknowledgement stops the response clock; resolution stops the resolution clock. Closure requires completed required corrective actions, approved required RCA, successful resolution validation, and `OPERATIONAL_EXCEPTION_CLOSE`. For high/critical cases, closer differs from resolver and RCA approver differs from RCA author. Optimistic versions govern every mutable case/action/RCA transition.

Corrective-action states are `OPEN -> IN_PROGRESS -> COMPLETED`, with reasoned `OPEN/IN_PROGRESS -> CANCELLED`. US-78 never uses those transitions to mutate the source aggregate. This lifecycle is `IMPLEMENTED_US78_ACCEPTANCE_PENDING` and adds no state to Routing, Delivery, Trip, Freight, Driver/Fleet, Fuel, Tracking, Notification, Document, or customer aggregates.

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
