# Hexagonal Architecture Rules

Status: Mandatory
Applies to: All enterprise-suite modules
Last reviewed: 2026-08-26

## Objective

Each business module is an independently evolvable bounded context. The suite may be deployed as a modular monolith or as separately deployed applications, but its source architecture remains `domain <- application/ports <- adapters`.

## Required Layers

### Domain

- Owns aggregates, entities, value objects, policies, domain services, domain events, and business exceptions.
- Uses only the language standard library and approved pure domain primitives.
- Must not import Spring, Jakarta Persistence, Hibernate, Jackson, HTTP, SQL, messaging, filesystem, or adapter types.
- Enforces tenant ownership and invariants in aggregate behavior.

### Ports

- Inbound ports expose cohesive use cases, commands, and queries.
- Outbound ports express persistence, clock, identity, event publication, and external-provider needs.
- Ports use domain or provider-neutral application types only.
- Ports must not expose JPA entities, framework transactions, HTTP requests/responses, or vendor SDK types.

### Application

- Orchestrates domain behavior through ports.
- Defines authorization-relevant intent, idempotency, consistency boundaries, and application DTOs.
- Receives tenant context explicitly; ambient context may be used only by an inbound adapter to construct that input.
- Does not access another module's repositories or tables.

### Adapters

- Inbound adapters resolve authentication, tenant context, validation, protocol mapping, and correlation IDs before invoking inbound ports.
- Outbound adapters implement ports for PostgreSQL, event buses, and external APIs.
- Persistence adapters map explicitly between JPA models and domain objects.
- Event adapters map provider-neutral domain/application events to versioned integration envelopes.

## Module Boundary Rules

- A module owns its domain model, schema, migrations, APIs, and published events.
- Direct repository injection, SQL, JPA relationships, or physical foreign keys across module boundaries are forbidden for new work.
- Synchronous integration uses a documented API or focused outbound port implemented by an adapter.
- Asynchronous integration uses a registered, versioned event contract.
- Shared code is limited to stable technical primitives. Business concepts remain in their owning bounded context.

## Transactions and Consistency

- A local transaction may update only one module-owned consistency boundary.
- Cross-module workflows use eventual consistency, idempotent consumers, retries, and compensating actions.
- Use an outbox/inbox pattern before claiming reliable event delivery.
- Optimistic concurrency requires an explicit version field and conflict behavior.

## Change Gate

Before implementation, identify the owning module, aggregate, inbound port, outbound ports, affected API/event contracts, tenant boundary, schema change, and knowledge-base files. Stop for architectural approval if ownership or transaction semantics are unclear.

## Automated Enforcement Baseline

Architecture Remediation Batch `P0-01` establishes a mandatory automated boundary gate in `ModulithBoundaryEnforcementTest`:

- Spring Modulith application-module verification and an acyclic top-level module graph;
- no dependency on another module's Spring Data repository;
- no JPA entity reference to an entity owned by another module;
- no operational core-domain dependency on adapters, infrastructure, or platform modules;
- no Reporting dependency on operational persistence packages; and
- a temporary, explicit legacy baseline for current top-level module dependency edges, with `shared` limited to the approved technical role.

The enforcement is architecture-only. It introduces no runtime behavior, REST contract, event contract, or database-schema change. Baseline membership is not architectural approval: P0-02/P0-03 must replace or govern legacy edges and remove them incrementally. New cross-module edges must be approved in the dependency registry before the baseline is expanded.

## Database Ownership Enforcement Baseline

Architecture Remediation Batch `P0-02` assigns every current Flyway table to exactly one top-level module and adds `DatabaseTableOwnershipArchitectureTest`. The gate requires exact Flyway-table registry coverage, keeps JPA table mappings and repositories inside the declared owner, and rejects new production JDBC access to foreign tables.

Two observed direct-SQL paths are recorded as violations, not approvals: Freight reporting joins Fleet-owned `vehicle`, and the platform `system` sample-data bootstrap probes Organization-owned `customer` before executing a multi-owner fixture. Their exact file/table pairs are temporarily frozen so access cannot expand. Replacing them requires owner-provided reporting/bootstrap contracts or owner-maintained read models and is reserved for P0-03. Historical migrations and physical foreign keys remain immutable factual legacy state.

P0-02 verification is complete: the bootstrap expectation was reconciled to the 128 permissions supplied by the current approved migration set, the architecture/bootstrap gate passed 36/36, and the full Java 21 suite passed 1,167 tests with 31 existing conditional skips.

## Published Contract Enforcement

Architecture Remediation Batch `P0-03` replaces the two baselined production-code violations. Freight reporting resolves Fleet-owned vehicle capacity facts through the published `FleetReportingQuery` contract instead of joining `vehicle`. The development sample-data bootstrap asks the Organization-owned `CustomerDataReadiness` contract whether customer data exists instead of directly querying `customer`; the existing opt-in multi-owner SQL fixture remains development provisioning infrastructure and does not grant runtime table ownership.

`ModulithBoundaryEnforcementTest` now treats the recorded dependency set as the approved graph and requires every cross-module Java dependency to target a type published directly in the provider module's root package. `DatabaseTableOwnershipArchitectureTest` has no remaining production foreign-SQL baseline. New dependency edges, nested implementation imports, foreign repositories, foreign JPA entities, and foreign production SQL remain prohibited.

Development PostgreSQL provisioning executes the idempotent `postgresql-sample-data.sql` fixture on every enabled application startup after Flyway has established the schema. Both Docker and local `run.sh` startup explicitly enable this development-only bootstrap. PostgreSQL's pre-schema container initialization directory is not used for the multi-owner fixture; H2 tests retain their dialect-specific fixture. Because the fixture is conflict-safe, the former cross-module Organization readiness query is no longer required.

P0-03 verification is complete: compilation passed, the focused architecture and affected integration tests passed, and the full Java 21 regression suite passed 1,168 tests with 31 existing conditional skips. The full-suite command explicitly disabled opt-in development sample-data for unrelated test contexts; `LocalSampleDataBootstrapIntegrationTest` overrides that property and passed with the fixture enabled.

## Aggregate Boundary Enforcement

Architecture Remediation Batch `P0-06` establishes an explicit aggregate-object-graph gate. Cross-aggregate and cross-module references remain UUID/value references; new JPA entity associations or cascades are rejected unless the parent/child ownership is deliberately added to the reviewed aggregate baseline. The current baseline contains only persistence children whose lifecycle is owned by their parent aggregate, including Delivery Exception evidence and the existing Freight aggregate children.

Vehicle is a narrow Fleet aggregate root and must receive explicit Category and Type aggregate references; it never fabricates reference identifiers and does not own documents, readings, maintenance, fuel, Trip, Driver, or Delivery history. Driver updates preserve the addressed aggregate-root identity; qualification, availability, compliance, and history records retain their separate lifecycles and repositories. Trip remains one aggregate for its currently coupled order, assignment-state, and execution lifecycle invariants; audit/dispatch/operational-event records remain separately persisted. A mechanical Trip split is not approved because it would require broader transaction/event redesign owned by P0-07.

P0-06 changes no REST contract, event contract, database schema, Tenant authority, or authorization behavior. Verification passed the focused domain/application/architecture gate (52 tests) and the full Java 21 regression suite (1,185 tests with 31 existing conditional skips).

## Transaction and Event Consistency Enforcement

Architecture Remediation Batch `P0-07` classifies invariant-protecting writes as owning-module ACID work, local secondary reactions as after-commit events, and independently retryable external delivery as durable asynchronous work. Business-critical writes must not be downgraded to events. Spring-local events must use the published `shared.AfterCommitEventPublisher` contract; direct module use of `ApplicationEventPublisher` is rejected by `ModulithBoundaryEnforcementTest`.

The shared Spring adapter registers publication with the active transaction and emits only after commit. Rollback suppresses publication, while a secondary consumer exception is isolated after commit. Calls made outside a transaction publish immediately. Notification owns Tenant enrichment for `OperationalNotificationEvent`; Trip and Fleet depend only on its published provider-neutral contract. This preserves the approved dependency graph and prevents domain/application packages from depending on Spring event infrastructure.

P0-07 introduces no broker, distributed transaction, saga, or database schema. Local Spring events remain non-durable. Any future external integration that requires independent retry or guaranteed delivery must first approve a database outbox/inbox contract with per-Tenant event identity, aggregate-local ordering, retry, retention, and idempotency semantics.
