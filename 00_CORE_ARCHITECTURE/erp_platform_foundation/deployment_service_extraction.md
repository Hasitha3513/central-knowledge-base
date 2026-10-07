# Deployment & Service Extraction Assessment

Status: **RECOMMENDED — OWNER DECISION REQUIRED**.

## Core alternatives

| Criterion | A: Modular ERP core | B: Microservice per domain | C: Modular core + selective extraction |
| :--- | :--- | :--- | :--- |
| Latency/consistency | Local calls/transactions where co-owned | Network and eventual consistency pervasive | Local by default; distributed only where justified |
| Scaling/failure isolation | Coarse deployment scaling/blast radius | Strong potential isolation at high operational cost | Targeted isolation for measured hotspots |
| Data ownership | Requires module enforcement | Physical ownership clearer but contracts multiply | Logical ownership first; physical separation when ready |
| Release/testing | Simpler release and end-to-end matrix | Many compatibility/deployment matrices | Moderate; requires contract tests at extracted boundaries |
| Operations/observability | Lowest initial burden | Highest tracing, SLO, on-call and incident burden | Adds burden only for justified capabilities |
| Productivity/future extraction | Fast start if boundaries enforced | Slow cross-domain change and local setup | Recommended balance; extraction seams designed, not pre-paid |

**Recommendation:** C. High usage or a domain name alone is not evidence for a service.

## Existing Transportation alternatives

| Alternative | Benefit | Risk | Assessment |
| :--- | :--- | :--- | :--- |
| T1 preserve independently deployable Transportation | Protects accepted investment, bounded migration and release independence | Needs explicit contracts and temporary compatibility | **RECOMMENDED — OWNER DECISION REQUIRED** |
| T2 merge into ERP Core | Fewer deployments and potentially local calls | Large regression/migration blast radius; blurs current ownership | Not recommended without evidence |
| T3 decompose first | Potential isolated scaling | Maximum premature distribution and delays reconciliation | Not recommended without capability evidence |

## SERVICE_EXTRACTION_SCORECARD

No universal numeric threshold applies. Each capability ADR must explain evidence both for benefits and distribution costs.

| Dimension | Evidence supporting extraction | Counterweight to assess |
| :--- | :--- | :--- |
| Independent scaling | Sustained, measured workload shape materially differs | Can vertical/horizontal modular deployment suffice? |
| Workload specialization | Kafka/streaming/search/file/compute/storage requirements | Is an adapter/store boundary enough without service split? |
| Availability isolation | Clear SLO and failure containment need | New network dependency may reduce end-to-end availability |
| Security/compliance isolation | Distinct privileges/data classifications/control operators | More endpoints, secrets and attack surface |
| Deployment cadence | Independent releases unblock real teams/work | Contract compatibility and coordinated business change |
| Data ownership | Aggregate/persistence cleanly separable | Synchronous coupling or shared transactions signal poor seam |
| Cross-domain coupling | Few stable inbound/outbound contracts | Chatty calls create a distributed monolith |
| Consistency cost | Eventual consistency is acceptable and reconcilable | Safety/financial invariants may need local consistency |
| Operational maturity | SLOs, tracing, alerting, runbooks and on-call owner exist | Unsupported service fleet creates reliability debt |
| Observability readiness | Correlation, metrics and replay/reconciliation proven | Invisible partial failure is unacceptable |

## Candidate specialized capabilities

| Capability | Current evidence | Classification |
| :--- | :--- | :--- |
| Tracking/Telemetry | Accepted Kafka durable buffer, Redis live projection, Timescale history, PostgreSQL control state | **EXISTING SPECIALIZED INFRASTRUCTURE**; independent deployment remains owner decision |
| Notifications | Implemented Transportation Notification with channel retry/suppression | **CANDIDATE FOR EXTRACTION / EVIDENCE REQUIRED** |
| Documents/File processing | Domain-specific evidence exists; no canonical generic owner | **CANDIDATE FOR EXTRACTION / EVIDENCE REQUIRED** |
| Search | No approved enterprise capability evidence | **KEEP IN MODULAR CORE / EVIDENCE REQUIRED** |
| Analytics/Reporting | Domain analytics and composition exist | **CANDIDATE FOR EXTRACTION / EVIDENCE REQUIRED** |
| Integration Gateway | Accepted dedicated Integration context and controlled adapter | **CANDIDATE FOR INDEPENDENT DEPLOYMENT / EVIDENCE REQUIRED** |
| Identity/Auth | Current Transport Identity; target transition unresolved | **CANDIDATE FOR EXTRACTION / SECURITY-AVAILABILITY EVIDENCE REQUIRED** |
| Background/scheduling | Context-owned jobs exist | **KEEP WITH OWNER unless workload/isolation evidence justifies shared infrastructure** |

## Common frontend

**RECOMMENDED — OWNER DECISION REQUIRED:** one customer ERP frontend codebase/product experience. Visibility is the intersection of organization capability entitlement, feature activation/configuration, user permission, resource scope and localization. Backend enforcement remains authoritative. Transportation may stay a separate backend without a separate login/product. Platform Management keeps a separate provider-privileged interface.

## Platform Identity staged transition

1. Inventory current Transport Account, Tenant membership, role/permission, JWT, actor and audit contracts.
2. Approve target authority, token trust, one-account-one-organization enforcement and provider/customer separation.
3. Establish stable account/tenant mappings and compatibility validation without copying credentials.
4. Pilot one bounded consumer and prove revocation, tenant isolation, rollback and audit correlation.
5. Migrate relying applications incrementally; reconcile dual observations while one authority remains explicit.
6. Retire legacy authority only after dependent contracts and recovery are accepted.

Big-bang credential/JWT replacement and silent role-name mapping are not recommended.
