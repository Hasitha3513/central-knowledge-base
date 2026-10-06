# ADR-SAAS-PLATFORM-MANAGEMENT-001: Organization Tenancy and SaaS Control Plane

Date: 2026-10-06
Status: ACCEPTED_TARGET_ARCHITECTURE / IMPLEMENTATION_NOT_STARTED_BY_THIS_CHANGE
Authority: Product owner approval of the Platform Management proposals and subsequent billing-policy clarification.
Change set: KB-SAAS-PLATFORM-POLICY-001
Scope: Knowledge-base documentation only; no application changes, migrations, customer provisioning, invoicing, payment collection, or production activation.

## Context and precedence

The enterprise product is an industry-agnostic, multi-tenant SaaS ERP, not only a Transport application. The owner's supported business-model labels are D2B, B2B, B2C, D2C, C2C, C2B, and B2G. These labels describe intended business relationships; they do not create additional tenancy boundaries or establish implemented workflow coverage. Organizations include individual traders and street vendors as well as larger enterprises.

This ADR replaces the shared-database assumption for the APPROVED TARGET SaaS isolation model. The existing Transportation implementation remains the separately documented shared-application/shared-database, row-discriminated baseline until a governed implementation and migration are accepted. This document does not retrospectively change that evidence, its membership constraints, the existing Transport MVP accounting, or any release authorization.

The canonical commercial rules are in [SaaS subscription and billing policy](saas_subscription_billing_policy.md). Its explicit product decisions supersede earlier proposals for prorated account creation or administrator exemptions. Unresolved boundary cases are recorded there, not silently resolved by this ADR.

## Decisions

### 1. Organization is the subscription and tenant boundary

- Every organization has its own subscription and dedicated operational tenant database.
- A parent holding company and every subsidiary/sub-organization each require their own subscription. Group membership neither supplies a free subscription nor pools tenant databases.
- Departments, branches, and sections remain within their owning organization's tenant. They do not require separate subscriptions or connection fees merely because an internal unit is added.
- One account working across several internal units is one account in that organization, not one billable seat per unit or role.
- Parent/subsidiary relationships do not grant data access. Group reporting requires explicit authorization and approved contracts; no automatic cross-database access is approved.
- A common login identity, when eventually supported across organizations, must retain distinct organization memberships, permissions, and billing attribution. The existing single-membership MVP is not changed here.

### 2. Common product, dedicated tenant operational data

Customers use a common ERP frontend/product codebase. Each organization receives a fresh operational database initialized from governed schema migrations and safe reference configuration, never another customer's records, credentials, or demonstration transactions.

Database-per-tenant does not imply one physical server or one backend deployment per organization. Dedicated compute topology, regional placement, capacity tiers, and routing implementation require their own execution design. The minimum accepted isolation requirement is a distinct operational database and authorized tenant-specific routing. There is no permission to fork the product code for every tenant.

Retain canonical tenant UUIDs, server-validated membership authority, resource authorization, tenant-bearing events, tenant-qualified caches/storage/jobs, database identity checks, and isolation tests. A browser-supplied organization ID or database name is not routing authority. Database-per-tenant is not a reason to remove existing tenant guards.

### 3. Provider-owned Platform Management System

A separate, secured Platform Management System is the provider's SaaS control plane. It owns the commercial organization/tenant registry, subscription lifecycle, pricing catalogue, one-time connection charges, user licensing, SaaS billing/payment evidence, provisioning orchestration, and platform operational oversight.

The control plane has its own management database and privileged provider administration boundary. Its tenant-attributed subscription and payment metadata may be managed centrally; this is not permission to pool customer ERP transactions in that database. Customers receive only their own authorized billing/administration self-service views.

The customer ERP data plane owns business operations. Customer Finance, Transport Billing, and the existing System resilience/integrity module must not become the SaaS subscription owner. Provider invoices and customer sales invoices are different business records.

The control plane may itself be implemented as a modular application. This ADR approves separation of the provider management system, not a general microservice decomposition of ERP business domains.

### 4. Organization administration and licensing

Each organization subscription includes exactly one designated complimentary Super Admin account. All other tenant user accounts, including delegated administrators and read-only users, are paid accounts under the monthly policy. Assigning an admin role does not create a free-account entitlement.

The organization Super Admin may delegate admin access to employee accounts. Delegated admins may configure other users' permissions only within their authorized ceiling and organization scope. Only the organization Super Admin may approve commercial account deactivation. Preserve last-administrator protection, governed-permission restrictions, audit evidence, and backend authorization.

The organization Super Admin is neither a provider platform administrator nor a database superuser. Provider identities are a different security boundary, not additional complimentary customer accounts. An ownership transfer must move the one complimentary entitlement without leaving two free accounts.

### 5. Commercial model

A standard one-time organization connection/setup fee is charged separately from user-account payments. It is not an advance against later monthly account charges. Recurring user billing follows active/billable account history and the full-month creation and five-day deactivation rules in the canonical policy.

No additional free-account category, branch charge, per-role charge, per-module charge, minimum paid-seat count, or monthly base fee is introduced by this approval. Exact prices, currency catalogue, and unresolved billing edge decisions remain explicit implementation prerequisites.

## Delivery and reconciliation

Read [Platform Management](../02_MODULES_KNOWLEDGE/platform_management.md) and [integration boundaries](../01_INTEGRATION_REGISTRY/platform_management_boundaries.md) before implementation.

Required order: freeze remaining boundary decisions; define ownership/contracts; prove isolated provisioning and routing with two test tenants; implement identity/licensing and subscriptions; verify payments, invoice evidence, and recovery; add self-service and operational fleet controls. Each is a separately scoped, reviewed coding task.

Existing Tenancy and Identity remain their current runtime owners. Any control-plane handoff, database split, membership expansion, or authority relocation requires a source-code inventory, compatibility plan, forward-only migration/provisioning design, tenant-isolation tests, rollback plan, and synchronized registry updates. No historical Flyway migration is changed or new migration number reserved by this ADR.

## Consequences

The accepted design requires automated database provisioning/upgrades, per-tenant routing and access protection, backup/restore verification, billing reconciliation, and a secure privileged control plane. Infrastructure cost remains even for a tenant with no paid accounts. Open questions must not be replaced with an invented minimum fee or unapproved account restriction.

Approval of this target is not evidence that any Platform Management schema, API, payment integration, role constant, lifecycle enum, or production capability exists.
