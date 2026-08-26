# Sales and CRM — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns accounts/contacts, leads, opportunities, activities, quotations, price agreements, sales orders, customer service cases, and relationship history. Transportation owns freight/trip execution; Finance owns invoices/credit/payment.

## Candidate Aggregates and Use Cases

- Account and Contact: customer relationship, addresses, preferences, status.
- Lead/Opportunity: qualification, stage, value, probability, activities.
- Quote: priced lines, validity, approvals, acceptance.
- Sales Order: commercial commitment, lines, fulfillment references, lifecycle.
- Case: request/complaint, SLA, assignment, resolution.
- Capture/qualify lead, manage opportunity, quote, accept, create/hold/cancel sales order, and resolve case.

## Integration Points

Publishes account, quote, and sales-order lifecycle events. Requests available-to-promise from Inventory, shipment planning/status from Transportation, and credit/invoice/payment status from Finance. Transportation freight orders may logically reference a sales order and account.

## Legacy Reconciliation

Transportation currently owns `customer` and freight/trip customer references. Customer/account source-of-truth and compatibility need an ADR before extraction. IDs must remain stable or use an explicit mapping contract.

## Candidate Tables (Not Approved)

`account`, `contact`, `address`, `lead`, `opportunity`, `activity`, `quote`, `quote_line`, `sales_order`, `sales_order_line`, `customer_case`, `price_agreement`.

## Open Decisions

Customer-master owner, duplicate matching, consent/privacy, pricing ownership, credit-check behavior, and order-to-freight orchestration.
