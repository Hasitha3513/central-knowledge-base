# Platform Management System

Lifecycle: ACCEPTED_REQUIREMENTS / IMPLEMENTATION_NOT_STARTED_BY_THIS_CHANGE
Scope: Provider-side SaaS control plane for the enterprise ERP suite.
Requirements approval recorded: 2026-10-06
Source: Product owner acceptance and explicit account-billing/deactivation clarification.

## Governing documents

- [Organization tenancy and control-plane ADR](../00_CORE_ARCHITECTURE/ADR-SAAS-PLATFORM-MANAGEMENT-001.md)
- [Canonical subscription and billing policy](../00_CORE_ARCHITECTURE/saas_subscription_billing_policy.md)
- [Target integration boundaries](../01_INTEGRATION_REGISTRY/platform_management_boundaries.md)
- [Current and target multi-tenancy standards](../00_CORE_ARCHITECTURE/multi_tenancy_standards.md)
- [Current runtime RBAC registry](../00_CORE_ARCHITECTURE/rbac_and_permissions.md)
- [Target operator guide](../03_USER_MANUALS/platform_management_guide.md)

## Mission and bounded context

Manage the provider's SaaS business: organization onboarding, independent subscriptions, one-time connection fees, monthly account licensing/billing, the one complimentary Super Admin entitlement per organization, payment evidence, dedicated-database provisioning, and platform operations.

This is separate from the eight customer business domains listed in `MASTER_AGENTS.md`. It does not turn the Transport `system` module into a subscription manager, implement the proposed Finance domain, or change Transport MVP completion/activation accounting.

### Organization boundary

Each organization, including a holding parent and every subsidiary, has a separate subscription and operational database. Departments, branches, and sections are tenant-internal structures and do not create subscriptions. A common ERP frontend serves the organizations; a dedicated operational database does not itself promise a dedicated server/backend process.

One Account = One Organization. Customer accounts cannot hold several organization memberships, switch tenants, share a login across organizations, or inherit parent/subsidiary access. A human serving several organizations uses a separate named account, credential/session, binding, licensing record, permissions, scope, and audit trail for each.

The accepted target username convention is `<recognizable-name>.<opaque-short-id>@<globally-unique-org-login-code>` (example `hasitha.7k4m@sce-lk`). Identity owns normalized global username uniqueness and immutable Account ID; Platform Management governs organization login-code allocation. Username text is never tenant, entitlement, role, database-routing, or billing authority. This is recommended target design, not verified implementation.

One designated organization Super Admin account is complimentary. All other tenant accounts are paid, including delegated Admins. Super Admin delegates administration; Admin configures permitted access; Super Admin approval is mandatory for commercial account deactivation. Provider platform administrators are a separate security boundary.

## Phase 1: Current MVP Scope (Active Implementation)

Status within this section: APPROVED_DELIVERY_REQUIREMENTS / NO_CODING_STARTED_BY_THIS_KB_TASK.

The scope below is accepted for controlled follow-on implementation; each code change still needs an explicitly scoped task and verified contracts. This header must not be interpreted as evidence of running Platform Management software.

| Capability | Minimum accepted outcome |
| :--- | :--- |
| Organization/tenant registry | Independent parent/subsidiary registration and subscriptions; tenant identity and group links; internal units do not create billable tenants. |
| Product/pricing catalogue | Standard connection fee and monthly account prices with currency/effective version; no invented amounts or extra module/branch charges. |
| Subscription management | Enrollment, initial-payment evidence, ongoing service entitlement, fifteen-calendar-day per-invoice grace, two-consecutive-overdue payment suspension, narrow recovery access, full-settlement restoration, and one-calendar-month audited provider override per approval. |
| Account licensing | One complimentary Super Admin per organization; all other accounts paid; identity-sourced historical account facts; no admin-role exemptions or per-branch duplicates. |
| Monthly billing | Full same-month fee for account creation on any date; original bill anchor; five-day approved/effective deactivation handling; late-month charge attribution and auditable adjustments. Resolve policy edge decisions before affected automation. |
| Invoices/payments | Stable month-close snapshot, detailed invoice and historical Active User Billing Report; separate connection/account lines; immutable policy/price/due evidence; full-settlement verification, idempotency, delinquency and reconciliation. No provider has been selected. |
| Tenant provisioning | Fresh dedicated database, approved migrations/reference data, secure credentials/routing, readiness checks, repeat-safe recovery and no copied customer/demo data. |
| Industry/product entitlement | Provider-approved organization capability set with effective/version evidence and fail-closed enforcement; never inferred or tenant-admin self-granted. |
| Release & tenant upgrade management | One governed schema line; approved manifests; per-database version/migration/backfill/verification; rollout rings; pause/retry/recovery, backup/restore and drift evidence. Modules retain migration semantics. |
| Platform security/support | Provider staff access, least privilege, audit trail, explicit support authorization, last-owner protection, and separation of security locks from commercial deactivation. |
| Tenant self-service | Own account counts, invoices/payment state, bill-generation/deactivation deadline and approval status; authorized administration and narrow payment/account-recovery access. |
| Operations/business dashboard | Organization/subscription counts, billable accounts, connection-fee collections, recurring billed amounts, outstanding invoices, provisioning failures, schema/backup health and operating-cost visibility. Monitoring is not permission to add usage fees. |

### Mandatory business invariants

The canonical billing policy is authoritative; do not maintain an inconsistent second algorithm here.

- The complimentary entitlement is unique per organization and transferable only through a protected, auditable ownership workflow.
- Role/permission delegation never creates a free account or a cross-tenant authority.
- Non-complimentary account creation creates a full charge in its own billing month without day-based proration.
- Both authorized Super Admin approval and effective deactivation must occur within the five-day post-generation window for the eligible current-bill adjustment. Request-only or late/failed execution retains the charge.
- A late approved deactivation may still stop access; it does not erase the charge already retained for that month.
- Bill generation, month attribution, effective account history, and price/policy versions are evidenced, not inferred from UI state or mutable current counts.
- Connection fees are separate from monthly account payments and are not advanced credit.
- Five-calendar-day deactivation adjustment and fifteen-calendar-day invoice grace are independent windows anchored to invoice generation; business-day and elapsed-hour reinterpretations are prohibited.
- Month-close billing evidence and Active User Billing Reports use authoritative account-month history and exclude authentication secrets, HR identifiers, government identifiers, database credentials, and unrelated personal data.
- Partial payment never settles an invoice, clears the common-shell overdue alert, resets delinquency, prevents an earned suspension, or restores service.
- Automatic payment suspension requires two consecutive monthly invoices to exhaust their own grace periods without full settlement; one overdue invoice is insufficient.
- One-calendar-month provider overrides are privileged, audited, non-renewing approvals that temporarily allow service but preserve debt, alert, delinquency, billing and reporting.
- No base monthly fee, extra free-account class, paid-seat minimum, module fee, or country-specific legal/accounting rule is silently added.

### Data ownership

The Platform Management database owns provider commercial records: tenant/subscription metadata, pricing, complimentary/paid entitlement evidence, account-month billing records, invoices/corrections, payment references, provisioning operations, and provider audit/health metadata.

Customer operational databases own their ERP facts. Identity owns login/membership/permission authority. Do not query foreign tables, copy authentication secrets into billing records, or use HR employee counts as billable login-account counts. Exact schemas and authority handoff are unimplemented and require design/reconciliation.

### Conceptual use cases and sequence

Register organization; verify owner; calculate connection and initial account charges; confirm payment server-side; provision/initialize dedicated database; verify readiness; establish the one free Super Admin entitlement; activate authorized paid accounts; grant/revoke delegated administration; generate monthly bill; request/approve/execute deactivation; reconcile bill consequences; monitor and recover provisioning/payment failures.

These are capability descriptions, not invented published Java signatures or REST endpoints. A failed payment or provisioning step must leave truthful, retryable evidence. Retries cannot duplicate the connection charge, account fee, tenant database, or complimentary entitlement.

Customer feature requests are classified as existing capability, configuration gap, UX gap, defect, reusable product gap, or out of scope. Generalize the real business rule, not a customer legacy screen. Reuse, configure, document, improve common UX, fix defects, or implement a reusable gap once in its owning context. Tenant-specific forks, `if tenant == X` branches, copied private data, and speculative generic engines/tables/APIs/events are prohibited.

One configurable customer frontend adapts to entitlement, activation/configuration, permissions, resource scope, and approved localization. Hidden UI is not authorization; backend use cases enforce the same gates. A separate provider interface remains a distinct privileged security boundary.

Tenant databases normally share one governed release line. Not every feature needs a migration. Required migrations are forward-only and module-owned; breaking change uses reviewed expand, migrate/backfill, switch, and contract techniques. Platform Management stages one approved package per database through verification, pilots, and rings, recording truthful independent outcomes rather than implying a fleet-wide transaction.

### Verified schema, ports, events and permissions

No Platform Management schema, migration number, table dictionary, endpoint, event/topic, permission constant, or production deployment has been verified or introduced by this documentation change. The current Tenancy/Identity authorities and existing System/Integration/Billing contracts are not silently moved or reused.

### Acceptance and readiness gates

All policy scenarios T01-T32 are required. Additional future gates cover isolated two-organization routing, group-access denial, approved unit scoping, provider-versus-tenant administration, tenant restore verification, pricing/version evidence, payment callback security, and partial-failure recovery. Tests were NOT RUN by this KB-only task.

Outstanding policy decisions D-01 through D-07 remain visible in the canonical policy. Do not implement an inferred resolution. Database split/backfill/rollback, deployment topology, identity handoff, and precise contract shapes need scoped implementation designs.

## Phase 2: Post-MVP / Future Roadmap (Deferred Scope)

Advanced cross-organization consolidation reporting, negotiated enterprise arrangements, automatic tax/localization packs, marketplace commerce workflows, additional payment providers, dedicated-compute tiers, and advanced analytics remain separately scoped work. Cross-organization reporting never creates multi-organization customer identities or automatic parent/subsidiary access. No candidate database tables or unapproved API/event families are created for them here.

Design must reconcile approved KB/ADRs, verified implementation, owner rules, usability/accessibility, security/privacy, applicable law, jurisdiction/industry compliance, operational practicality, scalability, and maintainability. Do not claim generic worldwide compliance; variable requirements need authoritative verification and controlled localization.

An industry/business-model label alone does not prove its workflow is implemented or compliant in every jurisdiction. Deferred work never changes the approved organization boundary, one complimentary account, full-month creation rule, or five-day deactivation policy without a versioned product decision.

## Controlled implementation sequence

1. Resolve applicable boundary decisions and inventory actual Tenancy/Identity/System contracts.
2. Define and approve the minimal control-plane security, organization/subscription ownership, and provisioning/routing contracts.
3. Prove isolated databases and recoverable onboarding with two test organizations.
4. Implement licensing, Super Admin delegation/approval, and immutable account-month billing evidence.
5. Integrate payment verification, original-bill deadlines, adjustments, and reconciliation.
6. Add tenant self-service and platform operational/business dashboards.
7. Independently verify security, billing scenarios, migration/recovery, and documentation before any production activation.

This sequence is a backlog outline, not authorization to implement the entire platform in one coding session.
