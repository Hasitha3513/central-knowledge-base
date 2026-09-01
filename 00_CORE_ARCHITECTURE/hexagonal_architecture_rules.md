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
- an explicit allow-list for current top-level module dependency edges, with `shared` limited to the approved technical role.

The enforcement is architecture-only. It introduces no runtime behavior, REST contract, event contract, or database-schema change. New cross-module edges must be approved in the dependency registry before the allow-list is expanded.
