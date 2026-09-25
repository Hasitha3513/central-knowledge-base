# System Module Knowledge Base

## 1. Module Overview
The `system` module owns cross-cutting system resilience, platform health monitoring, global failure degraded mode coordination (US-84), and data integrity anomaly detection, quarantine, and audited owner correction workflows (US-85).

---

## 2. Database Schema (PostgreSQL Flyway V112)

### Table: `data_integrity_finding`
Stores detected data anomalies across master data, meter sequences, trip telemetry attribution, duplicate keys, and orphaned transactional references.

| Column | Type | Nullable | Default | Constraints / Description |
| :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | | Primary Key |
| `tenant_id` | UUID | NO | | Multi-tenant isolation partition |
| `finding_key` | VARCHAR(64) | NO | | Deterministic SHA-256 finding hash |
| `finding_type` | VARCHAR(64) | NO | | `DUPLICATE_RECORD`, `ORPHANED_TRANSACTION`, `ODOMETER_MISMATCH`, `GPS_TRIP_MISMATCH`, `INVALID_MASTER_DATA` |
| `category` | VARCHAR(64) | NO | | `DUPLICATE`, `ORPHAN`, `MISMATCH`, `MASTER_DATA`, `REFERENCE_INTEGRITY` |
| `severity` | VARCHAR(32) | NO | | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `status` | VARCHAR(32) | NO | | `DETECTED`, `QUARANTINED`, `UNDER_REVIEW`, `RESOLVED`, `IGNORED` |
| `source_module` | VARCHAR(64) | NO | | Originating module identifier (e.g. `fleet`, `trip`, `organization`) |
| `resource_type` | VARCHAR(64) | NO | | Entity type (e.g. `VEHICLE_ODOMETER`, `TELEMETRY_RECORD`, `CUSTOMER`) |
| `resource_id` | UUID | YES | | Entity primary key |
| `description` | VARCHAR(1000) | NO | | Human-readable explanation of detected condition |
| `details_json` | TEXT | YES | | Structured diagnostic payload |
| `proposed_corrective_action` | VARCHAR(1000) | YES | | Recommended remediation instruction |
| `detected_at` | TIMESTAMPTZ | NO | | Detection timestamp (UTC) |
| `reviewed_by` | UUID | YES | | Auditor / operator actor ID |
| `reviewed_at` | TIMESTAMPTZ | YES | | Review timestamp (UTC) |
| `review_notes` | VARCHAR(2000) | YES | | Operational review comments |
| `resolution_action` | VARCHAR(64) | YES | | `MANUAL_CORRECTION`, `AUTO_RECONCILED`, `ACCEPTED_EXCEPTION`, `QUARANTINE_LIFTED` |
| `version` | BIGINT | NO | 0 | Optimistic locking counter |
| `created_at` | TIMESTAMPTZ | NO | | Row creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | | Row update timestamp |

#### Constraints & Indexes:
- `uq_data_integrity_finding_tenant_id`: `UNIQUE (id, tenant_id)`
- `uq_data_integrity_finding_key`: `UNIQUE (tenant_id, finding_key)`
- `idx_data_integrity_finding_tenant_status`: `ON data_integrity_finding (tenant_id, status, category, detected_at DESC)`
- `idx_data_integrity_finding_resource`: `ON data_integrity_finding (tenant_id, resource_type, resource_id)`

---

## 3. Inbound Ports & REST APIs

### Data Integrity Management
- `POST /api/v1/system/integrity/scans` (Execute bounded tenant integrity scan)
- `GET /api/v1/system/integrity/findings` (List findings with tenant isolation, category, and status filters)
- `GET /api/v1/system/integrity/findings/{findingId}` (Get single finding)
- `POST /api/v1/system/integrity/findings/{findingId}/quarantine` (Quarantine anomaly)
- `POST /api/v1/system/integrity/findings/{findingId}/review` (Start operator review)
- `POST /api/v1/system/integrity/findings/{findingId}/resolve` (Record owner-mediated resolution)
- `POST /api/v1/system/integrity/findings/{findingId}/ignore` (Acknowledge operational waiver)

### System Resilience & Degraded Mode (US-84)
- `GET /api/v1/system/resilience/health` (Probe system health overview)
- `GET /api/v1/system/resilience/degraded-mode` (Current degraded mode status)
- `POST /api/v1/system/resilience/degraded-mode/activate` (Activate controlled degraded mode)
- `POST /api/v1/system/resilience/degraded-mode/deactivate` (Restore normal operations)

---

## 4. Published Integration & Audit Events
- `DataIntegrityFindingDetectedV1` (Published when a new anomaly is registered)
- `DataIntegrityFindingQuarantinedV1` (Published when finding is placed in quarantine)
- `DataIntegrityFindingResolvedV1` (Published when finding is resolved via audited action)


## 5. Development Sample-Data Bootstrap
When the application runs with the `postgres` or `docker` profile and `app.dev.sample-data.enabled=true`, startup loads the single canonical, idempotent PostgreSQL sample fixture. It supplies tenant-scoped demonstration records for the original core modules plus Integration, operational exceptions, Driver payroll input, Transport Billing, delivery self-service, live Tracking, Compliance, operational disruptions, and the data-integrity queue. Runtime-generated control records such as nonces, command inboxes, leases, retry attempts, reviews, corrections, and episode/audit histories are intentionally left for their owning workflows so demonstration data cannot suppress or falsify operational work. The fixture is development-only, uses stable identifiers, does not activate external connections, and does not change production migrations or domain contracts. The H2 profile continues to load only its H2-specific fixture.
