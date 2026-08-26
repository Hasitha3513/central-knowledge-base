# Inventory and Warehouse — Blueprint

Lifecycle: PROPOSED.

## Scope and Ownership

Owns item master, stock-keeping units, warehouses, bins, lots/serials, stock balances, reservations, receipts, issues, transfers, counts, adjustments, valuation facts, and replenishment signals. Procurement owns purchasing; Maintenance owns work orders; Transportation owns movement execution.

## Candidate Aggregates and Use Cases

- Item: SKU, classification, units, lot/serial policy, status.
- Warehouse: location, bins, zones, capacity and handling policy.
- Stock Ledger: immutable movements and derived balances.
- Reservation: demand allocation, expiry, release, fulfillment.
- Receipt/Issue/Transfer/Count: controlled stock transitions and variance approval.
- Check availability, reserve/release, receive, pick/issue, transfer, count, adjust, and value inventory.

## Integration Points

Consumes purchase-order expectations, maintenance/project material demand, and transport delivery milestones. Publishes stock/reservation/receipt/issue/adjustment/reorder events. Provides available-to-promise and reservation commands through ports/APIs. Physical movement may reference transportation shipment/trip IDs logically.

## Candidate Tables (Not Approved)

`item`, `item_unit`, `warehouse`, `bin_location`, `stock_lot`, `stock_balance`, `stock_movement`, `stock_reservation`, `goods_receipt`, `goods_receipt_line`, `stock_issue`, `stock_issue_line`, `stock_count`, `stock_count_line`.

## Invariants

Tenant-scoped SKU/warehouse codes; immutable movement ledger; no negative stock unless an explicit policy allows it; unit conversion is deterministic; every balance change traces to a movement and idempotency key.

## Open Decisions

Valuation method, negative stock, lot/serial depth, quality inspection, consignment, multiple units, and warehouse automation protocols.
