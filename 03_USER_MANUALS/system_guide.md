# User Manual: System Resilience & Data Integrity

## 1. Overview
The **System Resilience & Data Integrity Console** provides system administrators and operational supervisors with centralized visibility and control over platform operational health (US-84) and domain data consistency (US-85).

---

## 2. Data Integrity Anomaly Lifecycle

```
       [ DETECTED ]
         /      \
  (quarantine)   \ (start review)
       /          \
      v            v
[ QUARANTINED ] -> [ UNDER_REVIEW ]
      \                 /
   (owner correction / waiver)
        \             /
         v           v
    [ RESOLVED ]  [ IGNORED ]
```

### Anomaly Classifications:
1. **Duplicate Records (`DUPLICATE`):** Duplicate business keys or identical external identifier bindings.
2. **Orphaned Transactions (`ORPHAN`):** Transaction records missing parent references.
3. **Mismatches (`MISMATCH`):** Odometer rollback anomalies, negative fuel progression, or GPS trip attribution conflicts.
4. **Master Data References (`MASTER_DATA`):** Inactive or missing customer, location, or vendor records used in active operational entities.

---

## 3. Operational Workflows

### 3.1 Triggering a Tenant Integrity Scan
1. Open the System Management Console and navigate to **Data Integrity -> Scans**.
2. Select the target scan categories (or Full Scan).
3. Enable **Auto-Quarantine Critical** if automatic isolation of critical severity anomalies is desired.
4. Trigger scan: The platform executes all registered domain validators and reports summary metrics.

### 3.2 Reviewing & Resolving Findings
1. Navigate to **Data Integrity -> Findings Queue**.
2. Filter by status (`DETECTED`, `QUARANTINED`, `UNDER_REVIEW`).
3. Click a finding to view its diagnostic payload (`detailsJson`) and proposed corrective action.
4. **Quarantine:** Isolate the finding if it represents active financial or operational risk.
5. **Remediate:** Perform the required correction inside the owning domain (e.g., submit reading correction in Fleet module).
6. **Resolve:** Mark the finding as `RESOLVED` with the selected resolution action (`MANUAL_CORRECTION`, `AUTO_RECONCILED`, `ACCEPTED_EXCEPTION`, `QUARANTINE_LIFTED`) and auditor notes.

---

## 4. Permissions & RBAC
- `SYSTEM_INTEGRITY_READ`: View integrity findings and scan summaries.
- `SYSTEM_INTEGRITY_MANAGE`: Trigger scans, quarantine findings, and record resolutions.
- `SYSTEM_RESILIENCE_MANAGE`: Activate/deactivate controlled degraded modes during major outages.


## 5. User-Risk Governance and Human Review

The US-87 first wave is an inactive, advisory-only Identity capability for repeated permission-ceiling denial
signals. An indicator is not proof of misconduct. It cannot automatically suspend or terminate a user, revoke
sessions, change permissions or MFA, block work, change compensation or create disciplinary action.

Production use requires accountable authorities to execute the G1–G10 governance package, provide a finite
Tenant rule interval, approve exact role/permission, reviewer and appeal-decider rosters, and identify key
custody, deployment, monitoring, rollback and acceptance owners. The governance runner and internal-review
surface are separate and default off. The four permissions are `USER_RISK_VIEW`,
`USER_RISK_EVIDENCE_VIEW`, `USER_RISK_REVIEW`, and `USER_RISK_APPEAL_DECIDE`; none is granted
automatically.

The review lifecycle preserves the original finding and disposition. One internal appeal may be requested
within 30 days, and its decider must differ from the subject and original reviewer. Immutable rule withdrawal
stops governed future evaluation without deleting historical findings, evidence, reviews, appeals or audit.
Activation manifests contain references and non-secret key IDs only—never credentials or signing material.

## 6. Local Demonstration Data
A local Docker or PostgreSQL startup with sample data enabled loads one canonical fixture containing representative records in the newer Phase 1 workspaces as well as the original core modules. Operators can immediately inspect Integration configuration, operational exceptions, payroll batches, billing records, delivery self-service, live vehicle positions, compliance evaluation evidence, active disruptions, and data-integrity findings. These records are synthetic, tenant-scoped, and intended only for local demonstration and testing; they do not indicate a real external integration, compliance approval, disruption, financial posting, or production telemetry event. Repeated startup is safe because the records use stable identifiers and conflict-safe inserts. Workflow-control rows such as nonces, retries, reviews, corrections, and audit histories appear only after the corresponding workflow runs.
