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
- **One Account = One Organization.** A customer account is immutably bound to exactly one organization and cannot hold multiple organization memberships, switch tenants, or gain parent/subsidiary access through role assignment. The same human may use the product for several organizations only through separate named accounts, credentials/sessions, organization bindings, permission/resource scopes, billing histories, and audit identities.

### 2. Common product, dedicated tenant operational data

Customers use a common ERP frontend/product codebase. Each organization receives a fresh operational database initialized from governed schema migrations and safe reference configuration, never another customer's records, credentials, or demonstration transactions.

The shared login infrastructure authenticates named accounts; it does not share credentials or identities across organizations. Authentication resolves one immutable organization binding server-side. Usernames are identifiers, never authority.

#### Accepted target username convention

Status: **ACCEPTED TARGET REQUIREMENT / RECOMMENDED TARGET DESIGN / NOT IMPLEMENTED BY THIS DOCUMENTATION CHANGE**. The earlier recommendation `<name-alias>.<system-user-code>.<organization-login-code>` (example `hasitha.u7k9q2.sce`) is superseded. The canonical human-facing format is:

`<recognizable-name>.<opaque-short-id>@<globally-unique-org-login-code>`

Example: `hasitha.7k4m@sce-lk`. The recognizable portion may be an alias such as `hasitha`, `nimal`, `maria`, or `john.smith`; it need not be a legal name and is not immutable identity. The opaque discriminator is generated for uniqueness and must not expose EPF, employee/payroll number, national ID, passport, telephone number, a database primary key, or a predictable sequential seat number. Its exact alphabet and length remain DECISION REQUIRED; the implementation must support very large platform volumes, avoid a four-digit sequential namespace, select sufficient namespace, and retry safely on collision.

The normalized canonical username is globally unique in the platform login namespace. Identity must enforce that uniqueness atomically using the same documented normalization as authentication; application-side check-then-insert, organization code, human naming, employment identifiers, or short-code probability are insufficient. On collision, generate another valid candidate without changing another account. The eventual Identity design must define case sensitivity, Unicode/case folding, allowed characters, maximum length, internationalization, enumeration resistance, and collision behavior; this KB task creates no constraint or migration.

Platform Management governs a globally unique, human-readable organization login code such as `SCE-LK` / `sce-lk`. Display name, abbreviation, and login code are not tenant identity; ordinary initials are not assumed unique. The organization UUID / `tenant_id` remains authoritative, and the final allocation algorithm is DECISION REQUIRED. The `@` form resembles email but is a username, not necessarily an email address or domain. Login UI must use `Username`, `Work ID`, or a later approved product term and keep username, contact email, display name, immutable account identity, and optional HR employee reference conceptually separate.

A username is not secret and credentials remain protected separately. The username suffix cannot grant organization, role, industry, subscription, database, or provider access. Editing a username, header, URL, browser storage, or token payload cannot alter server authority. A controlled rename preserves immutable Account ID, organization ID, ownership, sessions/security history as approved, permissions, billing/licensing/deactivation history, audit, entitlement, and database routing. Organization display-name changes do not rename identity. Any future organization-code rename, login aliases/continuity, or collision migration requires explicit Identity design. Canonical usernames should not be silently recycled to unrelated accounts; exact retention/anonymization follows applicable approved privacy/legal policy and remains DECISION REQUIRED.

Conceptually, account creation selects the organization, creates an immutable single-organization account, proposes a recognizable alias, generates an opaque discriminator, includes the governed organization login code, normalizes the candidate, atomically enforces global uniqueness, retries collisions, and completes with immutable account and tenant bindings. A later approved contract may supply the trusted account-created fact to billing; no event name is approved here. Authentication performs username lookup and credential verification, resolves immutable Account ID and the single organization binding, then applies subscription/industry entitlement, licensing, authorization, and trusted database routing. Database names and credentials are never derived from username text.

The trusted chain is `Authenticated Account -> Immutable Account ID -> Single Organization Binding -> Active Organization -> Active Subscription -> Approved Industry/Capability Entitlement -> Account Licensing Entitlement -> RBAC/ABAC/Resource Scope -> Trusted Tenant Database Binding -> Authorized Use Case`. Every gate is server-side and fail-closed.

Database-per-tenant does not imply one physical server or one backend deployment per organization. Dedicated compute topology, regional placement, capacity tiers, and routing implementation require their own execution design. The minimum accepted isolation requirement is a distinct operational database and authorized tenant-specific routing. There is no permission to fork the product code for every tenant.

Retain canonical tenant UUIDs, server-validated membership authority, resource authorization, tenant-bearing events, tenant-qualified caches/storage/jobs, database identity checks, and isolation tests. A browser-supplied organization ID or database name is not routing authority. Database-per-tenant is not a reason to remove existing tenant guards.

### 3. Provider-owned Platform Management System

A separate, secured Platform Management System is the provider's SaaS control plane. It owns the commercial organization/tenant registry, subscription lifecycle, pricing catalogue, one-time connection charges, user licensing, SaaS billing/payment evidence, provisioning orchestration, and platform operational oversight.

The control plane has its own management database and privileged provider administration boundary. Its tenant-attributed subscription and payment metadata may be managed centrally; this is not permission to pool customer ERP transactions in that database. Customers receive only their own authorized billing/administration self-service views.

The customer ERP data plane owns business operations. Customer Finance, Transport Billing, and the existing System resilience/integrity module must not become the SaaS subscription owner. Provider invoices and customer sales invoices are different business records.

The control plane may itself be implemented as a modular application. This ADR approves separation of the provider management system, not a general microservice decomposition of ERP business domains.

### 3.1 Industry and product capability entitlement

Platform Management authorizes each organization subscription's industry/product capability set. Entitlement is organization-level and is not inferred from names, codes, username suffixes, employee numbers, roles, database names, navigation visibility, or feature flags. Tenant administrators cannot self-grant an industry capability. Entitlement, activation/configuration, user authorization, resource scope, and database routing are separate decisions.

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

Existing Tenancy and Identity remain their current runtime owners. Any control-plane handoff, database split, or authority relocation requires a source-code inventory, compatibility plan, forward-only migration/provisioning design, tenant-isolation tests, rollback plan, and synchronized registry updates. Multi-membership expansion and customer tenant switching are not permitted by the accepted target. No historical Flyway migration is changed or new migration number reserved by this ADR.

Tenant ERP databases follow one governed schema release line by default. A global release is one approved migration package executed and recorded independently by each applicable tenant database, never one distributed SQL transaction. Rollout uses isolated and upgrade verification, internal/test tenants, pilots, then controlled rings with truthful partial status, pause/recovery, reconciliation, and drift detection. Schema readiness, backfill, entitlement, activation, and authorization remain separate. Business modules own migration meaning; Platform Management coordinates fleet rollout.

## Consequences

The accepted design requires automated database provisioning/upgrades, per-tenant routing and access protection, backup/restore verification, billing reconciliation, and a secure privileged control plane. Infrastructure cost remains even for a tenant with no paid accounts. Open questions must not be replaced with an invented minimum fee or unapproved account restriction.

Approval of this target is not evidence that any Platform Management schema, API, payment integration, role constant, lifecycle enum, or production capability exists.
