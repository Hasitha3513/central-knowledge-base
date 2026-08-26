# Finance and Accounting — Blueprint

Lifecycle: PROPOSED; no schema, API, or event in this file is implemented.

## Scope and Ownership

Owns chart of accounts, fiscal periods, journals, ledgers, receivables, payables, invoices, payments, tax accounting, bank reconciliation, budgets, fixed assets, currencies, and financial reporting. It does not own operational trips, purchase orders, stock, projects, employees, or maintenance work.

## Candidate Aggregates and Use Cases

- Ledger: account hierarchy, fiscal periods, balanced journal posting, close/reopen controls.
- Receivables: customer account, invoice, credit note, receipt, allocation, aging.
- Payables: supplier account, supplier invoice, debit note, payment run.
- Budget: budget version, cost-center/project allocation, commitment and availability checks.
- Fixed Asset: capitalization, depreciation, transfer, disposal.
- Create/approve/post/reverse journals; issue/settle invoices; approve payments; reconcile bank statements; close periods; calculate depreciation.

## Integration Points

Consumes completed trip/freight/fuel/maintenance charge facts, purchase-order/receipt/invoice facts, sales orders, payroll summaries, inventory valuation, and project cost facts. Publishes invoice/payment/journal/budget/asset outcomes. Synchronous consumers may request account, tax, fiscal-period, and budget validation through registered APIs.

## Data and Tenant Rules

Every financial row is tenant-scoped. Monetary values use decimal plus ISO-4217 currency. Journal entries are immutable after posting; corrections use reversals. Document numbers are unique per tenant and type. Period locks are enforced in the domain.

## Candidate Tables (Not Approved)

`account`, `fiscal_period`, `journal`, `journal_line`, `customer_account`, `supplier_account`, `invoice`, `invoice_line`, `payment`, `payment_allocation`, `budget`, `budget_line`, `fixed_asset`, `depreciation_entry`. Exact dictionaries must be added only with approved migrations.

## Open Decisions

Accounting basis, tax jurisdictions, functional/reporting currencies, consolidation, dimensions/cost centers, approval matrix, payment providers, and regulatory retention.
