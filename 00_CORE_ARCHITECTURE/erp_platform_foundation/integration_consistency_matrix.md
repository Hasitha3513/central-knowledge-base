# Integration & Consistency Matrix

Status: **RECOMMENDED — OWNER DECISION REQUIRED**. Entries are conceptual patterns only; no executable API, port, event or topic name is authorized.

## Communication standard

- Same runtime: framework-neutral in-process ports for focused validation/commands where immediate consistency is required.
- Independent deployments: stable tenant-qualified APIs for immediate decisions; committed domain/integration facts for decoupled propagation.
- Durable outbox/inbox only when delivery reliability and a real consumer require it; idempotency, ordering, retries, dead-letter/reconciliation and versioning must be explicit.
- Cross-domain reporting uses owner-published facts and consumer-owned read models; no foreign table reads.
- Existing mandatory prohibition remains: no cross-module repository injection, JPA entity reuse, foreign writes/joins/FKs, or shared-database ownership ambiguity.

| Workflow | Owner-local transaction | Candidate immediate interaction | After-commit/durable interaction | Eventual/read model | Reconciliation | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Purchase Order → Receipt → Stock → AP | Procurement PO; Inventory receipt/movement; Finance payable each local | Item/warehouse, budget and supplier validation | PO expectation, goods-receipt and supplier-invoice facts likely durable | PO/receipt/match/AP status views | Required for missing/duplicate/out-of-order facts | DECISION REQUIRED |
| Sales Order → Reservation → Delivery → AR | Sales order; Inventory reservation; Transport delivery; Finance AR local | ATP/reserve and credit checks | Accepted order, reservation, dispatch/delivery completion and invoice-source facts | Customer fulfillment/financial status | Required; no distributed transaction | DECISION REQUIRED |
| Maintenance Work → Parts → Vehicle availability | Maintenance work/hold; Inventory reservation/issue; Transport assignment local | Parts reservation and dispatch eligibility | Hold/release, consumption and cost facts | Vehicle readiness/work status | Safety-critical reconciliation required | DECISION REQUIRED |
| Transport Charge → Finance | Billing finalization/reversal local; Finance posting local | Optional duplicate/status query | Immutable finalized/reversal fact should be durable | Accounting acknowledgement/reconciliation view | Mandatory; export delivery is not posting | DECISION REQUIRED |
| Employee status → Driver eligibility | HRM employment local; Transport Driver decision local | Current employment/qualification validation for dispatch | Effective workforce changes | Driver eligibility projection | Required for missed/reordered changes | DECISION REQUIRED |
| Project procurement → Inventory/Finance | Project demand/budget intent; Procurement/Inventory/Finance local | Project/cost-code/budget validation | Commitment, receipt, issue and actual-cost facts | Project cost/commitment read model | Required across periods/corrections | DECISION REQUIRED |
| Delivery completion → billing/accounting | Delivery completion; Billing charge; Finance posting local | Eligibility/reference validation | Completion fact, charge finalization, accounting acceptance | End-to-end lineage view | Mandatory with stable source IDs | DECISION REQUIRED |

## Transaction rule

One local transaction may cover only state owned by that bounded context and database. A local `@Transactional` annotation cannot promise atomicity across deployments or owners. When a workflow crosses owners, use explicit state, idempotency, compensating action where valid, and reconciliation; never hide partial failure behind a global success flag.
