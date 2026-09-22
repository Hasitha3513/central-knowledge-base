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
