# Operations Bounded Context

Status: `US78_COMPLETE / FINAL_ACCEPTANCE_PASS`
Owner: Operations
Decision: `ADR-US78-OPERATIONAL-EXCEPTION-BOUNDARY.md`
Migration: V62; repository head V62

## Phase 1: Current MVP Scope

US-78 implements the cross-domain operational-exception lifecycle. `OperationalExceptionCase` owns confirmed classification/severity, same-Tenant assignment, response/resolution SLA, monotonic escalation, corrective actions, RCA, resolution, closure/reopen, and append-only history. Source modules retain detection, source state/meaning/evidence, and every business correction. Operations has no foreign repository, entity, table, JPA relationship, SQL access, or physical cross-module foreign key.

### Active producers and contracts

- Routing US-22 publishes an allow-listed `OperationalExceptionFactV1` atomically after authoritative disruption creation.
- Delivery US-62 publishes the same contract atomically after authoritative Delivery exception creation.
- Intake is at-least-once through the shared P1-01 `integration_outbox_event`. Consumer `operations-exception-intake` deduplicates by `UNIQUE (tenant_id, source_event_id)`.
- Operations publishes `OPERATIONAL_EXCEPTION_ESCALATED_V1` through P1-01. Notification owns recipient, channel, template, quiet-hour, suppression, delivery attempt, provider retry, and history behavior.
- US-68 remains read-only and viewing Planner data never creates a case.

### Lifecycle and policy

- States: `OPEN`, `ACKNOWLEDGED`, `IN_PROGRESS`, `RESOLVED`, `CLOSED`; reasoned resolution rejection/reopen returns to `IN_PROGRESS`.
- Escalation: monotonic `L0..L3`, not a lifecycle state. Critical intake starts at L1.
- Severity: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`; categories are `OPERATIONAL`, `SAFETY`, `COMPLIANCE`, `CUSTOMER`, `FINANCIAL`, `TECHNICAL`, `SECURITY`.
- Assignment: validated `ROLE_QUEUE` or same-Tenant eligible `USER`; critical cases cannot remain unassigned.
- SLA: 24x7 response/resolution targets of 8h/72h, 4h/24h, 1h/8h, and 15m/2h by ascending severity; at-risk at 75%. A Tenant-aware scanner locks and processes at most 50 due cases per Tenant.
- Corrective actions: `CORRECTIVE`/`PREVENTIVE`, with `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` state.
- RCA: mandatory and independently approved for high/critical. Cause is `PEOPLE`, `PROCESS`, `EQUIPMENT`, `EXTERNAL`, `SYSTEM_DATA`, `ENVIRONMENT`, or `UNKNOWN`.
- Closure: resolved, all required actions complete, required RCA approved, resolution validated, and authorized closer. High/critical closer differs from resolver and RCA approver differs from author.
- Mutable case/action/RCA records use expected versions and database optimistic locking. Stale commands return `OPERATIONAL_EXCEPTION_CONFLICT`.

### Security, privacy, and retention

V62 seeds `OPERATIONAL_EXCEPTION_VIEW`, `OPERATIONAL_EXCEPTION_MANAGE`, `OPERATIONAL_EXCEPTION_ASSIGN`, `OPERATIONAL_EXCEPTION_ESCALATE`, `OPERATIONAL_EXCEPTION_RCA`, `OPERATIONAL_EXCEPTION_CLOSE`, and `OPERATIONAL_EXCEPTION_AUDIT_VIEW`. All 17 literal `/api/v1/operational-exceptions...` routes and effective servlet-context routes are permission protected. Tenant authority comes from authenticated server context; request bodies cannot set Tenant, source identity, case reference, lifecycle/SLA/escalation state, or audit/approval actors.

Operations stores minimized facts and logical evidence/US-83 Document UUID references. It excludes POD media/signatures, medical data, full GPS tracks, financial documents, credentials, OTPs, addresses/contact destinations, provider bodies, and whole source objects. No delete/purge API or hard-coded retention duration exists; `RETENTION_POLICY_EXTERNAL_TO_US78` applies.

## Database schema (V62)

#### Table: `operational_exception_case`

- **Purpose:** Tenant-owned lifecycle aggregate and logical source reference.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID, indexed through all access paths)
- **Uniqueness:** `(id, tenant_id)`, `(tenant_id, case_reference)`, `(tenant_id, source_event_id)`
- **Indexes:** Tenant-leading status/open time, severity/status, assigned user/status, assigned role/status, response due, resolution due, escalation due, and source module/source ID.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Internal case ID |
| `tenant_id` | UUID | NO | - | Tenant scope | Tenant owner |
| `case_reference` | VARCHAR(16) | NO | - | Tenant-local UNIQUE; `OEX-` + 12 Crockford Base32 | Operator reference |
| `source_event_id` | UUID | NO | - | Tenant-local UNIQUE | Intake dedupe ID |
| `source_module` | VARCHAR(24) | NO | - | CHECK `ROUTING`,`DELIVERY` | Source owner |
| `source_type` | VARCHAR(80) | NO | - | Typed-contract allow list | Source exception type |
| `source_id` | UUID | NO | - | Logical reference; no physical FK | Source aggregate ID |
| `occurred_at` | TIMESTAMPTZ | NO | - | Trusted source occurrence | Source time |
| `summary_code` | VARCHAR(80) | NO | - | Source allow list | Safe summary code |
| `correlation_id` | VARCHAR(128) | YES | NULL | Trace only | Correlation ID |
| `category` | VARCHAR(24) | NO | - | Seven-value CHECK | Confirmed category |
| `severity` | VARCHAR(16) | NO | - | Four-value CHECK | Confirmed severity |
| `status` | VARCHAR(24) | NO | - | Five-value CHECK | Lifecycle state |
| `response_due_at` | TIMESTAMPTZ | NO | - | `<= resolution_due_at` | Response target |
| `resolution_due_at` | TIMESTAMPTZ | NO | - | SLA order CHECK | Resolution target |
| `next_escalation_at` | TIMESTAMPTZ | YES | NULL | Indexed due scan | Next escalation time |
| `acknowledged_at` | TIMESTAMPTZ | YES | NULL | Server time | Acknowledgement time |
| `resolved_at` | TIMESTAMPTZ | YES | NULL | Server time | Resolution time |
| `closed_at` | TIMESTAMPTZ | YES | NULL | Server time | Closure time |
| `assignment_type` | VARCHAR(24) | YES | NULL | CHECK role/user | Assignment discriminator |
| `assigned_user_id` | UUID | YES | NULL | Logical Identity reference; target CHECK | Assigned user |
| `assigned_role_code` | VARCHAR(80) | YES | NULL | Logical role reference; target CHECK | Assigned queue |
| `escalation_level` | VARCHAR(8) | NO | `L0` | CHECK `L0`..`L3` | Monotonic level |
| `resolution_note` | VARCHAR(2000) | YES | NULL | Plain text | Resolution note |
| `resolution_result_reference` | VARCHAR(160) | YES | NULL | Logical result reference | Safe result |
| `resolved_by` | UUID | YES | NULL | Logical Identity reference | Resolver |
| `closed_by` | UUID | YES | NULL | Logical Identity reference | Closer |
| `resolution_validated` | BOOLEAN | NO | `FALSE` | - | Validation fact |
| `version` | BIGINT | NO | `0` | CHECK >= 0; optimistic lock | Aggregate version |
| `created_at` | TIMESTAMPTZ | NO | - | Server controlled | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | Server controlled | Update time |

#### Table: `operational_exception_assignment_history`

- **Purpose:** Immutable assignment changes.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)
- **Internal FK:** `(case_id, tenant_id)` -> `operational_exception_case(id, tenant_id)` ON DELETE RESTRICT
- **Index:** `(tenant_id, case_id, occurred_at DESC, id)`

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | History ID |
| `tenant_id` | UUID | NO | - | Tenant scope | Tenant owner |
| `case_id` | UUID | NO | - | Tenant-consistent internal FK | Case |
| `from_type` | VARCHAR(24) | YES | NULL | Assignment-type CHECK | Prior type |
| `from_user_id` | UUID | YES | NULL | Logical Identity reference | Prior user |
| `from_role_code` | VARCHAR(80) | YES | NULL | Logical role reference | Prior queue |
| `to_type` | VARCHAR(24) | YES | NULL | Assignment-type CHECK | New type |
| `to_user_id` | UUID | YES | NULL | Logical Identity reference | New user |
| `to_role_code` | VARCHAR(80) | YES | NULL | Logical role reference | New queue |
| `actor_id` | UUID | NO | - | Logical Identity reference | Actor |
| `actor_username` | VARCHAR(128) | NO | - | Snapshot | Username |
| `reason` | VARCHAR(2000) | NO | - | Plain text | Required reason |
| `occurred_at` | TIMESTAMPTZ | NO | - | Server controlled | Change time |

#### Table: `operational_exception_corrective_action`

- **Purpose:** Case-owned corrective/preventive actions.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)
- **Internal FK:** `(case_id, tenant_id)` -> `operational_exception_case(id, tenant_id)` ON DELETE RESTRICT
- **Uniqueness:** `(id, tenant_id)`
- **Indexes:** `(tenant_id, case_id, created_at, id)` and partial `(tenant_id, status, due_at)` for active work.

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | Action ID |
| `tenant_id` | UUID | NO | - | Tenant scope | Tenant owner |
| `case_id` | UUID | NO | - | Tenant-consistent internal FK | Case |
| `action_type` | VARCHAR(24) | NO | - | CHECK corrective/preventive | Action type |
| `description` | VARCHAR(2000) | NO | - | Plain text | Required action |
| `owner_type` | VARCHAR(24) | NO | - | CHECK role/user | Owner type |
| `owner_user_id` | UUID | YES | NULL | Logical Identity reference; target CHECK | User owner |
| `owner_role_code` | VARCHAR(80) | YES | NULL | Logical role reference; target CHECK | Queue owner |
| `due_at` | TIMESTAMPTZ | YES | NULL | Indexed while active | Due time |
| `status` | VARCHAR(24) | NO | - | Four-value CHECK | State |
| `completed_at` | TIMESTAMPTZ | YES | NULL | Server controlled | Completion time |
| `evidence_reference` | VARCHAR(160) | YES | NULL | Logical source/US-83 reference | Evidence pointer |
| `cancellation_reason` | VARCHAR(2000) | YES | NULL | Plain text | Cancellation reason |
| `version` | BIGINT | NO | `0` | CHECK >= 0; optimistic lock | Action version |
| `created_at` | TIMESTAMPTZ | NO | - | Server controlled | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | Server controlled | Update time |

#### Table: `operational_exception_rca`

- **Purpose:** One bounded root-cause analysis per case.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)
- **Internal FK:** `(case_id, tenant_id)` -> `operational_exception_case(id, tenant_id)` ON DELETE RESTRICT
- **Uniqueness:** `(id, tenant_id)`, `(tenant_id, case_id)`

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | RCA ID |
| `tenant_id` | UUID | NO | - | Tenant scope | Tenant owner |
| `case_id` | UUID | NO | - | Tenant-consistent internal FK | Case |
| `cause_category` | VARCHAR(24) | NO | - | Seven-value CHECK | Cause category |
| `root_cause_code` | VARCHAR(80) | NO | - | Bounded code | Root cause |
| `summary` | VARCHAR(2000) | NO | - | Plain text | Summary |
| `contributing_factors` | VARCHAR(2000) | YES | NULL | Plain text | Factors |
| `author_id` | UUID | NO | - | Logical Identity reference | Author |
| `approver_id` | UUID | YES | NULL | Logical Identity reference; differs from author | Approver |
| `approved_at` | TIMESTAMPTZ | YES | NULL | Paired with approver | Approval time |
| `version` | BIGINT | NO | `0` | CHECK >= 0; optimistic lock | RCA version |
| `created_at` | TIMESTAMPTZ | NO | - | Server controlled | Creation time |
| `updated_at` | TIMESTAMPTZ | NO | - | Server controlled | Update time |

#### Table: `operational_exception_history`

- **Purpose:** Append-only business/security/lifecycle audit history.
- **Primary Key:** `id` (UUID)
- **Multi-Tenant Key:** `tenant_id` (UUID)
- **Internal FK:** `(case_id, tenant_id)` -> `operational_exception_case(id, tenant_id)` ON DELETE RESTRICT
- **Index:** `(tenant_id, case_id, occurred_at DESC, id)`

| Column Name | Data Type | Nullable | Default | Constraints / Logical FK | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | UUID | NO | - | PRIMARY KEY | History ID |
| `tenant_id` | UUID | NO | - | Tenant scope | Tenant owner |
| `case_id` | UUID | NO | - | Tenant-consistent internal FK | Case |
| `action` | VARCHAR(64) | NO | - | Bounded fact code | Action |
| `before_value` | VARCHAR(2000) | YES | NULL | Safe plain text | Prior value |
| `after_value` | VARCHAR(2000) | YES | NULL | Safe plain text | Result value |
| `reason` | VARCHAR(2000) | YES | NULL | Safe plain text | Reason |
| `actor_id` | UUID | NO | - | Logical Identity/system reference | Actor |
| `actor_username` | VARCHAR(128) | NO | - | Snapshot | Username |
| `correlation_id` | VARCHAR(128) | YES | NULL | Trace only | Correlation ID |
| `resulting_version` | BIGINT | NO | - | CHECK >= 0 | Case version |
| `occurred_at` | TIMESTAMPTZ | NO | - | Server controlled | Fact time |

## API and UI

The exact authenticated route family is `/api/v1/operational-exceptions`: list, detail, history, and explicit classify, acknowledge, assign, start, escalate, corrective-action create/start/complete, RCA record/approve, resolve, close, reject-resolution, and reopen commands. List defaults to 20/max 100; history defaults to 50/max 200. Search is limited to case-reference exact/prefix, registered summary code, or exact source UUID. Sort is allow-listed.

The operator feature is `/operations/exceptions` under shared `AppLayout`, with queue filters, safe source facts, badges, assignments, SLA indicators, permission-gated actions/RCA, and timeline. It is not an analytics dashboard or customer portal.

## Verification state

Independent final acceptance passes: focused Operations/Routing/Delivery/durable-publication 41/41; concurrency 6/6; PostgreSQL V1-V62 and Operations acceptance 3/3 against only `transport_logistics_acceptance`; related regressions 84/84; complete Maven 1,296 tests with 0 failures/errors and 15 skips in 05:06; architecture 46/46; real PostgreSQL-backed Chromium 6/6 in 19.3 seconds; TypeScript, Vitest 61 files/261 tests, production build, changed-file lint, Checkstyle, PMD, and SpotBugs. The 71 global ESLint errors remain confined to unchanged Delivery files and US-78 introduces none. US-78 is COMPLETE; overall accounting is 67/87 accepted and 20/87 remaining.

## Phase 2: Post-MVP / Future Roadmap

US-38 Fuel, US-55 Tracking, Trip, Cargo, Driver/Fleet, Compliance, and Integration may publish only after exact producer semantics are accepted. US-86 owns coordinated disruption constraints/replanning separately. Merge/parent-child graphs, configurable calendars, person load-balancing, customer visibility, analytics, generic manual incidents, and automated retention remain deferred.
