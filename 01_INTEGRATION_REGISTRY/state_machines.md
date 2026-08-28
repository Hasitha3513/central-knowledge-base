# State Machines and Lifecycles

## Delivery Operations

`MVP-1.3-DELIVERY-MODULE-FOUNDATION-001` introduces only the Delivery lifecycle baseline enum. It does not implement transitions, persistence, REST commands, events, assignment, proof-of-delivery, failure handling, redelivery, or return-to-origin workflows.

| State | Status |
| :--- | :--- |
| `DRAFT` | FOUNDATION_BASELINE |
| `READY_FOR_ASSIGNMENT` | FOUNDATION_BASELINE |
| `ASSIGNED` | FOUNDATION_BASELINE |
| `OUT_FOR_DELIVERY` | FOUNDATION_BASELINE |
| `DELIVERED` | FOUNDATION_BASELINE |
| `FAILED` | FOUNDATION_BASELINE |
| `REDELIVERY_SCHEDULED` | FOUNDATION_BASELINE |
| `RETURN_TO_ORIGIN` | FOUNDATION_BASELINE |
| `CANCELLED` | FOUNDATION_BASELINE |

Transition commands, guards, audit requirements, tenant behavior, idempotency, event emission, and compensations must be registered before implementing US-56 through US-62 workflows.

### US-56 Frozen Readiness Semantics

`MVP-1.3-US56-PRODUCT-DECISIONS-001` freezes, but does not implement, the US-56 lifecycle subset:

- Create produces `DRAFT`.
- Successful order-readiness validation produces `READY_FOR_ASSIGNMENT`.
- Changing priority, service type, window, instructions or references in `DRAFT`/`READY_FOR_ASSIGNMENT` produces `DRAFT` and requires revalidation.
- Assignment is not performed by US-56. Assignment target and target eligibility remain deferred.
- Later `ASSIGNED` or execution states make US-56 requirement fields immutable when those states are implemented.
