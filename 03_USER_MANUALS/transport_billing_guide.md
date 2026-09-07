# Transport Billing Guide

## Purpose and access

Transport Billing prepares operational charge records for eligible closed Trips or explicit terminal Freight facts. It does not issue tax invoices or claim accounting posting, payment, settlement, or Customer balance authority. Open **Billing → Transport Billing**. `BILLING_VIEW` is required to enter; actions additionally require the matching prepare, approve, finalize, or export permission.

## Create and prepare

Choose **Create billing record**, select `TRIP` or `FREIGHT_ORDER`, enter its UUID and the Tenant currency. The backend accepts only a Closed Trip or explicit Freight-owner Completed/Closed billing fact and validates the same-Tenant active Customer. Duplicate active source billing and stale versions are rejected.

For a draft, choose **Edit commercial facts**. Enter one positive base charge and any explicit non-negative surcharge, penalty, or credit adjustment with bounded provenance. Choose `NOT_SUPPLIED` when no external tax fact exists, or enter the externally supplied taxable amount, rate and tax evidence. Enter operational cost-centre codes totaling 100%. Billing reconciles arithmetic but does not determine legal tax.

## Validate, approve, finalize, and export

Select **Validate** to recheck the unchanged source, calculations, tax shape, cost centres, compliance decision and active controlled export configuration. The preparer cannot approve their own record; an independent user with `BILLING_APPROVE` selects **Approve**. An authorized finalizer then selects **Finalize**. Finalization locks source, Customer, lines, tax, allocations, totals, approval and billing number.

Select **Export controlled JSON** only on a finalized record. The record becomes `EXPORT_REQUESTED`; `EXPORTED` appears only after Integration writes the canonical private UTF-8 JSON file and records matching hash evidence. This confirms delivery only, never Finance import, posting, payment or settlement. Delivery failure leaves the record export-requested for governed Integration retry.

## Cancellation, reversal, history, and limits

Draft/validated records may be cancelled with a reason before approval. Finalized/exported corrections use **Create exact reversal**; the new reversal follows validation, independent approval and finalization and never rewrites the original commercial facts. One effective reversal is allowed. The detail view separates charge categories, tax, cost centres, source trace, approval, delivery evidence and append-only history.

Phase 1 has one source per regular record, one currency, no FX or consolidation, no pricing/tax engine, no Customer master editing, no GL/AR/AP/payment/banking, no manual external-success control, and no live accounting-system claim.
