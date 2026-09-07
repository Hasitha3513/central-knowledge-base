# ADR US47 Transport Billing Boundary

- **Status:** Accepted product decision; implementation not started
- **Date:** 2026-09-07
- **Decision task:** `US-47-TRANSPORT-BILLING-PRODUCT-DECISIONS-001`
- **Story accounting:** unchanged at 71 / 87 complete and 16 / 87 remaining

## Context

US-47 requires a Billing Officer to calculate completed Trip/Freight billing, visibly compose surcharges, penalties and cost-centre allocation, validate and finalize the billing record, and audit final changes. The source also names invoice reconciliation and tax handling but defines no tax jurisdiction/formula, official invoice authority, accounting acknowledgement, payment, cadence, or named ERP.

## Decision

1. A dedicated top-level `billing` bounded context is ratified because it owns an independent monetary aggregate and validation, approval, finalization, immutable history, reversal and external-handoff lifecycle.
2. Billing owns `TransportBillingRecord`, transport billable snapshots, explicit charge composition, authorized adjustments, cost-centre allocation, supplied tax facts, operational billing finalization, reversals, history, and the business meaning of an export request.
3. Finance/external accounting owns formal tax invoices, general ledger, AR/AP, accounting posting, fiscal close, payment, banking, settlement, remittance, cash application, credit/collections and customer balances. Organization owns Customer master. Source modules own operational lifecycle and facts. Integration owns technical exchange delivery.
4. Phase 1 source types are exactly `TRIP` and `FREIGHT_ORDER`. Trip eligibility is `CLOSED`. Freight requires an explicit owner-published `COMPLETED` or `CLOSED` billable fact; the current Freight model has no terminal lifecycle, so Billing cannot infer eligibility or query Freight persistence.
5. A regular record contains one source, one Customer logical reference and one Tenant-default ISO-4217 currency. Consolidated/customer-period billing and FX are deferred. Source uniqueness is `(tenant_id, source_type, source_id)` outside an explicit finalized reversal chain.
6. Exact line categories are `BASE_CHARGE`, `SURCHARGE`, `PENALTY` and `CREDIT_ADJUSTMENT`. Rates and quantities are explicit authorized inputs with provenance. No universal rate, surcharge, discount, penalty or pricing engine is approved.
7. Money uses `BigDecimal`, arithmetic precision 19, currency scale 2, `HALF_UP`; quantity/rate may use scale 4. The total is base plus surcharge plus penalty minus credit adjustment plus supplied tax. Negative regular totals fail.
8. Billing does not calculate jurisdictional tax. It stores supplied tax category/jurisdiction/taxable amount/rate/amount/exemption/provenance facts. `NOT_SUPPLIED` does not imply exempt or zero-rated. US-72 consumes minimized finalized tax/compliance facts and owns compliance decisions.
9. Cost centre is a required operational allocation reference, not a GL account. Allocations total 100.0000% of pre-tax subtotal and become immutable at finalization.
10. Lifecycle is `DRAFT -> VALIDATED -> APPROVED -> FINALIZED -> EXPORT_REQUESTED -> EXPORTED`, with reasoned draft-only `CANCELLED` and post-finalization `REVERSED` only after a new exact compensating reversal finalizes. Preparer and approver differ. Finalized content is immutable. No posted, booked, settled, paid or accounting-acknowledged state exists.
11. Operational numbers are Tenant/year scoped `TB-YYYY-NNNNNN`, gap-tolerant and never reused; they are not tax-invoice numbers. Mutations use command-scoped Tenant idempotency and optimistic versioning.
12. Phase 1 external mode is controlled US-73 `FILE_JSON_V1`. `TransportBillingExportRequestedV1` / `TRANSPORT_BILLING_V1` is a `FINANCIAL_CONFIDENTIAL`, durable P1-01 family consumed by `integration-outbound-exchange`. It is at-least-once, unordered, stable-ID, 32-KiB bounded, and contains minimized source/amount/tax/cost-centre/reversal facts. No second outbox or named/live accounting system is approved.
13. Exact permissions are `BILLING_VIEW`, `BILLING_PREPARE`, `BILLING_APPROVE`, `BILLING_FINALIZE` and `BILLING_EXPORT`. Tenant is server-derived and every aggregate/child/history/idempotency/event/reference is Tenant-scoped. No Finance-admin permission is introduced.
14. Billing owns only its tables and root contracts. Customer, Trip, Freight, Compliance and Integration references are logical UUIDs with no foreign repositories, SQL, JPA relationships, joins or physical foreign keys.

## Consequences

Implementation must establish a Freight owner completion contract before accepting a Freight source and an applicable US-72 decision path before claiming tax/billing compliance. PostgreSQL and real Chromium acceptance must prove exact calculations, Tenant/SoD/RBAC, source uniqueness, idempotency/races, immutable finalization, reversal, append-only audit, controlled file/hash evidence and absence of posting/payment claims. No migration number is reserved by this decision.
