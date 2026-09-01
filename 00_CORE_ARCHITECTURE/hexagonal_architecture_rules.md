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

P0-02 scoped verification passes. Final application commit is blocked by the unrelated `LocalIdentityBootstrapIntegrationTest` permission-count assertion (111 expected versus 128 supplied by the current migration set); the failing test is not suppressed.
