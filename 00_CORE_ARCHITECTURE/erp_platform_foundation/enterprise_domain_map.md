# Enterprise Domain Map

Status: **RECOMMENDED — OWNER DECISION REQUIRED**, except facts explicitly labelled otherwise.

| Domain | Responsibility / likely authoritative concepts | Must not own | Synchronous needs | Asynchronous/read-model needs | Consistency, scaling, security | Recommendation and open question |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Finance & Accounting | GL, fiscal periods, journals, AR/AP, formal invoices, payment accounting, banking, tax accounting, financial reporting | Trips, POs, stock, operational transport charges, provider SaaS billing | Account/period/tax/budget validation | Source facts in; posting/payment outcomes and finance read models out | Balanced posting/local ACID; strong SoD, retention and privacy | **RECOMMENDED — OWNER DECISION REQUIRED:** Finance authority; decide accounting/jurisdiction scope. |
| Human Resource Management | Person/Employee, Employment, position, assignment, leave, attendance and workforce qualifications | Login identity; trip assignment; transport operational eligibility | Current employment/availability/qualification validation | Effective-dated workforce changes; payroll summaries | Sensitive/regulated data; strong privacy; availability affects dispatch | **RECOMMENDED — OWNER DECISION REQUIRED:** split Employee/Employment from Driver operational profile. |
| Inventory & Warehouse | Item, SKU, warehouse/bin, lots/serials, movement ledger, balances, reservation, receipt/issue/count | Purchasing intent, maintenance work, transport execution, general ledger | Availability/reserve/release/item validation | Movements, receipts, reorder, valuation/read models | Local stock transaction and idempotency; high write volume possible | **RECOMMENDED — OWNER DECISION REQUIRED:** enterprise stock authority; preserve explicit specialized ledgers. |
| Procurement & Purchasing | Requisition, sourcing/RFQ, supplier selection, PO, contract and amendment lifecycle | Stock receipt/balance; AP/payment accounting | Budget, supplier/item/warehouse validation | PO expectations, receipt and invoice-match outcomes | Approval/SoD; workflow complexity; supplier privacy | **RECOMMENDED — OWNER DECISION REQUIRED:** reconcile Transport vendor/Fuel purchase boundary. |
| Project Management | Project, WBS, milestones, schedule, assignments, risks, budget intent/forecast | Finance actual postings; Trip execution | Project/cost-code/resource validation | Staffing, commitments, actuals, transport facts | Planning consistency; potentially compute-heavy scheduling later | **RECOMMENDED — OWNER DECISION REQUIRED:** own Project master; map legacy Transport UUIDs. |
| Sales & CRM | Business Account/Customer relationship, Contact, Lead, Opportunity, Quote, Sales Order, service case | Delivery/trip execution; formal AR/payment | ATP, credit and shipment planning/status | Account/order changes and fulfillment/finance read models | PII/consent and dedupe; customer lookup availability | **RECOMMENDED — OWNER DECISION REQUIRED:** canonical customer relationship; preserve current contact/token contracts during migration. |
| Vehicle Maintenance | Maintenance plan/work order, inspection/defect/breakdown, parts/labor, hold/release, return-to-service | Vehicle master; stock; Trip allocation; GL | Vehicle/meter, parts reservation, dispatch eligibility | Due/hold/work/downtime/cost facts | Safety-critical state and availability; strong audit | **RECOMMENDED — OWNER DECISION REQUIRED:** Maintenance lifecycle separate from Fleet master. |
| Transportation & Logistics | **CURRENT VERIFIED IMPLEMENTATION:** Fleet master/readings, Driver operational profile, Routing, Trip, Fuel, Freight, Delivery, operational Billing, Tracking and supporting contexts | Future Finance ledger, enterprise HR employment, enterprise stock/PO/project/customer masters once reconciled | Eligibility/reference/assignment decisions | Operational facts, external delivery and reporting projections | Mature modular app; Tracking specialized load; tenant isolation and existing acceptance | **RECOMMENDED — OWNER DECISION REQUIRED:** preserve independently deployable application and reconcile overlaps incrementally. |

## Platform interactions

- **ACCEPTED EXISTING DECISION:** Platform Management owns subscriptions, account licensing, capability entitlement, provider SaaS billing, tenant provisioning and release orchestration—not customer-domain master data.
- **RECOMMENDED — OWNER DECISION REQUIRED:** customer runtime consumes trusted tenant/subscription/capability decisions; schema presence, activation, entitlement and user permission stay separate.
- **RECOMMENDED — OWNER DECISION REQUIRED:** provider Control Plane remains independently deployable; the customer frontend remains one product experience.

## Reporting/read-model principle

Each owner publishes minimized facts or focused queries. Reporting owns composition/presentation and its own read models, not source truth. Cross-domain dashboards must not join foreign owner tables. Formal financial reports remain Finance-owned; operational analytics remain with the interpreting domain or an explicitly governed read model.


## Per-domain decision-preparation detail

### Finance & Accounting

- **Likely aggregates/data:** Ledger, Journal, Receivable, Payable, Customer/Supplier accounting account, formal Invoice, Payment allocation, Budget and Fixed Asset.
- **Dependencies:** synchronous account/period/tax/budget validation; durable operational source facts; Finance-owned accounting outcome/read models.
- **Transactions/reporting/scaling/security:** balanced postings and period locks are local; corrections reverse; formal financial reporting is Finance-owned; strong SoD, retention and sensitive banking controls apply. Scale alone does not require a service.
- **Platform/Transportation:** provider SaaS billing stays outside Finance; Transportation supplies immutable operational facts and never marks itself posted/paid.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-05. Decide accounting basis, jurisdictions, currencies, posting acknowledgement and reconciliation.

### Human Resource Management

- **Likely aggregates/data:** Employee, Employment, Position, Assignment, Qualification, Leave, Attendance and payroll inputs; not credentials/JWT or Trip assignment.
- **Dependencies:** synchronous current employment/availability checks; effective workforce facts; payroll summary/employee-cost read models.
- **Transactions/reporting/scaling/security:** effective-dated employment transactions are local; privacy, medical/disciplinary minimization and jurisdictional retention dominate. Dispatch availability creates availability pressure but not automatic extraction.
- **Platform/Transportation:** Account and Employee stay distinct; Transportation retains operational Driver and safety/dispatch eligibility pending record-by-record reconciliation.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-06. Classify current Driver records and define failure-safe dispatch behavior.

### Inventory & Warehouse

- **Likely aggregates/data:** Item/SKU, Warehouse/Bin, Lot/Serial, Movement Ledger, Balance, Reservation, Receipt, Issue, Transfer and Count; not PO intent or maintenance work.
- **Dependencies:** immediate availability/reserve/release; durable movement/receipt/reorder facts; valuation and availability read models.
- **Transactions/reporting/scaling/security:** movement and balance consistency is owner-local and idempotent; warehouse/resource scope and adjustment approval are required. High write load is measurable extraction pressure only.
- **Platform/Transportation:** Platform only entitles capability; Transportation consumes stock and retains explicitly approved specialized Fuel/Bunker ledger semantics.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-09. Decide valuation, negative stock and specialized-ledger boundary.

### Procurement & Purchasing

- **Likely aggregates/data:** Requisition, Sourcing/RFQ, Supplier Quote/Award, PO/Amendment and Supplier Contract; not Inventory balance/receipt or Finance payable/payment.
- **Dependencies:** budget, supplier, item and warehouse validation; PO/award facts and receipt/payment outcomes; commitment read models.
- **Transactions/reporting/scaling/security:** approvals and PO versioning are local; SoD and commercial confidentiality apply. Do not build a generic approval engine before proven reuse.
- **Platform/Transportation:** Platform has no purchasing authority; Fuel purchase/vendor compatibility must remain until Supplier and PO ownership is approved.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-10. Decide Supplier master, three-way match and Fuel boundary.

### Project Management

- **Likely aggregates/data:** Project, WBS/Work Item, dependency, milestone, resource plan, risk/issue, budget intent and forecast; not Finance actual posting.
- **Dependencies:** project/cost-code/resource validation; workforce, commitment, inventory issue and actual-cost facts; project status/cost read models.
- **Transactions/reporting/scaling/security:** baseline/revision is local; schedule computation may become specialized only with evidence; project/resource scope needs ABAC.
- **Platform/Transportation:** capability entitlement is Platform-owned; current Transport Project UUID/history must remain compatible and become logical references after approved migration.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-11. Decide budget intent versus Finance control and compatibility period.

### Sales & CRM

- **Likely aggregates/data:** Business Account/Contact, Lead, Opportunity, Activity, Quote, Sales Order and Case; not Delivery execution, formal AR Invoice or Payment.
- **Dependencies:** ATP/credit/shipment validation; account/order facts and fulfillment/finance read models.
- **Transactions/reporting/scaling/security:** sales lifecycle is local; customer PII, consent and duplicate matching are primary controls. Customer lookup availability is operationally important but not automatic service justification.
- **Platform/Transportation:** SaaS subscribing organization is not CRM Account; current Organization Customer/contact and Delivery self-service contracts require staged mapping.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-08. Decide canonical Customer ID and migration semantics.

### Vehicle Maintenance

- **Likely aggregates/data:** Maintenance Plan, Work Request/Order, Inspection/Defect, Breakdown, Labor/Parts execution, Vehicle Hold and return-to-service; not Vehicle master or Inventory stock.
- **Dependencies:** immediate Vehicle/meter/parts/dispatch-eligibility decisions; hold/release, work, consumption, downtime and cost facts; readiness read models.
- **Transactions/reporting/scaling/security:** work lifecycle is local; safety hold/release must be consistent, audited and highly available. Workload evidence, not domain importance, determines extraction.
- **Platform/Transportation:** Platform entitles capability only; Fleet Vehicle/readings remain current authority and maintenance_schedule compatibility must preserve assignment blocking.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-07. Decide return-to-service authority and synchronous eligibility SLA.

### Existing Transportation & Logistics

- **Authoritative aggregates/data:** **CURRENT VERIFIED IMPLEMENTATION** includes Vehicle/Driver/Route/Trip/Fuel/Freight/Delivery and specialized Tracking/Billing/Integration/Operations/Compliance/System facts documented in current module records.
- **Dependencies:** current published ports and registered events are authoritative; future enterprise dependencies must use compatibility contracts, never foreign persistence.
- **Transactions/reporting/scaling/security:** source transitions remain owner-local; tenant isolation and existing RBAC/audit evidence stay intact. Tracking is the verified specialized workload example. Reporting must retain metric lineage.
- **Platform interaction:** future tenant/subscription/entitlement/Identity decisions require approved contracts; provider administration never grants customer-domain mutation.
- **Recommendation/risks/questions:** **RECOMMENDED — OWNER DECISION REQUIRED** under D-ERP-02. Preserve T1 while ownership overlaps are reconciled; define later merge/extraction evidence rather than assume it.
