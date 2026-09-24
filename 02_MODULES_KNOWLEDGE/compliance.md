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

V114 now supplies the governed default-off policy lifecycle and initial permission-assignment mechanisms through separate Identity and Compliance owners. Enabling `app.compliance.api.enabled` still does not publish a policy, grant permissions, or authorize a jurisdiction. Override/appeal, policy authority decisions, named operational inputs and production acceptance remain open. The implementation does not reuse V113 US-87 tables or allowlists and does not create a generic governance platform.

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
