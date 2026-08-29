# State Machines and Lifecycles

## Delivery Operations

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
