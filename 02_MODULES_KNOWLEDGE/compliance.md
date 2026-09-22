# Compliance Module

## Phase 1: Current MVP Scope

US-72 is `COMPLETE_INACTIVE / POLICY_APPROVAL_PENDING` (Final Acceptance: `US-72-ENFORCE-COMPLIANCE-FINAL-ACCEPTANCE-001.md`). CS01 established framework-neutral structural domain contracts, CS02 established the tenant-isolated persistence schema (Flyway V108) and outbound persistence adapters, CS03 established typed, read-only compliance fact-provider contracts and anti-corruption adapters across Fleet, Freight, and Billing, CS04 established the deterministic evaluation engine (`EvaluateComplianceUseCase` / `ComplianceEvaluationEngine`) with pure domain check handlers and fail-closed composite rules, CS05 established the secure REST API (`ComplianceController`), RBAC/ABAC authorization (`SecuredComplianceApiUseCase`), four catalogued ungranted permissions, and immutable audit logging (`compliance_audit_event`, Flyway V109), and CS06 established the permission-aware, read-only operator Compliance workspace in React + TypeScript + Ant Design / Refine. It does not activate automated runtime policy enforcement or operational blocking. Story accounting remains 73/87 and Flyway head is V109.

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

D1–D11 remain proposed. Product and qualified policy authority must approve jurisdiction/policy scope, facts, mandatory/advisory classification, effects, precedence, authority, override/appeal, retention, privacy, security and acceptance before runtime operational enforcement. Missing authority/configuration is unavailable/unevaluated; it is neither affirmative clearance nor an automatic operational block.

## Phase 2: Post-MVP / Future Roadmap

Any generic rule language, external rules feed or additional jurisdiction remains separately governed in Phase 2.
