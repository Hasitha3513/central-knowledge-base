# Platform Management: Target Operator and Organization Guide

Status: APPROVED_WORKFLOW_REQUIREMENTS / SOFTWARE_NOT_IMPLEMENTED_BY_THIS_CHANGE
Date: 2026-10-06
Audience: Provider platform staff, organization Super Admins, and delegated organization Admins.

This guide explains approved future behavior. It does not claim that screens, buttons, endpoints, invoices, databases, or automated charges already exist. The [canonical billing policy](../00_CORE_ARCHITECTURE/saas_subscription_billing_policy.md) governs the rules and unresolved edge cases.

## Organization subscription

Register each organization separately. A parent holding company and each subsidiary require independent subscriptions. Departments, branches, and sections stay within the organization's subscription and do not have separate connection charges.

Each organization receives a dedicated operational tenant database and uses the common ERP product. Holding-company ownership does not automatically authorize access to subsidiary data.

Each customer account belongs to one organization only. There is no organization selector or tenant switcher. A person working with several organizations uses separate named accounts and credentials; permissions, billing, and audit remain separate. A username may look like `hasitha.u7k9q2.sce`, but its suffix and employee/EPF numbers grant no organization access.

The organization pays the standard connection/setup charge in addition to account charges. The connection charge is not an advance credited against monthly user bills. No actual prices are specified in this guide.

## Super Admin and delegated administration

Each organization subscription includes one designated complimentary Super Admin account. Every other tenant user account is paid, including delegated Admin and read-only accounts. Promoting an employee to Admin does not make the account free.

The Super Admin may delegate admin access. A delegated Admin may manage users' access only within authorized limits and the same organization. The Super Admin is not a provider platform administrator and cannot access other tenants, change SaaS prices, or obtain database superuser rights.

Do not share the Super Admin login as a substitute for employee accounts. Ownership transfer must preserve one complimentary entitlement and a viable organization owner.

## Monthly account charges

An ordinary account created on any date in a month incurs the full monthly fee for that same month. There is no day-based proration or automatic postponement to the next month. An account created after the initial bill still needs a same-month attributable charge through the approved billing-document mechanism.

Changing branches, departments, sections, roles, or authorized modules does not multiply that account's count within the organization. HR employee records and customer records are not automatically login accounts.

For a stable example with one Super Admin, two delegated Admins, and eight employee accounts, the complimentary account count is one and the paid-account count is ten. This illustrates counting, not an actual price or a full historical invoice algorithm.

## Account deactivation

1. Identify the account, the applicable month's bill, its original generation time, and its recorded five-day cutoff.
2. Obtain approval from the organization's current authorized Super Admin. An employee or delegated Admin request alone is insufficient.
3. Complete actual deactivation within the five-day window for the eligible bill-month adjustment. Approval without successful deactivation does not qualify.
4. Verify the effective inactive state and the audited billing outcome. Do not assume that clicking an approval action proves completion.

If deactivation is not completed within five days after bill generation, the applicable account charge remains in that month's bill. Later approved deactivation may still stop account access, but it does not erase that retained current-month fee.

Resending or reprinting the same bill does not start a new five-day period. Security locks may block access immediately but are not automatic commercial deactivations or fee exemptions.

An account both created and deactivated in the overlapping creation-month/five-day scenario is explicitly awaiting a precedence decision. Exact calendar-versus-elapsed-day cutoff handling is also recorded as a decision prerequisite. Operators and agents must not invent a free-account/refund rule or a different deadline.

## Provider Platform Management responsibilities

Provider staff manage organizations, subscriptions, price versions, licensing evidence, connection fees, invoices/payments, provisioning status, schema/backup health, and authorized support operations. Tenant self-service must expose only the organization's own billing and permitted administration.

Payment confirmation and database readiness are separate outcomes. A paid organization with failed provisioning must be shown truthfully as not ready and recovered without duplicate charges or duplicate resources. Customer ERP business records remain in their owning tenant systems.

Provider staff also govern each organization's approved industry/product capabilities. Tenant admins configure only within that entitlement; roles, organization codes, database names, menus, and feature flags cannot unlock another capability. The common frontend adapts to entitlement, configuration, permissions, scope, and approved localization, while backend enforcement remains authoritative.

Release operators apply one approved package in controlled stages: verified fresh/upgrade paths, internal/test tenants, pilots, then rings. Every database reports independent schema, backfill, verification, and recovery status. Partial rollout is truthful and recoverable, not an all-tenant transaction. Code deployment, schema readiness, entitlement, activation, and authorization are separate. Customer accounts never receive migration privileges or database credentials; missing routing fails closed.

Operational activation, billing automation, and migration require independent implementation acceptance. This documentation update changes no live account, subscription, bill, or database.
