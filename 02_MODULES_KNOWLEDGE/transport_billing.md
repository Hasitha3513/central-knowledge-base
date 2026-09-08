# Transport Billing

Lifecycle: `COMPLETE / FINAL_ACCEPTANCE_PASS_US47`.

## Phase 1 Current MVP Scope

The dedicated `billing` bounded context converts eligible completed transport activity into immutable operational billing records. Its aggregate is `TransportBillingRecord`. It owns source claims/snapshots, explicit charge composition, supplied tax facts, required cost-centre allocation, validation, independent approval, operational finalization, exact compensating reversal, append-only history, and controlled export requests.

Eligible source types are exactly Closed Trip and an explicit Freight-owner `COMPLETED` or `CLOSED` billable fact. Published provider-neutral Trip/Freight lookup contracts and the Freight-owned terminal fact projection are implemented. Delivery, Fuel, Driver payroll, Customer data, exceptions and analytics do not independently create Phase 1 bills. Organization owns Customer master; all external references are Tenant-scoped logical UUIDs.

A regular record is per-source, per-Customer and single Tenant-default ISO-4217 currency. Line categories are `BASE_CHARGE`, `SURCHARGE`, `PENALTY`, and `CREDIT_ADJUSTMENT`; commercial quantities/rates are authorized explicit inputs with provenance. Money is `BigDecimal`/`NUMERIC(19,2)`, precision 19, scale 2, `HALF_UP`; quantity/rate may use scale 4. Total equals base plus surcharge plus penalty minus credit adjustment plus supplied tax. No FX, universal pricing/rate catalogue, or generic discount engine is approved.

Billing does not calculate jurisdictional tax. It stores externally supplied category, jurisdiction, taxable amount, rate/amount, exemption and provenance facts; `NOT_SUPPLIED` is not an exemption claim. US-72 owns tax/billing compliance decisions. Cost-centre allocations are operational references, sum to 100.0000% of pre-tax subtotal and are not GL accounts.

Lifecycle is `DRAFT -> VALIDATED -> APPROVED -> FINALIZED -> EXPORT_REQUESTED -> EXPORTED`, plus reasoned draft-only `CANCELLED` and reversal-driven `REVERSED`. Preparer differs from approver. `FINALIZED` locks all commercial/source facts; it is not a tax invoice, posting, payment or settlement. Corrections after finalization use one new exact compensating `REVERSAL` record and preserve the original.

Phase 1 external mode is controlled US-73 `FILE_JSON_V1`. The P1-01 family is `TransportBillingExportRequestedV1` / `TRANSPORT_BILLING_V1`, classification `FINANCIAL_CONFIDENTIAL`, aggregate `TRANSPORT_BILLING_RECORD`, at-least-once and unordered. `EXPORTED` proves file/hash delivery only. Finance/external accounting retains tax invoice, GL, AR/AP, official posting, periods, payment, banking, settlement, remittance, cash application, credit/collections and Customer balances.

The implemented permissions are `BILLING_VIEW`, `BILLING_PREPARE`, `BILLING_APPROVE`, `BILLING_FINALIZE`, and `BILLING_EXPORT`. All records, children, history, keys, events and references use trusted server-derived Tenant identity. The API is the explicit `/api/v1/billing/records` list/detail/draft-lines/validate/approve/cancel/finalize/reversal/export/history family. No generic status, finalized edit/delete, Finance, payment, posting, tax filing, raw payload, retry or manual-success route exists.

## Implemented Persistence — V72

Flyway `V72__transport_billing_us47.sql` creates `transport_billing_record`, `transport_billing_line`, `transport_billing_cost_centre`, `transport_billing_tax_fact`, `transport_billing_source_claim`, and `transport_billing_history`, plus Freight-owned `freight_billing_fact`. Same-module relationships use Tenant-consistent composite foreign keys. Customer/Trip/Freight/Compliance/Integration references remain logical and Billing performs no foreign SQL.

`transport_billing_record` contains UUID `id`/`tenant_id`; unique Tenant billing number and create-idempotency identity; regular/reversal links; the complete minimized source type/ID/business number/terminal lifecycle/completion/version/SHA-256 snapshot; logical Customer ID; ISO currency and lifecycle; seven `NUMERIC(19,2)` amount projections; preparer/approval/finalization facts; logical Integration configuration/event references; validation/compliance facts; optimistic version and timestamps. It has Tenant-leading queue, Customer and source indexes, Tenant-consistent self foreign keys, and bounded type/lifecycle/currency/compliance/total checks.

`transport_billing_line` contains UUID/Tenant/record identity, category, reason, provenance, `NUMERIC(19,4)` quantity/unit rate and `NUMERIC(19,2)` amount, with non-negative/category checks. `transport_billing_cost_centre` contains code, `NUMERIC(7,4)` allocation, description and source with unique Tenant/record/code and percent checks. `transport_billing_tax_fact` is unique per Tenant/record and contains supplied/not-supplied status, category, jurisdiction, `NUMERIC(19,2)` taxable/tax amounts, `NUMERIC(9,4)` rate, exemption, provenance and SHA-256 snapshot with shape checks.

`transport_billing_source_claim` contains Tenant/record/source identity, active/released state and timestamps; a partial unique `(tenant_id,source_type,source_id) WHERE active` index enforces one effective regular claim. `transport_billing_history` contains action/from/to/actor/detail/time and optional command scope/key/request hash/result version; its Tenant/scope/key partial uniqueness persists idempotency. DB triggers prohibit history update/delete and released-record line/tax/cost-centre mutation. All child relationships use `(billing_record_id,tenant_id)` foreign keys and Tenant-leading indexes.

Concurrency verification uses independent PostgreSQL transactions synchronized by a `CyclicBarrier`. The deterministic nine-race matrix covers duplicate sources, same-key replay, same-key/different-request conflict, edit versus approval, double approval, double finalization, finalization versus cancellation, double reversal and double export. Command advisory locks derive from `tenantId + ":billing:" + scope + ":" + key`; reversal creation additionally locks the original record using scope `REVERSE_ORIGINAL`. This Tenant-scoped serialization complements V72 uniqueness and optimistic versions without a new migration. The matrix passes 9/9 with one effective history/outbox effect and no source mutation.

## Phase 2 Post-MVP Future Roadmap

Customer-period consolidation, mixed source batches, formal tax invoice issuance, live ERP/accounting adapters, inbound acknowledgement, posting/payment/settlement states, FX, pricing/rate-card engines, Customer contracts, Finance fiscal periods, and named surcharge/penalty catalogues require new product and integration decisions.

## Acceptance Expectation

PostgreSQL and real Chromium acceptance must prove eligible-source enforcement, deterministic money, supplied-tax behavior, cost-centre totals, five-permission RBAC, Tenant/IDOR isolation, preparer/approver segregation, source uniqueness, idempotency and races, finalized immutability, exact reversal, append-only history, durable controlled file/hash evidence, privacy and absence of posted/paid claims. Development data must not be used for destructive acceptance.

Independent technical closure passed against `transport_logistics_acceptance`: focused Billing 17/17, deterministic concurrency 9/9, full Maven 1,398 tests with zero failures/errors and 15 skipped, architecture 46/46, static/frontend gates, and real Chromium 7/7. US-47 remains acceptance-pending; no story accounting increment is authorized until hostile final acceptance.

Independent final acceptance passed with fresh evidence: focused Billing 17/17, deterministic concurrency 9/9, clean Flyway V1→V72, full Maven 1,398 tests with zero failures/errors and 15 skipped in 06:38, architecture 46/46, static/frontend gates, and real PostgreSQL-backed Chromium 7/7 in 37.0 seconds. Finance authority, source immutability, Tenant isolation, exact reversal, shared-outbox atomicity, controlled file/hash evidence, and delivery-only `EXPORTED` semantics pass. US-47 is COMPLETE; accounting is 72/87 with 15 remaining, Wave B is closed, and Wave C is active.
