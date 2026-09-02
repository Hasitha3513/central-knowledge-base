# RBAC and Permissions

## Authority Rules

- Runtime authority comes from authenticated identity, active server-side tenant membership, and role-template permissions assigned through tenant membership.
- Frontend visibility is not authorization.
- New backend actions must be enforced server-side and seeded through forward-only migrations before public API exposure.
- HTTP authorization is fail-closed: public/self-service routes and permission rules are explicit, and unmatched routes are denied.
- `IDENTITY_MANAGE` authorizes Tenant-local identity administration only. User reads, lists, updates, deactivation, and new membership creation are constrained to the authenticated actor's active Tenant.
- Identity administrators may assign only roles whose active permissions are a subset of the administrator's current server-resolved permissions. They may not update or delete a global role template while it is assigned outside their Tenant.
- JWT role and permission claims are informational token contents; every bearer request reloads the active user, active membership, active Tenant, active roles, and active permissions from server-side state.
- Offline Sync is the documented dynamic-permission exception at the HTTP edge: the batch requires authentication and the application service authorizes each operation against its existing operation-specific permission.

## P0-05 Contextual Authorization Decisions

- Implemented contextual facts: active Tenant membership, target-user Tenant ownership, actor permission ceiling, and cross-Tenant role-assignment protection.
- No generic ABAC/policy engine was introduced.
- No creator-versus-approver segregation rule was added because current Trip records and approved contracts do not provide an authoritative creator fact. This remains a governance/data-model prerequisite, not an implicit runtime rule.

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
| `DELIVERY_ZONE_CREATE` | Create delivery polygon zones | IMPLEMENTED (US-63) |
| `DELIVERY_ZONE_VIEW` | View delivery zones and resolve coordinates | IMPLEMENTED (US-63) |
| `DELIVERY_ZONE_UPDATE` | Update delivery zone boundaries and metadata | IMPLEMENTED (US-63) |
| `DELIVERY_ZONE_ACTIVATE` | Activate or deactivate delivery zones | IMPLEMENTED (US-63) |
| `DELIVERY_ZONE_OVERRIDE` | Override delivery zone assignments | IMPLEMENTED (US-63) |
| `DELIVERY_SLOT_CREATE` | Create delivery time slots | IMPLEMENTED (US-64) |
| `DELIVERY_SLOT_VIEW` | View delivery time slots and availability | IMPLEMENTED (US-64) |
| `DELIVERY_SLOT_UPDATE` | Update delivery time slot parameters | IMPLEMENTED (US-64) |
| `DELIVERY_SLOT_ACTIVATE` | Activate or close delivery time slots | IMPLEMENTED (US-64) |
| `DELIVERY_SLOT_ASSIGN` | Reserve and assign delivery time slots | IMPLEMENTED (US-64) |
| `DELIVERY_SLOT_OVERRIDE` | Override delivery slot capacity limits | IMPLEMENTED (US-64) |
| `DELIVERY_RIDER_CREATE` | Onboard new delivery riders | IMPLEMENTED (US-65) |
| `DELIVERY_RIDER_VIEW` | View delivery rider profiles and availability | IMPLEMENTED (US-65) |
| `DELIVERY_RIDER_UPDATE` | Update rider profiles and schedule shifts | IMPLEMENTED (US-65) |
| `DELIVERY_RIDER_ACTIVATE` | Activate, deactivate, or suspend riders | IMPLEMENTED (US-65) |
| `DELIVERY_RIDER_ASSIGN` | Assign, reassign, and unassign riders | IMPLEMENTED (US-65) |
| `DELIVERY_RIDER_OVERRIDE` | Override rider zone mismatch or capacity | IMPLEMENTED (US-65) |

Future Delivery APIs must seed and enforce the narrowest applicable permission before marking any story implemented.
