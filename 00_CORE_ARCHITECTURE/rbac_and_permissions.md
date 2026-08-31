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
| `DELIVERY_FAIL_RECORD` | Record failed delivery attempts and contact attempts | IMPLEMENTED_US59 |
| `DELIVERY_FAIL_VIEW` | View failed delivery attempts and failure history | IMPLEMENTED_US59 |
| `DELIVERY_FAIL_ESCALATE` | Escalate failed deliveries and resolve escalations | IMPLEMENTED_US59 |
| `DELIVERY_RETURN_INITIATE` | Initiate Return to Base (RTO) for failed deliveries | IMPLEMENTED_US59 |
| `DELIVERY_COMPLETE` | Complete successful delivery outside US-57 POD finalization | FROZEN_NOT_SEEDED; US-57 finalization uses `DELIVERY_POD_CAPTURE` atomically |
| `DELIVERY_REDELIVERY_SCHEDULE` | Schedule or reschedule redelivery attempts | IMPLEMENTED_US60 |
| `DELIVERY_REDELIVERY_VIEW` | View redelivery schedules and history | IMPLEMENTED_US60 |
| `DELIVERY_ANALYTICS_VIEW` | View delivery performance analytics, trends, and KPIs | IMPLEMENTED_US61 |
| `DELIVERY_EXCEPTION_CREATE` | Report specialized delivery exception cases | IMPLEMENTED (US-62) |
| `DELIVERY_EXCEPTION_VIEW` | View delivery exception cases and history | IMPLEMENTED (US-62) |
| `DELIVERY_EXCEPTION_MANAGE` | Investigate and update delivery exception cases | IMPLEMENTED (US-62) |
| `DELIVERY_EXCEPTION_RESOLVE` | Resolve or cancel delivery exception cases | IMPLEMENTED (US-62) |
| `DELIVERY_EXCEPTION_ESCALATE` | Escalate delivery exception cases | IMPLEMENTED (US-62) |

Future Delivery APIs must seed and enforce the narrowest applicable permission before marking any US-56 through US-62 story implemented.
