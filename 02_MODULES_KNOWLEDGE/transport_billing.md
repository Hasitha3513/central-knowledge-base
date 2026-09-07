# Transport Billing

Lifecycle: `PRODUCT_DECISIONS_FROZEN / IMPLEMENTATION_NOT_STARTED_US47`.

## Phase 1 Current MVP Scope

The dedicated `billing` bounded context converts eligible completed transport activity into immutable operational billing records. Its aggregate is `TransportBillingRecord`. It owns source claims/snapshots, explicit charge composition, supplied tax facts, required cost-centre allocation, validation, independent approval, operational finalization, exact compensating reversal, append-only history, and controlled export requests.

Eligible source types are exactly Closed Trip and an explicit Freight-owner `COMPLETED` or `CLOSED` billable fact. The current Freight Order lacks terminal lifecycle authority; Billing must fail closed until that provider contract exists. Delivery, Fuel, Driver payroll, Customer data, exceptions and analytics do not independently create Phase 1 bills. Organization owns Customer master; all external references are Tenant-scoped logical UUIDs.

A regular record is per-source, per-Customer and single Tenant-default ISO-4217 currency. Line categories are `BASE_CHARGE`, `SURCHARGE`, `PENALTY`, and `CREDIT_ADJUSTMENT`; commercial quantities/rates are authorized explicit inputs with provenance. Money is `BigDecimal`/`NUMERIC(19,2)`, precision 19, scale 2, `HALF_UP`; quantity/rate may use scale 4. Total equals base plus surcharge plus penalty minus credit adjustment plus supplied tax. No FX, universal pricing/rate catalogue, or generic discount engine is approved.

Billing does not calculate jurisdictional tax. It stores externally supplied category, jurisdiction, taxable amount, rate/amount, exemption and provenance facts; `NOT_SUPPLIED` is not an exemption claim. US-72 owns tax/billing compliance decisions. Cost-centre allocations are operational references, sum to 100.0000% of pre-tax subtotal and are not GL accounts.

Lifecycle is `DRAFT -> VALIDATED -> APPROVED -> FINALIZED -> EXPORT_REQUESTED -> EXPORTED`, plus reasoned draft-only `CANCELLED` and reversal-driven `REVERSED`. Preparer differs from approver. `FINALIZED` locks all commercial/source facts; it is not a tax invoice, posting, payment or settlement. Corrections after finalization use one new exact compensating `REVERSAL` record and preserve the original.

Phase 1 external mode is controlled US-73 `FILE_JSON_V1`. The P1-01 family is `TransportBillingExportRequestedV1` / `TRANSPORT_BILLING_V1`, classification `FINANCIAL_CONFIDENTIAL`, aggregate `TRANSPORT_BILLING_RECORD`, at-least-once and unordered. `EXPORTED` proves file/hash delivery only. Finance/external accounting retains tax invoice, GL, AR/AP, official posting, periods, payment, banking, settlement, remittance, cash application, credit/collections and Customer balances.

The frozen permissions are `BILLING_VIEW`, `BILLING_PREPARE`, `BILLING_APPROVE`, `BILLING_FINALIZE`, and `BILLING_EXPORT`. All records, children, history, keys, events and references use trusted server-derived Tenant identity. The API is the explicit `/api/v1/billing/records` list/detail/draft-lines/validate/approve/cancel/finalize/reversal/export/history family. No generic status, finalized edit/delete, Finance, payment, posting, tax filing, raw payload, retry or manual-success route is approved.

## Expected Persistence for Implementation

Expected Billing-owned Tenant tables are `transport_billing_record`, `transport_billing_line`, `transport_billing_cost_centre`, `transport_billing_tax_fact`, `transport_billing_source_claim`, and `transport_billing_history`. These are planning names only: no table exists and no migration version is reserved. Implementation must document exact post-migration dictionaries. Same-module relationships use Tenant-consistent foreign keys; Customer/Trip/Freight/Compliance/Integration references have no physical foreign key or foreign SQL.

## Phase 2 Post-MVP Future Roadmap

Customer-period consolidation, mixed source batches, formal tax invoice issuance, live ERP/accounting adapters, inbound acknowledgement, posting/payment/settlement states, FX, pricing/rate-card engines, Customer contracts, Finance fiscal periods, and named surcharge/penalty catalogues require new product and integration decisions.

## Acceptance Expectation

PostgreSQL and real Chromium acceptance must prove eligible-source enforcement, deterministic money, supplied-tax behavior, cost-centre totals, five-permission RBAC, Tenant/IDOR isolation, preparer/approver segregation, source uniqueness, idempotency and races, finalized immutability, exact reversal, append-only history, durable controlled file/hash evidence, privacy and absence of posted/paid claims. Development data must not be used for destructive acceptance.
