# Canonical ERP Platform Owner-Decision Register

Status: **DECISION PREPARATION ONLY**

This register contains exactly D-ERP-01 through D-ERP-20. Recommendations are not approvals. Every decision remains **DECISION REQUIRED**.

## D-ERP-01 — Core Deployment Strategy

- **Problem / question:** Choose modular core, microservice-per-domain, or modular core with selective extraction.
- **Existing verified context:** Core rules mandate modular-monolith/hexagonal boundaries; future core domains are proposed.
- **Available options:** A modular core; B domain microservices; C modular core plus selective extraction.
- **Recommended option:** C — Modular Core + Selective Service Extraction
- **Recommendation rationale:** Keeps local consistency and delivery simplicity while retaining evidence-driven extraction paths.
- **Advantages:** Lower latency and operational cost; cohesive transactions; evolutionary boundaries.
- **Risks / disadvantages:** A poorly enforced core can become coupled; later extraction still costs migration.
- **Impact on existing Transportation:** Preserves Transportation as a deployment variable and contract peer.
- **Impact on Platform Management:** Remains separate provider boundary under D-ERP-15.
- **Impact on tenant databases:** Compatible with one primary tenant ERP database plus justified specialized stores.
- **Impact on frontend:** One product experience can span deployments.
- **Security impact:** Fewer network trust boundaries initially; module authorization still mandatory.
- **Scalability/operations impact:** Avoids premature fleet complexity; requires architecture enforcement and observability.
- **Dependencies:** D-ERP-02, 13, 16, 17, 20
- **Questions the product owner must answer:** Approve C? Which measurable constraints would require earlier extraction?
- **Current status:** `DECISION REQUIRED`

## D-ERP-02 — Existing Transportation Position

- **Problem / question:** Decide whether Transportation remains independently deployable, merges into ERP Core, or is decomposed first.
- **Existing verified context:** Transportation is a substantial Spring Modulith application with accepted capabilities and specialized Tracking infrastructure.
- **Available options:** T1 preserve and integrate; T2 merge runtime; T3 decompose before integration.
- **Recommended option:** T1 — Preserve Transportation and integrate through explicit contracts
- **Recommendation rationale:** Protects working investment and limits high-risk rewrite while ownership is reconciled.
- **Advantages:** Compatibility-first; independent release/failure boundary; incremental migration.
- **Risks / disadvantages:** Network contracts and temporary duplicated masters require disciplined reconciliation.
- **Impact on existing Transportation:** Direct: Transportation remains intact; no rewrite or premature split.
- **Impact on Platform Management:** Consumes control-plane entitlement/tenant decisions through future approved contracts.
- **Impact on tenant databases:** Transportation and core persistence remain independently governed; no cross-database SQL.
- **Impact on frontend:** Common customer frontend may hide backend boundary.
- **Security impact:** Requires federated identity/authorization and tenant-qualified contracts.
- **Scalability/operations impact:** Two application operations and observability surfaces; avoids immediate decomposition cost.
- **Dependencies:** D-ERP-06–08, 12–14, 20
- **Questions the product owner must answer:** Approve T1? What conditions would later justify merge or extraction?
- **Current status:** `DECISION REQUIRED`

## D-ERP-03 — Tenant Database Schema Strategy

- **Problem / question:** Choose common full schema, subscribed-module schemas, or governed profiles.
- **Existing verified context:** Dedicated organization databases are accepted target architecture; capability mix varies.
- **Available options:** A common governed schema; B subscribed-only schemas; C governed profiles/module migration sets.
- **Recommended option:** A initially; review C only with evidence
- **Recommendation rationale:** Minimizes divergence, activation migrations, rollback paths, support variants and test combinations.
- **Advantages:** Uniform rollout, simpler support and later capability activation.
- **Risks / disadvantages:** Catalog/migration/backup overhead multiplies with tenant count even for empty tables.
- **Impact on existing Transportation:** No immediate Transportation schema change.
- **Impact on Platform Management:** Must orchestrate truthful per-database rollout and entitlement separately.
- **Impact on tenant databases:** Common schema per relevant tenant database; specialized stores remain separately justified.
- **Impact on frontend:** No direct impact; hidden capability is not authorization.
- **Security impact:** Uniform patching aids security; unused tables remain tenant-isolated.
- **Scalability/operations impact:** Simpler initially; measure fleet overhead before optimization.
- **Dependencies:** D-ERP-18, 19
- **Questions the product owner must answer:** Approve initial A? What measured evidence triggers profile evaluation?
- **Current status:** `DECISION REQUIRED`

## D-ERP-04 — Industry vs Capability Subscription

- **Problem / question:** Separate industry classification/defaults from actual capability entitlement.
- **Existing verified context:** Platform ADR already separates entitlement from user authorization; final bundle model is not approved.
- **Available options:** Hard-coded industry modules; industry default bundle plus explicit entitlements; entitlements only.
- **Recommended option:** Industry classification/default bundle plus explicit capability entitlements
- **Recommendation rationale:** Supports industry defaults without hard-coded exclusion; entitlement remains contractual authority.
- **Advantages:** Flexible cross-industry product; auditable exceptions and additions.
- **Risks / disadvantages:** Catalogue governance and dependency validation are required.
- **Impact on existing Transportation:** Transportation access becomes an entitlement, not an industry inference.
- **Impact on Platform Management:** Owns classification and organization capability contract.
- **Impact on tenant databases:** Schema presence remains independent from access entitlement.
- **Impact on frontend:** Visibility requires entitlement + activation + permission + scope + localization.
- **Security impact:** Prevents role or client flags from granting unsubscribed capability.
- **Scalability/operations impact:** Requires a versioned catalogue, dependency validation and entitlement observability; does not require per-industry deployments.
- **Dependencies:** D-ERP-03, 14, 15
- **Questions the product owner must answer:** Approve the four-layer distinction and who approves exceptions?
- **Current status:** `DECISION REQUIRED`

## D-ERP-05 — Finance Ownership

- **Problem / question:** Confirm Finance authority for GL/AP/AR/payment accounting and financial reporting.
- **Existing verified context:** Finance blueprint is PROPOSED; accepted US-47 boundary keeps operational transport charges outside Finance.
- **Available options:** Finance owns accounting consequences; operational domains own source facts; or duplicate accounting per domain.
- **Recommended option:** Finance owns formal accounting; source domains retain operational facts
- **Recommendation rationale:** Prevents duplicate ledgers and preserves bounded source meaning.
- **Advantages:** One fiscal authority; traceable postings; clear segregation from operational charge calculation.
- **Risks / disadvantages:** Requires robust reconciliation and idempotent integration.
- **Impact on existing Transportation:** Transport Billing remains charge authority; exports facts, never marks accounting paid.
- **Impact on Platform Management:** Provider SaaS billing remains outside customer Finance.
- **Impact on tenant databases:** Finance tables remain Finance-owned within chosen tenant topology.
- **Impact on frontend:** Unified financial views consume Finance read models.
- **Security impact:** Strong SoD, period locks, privacy and audit required.
- **Scalability/operations impact:** Accounting workload may scale separately later; distributed posting needs operations maturity.
- **Dependencies:** D-ERP-01, 13; D-ERP-08–11
- **Questions the product owner must answer:** Confirm boundary, accounting basis, jurisdictions and acknowledgement semantics?
- **Current status:** `DECISION REQUIRED`

## D-ERP-06 — HRM vs Transportation Driver Ownership

- **Problem / question:** Resolve Employee/Employment versus operational Driver authority.
- **Existing verified context:** Transportation currently owns Driver, licence, exception, violation, medical and drug-test records; HRM is PROPOSED.
- **Available options:** Keep all in Transportation; move all to HRM; split Employee/Employment from Driver operational profile.
- **Recommended option:** Split: HRM Employee/Employment; Transportation Driver operational eligibility
- **Recommendation rationale:** Separates workforce lifecycle from safety-critical transport execution without deleting current value.
- **Advantages:** Clear lifecycle authority; Transportation retains dispatch-specific decisions.
- **Risks / disadvantages:** Identity mapping, privacy, availability timing and historical API compatibility are difficult.
- **Impact on existing Transportation:** Requires stable logical Employee reference, compatibility bridge and staged migration.
- **Impact on Platform Management:** No direct commercial ownership; Account remains distinct from Employee.
- **Impact on tenant databases:** No cross-domain foreign keys; migration mapping must be tenant-qualified.
- **Impact on frontend:** Workforce and driver screens can remain one product experience.
- **Security impact:** Medical/disciplinary/qualification facts need least privilege and minimization.
- **Scalability/operations impact:** Synchronous dispatch eligibility may require high availability; change facts need reconciliation.
- **Dependencies:** D-ERP-12, 13, 20
- **Questions the product owner must answer:** Which current Driver records are workforce versus transport safety authority?
- **Current status:** `DECISION REQUIRED`

## D-ERP-07 — Vehicle Maintenance Ownership

- **Problem / question:** Resolve Fleet Vehicle master, Maintenance lifecycle, and Transportation availability consumption.
- **Existing verified context:** Transportation Fleet owns Vehicle and maintenance_schedule; Maintenance blueprint is PROPOSED.
- **Available options:** Keep schedule in Fleet; move all vehicle state to Maintenance; split master/operational maintenance/eligibility projection.
- **Recommended option:** Fleet owns Vehicle; Maintenance owns work lifecycle; Transportation consumes eligibility
- **Recommendation rationale:** Aligns identity, work execution and dispatch decisions without duplicating vehicle master.
- **Advantages:** Focused ownership and replaceable maintenance workflow.
- **Risks / disadvantages:** Safety hold timing and current assignment transaction semantics need careful compatibility.
- **Impact on existing Transportation:** Preserve Vehicle IDs/readings; bridge current schedule blocking before migration.
- **Impact on Platform Management:** No direct impact beyond entitlement and provisioning.
- **Impact on tenant databases:** Maintenance stores logical Vehicle references; no Fleet table access.
- **Impact on frontend:** Unified vehicle experience can compose owner projections.
- **Security impact:** Safety holds, override and audit require strict authorization.
- **Scalability/operations impact:** Maintenance may later justify separate workload only with evidence.
- **Dependencies:** D-ERP-09, 13, 20
- **Questions the product owner must answer:** Who owns final return-to-service and synchronous dispatch eligibility SLA?
- **Current status:** `DECISION REQUIRED`

## D-ERP-08 — Customer Ownership

- **Problem / question:** Resolve canonical customer/business-account relationship ownership.
- **Existing verified context:** Organization currently owns customer/contact facts used by Freight/Trip/Delivery; Sales/CRM is PROPOSED.
- **Available options:** Keep Organization owner; move canonical relationship to Sales/CRM; split platform Organization from CRM Account and transport projection.
- **Recommended option:** Sales/CRM canonical relationship; Transportation keeps transport projection/reference
- **Recommendation rationale:** Avoids duplicate customer lifecycle while preserving transport-specific facts and current contracts.
- **Advantages:** Single relationship history; reusable sales/order/customer service boundary.
- **Risks / disadvantages:** ID matching, consent, active-status semantics and current API compatibility are risky.
- **Impact on existing Transportation:** Requires stable ID mapping/compatibility API; Delivery token/contact boundaries stay intact until approved.
- **Impact on Platform Management:** Customer organization is not SaaS subscribing organization; boundaries must remain explicit.
- **Impact on tenant databases:** Logical Customer ID only across domains.
- **Impact on frontend:** Customer journey can remain unified.
- **Security impact:** PII/consent and tenant isolation require strict projection minimization.
- **Scalability/operations impact:** Customer lookup availability and migration reconciliation are operational dependencies.
- **Dependencies:** D-ERP-05, 10, 13
- **Questions the product owner must answer:** Approve Sales/CRM owner? How are existing Organization Customer IDs preserved?
- **Current status:** `DECISION REQUIRED`

## D-ERP-09 — Inventory Ownership

- **Problem / question:** Confirm Item/Warehouse/Stock/Movement/Reservation authority.
- **Existing verified context:** Inventory blueprint is PROPOSED; Fuel bunker stock is an existing specialized Transportation ledger.
- **Available options:** Enterprise Inventory owns general stock; domains own all stock; split general inventory from specialized operational ledgers.
- **Recommended option:** Inventory owns enterprise items/warehouses/stock/reservation; specialized Fuel ledger remains explicit exception
- **Recommendation rationale:** Creates one general stock authority without misclassifying bunker telemetry/ledger semantics.
- **Advantages:** Consistent availability and movement audit; reusable maintenance/procurement/project contracts.
- **Risks / disadvantages:** Three-way workflows and specialized consumables require boundaries.
- **Impact on existing Transportation:** Transportation consumes reservations/issues; current Fuel facts are not silently migrated.
- **Impact on Platform Management:** No direct control-plane ownership.
- **Impact on tenant databases:** Inventory-owned tenant tables; consumers use logical references/contracts.
- **Impact on frontend:** Common stock views can compose domain demand.
- **Security impact:** Stock adjustment/approval and warehouse scope require ABAC.
- **Scalability/operations impact:** High transaction volume may need tuning but not automatic service extraction.
- **Dependencies:** D-ERP-07, 10, 13
- **Questions the product owner must answer:** Confirm specialized Fuel exception and negative-stock/valuation policies?
- **Current status:** `DECISION REQUIRED`

## D-ERP-10 — Procurement Ownership

- **Problem / question:** Confirm requisition, sourcing, vendor selection and PO lifecycle authority.
- **Existing verified context:** Procurement blueprint is PROPOSED; Transportation currently has vendor/fuel-purchase concepts.
- **Available options:** Central Procurement; domain-specific purchasing; split enterprise PO from operational Fuel purchase.
- **Recommended option:** Procurement owns enterprise purchasing; reconcile Fuel-specific purchasing explicitly
- **Recommendation rationale:** Prevents duplicate PO lifecycle while preserving implemented fuel operations until migration is approved.
- **Advantages:** Unified approvals/suppliers/contracts and downstream receipt matching.
- **Risks / disadvantages:** Supplier-master and fuel-provider boundaries remain unresolved.
- **Impact on existing Transportation:** Compatibility protects Fuel APIs; no immediate migration.
- **Impact on Platform Management:** No provider SaaS billing ownership.
- **Impact on tenant databases:** Procurement tables tenant-owned; Inventory/Finance references logical.
- **Impact on frontend:** One purchasing experience can route specialized demand.
- **Security impact:** Approval/SoD and supplier data privacy required.
- **Scalability/operations impact:** Workflow complexity must not trigger premature generic engine.
- **Dependencies:** D-ERP-05, 09, 13
- **Questions the product owner must answer:** Who owns Supplier master and three-way match; how does Fuel purchasing reconcile?
- **Current status:** `DECISION REQUIRED`

## D-ERP-11 — Project Ownership

- **Problem / question:** Confirm Project master/work/budget-intent authority.
- **Existing verified context:** Transportation owns legacy project rows and Trip references; Project blueprint is PROPOSED.
- **Available options:** Transportation retains projects; Project Management owns master; shared generic project table.
- **Recommended option:** Project Management owns Project; Transportation retains logical project references
- **Recommendation rationale:** Removes duplicate master while Finance retains actual accounting.
- **Advantages:** Reusable project validation and planning across domains.
- **Risks / disadvantages:** UUID mapping, legacy foreign constraints and availability of validation are migration risks.
- **Impact on existing Transportation:** Trips preserve references through compatibility mapping/port; no history rewrite.
- **Impact on Platform Management:** No direct control-plane ownership.
- **Impact on tenant databases:** Project tables owned by Projects; Transport uses logical UUID after reconciliation.
- **Impact on frontend:** Unified project and transport views through composition.
- **Security impact:** Project access/resource scope must be enforced centrally and in domains.
- **Scalability/operations impact:** Synchronous validation requires availability strategy and cached/read-model decisions.
- **Dependencies:** D-ERP-05, 13, 20
- **Questions the product owner must answer:** Approve owner and compatibility duration; who owns project budget intent versus control?
- **Current status:** `DECISION REQUIRED`

## D-ERP-12 — Platform Identity

- **Problem / question:** Choose long-term shared Identity boundary and staged Transport handoff.
- **Existing verified context:** Transportation currently has authoritative accounts, tenant membership, roles/permissions and JWT; accepted target is one-account-one-organization.
- **Available options:** Retain per-app identities; big-bang shared Identity; staged shared Platform Identity transition.
- **Recommended option:** Staged shared Platform Identity; no big-bang rewrite
- **Recommendation rationale:** Preserves authentication continuity while converging account and authorization authority.
- **Advantages:** One login/account policy; consistent tenant binding and revocation.
- **Risks / disadvantages:** Token compatibility, account mapping, role semantic drift and outage blast radius.
- **Impact on existing Transportation:** Use adapters/dual validation only under explicit phases; preserve existing IDs/audit.
- **Impact on Platform Management:** Depends on trusted account/tenant/licensing facts but provider operators remain separate.
- **Impact on tenant databases:** Identity data stays outside business schemas except logical actor IDs.
- **Impact on frontend:** One customer login/session target; provider interface remains separate.
- **Security impact:** Highest-risk boundary: credentials, JWT, tenant binding, revocation and privilege escalation.
- **Scalability/operations impact:** May justify independent deployment only after availability/security readiness.
- **Dependencies:** D-ERP-02, 14, 15–17, 20
- **Questions the product owner must answer:** Who operates Identity, token authority, migration phases and rollback?
- **Current status:** `DECISION REQUIRED`

## D-ERP-13 — Cross-Domain Communication

- **Problem / question:** Set ports/APIs/events/durable delivery/read-model standard.
- **Existing verified context:** Existing rules already prohibit foreign repositories/tables and require registered contracts.
- **Available options:** Direct DB coupling; universal synchronous APIs; context-sensitive ports/APIs/events/read models.
- **Recommended option:** Context-sensitive contracts with existing prohibitions retained
- **Recommendation rationale:** Matches consistency need and deployment boundary without one transport for every interaction.
- **Advantages:** Clear ownership, testability, evolution and extraction readiness.
- **Risks / disadvantages:** Contract/version/retry/ordering governance adds work.
- **Impact on existing Transportation:** Preserves published Transportation ports/events; future contracts need separate approval.
- **Impact on Platform Management:** Uses explicit control-plane decisions, never database inspection.
- **Impact on tenant databases:** No foreign writes/FKs; local owner transactions only.
- **Impact on frontend:** Frontend uses backend composition/read models, not cross-domain data access.
- **Security impact:** Tenant, authorization, minimization, idempotency and audit are mandatory.
- **Scalability/operations impact:** Durability only where consumers/reliability justify it; observability required.
- **Dependencies:** All ownership decisions; D-ERP-17
- **Questions the product owner must answer:** Approve guidance and governance owner for contract review?
- **Current status:** `DECISION REQUIRED`

## D-ERP-14 — Common Frontend

- **Problem / question:** Choose one customer ERP experience across backend boundaries.
- **Existing verified context:** Accepted product target is one common customer product; current Transportation frontend exists.
- **Available options:** Separate UIs/logins; one codebase/product shell; federated micro-frontends.
- **Recommended option:** One customer ERP frontend codebase/product experience
- **Recommendation rationale:** Backend deployment need not fragment navigation, login or accessibility.
- **Advantages:** Consistent UX, entitlement/RBAC visibility and shared quality gates.
- **Risks / disadvantages:** Large frontend needs feature ownership and release discipline.
- **Impact on existing Transportation:** Transportation features integrate into common shell without backend merge.
- **Impact on Platform Management:** Provider Platform interface remains separate privileged UI.
- **Impact on tenant databases:** No direct database impact.
- **Impact on frontend:** Entitlement + activation + permission + scope + localization determine visibility.
- **Security impact:** Frontend hiding never replaces backend authorization.
- **Scalability/operations impact:** One release train initially; modular feature boundaries enable future options.
- **Dependencies:** D-ERP-04, 12, 15
- **Questions the product owner must answer:** Approve one codebase and integration/route ownership model?
- **Current status:** `DECISION REQUIRED`

## D-ERP-15 — Platform Management Deployment

- **Problem / question:** Decide whether provider Control Plane remains separately deployable and privileged.
- **Existing verified context:** Accepted ADR separates provider control plane from customer ERP responsibility/security.
- **Available options:** Embed in ERP runtime; separate deployable control plane; shared process with hard module boundary.
- **Recommended option:** YES — separate deployable Control Plane
- **Recommendation rationale:** Privileges, failure exposure and operational responsibilities differ materially.
- **Advantages:** Smaller trust boundary; independent provider operations and release control.
- **Risks / disadvantages:** Requires secure, highly available contracts and operational coordination.
- **Impact on existing Transportation:** Transportation consumes only entitlement/tenant/routing decisions; no provider admin access.
- **Impact on Platform Management:** Direct: remains provider owner of subscriptions, billing, provisioning and rollout.
- **Impact on tenant databases:** Own management database; orchestrates but does not own tenant-domain tables.
- **Impact on frontend:** Separate provider interface; customer billing recovery is narrowly exposed.
- **Security impact:** Strongest privilege isolation; support access must be audited.
- **Scalability/operations impact:** Independent availability/observability and fail-closed data-plane behavior needed.
- **Dependencies:** D-ERP-04, 12–14, 18
- **Questions the product owner must answer:** Approve separate deployment and define failure-mode behavior?
- **Current status:** `DECISION REQUIRED`

## D-ERP-16 — Specialized / Independent Services

- **Problem / question:** Decide whether specialized capabilities are extracted only with evidence.
- **Existing verified context:** Tracking already uses justified Kafka/Redis/Timescale/PostgreSQL; others vary.
- **Available options:** Keep all core; extract all named capabilities; selective evidence-driven extraction.
- **Recommended option:** YES — evidence-driven selective extraction
- **Recommendation rationale:** Preserves specialized needs without treating every capability/domain as a service.
- **Advantages:** Workload fit and failure/security isolation where valuable.
- **Risks / disadvantages:** Too many services create cost and distributed coupling.
- **Impact on existing Transportation:** Tracking evidence is preserved; Transportation submodules are not automatically services.
- **Impact on Platform Management:** Control Plane remains separately assessed under D-ERP-15.
- **Impact on tenant databases:** Specialized stores only for owning service/capability.
- **Impact on frontend:** One customer experience remains possible.
- **Security impact:** Documents/Identity may justify isolation; each needs threat evidence.
- **Scalability/operations impact:** Needs mature deployment, tracing, SLOs and on-call ownership.
- **Dependencies:** D-ERP-01, 17
- **Questions the product owner must answer:** Which candidates have measured need and an owning operations team?
- **Current status:** `DECISION REQUIRED`

## D-ERP-17 — Service Extraction Rule

- **Problem / question:** Define evidence and cost test for physical extraction.
- **Existing verified context:** High usage and domain importance are insufficient; Tracking is a concrete workload example.
- **Available options:** Universal threshold; subjective preference; qualitative evidence scorecard and explicit ADR.
- **Recommended option:** Qualitative scorecard plus capability-specific ADR
- **Recommendation rationale:** Balances benefit dimensions against distribution cost without false universal scoring.
- **Advantages:** Transparent decisions; avoids architecture fashion and hidden costs.
- **Risks / disadvantages:** Requires disciplined evidence collection and periodic review.
- **Impact on existing Transportation:** No current Transportation extraction without scorecard evidence.
- **Impact on Platform Management:** Its separation is assessed on privilege/security, not traffic alone.
- **Impact on tenant databases:** Data separability is mandatory before extraction.
- **Impact on frontend:** Frontend boundary is independent of service boundary.
- **Security impact:** Security benefit must exceed new network attack surface.
- **Scalability/operations impact:** Must fund retries, tracing, versioning, reconciliation, deployment and support.
- **Dependencies:** D-ERP-01, 16
- **Questions the product owner must answer:** Who reviews scorecards and what evidence/SLOs are mandatory?
- **Current status:** `DECISION REQUIRED`

## D-ERP-18 — Tenant Schema Migration Strategy

- **Problem / question:** Choose common package with staged, per-database execution and verification.
- **Existing verified context:** Platform target already describes coordinated rollout with truthful independent database outcomes.
- **Available options:** Simultaneous global transaction; ad hoc tenant migrations; governed package and staged fleet rollout.
- **Recommended option:** YES — governed package, staged rollout, independent histories
- **Recommendation rationale:** A global release is coordination, not distributed atomicity.
- **Advantages:** Repeatable versions; pilot/rings; pause/recovery; truthful partial status.
- **Risks / disadvantages:** Fleet orchestration, compatibility windows and drift handling are required.
- **Impact on existing Transportation:** Transportation release remains independent unless explicitly coordinated.
- **Impact on Platform Management:** Owns orchestration evidence; modules own migration semantics.
- **Impact on tenant databases:** Each tenant DB records its own history/result; unused tables may remain empty.
- **Impact on frontend:** Feature visibility waits for entitlement, activation and schema readiness.
- **Security impact:** Failed or outdated databases must fail closed for affected capability.
- **Scalability/operations impact:** Needs backup/restore verification, telemetry and operational runbooks.
- **Dependencies:** D-ERP-03, 15, 19
- **Questions the product owner must answer:** Approve rollout semantics, pause authority and recovery acceptance?
- **Current status:** `DECISION REQUIRED`

## D-ERP-19 — Future Schema Optimization Trigger

- **Problem / question:** Decide whether conditional schemas wait for measured inefficiency.
- **Existing verified context:** No tenant-count/overhead evidence currently justifies schema combinations.
- **Available options:** Optimize now; never optimize; review measured signals through future ADR.
- **Recommended option:** YES — postpone and require evidence/new ADR
- **Recommendation rationale:** Avoids schema-profile explosion while retaining a governed optimization path.
- **Advantages:** Lower current complexity; objective future review.
- **Risks / disadvantages:** Common-schema fleet overhead may grow before threshold is defined.
- **Impact on existing Transportation:** No current schema split.
- **Impact on Platform Management:** Collects fleet metrics without inferring product entitlements from schema.
- **Impact on tenant databases:** Review tenant/table/index counts, unused percentage, migration/backup/catalog/resource/support cost.
- **Impact on frontend:** No direct impact until future decision.
- **Security impact:** Uniform patch coverage initially.
- **Scalability/operations impact:** Metrics and periodic review ownership are needed.
- **Dependencies:** D-ERP-03, 18
- **Questions the product owner must answer:** Who owns measurements and when is formal review triggered?
- **Current status:** `DECISION REQUIRED`

## D-ERP-20 — Pre-Core Implementation Foundation Phase

- **Problem / question:** Require formal ownership/integration foundation approval before core implementation.
- **Existing verified context:** Core blueprints are proposed and overlap existing Transportation authority.
- **Available options:** Implement domains now; decide incrementally during coding; approve foundation first.
- **Recommended option:** YES — approve foundation before core-domain implementation
- **Recommendation rationale:** Prevents duplicate masters, incompatible contracts and premature distribution.
- **Advantages:** Clear sequence, ownership, migration and risk gates.
- **Risks / disadvantages:** Delays feature coding until decisions are made; analysis can become stale if not maintained.
- **Impact on existing Transportation:** Preserves implementation and defines reconciliation rather than rewrite.
- **Impact on Platform Management:** Clarifies control-plane/data-plane integration before dependence grows.
- **Impact on tenant databases:** Resolves schema/topology/release approach first.
- **Impact on frontend:** Sets common product and Identity transition boundaries.
- **Security impact:** Makes authorization/tenant authority explicit before expansion.
- **Scalability/operations impact:** Reduces rework; requires accountable owner review and dated ADRs.
- **Dependencies:** D-ERP-01–19
- **Questions the product owner must answer:** Approve the phase, decision owners, deadline and implementation gate?
- **Current status:** `DECISION REQUIRED`
