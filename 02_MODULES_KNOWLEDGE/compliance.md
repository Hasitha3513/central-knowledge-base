# Compliance Module

## Phase 1: Current MVP Scope

US-72 is `IMPLEMENTATION_IN_PROGRESS / CS01_COMPLETE_INACTIVE /
POLICY_APPROVAL_PENDING`. CS01 establishes framework-neutral structural contracts only. It does not
activate a jurisdiction, policy, evaluator, persistence model, permission, API, event, frontend workflow
or operational enforcement. Story accounting remains 73/87 and Flyway remains V105.

### Inactive structural domain

- Tenant-qualified policy identity and immutable version references.
- Opaque jurisdiction/scope references and half-open UTC effective-time values.
- Seven structural check identifiers: vehicle document, Driver eligibility, cargo document, hazmat,
  billing tax fact, regional operation and retention disposition eligibility.
- Evidence states, evaluation statuses and decision-effect vocabulary. No mandatory/advisory mapping,
  aggregation precedence or operational interpretation is approved.
- Minimized immutable source-fact references and evaluation request/check-result/result structures.
- An unavailable or unevaluated check cannot carry `ALLOW`, `BLOCK` or another decision effect.

### Inactive outbound fact-query contracts

The current source-contract review supports structural, implementation-free queries for:

- Fleet-owned Vehicle document facts;
- Fleet-owned Driver licence and minimized fitness outcomes;
- Freight-owned cargo/customs/hazmat facts; and
- Billing-owned supplied tax facts.

Every contract is Tenant- and effective-time-qualified. It carries logical IDs and minimized facts only;
Compliance may not query another module's repository or table. No adapter is registered and no runtime
query occurs in CS01. Regional-operation and retention-disposition ports remain deferred because their
source semantics are not approved.

## Policy and activation hold

D1–D11 remain proposed. Product and qualified policy authority must approve jurisdiction/policy scope,
facts, mandatory/advisory classification, effects, precedence, authority, override/appeal, retention,
privacy, security and acceptance before CS02 or any active behavior. Missing authority/configuration is
unavailable/unevaluated; it is neither affirmative clearance nor an automatic operational block.

## Phase 2: Post-MVP / Future Roadmap

No Phase-2 behavior was introduced by CS01. Any generic rule language, external rules feed or additional
jurisdiction remains separately governed.
