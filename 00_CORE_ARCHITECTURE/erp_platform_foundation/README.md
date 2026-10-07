# ERP Platform Domain Ownership & Integration Foundation

Status: **DECISION PREPARATION ONLY / NO IMPLEMENTATION AUTHORIZED**

## Purpose

This pack prepares product-owner review of enterprise domain ownership, deployment, persistence, identity, integration, consistency, frontend, migration, and service-extraction choices before implementation of the remaining core ERP systems. It creates no schema, migration, API, event, permission, class, deployment manifest, or production capability.

## Status vocabulary

- **CURRENT VERIFIED IMPLEMENTATION** — evidenced by current KB implementation/acceptance records.
- **ACCEPTED EXISTING DECISION** — already approved by an ADR or canonical policy; this pack does not reopen it.
- **RECOMMENDED — OWNER DECISION REQUIRED** — architecture recommendation only.
- **DECISION REQUIRED** — product-owner choice is still missing.
- **DEFERRED** — deliberately outside the current foundation decision.
- **NOT APPLICABLE** — no impact for the assessed boundary.

## Accepted constraints preserved

- **ACCEPTED EXISTING DECISION:** domain-first hexagonal architecture, tenant isolation, logical cross-module references, no foreign repository/table access, and registered contracts are mandatory.
- **ACCEPTED EXISTING DECISION:** one subscribing organization is one tenant/subscription boundary with a dedicated operational tenant database in the accepted target architecture. Parent and subsidiaries subscribe separately; internal units do not.
- **ACCEPTED EXISTING DECISION:** one customer account belongs to one organization; no tenant switching or multi-organization membership.
- **ACCEPTED EXISTING DECISION:** Platform Management is the provider SaaS owner; customer Finance and Transport Billing remain separate.
- **ACCEPTED EXISTING DECISION:** all existing SaaS commercial rules in `../saas_subscription_billing_policy.md` remain unchanged.
- **CURRENT VERIFIED IMPLEMENTATION:** Transportation is a mature modular application with implemented Fleet, Driver, Route, Trip, Fuel, Freight, Delivery, Notification, Offline Sync, Tracking, Integration, Billing, Operations, Compliance, Reporting and System capabilities. It is not disposable prototype code.

## Undecided foundation recommendation

**RECOMMENDED — OWNER DECISION REQUIRED:** logical modularity first; physical extraction only when workload, availability, security, deployment, ownership, and operational evidence outweigh distribution cost. The candidate target is a modular ERP core with selective service extraction, while preserving Transportation as an independently deployable domain application integrated through explicit contracts.

## Pack index

1. [Canonical decision register](decision_register.md) — exactly D-ERP-01 through D-ERP-20.
2. [Enterprise domain map](enterprise_domain_map.md) — responsibilities and per-domain assessments.
3. [Current Transportation inventory](current_transportation_inventory.md) — verified bounded contexts, ownership, dependencies and infrastructure.
4. [Current-to-target ownership matrix](ownership_matrix.md) — enterprise concept authority and reconciliation.
5. [Transportation reconciliation matrix](transportation_reconciliation_matrix.md) — overlaps with future domains.
6. [Integration and consistency matrix](integration_consistency_matrix.md) — conceptual interaction patterns, not executable contracts.
7. [Deployment and service extraction](deployment_service_extraction.md) — runtime alternatives, scorecard, specialized candidates, frontend and Identity transition.
8. [Tenant persistence and schema strategy](tenant_persistence_schema_strategy.md) — schema/topology alternatives, empty-table explanation, rollout and review signals.
9. [Risk register and recommended sequence](risk_register_and_sequence.md) — risks and post-decision delivery order.

## Implementation gate

All 20 decisions remain **DECISION REQUIRED**. Approval must be explicit and followed by scoped ADRs and contract work. Recommendations in this pack are not implementation authority and do not change Transportation MVP accounting, acceptance, release evidence, production activation, or current registries.


## Consistency-audit classification

- **Existing approved rules, not conflicts:** modular-monolith/hexagonal direction, tenant isolation, foreign-persistence prohibitions, dedicated organization database target, one-account-one-organization, provider/customer boundary, Transport Billing versus Finance, and Tracking ownership.
- **Historical/current evidence, not target approval:** Transportation currently owns Customer, Project, Driver/workforce-adjacent facts and maintenance schedules because it predates the proposed enterprise domains.
- **Recommendations, not contradictions:** future blueprint statements assigning HRM, Finance, Inventory, Procurement, Projects, Sales/CRM and Maintenance authority. They are reconciled through D-ERP-05 through D-ERP-11 and remain unapproved.
- **Genuine decisions requiring resolution:** Identity handoff, Customer/Project/Employee/Supplier/Maintenance migrations, core/Transportation deployment, schema strategy, entitlement model and exact cross-domain consistency contracts.
- **No silent rewrite:** existing acceptance, story accounting, migration history, current APIs/events and operational evidence remain unchanged.
