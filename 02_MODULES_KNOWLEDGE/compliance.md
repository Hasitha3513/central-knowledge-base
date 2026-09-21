# Compliance Module

## Phase 1: Current MVP Scope

US-72 is `IMPLEMENTATION_IN_PROGRESS / CS04_EVALUATION_ENGINE_COMPLETE / POLICY_APPROVAL_PENDING`. CS01 established framework-neutral structural domain contracts, CS02 established the tenant-isolated persistence schema (Flyway V108) and outbound persistence adapters, CS03 established typed, read-only compliance fact-provider contracts and anti-corruption adapters across Fleet, Freight, and Billing, and CS04 established the deterministic evaluation engine (`EvaluateComplianceUseCase` / `ComplianceEvaluationEngine`) with pure domain check handlers and fail-closed composite rules. It does not activate automated runtime policy enforcement, operational blocking, or UI workflows. Story accounting remains 73/87 and Flyway head is V108.

### Database Schema (Flyway V108)

All tables are strictly tenant-isolated (`tenant_id` leading on PKs and indexes) and owned by the `compliance` module:

- `compliance_policy`: Master definitions of compliance policies with `policy_code`, `name`, `jurisdiction_code`, and `policy_scope`.
- `compliance_policy_version`: Versioned policy configurations (`version_number`, `status` [DRAFT, PUBLISHED, SUPERSEDED, WITHDRAWN], half-open UTC `effective_from` / `effective_to` window, audit references).
- `compliance_policy_rule`: Granular rules per version (`check_type`, `is_mandatory`, `default_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `is_overrideable`, `rule_parameters_json`).
- `compliance_evaluation_record`: Immutable audit logs of compliance evaluations (`target_id`, `operation_type`, `evaluation_instant`, `overall_status` [EVALUATED, POLICY_UNAVAILABLE, POLICY_NOT_EFFECTIVE, SOURCE_FACTS_UNAVAILABLE, UNEVALUATED]).
- `compliance_check_result_record`: Per-rule check results (`check_type`, `evaluation_status`, `evidence_state` [PRESENT_VALID, MISSING, EXPIRED, CONFLICTING, UNKNOWN, NOT_APPLICABLE], `decision_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `reason_codes`).
- `compliance_fact_reference_record`: Minimized immutable evidence references (`source_module`, `fact_type`, `fact_id`, `fact_version`, `effective_at`).

### Inbound & Outbound Ports

- Inbound: `EvaluateComplianceUseCase` -> implemented by `ComplianceEvaluationEngine`.
- Outbound Persistence:
  - `CompliancePolicyPersistencePort` -> implemented by `JdbcCompliancePolicyPersistenceAdapter`
  - `ComplianceEvaluationPersistencePort` -> implemented by `JdbcComplianceEvaluationPersistenceAdapter`
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
