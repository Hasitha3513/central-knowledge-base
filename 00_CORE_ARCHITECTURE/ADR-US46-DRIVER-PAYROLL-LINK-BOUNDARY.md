# ADR-US46: Driver Payroll-Input Link Boundary

- **Status:** Accepted product decision; implementation and independent technical closure complete; final acceptance pending
- **Date:** 2026-09-07
- **Decision task:** `US-46-DRIVER-PAYROLL-LINK-PRODUCT-DECISIONS-001`

## Technical remediation authorization (2026-09-07)

`US-46-TECHNICAL-REMEDIATION-AUTHORIZATION-001` authorizes exactly one forward migration, V71 (verified free), for durable Tenant-scoped worker-mapping command idempotency. V1–V70 remain immutable. Same key/same canonical mapping request must replay without version or audit mutation; same key/different request must conflict deterministically. The authorization also requires correction of the already-frozen delivery-state, validation-time 32-KiB, correction lifecycle, Tenant-denial, PostgreSQL atomicity/concurrency, mapping-snapshot, source-immutability, and Chromium evidence gaps. It authorizes no new public API, permission, outbox, external acknowledgement, payroll behavior, or accounting change.
- **Story accounting:** unchanged at 70 / 87 accepted and 17 / 87 remaining

The authorized remediation is implemented and technically verified. V71 adds only the same-module, tenant-leading, append-only worker-mapping command record. PostgreSQL advisory transaction locks serialize mapping/batch idempotency and released-source approval claims; correction creation and outbox publication are atomic; successful controlled-file delivery projects `EXPORTED`; and PostgreSQL JSONB is re-canonicalized before file output so the delivered bytes match the recorded SHA-256. Dedicated PostgreSQL tests passed 18/18, including the deterministic nine-race matrix; full Maven passed 1,380/0/0/15 and real Chromium passed 7/7. US-46 remains acceptance pending; next task is `US-46-DRIVER-PAYROLL-LINK-TECHNICAL-CLOSURE-001-RERUN`.

Independent technical closure rerun passes. Fresh evidence is PostgreSQL 18/18 with concurrency 9/9 and clean V1→V71, full Maven 1,380/0/0/15 in 06:13, architecture 46/46, all static/frontend gates, and real PostgreSQL-backed Chromium 7/7 in 39.2 seconds. The boundary, lifecycle, four permissions, P1-01 at-least-once outbox, controlled file/hash semantics, privacy, and Payroll/HRMS authority remain unchanged. US-46 remains `IMPLEMENTATION_COMPLETE / ACCEPTANCE_PENDING`; accounting remains 70/87. Next task: `US-46-DRIVER-PAYROLL-LINK-FINAL-ACCEPTANCE-001`.

## Context

The Finance Officer must link Completed Trip earnings, allowances, overtime, and deductions to Drivers and export an audited payroll input. The source explicitly places final salary processing in Payroll/HRMS and supplies no statutory formula, cadence, named vendor, or live HRMS contract. Trip already publishes same-application assignment projections, and US-73 provides only the accepted outbound JSON file capability.

## Decision

1. Driver within Fleet owns the Tenant-scoped `DriverPayrollInputBatch`, source-backed lines, external worker-reference mapping, approval, immutable release, correction batches, and history. No new top-level bounded context is created.
2. Trip owns lifecycle/assignment facts; only same-Tenant `COMPLETED` or `CLOSED` Trips with actual end inside `[periodStart, periodEndExclusive)` and at/before `cutoffAt` qualify. Cross-module access uses published provider-neutral contracts only.
3. Exact categories are `TRIP_EARNING`, `ALLOWANCE`, `OVERTIME`, and `DEDUCTION`. Source-silent subtypes/statutory formulas are not invented. Authorized batch-scoped quantity/rate or fixed-amount inputs are snapshotted; single-currency amounts use scale 2 and `HALF_UP`. The operational provisional net input is additions less deductions and is never final net salary or amount payable.
4. Lifecycle is `DRAFT -> VALIDATED -> APPROVED -> EXPORT_REQUESTED -> EXPORTED`; `SUPERSEDED` requires a later exported correction batch. Approval freezes content. Correction is compensating and append-only. The preparer cannot approve the same batch.
5. Phase 1 approves only US-73 `FILE_EXCHANGE / FILE_JSON_V1 / OUTBOUND`, with business family `DRIVER_PAYROLL_INPUT_V1`, classification `FINANCIAL_CONFIDENTIAL`, and a controlled filesystem acceptance destination. No named HRMS, API, webhook, inbound acknowledgement, SFTP, or bidirectional integration is approved.
6. Driver publishes `DriverPayrollInputExportRequestedV1` through P1-01 in the release transaction to consumer `integration-outbound-exchange`. Delivery is at-least-once, idempotent by stable release event and Integration's accepted exchange key, globally unordered, and not proof of payroll import, posting, settlement, or payment.
7. Payload is minimized to batch/period/cutoff/currency/totals, Driver UUID plus opaque external worker reference, and source-backed line facts. Names, contacts, medical/licence/drug-test data, bank/tax/pension data, credentials, raw notes, and unrestricted metadata are prohibited.
8. Payroll/HRMS owns employee master, pay calendars, salary runs, salary calculation, statutory/voluntary deductions, benefits, payslips, final settlement, and payment initiation. Finance owns ledger, tax accounting, bank, payment, and posting outcomes. Scheduling may supply a future duty baseline only through a separately accepted contract.
9. Four permissions are frozen but unseeded: `DRIVER_PAYROLL_VIEW`, `DRIVER_PAYROLL_PREPARE`, `DRIVER_PAYROLL_APPROVE`, and `DRIVER_PAYROLL_EXPORT`. Tenant authority is server-derived and every row, reference, event, idempotency key, and worker path remains Tenant-scoped.
10. The current Flyway head is V67. This decision reserves no migration number and modifies no application code, API, schema, or story accounting.

## Consequences

Implementation must extend Integration's explicit event/classification/schema allow-list before processing this financial contract, implement the frozen Driver aggregate/API/UI and Tenant persistence, and prove real PostgreSQL plus controlled-file Chromium evidence. Integration retains external delivery state; Driver retains only stable safe references/projections. No second outbox, payroll engine, foreign persistence, distributed transaction, exactly-once claim, or acknowledgement state is permitted.
