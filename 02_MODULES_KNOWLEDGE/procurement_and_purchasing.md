# Procurement and Purchasing — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns supplier sourcing references, requisitions, approvals, requests for quotation, quotations, purchase orders, amendments, supplier contracts, and purchasing compliance. Inventory owns receipts/stock; Finance owns supplier accounting and payment.

## Candidate Aggregates and Use Cases

- Requisition: requester, cost allocation, lines, approval state.
- Sourcing Event: invited suppliers, bids, evaluation, award.
- Purchase Order: supplier, commercial terms, lines, delivery schedule, lifecycle/version.
- Supplier Contract: validity, pricing, limits, service levels.
- Request, approve/reject, source, award, issue/amend/cancel order, acknowledge, and close.

## Integration Points

Consumes stock replenishment and maintenance/project demand, supplier/account status, budget availability, and receipt outcomes. Publishes requisition, award, purchase-order, and contract lifecycle events. Calls Finance for budget controls and Inventory for item/warehouse validation through ports.

## Legacy Reconciliation

Transportation currently stores a `vendor` table and fuel purchases. Supplier master ownership and the boundary between operational fuel purchasing and enterprise procurement require an ADR; preserve existing fuel APIs until migration is approved.

## Candidate Tables (Not Approved)

`supplier`, `requisition`, `requisition_line`, `approval_step`, `rfq`, `rfq_supplier`, `supplier_quote`, `purchase_order`, `purchase_order_line`, `delivery_schedule`, `supplier_contract`.

## Open Decisions

Supplier master owner, approval engine, three-way matching ownership, catalog integration, tendering rules, and fuel-specific purchasing boundaries.
