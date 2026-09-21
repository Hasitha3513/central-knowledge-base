# Compliance Module

## Phase 1: Current MVP Scope

US-72 is `IMPLEMENTATION_IN_PROGRESS / CS03_FACT_INTEGRATION_COMPLETE / POLICY_APPROVAL_PENDING`. CS01 established framework-neutral structural domain contracts, CS02 established the tenant-isolated persistence schema (Flyway V108) and outbound persistence adapters, and CS03 established typed, read-only compliance fact-provider contracts and anti-corruption adapters across Fleet, Freight, and Billing. It does not activate automated runtime policy evaluation, operational blocking, or UI workflows. Story accounting remains 73/87 and Flyway head is V108.

### Database Schema (Flyway V108)

All tables are strictly tenant-isolated (`tenant_id` leading on PKs and indexes) and owned by the `compliance` module:

- `compliance_policy`: Master definitions of compliance policies with `policy_code`, `name`, `jurisdiction_code`, and `policy_scope`.
- `compliance_policy_version`: Versioned policy configurations (`version_number`, `status` [DRAFT, PUBLISHED, SUPERSEDED, WITHDRAWN], half-open UTC `effective_from` / `effective_to` window, audit references).
- `compliance_policy_rule`: Granular rules per version (`check_type`, `is_mandatory`, `default_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `is_overrideable`, `rule_parameters_json`).
- `compliance_evaluation_record`: Immutable audit logs of compliance evaluations (`target_id`, `operation_type`, `evaluation_instant`, `overall_status` [EVALUATED, POLICY_UNAVAILABLE, POLICY_NOT_EFFECTIVE, SOURCE_FACTS_UNAVAILABLE, UNEVALUATED]).
- `compliance_check_result_record`: Per-rule check results (`check_type`, `evaluation_status`, `evidence_state` [PRESENT_VALID, MISSING, EXPIRED, CONFLICTING, UNKNOWN, NOT_APPLICABLE], `decision_effect` [ALLOW, ADVISORY, RESTRICT, BLOCK, UNKNOWN], `reason_codes`).
- `compliance_fact_reference_record`: Minimized immutable evidence references (`source_module`, `fact_type`, `fact_id`, `fact_version`, `effective_at`).

### Outbound Persistence Ports & Adapters

- `CompliancePolicyPersistencePort` -> implemented by `JdbcCompliancePolicyPersistenceAdapter`
- `ComplianceEvaluationPersistencePort` -> implemented by `JdbcComplianceEvaluationPersistenceAdapter`

### Outbound Fact-Query Ports & Anti-Corruption Adapters (CS03)

Compliance defines framework-neutral outbound query ports for operational evidence retrieval, implemented by anti-corruption adapters consuming published root contracts from authoritative modules:

- `VehicleDocumentFactsQuery` -> implemented by `ComplianceVehicleDocumentFactAdapter` consuming `FleetVehicleDocumentLookup` (Fleet root contract).
- `DriverEligibilityFactsQuery` -> implemented by `ComplianceDriverEligibilityFactAdapter` consuming `FleetDriverComplianceLookup` (Fleet root contract).
- `CargoHazmatFactsQuery` -> implemented by `ComplianceCargoHazmatFactAdapter` consuming `FreightCargoHazmatLookup` (Freight root contract).
- `BillingTaxFactsQuery` -> implemented by `ComplianceBillingTaxFactAdapter` consuming `BillingComplianceTaxLookup` (Billing root contract).

Every contract is strictly Tenant- and effective-time-qualified. It carries logical IDs and minimized facts only; Compliance does not query foreign repositories, entities, or database tables. Regional-operation and retention-disposition ports remain deferred in Phase 1.

### Inactive structural domain

- Tenant-qualified policy identity and immutable version references.
- Opaque jurisdiction/scope references and half-open UTC effective-time values.
- Seven structural check identifiers: vehicle document, Driver eligibility, cargo document, hazmat, billing tax fact, regional operation and retention disposition eligibility.
- Evidence states, evaluation statuses and decision-effect vocabulary.
- Minimized immutable source-fact references and evaluation request/check-result/result structures.
- An unavailable or unevaluated check cannot carry `ALLOW`, `BLOCK` or another decision effect.

## Policy and activation hold

D1–D11 remain proposed. Product and qualified policy authority must approve jurisdiction/policy scope, facts, mandatory/advisory classification, effects, precedence, authority, override/appeal, retention, privacy, security and acceptance before runtime operational enforcement. Missing authority/configuration is unavailable/unevaluated; it is neither affirmative clearance nor an automatic operational block.

## Phase 2: Post-MVP / Future Roadmap

Any generic rule language, external rules feed or additional jurisdiction remains separately governed in Phase 2.
