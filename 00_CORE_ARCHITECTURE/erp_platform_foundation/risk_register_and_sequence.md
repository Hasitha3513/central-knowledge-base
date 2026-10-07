# Risk Register & Recommended Implementation Sequence

Status: **RECOMMENDED — OWNER DECISION REQUIRED**.

## Risk register

| Risk | Consequence | Mitigation / decision gate |
| :--- | :--- | :--- |
| Distributed monolith | Chatty synchronous calls plus independent failure without autonomy | Score coupling before extraction; minimize stable contracts; keep local consistency where owned |
| Duplicated master data | Conflicting Employee/Customer/Project/Vehicle/Supplier truth | Approve ownership matrix; stable logical IDs; compatibility and reconciliation |
| Ownership ambiguity | Two modules mutate one lifecycle | One authoritative owner and explicit projections; no shared tables |
| Cross-module database access | Tenant leakage and hidden coupling | Existing prohibition, architecture tests and owner contracts |
| Microservice over-decomposition | Latency, partial failure, operational cost | D-ERP-17 scorecard and capability ADR |
| Tenant database explosion | Excess connections, backups, migrations and support | Prefer P3; measure fleet cost before further separation |
| Schema-profile explosion | Combinatorial versions and recovery paths | Common schema initially; future evidence/new ADR |
| Distributed transaction complexity | Inconsistent stock/accounting/workflow state | Local owner transactions, durable facts, idempotency, compensation/reconciliation |
| Identity duplication | Conflicting credentials, membership and revocation | Staged authority transition; no credential copying; explicit token trust |
| Inconsistent authorization | Entitlement/permission drift and privilege escalation | Separate entitlement/activation/permission/scope; backend enforcement and audit |
| Frontend fragmentation | Multiple logins, inconsistent UX/accessibility | One product shell/codebase recommendation; provider UI separate |
| Migration partial failure | Tenant version drift and unsafe activation | Staged per-DB execution, truthful status, pause/recovery and compatibility windows |
| Event contract drift | Broken consumers, replay ambiguity | Versioned registry, schema compatibility, producer/consumer tests and reconciliation |
| Reporting ownership | Metrics redefined or foreign data copied excessively | Owner-defined metrics/facts; lineage-aware read models; no foreign SQL |
| Premature generic workflow/config engine | Weak domain rules and unbounded complexity | Implement explicit domain lifecycles first; generalize only repeated proven needs |
| Transportation rewrite bias | Loss of accepted behavior and delayed ERP delivery | T1 preservation, characterization and incremental reconciliation |
| Sensitive workforce/customer replication | Privacy and compliance exposure | Minimized projections, logical references, retention/access decisions |
| Specialized-store sprawl | Unsupported infrastructure and inconsistent recovery | Evidence-driven extraction, named owner, SLO/runbooks and lifecycle cost |

## Highest-priority owner decisions

1. D-ERP-20 foundation gate and accountable decision owners.
2. D-ERP-01/02/15 deployment boundaries.
3. D-ERP-05–12 authoritative ownership and Identity transition.
4. D-ERP-03/18/19 tenant schema, persistence and fleet rollout.
5. D-ERP-13/17 integration and extraction governance.
6. D-ERP-04/14 entitlement and common customer experience.

## Recommended post-approval implementation sequence

1. Product owner answers D-ERP-01 through D-ERP-20; record accepted ADRs without changing historical evidence.
2. Freeze the enterprise domain map, ownership matrix and tenant/Identity authorities.
3. Define contract standards, security classifications, consistency/reconciliation rules and acceptance gates.
4. Prove Platform Management-to-customer-runtime tenant/entitlement boundary with isolated non-production organizations.
5. Establish staged Identity compatibility and common frontend shell boundaries.
6. Build the modular ERP core foundation with tenant isolation and no business-domain implementation shortcuts.
7. Implement one core domain at a time in dependency order, starting only after its ownership and upstream contracts are approved.
8. Reconcile Transportation overlaps through characterization, stable mappings and reversible cutovers; never rewrite by default.
9. Introduce read models/reporting from owner-published facts with lineage and privacy controls.
10. Evaluate specialized extraction only through the D-ERP-17 scorecard and a capability-specific ADR.
11. Measure tenant schema fleet signals and revisit optimization only through D-ERP-19.

This sequence is planning guidance, not authorization to implement Finance, HRM, Inventory, Procurement, Projects, Sales/CRM, Maintenance, Identity migration, or deployment changes.
