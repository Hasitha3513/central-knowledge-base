# RBAC and Permissions

## Authority Rules

- Runtime authority comes from authenticated identity, active server-side tenant membership, and role-template permissions assigned through tenant membership.
- Frontend visibility is not authorization.
- New backend actions must be enforced server-side and seeded through forward-only migrations before public API exposure.

## Delivery Operations Permission Catalogue

US-56 seeds and enforces its four Delivery permissions through V46 and Spring Security. Later permissions remain frozen and unseeded until their owning stories are implemented.

| Permission | Purpose | Implementation status |
| :--- | :--- | :--- |
| `DELIVERY_VIEW` | View tenant-scoped Delivery Orders | IMPLEMENTED_US56 |
| `DELIVERY_CREATE` | Create Delivery Orders | IMPLEMENTED_US56 |
| `DELIVERY_UPDATE` | Update editable Delivery requirements | IMPLEMENTED_US56 |
| `DELIVERY_ASSIGN` | Validate readiness for later assignment; no assignment occurs in US-56 | IMPLEMENTED_US56 |
| `DELIVERY_EXECUTE` | Execute delivery progress commands | FROZEN_NOT_SEEDED |
| `DELIVERY_POD_CAPTURE` | Create/resume online POD drafts, manage draft evidence and finalize valid proof | IMPLEMENTED_US57 |
| `DELIVERY_POD_VIEW` | View privacy-sensitive POD metadata and stream/download evidence within the Tenant | IMPLEMENTED_US57 |
| `DELIVERY_COMPLETE` | Complete successful delivery outside US-57 POD finalization | FROZEN_NOT_SEEDED; US-57 finalization uses `DELIVERY_POD_CAPTURE` atomically |
| `DELIVERY_FAIL` | Record delivery failure | FROZEN_NOT_SEEDED |
| `DELIVERY_REDELIVER` | Schedule or manage redelivery | FROZEN_NOT_SEEDED |
| `DELIVERY_EXCEPTION_MANAGE` | Manage Delivery exceptions | FROZEN_NOT_SEEDED |
| `DELIVERY_REPORT_VIEW` | View Delivery reports | FROZEN_NOT_SEEDED |

Future Delivery APIs must seed and enforce the narrowest applicable permission before marking any US-56 through US-62 story implemented.
