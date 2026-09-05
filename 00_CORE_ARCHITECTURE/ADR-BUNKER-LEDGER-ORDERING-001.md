# ADR: Canonical Bunker Ledger Ordering

**Decision ID:** `BUNKER-LEDGER-ORDERING-AUTHORIZATION-001`  
**Status:** APPROVED  
**Date:** 2026-09-05  
**Owner:** Fuel  
**Classification:** Non-story technical governance

## Context

`bunker_stock_movement` currently has business `occurred_at`, audit `created_at`, and a random UUID. None represents the serialized order in which stock mutations update a Tank. Business time may be backdated, creation time is captured before the Tank lock, and random UUID order is semantically meaningless. Timestamp/UUID sorting is therefore rejected as the canonical ledger order.

## Decision

Add an internal `ledger_sequence BIGINT`, monotonically increasing within `(tenant_id, tank_id)`. Allocate `MAX(ledger_sequence) + 1` only after acquiring the existing authoritative Tank pessimistic write lock and persist it in the same ACID transaction as stock validation, Tank balance update, and movement insertion. No global sequence or Tank counter is authorized.

The highest committed sequence is the latest serialized Tank mutation. Its resulting balance must equal the Tank balance at that mutation. `occurred_at` remains business/source time; `created_at` remains persistence/audit time. Clients neither provide nor depend on the sequence.

The forward migration, V65 only if still free, may add/backfill the column, make it `NOT NULL`, enforce `UNIQUE (tenant_id, tank_id, ledger_sequence)`, and add `(tenant_id, tank_id, ledger_sequence DESC)`. Historical migrations remain immutable. Latest-first ledger queries order primarily by sequence.

## Legacy canonicalization

Within each Tenant/Tank, legacy rows are deterministically assigned contiguous sequences by `occurred_at ASC, created_at ASC, id ASC`. This is explicitly a migration canonical order, not a claim about historical commit order. Quantity, type, balance, timestamps, actor, references, and row population are never rewritten.

The migration fails closed for missing or cross-Tenant Tank ownership, invalid/duplicate/non-contiguous sequence results, logical duplicates or invalid quantity/balance facts, a canonical ledger tail inconsistent with current Tank stock, or non-zero Tank stock without movement history. Reconciliation of such data requires separate governance; synthetic movements are forbidden.

## Consequences and verification

All opening balance, purchase receipt, Fuel Issue, adjustment, and transfer movements share the allocator. Rollback commits neither movement nor sequence. The index keeps allocation/query Tank-scoped and bounded; sequences are independent across Tanks and Tenants and are not idempotency keys.

Remediation must verify same-time and backdated movements, receipt/issue and same-type concurrency, rollback, monotonic independence, uniqueness, legacy backfill/fail-closed cases, latest balance equality, clean PostgreSQL migration from V1, focused/full regression, architecture, static analysis, and diff hygiene using only `transport_logistics_acceptance`.

US-35 remains `IMPLEMENTATION_COMPLETE / ACCEPTANCE_BLOCKED`; accounting remains 68/87 accepted and 19/87 remaining. Next task: `US-35-FUEL-CARDS-ACCEPTANCE-REMEDIATION-002`.
