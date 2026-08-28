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
