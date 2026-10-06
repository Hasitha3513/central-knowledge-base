# SaaS Subscription, Account Billing, and Deactivation Policy

Policy ID: SAAS-SUBSCRIPTION-BILLING-001
Version: 1
Approved requirements recorded: 2026-10-06
Status: ACCEPTED_BUSINESS_REQUIREMENTS / NOT_IMPLEMENTED_BY_THIS_CHANGE
Owner: Provider-side Platform Management; Identity remains authoritative for account/membership lifecycle and authorization facts.
Architecture: [ADR-SAAS-PLATFORM-MANAGEMENT-001](ADR-SAAS-PLATFORM-MANAGEMENT-001.md).

This is the canonical target commercial/account-administration policy. Existing application RBAC and billing behavior must not be described as changed until independently implemented and verified. Labels below are requirements identifiers, not production enums or permission constants.

## 1. Organization and subscription rules

| ID | Approved rule |
| :--- | :--- |
| ORG-01 | Every registered organization subscribes independently and receives a dedicated operational tenant database. |
| ORG-02 | Both a holding/parent organization and every subsidiary require their own subscriptions. No inherited or pooled subscription exemption is granted. |
| ORG-03 | Departments, branches, and sections are internal organizational units, not separately subscribed tenants. Creating one does not create a separate subscription or connection charge. |
| ORG-04 | Account billing belongs to the organization; changing an account's role, department, branch, section, or number of authorized modules does not multiply its seat count. |
| ORG-05 | Group ownership and a common payer do not merge subscriptions, databases, permission scopes, or billing evidence. |

## 2. Complimentary Super Admin and paid accounts

| ID | Approved rule |
| :--- | :--- |
| ACC-01 | Each organization subscription includes exactly one designated complimentary Super Admin account. |
| ACC-02 | Every other tenant user account is paid. Delegated Admin, employee, read-only, or any other role/category is not an additional free-account exception. |
| ACC-03 | The complimentary benefit is attached to the designated organization Super Admin entitlement, not a test such as 'has an admin permission'. Role renaming or adding admin access cannot remove a charge. |
| ACC-04 | Super Admin may assign admin access to employee accounts. Delegated Admin may configure other users' access within the authorized organization, permission ceiling, and resource scope. |
| ACC-05 | Commercial account deactivation requires approval by the current authorized Super Admin of that organization. A delegated Admin, employee, or payment integration cannot bypass this approval. |
| ACC-06 | Preserve the last viable organization owner/admin. Ownership transfer must atomically transfer the single complimentary entitlement and leave an auditable trail, not create a second free account. |
| ACC-07 | Named accounts and explicit membership identity are required. Shared credentials are not a mechanism for avoiding paid-account licensing. |

Provider platform operators are not organization Super Admins. Their provider-side identities do not grant an extra free tenant membership. A provider operator who also receives an ordinary customer tenant account is subject to that tenant's account policy.

An employee/customer/supplier business record is not itself a login account. A bare invitation that has not created an account is distinct from account creation; an invitation workflow that actually creates an account must not silently exempt it from BILL-03.

## 3. Connection fee and monthly account charges

| ID | Approved rule |
| :--- | :--- |
| BILL-01 | A standard one-time connection/setup fee is payable for the organization subscription in addition to account charges. It is not an advance payment or an automatic credit against subsequent monthly fees. |
| BILL-02 | Recurring charges are based on organization account billing history, excluding the designated complimentary Super Admin. 'Active' means access-enabled membership, not 'logged in this month' or concurrent sessions. Inactive accounts retain past charges already earned under this policy. |
| BILL-03 | Every non-complimentary account created at any time during a billing month incurs the full applicable monthly account fee for that same month. Early-month, mid-month, and final-day creation are not prorated and must not be deferred to the next month's charge. |
| BILL-04 | Billing must use historical account/entitlement facts and effective timestamps, not only a month-end query of accounts currently marked ACTIVE. Deleting, locking, changing roles, or late-deactivating an account cannot erase billable history. |
| BILL-05 | Do not charge the same account twice for the same monthly fee merely because a message is retried, it has several roles/internal-unit assignments, or its status is replayed. |
| BILL-06 | No additional free-account class or unapproved branch/module/role fee, minimum paid-seat quantity, or recurring organization base fee is introduced. |

A simple month with a stable set of chargeable accounts has:

`monthly account charge = sum of the applicable full monthly prices for chargeable accounts in that organization and billing month`.

The initial bill additionally contains the separate connection-fee line. Exact rate values, currencies, taxes, and discounts are not invented here. The invoice must preserve its price/policy version and account-month attribution.

The number of currently active accounts minus one is not a complete historical billing algorithm: the free entitlement must actually be designated, new-account charges must be retained, and the deactivation rules below must be applied.

## 4. Five-day post-bill deactivation rule

| ID | Approved rule |
| :--- | :--- |
| DEACT-01 | The deactivation window starts when the relevant month's bill is generated, not when the account was created, when a user reads an email, when payment is due, or when a payment succeeds. |
| DEACT-02 | For the timely-deactivation benefit, Super Admin approval AND effective account deactivation must both be completed within a maximum of five days after that bill is generated. A request alone or approval without completed deactivation is insufficient. |
| DEACT-03 | Timely approved/effective deactivation removes the applicable recurring account charge from that bill's month, subject to the explicitly unresolved creation-month overlap in Decision D-01. The action and any correction must remain auditable. |
| DEACT-04 | If the account has not actually been deactivated within the five-day window, its applicable full account charge remains included in that month's bill. A late approval/deactivation does not retroactively remove that charge. |
| DEACT-05 | A late deactivation is not prohibited: it still requires Super Admin approval, stops ordinary account access, and is considered for subsequent billing periods. The expired billing-adjustment window is not an instruction to leave an unwanted account able to log in. |
| DEACT-06 | A rejected/unapproved request does not deactivate the account and does not remove the charge. It cannot be approved by another tenant's Super Admin. |
| DEACT-07 | Record the relevant bill and billing month, original generation instant, calculated deadline, request, approver identity, approval instant, effective deactivation instant, and billing outcome. Reprinting, resending, retrying, or making an administrative correction to the same bill must not reset the original window. |
| DEACT-08 | Timing is established by trusted server-side facts. Do not backdate an action, accept client timestamps as authority, or silently label a failed deactivation successful to meet the deadline. |

The product rule is five days; the exact clock/calendar convention is tracked in D-02. No five-business-day extension or deadline calculated from the payment due date is authorized. Tests must cover the approved deadline exactly and immediately after it once D-02 is resolved.

Security containment is separate from commercial deactivation. A compromised account may need immediate access blocking under security policy without waiting for billing approval. Such a lock does not by itself establish approved deactivation or a fee exemption. Preserve existing security controls and last-owner recovery safeguards.

## 5. Bill integrity and late-month creation

A full-month account charge belongs to the account's creation month even if that month's initial bill has already been generated. The implementation must support a traceable bill adjustment or supplemental document attributable to that same month; it may not silently shift the charge to the next month's service period.

Do not overwrite or delete finalized/paid invoice evidence. A permitted adjustment must identify the original bill, account-month, rule, amount, actor, and correction evidence. Draft adjustment versus finalized-bill correction mechanics, settlement/refund treatment, and invoice-generation schedule require approved execution decisions; no unreviewed accounting or jurisdiction-specific rule is created by this document.

Payment retries, provider callbacks, seat-event retries, and provisioning retries must be idempotent. Payment received, environment ready, and operational access enabled remain separately evidenced facts. A successful browser redirect is not authoritative payment evidence.

## 6. Required acceptance scenarios (not executed by this documentation change)

| ID | Scenario | Required outcome |
| :--- | :--- | :--- |
| T01 | Parent organization plus two subsidiaries | Three subscriptions, dedicated databases, and designated complimentary Super Admin entitlements. |
| T02 | Add a branch, department, or section | No additional subscription, connection fee, or duplicate account seat. |
| T03 | One Super Admin, two delegated Admins, eight employees | One complimentary account; ten other accounts subject to paid-account policy. |
| T04 | Promote a paid employee to Admin | Remains paid; account identity and historical charge unchanged. |
| T05 | Attempt to grant a second free entitlement | Rejected, including concurrent ownership-transfer attempts. |
| T06 | Create an ordinary account on the first, middle, or last day | Full fee belongs to that creation month; no day-based proration. |
| T07 | Create an account after the month's initial bill | Same-month attributable full charge through governed adjustment/supplemental evidence. |
| T08 | Ordinary ongoing account; approval and deactivation both within window | Applicable recurring charge removed for that bill's month with evidence. |
| T09 | Request within window; approval/effective deactivation after deadline | Full current-month charge retained. |
| T10 | Approval within window; actual deactivation fails or is late | No false success; full current-month charge retained. |
| T11 | Deactivation after deadline with valid approval | Access stops; current-month charge retained; later periods use effective inactive history. |
| T12 | Delegated Admin attempts unapproved commercial deactivation | Denied; no licensing or billing side effect. |
| T13 | Foreign-tenant Super Admin attempts approval | Denied without cross-tenant disclosure. |
| T14 | Reprint/resend/retry a generated bill | Original five-day anchor unchanged; no duplicate fee. |
| T15 | Duplicate/out-of-order account or payment facts | Reconciled by authoritative identity/version; no duplicate charge or free entitlement. |
| T16 | Temporary security lock | Access may be blocked; no automatic commercial deactivation/fee exemption. |
| T17 | Same paid account has multiple roles/branches/modules | One account identity for that organization's account-month charge. |
| T18 | Super Admin-only organization | No additional base charge or minimum paid-seat rule invented; entitlement scope remains D-03. |
| T19 | Same account newly created and timely deactivated within overlapping window | DECISION_REQUIRED under D-01; no invented precedence or automatic billing adjustment. |
| T20 | Exact cutoff, immediately late, month boundary, timezone/DST boundary | Deterministic behavior after D-02 is resolved; no inferred timezone convention. |
| T21 | Connection/provisioning/payment retry | One approved connection-fee obligation; no duplicate collection/provisioning. |
| T22 | Existing inactive account with no chargeable activity in a later month | Not charged merely because its retained audit/account row still exists. |

These are acceptance requirements, not a claim of passing application tests.

## 7. Explicit unresolved execution decisions

The approved commercial rules above remain recorded. The following missing edge details block only the affected automation/design, not this documentation update. They must not be filled by an agent's preferred pricing model.

| ID | Unresolved item | Required treatment |
| :--- | :--- | :--- |
| D-01 | A new account incurs a full creation-month fee under BILL-03, but the same account may also satisfy that bill's five-day deactivation rule. The owner has not stated precedence between those two rules. | Preserve both approved rules. Do not invent a creation-month refund, a renewal-only restriction on DEACT-03, or a new minimum charge. Obtain the explicit overlap decision before automating that case. |
| D-02 | Billing timezone, bill-generation schedule, and whether 'five days' is elapsed 120 hours or a defined calendar-day cutoff, including the exact inclusive endpoint. | Persist the original bill anchor. Define/test an explicit five-day deadline convention; no extension to business days or substitution of a due-date anchor. |
| D-03 | Operational-feature scope of the complimentary Super Admin and economics when no paid accounts exist. | Keep the one account complimentary. Do not add a monthly base fee, paid-seat minimum, or administration-only limitation without a product decision. |
| D-04 | Rates/currencies, reactivation and repeated create/deactivate behavior, draft/final invoice correction mechanics, payment timing, refunds, and nonpayment recovery duration. | No invented prices, proration, credits, discounts, reconnection charges, or deletion policy. Resolve the relevant item before execution. |
| D-05 | A request approved in time but operationally blocked by a platform failure. | Record actual outcome under DEACT-02/04; any exceptional customer remedy requires explicit authorization, never backdated success. |

## 8. Implementation gate and related records

Read [Platform Management](../02_MODULES_KNOWLEDGE/platform_management.md), [target integration boundaries](../01_INTEGRATION_REGISTRY/platform_management_boundaries.md), [Multi-Tenancy Standards](multi_tenancy_standards.md), and the existing [RBAC registry](rbac_and_permissions.md).

The existing ordinary Identity user-deactivation authority must be reconciled with Super Admin approval before target activation. This file supplements the current runtime RBAC registry with approved target requirements; it does not add a live permission constant or change an existing endpoint. The future implementation must update the canonical RBAC, state-machine, API, event, dependency, and owning-module registries with exact verified contracts in the same change.
