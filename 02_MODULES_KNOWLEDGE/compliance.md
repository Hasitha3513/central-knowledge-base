# Compliance Module

## Phase 1: Current MVP Scope

US-72 is `COMPLETE_INACTIVE / POLICY_APPROVAL_PENDING` (Final Acceptance: `US-72-ENFORCE-COMPLIANCE-FINAL-ACCEPTANCE-001.md`). CS01 established framework-neutral structural domain contracts, CS02 established the tenant-isolated persistence schema (Flyway V108) and outbound persistence adapters, CS03 established typed, read-only compliance fact-provider contracts and anti-corruption adapters across Fleet, Freight, and Billing, CS04 established the deterministic evaluation engine (`EvaluateComplianceUseCase` / `ComplianceEvaluationEngine`) with pure domain check handlers, CS05 established the default-off read/evaluate REST API (`ComplianceController`), RBAC/ABAC authorization (`SecuredComplianceApiUseCase`), four catalogued ungranted permissions, and immutable audit logging (`compliance_audit_event`, Flyway V109), and CS06 established the permission-aware, read-only operator Compliance workspace in React + TypeScript + Ant Design / Refine. It does not activate automated runtime policy enforcement or operational blocking. Current MVP accounting is 76/87; branch migration heads may be later than V109 without changing the US-72 schema.

### Database Schema (Flyway V108 & V109)

All tables are strictly tenant-isolated (`tenant_id` leading on PKs and indexes) and owned by the `compliance` module:

- `compliance_policy`: Master definitions of compliance policies with `policy_code`, `name`, `jurisdiction_code`, and `policy_scope`.
- `compliance_policy_version`: Versioned policy configurations (`version_number`, `status` [DRAFT, PUBLISHED, SUPERSEDED, WITHDRAWN], half-open UTC `effective_from` / `effective_to` window, audit references).
- `compliance_policy_rule`: Granular rules per version (`check_type`, `is_mandatory`, `default_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `is_overrideable`, `rule_parameters_json`).
- `compliance_evaluation_record`: Immutable audit logs of compliance evaluations (`target_id`, `operation_type`, `evaluation_instant`, `overall_status` [EVALUATED, POLICY_UNAVAILABLE, POLICY_NOT_EFFECTIVE, SOURCE_FACTS_UNAVAILABLE, UNEVALUATED]).
- `compliance_check_result_record`: Per-rule check results (`check_type`, `evaluation_status`, `evidence_state` [PRESENT_VALID, MISSING, EXPIRED, CONFLICTING, UNKNOWN, NOT_APPLICABLE], `decision_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `reason_codes`).
- `compliance_fact_reference_record`: Minimized immutable evidence references (`source_module`, `fact_type`, `fact_id`, `fact_version`, `effective_at`).
- `compliance_audit_event` (V109): Append-only immutable audit trail protected by `guard_compliance_audit_immutable()` trigger (`id`, `tenant_id`, `actor_id`, `action_code`, `resource_id`, `resource_type`, `correlation_id`, `payload`, `created_at`).

### Permissions Matrix & Operator Capabilities (V109 & CS06)

- `COMPLIANCE_EVALUATE`: Allows executing point-in-time policy evaluations via `POST /api/v1/compliance/evaluations` and the Run Evaluation workspace form.
- `COMPLIANCE_EVALUATION_VIEW`: Allows inspecting evaluation summaries via `GET /api/v1/compliance/evaluations/{id}` and the Evaluation Lookup tab.
- `COMPLIANCE_EVIDENCE_VIEW`: Allows inspecting granular check results and fact references via `GET /api/v1/compliance/evaluations/{id}/evidence` in a segregated drawer.
- `COMPLIANCE_POLICY_VIEW`: Allows reading tenant compliance policies and version rules via `GET /api/v1/compliance/policies` and `GET /api/v1/compliance/policies/{id}`.

### Frontend Operator Workspace Architecture (CS06)

- Workspace Route: `/compliance`
- Components:
  - `ComplianceWorkspacePage`: Tabbed operator dashboard with persistent advisory notices.
  - `CompliancePolicyList`: Read-only summary table of tenant compliance policies.
  - `CompliancePolicyDetailModal`: Read-only inspector for policy version rules.
  - `ComplianceEvaluationForm`: Controlled form with client-side UUID validation.
  - `ComplianceEvaluationSummary`: Presentation of evaluation outcome and per-check results.
  - `ComplianceEvidenceDrawer`: Segregated drawer for data-minimized fact provenance.
- Multi-Tenancy: `useComplianceSession()` invalidates caches on tenant/session changes.
- Read-Only Guard: Strictly no mutating policy authoring, form editing, or publishing controls.

### Inbound & Outbound Ports

- Inbound:
  - `EvaluateComplianceUseCase` -> implemented by `ComplianceEvaluationEngine`.
  - `ComplianceApiUseCase` -> implemented by `ComplianceApiService` (secured by `SecuredComplianceApiUseCase`).
- Outbound Persistence:
  - `CompliancePolicyPersistencePort` -> implemented by `JdbcCompliancePolicyPersistenceAdapter`.
  - `ComplianceEvaluationPersistencePort` -> implemented by `JdbcComplianceEvaluationPersistenceAdapter`.
  - `ComplianceAuditPort` -> implemented by `JdbcComplianceAuditAdapter`.
- Outbound Fact Queries:
  - `VehicleDocumentFactsQuery` -> implemented by `ComplianceVehicleDocumentFactAdapter` consuming `FleetVehicleDocumentLookup`.
  - `DriverEligibilityFactsQuery` -> implemented by `ComplianceDriverEligibilityFactAdapter` consuming `FleetDriverComplianceLookup`.
  - `CargoHazmatFactsQuery` -> implemented by `ComplianceCargoHazmatFactAdapter` consuming `FreightCargoHazmatLookup`.
  - `BillingTaxFactsQuery` -> implemented by `ComplianceBillingTaxFactAdapter` consuming `BillingComplianceTaxLookup`.

### Pure Domain Check Handlers (CS04)

- `VehicleDocumentEligibilityCheckHandler`: Evaluates vehicle document status and validity interval.
- `DriverEligibilityCheckHandler`: Evaluates driver licence status/validity and medical fitness.
- `CargoDocumentEligibilityCheckHandler`: Evaluates customs documentation for applicable freight.
- `HazmatEligibilityCheckHandler`: Evaluates dangerous goods classifications and declarations.
- `BillingTaxFactEligibilityCheckHandler`: Evaluates supplied tax status and consistency.
- `UnsupportedCheckHandler`: Explicit fail-safe for deferred checks (`REGIONAL_OPERATION_ELIGIBILITY`, `RETENTION_DISPOSITION_ELIGIBILITY`) returning `UNEVALUATED`.

### Policy and activation hold

D1–D11 remain pending qualified approval. Product and qualified policy authority must approve jurisdiction/policy scope, facts, mandatory/advisory classification, effects, precedence, authority, override/appeal, retention, privacy, security and acceptance before production evaluation or operational enforcement. Missing authority/configuration is unavailable/unevaluated; it is neither affirmative clearance nor an automatic operational block.

V114 now supplies the governed default-off policy lifecycle and initial permission-assignment mechanisms through separate Identity and Compliance owners. Enabling `app.compliance.api.enabled` still does not publish a policy, grant permissions, or authorize a jurisdiction. The provisional V115 exact-target override lifecycle is technically implemented; appeals (`APPEAL_PHASE1_DEFERRED`), formal policy authority decisions, named operational inputs and production acceptance remain open. The implementation does not reuse V113 US-87 tables or allowlists and does not create a generic governance platform.

## Phase 2: Post-MVP / Future Roadmap

Any generic rule language, external rules feed or additional jurisdiction remains separately governed in Phase 2.


## V114 Governance Operations (Technically Complete, Production Inactive)

Flyway V114 adds separate default-off signed one-shot boundaries. Identity owns permission grant/removal for only
the four existing Compliance codes; Compliance owns immutable policy publication, forward replacement,
withdrawal and read-back. Neither runner is HTTP-accessible. Both require explicit enablement, safe non-web
runtime, scheduling and Kafka listeners disabled, an allow-listed deployment key ID, a signed bounded command
and Tenant-qualified existing records. Automatic grants remain zero.

The prior text describing publication and initial permission assignment as missing is superseded by V114.
Qualified D1-D11 authority decisions, named Tenant/role/key inputs, controlled activation, operational
acceptance and sign-off remain missing. The API remains default-off and there is still no operational blocking,
override/appeal workflow or production policy.

### Table: `identity_compliance_governance_command` (Identity-owned)

- **Purpose:** Immutable idempotency/result record for signed Compliance permission grant/removal.
- **Primary Key:** `(tenant_id, command_id)`
- **Multi-Tenant Key:** `tenant_id`

| Column | Type | Nullable | Constraints / description |
|---|---|---:|---|
| tenant_id, command_id | UUID | NO | Composite primary key |
| operation | VARCHAR(40) | NO | GRANT_COMPLIANCE_PERMISSIONS or REMOVE_COMPLIANCE_PERMISSIONS |
| canonical_fingerprint | CHAR(64) | NO | lowercase SHA-256 |
| key_id | VARCHAR(80) | NO | approved deployment key reference |
| approval_reference | VARCHAR(128) | NO | provenance only |
| issued_at, expires_at, completed_at | TIMESTAMPTZ | NO | maximum ten-minute command validity |
| result_code | VARCHAR(64) | NO | committed result |
| affected_role_ids | JSONB | NO | UUID array only |
| retain_until | TIMESTAMPTZ | NO | completed_at plus 180 days |

Retention index: `(retain_until, tenant_id, command_id)`. Updates and deletes are trigger-rejected.

### Table: `identity_compliance_governance_audit_event` (Identity-owned)

Tenant-qualified immutable minimized audit with `(tenant_id,id)` primary key, command identity, bounded
operation/key/provenance/outcome/result fields, `occurred_at`, and `retain_until = occurred_at + 180 days`.
History index: `(tenant_id, occurred_at DESC, id DESC)`.

### Table: `compliance_governance_command`

Same replay/provenance/time/retention shape as the Identity command table, with `affected_ids` and operations
`PUBLISH_POLICY_VERSION`, `REPLACE_POLICY_VERSION`, or `WITHDRAW_POLICY_VERSION`. Primary key is
`(tenant_id,command_id)`; retention index is `(retain_until,tenant_id,command_id)`; rows are immutable.

### Table: `compliance_policy_version_closure`

- **Purpose:** Immutable effective-dated closure for a published version.
- **Primary Key:** `(tenant_id,id)`
- **Relationships:** same-Tenant FKs to policy version and governance command; one closure per version.
- **Fields:** policy_version_id, effective_at, reason_code (REPLACED/WITHDRAWN), approval_reference,
  governance_command_id, created_at, retain_until.
- **Index:** `(tenant_id,policy_version_id,effective_at DESC,id DESC)`.
- **Behavior:** new selection excludes a version whose closure is effective at processing, including a
  backdated new request; committed historical evaluations remain readable.

### Table: `compliance_governance_audit_event`

Tenant-qualified immutable minimized audit with `(tenant_id,id)` primary key. It records command identity,
operation, key reference, approval provenance, outcome/result, occurred_at and 180-day retain_until. History
index: `(tenant_id,occurred_at DESC,id DESC)`.

V114 also trigger-protects published `compliance_policy_version` and its rules from update/delete. Frontend
session or Tenant transitions invalidate Compliance queries and clear lookup/result/evidence state.


## Governance code-review remediation

The US-72 remediation serializes policy selection/evaluation/immutable decision persistence with withdrawal through one Tenant/policy transaction-scoped PostgreSQL advisory lock. Replacement is restricted to the expected latest version of the same Tenant/policy and may overlap only that exact prior interval at the forward closure boundary. Policy read-back returns the exact policy/version identity, finite effective interval, ordered bounded rule summary, and optional closure state. Compliance permission read-back returns only the four approved permissions for requested same-Tenant roles. V114 and all historical migrations remain unchanged; Compliance remains default-off and unaccepted.


## V115 Provisional D7 Override Governance (Technically Complete, Production Inactive)

V115 implements an exact Tenant/policy-version/check/operation-type/operation-ID override lifecycle through
the existing signed default-off one-shot governance runner. Only `RESTRICT` or `BLOCK` may be overridden, and
the derived effective result is `ADVISORY` during `[valid_from, valid_until)`, for at most four hours. Requester
and approver, and original evaluator and approver, must differ. Formal D1–D11 approval is still pending.

#### Table: `compliance_override_governance_command`

- **Purpose:** Immutable idempotency/result record for successful signed D7 mutation commands.
- **Primary Key:** (`tenant_id`, `command_id`)
- **Multi-Tenant Key:** `tenant_id`

| Column | Type | Nullable | Constraints / description |
|---|---|---:|---|
| `tenant_id` | UUID | NO | Tenant scope |
| `command_id` | UUID | NO | Stable command identity |
| `operation` | VARCHAR(40) | NO | REQUEST/APPROVE/REJECT/REVOKE only |
| `canonical_fingerprint` | CHAR(64) | NO | Lowercase SHA-256 hex |
| `key_id` | VARCHAR(80) | NO | Trusted signing-key reference |
| `approval_reference` | VARCHAR(128) | NO | Approval provenance |
| `issued_at`, `expires_at`, `completed_at` | TIMESTAMPTZ | NO | Command interval <=10 minutes; completion not before issue |
| `result_code` | VARCHAR(64) | NO | Minimized result |
| `affected_ids` | JSONB | NO | JSON array of logical UUIDs |
| `retain_until` | TIMESTAMPTZ | NO | Exactly completion +180 days |

Index: `(retain_until, tenant_id, command_id)`. Update/delete is trigger-rejected.

#### Table: `compliance_override`

- **Purpose:** Exact-target D7 request and immutable-material lifecycle.
- **Primary Key:** (`tenant_id`, `id`)
- **Multi-Tenant Key:** `tenant_id`

| Column | Type | Nullable | Constraints / description |
|---|---|---:|---|
| `tenant_id`, `id` | UUID | NO | Tenant-qualified identity |
| `policy_version_id` | UUID | NO | Same-Tenant FK to policy version |
| `check_type` | VARCHAR(64) | NO | Seven frozen Compliance check identifiers |
| `operation_type` | VARCHAR(80) | NO | Bounded code format |
| `operation_id` | UUID | NO | Exact operation target |
| `source_evaluation_id` | UUID | NO | Same-Tenant FK to immutable evaluation |
| `original_effect` | VARCHAR(16) | NO | RESTRICT or BLOCK |
| `override_effect` | VARCHAR(16) | NO | ADVISORY only |
| `reason_code` | VARCHAR(40) | NO | Three frozen D7 reasons |
| `note` | VARCHAR(256) | YES | Optional minimized note |
| `requester_id`, `original_evaluator_id` | UUID | NO | SoD actors |
| `approver_id` | UUID | YES | Must differ from requester and evaluator |
| `valid_from`, `valid_until` | TIMESTAMPTZ | NO | Positive half-open interval <=PT4H |
| `state` | VARCHAR(16) | NO | REQUESTED/APPROVED/REJECTED/REVOKED |
| `decision_at`, `revoked_at` | TIMESTAMPTZ | YES | Required by lifecycle state |
| `revoked_by` | UUID | YES | Required only when revoked |
| `approval_reference` | VARCHAR(128) | YES | Required when approved |
| `request_command_id` | UUID | NO | Same-Tenant deferred FK to command |
| `decision_command_id`, `revocation_command_id` | UUID | YES | Same-Tenant deferred command FKs |
| `created_at` | TIMESTAMPTZ | NO | Creation time |
| `version` | BIGINT | NO | Optimistic lifecycle version; default 0 |

Tenant-leading indexes support exact active-target lookup, operation history and pending work. A trigger freezes
material fields, permits only REQUESTED→APPROVED/REJECTED and APPROVED→REVOKED, and rejects deletion.

#### Table: `compliance_override_audit_event`

- **Purpose:** Immutable minimized D7 command/audit outcome.
- **Primary Key:** (`tenant_id`, `id`)
- **Multi-Tenant Key:** `tenant_id`

| Column | Type | Nullable | Constraints / description |
|---|---|---:|---|
| `tenant_id`, `id` | UUID | NO | Tenant-qualified audit identity |
| `override_id` | UUID | YES | Same-Tenant FK when the override exists |
| `command_id`, `actor_user_id` | UUID | NO | Command and actor references |
| `action` | VARCHAR(32) | NO | REQUESTED/APPROVED/REJECTED/REVOKED/READ_BACK/DRY_RUN vocabulary |
| `key_id` | VARCHAR(80) | NO | Signing-key reference, never key material |
| `approval_reference` | VARCHAR(128) | NO | Approval provenance |
| `outcome` | VARCHAR(24) | NO | Minimized outcome vocabulary |
| `result_code` | VARCHAR(64) | NO | Safe result code |
| `occurred_at`, `retain_until` | TIMESTAMPTZ | NO | Retention exactly 180 days |

Index: `(tenant_id, override_id, occurred_at DESC, id DESC)`. Update/delete is trigger-rejected.

## Evaluation Authorization Closure (EXT-COMP-02-REM-07)

The final packaged-browser evaluation 403 was traced to the controlled Playwright origin (`http://localhost:5174`) being absent from that test backend's CORS environment, not to Compliance RBAC. Production authorization remains least-privilege: `POST /api/v1/compliance/evaluations` requires `COMPLIANCE_EVALUATE`; view-only, policy-view-only and evidence-view-only users remain forbidden; unauthenticated access remains rejected; and bounded source facts remain Tenant-qualified behind published module ports. The correction configures only the Playwright backend origin and adds response-level regression assertions. Fresh evidence passed the direct permission matrix, focused Compliance/architecture 92/92, real PostgreSQL-backed Chromium 6/6, and complete Maven 2,191/2,191. No production CORS default, permission, automatic grant, API, migration or policy activation changed. Software status is `US72_ACTIVATION_EXECUTION_READY`; authority status remains `US72_POLICY_APPROVAL_PENDING`, and appeals remain `APPEAL_PHASE1_DEFERRED`.
