# User Guide: Compliance Workspace (Operator Manual)

## 1. Overview & Operational Notice

The **Compliance Workspace** provides an operator interface for inspecting compliance policies, running on-demand point-in-time evaluations, inspecting evaluation results, and examining data-minimized evidence provenance.

> [!IMPORTANT]
> **Advisory Evaluation Notice:**
> Compliance evaluations in the current release are **advisory evaluation records only**. Operational enforcement is not active in this release; evaluations do not automatically block trips, dispatches, vehicles, or drivers.

---

## 2. Prerequisites & Permissions

Access to compliance features is governed by four granular permissions:

| Permission Code | Capabilities | Description |
| :--- | :--- | :--- |
| `COMPLIANCE_POLICY_VIEW` | View Policies Tab, Policy List, Rule Inspection Modal | Inspect active tenant policies, version numbers, and rule configurations. |
| `COMPLIANCE_EVALUATE` | Run Evaluation Tab, Execute Evaluation Form | Submit point-in-time compliance evaluation requests for operational entities. |
| `COMPLIANCE_EVALUATION_VIEW` | Evaluation Lookup Tab, View Evaluation Summaries | Search evaluation records by UUID and view overall status and check results. |
| `COMPLIANCE_EVIDENCE_VIEW` | Evidence & Provenance Drawer | Inspect detailed source fact metadata, version hashes, and effective timestamps. |

If a user lacks all compliance permissions, navigation to the `/compliance` route displays an Access Denied / Unauthorized message.

---

## 3. Navigation & Workspace Layout

1. Navigate to **Compliance** in the main sidebar (URL: `/compliance`).
2. The workspace is organized into four main tabs:
   - **Overview:** General system notices, capability indicators, and quick links.
   - **Policies:** Complete register of configured tenant compliance policies (requires `COMPLIANCE_POLICY_VIEW`).
   - **Run Evaluation:** Interactive form for executing an on-demand evaluation (requires `COMPLIANCE_EVALUATE`).
   - **Evaluation Lookup:** On-demand lookup tool for retrieving past evaluation summaries by ID (requires `COMPLIANCE_EVALUATION_VIEW`).

---

## 4. Workflows

### 4.1. Inspecting Policies & Version Rules
1. Open the **Policies** tab.
2. Review the list of policies showing Code, Name, Jurisdiction, Scope, and Creation Date.
3. Click **"View Details"** on any policy row to open the policy rules modal.
4. The modal displays published versions, effective half-open date windows, and individual check rules with their mandatory/advisory classification and default effect.
5. *Note: Policy configuration is strictly read-only.*

### 4.2. Executing an On-Demand Compliance Evaluation
1. Open the **Run Evaluation** tab.
2. Enter the target entity's **Operation Type** (e.g. `TRIP_DISPATCH`, `VEHICLE_ALLOCATION`, `DRIVER_ASSIGNMENT`).
3. Enter the target entity's **Operation UUID**.
4. Select the target **Jurisdiction** (e.g., `LK`, `DEFAULT`) and **Policy Scope** (e.g., `OPERATIONAL`, `SAFETY`).
5. Select one or more **Check Types** to execute (e.g., `VEHICLE_DOCUMENT_ELIGIBILITY`, `DRIVER_ELIGIBILITY`, `CARGO_DOCUMENT_ELIGIBILITY`, `HAZMAT_ELIGIBILITY`, `BILLING_TAX_FACT_ELIGIBILITY`).
6. Optionally provide related entity UUIDs (Vehicle ID, Driver ID, Manifest ID, Billing Record ID).
7. Click **"Execute Evaluation"**.
8. The result summary is displayed upon completion.

### 4.3. Understanding Evaluation Outcomes & Effects
Evaluation outcomes are rendered with semantic status tags:

- **EVALUATED:** Evaluation completed successfully against available authoritative facts.
- **POLICY_UNAVAILABLE:** No policy matched the requested jurisdiction or scope.
- **POLICY_NOT_EFFECTIVE:** Policy version is outside its effective time window.
- **SOURCE_FACTS_UNAVAILABLE:** Required authoritative source facts were missing or unreachable.
- **UNEVALUATED:** Evaluation could not reach a determination (e.g. unsupported check type).

Decision effects:
- **ALLOW:** Compliance rules satisfied.
- **ADVISORY:** Non-critical warning (operational actions permitted).
- **RESTRICT / BLOCK:** Blocking policy effect produced (advisory in current release; no automatic operational lockout).

### 4.4. Inspecting Evidence & Provenance
1. From any evaluation summary view, if you hold `COMPLIANCE_EVIDENCE_VIEW`, click **"View Evidence & Provenance"**.
2. A side drawer opens displaying the exact source module, fact type, fact ID, fact version, and validity timestamp.
3. Sensitive raw data (document binary contents, driver medical diagnoses, raw tax formulas) is never stored or exposed.

---

## 5. Security & Session Handling

- Switching tenants or logging out automatically clears all cached evaluation results and policy details.
- Access to evidence is decoupled from summary records to enforce strict segregation of duties.
- If the compliance API is inactive in the deployment environment (`app.compliance.api.enabled=false`), an informative advisory banner explains that the workspace is unavailable.
