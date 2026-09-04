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
- US-70 customer self-service uses possession authorization, not operator RBAC: a 256-bit opaque token is bound server-side to one Tenant, Delivery, Customer/contact context, expiry, and allow-listed actions. Public HTTP matching never makes the resource public; invalid, expired, revoked, mismatched, and cross-Tenant credentials fail closed. Customer principals receive no `DELIVERY_*`, `NOTIFICATION_*`, Identity role, or tenant membership. Automated Notification-time issuance and internal lifecycle/security revocation are frozen; manual operator token administration API/UI is deferred and no new permission is frozen.

## P0-05 Contextual Authorization Decisions

- Implemented contextual facts: active Tenant membership, target-user Tenant ownership, actor permission ceiling, and cross-Tenant role-assignment protection.
- No generic ABAC/policy engine was introduced.
- No creator-versus-approver segregation rule was added because current Trip records and approved contracts do not provide an authoritative creator fact. This remains a governance/data-model prerequisite, not an implicit runtime rule.

## Fuel Performance Permission Decision (US-37)

US-37 freezes one new read-only capability, `FUEL_PERFORMANCE_VIEW`, for Tenant-scoped vehicle and privacy-sensitive Driver Fuel performance analytics. Existing `FUEL_ISSUE_VIEW`, `FUEL_COST_VIEW`, and `REPORT_VIEW` do not imply this authority. It remains `FROZEN_NOT_SEEDED` until implementation; the product-decision task creates no migration. The permission grants no write, threshold configuration, export, Driver discipline, US-38 exception, or raw-source access. Tenant authority is server-derived, cross-Tenant Vehicle/Driver identifiers are safe not-found, and frontend visibility never substitutes for backend authorization.

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

## Notification Permission Catalogue

US-69 reuses the existing US-77 Notification permissions; it introduces no Delivery notification-send permission.

| Permission | Purpose | Implementation status |
| :--- | :--- | :--- |
| `NOTIFICATION_VIEW` | View the authenticated internal user's own IN_APP notification inbox | IMPLEMENTED_US77 |
| `NOTIFICATION_RULE_VIEW` | View Notification rules, templates, masked delivery/attempt diagnostics, and effective customer notification preferences within the active Tenant | IMPLEMENTED_US77_US69 |
| `NOTIFICATION_RULE_MANAGE` | Manage rules/templates and replace a same-Tenant customer's complete Email/SMS preference profile | IMPLEMENTED_US77_US69 |

US-69 history and preference reads require `NOTIFICATION_RULE_VIEW`; preference replacement requires `NOTIFICATION_RULE_MANAGE`. Literal external `/api/v1/...` paths and servlet-context-relative paths are covered by security regressions. Customer identifiers resolve under the server-derived Tenant, cross-Tenant targets return safe `404`, and client-supplied Tenant or recipient authority is never accepted.

## External Integration Permission Catalogue

US-73 seeds these permissions through V61 and enforces them server-side on the literal and effective Integration routes. US-73 is independently accepted and complete.

| Permission | Purpose | Implementation status |
| :--- | :--- | :--- |
| `INTEGRATION_VIEW` | View same-Tenant configurations, mappings, health, and masked exchange status | IMPLEMENTED_US73 |
| `INTEGRATION_MANAGE` | Create and update `DRAFT` or `DISABLED` configurations and declarative mappings | IMPLEMENTED_US73 |
| `INTEGRATION_TEST` | Run the bounded, non-business connection probe | IMPLEMENTED_US73 |
| `INTEGRATION_ACTIVATE` | Enable or disable an eligible same-Tenant configuration | IMPLEMENTED_US73 |
| `INTEGRATION_AUDIT_VIEW` | View same-Tenant Integration exchange and masked attempt diagnostics | IMPLEMENTED_US73 |
| `INTEGRATION_RECONCILE` | Reserved for a later approved reconciliation mutation workflow; grants no US-73 action | SEEDED_RESERVED_NO_ROUTE_US73 |

Activation of future `FINANCIAL` or `RESTRICTED` exchanges requires a different authorized actor from the last configuration author. The accepted non-sensitive filesystem probe does not require dual control. Frontend visibility never substitutes for backend authorization, and all Tenant scope comes from active server-side context.

## Operational Exception Permission Catalogue (Implemented, Acceptance Pending)

US-78 V62 seeds seven narrow Operations permissions. They are enforced on every literal `/api/v1/operational-exceptions/**` route, its effective servlet-context form, and the owning use case.

| Permission | Purpose | Status |
| :--- | :--- | :--- |
| `OPERATIONAL_EXCEPTION_VIEW` | View same-Tenant queue and non-sensitive case detail | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_MANAGE` | Acknowledge/start, classify with reason, manage notes/actions, and resolve | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_ASSIGN` | Assign and reassign to validated user/role queue | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_ESCALATE` | Perform manual escalation without changing severity | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_RCA` | View, author, and approve sensitive RCA subject to SoD | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_CLOSE` | Validate/reject resolution, close, and reasoned reopen | IMPLEMENTED_US78_ACCEPTANCE_PENDING |
| `OPERATIONAL_EXCEPTION_AUDIT_VIEW` | View full immutable case history | IMPLEMENTED_US78_ACCEPTANCE_PENDING |

Contextual authorization is explicit: active Tenant, case Tenant, category/sensitivity, severity, assignment, lifecycle state, and recorded SoD actors. For high/critical cases the closer differs from the resolver and the RCA approver differs from the RCA author. Low/medium cases do not require dual control. No generic ABAC engine, customer/external authority, or cross-Tenant admin bypass is approved.
