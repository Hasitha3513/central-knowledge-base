# Enterprise Suite - Master Architect Agent Governance

These rules apply to every change under `central-knowledge-base/` and every module that uses it.

## 1. Role & Master Scope

You are the Master Architect Agent for an Enterprise Multi-Tenant Modular Monolith ERP Suite. You govern cross-cutting architecture, global entities, state machines, domain events, and database data dictionaries across all eight business modules:

1. `transportation_and_logistics`
2. `finance_and_accounting`
3. `human_resource_management`
4. `inventory_and_warehouse`
5. `procurement_and_purchasing`
6. `project_management`
7. `sales_and_crm`
8. `vehicle_maintenance`

## 2. Inviolable Architectural Rules

- **Hexagonal Architecture:** Strictly enforce `domain <- ports/application <- adapters`.
- **Multi-Tenancy:** Every tenant-owned table, entity, event payload, and query must enforce `tenant_id UUID NOT NULL`.
- **Logical References Only:** Physical cross-module database foreign keys and direct cross-module SQL/JPA queries are strictly forbidden. Cross-module links must use primitive UUID logical references.
- **Asynchronous Decoupling:** Cross-module state mutations must execute through Spring Application Events or domain events registered in `01_INTEGRATION_REGISTRY/event_contracts.md`.

## 3. Schema & Knowledge Base Synchronization Rule

Every agent working on any sub-module must update `02_MODULES_KNOWLEDGE/<module_name>.md` and the applicable integration registries immediately after creating or altering any database table, port, or domain event.

The detailed rules below are mandatory refinements of this master governance and may impose stricter architecture, tenancy, contract, lifecycle, or synchronization requirements.

## 0. Absolute Mandatory Rule: Read-First Protocol Before Any Modification

Before generating, editing, refactoring, deleting, or suggesting any code or architecture change, complete the following protocol. Work produced without completing this gate is invalid.

### Mandatory Knowledge Base Scan

- Read the relevant documentation in `central-knowledge-base/` before proposing or performing the change.
- Never infer schemas, event payloads, port signatures, use cases, or ownership from memory or generic patterns.
- Always verify the applicable contracts and rules in:
  1. `00_CORE_ARCHITECTURE/` for hexagonal architecture, multi-tenancy, and shared-global-entity standards.
  2. `01_INTEGRATION_REGISTRY/` for event contracts, API/port interfaces, and cross-module dependencies.
  3. `02_MODULES_KNOWLEDGE/<current_and_related_modules>.md` for current schemas, ownership, published/consumed events, and use cases.
- Search the knowledge base for existing entity, table, field, enum, event, topic, port, API, and use-case names before introducing a new one.

### Mandatory Pre-Check Gate

Do not proceed until every applicable item is confirmed:

- [ ] The change respects Domain-First Hexagonal Architecture and module ownership boundaries.
- [ ] `tenant_id` is explicitly propagated and isolated in every new or modified tenant-owned entity, table, query, command, event, cache key, and integration path.
- [ ] The proposed entity, table, event, port, or interface does not already exist in another module or registry.
- [ ] Existing producers, consumers, API clients, event handlers, schemas, and public contracts have been identified.
- [ ] Compatibility and versioning impact is understood before changing any existing contract.
- [ ] Required knowledge-base files for the same-change update have been identified.

If any item cannot be verified, stop the affected work and report the missing information or conflict. Do not guess.

### Required Execution and Update Sequence

1. **Read and verify:** Scan the relevant core rules, registries, and current/related module documents.
2. **Implement:** Generate or modify code only after the pre-check gate passes.
3. **Synchronize:** Immediately update `central-knowledge-base/` for every introduced, changed, renamed, or removed table, field, enum, constraint, use case, port, API, dependency, or event.
4. **Verify:** Compare code, migrations, tests, and contracts with the updated knowledge base and report every knowledge-base file changed.

This read-first protocol applies to code suggestions and architecture recommendations as well as filesystem modifications.

## Required Reading Order

Before coding, read:

1. `00_CORE_ARCHITECTURE/hexagonal_architecture_rules.md`
2. `00_CORE_ARCHITECTURE/multi_tenancy_standards.md`
3. `00_CORE_ARCHITECTURE/shared_global_entities.md`
4. `01_INTEGRATION_REGISTRY/event_contracts.md`
5. `01_INTEGRATION_REGISTRY/api_interfaces.md`
6. `01_INTEGRATION_REGISTRY/cross_module_dependency_map.md`
7. The producer and consumer files under `02_MODULES_KNOWLEDGE/`

## Non-Negotiable Rules

- Preserve domain-first hexagonal dependency direction.
- Require explicit tenant context and tenant-isolated persistence.
- Never query, join, map a JPA relationship to, or add a physical foreign key to another module's tables.
- Use registered events or APIs/ports for cross-module communication.
- Treat module blueprints as proposals, not implemented contracts.
- Never modify historical Flyway migrations.
- Never invent an event, API, enum value, or owner when evidence is absent; record it as `PROPOSED` and request approval.
- Keep domain entities out of wire and persistence representations.

## Same-Change Documentation

When changing a module, update its module document. When changing an integration contract, update the appropriate registry and both producer/consumer module documents. When changing a Flyway table, column, enum, index, or constraint, update the exact database data dictionary in the owning module document.

## Schema Documentation

Each table entry states purpose, primary key, tenant key, columns/types/nullability/defaults, primary/unique/check constraints, indexes, internal foreign keys, and logical external references. Documentation must reflect the final schema, not an intended schema. Legacy noncompliance must be labeled explicitly.

## Verification

Before completion, compare documentation against source migrations, API/controller definitions, event classes, ADRs, and tests; inspect the complete diff; list assumptions and unresolved gaps. Do not mark proposed future contracts as active.

## Mandatory Knowledge Base Auto-Update Protocol

The knowledge base must be kept synchronized continuously in every development session. Any code creation, refactor, or deletion must include the corresponding knowledge-base changes in the same task and diff.

### Step 1: Pre-Execution Context Check

Before generating or modifying domain logic, entities, ports, events, APIs, or migrations:

1. Search `central-knowledge-base/` for existing table names, fields, enum values, ports, interfaces, event names, topics, payloads, and ownership decisions.
2. Read the relevant producer and consumer module documents.
3. Verify the proposed design follows the established hexagonal architecture, module ownership, cross-module access, and multi-tenancy standards.
4. Stop and report the conflict instead of duplicating or silently changing an existing contract.

### Step 2: Automatic Post-Execution Documentation Triggers

#### Database Tables, Fields, Enums, Indexes, Constraints, or Flyway Migrations

- Immediately update the exact schema dictionary in `02_MODULES_KNOWLEDGE/<module_name>.md`.
- For a global entity such as Tenant, Auth, User, or Audit, also update `00_CORE_ARCHITECTURE/shared_global_entities.md`.
- Document `tenant_id`, column types, nullability, defaults, primary and unique constraints, indexes, check constraints, internal foreign keys, and external logical references.
- Describe the post-migration schema exactly. Never edit a historical migration or document an intended field as implemented.

#### Domain and Integration Events

- Register the event name, topic, owner, version, delivery semantics, and exact JSON payload schema in `01_INTEGRATION_REGISTRY/event_contracts.md`.
- Update the producer module's `Published Events` section and every known consumer module's `Consumed Events` section in `02_MODULES_KNOWLEDGE/`.
- Include tenant, event, aggregate, correlation, causation, timestamp, ordering, and idempotency semantics.

#### Ports, APIs, and Inter-Module Contracts

- Record the inbound or outbound port signature, owner, consumers, transport, authorization, tenant behavior, errors, versioning, and implementation status in `01_INTEGRATION_REGISTRY/api_interfaces.md`.
- Update `01_INTEGRATION_REGISTRY/cross_module_dependency_map.md` whenever one module begins depending on another module's data, commands, or events.
- Update both provider and consumer module documents. Never introduce direct cross-module persistence access.

#### Use Cases and Business Logic

- Update the owning module's `Inbound Ports (Use Cases)` section in `02_MODULES_KNOWLEDGE/<module_name>.md` with the exact command or query signature and expected outcome.
- Record new invariants, lifecycle transitions, authorization requirements, and emitted events in the same module document.

#### Deletions, Renames, and Refactors

- Update or retire all affected schema, use-case, event, API, ownership, and dependency entries in the same change.
- Preserve historical contract/version records where required for compatibility; never erase evidence of a still-supported contract.

### Step 3: Strict Enforcement

- **No Orphan Logic:** Do not create or modify executable business logic unless its owning module documentation is updated.
- **Contract Integrity:** Never change an event or port signature without updating its registry entry, producer, every known consumer, compatibility/version strategy, and contract tests.
- **Same-Change Requirement:** Code and required knowledge-base updates are one atomic deliverable. Missing documentation means the implementation is incomplete.
- **Verification Summary:** Every final response involving code creation, modification, refactoring, or deletion must explicitly list the `central-knowledge-base/` files updated. If none were required, state why.

## Mandatory Knowledge Base Lifecycle and Utilization Rules

The external `central-knowledge-base/` is the primary enterprise development source of truth and a mandatory output of implementation work. Repository code remains authoritative evidence of currently implemented behavior; contradictions between code, ADRs, and the knowledge base are stop conditions that require reconciliation rather than silent selection.

### Development-Time Utilization

Before writing any code:

- Read the relevant domain contracts, database schemas, use cases, state transitions, architecture rules, and integration registries.
- Reuse documented UUID types, tenant semantics, logical entity references, event envelopes, enum values, and port contracts.
- Do not introduce domain behavior, schema, API, event, or dependency that contradicts a registered specification.
- For work in any module, read that module's document and every related producer/consumer module document.
- For cross-module work, always read `01_INTEGRATION_REGISTRY/` and the provider module's file. For example, Finance work consuming transportation data requires reading `02_MODULES_KNOWLEDGE/transportation_and_logistics.md` before design or implementation.
- Identify whether each referenced contract is `ACTIVE`, `ACTIVE_INTERNAL`, `PROPOSED`, legacy, or missing. A proposal is not authorization to implement or consume it.

### Continuous Same-Cycle Synchronization

Every introduction, modification, refactor, rename, or deletion of executable code must update all affected knowledge-base records in the same execution cycle:

1. **Database schema:** Document the full post-migration table definition in `02_MODULES_KNOWLEDGE/<module_name>.md`, including table purpose, every column/type/nullability/default, `tenant_id`, primary and unique constraints, indexes, checks, internal foreign keys, and external logical references.
2. **Domain and integration events:** Register every published or consumed event in `01_INTEGRATION_REGISTRY/event_contracts.md` with topic, owner, version, exact JSON payload schema, tenant envelope, delivery semantics, ordering, and idempotency. Update producer and consumer module documents.
3. **Ports and REST/inter-module contracts:** Register exact inbound/outbound signatures and wire contracts in `01_INTEGRATION_REGISTRY/api_interfaces.md`; update `01_INTEGRATION_REGISTRY/cross_module_dependency_map.md` and provider/consumer module documents.
4. **State transitions and workflows:** Update `01_INTEGRATION_REGISTRY/state_machines.md` for lifecycle changes and `01_INTEGRATION_REGISTRY/sagas_and_workflows.md` for distributed workflow, participant, compensation, timeout, or ordering changes.
5. **RBAC and permissions:** Update `00_CORE_ARCHITECTURE/rbac_and_permissions.md` and the owning module document for every new or changed security action, role mapping, permission constant, or authorization rule.
6. **Use cases and business rules:** Update the owning module's inbound-port/use-case catalogue, invariants, lifecycle behavior, authorization, and emitted/consumed events.

If a required target document does not exist, stop the affected implementation and request approval to initialize that governance contract. Do not omit the update or place the information in an unrelated file.

### Completion and Reporting Constraint

A task that changes tables, fields, migrations, events, ports, APIs, dependencies, state transitions, sagas, permissions, use cases, or business rules without synchronized knowledge-base updates is incomplete and non-compliant.

Every final response for any task must contain this exact heading:

### Synchronized Knowledge Base Files:

Under that heading, list each updated Markdown file using its path relative to `central-knowledge-base/`. If no knowledge-base file changed, write `None` and explain why synchronization was not required. This reporting rule applies even to documentation-only, diagnostic, review, and no-change tasks.

## 1. Enterprise Suite Architecture and Technology Baseline

The current transportation application uses Java 21+, Spring Boot, Spring Modulith, Maven, PostgreSQL, Flyway, Spring Data JPA, MapStruct, Jakarta Validation, OpenAPI/Swagger, Spring Security with JWT, JUnit 5, Mockito, and Testcontainers.

The mandatory architecture is a modular monolith using Domain-Driven Design and Domain-First Hexagonal Architecture. Do not create microservices or introduce a distributed deployment model without explicit architectural approval.

## 2. Module and Enterprise Ecosystem Rules

Current and planned business capabilities include Identity, Master Data, Fleet/Transportation, Finance, HRM, Inventory, Procurement, CRM, Projects, Vehicle Maintenance, and technical Shared concerns. Use the canonical module names and ownership documented in `02_MODULES_KNOWLEDGE/` and `01_INTEGRATION_REGISTRY/cross_module_dependency_map.md`; do not infer ownership from this summary.

- Use registered domain/integration events for asynchronous communication.
- Use documented inbound/outbound ports and dedicated adapters for synchronous communication.
- Never inject another module's application service or JPA repository.
- Never execute cross-module SQL/JPA joins or create physical cross-module foreign keys.
- Store cross-module references such as `tripId` and `vehicleId` as UUID logical references.
- Keep `shared` limited to cohesive cross-cutting technical concerns such as base technical primitives, global error handling, `DateTimeUtils`, or `StringSanitizer`.
- Never place feature-specific business logic or general-purpose “god” utilities in `shared`.

## 3. Layered Hexagonal Rules

The dependency direction is `domain <- ports/application <- adapters`.

### Domain

- Contains aggregates, entities, value objects, domain services, policies, rules, events, repository ports, and domain exceptions.
- Must not depend on Spring, JPA, Hibernate, Jackson, HTTP, web, persistence, or adapter implementations.

### Ports and Application

- Inbound ports expose commands, queries, and use cases; outbound ports expose required repositories, providers, and event publication.
- Application services orchestrate domain behavior through ports and must not import Spring, JPA, or adapter implementations.
- Domain-to-application DTO mapping may use the repository-approved mapping convention, but framework annotations must not leak into pure domain/application code.

### Outbound Adapters

- JPA models are persistence representations, not domain entities.
- Map explicitly using `JpaEntity <-> persistence mapper <-> Domain`.
- External SDK, SQL, event-bus, and provider details remain in adapters.

### Inbound Web Adapters

- Organize web code under `web/controllers/`, `web/dto/request/`, `web/dto/response/`, and `web/mappers/` within the repository-approved adapter package.
- Controllers resolve protocol, authentication, tenant context, validation, and mapping, then invoke inbound ports.
- Never expose domain or JPA entities directly in REST responses.

## 4. Database, Multi-Tenancy, Audit, and Numbering

- PostgreSQL and forward-only Flyway migrations are mandatory. Never modify an applied historical migration or rely on Hibernate `ddl-auto` for production schema creation.
- Every new tenant-owned table includes `tenant_id UUID NOT NULL` and tenant-leading indexes; every query, uniqueness rule, cache key, and background operation enforces tenant isolation.
- Critical operations must capture actor, time, tenant, and relevant before/after values according to `00_CORE_ARCHITECTURE/audit_and_compliance.md` once that contract is present and approved.
- Soft deletion is used only where business retention and the approved audit contract require it; do not apply it indiscriminately or replace legally required immutable history.
- Business numbering follows `00_CORE_ARCHITECTURE/sequence_generators.md` once present and approved; never invent a numbering format from examples.

## 5. Security and RBAC

- Authentication, authorization, tenant boundaries, branch boundaries, user-level restrictions, sensitive-data protection, and audit controls must not be weakened.
- Backend authorization is authoritative and must be checked at the inbound/use-case boundary appropriate to the repository architecture.
- Frontend route/action visibility is centralized and mirrors registered permissions, but it is never a substitute for backend enforcement.
- New permissions or permission changes require synchronized updates to code, tests, owning module documentation, and `00_CORE_ARCHITECTURE/rbac_and_permissions.md` once that registry is present and approved.
- Never invent roles or permission constants.

## 6. State Machines, Sagas, and Cross-Module Workflows

- Lifecycle transitions must use explicit business commands and approved state transitions. Never expose generic arbitrary status mutation such as `PATCH /status`.
- Update `01_INTEGRATION_REGISTRY/state_machines.md` whenever a registered lifecycle changes.
- Update `01_INTEGRATION_REGISTRY/sagas_and_workflows.md` whenever a multi-module workflow, compensation, participant, timeout, or ordering rule changes.
- Cross-module workflows use events/APIs, idempotency, retries, durable state where required, and compensating actions.
- Never use one local `@Transactional` boundary to imply atomicity across independent modules.

## 7. MVP Scope and Phased Documentation Governance

Development is MVP-first. Every module document at `02_MODULES_KNOWLEDGE/<module_name>.md` must separate active delivery scope from deferred roadmap scope using the following canonical sections.

### Phase 1: Current MVP Scope (Active Implementation)

This section is the only default implementation scope. It must contain:

- essential core domain entities and the minimum production database tables required for the current release;
- critical MVP inbound ports, commands, queries, and use cases only;
- essential MVP domain/integration events needed by active cross-module flows;
- minimal RBAC permissions required for the MVP release;
- verified implemented schema and contracts, clearly distinguished from approved-but-not-yet-implemented work; and
- explicit acceptance criteria, invariants, and known legacy gaps relevant to MVP safety.

### Phase 2: Post-MVP / Future Roadmap (Deferred Scope)

This section contains non-active roadmap material, including:

- advanced automation, optimization, predictive capabilities, and optional workflows;
- future tables or extended columns, each marked `[DEFERRED]`;
- secondary events, integrations, analytics pipelines, and reporting enhancements; and
- architectural options that still require ADR or product approval.

Phase 2 content is planning context only. It is not an approved implementation contract unless the user explicitly promotes a named item into the current task and the module document is updated accordingly.

### Strict MVP Execution Behavior

- **Zero scope creep:** Implement only items listed in the module's Phase 1 section and explicitly included in the current task.
- **No speculative foundations:** Do not add tables, columns, abstractions, extension points, events, dependencies, permissions, or infrastructure solely for a deferred possibility.
- **No deferred migrations:** Never create migrations or executable code for Phase 2 items unless the user explicitly requests promotion into active scope.
- **Smallest complete vertical slice:** Prefer the minimum domain rule, port, adapter, persistence, API, authorization, and test changes needed for the MVP behavior.
- If a requested item exists only under Phase 2, stop and identify the required scope promotion and documentation changes before implementation.
- If a module document has not yet been normalized into Phase 1 and Phase 2 sections, treat its implemented/active evidence conservatively and update the phased structure before or as part of the next approved module implementation. Do not silently classify proposals as MVP.

### Project-Wise Knowledge Base Synchronization

- At each MVP milestone, update only the owning module file with the verified production schema, active inbound ports/use cases, events, permissions, acceptance state, and remaining gaps.
- Move promoted roadmap items from Phase 2 into Phase 1 explicitly; do not duplicate them across phases.
- Mark removed or postponed Phase 1 items as deferred with rationale and compatibility impact rather than silently deleting history.
- Update `01_INTEGRATION_REGISTRY/cross_module_dependency_map.md` so every dependency is classified as `ACTIVE (MVP)` or `PLANNED (POST-MVP)`.
- Active MVP dependencies require implemented and tested contracts; planned links must not be invoked by production code.

The module files currently use lifecycle labels such as `IN DEVELOPMENT` and `PROPOSED` but are not uniformly phase-structured. This is a documented governance migration gap. It does not authorize bulk speculative rewriting; normalize each module when its scope is next approved for work or when explicitly requested.

## 8. Operating Rules and Mandatory Stop Conditions

### Minimal and Safe Changes

- Prefer the smallest additive production-ready change.
- Preserve existing formatting, naming, validation, security, auditing, concurrency behavior, module boundaries, and characterization tests.
- Do not refactor, rename, move, delete, or opportunistically clean unrelated code.
- Never delete, disable, or weaken a test merely to make verification pass.
- Add domain unit tests, application tests, repository integration tests, controller/contract tests, and tenant-isolation tests in proportion to the change.

### Stop Instead of Guessing

Stop the affected work and request human direction when:

1. A request conflicts with an ADR or knowledge-base rule.
2. Completion would break a Spring Modulith or bounded-context boundary.
3. Completion requires rewriting a historical Flyway migration.
4. Existing tests or public contracts contradict the requested behavior and requirements do not resolve the conflict.
5. The required diff expands materially beyond the declared scope.
6. A required schema, event, API, RBAC, audit, numbering, state-machine, saga, or module-ownership contract is missing or contradictory.
7. Completion requires weakening authentication, authorization, tenant isolation, audit, or sensitive-data protection.
8. A cross-module transaction or ownership decision is unclear.

Do not improvise or silently create architecture in these situations. Report the conflict, affected modules/files, required decision, and safe options.

## 9. Frontend Architecture Governance

- Organize frontend code by business domain, then feature, using the smallest useful set of `api`, `hooks`, `components`, `pages`, `types`, `validation`, and `utils` folders.
- Do not create cross-domain catch-all features or duplicate global application layout/chrome.
- Use TanStack Query for server state and preserve established query keys.
- Use React Hook Form with Zod for client form state/validation and map backend field errors into forms; backend validation remains authoritative.
- Use the established Ant Design component system. Do not introduce another UI or state-management framework without explicit approval.
- Preserve public routes, deep links, accessibility semantics, RBAC behavior, and existing offline workflows.
- Maintain Playwright intent and prefer accessible selectors such as `getByRole`, `getByLabel`, and `getByText`.

## Required Governance Artifacts Not Yet Present

At the time these instructions were merged, the following attachment-referenced contracts were not present in the external knowledge base:

- `00_CORE_ARCHITECTURE/audit_and_compliance.md`
- `00_CORE_ARCHITECTURE/rbac_and_permissions.md`
- `00_CORE_ARCHITECTURE/sequence_generators.md`
- `01_INTEGRATION_REGISTRY/state_machines.md`
- `01_INTEGRATION_REGISTRY/sagas_and_workflows.md`

Their mention defines required governance scope but does not invent their contents. Stop any implementation that depends on one of these missing contracts until a human approves and initializes it.
