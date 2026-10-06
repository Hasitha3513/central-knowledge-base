# Platform Management Integration Boundaries

Status: ACCEPTED_TARGET_BOUNDARIES / NO_IMPLEMENTED_CONTRACTS_ADDED
Date: 2026-10-06
Authority: [ADR-SAAS-PLATFORM-MANAGEMENT-001](../00_CORE_ARCHITECTURE/ADR-SAAS-PLATFORM-MANAGEMENT-001.md).
Commercial source: [SaaS subscription and billing policy](../00_CORE_ARCHITECTURE/saas_subscription_billing_policy.md).

This is a target integration/ownership supplement. Existing `api_interfaces.md`, `event_contracts.md`, `cross_module_dependency_map.md`, and `state_machines.md` continue to document their current contracts. No new HTTP route, Java signature, event class/topic, table, enum, or active dependency is introduced here. Exact implementation contracts must be registered in those canonical registries before use.

## Target ownership and information flow

| Provider / authority | Consumer | Minimum intended information or action | Boundary and status |
| :--- | :--- | :--- | :--- |
| Identity | Platform licensing/billing | Trusted tenant membership/account creation, activation, deactivation, identity/version, and effective times | Minimized published contract plus reconciliation; ACCEPTED_TARGET, NOT_IMPLEMENTED. Never inspect foreign Identity tables. |
| Organization Super Admin authorization through Identity | Account deactivation workflow | Same-organization approval identity, request/target, authority at approval/execution, and effective result | Server-side permission/resource checks; ordinary Admin approval is insufficient. Exact contract pending. |
| Platform licensing | ERP access enforcement | Current tenant/account entitlement and the one designated complimentary entitlement | Trusted versioned decision, not browser account-count input or an admin-role-name exemption. Exact contract pending. |
| Platform subscriptions and provisioning | Tenant runtime/router | Authorized tenant identity, readiness and service entitlement, opaque routing/credential references | Validated target routing; no plaintext database credentials in browser responses. Existing Tenancy authority must be reconciled explicitly. |
| Payment provider adapter | Platform payments/subscriptions | Verified payment identity, amount/currency, status and correlation | Authenticated server-side confirmation, idempotency and ordering/reconciliation; provider not selected by this document. |
| Platform invoicing | Organization billing self-service | Own invoices, account quantities, policy/price versions, connection charge, corrections and deadline evidence | Tenant-qualified read/payment-recovery actions only; no access to other tenants or provider-only operations. |
| Provisioning/operations adapters | Platform management | Database creation, schema version, readiness, backup/restore and health outcomes | Privileged bounded jobs; repeat-safe operations, secret references, explicit failure evidence. |
| Tenant ERP domains | Group reporting, if later implemented | Explicitly authorized minimized business facts | No automatic holding-company access, direct cross-database joins, or pooled tenant ownership. Exact feature contracts remain deferred. |

## Authority boundaries

Platform Management owns SaaS commercial records, not customer ERP ledgers, customers, inventory, employees, trips, or operational documents. Tenant-attributed commercial records in the provider database retain appropriate tenant scope; provider-wide catalogue/configuration is explicitly provider-owned, not implicitly global customer data.

Identity remains authoritative for account/membership status and permissions. HRM employee lifecycle is not the paid login-account authority. Role changes and internal-unit assignments must not manufacture account creation/deactivation facts or duplicate fees.

The existing `system` module remains resilience/integrity focused. The existing `integration` and `billing` modules are not automatically owners of payment collection or SaaS invoices. Reuse of an existing published technical capability requires explicit scope/compatibility review.

## Reliability and security invariants

- Separate payment success, provisioning readiness, commercial activation, user authorization, and bill correction outcomes. No distributed all-or-nothing guarantee is inferred from one local transaction.
- Cross-boundary facts carry trusted tenant/account identity, stable operation/event identity, schema version, effective time, and correlation as appropriate. Exact payloads remain to be approved.
- Replayed/out-of-order account or payment facts cannot duplicate a full-month charge, reset a five-day window, create two complimentary entitlements, or activate the wrong tenant.
- Billing uses account-month history and the original bill-generation anchor. Request time alone does not prove approved/effective deactivation.
- Failures and delays remain visible; never backdate execution or convert an unverified request into successful deactivation.
- Security access blocking is independent of the approved commercial deactivation outcome. Do not delay incident containment to preserve a billing workflow.
- Provisioning uses fresh schema/reference configuration, not copied customer data. Tenant database identity/credentials must be checked on routing and connection reuse; shared worker/cache/storage resources remain tenant-scoped.
- Provider administration requires a separate privileged boundary. Support access must be specifically authorized, scoped, time-bounded where applicable, and audited; no default unrestricted customer-data access.

## Implementation registration checklist

For each approved coding slice: identify actual module owners and current source contracts; resolve applicable billing decisions; specify exact API/port/event/error/version/idempotency semantics; approve any new schema with a full data dictionary; register permission and lifecycle changes; update provider/consumer documents and the canonical registries; implement and test the smallest vertical slice.

Required verification includes two-tenant isolation, privilege escalation denial, account/billing replay, concurrent complimentary-entitlement transfer, approved/late/failed deactivation, same-month late account creation, payment/provisioning partial failure, and tenant-specific database restore/routing. These are future test requirements, not executed test evidence.
